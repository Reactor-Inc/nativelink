# Phase 1: Standalone actiond Worker Spike

## Objective

Prove that NativeLink can schedule work to an actiond worker without changing
NativeLink's existing scheduler or worker config model.

The spike should use a standalone adapter binary that speaks NativeLink WorkerApi
on one side and actiond REAPI on the other.

## Success Criteria

- NativeLink runs as the Bazel-facing scheduler/CAS/AC on LAN.
- A standalone actiond adapter connects as a worker.
- A trivial Bazel action executes through NativeLink on actiond.
- The action result is returned only after output blobs are uploaded to
  NativeLink CAS.
- Worker-local actiond AC is not authoritative.
- The design works for at least one actiond mode: direct Linux or VM.

## Tasks

- [ ] Reproduce NativeLink's existing local multi-worker example.
- [ ] Run a local actiond worker with a dedicated root and local-only listener.
- [ ] Add a standalone adapter binary or scratch crate inside this repo.
- [ ] Connect adapter to NativeLink WorkerApi.
- [ ] Advertise actiond worker platform properties.
- [ ] Avoid repeated platform-property names until NativeLink's worker
      capability model is extended.
- [ ] Receive `StartExecute` and parse `ExecuteRequest`.
- [ ] Reject unsupported digest functions; first spike is SHA-256 only.
- [ ] Preserve or translate action platform properties intentionally before
      calling actiond.
- [ ] Copy Action, Command, Directory, Tree, and file blobs from NativeLink CAS
      into actiond CAS.
- [ ] Call actiond `Execution/Execute`.
- [ ] Send `ExecuteComplete` after actiond execution if the adapter follows
      NativeLink's existing worker lifecycle.
- [ ] Traverse output digests in `ActionResult`.
- [ ] Reject unsupported output types explicitly.
- [ ] Upload outputs from actiond CAS into NativeLink CAS.
- [ ] Update NativeLink AC when the action is cacheable and central CAS
      completeness is confirmed.
- [ ] Report successful `ExecuteResult` to NativeLink.
- [ ] Return infrastructure errors for setup/upload failures.
- [ ] Handle NativeLink kill requests with a documented unsupported or
      best-effort behavior.
- [ ] Run one small Bazel build against NativeLink using the actiond worker.
- [ ] Document exact commands and logs.

## Suggested Agent Split

- Oversight: verify the data flow and AC/CAS correctness before implementation.
- Implementation A: WorkerApi client connection and lifecycle.
- Implementation B: NativeLink-to-actiond CAS copy.
- Implementation C: actiond Execute call and result conversion.
- Implementation D: output traversal and upload.
- Mechanical: example config, README commands, focused test fixtures.

## Initial Shortcuts Allowed

- One action at a time.
- Eager input tree copy.
- No cloud overflow.
- No action locality scheduler hints.
- Static worker config.
- No persistent adapter state beyond process lifetime.
- No reliable action cancellation if actiond has already spawned the process.

## Shortcuts Not Allowed

- Returning success before central CAS contains outputs.
- Sharing one actiond ext4 CAS image across workers.
- Exposing actiond's local REAPI endpoint to the LAN.
- Falling back to cloud.
- Treating actiond local AC as canonical.

## Test Plan

- Unit test output digest traversal.
- Unit test missing input classification.
- Integration test with a fake NativeLink CAS and fake actiond endpoint if
  writing full e2e is too expensive initially.
- Manual Bazel smoke through a real NativeLink server and one actiond worker.

## Exit Decision

If the standalone adapter is small and reliable, keep iterating toward a
NativeLink-native worker type. If it fights NativeLink's protocol or storage
model, stop and reassess whether a separate actiond pool coordinator is simpler.
