# 01: Clean actual orphan transport before admission

**Status:** ready-for-agent
**Blocked by:** None (can start immediately once external blockers land).
**External blockers:** ADR-0006:01.
**Source:** US-013-AC4.
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

A new primary examines real prior application sockets, exits old masters and removes only confirmed stale/terminated residue before admission. Unsafe or unresolved ownership stays visibly blocked; no old master is adopted.

External input producers: ADR-0006:01 Runtime::begin issues actual StartupOwnership/RuntimeDirectory after sole-primary/runtime-directory validation. All shared declarations are also from ADR-0006:01; cleanup_orphans here is the first actual CleanupReport issuer. Runtime::admit in ADR-0006:02 consumes this report after durable E4 recovery; it is not an already-existing API.

The two infallible borrowed generation/reservation projections also land here so ADR-0002:04 and ADR-0003:04 can compile before direct startup 02. They issue no concrete MasterGeneration or StartReservation; only 02 creates runtime bindings. Their final issued-state invariants are first exercised by actual startup in 02, not claimed established by an unused getter in 01.

## Types

```rust
CleanupReport // from ADR-0006:01
CleanupReportView // from ADR-0006:01
MasterGeneration // from ADR-0006:01
MasterGenerationView // from ADR-0006:01
StartReservation // from ADR-0006:01
StartReservationView // from ADR-0006:01
StartupOwnership // from ADR-0006:01
RuntimeDirectory // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `cleanup_orphans(startup: &E6::StartupOwnership, directory: &E6::RuntimeDirectory) -> Result<CleanupReport, E6::Failure>`

- **Intent.** Establish actual old-transport cleanup before primary admission.
- **Where.** `src/infrastructure/ssh/mod.rs`
- **Reads.** StartupOwnership, RuntimeDirectory, actual confined sockets.
- **Writes.** CleanupReport and old application-owned sockets.
- **Pre.** Current startup primary; directory UID, 0700 permissions and entry confinement validate.
- **Post.** Enumerates actual residue, sends isolated system ssh -O exit to each application socket, waits for confirmed termination or conclusive staleness, removes only those sockets; never adopts a master.
- **Invariants.** INV-11.
- **Errors.** Unsafe entry, uncertain live ownership, failed exit or removal remains unresolved and prevents admission; unrelated sockets remain untouched.
- **Idempotency.** Repeated cleanup of absent/stale entries is harmless; crash requires a fresh actual scan.

### `CleanupReport::inspect_cleanup(&self) -> CleanupReportView<'_>`

- **Intent.** Expose actual cleanup facts to Runtime without granting transport authority.
- **Where.** `src/infrastructure/ssh/mod.rs`
- **Reads.** CleanupReport.
- **Writes.** None.
- **Pre.** Report returned by cleanup_orphans.
- **Post.** Borrows startup generation, visited/removed sockets and unresolved failures unchanged.
- **Invariants.** INV-11.
- **Errors.** Infallible projection; no filesystem work.
- **Idempotency.** Repeatable read; no admission or cleanup is implied.

### `MasterGeneration::inspect_generation(&self) -> MasterGenerationView<'_>`

- **Intent.** Read an issued generation identity without issuing another authority.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** MasterGeneration.
- **Writes.** None.
- **Pre.** Manager-issued immutable generation.
- **Post.** Borrows exact identity and number; view cannot be accepted as launch or consumer authority.
- **Invariants.** INV-1, INV-2.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable; cloning issued generation never creates a new master.

### `StartReservation::inspect_reservation(&self) -> StartReservationView<'_>`

- **Intent.** Let configuration and trust inspect the original unique start binding.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** StartReservation.
- **Writes.** None.
- **Pre.** Owner-created live reservation.
- **Post.** Borrows exact master/start ID/control path; original reservation moves through trust and compilation, is never reconstructed from view.
- **Invariants.** INV-1, INV-6, INV-7.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeated reads do not duplicate or consume reservation; no Clone for reservation.

## Invariants

| ID     | Statement                                                                  | Role      | Proof                                                  |
| ------ | -------------------------------------------------------------------------- | --------- | ------------------------------------------------------ |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan. | establish | Future test `transport_orphan_cleanup`; first proof 01 |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Bootstrap cleanup; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `StartupOwnership` ← external
- In: `RuntimeDirectory` ← external
- Out: `CleanupReport` → 02

## Steps

1. Write the failing actual Incus/OpenSSH orphan-residue test, including responsive old master, conclusively stale socket, unsafe symlink/foreign UID/permissions and failed removal. Use only isolated runtime directories.
2. Implement cleanup_orphans and inspect_cleanup against validated current StartupOwnership/RuntimeDirectory; use real ssh -O exit and observed disappearance, not adoption or path-only deletion.
3. Connect actual CleanupReport to E6 admission failure/reporting; preserve empty actual scan success and repeatability, never fabricate successful recovery.
4. Implement inspect_generation and inspect_reservation as exact borrowed projections in the canonical connection manager module, without exposing constructors, cloning StartReservation or minting runtime generations. Actual issuance and corresponding runtime invariant proofs remain in 02.
5. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-013-AC4`: Given crash residue is encountered on a later startup, when ADR-0006 performs cleanup, then the connection manager participates in exiting old masters and removing old sockets and never adopts an orphan as a new live connection.
- [ ] Admission remains blocked if any actual orphan ownership/removal is unresolved; repeated cleanup never starts/adopts a master and never touches unrelated user sockets.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test transport_workflows transport_orphan_cleanup -- --exact
rustfmt --edition 2024 --check src/application/connection_manager.rs src/infrastructure/ssh/mod.rs tests/transport_workflows.rs tests/support/transport.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Public functions introduced by local tickets 02, 03, 04, 05; use them without adding duplicate APIs. Later private refinement of an earlier owned boundary is limited to this ticket's Goal.
- ADR-0002 inventory/import/generated policy, ADR-0003 trust eligibility/acceptance, ADR-0004 Verifier/Audit Trail/outcomes, ADR-0005 metric/backoff policy and ADR-0006 primary/admission/lease state remain with their canonical owners.
- Never add a second SSH library, arbitrary public launch/command API, privileged root shell, secret persistence, user-config runtime fallback, automatic mutation replay or orphan adoption.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
