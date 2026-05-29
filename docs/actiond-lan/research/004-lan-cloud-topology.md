# LAN And Cloud Topology

## Office-First Deployment

The primary deployment target is an office-local rack or lab server:

```text
Bazel clients
  -> office NativeLink endpoint
      -> office CAS/AC on local storage
      -> office scheduler
          -> actiond adapters on developer or rack machines
              -> local actiond VM/direct workers
```

All default traffic stays on the physical LAN. The office NativeLink instance is
the public REAPI endpoint for users in that office.

Use separate listeners for public REAPI and private control-plane traffic:

- public REAPI listener: Bazel clients, limited services, office auth policy
- private worker/control listener: WorkerApi, admin, health, internal auth

WorkerApi has a different permission profile than public REAPI and should not be
published as an unauthenticated LAN-wide endpoint.

## Data Residency Policy

Use explicit policy, not implicit fallback:

- `data_residency=office_only`: never forward inputs, action metadata, outputs,
  or cache entries to cloud.
- `data_residency=cloud_eligible`: may overflow to cloud when scheduler policy
  allows.
- `data_residency=cloud_required`: route only to cloud workers, useful for
  worker types not present in office.

This can begin as platform properties. Longer term, the policy should be backed
by config-level allowlists and instance-level defaults so users cannot
accidentally leak office-only builds by omitting a property.

## Cache Namespaces

Use separate cache namespaces for trust and locality:

- office canonical CAS/AC
- cloud CAS/AC
- optional imported-results namespace for reviewed cloud outputs

Do not share a writable AC namespace between office and cloud by default. A cloud
AC hit is only useful to office if the referenced output blobs are present in the
office CAS or are allowed to be imported under policy.

Do not put a cloud object store behind an office `fast_slow` store unless cloud
mirroring is explicitly desired. A fast/slow store can make writes visible to the
slow backend as part of normal cache operation; that is not the same as
policy-gated overflow.

## Cloud Overflow Options

### Option A: Manual Routing

Users select an instance or platform property that routes to cloud.

This is easiest and safest. It should be the first cloud experiment.

Recommended first namespaces:

- `office-lan`: office CAS/AC, office workers only
- `office-cloud`: cloud CAS/AC, cloud workers only, opt-in use
- optional `office-imported`: office-visible imported cloud results after
  policy-approved output copy

### Option B: Scheduler Fallback

A new scheduler wrapper tries local workers first, then forwards cloud-eligible
actions to an upstream cloud scheduler when local capacity is exhausted.

Inputs may be uploaded to cloud only after the scheduler has verified the action
is cloud-eligible. Office-only actions remain queued or fail with a capacity
message.

NativeLink's current `GrpcScheduler` is useful for forwarding to one upstream
scheduler, but it is not a local-first overflow router by itself. Automatic
overflow likely needs a new scheduler wrapper.

Cloud overflow should be modeled as elastic helper capacity, not cache
federation. Borrow the Incredibuild-style pool lifecycle where useful: pre-wake
cloud workers, keep a small warm/sleep pool when justified, scale out only for
eligible work, drain idle workers, and retain an emergency disable switch.

### Option C: Capability Routing

Certain worker types, such as GPUs or large memory machines, live only in cloud.
Platform properties route those actions directly to cloud while normal work
stays local.

## Why Not Cloud-First

Cloud-first remote execution adds:

- WAN latency on CAS reads/writes
- egress cost for output downloads and cache churn
- off-site source and generated artifact handling
- harder debugging when workers are unavailable
- more credential and access-policy surface area

For office development, a local rack endpoint is the right default. Cloud is a
burst or special-capability extension.

## Metrics Needed For Overflow Decisions

- local queue depth by platform and priority
- local runnable worker count by platform
- worker utilization and idle tokens
- worker admission state and rejection reason
- worker-local CAS hit rate, reused bytes, and eviction pressure
- p50/p95 input upload time to cloud
- p50/p95 output import time from cloud
- action retry and eviction rates
- cloud spend and egress estimates
- overflow rejection reason
- bytes exported to cloud
- bytes imported from cloud

No metric label should include action digest, target label, source path, user,
branch, invocation ID, argv, env, or stderr.

## Open Questions

- Should cloud overflow use an upstream NativeLink scheduler via `GrpcSpec`, or
  a new explicit overflow scheduler wrapper?
- Should office and cloud use different REAPI instance names or one instance
  name with policy properties?
- Is cloud result import allowed automatically for cloud-eligible actions, or
  should only direct cloud clients consume cloud AC entries?
- What default policy should apply when an action has no data-residency property?
- What is the policy authority: instance name, Bazel platform property, user or
  repo allowlist, or an external admission service?
- How should cloud scheduler auth be represented if the forwarding path needs
  static or forwarded request headers?
- Is overflow single-run after a threshold, or speculative LAN/cloud racing with
  cancellation?
- Should cloud pool lifecycle be managed by NativeLink, an external autoscaler,
  or a cloud-provider-specific controller?
