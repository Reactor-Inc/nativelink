# Research Summary

## Recommendation

Use NativeLink as the central office scheduler and canonical CAS/Action Cache,
then add an actiond worker adapter that lets actiond instances join the
NativeLink worker pool.

This is a better first project than a full actiond Rust rewrite. NativeLink
already has the scheduler, REAPI service assembly, store graph, worker API,
platform-property matching, and OpenTelemetry scaffolding. actiond's
differentiator is the VM/actiondfs execution path, so the first integration
should preserve actiond as an execution backend rather than replacing it.

## NativeLink Concepts To Reuse

- Config-driven services and store graphs, assembled in `src/bin/nativelink.rs`.
- A private worker API separate from public REAPI services.
- Scheduler-owned worker registration, heartbeat, and platform properties.
- Store wrappers for verification, cache metrics, fast/slow storage, and
  completeness checking.
- OpenTelemetry-first metrics with Prometheus/Grafana examples.
- Cache lookup as a scheduler wrapper around an inner scheduler.

## Incredibuild Research Takeaways

Incredibuild is a useful comparison for office worker-pool ergonomics, but it is
not the right coherence model for Bazel REAPI.

The concepts worth borrowing are worker admission/drain policy, helper-cache
metrics, cache-miss diagnostics, compute-saved dashboards, and cloud warm-pool
lifecycle. The concepts to avoid are transparent remote file/process
virtualization, arbitrary worker fetches from developer machines, and any
worker-local cache becoming authoritative.

The NativeLink + actiond design should therefore stay scheduler-centric:
NativeLink owns action leases, canonical CAS, and canonical AC. actiond owns
execution and a disposable worker-local CAS.

## actiond Concepts To Preserve

- VM mode has a guest-owned ext4 CAS and Action Cache exposed over vsock.
- actiondfs resolves REAPI input trees lazily from guest-local CAS and delegates
  file reads to backing files.
- Direct Linux mode uses materialized/bind-mounted inputs and is not equivalent
  to the VM/actiondfs fast path.
- Standalone actiond packaging and VM lifecycle should remain independent of
  NativeLink until the integration is proven.

## Core Design Decision

NativeLink's office CAS/AC is authoritative. actiond worker-local CAS state is a
read-through/write-through execution cache.

That gives a simple coherence rule:

1. Bazel uploads to NativeLink.
2. NativeLink scheduler assigns a lease to an actiond worker.
3. The actiond adapter copies missing inputs from NativeLink CAS into actiond.
4. actiond executes locally.
5. The adapter uploads outputs from actiond CAS back to NativeLink CAS.
6. The adapter reports the result only after output CAS upload succeeds.
7. NativeLink AC is updated only after central CAS completeness is true.

## Biggest Open Questions

- Whether to implement the first adapter as an external sidecar binary using
  NativeLink's worker API, or as a new NativeLink `WorkerConfig` variant.
- Whether the NativeLink scheduler should support local-first cloud fallback as
  a new scheduler wrapper or through composition of existing scheduler types.
- How much of the actiond input tree transfer should be eager in the first spike
  versus incremental/lazy.
- Whether worker-local actiond AC should be disabled for centrally scheduled
  actions to avoid stale local hits.
- How to represent office-only, cloud-eligible, and cloud-required actions in
  platform properties without leaking high-cardinality data into metrics.
- How to implement Incredibuild-style worker admission thresholds without
  allowing laptops or foreground developer machines to become unreliable workers.
- Which cache-miss reasons and compute-saved metrics are actionable without
  exposing target labels, argv, env, paths, or user identifiers.

## Useful Source Anchors

- NativeLink worker API: `nativelink-proto/com/github/trace_machina/nativelink/remote_execution/worker_api.proto`
- NativeLink worker implementation: `nativelink-worker/src/local_worker.rs`
- NativeLink execution server: `nativelink-service/src/execution_server.rs`
- NativeLink scheduler: `nativelink-scheduler/src/simple_scheduler.rs`
- NativeLink store composition: `nativelink-store/src/*_store.rs`
- NativeLink metrics examples: `deployment-examples/metrics/README.md`
- actiond architecture: `../actiond/ARCHITECTURE.md`
- actiond shared library boundary: `../actiond/src/BUILD.bazel`
- Incredibuild comparison: `research/005-incredibuild-comparison.md`
