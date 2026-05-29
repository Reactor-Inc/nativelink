# Phase 4: Optional Cloud Overflow

## Objective

Allow the office NativeLink deployment to coordinate with a cloud NativeLink
cluster only for explicitly eligible actions, while preserving office-first data
residency and performance.

## Success Criteria

- Office-only actions never upload inputs, action metadata, outputs, or AC
  entries to cloud.
- Cloud-eligible actions can overflow under explicit policy.
- Cloud-required actions route to cloud when no office worker type exists.
- Office and cloud cache namespaces are separated.
- Cloud result import policy is explicit and observable.
- Metrics show when and why cloud overflow happened.
- Cloud workers behave like explicitly admitted elastic helper capacity, with
  warmup, drain, idle timeout, and emergency disable controls.

## Tasks

- [ ] Define platform properties for data residency and pool eligibility.
- [ ] Define default policy for missing data-residency property.
- [ ] Build a manual cloud route first using separate instance names, such as
      `office-lan` and `office-cloud`.
- [ ] Keep cloud object stores out of office `fast_slow` stores unless explicit
      mirroring is approved.
- [ ] Prototype a local-first overflow scheduler wrapper.
- [ ] Add policy checks before any cloud CAS upload.
- [ ] Decide whether cloud worker lifecycle is owned by NativeLink, an external
      autoscaler, or a cloud-provider-specific controller.
- [ ] Add optional cloud warm-pool controls: pre-wake, max active workers, idle
      timeout, and scale-to-zero behavior.
- [ ] Keep cloud AC separate from office AC.
- [ ] Add optional output import from cloud CAS to office CAS.
- [ ] Add metrics for overflow attempts, accepted overflow, rejected overflow,
      cloud upload bytes, cloud output import bytes, and cloud latency.
- [ ] Add runbook for disabling cloud overflow instantly.
- [ ] Add tests proving office-only actions are rejected from cloud routing.

## Scheduler Shape

The likely implementation is a scheduler wrapper:

```text
CacheLookup
  -> ResidencyPolicyScheduler
      -> LocalFirstOverflowScheduler
          -> office scheduler
          -> cloud gRPC scheduler
```

The wrapper should decide before inputs are copied to cloud. It should never rely
on cloud-side rejection as the only protection.

NativeLink's existing `GrpcScheduler` can forward to one upstream scheduler, but
it is not enough for automatic local-first overflow because it does not decide
between LAN and cloud based on queue state, residency policy, worker match, or
cost. Use it for manual routing or as the cloud leg behind a new wrapper.

## Cloud Cache Policy

- Cloud CAS/AC is not the office source of truth.
- Office AC hits should only reference blobs in office CAS.
- Cloud result import should copy outputs into office CAS before making an
  office-visible AC entry.
- Office-only action results should not be inserted into cloud AC.
- Cloud AC should be disabled, read-only, or isolated until policy is explicit.

Cloud overflow should not be implemented by placing a cloud object store behind
the office `fast_slow` store. That turns ordinary office cache writes into cloud
data movement and bypasses residency admission.

## Cloud Worker Pool Policy

- Cloud pool capacity starts disabled.
- Manual routing is the first supported mode.
- Automatic overflow requires a local queue/capacity threshold and explicit
  `cloud_eligible` or `cloud_required` admission.
- Workers should pre-warm only when policy allows the corresponding data class to
  run in cloud.
- Idle workers should drain and terminate after a configured timeout.
- Operators need an immediate disable path that rejects new cloud leases and
  prevents new cloud CAS uploads.

## Agent Split

- Oversight: data residency and trust-boundary design.
- Implementation A: policy model and config.
- Implementation B: scheduler wrapper.
- Implementation C: cloud CAS upload/import path.
- Implementation D: cloud worker pool lifecycle hooks.
- Implementation E: metrics and dashboards.
- Mechanical: config examples and runbooks.

## Exit Decision

Cloud overflow should remain disabled by default until failure drills prove that
office-only data cannot leave the LAN through normal config mistakes, scheduler
fallback, retry behavior, or AC hit/import paths.
