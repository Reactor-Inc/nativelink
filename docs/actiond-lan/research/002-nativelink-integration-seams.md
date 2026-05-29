# NativeLink Integration Seams

## Service Assembly

`src/bin/nativelink.rs` is the main composition point. It:

- creates a `StoreManager`
- constructs configured stores with `store_factory`
- constructs schedulers with `scheduler_factory`
- wires public REAPI services for CAS, AC, ByteStream, Execution, Operations,
  Capabilities, health, admin, and WorkerApi
- starts configured local workers after listeners are ready

This is the natural place for a future NativeLink-native `ActiondWorkerConfig`.

## Worker API

`worker_api.proto` defines a bidirectional scheduler-worker stream:

- the worker starts with `ConnectWorkerRequest`
- the scheduler returns a stable worker ID
- the scheduler sends `StartExecute`, `Disconnect`, keepalive, and kill requests
- the worker sends keepalive, going-away, execute-complete, and execute-result

This is a good shape for actiond because office workers can initiate outbound
connections to the rack scheduler. It also avoids public inbound worker ports on
developer machines.

For a first spike, implement an external `actiond-worker-adapter` binary that
uses this same WorkerApi client. That avoids changing NativeLink config and
keeps the integration reversible.

NativeLink currently converts worker platform properties into a map by name when
workers connect. Do not rely on repeated properties with the same name, such as
multiple `libc` values, until that model is extended. For the first spike, use
single-valued capability keys or register one adapter process per capability
profile.

## Existing Local Worker Assumptions

`nativelink-worker/src/local_worker.rs` is useful as a protocol reference, but it
is not the actiond worker implementation we want to reuse directly.

Important assumptions:

- `new_local_worker()` expects the CAS store to be a `FastSlowStore`.
- The local worker's fast store must be filesystem-backed because execution
  setup uses hardlinks.
- The worker uploads outputs and caches action results before reporting final
  execution response.
- The worker can send `ExecuteComplete` after execution, then finish output/AC
  upload, then send final `ExecuteResult`.
- The worker supports a precondition script, max inflight tasks, platform
  properties, and graceful going-away behavior.

For actiond, the adapter should keep the NativeLink worker protocol but replace
local hardlink materialization with actiond's REAPI-facing execution backend.

## Store Graph

NativeLink stores are composable. The relevant patterns are:

- `fast_slow_store`: local fast cache plus shared slow store.
- `completeness_checking_store`: checks AC hits against CAS outputs.
- `cache_metrics_store`: opt-in low-cardinality cache metrics.
- `verify_store`: validates CAS digest and size before accepting writes.
- `grpc_store`: can proxy to another CAS.

The office design should use NativeLink stores for the canonical office CAS/AC
and treat actiond's CAS as a worker-side cache outside the NativeLink store
graph until there is a strong reason to make it a first-class store.

## Scheduler Hooks

The current simple scheduler already has several useful building blocks:

- platform property manager
- worker registry and keepalive timeout
- max job retries
- worker allocation strategy
- worker capability index
- cache lookup scheduler wrapper
- gRPC scheduler wrapper for forwarding
- property modifier scheduler wrapper

The likely missing piece is a local-first overflow scheduler:

1. Try the office scheduler for eligible actions.
2. If no matching local worker exists, or office queue pressure exceeds policy,
   optionally forward to a cloud scheduler.
3. Never forward actions whose policy says `data_residency=office_only`.

## First Integration Spike

Build a standalone sidecar adapter first:

- connect to NativeLink WorkerApi as an actiond worker
- advertise `executor=actiond`, `pool=office`, arch, OS, VM/direct mode, and
  resource properties
- receive `StartExecute`
- copy required CAS inputs from NativeLink CAS to actiond CAS
- call actiond `Execution/Execute`
- send `ExecuteComplete` after actiond execution if following NativeLink's
  existing worker lifecycle
- copy outputs from actiond CAS to NativeLink CAS
- update NativeLink AC if the worker path owns AC updates for this action
- send final `ExecuteResult` according to NativeLink's worker protocol

## Tests To Extend First

- `nativelink-service/tests/worker_api_server_test.rs`: worker stream behavior.
- `nativelink-worker/tests/local_worker_test.rs`: registration and result upload
  lifecycle references.
- `nativelink-worker/tests/multi_worker_cas_test.rs`: CAS sharing assumptions.
- `nativelink-scheduler/tests/worker_capability_index_test.rs`: platform
  matching behavior.

Only after this works should the adapter become a NativeLink-native worker type.
