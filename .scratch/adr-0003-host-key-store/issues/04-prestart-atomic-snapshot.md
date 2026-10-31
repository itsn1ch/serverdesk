# 04: Gate reserved starts and regenerate atomic authoritative trust snapshots

**Status:** needs-info
**Blocked by:** 01: Open canonical stored-key evidence and distribute one trust core, 02: Human policy decision, 03: Review local trust evidence, exact fingerprints and Adoption drift without acceptance.
**External blockers:** ADR-0006:01, ADR-0001:01.
**Source:** `US-006-AC1`, `US-006-AC2`, `US-006-AC3`, `US-006-AC4`, `US-008-AC1`, `US-008-AC2`, `US-008-AC3`, `US-008-AC4`, `US-008-AC5`, `US-009-AC2`, `US-011-AC2`, `US-011-AC3`.
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

A reserved start is blocked before transport on NONE, mismatch or unresolved policy, with the original accepted key preserved. Only approved eligibility plus a newly regenerated atomic SQLite-derived known_hosts can return the original resolution/reservation and matching positive TrustStart.

## Types

```rust
TrustPermission // from ADR-0006:01
TrustSnapshot // from ADR-0006:01
TrustStart // from ADR-0006:01
TrustStartView // from ADR-0006:01
PreparedStart // from ADR-0006:01
DiscoveryPath // from ADR-0006:01
TrustPending // from ADR-0006:01
TrustCompletion // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E2::Configuration // from ADR-0006:01
E2::ResolvedConnection // from ADR-0006:01
E1::StartReservation // from ADR-0006:01
E1::RouteAccess // from ADR-0006:01
E1::MasterGeneration // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. TrustEvidence and positive trust/prepared wrappers remain private; Store-created observations are immutable readable facts, not authority.

## Functions

### `TrustGate::prepare(&self, resolved: E2::ResolvedConnection, reservation: E1::StartReservation, discovery: DiscoveryPath) -> Result<PreparedStart, E6::Failure>`

- **Intent.** Authorize exactly one reserved master start against current accepted trust and a fresh atomic snapshot.
- **Where.** `src/application/trust.rs`.
- **Pre.** Resolution occurs first; identity/reservation/discovery path agree; Jump path, if any, names an already-permitted registered current generation; Direct/Jump eligibility policy has been approved.
- **Post.** Keyscan evidence is checked against accepted keys with ssh-keygen -F preflight before master launch. NONE, MISMATCH or unresolved eligibility produces no permission. On success atomically regenerates known_hosts from authoritative SQLite and returns PreparedStart containing the ORIGINAL resolved and reservation values plus matching private TrustStart/digest/start ID. Core delegates owned I/O workers and keeps its mailbox serviced.
- **Errors.** HostKeyUntrusted, HostKeyMismatch, unavailable policy, stale binding or storage/generation/atomic replacement failure yields no PreparedStart; preserves accepted key and forbids stale snapshot/user trust fallback. E1 retains attempt metadata to retire a failed consumed reservation without reconstructing authority. Classification is not human stderr or exit 255.
- **Idempotency.** Submit a reservation once. Success hands it back for one compilation; every startup/reconnect/new master requires a fresh reservation and regeneration. Generated-file edits never enter SQLite.

### `PreparedStart::into_prepared_start(self) -> (E2::ResolvedConnection, E1::StartReservation, TrustStart)`

- **Intent.** Return the original owned start inputs with their matching positive trust result.
- **Where.** `src/application/trust.rs`.
- **Pre.** A successful owner-created PreparedStart exists.
- **Post.** Moves the original resolution and reservation together with exactly their TrustStart; no scalar reconstruction, clone or dropped ownership.
- **Errors.** Infallible; no I/O.
- **Idempotency.** Single consumption; cannot authorize a second start.

### `TrustStart::inspect_trust_start(&self) -> TrustStartView<'_>`

- **Intent.** Supply compilation with the exact permission and authoritative snapshot binding.
- **Where.** `src/application/trust.rs`.
- **Pre.** A live TrustGate-created TrustStart exists.
- **Post.** Borrows matching start ID, identity, endpoint uses, store digest, generated path and generation time; a constructed view cannot stand in for TrustStart.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable borrowed read; wrapper remains non-cloneable.

## Invariants

| ID    | Statement                                                                                                                                                                               | Role      | Proof                                                                          |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | preserve  | Future test `trust_authoritative_store_migration`; first proof owner 01        |
| INV-2 | Review, drift, discovery, connectivity and generated-file edits never accept, replace, delete or promote authoritative Host Keys.                                                       | preserve  | Future test `trust_readonly_review_provenance`; first proof owner 03           |
| INV-3 | Verified evidence requires exact independent complete SHA256 comparison; Adoption and TOFU retain their distinct provenance and cannot imply Verified.                                  | preserve  | Future test `trust_exact_fingerprint_evidence`; first proof owner 03           |
| INV-4 | NONE, MISMATCH and unresolved Direct/Jump eligibility cannot produce TrustStart or authorize a master.                                                                                  | establish | Future test `trust_prestart_absent_mismatch_eligibility`; first proof owner 04 |
| INV-5 | Each positive start owns its original resolution/reservation and a fresh atomic SQLite-derived matching snapshot; failed regeneration never uses an old file.                           | establish | Future test `trust_atomic_snapshot_handback`; first proof owner 04             |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | preserve  | Future test `trust_single_core_ports`; first proof owner 01                    |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Master preparation: all gate/snapshot/handback steps; E2:04 and E1:02 consume outputs later, never prerequisites.

- In: `TrustGate` ← 01
- In: `TrustEvidence` ← 03
- In: `ResolvedConnection` ← external
- In: `StartReservation` ← external
- Out: `PreparedStart` → final

External producer and gate details: ADR-0006:01 declares shared records. ADR-0001:01 supplies start-data prerequisites; actual StartReservation is later minted by ADR-0001:02 and is handed back unchanged. Real preexisting accepted keys may populate isolated SQLite fixtures; no accepting a scan as trust. Local 03 supplies the sealed evidence-construction/comparison boundary; prepare performs FRESH scan/preflight, never reuses a stale TrustReview as authorization. Actual launch is downstream E2:04/E1:02.

## Steps

1. Consume approved 02 membership policy and update Design exactly; without it keep Direct/Jump use unavailable and do not choose a permissive default.
2. Write failing real keyscan/ssh-keygen -F preflight tests for NONE/mismatch and accepted-key preservation; implement prepare without human stderr or exit-255 classification.
3. Write failing per-start regeneration, tampered-file replacement and stale snapshot rejection tests with actual preexisting accepted SQLite keys; atomically generate only authoritative content.
4. Inject unreadable Store, rename/generation failures and disk-full exclusively on an isolated container filesystem; no positive result or fallback may escape.
5. Implement PreparedStart handback of the original reservation/resolution and inspect_trust_start; bind worker completions to current start ID/identity/digest while continuing mailbox service.
6. Test startup/reconnect use NEW reservations and regenerated snapshots; preserve GTK trust-failure/Connection State distinction and actionable errors. Actual target launch/all-consumer bypass proof is consumed in ADR-0001:02/03; this gate must not depend on that consumer.
7. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-006-AC1`: Given no accepted Host Key and Trust State NONE, when Connect or connect_on_startup would start a master, then connection is blocked as HostKeyUntrusted and the user is offered explicit trust acceptance instead of auto-acceptance.
- [ ] `US-006-AC2`: Given a stored Host Key and a different discovered Host Key, when the pre-start check runs using ssh-keygen -F against the keyscan result, then the mismatch blocks master start and is exposed as MISMATCH trust evidence and HostKeyMismatch/HOST_KEY_MISMATCH in the appropriate error and Connection State surfaces.
- [ ] `US-006-AC3`: Given a key mismatch, when the connection is rejected, then the previously accepted key is not silently overwritten and no terminal, SFTP, Operation, or Collector starts through a newly accepted replacement.
- [ ] `US-006-AC4`: Given a mismatch or trust failure, when the error is classified, then classification comes from the trust preflight rather than parsing human ssh stderr or assuming exit code 255 uniquely identifies the cause.
- [ ] `US-008-AC1`: Given any requested new master start, including startup connection and reconnect, when trust prerequisites are satisfied, then the application's known_hosts is regenerated atomically from SQLite before that master is allowed to start.
- [ ] `US-008-AC2`: Given a manually edited application-generated known_hosts, when the next master start is prepared, then regeneration replaces those edits with SQLite-derived content and does not import the edits into the Host Key Store.
- [ ] `US-008-AC3`: Given a generated trust file ready for master start, when system OpenSSH is launched, then only the application-generated file is used for Host Key trust; user/global known_hosts and dynamic known-host commands cannot supply an alternative acceptance path.
- [ ] `US-008-AC4`: Given generation or atomic replacement failure, including disk-full in an isolated test filesystem, when a master start is requested, then no partially generated file is treated as complete, no old snapshot is used as a fallback for that start, and the start fails closed with a storage/generation error.
- [ ] `US-008-AC5`: Given a successful earlier connection and a later failed regeneration, when another master is requested, then that earlier success does not waive regeneration or authorize the later start.
- [ ] `US-009-AC2`: Given an unreadable or otherwise unavailable authoritative Host Key Store, when trust is needed for a master start, then the start is blocked rather than falling back to user known_hosts, generated-file edits, or assumed trust.
- [ ] `US-011-AC2`: Given no stored key, unresolved trust, observed drift, or a failed trust workflow, when the relevant view is opened, then it presents the corresponding empty/unverified, stale-external-evidence warning, or error state without confusing it with Server downtime.
- [ ] `US-011-AC3`: Given an error, when its details are shown, then the UI identifies what failed, the affected Server/key, whether a change may have occurred, available recovery actions, and the related Operation details where applicable.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test trust_workflows trust_prestart_absent_mismatch_eligibility
cargo test --test trust_workflows trust_atomic_snapshot_handback
cargo test --test trust_gtk trust_snapshot_failure_gtk
rustfmt --edition 2024 --check src/domain/trust.rs src/application/trust.rs src/infrastructure/storage/host_keys.rs src/ui/trust.rs src/infrastructure/storage/mod.rs tests/trust_workflows.rs tests/trust_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/00-overview.md .scratch/adr-0003-host-key-store/issues/04-prestart-atomic-snapshot.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add automatic acceptance/promotion/synchronization, remote shell trust mutation, direct unaudited HostKey writes, fake capability constructors, or a second trust/audit owner.
