---
title: 'Reducing 48 Database Queries to 1: A Django API Optimization Story'
description: 'How I took a critical Django REST Framework endpoint from ~48 N+1 queries down to 11 cold / 1 warm, using phased query batching, BoolOr aggregation, and a three-tier cache.'
pubDate: 'Mar 25 2026'
---

This is the story of optimizing the single most critical API endpoint in a multi-tenant SaaS platform. The endpoint — let's call it `/api/v2/access/me` — was hit on every page load by every logged-in user. With 4,100+ active users, it was the hottest path in the system.

The v1 implementation had ~48 database queries per request. The v2 rewrite brought that down to 11 queries cold (first request) and 1 query warm (cached). Here's how.

## The Problem

The endpoint returns a user's access profile: which data sources they can see, their permission levels, associated resource profiles, active sessions, and group memberships. It's a complex response that touches 10+ database tables across a Django/DRF backend backed by PostgreSQL.

The v1 implementation used Django REST Framework serializers in the standard way. The root serializer had nested serializers. Those nested serializers had `SerializerMethodField`s that each made their own database queries. And the outermost loop iterated over data sources — so every per-source query was an N+1.

The worst offender was a method called `is_premium(source_id)` that was called once per data source. With 15–20 sources per customer, that's 15–20 identical-pattern queries that could have been a single aggregation.

## Step 1: Map Every Query

Before optimizing anything, I used Django's `CaptureQueriesContext` to record every SQL query the endpoint executed. I documented each one: which table it hit, which serializer triggered it, and whether it was duplicated across the loop.

The breakdown:
- **~15 queries** from `is_premium()` — one per data source (N+1)
- **~8 queries** from resource profile serialization — `select_related` was missing
- **~6 queries** from polymorphic model resolution — ContentType lookups
- **~4 queries** from group/permission checks
- **~15 miscellaneous** — VPC groups, session data, external app URLs

Total: ~48 queries for a typical user with 15 data sources.

## Step 2: Eliminate N+1 with BoolOr Aggregation

The `is_premium()` call was doing this per data source:

```python
def is_premium(self, user, source_id):
    return GroupSourcePremium.objects.filter(
        source_id=source_id,
        group__in=user.groups.all(),
    ).filter(
        Q(has_premium_dataset=True) | Q(has_vpc_group=True)
    ).exists()
```

Called 15 times = 15 queries. The fix was a single aggregation query upfront:

```python
premium_map = (
    GroupSourcePremium.objects
    .filter(
        source_id__in=source_ids,
        group__in=user.groups.all(),
    )
    .values('source_id')
    .annotate(
        any_premium_dataset=BoolOr('has_premium_dataset'),
        any_vpc_group=BoolOr('has_vpc_group'),
    )
)
```

One query. The result is a dict keyed by `source_id`. During response construction, it's a dict lookup instead of a database hit.

## Step 3: Bypass Polymorphic Overhead

The v1 code resolved the user's organization through Django's polymorphic model hierarchy: `Organization → Customer` via ContentType. Each resolution cost 4 queries (ContentType lookup, downcast, etc.).

The fix: use `user.organization_id` directly. We already know it's a Customer in this context — the endpoint is behind authentication middleware that guarantees it. No need for the ORM to prove it at runtime.

```python
# Before: 4 queries
customer = user.organization  # triggers polymorphic resolution

# After: 0 queries
customer_id = user.organization_id  # just an integer FK
```

## Step 4: Merge Redundant Queries

Several pieces of data were fetched in separate queries that could be merged:

**Data source details**: v1 fetched IDs in one query, then details in another. v2 uses a single `.values()` call with FK chain traversal:

```python
sources = (
    CustomerToSource.objects
    .filter(customer_id=customer_id)
    .select_related('source', 'source__organization', 'source__cloud_provider')
    .values(
        'source_id', 'source__organization__name',
        'source__organization__icon', 'source__cloud_provider__name',
        'fcap_url', 'remote_desktop_url', 'updated_at',
    )
    .order_by('source__rank')
)
```

**Active + recent sessions**: v1 made two separate queries. v2 combines them:

```python
sessions = (
    Session.objects
    .filter(user=user)
    .filter(
        Q(status='active') |
        Q(profile__source_id__in=source_ids, launched_successfully=True, status='inactive')
    )
    .select_related('profile')
    .order_by('profile__source_id', '-created_at')
)
```

## Step 5: Pre-fetch All Serializer Data

v1 let DRF serializers lazily fetch related objects. v2 pre-fetches everything upfront:

```python
profiles = (
    ResourceProfile.objects
    .filter(
        Q(user_assignments__user=user) |
        Q(entity_assignments__object_id=customer_id, is_default=True)
    )
    .select_related('source', 'source__organization')
    .distinct()
)

entity_images = (
    EntityResourceProfile.objects
    .filter(profile__in=profiles, object_id=customer_id)
    .select_related('image')
)
```

All data is fetched in the query phase. The response construction phase does zero database hits — it's pure dict building and list comprehension.

## Step 6: Three-Tier Caching

Not all data changes at the same rate. The caching strategy uses three tiers:

| Tier | Scope | TTL | What's cached |
|------|-------|-----|---------------|
| Customer-level | Shared across all users in an org | 24 hours | Data source details, cloud provider info, org metadata |
| Source-level | Shared across all users for a source | 24 hours | Source IDs, names, icons, rankings |
| User-level | Per-user | 1 hour | Premium status, resource profiles, group memberships, sessions |

One field is **never cached**: `has_premium_account`. This is the billing-critical flag that determines whether a user has an active paid account. It's always fetched live from the database, even on warm path:

```python
premium_accounts = (
    PremiumAccess.objects
    .filter(source_id__in=source_ids, user=user)
    .values_list('source_id', 'has_account')
)
```

This is the only query on the warm path. Everything else comes from cache.

## Step 7: Drop the Serializer

The final architectural change: I replaced the DRF serializer entirely with a view-level dict builder. DRF serializers are great for simple CRUD, but for a complex read-only endpoint with 20+ response fields sourced from 10+ pre-fetched querysets, they add overhead without value.

```python
def _build_response(self, user, sources, premium_map, profiles, ...):
    return [
        {
            'id': source['source_id'],
            'name': self._display_name(source, obfuscation),
            'premium_user': premium_map.get(source['source_id'], False),
            'has_premium_account': accounts.get(source['source_id'], False),
            'profiles': self._build_profiles(source['source_id'], profiles, images),
            # ...
        }
        for source in sources
    ]
```

Serialized with `orjson` for fast JSON encoding:

```python
return Response(
    orjson.loads(orjson.dumps(response_data)),
    status=200,
)
```

## The Results

| Metric | v1 | v2 (cold) | v2 (warm) |
|--------|-----|-----------|-----------|
| Database queries | ~48 | 11 | 1 |
| Query reduction | — | 77% | 98% |

The cold path runs 11 phased queries in a deterministic order. The warm path runs a single lightweight `values_list` query for the billing flag. Cache invalidation is handled by Django signals on the relevant models — when a user's groups change, their user-level cache is invalidated; when a source's metadata changes, the source-level cache is invalidated.

The v1 endpoint was left untouched for side-by-side comparison. Both versions ran in production simultaneously, with a feature flag controlling which one served traffic.

## Takeaways

1. **Map every query before optimizing.** `CaptureQueriesContext` is your best friend. You can't fix what you can't see.
2. **N+1s in serializers are the usual suspect.** Any `SerializerMethodField` that touches the database is a potential N+1. Pre-fetch and pass via context, or eliminate the serializer.
3. **Aggregation beats iteration.** `BoolOr`, `Count`, `Exists` — SQL can do in one query what Python does in N.
4. **Polymorphic models are expensive.** If you know the concrete type, skip the ORM's resolution. Use the FK directly.
5. **Cache at the right granularity.** Not everything changes at the same rate. Tier your cache by volatility, not by convenience.
6. **Always keep one field live.** For billing-critical or security-critical data, never trust the cache. One extra query is worth the correctness guarantee.
