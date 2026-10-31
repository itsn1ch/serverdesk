# 07: Plan exact revision-bound changes and required consent

**Status:** needs-info
**Blocked by:** 01 Approve local trust lifecycle and evidence policy, 02 Approve six service Verifier and Reconciliation policies, 03 Approve per-type file and transfer evidence policies, 04 Inspect permanent audit and durably recover interrupted Operations, 05 Review and admit local trust Operations before effects, 06 Inspect Services and bounded file-opening evidence without mutation privilege
**External blockers:** ADR-0006:01, ADR-0003:03, ADR-0006:03
**Source:** US-002-AC1, US-002-AC2, US-002-AC3, US-002-AC4, US-002-AC5, US-002-AC6, US-003-AC1, US-003-AC2, US-003-AC3, US-003-AC4, US-003-AC5, US-004-AC1, US-004-AC2, US-004-AC3, US-004-AC4, US-004-AC5, US-004-AC6, US-013-AC1, US-013-AC2, US-013-AC3, US-013-AC4, US-005-AC1, US-006-AC3, US-006-AC4
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

After accepted human policy updates, implement plan_operation across explicit Server resolution, typed unit/path/mode validation, actual current preflight, classification and per-command/per-argument sudo -n -l checks. Missing policies continue returning Blocked, not an approximate executable plan.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
OperationRequest // from ADR-0006:01
AdministrativeChange // from ADR-0006:01
OperationClass // from ADR-0006:01
OperationPlan // from ADR-0006:01
PlanResult // from ADR-0006:01
Confirmation // from ADR-0006:01
OperationDecision // from ADR-0006:01
ResourceScope // from ADR-0006:01
LocalTrustPlan // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01, ADR-0003:03, ADR-0006:03. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::plan_operation(&self, request: OperationRequest) -> Result<PlanResult, E6::Failure>`

- **Intent.** Present an executable change only when its exact target, authority and evidence policy are complete.
- **Where.** `src/application/operations.rs`.
- **Pre.** Explicit user intent and Server context; valid typed arguments.
- **Post.** Local Trust requests delegate the same prepare_local_trust stage and retain its sealed plan; no parallel trust policy/workflow. Records Requested and validation honestly. Missing policy records a blocked/no-effect outcome and required decisions, never a confirmable dispatch. Complete applicable policy is required before Planning/AwaitingConfirmation and immutable revision/impact/scopes/acknowledgements.
- **Errors.** Storage failure returns no false durable request; validation/permission/external-change/Capability/policy failures perform no mutation and retain any already-recorded facts.
- **Idempotency.** Each explicit new request creates one Operation; redelivery is not a new intent. Crash before dispatch cannot execute an unconfirmed plan.

## Invariants

| ID     | Statement                                                                                                                                           | Role     | Proof                                                                 |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | preserve | `confirmed_revision_write_ahead_single_dispatch`; first proof 05      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | preserve | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve | `operation_projection_privacy_and_exact_binding`; first proof 04      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | preserve | `local_trust_commit_joined_outcome_and_release`; first proof 05       |

## Flow

Plan flow 4 steps 1–2; local trust delegates canonical flow 2 rather than creating a second workflow.

- In: `OperationHistory` ← 04
- In: `LocalTrustPlan` ← 05
- In: `ResourceObservation` ← 06
- Out: `OperationPlan` → 08
- In: `OperationRequest` ← external
- Out: `OperationPlan` → final

## Steps

1. Write the failing behavior proof for After accepted human policy updates, implement plan_operation across explicit Server resolution, typed unit/path/mode validation, actual current preflight, classification and per-command/per-argument sudo -n -l checks. Missing policies continue returning Blocked, not an approximate executable plan. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Use only one private fixed-template/argument encoder design: validate accepted unit grammar and preserve literal shell-significant values; payload is separate SFTP. Suggested sudoers text is guidance only, never installed. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Persist Requested/Validating/Planning/AwaitingConfirmation facts and immutable target, impact, scopes, revision and applicable acknowledgements. Quick Overview restart enters the same explicit Server workflow. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for E3 prepare_change supplies exact local trust semantics; E6 supplies canonical identity/resource arbitration. Local trust has no fabricated SSH/reconnect prerequisite. Consent includes destructive impact, destination overwrite, unmanaged resource and trust replacement/deletion as applicable. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for External-file conflict requires reload or new explicit overwrite revision, not a boolean bypass. A local lease does not authorize unmanaged changes or prevent external changes. Then implement this boundary until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-002-AC1`: Given a Server connecting, when Capability detection runs, then systemd and busctl JSON support are detected and sudo permissions are assessed per Operation and arguments rather than summarized as a single unrestricted-sudo flag.
- [ ] `US-002-AC2`: Given a requested privileged change, when preflight checks authority, then `sudo -n -l` checks the exact absolute command path and exact arguments that the Operation would execute.
- [ ] `US-002-AC3`: Given an authorized service mutation, when execution begins, then the script remains the login user and only the privileged command uses `sudo -n /usr/bin/systemctl <verb> <unit>`.
- [ ] `US-002-AC4`: Given insufficient NOPASSWD permission, when I attempt that mutation, then it is blocked with the affected command's missing permission and no root shell, sudo password prompt, stored sudo password or D-Bus/polkit mutation fallback is attempted.
- [ ] `US-002-AC5`: Given a missing privilege, when I view a suggested sudoers snippet, then it is guidance text only and the application neither installs it nor changes sudoers.
- [ ] `US-002-AC6`: Given permissions differ between verbs or units, when I inspect available actions, then permission for one verb/unit does not authorize another, and permitted read-only service information remains usable independently.
- [ ] `US-003-AC1`: Given a structured remote Operation, when it is dispatched, then it selects a fixed POSIX sh template sent through stdin to `sh -s -- <args>` rather than accepting arbitrary command text.
- [ ] `US-003-AC2`: Given a supplied argument, when validation runs, then its accepted domain type is checked before dispatch, including systemd unit-name grammar for unit arguments, and invalid input produces ValidationError without executing the remote mutation.
- [ ] `US-003-AC3`: Given valid values containing shell-significant characters, when they are encoded, then the single tested argument-quoting function preserves the literal values and no command interpolation occurs.
- [ ] `US-003-AC4`: Given a completed script exchange, when the workflow interprets it, then it consumes machine-readable key/value or NUL-delimited records and meaningful exit codes, not a human-oriented table or a presumed success inferred from SSH stderr.
- [ ] `US-003-AC5`: Given a file transfer accompanying an Operation, when file bytes are transferred, then payload uses its separate SFTP channel rather than the script's stdin or an interpolated command argument.
- [ ] `US-004-AC1`: Given a structured request, when planning presents it, then its Server context, affected resource, action and classification are explicit rather than inferred from an unrelated currently selected Server.
- [ ] `US-004-AC2`: Given the classification step, when the request is assessed, then it is classified as read-only, idempotent mutation, non-idempotent mutation or destructive mutation without treating every action as harmless because the transport command can be repeated.
- [ ] `US-004-AC3`: Given a destructive mutation, when execution is requested, then it cannot proceed until I explicitly confirm the described impact.
- [ ] `US-004-AC4`: Given an existing destination that will be overwritten, when I request the change, then overwrite confirmation is required before replacement.
- [ ] `US-004-AC5`: Given a remote resource that is not a Managed Resource, when a workflow proposes changing it, then it requires my explicit consent rather than assuming application ownership.
- [ ] `US-004-AC6`: Given a service restart quick action in Overview, when I invoke it, then it enters this same planned and verified Operation workflow for the explicitly shown Server rather than bypassing it.
- [ ] `US-013-AC1`: Given a requested Host Key trust change, when it is applied, then it participates in the Operation engine and sanitized Audit Trail rather than writing around them.
- [ ] `US-013-AC2`: Given replacement or deletion of a trust entry, when I proceed, then explicit consent and the affected Server and Host Key identity are part of the request rather than automatic acceptance of observed drift.
- [ ] `US-013-AC3`: Given a trust mutation type without its own verifiable persisted-state outcome and Reconciliation rule, when the action is considered for enablement, then it remains unavailable under the same evidence prerequisite; no remote shell or remote-reconnect prerequisite is fabricated for a local SQLite trust commit.
- [ ] `US-013-AC4`: Given an accepted trust mutation, when its outcome is consumed by connection handling, then the ADR-0003 epic remains responsible for authoritative trust semantics and atomic generated known_hosts behavior; an Operation success does not independently bypass its connection gate.
- [ ] `US-005-AC1`: Given a new structured mutation, when it progresses, then its workflow exposes the applicable REQUESTED, VALIDATING, PLANNING, AWAITING_CONFIRMATION, RUNNING and VERIFYING stages rather than skipping directly from a click to success.
- [ ] `US-006-AC3`: Given any service action whose own Verifier or read-only Reconciliation predicate is missing, when I inspect or attempt that action, then it is unavailable with the missing evidence-policy reason even if connection and sudo permission are otherwise ready; another verb's policy does not enable it.
- [ ] `US-006-AC4`: Given an action with an evidence policy that does not cover the selected unit's semantics, when I inspect or attempt that action, then it is unavailable with that unsupported-Capability reason rather than silently using another unit type's evidence, interpreting active state as enablement evidence or assuming disable also stops the unit.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_planning
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/domain/resource_observation.rs src/infrastructure/systemd/mod.rs src/ui/operations.rs tests/operations_planning.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.

## Comments

E4:01 L1 and E4:02 S decisions are approved and remain separate from runtime acceptance. Consume the complete canonical L1/S case, proof, interruption and Reconciliation tables, not a generic success predicate. Exactly six service variants exist in EnabledMutation; LocalTrustApproval has exactly Accept, Replace and Delete. Each private case is selected only within the original OperationRequest, identity, plan revision, evidence, consent and lease context. No public property or dialog mints authority. File policy 03 remains needs-info after rejection of cooperative F. This whole ticket remains needs-info through its listed file-policy dependency chain. No file variant or download proof case is approved. All source criteria and first Proof owners are retained; unavailable controls do not complete implementation.
