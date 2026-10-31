# 05: Review and admit local trust Operations before effects

**Status:** needs-info
**Blocked by:** 01 Approve local trust lifecycle and evidence policy, 04 Inspect permanent audit and durably recover interrupted Operations
**External blockers:** ADR-0006:01, ADR-0003:03, ADR-0006:03
**Source:** US-013-AC1, US-013-AC2, US-013-AC3, US-013-AC4, US-004-AC1, US-004-AC3, US-004-AC5, US-005-AC1, US-005-AC2, US-005-AC3, US-005-AC4, US-005-AC5
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

After actual audit04 and approved human local policy01, implement prepare_local_trust using Configuration resolution and the pure E3:03 prepare_change to validate actual current stored keys and produce one sealed LocalTrustPlan. No SSH, remote read, service policy or E3 commit prerequisite is allowed.

Dependent enabled implementation remains needs-info until blocking human policies are approved. The verbatim source criteria retain the full required behavior; unavailable controls are not represented as completed mutation delivery.

## Types

```rust
LocalTrustPlan // from ADR-0006:01
LocalTrustAdmitted // from ADR-0006:01
OperationRequest // from ADR-0006:01
OperationPlan // from ADR-0006:01
OperationDecision // from ADR-0006:01
Confirmation // from ADR-0006:01
ResourceScope // from ADR-0006:01
LocalTrustEnvelope // from ADR-0006:01
LocalTrustEnvelopeView // from ADR-0006:01
AuditTail // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
PreparedTrustChange // from ADR-0006:01; actual semantic producer ADR-0003:03
```

Foreign runtime inputs come from ADR-0006:01, ADR-0003:03, ADR-0006:03. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method. LocalTrustPlan/admission/finish stages are E4 workflow boundaries; only the registered E4 commit worker invokes the restricted finisher. E3:05 later installs the actual E3 commit invocation and preserves the already-implemented completion path; before that, admission is real but no completed mutation is claimed.

## Functions

### `OperationEngine::prepare_local_trust(&self, request: OperationRequest) -> Result<LocalTrustPlan, E6::Failure>`

- **Intent.** Review the exact stored-state local trust change without requiring remote transport.
- **Where.** `src/application/operations.rs`.
- **Pre.** Explicit Trust request; accepted E4:01 local specialization and E3 semantic validation; no remote prerequisites.
- **Post.** Use actual stored keys/preconditions and E3 prepare_change to persist Requested/Validating/Planning/AwaitingConfirmation and return sealed exact plan/prepared delta. Present Server/Host Key identity, impact and all applicable acknowledgements. Missing policy blocks without admission.
- **Errors.** Validation, unsupported policy, stored-key/revision conflict or Store failure admits no change; preserve already durable facts.
- **Idempotency.** One explicit request creates one local Operation; no blind redelivery or remote action.

### `LocalTrustPlan::inspect_local_trust_plan(&self) -> &OperationPlan`

- **Intent.** Present the sealed local trust plan without exposing its authority constructor.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live engine-created LocalTrustPlan exists.
- **Post.** Borrow the exact plan; no prepared delta/envelope is fabricated or cloned.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable, receiver remains owned and borrowed view cannot outlive it.

### `OperationEngine::admit_local_trust(&self, plan: LocalTrustPlan, decision: OperationDecision) -> Result<LocalTrustAdmitted, E6::Failure>`

- **Intent.** Admit one exact confirmed local trust change after real lease and durable write-ahead.
- **Where.** `src/application/operations.rs`.
- **Pre.** Approved local trust policy, current exact revision, complete consent and Ready lock admission; same current stored preconditions.
- **Post.** Acquire the real identity/scopes lease, commit Running before key effects, reserve exact next committed-change event UUID/ordinal and return original PreparedTrustChange paired with LocalTrustEnvelope. Engine retains the lease/active local-work record; no trust write happens here.
- **Errors.** Reject/missing consent/stale revision/key conflict/lease conflict/write-ahead failure returns no envelope and no key change; settle any acquired ownership honestly.
- **Idempotency.** A revision is consumed at most once; ambiguous delivery requires actual audit read, never duplicate admission.

### `LocalTrustAdmitted::into_local_trust_admitted(self) -> (E3::PreparedTrustChange, LocalTrustEnvelope)`

- **Intent.** Move the original prepared local delta and its paired admission to the trust executor.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live engine-created admitted value exists.
- **Post.** Move the exact PreparedTrustChange and LocalTrustEnvelope once; engine retains its actual lease until settled completion.
- **Errors.** Infallible; no authority allocation.
- **Idempotency.** Single consumption; the moved receiver cannot be used twice.

### `OperationEngine::finish_local_trust(&self, operation_id: Uuid, result: Result<E3::TrustCommitObservation, E6::Failure>) -> Result<Operation, E6::Failure>`

- **Intent.** Settle one registered local trust commit after actual worker join and release its retained lease.
- **Where.** `src/application/operations.rs`.
- **Pre.** Only the registered E4-owned local commit worker may invoke after the actual commit future/local work join; operation ID matches retained active admission.
- **Post.** Validate actual tracked complete work set and Store settlement; evaluate approved local persisted-state proof. Sufficient evidence yields Succeeded/Failed; insufficient evidence yields actual Unknown, never assumed no effects. After settled conservative persistence, create WorkJoined and release retained lease through the still-serviced mailbox; no local active lease is abandoned.
- **Errors.** Caller-supplied Error is not proof of join. Missing/mismatched worker, unjoined work or unsettled Store returns incomplete ownership without release; ambiguous commit is reread from actual Store and stays Unknown unless proved.
- **Idempotency.** Duplicate matching completion returns retained disposition; cannot release another lease or execute/commit again.

## Invariants

| ID     | Statement                                                                                                                                           | Role      | Proof                                                             |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | preserve  | `missing_type_policy_blocks_every_action`; first proof 04         |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | establish | `confirmed_revision_write_ahead_single_dispatch`; first proof 05  |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04   |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04    |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04 |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04  |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04  |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | establish | `local_trust_commit_joined_outcome_and_release`; first proof 05   |

## Flow

Local trust flow2: prepare → review → exact consent → real lease/write-ahead → consume paired admission; registered actual commit completion → private finish → joined release.

- In: `OperationEngine` ← 04
- Out: `LocalTrustPlan` → 07
- In: `OperationDecision` ← external
- Out: `LocalTrustAdmitted` → final

## Steps

1. Write the failing behavior proof for After actual audit04 and approved human local policy01, implement prepare_local_trust using Configuration resolution and the pure E3:03 prepare_change to validate actual current stored keys and produce one sealed LocalTrustPlan. No SSH, remote read, service policy or E3 commit prerequisite is allowed. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Present exact affected Server/Host Key identity, action, impact and revision through inspect_local_trust_plan; require applicable requested-change/destructive/unmanaged/replacement/deletion/TOFU consent from that same revision. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Implement admit_local_trust with actual E6:03 identity/resource lease acquisition and fresh stored preconditions. Commit Running before any trust change, retain the lease in the E4 active-work map, and create exactly one LocalTrustAdmitted containing the original prepared delta and sealed envelope with actual tail/paired UUID/ordinal. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Consume admitted parts only through into_local_trust_admitted, supplying E3:05 commit with the exact original delta/envelope. This ticket demonstrates genuine local SQLite/consent/write-ahead admission before transport; it does not invent a trust commit result or claim full mutation completion before the real executor exists. Then implement this boundary until that proof passes.
5. Write the failing behavior proof for Use the E4-owned registered commit worker to call actual E3 commit once it exists, then invoke the restricted finish_local_trust only after the commit future and actual local work join. Validate work IDs/operation identity and actual persisted tail/read-back; caller Error alone never proves work ended. Settle approved local proof or Unknown, then create WorkJoined and acknowledge real lease release. The early local admission proof is available before E3:05; its completed backend journey is preserved by E3:05 actual implementation without adding a backward blocker. Then implement this boundary until that proof passes.
6. Write the failing behavior proof for Later public plan_operation/decide_operation delegate these SAME canonical stages rather than implementing a second trust workflow or policy owner. Missing local policy remains unavailable; stale consent/key assumptions, conflicts and failed write-ahead never issue positive admission. Then implement this boundary until that proof passes.
7. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-013-AC1`: Given a requested Host Key trust change, when it is applied, then it participates in the Operation engine and sanitized Audit Trail rather than writing around them.
- [ ] `US-013-AC2`: Given replacement or deletion of a trust entry, when I proceed, then explicit consent and the affected Server and Host Key identity are part of the request rather than automatic acceptance of observed drift.
- [ ] `US-013-AC3`: Given a trust mutation type without its own verifiable persisted-state outcome and Reconciliation rule, when the action is considered for enablement, then it remains unavailable under the same evidence prerequisite; no remote shell or remote-reconnect prerequisite is fabricated for a local SQLite trust commit.
- [ ] `US-013-AC4`: Given an accepted trust mutation, when its outcome is consumed by connection handling, then the ADR-0003 epic remains responsible for authoritative trust semantics and atomic generated known_hosts behavior; an Operation success does not independently bypass its connection gate.
- [ ] `US-004-AC1`: Given a structured request, when planning presents it, then its Server context, affected resource, action and classification are explicit rather than inferred from an unrelated currently selected Server.
- [ ] `US-004-AC3`: Given a destructive mutation, when execution is requested, then it cannot proceed until I explicitly confirm the described impact.
- [ ] `US-004-AC5`: Given a remote resource that is not a Managed Resource, when a workflow proposes changing it, then it requires my explicit consent rather than assuming application ownership.
- [ ] `US-005-AC1`: Given a new structured mutation, when it progresses, then its workflow exposes the applicable REQUESTED, VALIDATING, PLANNING, AWAITING_CONFIRMATION, RUNNING and VERIFYING stages rather than skipping directly from a click to success.
- [ ] `US-005-AC2`: Given a validated and authorized plan, when execution is about to start, then the Operation's RUNNING state is committed in a SQLite transaction before remote execution begins.
- [ ] `US-005-AC3`: Given that the write-ahead transaction fails, when execution is considered, then no remote mutation is started and a StorageError explains that execution was prevented.
- [ ] `US-005-AC4`: Given a persisted Operation, when I inspect its record, then the accepted Operation metadata identifies its Server, resource, action, sanitized parameters, classification, status, timing, verification, sanitized error and correlation identifier as applicable to its stage.
- [ ] `US-005-AC5`: Given an Operation requiring multiple related local persistence changes, when those changes are committed, then transactions and active foreign keys preserve their consistency and no storage value is silently truncated.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_local_trust
rustfmt --edition 2024 --check src/application/operations.rs src/domain/operation.rs src/infrastructure/storage/audit.rs src/ui/operations.rs tests/operations_local_trust.rs
```

Use actual SQLite/Configuration/TrustCore semantic preparation and Runtime identity/resource admission. Assert exact consent/revision/write-ahead, no key effect before commit, pending ownership and invalid worker result refusal; preserve the full actual E3:05 commit/join/lease-release journey once its real executor exists. No fabricated positive envelope/result is acceptable. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.
