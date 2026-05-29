# Phase 3: LAN Production Hardening

## Objective

Make the office deployment usable by a real team on a physical LAN without
cloud dependency.

## Success Criteria

- Rack NativeLink instance is the single office REAPI endpoint.
- Workers connect outbound with authenticated identities.
- Office CAS/AC has predictable durability and GC behavior.
- Metrics and dashboards show pool health.
- Worker drain and maintenance are documented.
- A developer machine can join and leave the pool safely.
- Workers decline new work when admission thresholds say the machine is not idle
  or healthy enough.
- Cache dashboards show hit rate, miss reasons, worker-local reuse, and estimated
  compute saved without high-cardinality labels.

## Tasks

- [ ] Define recommended rack hardware profile.
- [ ] Define CAS/AC storage layout and backup policy.
- [ ] Configure mTLS for scheduler, workers, and admin endpoints.
- [ ] Add worker identity and certificate enrollment process.
- [ ] Add firewall guidance: NativeLink public endpoint, worker outbound only,
      actiond local endpoint not exposed.
- [ ] Add Grafana dashboards for scheduler, workers, CAS/AC, and actiondfs.
- [ ] Add cache diagnostics dashboards for AC hit rate, CAS bytes served,
      worker-local CAS/helper-cache reuse, miss classes, and estimated compute
      saved.
- [ ] Add alerts for queue depth, worker loss, CAS errors, AC completeness
      failures, and output upload failures.
- [ ] Add worker admission policy for CPU load, memory, disk, network health,
      user-active state, and configured foreground-process exclusions.
- [ ] Add alerts for workers stuck overloaded, drained too long, or repeatedly
      rejected by admission policy.
- [ ] Add worker drain/runbook.
- [ ] Add GC policy and disk-full behavior.
- [ ] Run repeated small and medium Bazel workloads across multiple workers.
- [ ] Run an actiond VM smoke where applicable.

## Metrics

Keep labels low-cardinality:

- pool
- worker_id
- worker_mode
- scheduler
- instance_name
- platform class
- cache operation
- result class
- failure class
- admission state
- cache miss class

Do not label metrics with:

- action digest
- target label
- source path
- argv
- env
- stderr
- user
- branch
- invocation ID
- client IP

## Failure Drills

- Kill an actiond worker during execution.
- Restart actiond VM while adapter is connected.
- Fill worker-local disk.
- Fill central CAS disk.
- Drop LAN connectivity for one worker.
- Corrupt or delete a central CAS output referenced by AC.
- Rotate worker credentials.
- Simulate user activity or high foreground CPU on a developer workstation and
  verify the worker stops accepting new leases.
- Fill worker-local CAS enough to trigger admission rejection, eviction, or drain
  behavior.

## Agent Split

- Mechanical: dashboards, example configs, runbook formatting.
- Implementation: mTLS config, health endpoints, admission policy, metrics.
- Oversight: security review and data-residency review.
- Verification: failure drills and smoke script execution.
