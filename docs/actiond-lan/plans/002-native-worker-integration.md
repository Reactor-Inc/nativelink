# Phase 2: NativeLink-Native actiond Worker

## Objective

Turn the Phase 1 adapter into a first-class NativeLink worker type that can be
configured, tested, observed, and operated like other NativeLink components.

## Success Criteria

- NativeLink config supports an actiond worker entry.
- The worker connects to a configured actiond endpoint.
- The worker reports stable platform properties and max inflight capacity.
- CAS transfer and output upload are tested.
- Worker drain and graceful shutdown work.
- Metrics distinguish NativeLink worker time, CAS transfer time, actiond execute
  time, and output upload time.
- Metrics report worker-local CAS/helper-cache hits, misses, reused bytes,
  evictions, and disk pressure.

## Candidate Code Areas

- `nativelink-config/src/cas_server.rs`
  - add `ActiondWorkerConfig`
  - extend `WorkerConfig`
- `nativelink-worker/src/`
  - add actiond worker module
  - add actiond REAPI client wrapper
  - add CAS sync utilities
- `nativelink-proto/`
  - reuse existing REAPI generated types where possible
- `src/bin/nativelink.rs`
  - instantiate actiond worker config
- `nativelink-worker/tests/`
  - protocol, CAS sync, output upload, shutdown

## Tasks

- [ ] Promote adapter configuration into `nativelink-config`.
- [ ] Add an actiond worker implementation behind a feature or new config
      variant.
- [ ] Reuse NativeLink's WorkerApi lifecycle behavior where possible.
- [ ] Implement bounded concurrent CAS copy.
- [ ] Implement output completeness checks before reporting completion.
- [ ] Add worker-level metrics.
- [ ] Add health checks for actiond endpoint reachability.
- [ ] Add worker admission state reporting for idle, busy, drained, overloaded,
      disk-pressure, and user-active states.
- [ ] Add drain and going-away support.
- [ ] Add tests for actiond endpoint failure, CAS upload failure, and scheduler
      disconnect.
- [ ] Add docs and example LAN config.

## Design Constraints

- actiond remains an execution backend, not a shared CAS backend.
- The NativeLink CAS/AC remains canonical.
- The actiond endpoint is local to the adapter machine by default.
- VM/actiondfs behavior is preserved by copying blobs through actiond REAPI.
- The worker type should work with both actiond direct Linux and VM modes.
- Incredibuild-style process virtualization is explicitly out of scope. Workers
  execute declared REAPI actions; they do not fetch arbitrary source files from
  developer machines.

## Agent Split

- Implementation A: config structs and config tests.
- Implementation B: actiond REAPI client wrapper.
- Implementation C: CAS sync and output traversal.
- Implementation D: worker lifecycle integration.
- Implementation E: metrics and health checks.
- Oversight: review data-flow correctness and failure semantics after each slice.

## Exit Decision

Proceed to LAN hardening when a real Bazel smoke can run repeatedly through
NativeLink to at least two actiond workers without output CAS gaps or scheduler
state leaks.
