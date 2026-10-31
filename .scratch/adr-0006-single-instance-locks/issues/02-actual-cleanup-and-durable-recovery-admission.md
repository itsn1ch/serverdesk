# 02: Admit only after actual orphan cleanup and durable Unknown recovery

**Status:** ready-for-agent
**Blocked by:** 01 Establish sole primary, compatible SQLite and preadmission runtime
**External blockers:** ADR-0001:01, ADR-0002:01, ADR-0003:01, ADR-0004:04
**Source:** US-001-AC4, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-008-AC5, US-008-AC6, US-009-AC1, US-009-AC2, US-009-AC3, US-009-AC4, US-009-AC5, US-009-AC6, US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4, US-010-AC1, US-010-AC7
**Contract:** `.scratch/adr-0006-single-instance-locks/spec.md`, section `# Design`; the five imported Store projections retain their canonical E2/E3/E4/E5 full contracts and invariant namespaces. Contract wins; stop and report a conflict.

## Goal

Consume actual Store from01 and actual lower foreign Configuration/TrustCore/executor construction. Create the actual E4 audit owner without remote transport or a mutation policy prerequisite.

## Types

```rust
CleanupReport // from 01
RecoveryReport // from 01
ReadyOwnership // from 01
OwnershipView // from 01
StartupSnapshot // from 01
StartupChanged // from 01
```

01 is the genuine declaration producer in canonical modules, and implements only the five pure Store projections. Actual report/capability issuers are the External blockers and earlier owner methods, never declarations or observation DTOs. ReadyOwnership is Clone only for the SAME admitted generation through Arc<RuntimeDirectory>; StartupOwnership and ResourceLease remain private and non-Clone.

## Functions

### `Runtime::admit(&self, cleanup: E1::CleanupReport, recovery: E4::RecoveryReport) -> Result<ReadyOwnership, Failure>`

- **Intent.** Open managed admission only after actual current-generation safety results exist.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Both reports match startup generation; cleanup has no unresolved socket/ownership failure; E4 recovery transactions actually committed.
- **Post.** Ready is published; unresolved Unknown Operations remain visible; ReadyOwnership is minted for this generation only.
- **Errors.** Invalid/incomplete report yields blocked ConfigurationConflict/StorageError with no Ready capability.
- **Idempotency.** Same successful admission returns same-generation authority; cannot admit from older reports.

### `ReadyOwnership::inspect_ready_ownership(&self) -> OwnershipView<'_>`

- **Intent.** Read the generation whose startup safety checks completed.
- **Where.** `src/application/runtime.rs`.
- **Pre.** A live owner-issued receiver exists.
- **Post.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; same-generation Ready clones share the existing Arc directory; references cannot outlive receiver.

## Invariants

| ID    | Statement                                                                                                                  | Role      | Proof                                                                |
| ----- | -------------------------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------- |
| INV-1 | Only the current GtkApplication primary opens operational Store and mints current-generation ownership capabilities.       | preserve  | `single_primary_real_store_and_preadmission`; first proof 01         |
| INV-4 | Ready follows actual orphan non-adoption and durable interrupted Running/Verifying to Unknown recovery.                    | establish | `actual_orphans_and_durable_unknown_before_admit`; first proof 02    |
| INV-5 | GTK owns widgets and remains responsive with truthful ownership/recovery/shutdown states.                                  | preserve  | `runtime_gtk_preadmission_and_error_states`; first proof 01          |
| INV-8 | Store accepts only sealed canonical-owner writes; immutable projections expose exact data and never authorize a new write. | preserve  | `store_sealed_owner_requests_and_projection_privacy`; first proof 01 |

## Flow

Design flow 2: actual confined cleanup and durable interrupted recovery → matching reports → admit.

- In: `StartupOwnership` ← 01
- In: `Store` ← 01
- Out: `ReadyOwnership` → 03
- Out: `ReadyOwnership` → 04
- In: `CleanupReport` ← external
- Out: `StartupSnapshot` → final

Actual lower foreign constructors/reports come from ADR-0001:01, ADR-0002:01, ADR-0003:01 and ADR-0004:04. Neither E4:05 local policy nor an ordinary connected master is a prerequisite to startup audit recovery.

## Steps

1. Write the failing behavior proof for Consume actual Store from01 and actual lower foreign Configuration/TrustCore/executor construction. Create the actual E4 audit owner without remote transport or a mutation policy prerequisite. Then implement until that proof passes.
2. Write the failing behavior proof for Pass current StartupOwnership and confined RuntimeDirectory to actual E1 cleanup_orphans. Observe exit request for each leftover application socket and safe removal, never adoption or cleanup of unrelated user SSH connections; responsive master and dead stale socket are distinct cases. Then implement until that proof passes.
3. Write the failing behavior proof for Pass the same startup authority to E4 recover_interrupted. Read actual persisted Running/Verifying audit and transactionally append Unknown/reconciliation-needed while retaining original request/evidence. Failed read/migration/commit does not produce a successful report. Then implement until that proof passes.
4. Write the failing behavior proof for Implement admit by inspecting both real reports and matching current startup generation; only complete cleanup plus durable recovery can mint ReadyOwnership. Missing/incomplete/old reports fail closed, including connect_on_startup; no empty backend or fixture readiness is permitted. Then implement until that proof passes.
5. Write the failing behavior proof for Render actual recovery loading/empty/Unknown/blockers and actionable resource-specific details with keyboard access; Unknown evidence is not Server downtime. The prefix exposes no managed Connect/mutation before their actual owner methods exist. Then implement until that proof passes.
6. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-001-AC4`: Given that the previous application owner has exited, when a later launch becomes the owner, then it performs the startup recovery workflow rather than assuming the previous in-memory locks or master ownership survived.
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
- [ ] `US-011-AC1`: Given startup cleanup, recovery persistence, preflight, or Reconciliation is running, when I interact with GTK, then the main thread remains responsive and affected work has a loading/recovery presentation rather than an unsupported success claim.
- [ ] `US-011-AC2`: Given no interrupted Operations, when recovery-related results are shown, then an honest empty state is available; given unresolved Operations, then UNKNOWN and unavailable or stale evidence remain visible rather than being presented as Server downtime.
- [ ] `US-011-AC3`: Given a cleanup failure, storage failure, conflict, or unresolved outcome, when the error is displayed, then it identifies what failed, the affected Server/resource, whether a change may have occurred, available actions, and related Operation details where applicable.
- [ ] `US-011-AC4`: Given recovery details or a rejected-conflict surface, when I use keyboard navigation, then I can reach the details and available actions without a pointer or a forced automatic mutation retry.
- [ ] `US-010-AC1`: Given an UNKNOWN interrupted Operation requiring Reconciliation, when the affected Server has not reconnected, then the record remains unresolved and startup does not replay its mutation to obtain an answer.
- [ ] `US-010-AC7`: Given a proposed Operation type without its own Verifier and Reconciliation predicate, when its availability is evaluated, then startup recovery does not supply a generic success fallback and the shared engine's prerequisite for enabling that mutation remains in force.
- [ ] Establish each invariant Proof listed here and preserve earlier Proofs. Full runtime integration criteria are exercised again by04 with actual domain issuers, not forged early capabilities.
- [ ] No declaration-only owner, no-op backend, empty Store fallback or unsupported Ready/completed-outcome claim is introduced.

## Verify

Future implementation gates: author these exact affected tests/harnesses here; none are claimed present or passing now.

```sh
cargo test --test runtime_recovery_admission
rustfmt --edition 2024 --check src/application/runtime.rs src/infrastructure/storage/mod.rs src/main.rs src/ui/runtime.rs tests/runtime_recovery_admission.rs
```

Use actual confined system OpenSSH responsive orphan and dead stale socket cleanup plus SQLite persisted Running/Verifying interruptions and failed recovery transactions; show actual loading/empty/Unknown/blockers and keyboard details. Actual GTK surfaces must remain responsive and keyboard accessible; errors/Unknown/stale state remain honest. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

No foreign semantic writer transfer, future owner constructor, generic proof fallback, remote/distributed lock, persisted lease, new scope hierarchy, hidden mutation queue, automatic mutation replay, schema/audit reset/deletion, secret/file-content persistence or permanent greeting/wiring/text tests. Only04 owns final production launch composition.

## Comments

L1 is approved but does not weaken startup admission. Every interrupted Running/Verifying Operation, local or remote, first becomes durable Unknown with retained evidence. This recovery report proves interruption processing only, not any mutation outcome. Later local trust Reconciliation uses E4 L1 after actual worker closure and settled sole-primary SQLite without SSH/reconnect; remote cases retain their current gated reconnect requirement. File policy remains independently unapproved.
