# 06: Inspect Services and bounded file-opening evidence without mutation privilege

**Status:** needs-info
**Blocked by:** 04 Inspect permanent audit and durably recover interrupted Operations
**External blockers:** ADR-0006:01, ADR-0001:03
**Source:** US-001-AC1, US-001-AC2, US-001-AC3, US-001-AC4, US-001-AC5, US-001-AC6, US-015-AC1, US-015-AC2
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`. Contract wins; stop and report a conflict.

## Goal

Use the actual existing-master E1 ConsumerAccess and ConsumerTarget::Server; bind once and reject replacement owner. Consume actual typed consumer event receivers; neither Connect nor a new transport is allowed.

Read-only behavior needs no service mutation policy. Its end-to-end transport verification remains blocked by the unresolved trust prerequisites of ADR-0001:03.

## Types

```rust
ResourceQuery // from ADR-0006:01
ResourceObservation // from ADR-0006:01
ServiceRead // from ADR-0006:01
UnitInfo // from ADR-0006:01
JournalRecord // from ADR-0006:01
ServiceSummary // from ADR-0006:01
TextDocument // from ADR-0006:01
FileVersion // from ADR-0006:01
TextOpenRequest // from ADR-0006:01
Store // from ADR-0006:01; actual Store issuer ADR-0006:01
Configuration // from ADR-0006:01; actual issuer ADR-0002:01
TrustExecutor // from ADR-0006:01; actual issuer ADR-0003:01
LockAuthority // from ADR-0006:01; actual issuer ADR-0006:01, arbitration ADR-0006:03
```

Foreign runtime inputs come from ADR-0006:01, ADR-0001:03. All referenced declarations are genuinely produced by ADR-0006:01, not preexisting application code. The declaration cohort implements only the five canonical infallible Store projections; every positive runtime capability still comes from its actual owner method.

## Functions

### `OperationEngine::bind_transport(&self, consumers: E1::ConsumerAccess) -> Result<(), E6::Failure>`

- **Intent.** Make real remote reads available after the connection owner exists.
- **Where.** `src/application/operations.rs`.
- **Pre.** Same primary generation; not previously bound to a different owner.
- **Post.** Remote requests use this E1 capability; no connection or mutation starts during binding.
- **Errors.** ConfigurationConflict for different/replaced binding leaves original intact.
- **Idempotency.** Rebinding the same capability is harmless; never creates a second transport.

### `OperationEngine::inspect_resources(&self, query: ResourceQuery) -> Result<ResourceObservation, E6::Failure>`

- **Intent.** Read required service details or file-opening evidence independently of mutation privileges.
- **Where.** `src/application/operations.rs`.
- **Pre.** Typed/bounded input, bound transport, current usable master and relevant read Capability.
- **Post.** Structured service mechanisms only; text is UTF-8 ≤1 MiB with mtime/size/hash snapshot; returned context stays explicit. Action availability is per verb/unit, exact permission and policy.
- **Errors.** Validation, permission, unavailable Capability/policy or transport failure is not empty/success; no privileged fallback.
- **Idempotency.** Repeats read current state without mutation; reads can remain usable while all mutations are unavailable.

## Invariants

| ID     | Statement                                                                                                     | Role      | Proof                                                                 |
| ------ | ------------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------- |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.          | preserve  | `missing_type_policy_blocks_every_action`; first proof 04             |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission. | preserve  | `interrupted_audit_recovery_is_durable_unknown`; first proof 04       |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                       | preserve  | `audit_is_permanent_append_only_and_sanitized`; first proof 04        |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.       | preserve  | `audit_payload_ordinals_tail_conflict_and_reload`; first proof 04     |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages. | preserve  | `operation_mailbox_progress_during_pending_work`; first proof 04      |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.  | establish | `services_and_text_reads_are_independent_and_bounded`; first proof 06 |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.   | preserve  | `operation_projection_privacy_and_exact_binding`; first proof 04      |

## Flow

Read resources flow3: existing E1 consumer → typed current read → actual observation.

- In: `OperationEngine` ← 04
- Out: `ResourceObservation` → 07
- In: `ResourceQuery` ← external
- Out: `ResourceObservation` → final

## Steps

1. Write the failing behavior proof for Use the actual existing-master E1 ConsumerAccess and ConsumerTarget::Server; bind once and reject replacement owner. Consume actual typed consumer event receivers; neither Connect nor a new transport is allowed. Then implement this boundary until that proof passes.
2. Write the failing behavior proof for Read Units/Summary with busctl --json=short on org.freedesktop.systemd1 without sudo; read properties/dependencies with systemctl show and bounded logs with journalctl -u <unit> -o json -n <N>. Missing list-units JSON does not disable these workflows. Then implement this boundary until that proof passes.
3. Write the failing behavior proof for Offer actual text-opening evidence only for the required edit boundary: reject binary or >1 MiB, hash the actual SFTP read, record full explicit Server/path/mtime/size/hash. Do not add a directory browser or privileged editor. Then implement this boundary until that proof passes.
4. Write the failing behavior proof for Present loading, actual successful empty, dated results and precise read/capability failure independently of per-verb mutation privilege. Preserve audit/recovery and pending-mailbox behavior. Then implement this boundary until that proof passes.
5. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a connected Server with the required Capability, when I open Services, then unit discovery uses `busctl --json=short` against `org.freedesktop.systemd1` without sudo rather than parsing a human-readable unit table.
- [ ] `US-001-AC2`: Given a discovered unit, when I inspect it, then structured properties come from `systemctl show -p <properties> [--value]` and expose unit name, description, ActiveState, SubState, Result and MainPID, with restart count and memory shown when available.
- [ ] `US-001-AC3`: Given access to a unit's journal, when I request recent logs, then the workflow obtains structured records with `journalctl -u <unit> -o json -n <N>` and displays them without human-table parsing.
- [ ] `US-001-AC4`: Given an inspectable unit, when I request its dependencies, then the Services workflow exposes its dependencies rather than offering only mutation controls.
- [ ] `US-001-AC5`: Given that `systemctl list-units` has no JSON mode, when the documented discovery and property mechanisms are available, then Services remains available.
- [ ] `US-001-AC6`: Given unavailable systemd, unavailable required busctl JSON support or denied read access, when I inspect the affected feature, then it is unavailable with its detected reason rather than silently empty or implemented through a privileged fallback.
- [ ] `US-015-AC1`: Given a Services or Operation view still fetching data, when I open it, then it visibly shows loading rather than an invented empty or successful state.
- [ ] `US-015-AC2`: Given a successful discovery with no units or an empty Operation history, when the relevant view is shown, then it presents a genuine empty state distinct from an error or pending request.
- [ ] Establish each invariant Proof listed here and keep preserved Proofs passing.
- [ ] Keep exact source criteria and all required enabled outcomes; missing policies remain explicit blockers rather than reduced scope.

## Verify

Future implementation gates: author these exact affected tests in this ticket; none are claimed present or passing now.

```sh
cargo test --test operations_reads
rustfmt --edition 2024 --check src/application/operations.rs src/domain/resource_observation.rs src/infrastructure/systemd/mod.rs src/infrastructure/sftp/mod.rs src/ui/operations.rs src/domain/mod.rs src/infrastructure/mod.rs tests/operations_reads.rs
```

Use isolated Incus/LXD Ubuntu 24.04 with systemd255, sshd, non-root user, granular sudoers and actual system OpenSSH/SQLite/SFTP; inspect actual resources and persisted evidence, not mocked SSH success. Actual GTK journeys cover this slice’s affected loading/empty/stale/error/Unknown/keyboard/responsiveness surfaces. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

Do not duplicate functions owned by another local/foreign introducing ticket. No generic proof fallback, invented human policy, second trust workflow, terminal-command audit, full file browser, privileged editing, remote lock, automatic mutation retry or audit deletion/reset. Early views and observations cannot mint authority; the actual scope data grants no permission.
