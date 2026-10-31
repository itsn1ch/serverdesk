# 08: Execute all six service verbs after write-ahead and verify outcomes

**Status:** needs-info
**Blocked by:** 02 Approve six service Verifier and Reconciliation policies, 07 Plan exact revision-bound changes and required consent
**External blockers:** ADR-0006:01, ADR-0003:05
**Source:** US-005-AC1, US-005-AC2, US-005-AC3, US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4, US-007-AC5, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-008-AC5
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Implement decide_operation exactly once; accept only the current AwaitingConfirmation revision plus every required acknowledgement, current admission and atomic E6 lease. Fresh preflight/permission failures or a conflict cannot dispatch.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
ExecutionPermit // from ADR-0006:01
ExecutionPermitView // from ADR-0006:01
RemoteDispatch // from ADR-0006:01
RemoteDispatchView // from ADR-0006:01
EnabledMutation // from ADR-0006:01
LocalTrustApproval // from ADR-0006:01
ServiceVerb // from ADR-0006:01
ServiceEvidence // from ADR-0006:01
VerificationEvidence // from ADR-0006:01
LocalTrustAdmitted // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01, ADR-0003:05. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::decide_operation(&self, operation_id: Uuid, decision: OperationDecision) -> Result<Operation, E6::Failure>`

- **Intent.** Consume or reject the exact reviewed plan without duplicate dispatch.
- **Where.** `src/application/operations.rs`.
- **Pre.** Matching AwaitingConfirmation revision; all required acknowledgements; admission open; complete applicable policy; atomic identity/scope lease acquisition succeeds.
- **Post.** Reject before effects records Cancelled. Confirm rechecks assumptions and authority, commits Running, then permits one dispatch; execution enters Verifying; only approved proof yields Succeeded/Failed, otherwise Unknown. Local trust delegates the same admit_local_trust stage, consumes LocalTrustAdmitted into its original delta/envelope, invokes actual E3 commit, and settles approved local outcome through the canonical completion path; no second trust workflow.
- **Errors.** Stale revision/assumptions, conflict, missing acknowledgement/policy/permission or failed write-ahead causes no dispatch. After possible effects, errors preserve uncertainty; failed final persistence never claims a durable outcome.
- **Idempotency.** Consumed revision cannot dispatch again; duplicate returns recorded result/conflict. Crash after Running is recovered Unknown, never retried.

## Invariants

| ID     | Statement                                                                                                                                           | Role      | Proof                                                                 |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve  | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | preserve  | `confirmed_revision_write_ahead_single_dispatch`; first proof 05      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-4  | Remote mutation execution uses fixed typed literal login-user scripts and exact granular sudo only.                                                 | establish | `literal_arguments_and_exact_sudo_authority`; first proof 08          |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-11 | Insufficient effect evidence yields Unknown; successful process termination alone never establishes success.                                        | establish | `service_effect_evidence_unknown_boundary`; first proof 08            |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | preserve  | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | preserve  | `local_trust_commit_joined_outcome_and_release`; first proof 05       |

## Flow

Plan/dispatch flow 4 steps 3–5/8; exact service verbs and actual local trust public workflow integration.

- In: `OperationPlan` ← 07
- Out: `ExecutionPermit` → 09
- Out: `Operation` → 10
- In: `OperationDecision` ← external
- Out: `Operation` → final

## Steps

1. Write the failing behavior proof for Implement decide_operation exactly once; accept only the current AwaitingConfirmation revision plus every required acknowledgement, current admission and atomic E6 lease. Fresh preflight/permission failures or a conflict cannot dispatch. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Commit Running with actual SQLite transaction before any remote effect. Inject write-ahead failure and prove no remote mutation starts; duplicate/ambiguous confirmation cannot consume a revision twice. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Dispatch exactly Start, Stop, Restart, Reload, Enable and Disable through their respective fixed POSIX stdin scripts as login user. Only the exact privileged systemctl command receives sudo -n; no sudo shell/password or polkit fallback. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Process exit enters Verifying. Exercise each human-approved supported semantics predicate and read-only Reconciliation policy; unsupported type/unit semantics remains unavailable. For restart retain boot_id, applicable remote timestamps, actual state/result samples and approved stabilization; occurrence alone is not healthy success. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Integrate E3 local trust commit using sealed LocalTrustEnvelope with expected tail, write-ahead and exact paired event ID/ordinal. Store key mutation/read-back and fact are atomic; E3 retains trust authority. Final persistence failure never claims a durable outcome. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-005-AC1`: Given a new structured mutation, when it progresses, then its workflow exposes the applicable REQUESTED, VALIDATING, PLANNING, AWAITING_CONFIRMATION, RUNNING and VERIFYING stages rather than skipping directly from a click to success.
- [ ] `US-005-AC2`: Given a validated and authorized plan, when execution is about to start, then the Operation's RUNNING state is committed in a SQLite transaction before remote execution begins.
- [ ] `US-005-AC3`: Given that the write-ahead transaction fails, when execution is considered, then no remote mutation is started and a StorageError explains that execution was prevented.
- [ ] `US-006-AC1`: Given a connected Server and a unit with the necessary detected Capability, exact permission and complete operation-specific evidence policy, when I open its service actions, then I can choose the supported start, stop, restart, reload, enable and disable actions by their actual names rather than being limited to restart or an arbitrary command field.
- [ ] `US-006-AC2`: Given an available service action and any required confirmation, when I choose Start, Stop, Restart, Reload, Enable or Disable, then the Operation details identify the selected Server, unit and corresponding action, and execution requests exactly `systemctl start`, `systemctl stop`, `systemctl restart`, `systemctl reload`, `systemctl enable` or `systemctl disable` respectively for that unit through the authorized Operation workflow, without substituting another verb or adding an unrequested mutation.
- [ ] `US-006-AC3`: Given any service action whose own Verifier or read-only Reconciliation predicate is missing, when I inspect or attempt that action, then it is unavailable with the missing evidence-policy reason even if connection and sudo permission are otherwise ready; another verb's policy does not enable it.
- [ ] `US-006-AC4`: Given an action with an evidence policy that does not cover the selected unit's semantics, when I inspect or attempt that action, then it is unavailable with that unsupported-Capability reason rather than silently using another unit type's evidence, interpreting active state as enablement evidence or assuming disable also stops the unit.
- [ ] `US-007-AC1`: Given a unit whose semantics support the documented evidence, when restart preflight runs, then it captures remote boot_id and the unit's ActiveEnterTimestampMonotonic and/or ExecMainStartTimestampMonotonic as applicable to those semantics.
- [ ] `US-007-AC2`: Given the same boot_id before and after execution, when the applicable monotonic unit timestamp increases, then the documented evidence can establish that a restart occurred but does not alone establish healthy success.
- [ ] `US-007-AC3`: Given occurrence evidence, when health is evaluated, then ActiveState, SubState and Result must satisfy the operation-specific expectations throughout the defined stabilization window before healthy success is reported.
- [ ] `US-007-AC4`: Given a changed boot_id during restart, when the result is assessed, then the reboot is not automatically interpreted as either successful restart or failure.
- [ ] `US-007-AC5`: Given unsupported timestamp semantics, missing evidence or an incomplete health/stabilization policy, when restart proof is requested, then the workflow does not invent timestamps, an expected state or a stabilization duration to declare success.
- [ ] `US-008-AC1`: Given a mutation that has executed, when the process exits, then the Operation enters VERIFYING rather than treating exit status alone as SUCCEEDED.
- [ ] `US-008-AC2`: Given sufficient operation-specific proof of success, when verification finishes, then and only then is SUCCEEDED presented.
- [ ] `US-008-AC3`: Given sufficient operation-specific proof of failure, when verification finishes, then FAILED is recorded with its available evidence rather than being confused with an unobserved remote outcome.
- [ ] `US-008-AC4`: Given SSH loss after a mutation may have begun or otherwise insufficient evidence, when the outcome is assessed, then UNKNOWN is recorded with UnknownRemoteOutcome as appropriate, not an unproven FAILED, CANCELLED or SUCCEEDED.
- [ ] `US-008-AC5`: Given a VerificationError, when the Verifier cannot establish a definite outcome, then the error preserves the Unknown Outcome boundary rather than converting inability to verify into proof that no remote effect occurred.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_services
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/infrastructure/storage/audit.rs src/infrastructure/systemd/mod.rs src/ui/operations.rs tests/operations_services.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.

## Comments

E4:01 L1 and E4:02 S decisions are approved and remain separate from runtime acceptance. Consume the complete canonical L1/S case, proof, interruption and Reconciliation tables, not a generic success predicate. Exactly six service variants exist in EnabledMutation; LocalTrustApproval has exactly Accept, Replace and Delete. Each private case is selected only within the original OperationRequest, identity, plan revision, evidence, consent and lease context. No public property or dialog mints authority. File policy 03 remains needs-info after rejection of cooperative F. This whole ticket remains needs-info through its listed file-policy dependency chain. No file variant or download proof case is approved. All source criteria and first Proof owners are retained; unavailable controls do not complete implementation.

### Approved S implementation input

Use the complete canonical E4 supported-unit matrix and each separate Start/Stop/Restart/Reload/Enable/Disable success, definite-failure, Unknown and read-only Reconciliation predicate. Running-service health is active/running/success with stable invocation and nonzero MainPID. Retained oneshot health is active/exited/success with MainPID zero and successful invocation completion. Completed oneshot completion is inactive/dead/success with MainPID zero and a new successful execution record, not daemon health. Reload is running-service only with actual notify-reload completion or documented synchronous ExecReload; signal-only asynchronous reload is unsupported.

Capture actual applicable baseline timestamps and boot_id. Restart occurrence is not health. S requires a complete observable 10-second window with observations at every 1-second boundary including the start/end, same boot, and no contrary intervening evidence. Missing observations are Unknown; a contrary observed window fails rather than resetting until eventual success. Reconciliation may prove original occurrence then use a new complete later window, explicitly reporting later health rather than health during the missing interval.

Enable/Disable prove the exact persistent state and full resolved install-link impact independently from ActiveState, with explicit no-op cases. Never add --now, another service verb, reset-failed, unmask or a separate daemon-reload. Preserve partial effects and honest uncertainty. All impact, definition, alias, manager, permission and property assumptions must be actually detected, bound to the original request/plan/identity/lease and rechecked at admission. Existing RequestedChange and applicable DestructiveImpact/UnmanagedResource consents cover the exact reviewed impact; consent does not create a missing technical property. The full canonical predicates, including exclusions and per-verb failure/correlation rules, govern beyond this summary. The whole ticket still awaits file-policy 03 through 07.
