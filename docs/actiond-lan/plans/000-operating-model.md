# Operating Model

## Goal

Use a high-parallelism agent workflow without losing architectural control.
Cheap/fast agents handle mechanical edits and local investigation. Mid-weight
agents own coherent implementation slices. Strongest agents own design review,
cross-cutting correctness, and merge decisions.

## Agent Tiers

### Oversight Agents

Use the strongest model for:

- architecture decisions
- API and protocol review
- scheduler/CAS/AC correctness review
- security and data-residency review
- final integration review
- release readiness decisions

Oversight agents should usually not make broad code edits. They should inspect,
challenge assumptions, and produce concrete findings.

### Implementation Agents

Use mid-weight models for coherent code slices with clear ownership:

- actiond adapter WorkerApi client
- CAS copy pipeline
- ActionResult output traversal
- NativeLink `ActiondWorkerConfig`
- scheduler overflow wrapper
- metrics instrumentation
- integration tests

Each implementation agent gets a disjoint write set and a narrow done condition.

### Mechanical Agents

Use cheap/fast models for:

- adding repetitive config fields
- updating generated docs or examples
- writing test fixtures
- renaming metrics
- filling TODO checklists
- small Markdown edits
- running focused searches

Mechanical agents should not own design decisions.

## Concurrency Rules

- One owner per file or module group.
- No agent reverts unrelated changes.
- Every worker reports changed files and tests run.
- Use small patches with reviewable boundaries.
- Prefer additive adapters and feature flags before changing existing behavior.
- Keep public REAPI behavior stable unless a plan explicitly changes it.

## Handoff Template

Each subagent handoff should use this packet:

```md
Task:
Scope:
Files inspected:
Files changed:
Behavior changed:
Tests run:
Tests not run:
Risks / assumptions:
Open questions:
Suggested next agent:
```

Implementation agents should also include:

```md
Contract checked:
Failure modes considered:
Compatibility impact:
```

Verification agents should include exact commands and pass/fail status.

## Review Gates

### Before Parallel Work

- Strongest coordinator writes subsystem boundaries.
- Conflict-prone files have one owner.
- Public contracts and invariants are stated before implementation begins.

### Design Gate

- Data flow documented.
- Failure classes listed.
- Cache ownership rule stated.
- Data-residency behavior explicit.

### Per-Slice Gate

- Run focused tests first, such as `bazel test //nativelink-store/...`,
  `bazel test //nativelink-scheduler/...`,
  `bazel test //nativelink-worker/...`, or
  `bazel test //nativelink-service/...`.
- Report tests not run, with a reason.
- Do not broaden to full-repo tests until the focused slice is green or the
  failure is understood.

### Prototype Gate

- One Bazel target executes through NativeLink to actiond.
- Outputs are present in NativeLink CAS before result completion.
- AC is not updated on incomplete CAS upload.
- actiond VM/direct mode selected intentionally.

### Cross-Crate Gate

- Build `//:nativelink` after cross-crate Rust behavior changes.
- Run `cargo test --all --profile=smol` when Cargo-side Rust behavior changes.
- For config/protocol changes, run config loading tests and relevant service
  tests before broad validation.

### Production Gate

- mTLS configured.
- Worker drain works.
- Metrics dashboards cover scheduler, worker, CAS, and actiondfs.
- Cloud overflow cannot run office-only actions.
- Failure drills documented.

### Repo Confidence Gate

- Before claiming repo-wide readiness, run the CI-like Bazel test command used
  by NativeLink contributors:

```bash
bazel test //... --lockfile_mode=error --extra_toolchains=@rust_toolchains//:all --verbose_failures
```

## Project Tracking

Use the phase files in this directory as the living source of truth. Update the
checkboxes when work lands. Add short notes under each phase rather than
creating untracked side plans.

## Conflict-Prone Files

Only one agent should own these at a time:

- `MODULE.bazel`
- `MODULE.bazel.lock`
- `Cargo.toml`
- `Cargo.lock`
- `.bazelrc`
- `flake.lock`
- shared config schema files
- generated proto or dependency metadata

Use separate worktrees or branches for parallel implementation, then merge
through the coordinator.
