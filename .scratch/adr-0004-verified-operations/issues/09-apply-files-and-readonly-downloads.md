# 09: Apply file mutations and independently admit read-only downloads

**Status:** needs-info
**Blocked by:** 03 Approve per-type file and transfer evidence policies, 08 Execute all six service verbs after write-ahead and verify outcomes
**External blockers:** ADR-0006:01
**Source:** US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4, US-011-AC5, US-012-AC1, US-012-AC2, US-012-AC3, US-012-AC4, US-012-AC5
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Deepen the existing plan/decide workflow without owning it again. Apply approved creation/write/delete/rename/chmod/upload policies, current login-user permissions, exact paths and opening version checks; reject traversal/symlink redirection and external mtime/size/hash conflict.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
Transfer // from ADR-0006:01
TransferState // from ADR-0006:01
TransferProgress // from ADR-0006:01
TransferDispatch // from ADR-0006:01
TransferDispatchView // from ADR-0006:01
DownloadRequest // from ADR-0006:01
DownloadAdmission // from ADR-0006:01
DownloadDispatch // from ADR-0006:01
DownloadDispatchView // from ADR-0006:01
DownloadProofApproval // from ADR-0006:01
FileChange // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

No new public functions. Deepen the existing `OperationEngine::plan_operation` and `OperationEngine::decide_operation` boundaries from07/08 with the approved per-type file/transfer behavior; reuse the early opaque projections from04 and common Store projections from ADR-0006:01.

## Invariants

| ID     | Statement                                                                                                                                           | Role      | Proof                                                                 |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve  | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | preserve  | `confirmed_revision_write_ahead_single_dispatch`; first proof 05      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-4  | Remote mutation execution uses fixed typed literal login-user scripts and exact granular sudo only.                                                 | preserve  | `literal_arguments_and_exact_sudo_authority`; first proof 08          |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-6  | File mutation cannot silently change target, overwrite consent, privilege or opening assumptions.                                                   | establish | `file_external_version_and_path_promotion_safety`; first proof 09     |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-9  | Downloads use read-only typed admission, never mutation admission, and require their own complete proof policy.                                     | establish | `download_read_only_admission_and_missing_proof`; first proof 09      |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-11 | Insufficient effect evidence yields Unknown; successful process termination alone never establishes success.                                        | preserve  | `service_effect_evidence_unknown_boundary`; first proof 08            |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | preserve  | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | preserve  | `local_trust_commit_joined_outcome_and_release`; first proof 05       |

## Flow

Plan/dispatch flow4 steps6–8; private file executor and separate read-only download admission refinements.

- In: `ExecutionPermit` ← 08
- Out: `Transfer` → 10
- In: `OperationRequest` ← external
- Out: `Transfer` → final

## Steps

1. Write the failing behavior proof for Deepen the existing plan/decide workflow without owning it again. Apply approved creation/write/delete/rename/chmod/upload policies, current login-user permissions, exact paths and opening version checks; reject traversal/symlink redirection and external mtime/size/hash conflict. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Upload/write payload uses separate SFTP to a temporary file in the destination directory. Atomic posix-rename is used only when available; absent extension requires prior warning and additional explicit non-atomic consent. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Separate AdministrativeChange::Download from FileChange mutation. Its own DownloadProofApproval remains mandatory; once approved it produces DownloadAdmission/DownloadDispatch only, never EnabledMutation/ExecutionPermit. E1 performs read-only SFTP against an existing bound connection, with explicit local destination and overwrite consent. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Exercise actual payload interruption before/after rename, source/destination mismatch, absent hashing, partial local download and cancellation. Size and optional actual hash are evidence, not automatic complete predicates. Persist paths/size/results only; never file bytes. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Retain full transfer lifecycle Queued/Transferring/Verifying/Completed/Failed/Cancelled/Unknown as applicable; bytes sent, rename return or partial output alone cannot establish completion or rollback. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-011-AC1`: Given file write/edit, delete, rename, chmod or upload overwriting a file, when I initiate the change, then it passes through the Operation engine with its own Verifier and read-only Reconciliation policy before that mutation type is enabled.
- [ ] `US-011-AC2`: Given an SSH user without write permission, when a file mutation is requested, then the file is read-only or the request is rejected with a clear reason; no privileged file-editing fallback is offered in MVP.
- [ ] `US-011-AC3`: Given an edited file whose mtime, size or hash differs from the opening snapshot, when I save, then FileChangedExternally blocks the write and offers reload or explicit overwrite rather than silently replacing the external change.
- [ ] `US-011-AC4`: Given a mutation involving a symlink or a path-traversal attempt, when its destination is validated, then it cannot silently redirect an approved change to an unintended resource.
- [ ] `US-011-AC5`: Given another administrative file mutation exposed by the file manager, including file or directory creation, when its workflow is enabled, then it cannot bypass the same engine and per-type evidence prerequisite merely because it is not an overwrite.
- [ ] `US-012-AC1`: Given an upload or file write, when the payload is applied, then data is written to a temporary file in the destination directory and promoted using `posix-rename@openssh.com` when that extension is available.
- [ ] `US-012-AC2`: Given that atomic rename is unavailable, when non-atomic overwrite is proposed, then its warning and additional confirmation occur before overwrite rather than silently weakening atomicity.
- [ ] `US-012-AC3`: Given a transferred file entering VERIFYING, when destination contents are assessed, then size is compared and an optional hash comparison is used when `sha256sum` is available; unsupported hashing is not represented as a successful hash check.
- [ ] `US-012-AC4`: Given failure before or after rename, when completion is assessed, then partial transfer is never COMPLETED and the result distinguishes a proven failed change from a potentially applied change requiring UNKNOWN and Reconciliation.
- [ ] `US-012-AC5`: Given a new upload or download, when the transfer is recorded, then its path, size and result are retained as metadata without file content; its transfer lifecycle distinguishes QUEUED, TRANSFERRING, VERIFYING, COMPLETED, FAILED, CANCELLED and UNKNOWN where applicable.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_files
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/domain/resource_observation.rs src/infrastructure/sftp/mod.rs src/ui/operations.rs tests/operations_files.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.
