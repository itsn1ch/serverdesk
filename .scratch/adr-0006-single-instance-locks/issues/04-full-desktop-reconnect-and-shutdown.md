# 04: Launch the complete GTK desktop, reconcile reconnects and shut down safely

**Status:** needs-info
**Blocked by:** 02 Admit only after actual orphan cleanup and durable Unknown recovery, 03 Arbitrate identity/resource conflicts and require joined-work release
**External blockers:** ADR-0001:05, ADR-0002:03, ADR-0003:06, ADR-0004:11, ADR-0005:05
**Source:** US-001-AC1, US-001-AC2, US-001-AC3, US-001-AC4, US-002-AC1, US-002-AC2, US-002-AC3, US-002-AC4, US-003-AC1, US-003-AC2, US-003-AC3, US-004-AC1, US-004-AC2, US-004-AC3, US-004-AC4, US-005-AC1, US-005-AC2, US-005-AC3, US-005-AC4, US-005-AC5, US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-006-AC5, US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-008-AC5, US-008-AC6, US-009-AC1, US-009-AC2, US-009-AC3, US-009-AC4, US-009-AC5, US-009-AC6, US-010-AC1, US-010-AC2, US-010-AC3, US-010-AC4, US-010-AC5, US-010-AC6, US-010-AC7, US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4
**Contract:** `.scratch/adr-0006-single-instance-locks/spec.md`, section `# Design`; the five imported Store projections retain their canonical E2/E3/E4/E5 full contracts and invariant namespaces. Contract wins; stop and report a conflict.

## Goal

Implement launch as the real production GTK entry replacing temporary prefix composition. Restricted askpass dispatches only E1 serve_askpass; duplicate desktop launch activates primary only. Primary follows actual Store/cleanup/recovery/admit paths without reset, truncation or Ready shortcuts.

Complete composition remains needs-info while transitive human policy gates are unresolved; all verbatim enabled source journeys remain required, not replaced by unavailable controls.

## Types

```rust
Invocation // from 01
ApplicationPaths // from 01
DesktopServices // from 01
LifecycleOwners // from 01
ShutdownCompletion // from 01
RuntimeConnectionUpdate // from 01
OperationQuiesced // from 01
ReconciliationBatch // from 01
ConnectionView // from 01
ConnectionChanged // from 01
TransportUi // from 01
InventoryCommit // from 01
SettingsCommit // from 01
```

01 is the genuine declaration producer in canonical modules, and implements only the five pure Store projections. Actual report/capability issuers are the External blockers and earlier owner methods, never declarations or observation DTOs. ReadyOwnership is Clone only for the SAME admitted generation through Arc<RuntimeDirectory>; StartupOwnership and ResourceLease remain private and non-Clone.

## Functions

### `launch(invocation: Invocation, paths: ApplicationPaths) -> Result<(), Failure>`

- **Intent.** Run or activate one responsive desktop owner with real safety processing before managed work.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Main thread; local required facilities accessible.
- **Post.** Askpass only calls E1 helper mode. Secondary desktop only activates primary. Primary follows the bootstrap flow below, presents actual progress/failure, and exposes full managed services only after admission. Returns when lifecycle ends.
- **Errors.** Registration/storage/migration/cleanup/recovery failure blocks admission and presents specific affected resources; no orphan adoption or assumed empty audit.
- **Idempotency.** Duplicate launch cannot repeat primary startup, clear locks or open competing Store/master ownership.

### `Runtime::install_owners(&self, owners: LifecycleOwners) -> Result<(), Failure>`

- **Intent.** Install actual shutdown participants and Runtime’s own current-connection recovery feed.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Same current primary; cannot replace an installed different owner set.
- **Post.** Worker calls its own connections.observe; OwnersSubscribed installs initial snapshot/sequence before success. Subsequent updates enter RuntimeConnectionUpdate. Runtime schedules one worker per actual Connected identity/generation and receives ReconciliationCompleted; E5 uses a separate receiver. On stream loss, re-observe and process a real replacement snapshot.
- **Errors.** Observe/owner conflict leaves installation incomplete and managed startup controls blocked; never silently lose recovery scheduling.
- **Idempotency.** Identical concurrent installs join; completed install reuses its feed and never consumes E5’s receiver.

### `Runtime::shutdown(&self) -> Result<StartupSnapshot, Failure>`

- **Intent.** End every owned workflow without claiming that remote effects were undone.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Current primary.
- **Post.** Publish Stopping, close Acquire and reconnect scheduling, store waiters and launch E5/E4 quiescence workers; immediately resume mailbox. Release requests remain serviceable. Their successful completions trigger E1 quiescence, then Store close. Only owner-side ShutdownCompleted advances phases; actual completion yields Stopped.
- **Errors.** Failed/unjoined/persistence completion keeps shutdown incomplete with truthful ownership/effects; never publishes false Stopped.
- **Idempotency.** Concurrent callers join one shutdown; stale completion cannot advance another phase. Abrupt death uses next-start recovery, never mutation replay.

## Invariants

| ID    | Statement                                                                                                                             | Role      | Proof                                                                         |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------- |
| INV-1 | Only the current GtkApplication primary opens operational Store and mints current-generation ownership capabilities.                  | preserve  | `single_primary_real_store_and_preadmission`; first proof 01                  |
| INV-2 | Equal Connection Identity/resource-scope keys have at most one lease; rejected requests never queue.                                  | preserve  | `concurrent_alias_conflict_and_independent_scopes`; first proof 03            |
| INV-3 | A lease is released only for matching Operation/generation after all owned conflicting work joins, regardless of UI or outcome label. | preserve  | `lease_requires_matching_joined_work`; first proof 03                         |
| INV-4 | Ready follows actual orphan non-adoption and durable interrupted Running/Verifying to Unknown recovery.                               | preserve  | `actual_orphans_and_durable_unknown_before_admit`; first proof 02             |
| INV-5 | GTK owns widgets and remains responsive with truthful ownership/recovery/shutdown states.                                             | preserve  | `runtime_gtk_preadmission_and_error_states`; first proof 01                   |
| INV-6 | Runtime services Release while shutdown awaits E4; lifecycle changes only on owner completion messages.                               | establish | `shutdown_services_release_until_quiesced`; first proof 04                    |
| INV-7 | Runtime owns a separate real connection feed and schedules only identity-matching read-only recovery on actual usable generations.    | establish | `runtime_separate_feed_reobserve_and_identity_reconciliation`; first proof 04 |
| INV-8 | Store accepts only sealed canonical-owner writes; immutable projections expose exact data and never authorize a new write.            | preserve  | `store_sealed_owner_requests_and_projection_privacy`; first proof 01          |

## Flow

Design flow 4: main calls real launch → actual safety bootstrap → actual owners/separate feeds → read-only identity-bound recovery → worker/completion shutdown.

- In: `Runtime` ← 01
- In: `Store` ← 01
- In: `ReadyOwnership` ← 02
- In: `LockAuthority` ← 03
- In: `Invocation` ← external
- Out: `DesktopServices` → final

Actual E1/E2/E3/E4/E5 owner methods come from the qualified External blockers. src/main.rs calls launch in src/application/runtime.rs; only Runtime’s private GTK registration adapter can construct its opaque primary/directory state. E5 and Runtime each obtain their own ConnectionManager::observe receiver.

## Steps

1. Write the failing behavior proof for Implement launch as the real production GTK entry replacing temporary prefix composition. Restricted askpass dispatches only E1 serve_askpass; duplicate desktop launch activates primary only. Primary follows actual Store/cleanup/recovery/admit paths without reset, truncation or Ready shortcuts. Then implement until that proof passes.
2. Write the failing behavior proof for Install real main-thread transport authentication/VTE endpoints, construct actual E1 manager/consumer/routes from Ready authority, bind E4 transport, assemble E3 Some(routes) over the SAME TrustCore, and start E5 with its own E1 subscription and validated persisted settings. Then implement until that proof passes.
3. Write the failing behavior proof for Implement install_owners using Runtime’s separate actual ConnectionManager::observe call and worker completion. Initial snapshot/sequence must install before managed startup controls or saved connect_on_startup evaluation. It must never consume E5 receiver. Then implement until that proof passes.
4. Write the failing behavior proof for Schedule one read-only E4 reconcile_connected worker per actual Connected identity/generation, maintain sequence/generation tracking, re-observe after stream loss and use real replacement snapshots. Recheck original identity in E4/E1 so retargeted aliases cannot reconcile on another Server. Missing/ambiguous/reboot-invalidated evidence remains Unknown; no mutation replay occurs. Then implement until that proof passes.
5. Write the failing behavior proof for Shutdown publishes Stopping, rejects Acquire/new reconnect scheduling and launches E5/E4 quiesce workers while mailbox continues Release and completion handling. After actual E5 and E4 OperationQuiesced acknowledgements, join E1 descendants/masters and close Store last. Only owner completion messages advance to Stopped. Then implement until that proof passes.
6. Write the failing behavior proof for Exercise actual GTK duplicate launch while a real master/lease is active; resource conflict, independent execution overlap, preflight external change, cancellation before/after effects, delayed verification/cleanup, shutdown lease releases, abrupt death Running/Verifying and repeated recovery, all through real OpenSSH/SQLite/systemd. Then implement until that proof passes.
7. Write the failing behavior proof for Keep loading/empty/stale/Unknown/error surfaces and keyboard-only details/actions responsive throughout. Failed cleanup/storage/reconnect/release never claims successful recovery or Stopped; a new primary after crash repeats actual non-adopting cleanup and durable Unknown recovery. Then implement until that proof passes.
8. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-001-AC1`: Given an active ServerDesk GtkApplication instance, when I launch the application again, then no second independent application owner opens its own operational ownership of the same SQLite state, application ControlMasters, or lock registry.
- [ ] `US-001-AC2`: Given simultaneous launch attempts, when application ownership is established, then only one owner performs application startup recovery and admits new managed connection or mutation work.
- [ ] `US-001-AC3`: Given an active first instance with a locked resource, when a second launch occurs, then the duplicate launch does not clear that lock, start a competing master, or interrupt the first instance's running Operation.
- [ ] `US-001-AC4`: Given that the previous application owner has exited, when a later launch becomes the owner, then it performs the startup recovery workflow rather than assuming the previous in-memory locks or master ownership survived.
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
- [ ] `US-008-AC1`: Given leftover application control sockets at startup, when the sole owner performs recovery, then each remaining socket receives an ssh -O exit request and the old socket is removed before it can be used for a new managed connection.
- [ ] `US-008-AC2`: Given a still-responsive orphan application master, when startup finds its socket, then ServerDesk requests its exit and never adopts it as the new owner's active master.
- [ ] `US-008-AC3`: Given an old socket without a live responsive master, when recovery handles it, then the stale socket is removed and is not interpreted as evidence of CONNECTED.
- [ ] `US-008-AC4`: Given connect_on_startup on an affected Server, when startup recovery is in progress, then that setting does not bypass orphan/socket cleanup or cause a new managed master to coexist with an uncleared previous owner for that identity.
- [ ] `US-008-AC5`: Given cleanup cannot complete, when startup presents the failure, then it identifies the affected connection/resource and available corrective action without claiming successful cleanup, adopting the old master, or admitting affected work on assumed ownership.
- [ ] `US-008-AC6`: Given startup-owned cleanup, when application SSH resources are inspected, then cleanup is confined to application-owned runtime masters/sockets rather than treating unrelated user SSH connections as resources to adopt or terminate.
- [ ] `US-009-AC1`: Given an Operation durably left RUNNING by the previous process, when startup recovery reads it, then its recovered result is UNKNOWN and it is marked as needing Reconciliation instead of being treated as failed, cancelled, or successfully complete.
- [ ] `US-009-AC2`: Given an Operation durably left VERIFYING by the previous process, when startup recovery reads it, then its recovered result is UNKNOWN and it is marked as needing Reconciliation even if the remote command had exited successfully.
- [ ] `US-009-AC3`: Given recovery of an interrupted Operation, when its details are shown, then the original request, available verification evidence, and Audit Trail remain available and the UI says a remote change may have occurred.
- [ ] `US-009-AC4`: Given interrupted Operations and enabled startup connection, when the new owner starts managed work, then startup recovery establishes their UNKNOWN/reconciliation-needed status rather than presenting the old RUNNING/VERIFYING records as current owned execution.
- [ ] `US-009-AC5`: Given failure to persist required recovery changes, when startup recovery completes unsuccessfully, then the affected workflow reports StorageError and does not claim that the interrupted outcome has been safely recovered or reconciled.
- [ ] `US-009-AC6`: Given a crash followed by more than one later application start, when the same unresolved Operation is inspected, then it is not erased, silently retried, or converted to success merely because no previous lock or local process remains.
- [ ] `US-010-AC1`: Given an UNKNOWN interrupted Operation requiring Reconciliation, when the affected Server has not reconnected, then the record remains unresolved and startup does not replay its mutation to obtain an answer.
- [ ] `US-010-AC2`: Given that the affected Server reconnects under the normal current trust/authentication policy, when Reconciliation runs, then it reads remote state using that Operation type's accepted Reconciliation predicate and performs no replay of the original mutation.
- [ ] `US-010-AC3`: Given sufficient evidence meeting the operation-specific success predicate, when read-only Reconciliation completes, then the outcome may become SUCCEEDED and its evidence is available in Operation details.
- [ ] `US-010-AC4`: Given sufficient evidence establishing the operation-specific failure predicate, when read-only Reconciliation completes, then the outcome may become FAILED without repeating the mutation.
- [ ] `US-010-AC5`: Given unavailable, ambiguous, or insufficient evidence, including a failed reconnect or a remote reboot that invalidates evidence, when Reconciliation cannot determine the result, then the outcome remains UNKNOWN rather than fabricating success or failure.
- [ ] `US-010-AC6`: Given an unresolved or reconciled Operation, when recovery finishes, then there is no automatic mutation retry; any later mutation requires a new explicit request through current validation, preflight, confirmation where required, and lock arbitration.
- [ ] `US-010-AC7`: Given a proposed Operation type without its own Verifier and Reconciliation predicate, when its availability is evaluated, then startup recovery does not supply a generic success fallback and the shared engine's prerequisite for enabling that mutation remains in force.
- [ ] `US-011-AC1`: Given startup cleanup, recovery persistence, preflight, or Reconciliation is running, when I interact with GTK, then the main thread remains responsive and affected work has a loading/recovery presentation rather than an unsupported success claim.
- [ ] `US-011-AC2`: Given no interrupted Operations, when recovery-related results are shown, then an honest empty state is available; given unresolved Operations, then UNKNOWN and unavailable or stale evidence remain visible rather than being presented as Server downtime.
- [ ] `US-011-AC3`: Given a cleanup failure, storage failure, conflict, or unresolved outcome, when the error is displayed, then it identifies what failed, the affected Server/resource, whether a change may have occurred, available actions, and related Operation details where applicable.
- [ ] `US-011-AC4`: Given recovery details or a rejected-conflict surface, when I use keyboard navigation, then I can reach the details and available actions without a pointer or a forced automatic mutation retry.
- [ ] Establish each invariant Proof listed here and preserve earlier Proofs. Full runtime integration criteria are exercised again by04 with actual domain issuers, not forged early capabilities.
- [ ] No declaration-only owner, no-op backend, empty Store fallback or unsupported Ready/completed-outcome claim is introduced.

## Verify

Future implementation gates: author these exact affected tests/harnesses here; none are claimed present or passing now.

```sh
cargo test --test runtime_desktop_lifecycle
rustfmt --edition 2024 --check src/application/runtime.rs src/infrastructure/storage/mod.rs src/main.rs src/ui/runtime.rs tests/runtime_desktop_lifecycle.rs
```

Run actual production GTK entry under a session bus and isolated Incus/LXD Ubuntu24.04/systemd255/sshd with actual OpenSSH, granular sudoers and SQLite. Cover simultaneous/duplicate launches during active master/lease, independent execution overlap, external change, delayed verification/cancel/cleanup, active shutdown Release, both Running/Verifying crashes, repeated starts, separate/lost feeds, alias retargeting and read-only Reconciliation with no mutation replay. Actual GTK surfaces must remain responsive and keyboard accessible; errors/Unknown/stale state remain honest. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

No foreign semantic writer transfer, future owner constructor, generic proof fallback, remote/distributed lock, persisted lease, new scope hierarchy, hidden mutation queue, automatic mutation replay, schema/audit reset/deletion, secret/file-content persistence or permanent greeting/wiring/text tests. Only04 owns final production launch composition.

## Comments

E1, T3, L1 and S decisions are approved. Full desktop remains needs-info through file-policy 03, E4:07/08/09/10/11 and E3:06. Do not relabel a partial desktop as full completion. L1 is the explicit local-only exception to the preserved US-010-AC1/AC2 remote reconnect wording. After startup durably marks interrupted work Unknown, authoritative local Store and sole ownership allow local read-only Reconciliation only after actual original-worker closure and settled SQLite. Apply historical attributed plan/event/read-back proof and show current trust separately. Remote cases still wait for current gated reconnect and original identity. No mutation replay or fabricated reports are allowed. Actual final launch, reconnect feeds and shutdown proofs remain unrun.
