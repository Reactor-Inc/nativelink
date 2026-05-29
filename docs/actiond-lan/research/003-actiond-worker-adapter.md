# actiond Worker Adapter

## Current actiond Shape

actiond is already a REAPI worker/cache. It exposes the public Bazel-facing
services needed for local remote execution:

- `Execution/Execute`
- CAS `FindMissingBlobs`, `BatchUpdateBlobs`, `BatchReadBlobs`, `GetTree`
- `ByteStream/Read` and `ByteStream/Write`
- ActionCache `GetActionResult` and `UpdateActionResult`
- `Capabilities/GetCapabilities`

It has two execution families:

- direct Linux execution with chroot, private mount/network namespaces, bind
  mounts, and cgroups
- VM execution where the host forwards REAPI over vsock into a Linux guest

In VM mode, the guest owns CAS and AC on ext4. The host should not mirror a
second CAS, and multiple VMs must not share one writable ext4 image.

## Adapter Role

The adapter is a protocol bridge:

```text
NativeLink WorkerApi
  <-> actiond-worker-adapter
      <-> actiond local REAPI endpoint
          <-> actiond VM/direct executor
```

The adapter should be colocated with the actiond instance. In the common case it
connects to `127.0.0.1:<actiond>` and opens one outbound TLS/mTLS connection to
the office NativeLink scheduler.

## Input Transfer

The adapter must ensure actiond has all inputs before calling Execute.

First-spike input transfer can be eager:

1. Read the `Action` from NativeLink CAS.
2. Read the `Command`.
3. Traverse the input root tree from NativeLink CAS.
4. Ask actiond CAS for missing digests.
5. Copy missing Action, Command, Directory, Tree, and file blobs from NativeLink
   CAS to actiond CAS.
6. Call actiond Execute.

This is simpler than a lazy proxy and gives clear correctness. Later work can
optimize with digest presence indexes, batching, ByteStream ranges, or worker
blob locality hints.

## Execution Request Policy

The first adapter should force or emulate centralized cache ownership:

- Let NativeLink handle cache lookup.
- Prefer `skip_cache_lookup=true` when calling actiond for centrally scheduled
  work.
- Treat actiond AC as local and non-authoritative.
- Only publish results to NativeLink AC after NativeLink CAS has all referenced
  outputs.

This avoids stale worker-local AC hits returning results that the central CAS
cannot serve.

Incredibuild's Helper Cache is a useful analogy for worker-local actiond CAS: it
is valuable for avoiding repeated transfers, but it is not the authoritative
cache. The adapter must not promote actiond AC hits to office-visible results
unless every referenced output blob is present in NativeLink CAS.

The adapter must also preserve platform information in the REAPI `Action` or
`Command` that actiond executes. NativeLink's `StartExecute.platform` is useful
for scheduling, but actiond reads platform requirements from the action payload
it receives. Required properties such as `libc`, `mutates_inputs`, and cgroup
limits must therefore be present in the actual REAPI action/command path or be
translated intentionally before execution.

## Output Transfer

After actiond returns an `ExecuteResponse`, the adapter must upload all output
digests referenced by the `ActionResult` back to NativeLink CAS:

- output files
- stdout/stderr digests
- output directory tree digests
- nested tree file digests

Only after this upload is complete should the adapter report success to
NativeLink. If upload fails, return an infrastructure error so the scheduler can
retry on another worker.

## actiondfs Constraints

Do not bypass actiond's VM/actiondfs model by mounting NativeLink's CAS directly
inside the guest.

The worker-local actiond CAS should stay guest-local so actiondfs can preserve
its performance properties:

- guest filesystem page cache
- backing-file reads
- `splice_read`
- `mmap`
- VM-lifetime actiondfs directory cache

The adapter copies data into that CAS through actiond's REAPI surface instead of
sharing filesystems across workers.

## Failure Classes

- Missing input in NativeLink CAS: return `FAILED_PRECONDITION` or scheduler
  input error. Bazel should re-upload if appropriate.
- actiond setup failure: infrastructure failure; retry if lease policy allows.
- action exit non-zero: normal action result; do not retry as infra.
- output upload failure: infrastructure failure; do not update AC.
- worker disconnect: let NativeLink lease expiry requeue.
- actiond VM restart: clear adapter-local presence hints; worker-local CAS may
  survive only if actiond preserves its ext4 image.
- kill request: actiond currently has no NativeLink-style operation kill API.
  The first adapter should reject or best-effort cancel and document that
  scheduler kill does not reliably terminate the spawned action until actiond
  grows a cancellation/control path.
- unsupported digest function: actiond currently advertises SHA-256 only. The
  adapter must reject or translate non-SHA-256 requests before execution.
- unsupported output type: actiond output collection is file/directory oriented.
  Symlink outputs and any richer NativeLink worker behavior need explicit
  support or a clear rejection path.

## Performance Notes

The simple eager-copy spike is correctness-first. For real LAN use, the adapter
must avoid re-copying blobs that are already in actiond's persistent worker CAS:

- batch `FindMissingBlobs` against actiond before upload
- retain adapter-local presence hints only as an optimization
- clear hints after actiond VM or root restart
- prefer persistent actiond worker roots over per-action roots
- measure input prefill time separately from actiond execution time
- report worker-local CAS hit/miss counts, reused bytes, uploaded bytes,
  evictions, and disk pressure as helper-cache metrics

The adapter should not copy Incredibuild's transparent file virtualization model.
Bazel REAPI actions must execute from declared inputs and digests. A worker that
needs arbitrary files from a developer machine is a correctness bug, not a cache
miss.

## Security Boundaries

- The adapter must not expose actiond's local REAPI endpoint on the LAN by
  default.
- The adapter's scheduler connection should use mTLS for office deployments.
- Worker identity should be stable enough for metrics and drain, but not derived
  from user names or high-cardinality invocation IDs.
- Logs must avoid command argv, env, source paths with secrets, and stderr unless
  explicitly configured for local debugging.
