# 04: Inspect permanent audit and durably recover interrupted Operations

**Status:** ready-for-agent
**Blocked by:** None (can start immediately)
**External blockers:** ADR-0006:01, ADR-0002:01, ADR-0003:01
**Source:** US-005-AC4, US-005-AC5, US-010-AC1, US-014-AC1, US-014-AC2, US-014-AC3, US-014-AC4, US-014-AC5, US-014-AC6
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Through the real primary Store and actual SQLite migrations, introduce audit reading and permanent Operation details, including actual empty versus failed read. Do not require SSH or a completed mutation policy.

## Types

```rust
OperationEngine // from ADR-0006:01
Operation // from ADR-0006:01
OperationIntent // from ADR-0006:01
OperationRecorded // from ADR-0006:01
OperationFact // from ADR-0006:01
AuditTail // from ADR-0006:01
AuditAppend // from ADR-0006:01
AuditBatchData // from ADR-0006:01
AuditStoreRequest // from ADR-0006:01
AuditStoreResult // from ADR-0006:01
OperationHistory // from ADR-0006:01
OperationQuery // from ADR-0006:01
OperationChanged // from ADR-0006:01
OperationMessage // from ADR-0006:01
RecoveryReport // from ADR-0006:01
RecoveryReportView // from ADR-0006:01
OperationCompletion // from ADR-0006:01
ExecutionPermit // from ADR-0006:01
ExecutionPermitView // from ADR-0006:01
RemoteDispatch // from ADR-0006:01
RemoteDispatchView // from ADR-0006:01
TransferDispatch // from ADR-0006:01
TransferDispatchView // from ADR-0006:01
ReconciliationRead // from ADR-0006:01
ReconciliationReadView // from ADR-0006:01
LocalTrustEnvelope // from ADR-0006:01
LocalTrustEnvelopeView // from ADR-0006:01
WorkJoined // from ADR-0006:01
WorkJoinedView // from ADR-0006:01
DownloadAdmission // from ADR-0006:01
DownloadDispatch // from ADR-0006:01
DownloadDispatchView // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01, ADR-0002:01, ADR-0003:01. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::spawn(store: E6::Store, configuration: E2::Configuration, trust: E3::TrustExecutor, locks: E6::LockAuthority) -> OperationEngine`

- **Intent.** Establish an actual audit/recovery and policy-gated Operation owner without requiring SSH bootstrap.
- **Where.** `src/application/operations.rs`.
- **Pre.** Primary Store contains real compatible operations/events tables.
- **Post.** Audit queries and startup recovery work immediately; remote methods explicitly lack transport until bound; no mutation or fabricated empty-store shortcut.
- **Errors.** Subsequent real Store failures remain StorageError.
- **Idempotency.** Once per primary; construction neither clears audit nor resumes work.

### `OperationEngine::inspect_operations(&self, query: OperationQuery) -> Result<OperationHistory, E6::Failure>`

- **Intent.** Show permanent intent, progress, evidence and unresolved outcomes.
- **Where.** `src/application/operations.rs`.
- **Pre.** Valid bounded query where a limit applies.
- **Post.** Folds actual facts; Recent includes Unknown; empty is distinct from storage failure.
- **Errors.** ValidationError/StorageError changes no audit or remote state.
- **Idempotency.** Read-only across repeated calls/restarts.

### `OperationEngine::observe(&self) -> Result<E6::Subscription<OperationHistory, OperationChanged>, E6::Failure>`

- **Intent.** Deliver durable Operation progress to responsive presentation.
- **Where.** `src/application/operations.rs`.
- **Pre.** Primary actor exists.
- **Post.** Snapshot and subsequent events share an observation boundary; committed state only is described as durable.
- **Errors.** Store/owner failure is explicit; no false empty history.
- **Idempotency.** Subscription never requests mutation or replay.

### `OperationEngine::recover_interrupted(&self, startup: &E6::StartupOwnership) -> Result<RecoveryReport, E6::Failure>`

- **Intent.** Durably expose prior Running/Verifying work as unresolved before new admission.
- **Where.** `src/application/operations.rs`.
- **Pre.** Current primary holds startup; remote dispatch remains closed.
- **Post.** Appends interruption facts for every Running/Verifying Operation, yielding Unknown/reconciliation-needed with original intent/evidence retained; returns actual committed IDs.
- **Errors.** Read/append/migration/storage failure returns StorageError and no recovery-complete report.
- **Idempotency.** Already Unknown remains unresolved without another interruption conversion; repeated starts never erase/retry it.

### `RemoteDispatch::inspect_remote_dispatch(&self) -> RemoteDispatchView<'_>`

- **Intent.** Read the exact admitted typed remote request without issuing authority.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `TransferDispatch::inspect_transfer_dispatch(&self) -> TransferDispatchView<'_>`

- **Intent.** Read the admitted upload/write payload binding.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `ReconciliationRead::inspect_reconciliation_read(&self) -> ReconciliationReadView<'_>`

- **Intent.** Read the original Operation and approved read-only policy reference.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `WorkJoined::inspect_joined_work(&self) -> WorkJoinedView<'_>`

- **Intent.** Read the actual lease/Operation/joined-work IDs for E6 release validation.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `RecoveryReport::inspect_recovery(&self) -> RecoveryReportView<'_>`

- **Intent.** Read the actually committed startup recovery result.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `DownloadDispatch::inspect_download_dispatch(&self) -> DownloadDispatchView<'_>`

- **Intent.** Read the specifically admitted read-only download without receiving mutation authority.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact immutable expected ConnectionIdentity plus existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

### `ExecutionPermit::inspect_execution_permit(&self) -> ExecutionPermitView<'_>`

- **Intent.** Read the admitted operation/revision/write-ahead/lease binding.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact immutable expected ConnectionIdentity plus existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

## Invariants

| ID     | Statement                                                                                                     | Role      | Proof                                                             |
| ------ | ------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.          | establish | `missing_type_policy_blocks_every_action`; first proof 04         |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission. | establish | `interrupted_audit_recovery_is_durable_unknown`; first proof 04   |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                       | establish | `audit_is_permanent_append_only_and_sanitized`; first proof 04    |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.       | establish | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04 |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages. | establish | `operation_mailbox_progress_during_pending_work`; first proof 04  |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.   | establish | `operation_projection_privacy_and_exact_binding`; first proof 04  |

## Flow

History/startup recovery flow1; all early read projections and data-only decomposition boundaries.

- Out: `OperationEngine` → 05
- Out: `OperationEngine` → 06
- Out: `OperationHistory` → 07
- In: `OperationQuery` ← external
- Out: `OperationHistory` → final

## Steps

1. Write the failing behavior proof for Through the real primary Store and actual SQLite migrations, introduce audit reading and permanent Operation details, including actual empty versus failed read. Do not require SSH or a completed mutation policy. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Implement append-only intent/events and sanitizer before constructing any AuditAppend. Encode ordinal only in sanitized_payload; E4 reserves contiguous successors per Operation, Store atomically checks expected tails/FK/UUIDs/ordinals, and fold uses validated ordinals. Never SQL rowid chronology. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Exercise conflicting/concurrent appends, reversed retrieval, clock rollback, matching/conflicting duplicates and ambiguous write completion; actual history/tail read must precede a later reservation. Deny audit UPDATE/DELETE and preserve all evidence across restart. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Recover actual persisted Running/Verifying into Unknown/reconciliation-needed with one durable interruption fact, retained request/evidence and real report IDs; missing tables, migration/read/write failure must never return successful empty recovery. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Keep actor mailbox responsive while storage work is pending; record pending replies and complete through owner messages. Every service/file/trust/download action lacking its own policy is unavailable with the specific missing decision, not presented as delivered enabled behavior. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-005-AC4`: Given a persisted Operation, when I inspect its record, then the accepted Operation metadata identifies its Server, resource, action, sanitized parameters, classification, status, timing, verification, sanitized error and correlation identifier as applicable to its stage.
- [ ] `US-005-AC5`: Given an Operation requiring multiple related local persistence changes, when those changes are committed, then transactions and active foreign keys preserve their consistency and no storage value is silently truncated.
- [ ] `US-010-AC1`: Given a previous RUNNING or VERIFYING Operation interrupted by application termination, when the startup recovery owned by the ADR-0006 epic processes it, then the Operation is UNKNOWN and marked for Reconciliation rather than resumed as if still executing.
- [ ] `US-014-AC1`: Given any structured mutation in scope, when its lifecycle progresses, then its Operation and associated operation_events contribute to the Audit Trail; the accepted coverage target is all structured mutations, not terminal command capture.
- [ ] `US-014-AC2`: Given an existing audit record or event, when later lifecycle information is recorded, then the append-only Audit Trail preserves prior evidence and is not automatically deleted in MVP.
- [ ] `US-014-AC3`: Given arguments, remote errors or verification details containing sensitive material, when persistence occurs, then parameters, sanitized_error and sanitized_payload contain no secret, password, passphrase or file contents.
- [ ] `US-014-AC4`: Given a terminal command typed directly by me, when it executes, then it remains outside the Operation engine and Audit Trail rather than being falsely claimed as audited structured administration.
- [ ] `US-014-AC5`: Given a recent UNKNOWN Operation, when Overview shows recent Operations, then it remains included with its actual unresolved status rather than disappearing from recent activity.
- [ ] `US-014-AC6`: Given a schema upgrade or database initialization, when operation storage is prepared, then versioned migrations, UUID identities and active foreign keys preserve accepted metadata relationships without introducing automatic audit retention or export behavior.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_audit
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/infrastructure/storage/audit.rs src/ui/operations.rs src/domain/mod.rs src/application/mod.rs src/infrastructure/mod.rs src/ui/mod.rs tests/operations_audit.rs
```

Use actual SQLite versioned migrations, append-only/FK/ordinal guards, reversed physical retrieval, ambiguous writes and durable interrupted records; missing/read/write/schema failures are not empty. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.
