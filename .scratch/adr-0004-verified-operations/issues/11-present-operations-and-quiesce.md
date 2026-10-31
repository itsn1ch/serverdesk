# 11: Present actionable Operation states and quiesce owned work

**Status:** needs-info
**Blocked by:** 10 Cancel and reconcile without replay or premature lease release
**External blockers:** ADR-0006:01
**Source:** US-014-AC1, US-014-AC2, US-014-AC3, US-014-AC4, US-014-AC5, US-014-AC6, US-015-AC1, US-015-AC2, US-015-AC3, US-015-AC4, US-015-AC5, US-016-AC1, US-016-AC2, US-016-AC3, US-016-AC4, US-016-AC5
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Build actual GTK Operation/Services presentation using existing history/observation boundaries: loading, genuine empty, progress, unresolved Unknown, stale-after-disconnect and precise categories/actions/details. Keyboard-only discovery, confirmation and details must work.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
OperationQuiesced // from ADR-0006:01
OperationCompletion // from ADR-0006:01
OperationChanged // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::quiesce(&self) -> Result<OperationQuiesced, E6::Failure>`

- **Intent.** End administrative ownership and release its actual leases before transport shutdown.
- **Where.** `src/application/operations.rs`.
- **Pre.** New Runtime admission is closed.
- **Post.** Stored waiters and stop/join workers keep the mailbox serviced. The owner validates complete consumer/task coverage, settles conservative outcomes, creates WorkJoined, moves it with ResourceLease into a release worker, and waits for LeaseReleased. OperationQuiesced contains ordinary completed IDs only after acknowledgements.
- **Errors.** Unjoined work, persistence failure or rejected/incomplete release leaves quiescence incomplete and identifies the Operation/resource; no forged joined proof.
- **Idempotency.** Calls join one pending quiescence; stale/duplicate completions cannot release another lease.

## Invariants

| ID     | Statement                                                                                                                                           | Role      | Proof                                                                 |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve  | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | preserve  | `confirmed_revision_write_ahead_single_dispatch`; first proof 05      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-4  | Remote mutation execution uses fixed typed literal login-user scripts and exact granular sudo only.                                                 | preserve  | `literal_arguments_and_exact_sudo_authority`; first proof 08          |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-6  | File mutation cannot silently change target, overwrite consent, privilege or opening assumptions.                                                   | preserve  | `file_external_version_and_path_promotion_safety`; first proof 09     |
| INV-7  | GTK alone owns widgets and presents honest loading, empty, stale, error and progress states.                                                        | establish | `operations_gtk_keyboard_states_and_responsiveness`; first proof 11   |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-9  | Downloads use read-only typed admission, never mutation admission, and require their own complete proof policy.                                     | preserve  | `download_read_only_admission_and_missing_proof`; first proof 09      |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-11 | Insufficient effect evidence yields Unknown; successful process termination alone never establishes success.                                        | preserve  | `service_effect_evidence_unknown_boundary`; first proof 08            |
| INV-12 | Cancellation retains leases until all tracked work joins; Reconciliation performs reads only and never replays a mutation.                          | preserve  | `cancel_reconcile_joined_work_and_no_replay`; first proof 10          |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | preserve  | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | preserve  | `local_trust_commit_joined_outcome_and_release`; first proof 05       |

## Flow

Cancellation/Reconciliation flow 5 steps 5–6; main-context presentation and actual joined quiescence.

- In: `WorkJoined` ← 10
- In: `OperationQuery` ← external
- Out: `OperationQuiesced` → final

## Steps

1. Write the failing behavior proof for Build actual GTK Operation/Services presentation using existing history/observation boundaries: loading, genuine empty, progress, unresolved Unknown, stale-after-disconnect and precise categories/actions/details. Keyboard-only discovery, confirmation and details must work. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Keep widgets exclusively on main context; Tokio workers communicate through channels. Test responsiveness during delayed reads, long execution/verification, reconnect and unreachable Server. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Implement quiesce with stored waiters and expected work sets. Prevent new dispatch, stop/join workers while mailbox stays serviced, settle conservative durable outcomes and move WorkJoined plus lease into release workers. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Return ordinary OperationQuiesced IDs only after every release acknowledgement; never return consumed WorkJoined. Duplicate/stale completion cannot release another lease or resurrect work. Failure leaves shutdown incomplete with affected resource and possible effects. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Integrate Runtime Stopping with active Reconciliation/verification and delayed cleanup; Runtime keeps Release serviced, then stops transport only after E4 actual completion. Preserve terminal exclusion, append-only audit and all policy gates. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-014-AC1`: Given any structured mutation in scope, when its lifecycle progresses, then its Operation and associated operation_events contribute to the Audit Trail; the accepted coverage target is all structured mutations, not terminal command capture.
- [ ] `US-014-AC2`: Given an existing audit record or event, when later lifecycle information is recorded, then the append-only Audit Trail preserves prior evidence and is not automatically deleted in MVP.
- [ ] `US-014-AC3`: Given arguments, remote errors or verification details containing sensitive material, when persistence occurs, then parameters, sanitized_error and sanitized_payload contain no secret, password, passphrase or file contents.
- [ ] `US-014-AC4`: Given a terminal command typed directly by me, when it executes, then it remains outside the Operation engine and Audit Trail rather than being falsely claimed as audited structured administration.
- [ ] `US-014-AC5`: Given a recent UNKNOWN Operation, when Overview shows recent Operations, then it remains included with its actual unresolved status rather than disappearing from recent activity.
- [ ] `US-014-AC6`: Given a schema upgrade or database initialization, when operation storage is prepared, then versioned migrations, UUID identities and active foreign keys preserve accepted metadata relationships without introducing automatic audit retention or export behavior.
- [ ] `US-015-AC1`: Given a Services or Operation view still fetching data, when I open it, then it visibly shows loading rather than an invented empty or successful state.
- [ ] `US-015-AC2`: Given a successful discovery with no units or an empty Operation history, when the relevant view is shown, then it presents a genuine empty state distinct from an error or pending request.
- [ ] `US-015-AC3`: Given any workflow error, when the UI reports it, then it identifies what failed, the affected resource, whether change may have occurred, available actions and a link to Operation details where applicable.
- [ ] `US-015-AC4`: Given permission, unsupported Capability, invalid input, execution, verification, external-file-change, storage or unknown-outcome failures, when displayed, then they retain their corresponding accepted error distinction rather than collapsing into a generic connection failure.
- [ ] `US-015-AC5`: Given keyboard-only use or stale service observations after disconnect, when I navigate service details, action confirmation and Operation details, then controls remain keyboard-accessible and stale observations are visibly distinguished from current evidence.
- [ ] `US-016-AC1`: Given a connected Server and an Operation, when remote work starts, then it consumes the ADR-0001 epic's existing system OpenSSH master under ADR-0002 generated configuration rather than establishing a competing SSH implementation or bypassing trust policy.
- [ ] `US-016-AC2`: Given another conflicting Operation on the same Connection Identity and resource scope, including a Server alias with that identity, when the request is made, then the ADR-0006 lock owner rejects the conflict explicitly rather than silently queuing it; nonconflicting resources are not globally serialized by this epic.
- [ ] `US-016-AC3`: Given resources that may have changed outside the application, when preflight reads run, then they assess current state without claiming local locks prevent external changes or acquiring a new remote lock mechanism.
- [ ] `US-016-AC4`: Given Operation completion, failure or cancellation, when cleanup hands work back to the lock owner, then a pending stop is not represented as proof that owned remote work no longer conflicts; ADR-0006 owns safe lock release and startup cleanup.
- [ ] `US-016-AC5`: Given long execution, verification, reconnect or an unreachable Server, when I use the application, then GTK remains responsive, Tokio performs remote work without accessing widgets and presentation updates arrive through channels scheduled on glib::MainContext with a single owner for each state.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_gtk_shutdown
rustfmt --edition 2024 --check src/application/operations.rs src/ui/operations.rs tests/operations_gtk_shutdown.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.

## Comments

E4:01 L1 and E4:02 S decisions are approved and remain separate from runtime acceptance. Consume the complete canonical L1/S case, proof, interruption and Reconciliation tables, not a generic success predicate. Exactly six service variants exist in EnabledMutation; LocalTrustApproval has exactly Accept, Replace and Delete. Each private case is selected only within the original OperationRequest, identity, plan revision, evidence, consent and lease context. No public property or dialog mints authority. File policy 03 remains needs-info after rejection of cooperative F. This whole ticket remains needs-info through its listed file-policy dependency chain. No file variant or download proof case is approved. All source criteria and first Proof owners are retained; unavailable controls do not complete implementation.
