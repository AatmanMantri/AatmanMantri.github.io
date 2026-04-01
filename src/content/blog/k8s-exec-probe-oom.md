---
title: 'How exec Probes Were Silently Killing Our Kubernetes Pods'
description: 'A debugging story: pods kept going OOM, and the culprit was the liveness probe itself.'
pubDate: 'Mar 28 2026'
---

We had a problem that kept showing up in production: pods were getting OOM-killed. Not under heavy load. Not during deployments. Just... randomly. Memory usage would climb steadily over hours until the kernel killed the process.

The services themselves were stateless Go binaries. They didn't leak goroutines. They didn't cache unboundedly. And yet, `kubectl top pods` told a different story every time we checked.

This is the story of how I traced the issue to something I never would have suspected: our liveness probes.

## The Symptom

We run a multi-service backend on Kubernetes — several Go microservices behind an API gateway, deployed across multiple environments. Each service had liveness and readiness probes configured as `exec` probes:

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "curl -sf http://localhost:8080/healthz || exit 1"
  initialDelaySeconds: 15
  periodSeconds: 10
```

This is a pattern you'll find in dozens of Helm chart examples and blog posts. It works. Until it doesn't.

Pods in one of our environments started getting OOM-killed every 12–18 hours. The Go service itself was using maybe 50–80MB of RSS memory. The pod memory limit was 256MB. Should have been plenty.

## The Investigation

I started with the obvious: profiling the Go service with `pprof`. Heap profiles looked clean. No goroutine leaks. No growing caches. The Go runtime was well-behaved.

But `kubectl top pods` showed memory climbing past 200MB. Where was it going?

I exec'd into a pod and checked `/proc/meminfo` and the cgroup memory stats. That's when I noticed something: the `slab` memory was enormous. Slab memory is kernel memory used for caching filesystem metadata, process descriptors, and other kernel objects. It's not the application's fault — it's the kernel allocating memory on behalf of processes running inside the container's cgroup.

## The Root Cause

Every 10 seconds, the `exec` liveness probe spawned:
1. A new process (`/bin/sh`)
2. Which spawned another process (`curl`)
3. Both ran to completion (or timeout)
4. Their kernel-side metadata (slab entries) were allocated inside the pod's cgroup

The problem: slab memory cleanup is lazy. The kernel doesn't immediately reclaim slab entries from short-lived processes. In a container cgroup, this memory accumulates. Each probe execution left behind a small residue of kernel slab memory. Over thousands of probe invocations (10s interval × 6/min × 60/min × 18h = ~6,480 invocations), this added up to hundreds of megabytes.

The OOM killer sees cgroup memory usage as application memory + kernel slab memory. Once the total exceeded the pod's memory limit, the container got killed.

## The Fix

Replace `exec` probes with `tcpSocket` probes:

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

`tcpSocket` probes are handled by the kubelet directly — it opens a TCP connection to the specified port. No child processes. No shell. No curl. No slab accumulation.

For services that need to check a specific HTTP endpoint, `httpGet` probes are the right choice:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

Like `tcpSocket`, the kubelet handles `httpGet` probes natively without spawning any processes.

After deploying the change across all environments, the OOM kills stopped completely. Memory usage became flat and predictable.

## The Lesson

`exec` probes are the most flexible probe type in Kubernetes. They're also the most dangerous for long-running pods. Every invocation spawns at least one process (usually two: a shell + the actual command), and the kernel slab memory from those short-lived processes accumulates inside the container's cgroup.

If your pods are mysteriously going OOM without an obvious application memory leak, check your probe configuration. If you're using `exec` probes with `curl` or `wget`, switch to `httpGet` or `tcpSocket`. The kubelet can do the same health check without the memory overhead.

This isn't a well-documented issue. The Kubernetes docs mention that `exec` probes "execute a command inside the container," but they don't warn about the slab memory implications. Now you know.
