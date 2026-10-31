# 03: Arbitrate identity/resource conflicts and require joined-work release

**Status:** ready-for-agent
**Blocked by:** 02 Admit only after actual orphan cleanup and durable Unknown recovery
**External blockers:** None
**Source:** US-002-AC1, US-002-AC2, US-002-AC3, US-002-AC4, US-003-AC1, US-003-AC2, US-003-AC3, US-004-AC1, US-004-AC2, US-004-AC3, US-004-AC4, US-005-AC1, US-005-AC2, US-005-AC3, US-005-AC4, US-005-AC5, US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-006-AC5, US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4
**Contract:** `.scratch/adr-0006-single-instance-locks/spec.md`, section `# Design`; the five imported Store projections retain their canonical E2/E3/E4/E5 full contracts and invariant namespaces. Contract wins; stop and report a conflict.

## Goal

Use exact complete Connection Identity and operation-specific equality scopes, never inventory UUID/display state. Implement all-or-none in-memory key arbitration, immediate explicit conflict NoChange/incumbent details and no hidden queue. Do not create a scope hierarchy or remote lock.

## Types

```rust
ConnectionIdentity // from 01
ResourceScope // from 01
ResourceLease // from 01
ResourceLeaseView // from 01
LockRequest // from 01
LockResult // from 01
WorkJoined // from 01
WorkJoinedView // from 01
```

01 is the genuine declaration producer in canonical modules, and implements only the five pure Store projections. Actual report/capability issuers are the External blockers and earlier owner methods, never declarations or observation DTOs. ReadyOwnership is Clone only for the SAME admitted generation through Arc<RuntimeDirectory>; StartupOwnership and ResourceLease remain private and non-Clone.

## Functions

### `LockAuthority::arbitrate(&self, request: LockRequest) -> Result<LockResult, Failure>`

- **Intent.** Admit nonconflicting owned work immediately and release it only after joined completion.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Acquire has current Ready admission and nonempty unique planner-produced scopes; Release matches lease/Operation/current owner and sealed joined-work proof; Release stays enabled during Stopping.
- **Post.** Acquire atomically grants all keys or immediately rejects conflict with incumbent Operation details and NoChange; independent keys do not wait. Release removes only that lease’s keys after work is joined.
- **Errors.** ConfigurationConflict identifies incumbent/resource and performs no mutation; invalid release changes no ownership. Rejected acquisition is never queued.
- **Idempotency.** Repeated matching release is harmless after validated completion. A new mutation after rejection/cancellation is a new explicit request with new preflight/confirmation.

### `ResourceLease::inspect_lease(&self) -> ResourceLeaseView<'_>`

- **Intent.** Read the actual Operation, identity and scope lease binding.
- **Where.** `src/application/runtime.rs`.
- **Pre.** A live owner-issued receiver exists.
- **Post.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

## Invariants

| ID    | Statement                                                                                                                             | Role      | Proof                                                                |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------- |
| INV-1 | Only the current GtkApplication primary opens operational Store and mints current-generation ownership capabilities.                  | preserve  | `single_primary_real_store_and_preadmission`; first proof 01         |
| INV-2 | Equal Connection Identity/resource-scope keys have at most one lease; rejected requests never queue.                                  | establish | `concurrent_alias_conflict_and_independent_scopes`; first proof 03   |
| INV-3 | A lease is released only for matching Operation/generation after all owned conflicting work joins, regardless of UI or outcome label. | establish | `lease_requires_matching_joined_work`; first proof 03                |
| INV-4 | Ready follows actual orphan non-adoption and durable interrupted Running/Verifying to Unknown recovery.                               | preserve  | `actual_orphans_and_durable_unknown_before_admit`; first proof 02    |
| INV-5 | GTK owns widgets and remains responsive with truthful ownership/recovery/shutdown states.                                             | preserve  | `runtime_gtk_preadmission_and_error_states`; first proof 01          |
| INV-8 | Store accepts only sealed canonical-owner writes; immutable projections expose exact data and never authorize a new write.            | preserve  | `store_sealed_owner_requests_and_projection_privacy`; first proof 01 |

## Flow

Design flow3: exact identity/scope Acquire → immediately grant/reject → matched actual joined Release.

- In: `LockAuthority` ← 01
- In: `ReadyOwnership` ← 02
- Out: `LockAuthority` → 04
- In: `LockRequest` ← external
- Out: `LockResult` → final

LockRequest contains explicit current identity and immutable scope data from the authorized workflow/test seam. ResourceScope fields are readable immutable equality-key data, never execution authority. Full actual E4 WorkJoined issuance and service integration are preserved/exercised in04 after real producer methods exist; no proof is forged for this pure arbitration/refusal slice.

## Steps

1. Write the failing behavior proof for Use exact complete Connection Identity and operation-specific equality scopes, never inventory UUID/display state. Implement all-or-none in-memory key arbitration, immediate explicit conflict NoChange/incumbent details and no hidden queue. Do not create a scope hierarchy or remote lock. Then implement until that proof passes.
2. Write the failing behavior proof for Exercise concurrent aliases, alias rename/selection, independent scopes on one identity and independent identities; real arbitration must overlap eligible ownership rather than globally serialize. Rejected requests never execute after release without a new explicit request. Then implement until that proof passes.
3. Write the failing behavior proof for Inspect ResourceLease through its borrowed view. Release consumes exact current-generation lease and E4-sealed WorkJoined, checks Operation/lease identity and joined coverage, then removes only matching keys. Caller-created read views, cancellation intent, exit code, closed dialogs, error or Unknown never suffice. Then implement until that proof passes.
4. Write the failing behavior proof for Separate the directly observable pure arbitration/refusal proof here from the full real service execution/verification/cancel integration in04. Do not forge WorkJoined or fake owner constructors to claim mutation integration before the E4 issuer exists. Then implement until that proof passes.
5. Write the failing behavior proof for Present actual conflict/readiness failures and next explicit actions accessibly. Keep local locks distinct from remote exclusion, unmanaged-resource consent, current preflight and Verifier evidence; new explicit requests must redo current safety checks. Then implement until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-002-AC1`: Given two Server entries with the same host, port, user, Jump Server route, and authentication context, when Operations target the same underlying resource scope through those entries, then they resolve to the same Connection Identity/resource-scope conflict rather than separate inventory-ID locks.
- [ ] `US-002-AC2`: Given an admitted Operation with an active resource lock, when another Operation requests that same Connection Identity and resource scope, then the second request is explicitly rejected before conflicting mutation execution begins.
- [ ] `US-002-AC3`: Given concurrent admission attempts for the same Connection Identity/resource-scope key, when arbitration completes, then at most one conflicting Operation is admitted and the other receives the explicit conflict result.
- [ ] `US-002-AC4`: Given an inventory alias rename or a change of selected Server view without changing Connection Identity, when another request targets the already locked resource, then those presentation changes do not grant a second lock owner.
- [ ] `US-003-AC1`: Given an active Operation on one resource scope and another otherwise eligible Operation on an independent scope of the same Connection Identity, when the second request is made, then the first lock alone does not reject or defer it as a global per-connection mutation lock.
- [ ] `US-003-AC2`: Given an active Operation for one Connection Identity, when another otherwise eligible Operation targets an independent Connection Identity/resource-scope key, then the first lock alone does not globally serialize the two.
- [ ] `US-003-AC3`: Given independent eligible Operations with work still active, when both are admitted, then their externally observable execution can overlap without waiting for the unrelated Operation's completion solely because of resource locking.
- [ ] `US-004-AC1`: Given a currently locked resource, when I request a conflicting Operation, then the UI explicitly identifies the affected Server/resource and the conflict instead of appearing to accept a future execution.
- [ ] `US-004-AC2`: Given a rejected conflicting request, when the original lock is later released, then the rejected request does not automatically execute and no hidden mutation queue replays it.
- [ ] `US-004-AC3`: Given a conflict rejection, when I inspect details, then the UI indicates that this rejected request has not performed its remote mutation and provides the related Operation details and available next actions where applicable.
- [ ] `US-004-AC4`: Given that the conflicting Operation later completes, when I choose to attempt the action again, then it is a new explicit request subject to current validation, preflight, confirmation, and lock arbitration rather than a resumed hidden queue item.
- [ ] `US-005-AC1`: Given a resource changed outside ServerDesk before its Operation preflight, when the operation-specific read detects changed assumptions, then that evidence is presented to the Operation workflow and the mutation does not silently proceed under stale assumptions.
- [ ] `US-005-AC2`: Given a local resource lock held by ServerDesk, when another tool changes the remote resource, then ServerDesk does not claim that its local lock prevented the change or that the lock proves the remote state is unchanged.
- [ ] `US-005-AC3`: Given an Operation using this lock system, when its remote behavior is observed, then no remote lock is introduced by this epic and the local lock does not establish a remote-exclusive ownership guarantee.
- [ ] `US-005-AC4`: Given a remote resource that is not a Managed Resource, when an Operation is proposed, then holding or acquiring a local lock does not authorize changing it without the explicit consent required by the product ownership rule.
- [ ] `US-005-AC5`: Given an external change after an earlier preflight observation, when final evidence is insufficient for the operation-specific Verifier, then the local lock is not substituted for verification and the shared engine does not report false success.
- [ ] `US-006-AC1`: Given an admitted mutation whose execution has finished but whose owned verification is still active, when a conflicting Operation is requested, then it remains rejected until the owned conflicting workflow has finished rather than being admitted merely because the command exited.
- [ ] `US-006-AC2`: Given a completed and cleaned-up owned Operation, when a later explicit request targets the same scope, then no stale in-memory lock from that Operation prevents admission.
- [ ] `US-006-AC3`: Given an execution error or transport failure with application-owned conflicting work still active, when the error is displayed, then error presentation alone does not release the lock prematurely.
- [ ] `US-006-AC4`: Given an error path after all application-owned conflicting execution, verification, and cleanup have ended, when another explicit eligible request targets the scope, then an abandoned lock from the ended workflow does not block it.
- [ ] `US-006-AC5`: Given an UNKNOWN outcome, when lock ownership is assessed, then UNKNOWN is neither treated as proof of no remote effects nor used by itself as a reason to retain a lock after all application-owned conflicting work has ended; subsequent action still uses current preflight and the shared Operation safety rules.
- [ ] `US-007-AC1`: Given a cancellation request while owned work can still conflict, when cancellation is requested or acknowledged by the UI, then the lock is not released solely because the request was made or a progress surface closed.
- [ ] `US-007-AC2`: Given cancellation before any remote effect and evidence confirming that no remote effect occurred, when the shared Operation workflow finishes, then CANCELLED may be reported and the lock is released after owned conflicting work is cleaned up.
- [ ] `US-007-AC3`: Given cancellation after remote execution may have begun and insufficient evidence of its outcome, when cancellation finishes, then the Operation is UNKNOWN rather than CANCELLED or presumed FAILED, and the UI explains that a change may have occurred.
- [ ] `US-007-AC4`: Given the cancelled workflow has no remaining application-owned conflicting work, when cleanup ends, then a subsequent explicit request is not blocked by an abandoned lock and is not an automatic retry of the cancelled request.
- [ ] Establish each invariant Proof listed here and preserve earlier Proofs. Full runtime integration criteria are exercised again by04 with actual domain issuers, not forged early capabilities.
- [ ] No declaration-only owner, no-op backend, empty Store fallback or unsupported Ready/completed-outcome claim is introduced.

## Verify

Future implementation gates: author these exact affected tests/harnesses here; none are claimed present or passing now.

```sh
cargo test --test runtime_resource_locks
rustfmt --edition 2024 --check src/application/runtime.rs src/ui/runtime.rs tests/runtime_resource_locks.rs
```

Exercise actual concurrent Runtime arbitration/refusal and explicit conflict presentation, including alias-equivalent identity keys and independent scopes. Do not fabricate WorkJoined or claim full service execution before E4 actual issuers;04 adds real execution/verification/cancellation preservation journeys. Actual GTK surfaces must remain responsive and keyboard accessible; errors/Unknown/stale state remain honest. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

No foreign semantic writer transfer, future owner constructor, generic proof fallback, remote/distributed lock, persisted lease, new scope hierarchy, hidden mutation queue, automatic mutation replay, schema/audit reset/deletion, secret/file-content persistence or permanent greeting/wiring/text tests. Only04 owns final production launch composition.
