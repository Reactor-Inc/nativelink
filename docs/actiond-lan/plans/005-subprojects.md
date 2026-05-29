# Subproject Tracker

This file tracks cross-phase ownership areas. Phase plans remain the source of
truth for detailed task order; this tracker makes it easier to assign parallel
agents without overlapping write sets.

## SP1: NativeLink WorkerApi Adapter

- [ ] Standalone adapter binary connects to WorkerApi.
- [ ] Worker registration advertises actiond mode and capacity.
- [ ] Keepalive, disconnect, going-away, and retry behavior documented.
- [ ] Kill request behavior explicitly unsupported or best-effort.
- [ ] Later: promote to NativeLink-native `WorkerConfig::Actiond`.

Owner type: mid-weight implementation, strongest review.

## SP2: CAS Prefill And Output Upload

- [ ] Traverse Action, Command, input Directory tree, and input file digests.
- [ ] Batch actiond `FindMissingBlobs`.
- [ ] Upload missing blobs to actiond CAS.
- [ ] Traverse ActionResult outputs, stdout, stderr, and output directory trees.
- [ ] Upload output blobs to NativeLink CAS before final result.
- [ ] Add completeness checks before AC update.

Owner type: mid-weight implementation, strongest correctness review.

## SP3: Platform And Capability Model

- [ ] Define single-valued first-spike platform keys.
- [ ] Decide how to represent multiple actiond runtimes such as libc variants.
- [ ] Add tests for worker capability matching.
- [ ] Define office/cloud routing keys.
- [ ] Define default policy for missing data-residency property.

Owner type: strongest design, mid-weight implementation.

## SP4: Scheduler And Overflow

- [ ] Manual `office-lan` and `office-cloud` instance routing.
- [ ] Cloud `GrpcScheduler` leg for explicit routing.
- [ ] Local-first overflow scheduler wrapper design.
- [ ] Queue-pressure and worker-match trigger policy.
- [ ] Data-residency enforcement before cloud CAS upload.

Owner type: strongest design, mid-weight implementation.

## SP5: Security And Deployment

- [ ] Separate public REAPI and private worker/control listeners.
- [ ] mTLS for worker/control plane.
- [ ] Rack deployment storage layout.
- [ ] Worker certificate enrollment and rotation.
- [ ] Firewall guidance and runbook.

Owner type: strongest security review, mechanical docs support.

## SP6: Metrics And Dashboards

- [ ] Store metrics wrappers for office CAS/AC.
- [ ] Worker metrics for CAS prefill, actiond execute, output upload.
- [ ] Worker-local CAS/helper-cache metrics for hits, misses, reused bytes,
      uploaded bytes, evictions, and disk pressure.
- [ ] Scheduler metrics for queue depth, worker liveness, retries, overflow.
- [ ] Grafana dashboards for pool, worker, cache, actiondfs, and cloud overflow.
- [ ] Cache UX dashboards for AC hit rate, miss classes, local worker reuse, and
      estimated compute saved.
- [ ] Label-cardinality review.

Owner type: mid-weight implementation, cheap dashboard/docs edits, strongest
observability review.

## SP7: Test Harnesses

- [ ] Unit tests for digest traversal and output completeness.
- [ ] WorkerApi adapter lifecycle tests.
- [ ] Fake actiond endpoint test.
- [ ] Real NativeLink + actiond smoke.
- [ ] Multi-worker LAN smoke.
- [ ] Cloud overflow policy rejection tests.

Owner type: mid-weight verification and implementation, cheap fixture edits.

## SP8: Upstream Contribution Hygiene

- [ ] Keep actiond-specific changes isolated behind config.
- [ ] Avoid broad dependency or lockfile churn.
- [ ] Add docs/examples near NativeLink conventions.
- [ ] Prepare small upstreamable PR slices when possible.
- [ ] Track any FSL/upstream contribution assumptions separately from code.

Owner type: strongest coordinator, cheap docs support.

## SP9: Worker Admission And Drain

- [ ] Define worker admission inputs: CPU load, memory, disk, network health,
      user-active state, and configured foreground-process exclusions.
- [ ] Define admission states: idle, busy, drained, overloaded, disk-pressure,
      user-active, unreachable, and shutting-down.
- [ ] Expose admission state to scheduler metrics without high-cardinality labels.
- [ ] Stop accepting new leases when admission policy rejects the worker.
- [ ] Define drain behavior for laptop sleep, office maintenance, credential
      rotation, and actiond VM restart.
- [ ] Add failure drills for user activity, high CPU, low disk, and worker sleep.

Owner type: mid-weight implementation, strongest operations review.

## SP10: Incredibuild Comparison Follow-Through

- [ ] Keep the Incredibuild comparison note current as implementation decisions
      land.
- [ ] Verify that no design adopts transparent process virtualization or
      arbitrary source fetches from developer machines.
- [ ] Review cloud overflow against the helper-pool analogy without weakening
      data-residency gates.
- [ ] Review worker-local CAS wording so it stays a helper cache, never a
      canonical cache.

Owner type: strongest design review, cheap docs support.
