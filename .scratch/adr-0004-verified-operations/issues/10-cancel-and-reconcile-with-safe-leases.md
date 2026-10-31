# 10: Cancel and reconcile without replay or premature lease release

**Status:** needs-info
**Blocked by:** 01 Approve local trust lifecycle and evidence policy, 02 Approve six service Verifier and Reconciliation policies, 03 Approve per-type file and transfer evidence policies, 08 Execute all six service verbs after write-ahead and verify outcomes, 09 Apply file mutations and independently admit read-only downloads
**External blockers:** ADR-0006:01
**Source:** US-009-AC1, US-009-AC2, US-009-AC3, US-009-AC4, US-010-AC1, US-010-AC2, US-010-AC3, US-010-AC4, US-010-AC5
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Implement cancel_operation as recorded intent followed by nonblocking stop/join workers. Before effects, actual no-effect proof permits Cancelled; after possible effects without sufficient proof record Unknown. Dialog closure and error presentation never release leases.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
ReconciliationRead // from ADR-0006:01
ReconciliationReadView // from ADR-0006:01
ReconciliationBatch // from ADR-0006:01
WorkJoined // from ADR-0006:01
WorkJoinedView // from ADR-0006:01
OwnedWorkJoined // from ADR-0006:01
OperationCompletion // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::cancel_operation(&self, operation_id: Uuid) -> Result<Operation, E6::Failure>`

- **Intent.** Stop owned work without confusing a stop request with absence of effects.
- **Where.** `src/application/operations.rs`.
- **Pre.** Operation exists.
- **Post.** Records cancellation request, prevents further dispatch, stops relevant consumers and joins execution/verification/cleanup. Cancelled requires proof of no effects; possible effects with insufficient evidence become Unknown; lease lasts until all owned conflicting work ends.
- **Errors.** Cleanup/storage/verification failure preserves active ownership and uncertainty; dialog dismissal is not completion.
- **Idempotency.** Repeated calls join cancellation or return recorded disposition; never dispatch/retry/rollback.

### `OperationEngine::reconcile_operation(&self, operation_id: Uuid) -> Result<Operation, E6::Failure>`

- **Intent.** Investigate Unknown using the original type’s approved read-only predicate.
- **Where.** `src/application/operations.rs`.
- **Pre.** Unknown; complete applicable Reconciliation policy; remote case has current gated connection. Local trust has its separately approved local rule, not a remote reconnect requirement.
- **Post.** Appends proved Succeeded/Failed or still Unknown; holds appropriate lease through owned work; executes no mutation or rollback.
- **Errors.** Missing policy, lost connection, reboot-invalidated/insufficient evidence leaves Unknown and explains why; storage failure does not claim recorded resolution.
- **Idempotency.** Repeated reads may append new evidence, never repeat the change.

### `OperationEngine::reconcile_connected(&self, connection: E1::Connection) -> Result<ReconciliationBatch, E6::Failure>`

- **Intent.** Investigate eligible unresolved remote work after a current gated connection becomes usable.
- **Where.** `src/application/operations.rs`.
- **Pre.** Observation is Connected with a generation; E1 rechecks current admission and original identity when each consumer opens.
- **Post.** Select only matching recorded identity/Server remote Operations; schedule the same reconcile_operation behavior through workers while servicing mailbox. Local trust is excluded; missing policies stay Unknown in unavailable. No Connect or mutation occurs.
- **Errors.** Stale/lost/retargeted connection prevents reads and retains Unknown; Store failure never claims durable resolution.
- **Idempotency.** Coalesce the same Operation/generation; new connected generations or explicit reads can collect evidence but never replay a mutation.

## Invariants

| ID     | Statement                                                                                                                                           | Role      | Proof                                                                 |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve  | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | preserve  | `confirmed_revision_write_ahead_single_dispatch`; first proof 05      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-4  | Remote mutation execution uses fixed typed literal login-user scripts and exact granular sudo only.                                                 | preserve  | `literal_arguments_and_exact_sudo_authority`; first proof 08          |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-6  | File mutation cannot silently change target, overwrite consent, privilege or opening assumptions.                                                   | preserve  | `file_external_version_and_path_promotion_safety`; first proof 09     |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-9  | Downloads use read-only typed admission, never mutation admission, and require their own complete proof policy.                                     | preserve  | `download_read_only_admission_and_missing_proof`; first proof 09      |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-11 | Insufficient effect evidence yields Unknown; successful process termination alone never establishes success.                                        | preserve  | `service_effect_evidence_unknown_boundary`; first proof 08            |
| INV-12 | Cancellation retains leases until all tracked work joins; Reconciliation performs reads only and never replays a mutation.                          | establish | `cancel_reconcile_joined_work_and_no_replay`; first proof 10          |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | preserve  | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | preserve  | `local_trust_commit_joined_outcome_and_release`; first proof 05       |

## Flow

Cancellation/Reconciliation flow5 steps1–4; retained original identity and read-only outcome proof.

- In: `Operation` ← 08
- In: `Transfer` ← 09
- Out: `WorkJoined` → 11
- In: `Connection` ← external
- Out: `ReconciliationBatch` → final

## Steps

1. Write the failing behavior proof for Implement cancel_operation as recorded intent followed by nonblocking stop/join workers. Before effects, actual no-effect proof permits Cancelled; after possible effects without sufficient proof record Unknown. Dialog closure and error presentation never release leases. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Implement reconcile_operation with the original type-specific approved read-only predicate; missing/ambiguous/reboot-invalidated evidence retains Unknown. Local trust uses only its approved local SQLite rule, not a fake reconnect requirement. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Implement reconcile_connected over actual current gated E1 Connection. Match immutable recorded identity and Server before scheduling the same workflow; recheck expected identity at ConsumerAccess::open. A retargeted alias, stale generation or lost connection must not read another machine. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Coalesce Operation/generation scheduling. Never replay any mutation, including idempotent ones; a later change requires a fresh explicit request with validation, preflight, consent and arbitration. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Validate complete tracked consumer-end/task-join coverage and settled persistence, then create sealed WorkJoined. Move proof and ResourceLease into release worker; only actual release acknowledgement completes ownership cleanup. Unknown alone neither releases early nor retains a finished lease forever. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-009-AC1`: Given a request cancelled before any possible remote effect, when absence of effects is established, then the result can be CANCELLED.
- [ ] `US-009-AC2`: Given cancellation after remote execution may have started, when absence of effects cannot be established, then the result is UNKNOWN rather than CANCELLED.
- [ ] `US-009-AC3`: Given cancellation of an in-progress transfer, when only part of its work is known complete, then neither the transfer nor its enclosing mutation is shown as completed or successfully undone without verification.
- [ ] `US-009-AC4`: Given pending cancellation, when I inspect the UI, then it distinguishes the request to stop from confirmed no-effect cancellation and does not promise rollback.
- [ ] `US-010-AC1`: Given a previous RUNNING or VERIFYING Operation interrupted by application termination, when the startup recovery owned by the ADR-0006 epic processes it, then the Operation is UNKNOWN and marked for Reconciliation rather than resumed as if still executing.
- [ ] `US-010-AC2`: Given an unresolved remote Operation and a subsequent connection to its Server, when Reconciliation runs, then it reads remote state through that Operation type's predicate without re-executing the mutation.
- [ ] `US-010-AC3`: Given sufficient read-only evidence after reconnect, when Reconciliation completes, then the result may become SUCCEEDED or FAILED with the evidence retained.
- [ ] `US-010-AC4`: Given still-insufficient evidence, when Reconciliation completes, then the result remains UNKNOWN and is still visibly unresolved.
- [ ] `US-010-AC5`: Given any transport failure, verifier failure or Unknown Outcome, when normal handling or recovery continues, then no mutation is automatically retried, including an idempotent mutation.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_reconciliation
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/infrastructure/storage/audit.rs src/infrastructure/systemd/mod.rs src/infrastructure/sftp/mod.rs src/ui/operations.rs tests/operations_reconciliation.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.
