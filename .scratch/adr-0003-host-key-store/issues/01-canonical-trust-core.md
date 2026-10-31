# 01: Open canonical stored-key evidence and distribute one trust core

**Status:** ready-for-agent
**Blocked by:** None (can start immediately).
**External blockers:** ADR-0006:01.
**Source:** `US-001-AC1`, `US-001-AC5`, `US-009-AC3`, `US-009-AC4`, `US-009-AC5`, `US-009-AC6`.
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Real compatible SQLite retains accepted Host Key public evidence and migration history under one canonical trust owner. Bootstrap obtains separate typed core/gate/executor handles without accepting a discovered key or needing transport.

## Types

```rust
HostKey // from ADR-0006:01
TrustState // from ADR-0006:01
TrustProvenance // from ADR-0006:01
TrustCore // from ADR-0006:01
TrustGate // from ADR-0006:01
TrustExecutor // from ADR-0006:01
TrustPorts // from ADR-0006:01
TrustStoreRequest // from ADR-0006:01
TrustMessage // from ADR-0006:01
TrustPending // from ADR-0006:01
TrustCompletion // from ADR-0006:01
TrustCommitObservation // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. TrustEvidence and positive trust/prepared wrappers remain private; Store-created observations are immutable readable facts, not authority.

## Functions

### `TrustCore::spawn(store: E6::Store) -> TrustPorts`

- **Intent.** Establish the one canonical trust owner before any transport exists.
- **Where.** `src/application/trust.rs`.
- **Pre.** Compatible migrated primary Store exists.
- **Post.** Returns core/gate/executor handles to one serviced mailbox; reads actual accepted host_keys fields through Store, preserving migrations/provenance without accepting or scanning anything.
- **Errors.** Actual read/migration failures surface as StorageError on use, never fabricated NONE or silent truncation.
- **Idempotency.** One trust owner per primary; construction does not rewrite stored keys.

### `TrustPorts::into_trust_ports(self) -> (TrustCore, TrustGate, TrustExecutor)`

- **Intent.** Distribute the bootstrap handles of the same trust actor without a second writer.
- **Where.** `src/application/trust.rs`.
- **Pre.** A live TrustPorts value from TrustCore::spawn exists.
- **Post.** Moves exactly its existing core, gate and executor handles; no actor or HostKey is created.
- **Errors.** Infallible; no I/O.
- **Idempotency.** Single consuming call; cannot consume the same value twice.

## Invariants

| ID    | Statement                                                                                                                                                                               | Role      | Proof                                                                   |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------- |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | establish | Future test `trust_authoritative_store_migration`; first proof owner 01 |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | establish | Future test `trust_single_core_ports`; first proof owner 01             |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Bootstrap and review: actual Store/core/ports steps only; later 03 adds review presentation.

- In: `Store` ← external
- Out: `TrustCore` → 03
- Out: `TrustGate` → 04
- Out: `TrustExecutor` → 05
- Out: `TrustExecutor` → 03

External producer and gate details: ADR-0006:01 supplies actual compatible Store and shared declarations; semantic HostKey ownership remains TrustCore. Fixture accepted keys are actual preexisting committed SQLite records, not discovery accepted by this ticket. No target master is assumed.

## Steps

1. Write failing real SQLite read/reopen/versioned migration tests with independently preexisting accepted fixture keys, preserving every accepted field and provenance; do not call fabricated empty storage a recovery result.
2. Implement TrustCore::spawn and consuming into_trust_ports to create exactly one mailbox owner with three purpose-restricted handles.
3. Add accepted host_keys UUID/FK checks without introducing unaccepted relationships or schema; report failed migration/incompatible/missing store as StorageError without truncation.
4. Check persisted public evidence and sanitized error payloads exclude secrets/private keys/file contents; do not add mutation or discovery acceptance.
5. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a successfully committed trust decision, when the application is reopened, then the Host Key Store retains its hostname, port, key algorithm, public key, SHA256 fingerprint, Trust State, source, verification time where applicable, and creation time using the accepted host_keys fields.
- [ ] `US-001-AC5`: Given two inventory Server entries with the same Connection Identity, when either entry uses stored trust, then both consult the authoritative SQLite Host Key Store rather than maintaining independent authoritative generated-file decisions.
- [ ] `US-009-AC3`: Given a versioned SQLite migration required for the Host Key Store, when it succeeds, then existing Host Keys and evidence remain available without silent truncation.
- [ ] `US-009-AC4`: Given a newly created Host Key record and the active storage session, when their accepted storage invariants are inspected, then its primary key is a UUID and SQLite foreign-key enforcement is enabled without adding any relationship absent the accepted schema.
- [ ] `US-009-AC5`: Given trust workflows and their Audit Trail, when persisted data is inspected, then the accepted public key/evidence fields do not introduce stored passwords, passphrases, private-key material, or file contents into SQLite or sanitized audit payloads.
- [ ] `US-009-AC6`: Given a required Host Key Store migration that fails, when affected trust workflows are requested, then StorageError is reported rather than proceeding with incompatible state or claiming a successful migration.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test trust_workflows trust_authoritative_store_migration
cargo test --test trust_workflows trust_single_core_ports
rustfmt --edition 2024 --check src/domain/trust.rs src/application/trust.rs src/infrastructure/storage/host_keys.rs src/domain/mod.rs src/application/mod.rs src/infrastructure/storage/mod.rs tests/trust_workflows.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/00-overview.md .scratch/adr-0003-host-key-store/issues/01-canonical-trust-core.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add automatic acceptance/promotion/synchronization, remote shell trust mutation, direct unaudited HostKey writes, fake capability constructors, or a second trust/audit owner.
