# NativeLink + actiond LAN Execution

This directory tracks the design for using NativeLink as the central scheduler,
CAS, and Action Cache for an office-local pool of actiond workers.

The working assumption is that the primary NativeLink deployment runs on office
LAN hardware, such as a server rack machine with local NVMe or office-local
storage. The default mode must keep source inputs, action metadata, outputs, and
cache traffic inside the physical office network. Cloud capacity is an optional
overflow path for explicitly eligible actions, not the default execution path.

## Current Direction

- NativeLink is the central REAPI endpoint for Bazel.
- NativeLink owns the canonical CAS and Action Cache for the office pool.
- actiond workers connect from office machines and execute actions through their
  existing VM/direct execution modes.
- Worker-local actiond CAS state is a disposable execution cache, not the source
  of truth.
- Cloud overflow is policy-driven and separated by instance, platform property,
  scheduler policy, and cache namespace.

## Files

- `research/001-summary.md`: current high-level findings and decisions.
- `research/002-nativelink-integration-seams.md`: NativeLink code/config seams
  relevant to actiond workers.
- `research/003-actiond-worker-adapter.md`: actiond-specific adapter constraints
  and data flow.
- `research/004-lan-cloud-topology.md`: LAN-first topology, data residency, and
  cloud overflow model.
- `research/005-incredibuild-comparison.md`: comparison with Incredibuild's
  Coordinator/Initiator/Helper, helper-cache, build-cache, and cloud-bursting
  model.
- `plans/000-operating-model.md`: process, review gates, and subagent model.
- `plans/001-spike-actiond-worker.md`: first executable integration spike.
- `plans/002-native-worker-integration.md`: NativeLink-native actiond worker
  integration.
- `plans/003-lan-production-hardening.md`: office deployment hardening.
- `plans/004-cloud-overflow.md`: optional cloud overflow scheduler path.
- `plans/005-subprojects.md`: cross-phase subproject tracker.

## Non-Goals For The First Spike

- Replacing actiond's executor, VM host, or actiondfs implementation.
- Making worker-local CAS authoritative.
- Multi-writer shared ext4 images or shared actiond `/cas` mounts.
- Automatic cloud execution for arbitrary office builds.
- Slug embedding. That remains a longer-term option after the worker path is
  proven.
