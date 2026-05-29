# Incredibuild Comparison

## Summary

Incredibuild is a useful topology analogy, but not a cache-coherence blueprint
for the NativeLink + actiond plan.

The public Incredibuild model is:

```text
Initiator machine
  -> Coordinator allocates Helper agents
      -> Helpers run remote processes through process virtualization
      -> outputs synchronize back to the Initiator
```

The planned NativeLink + actiond model is:

```text
Bazel clients
  -> office NativeLink REAPI endpoint
      -> canonical office CAS/AC
      -> NativeLink scheduler
          -> actiond workers with disposable worker-local CAS
```

The practical conclusion is to borrow Incredibuild's operational ideas while
keeping NativeLink as the authoritative scheduler, CAS, and Action Cache.

## Incredibuild Concepts

Public docs describe these roles:

- Coordinator: tracks agents and assigns Helper capacity when an Initiator
  requests resources.
- Initiator: owns the distributed build session, allocates tasks to Helpers,
  monitors status, and synchronizes outputs back.
- Helper: contributes idle compute. Helpers can execute work without a normal
  local source checkout/toolchain by relying on process/file virtualization and
  a helper-side file cache.
- Build Cache: a separate task-avoidance layer that checks local cache first,
  then a shared/remote cache, and executes on a miss.
- Cloud: cloud VMs can act as Helper agents, often using a warm/sleep pool plus
  on-demand scale-out.

Relevant public docs:

- https://docs.incredibuild.com/win/latest/windows/components_and_architecture.html
- https://docs.incredibuild.com/win/latest/windows/process_virt_overview.html
- https://docs.incredibuild.com/win/10_13/windows/initiator.html
- https://docs.incredibuild.com/win/10_33_0/windows/build_cache_overview.htm
- https://docs.incredibuild.com/lin/latest/linux/build_avoidance.htm
- https://docs.incredibuild.com/cloud/cloud_understanding.html
- https://docs.incredibuild.com/cloud/cloud_settings.html

## Matches

- Office-local control plane: Incredibuild has a Coordinator; the plan has an
  office NativeLink instance.
- Idle worker pool: Incredibuild Helpers contribute idle CPU; actiond workers can
  contribute office-local compute.
- Disposable worker locality: Incredibuild Helper Cache avoids re-fetching files;
  actiond worker-local CAS should avoid re-copying blobs.
- Cache economics: both systems need cache-hit reporting, miss diagnostics, and
  "compute saved" style visibility.
- Hybrid bursting: Incredibuild Cloud is analogous to an explicitly gated cloud
  overflow lane.

## Divergences

- Incredibuild is Initiator-centric. NativeLink should be scheduler-centric and
  own action leases plus canonical CAS/AC state.
- Incredibuild hides many remote execution details through process
  virtualization. Bazel REAPI requires declared inputs, digests, command,
  environment, platform, and output metadata to be correct.
- Incredibuild Helper Cache is not equivalent to the authoritative Bazel CAS.
  actiond worker-local CAS must remain disposable and non-authoritative.
- Incredibuild cloud bursting is marketed as elastic capacity. The office plan
  requires explicit `office_only`, `cloud_eligible`, or `cloud_required` policy
  before any cloud upload or execution.

## Borrow

- Agent admission policy: worker should accept work only when CPU load, memory,
  disk, network health, and user-active state are within configured thresholds.
- Drain semantics: workers need an easy "stop accepting new leases, finish or
  cancel current work" path for laptop sleep, user activity, maintenance, and
  credential rotation.
- Helper-cache metrics: report worker-local CAS hits/misses, prefill bytes,
  reused bytes, evictions, and disk pressure.
- Build-cache UX: report AC hit rate, local worker-cache hit rate, shared CAS
  bytes served, miss reasons, and estimated compute saved.
- Cloud pool lifecycle: model cloud overflow as a warm pool with pre-wake,
  scale-out, idle timeout, and emergency disable controls.

## Do Not Borrow

- Do not use transparent process virtualization to paper over undeclared Bazel
  inputs.
- Do not let remote workers fetch arbitrary source files from developer
  machines.
- Do not make client-local or worker-local CAS/AC the source of truth.
- Do not use a distributed network filesystem as the coherence layer.
- Do not share writable AC state between office and cloud.
- Do not make cloud overflow automatic for actions that have not been admitted by
  residency policy.

## Plan Impact

The Incredibuild research reinforces the current architecture:

1. Keep NativeLink as the office-canonical scheduler, CAS, and AC.
2. Treat actiond worker-local CAS as a helper cache.
3. Add worker admission/drain as a first-class production-hardening track.
4. Add cache-miss and compute-saved dashboards as observability goals.
5. Model cloud overflow as explicit, policy-gated elastic capacity.
