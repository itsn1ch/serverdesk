# 01: Read saved inventory and resolve complete Connection Identities

**Status:** ready-for-agent
**Blocked by:** None (can start immediately).
**External blockers:** ADR-0006:01.
**Source:** `US-001-AC1`, `US-001-AC3`, `US-001-AC4`, `US-010-AC5`.
**Contract:** `.scratch/adr-0002-generated-ssh-config/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

A real saved Server can be inspected and resolved into its complete immutable endpoint/route/authentication identity without opening a connection. Storage failures remain actionable failures rather than empty inventory.

## Types

```rust
Configuration // from ADR-0006:01
InventorySnapshot // from ADR-0006:01
Server // from ADR-0006:01
AuthProfile // from ADR-0006:01
AuthMethod // from ADR-0006:01
AuthenticationContext // from ADR-0006:01
ConnectionEndpoint // from ADR-0006:01
ConnectionIdentity // from ADR-0006:01
ResolvedConnection // from ADR-0006:01
ConnectionEndpointView // from ADR-0006:01
ConnectionIdentityView // from ADR-0006:01
ResolvedConnectionView // from ADR-0006:01
InventoryStoreRequest // from ADR-0006:01
InventoryStoreResult // from ADR-0006:01
ConfigurationMessage // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. InventoryCommit and CompiledConfiguration fields/constructors remain private to their canonical owners.

## Functions

### `Configuration::spawn(store: E6::Store) -> Configuration`

- **Intent.** Establish one inventory/configuration owner over actual primary storage.
- **Where.** `src/application/configuration.rs`.
- **Pre.** Store is open, compatible, migrated and owned by the primary.
- **Post.** Creates one serviced owner mailbox; performs no import, SSH, trust lookup or invented empty read.
- **Errors.** Subsequent requests report actual StorageError; construction never substitutes an empty inventory.
- **Idempotency.** One composition per primary; construction does not rewrite storage.

### `Configuration::inventory(&self) -> Result<InventorySnapshot, E6::Failure>`

- **Intent.** Display saved Server and Auth Profile facts without connecting.
- **Where.** `src/application/configuration.rs`.
- **Pre.** Configuration addresses the compatible primary Store.
- **Post.** Returns one committed revision, including preserved environment discriminators; an empty snapshot follows only a successful empty query.
- **Errors.** StorageError leaves persisted facts unchanged and never becomes an empty-success view.
- **Idempotency.** Read-only; later calls may observe later committed revisions.

### `Configuration::resolve(&self, server_id: Uuid) -> Result<ResolvedConnection, E6::Failure>`

- **Intent.** Provide the complete saved structural identity before any trust or transport work.
- **Where.** `src/application/configuration.rs`.
- **Pre.** The external Server identifier exists; every saved hop/profile reference exists and its effective values validate.
- **Post.** Resolves the complete finite ordered route and non-secret authentication context; structural equality excludes display metadata; returns relevant revision without SSH, trust lookup or source evaluation.
- **Errors.** Missing reference, invalid field, cyclic/unregistered route, ConfigurationConflict or StorageError returns no resolution; no alias/local fallback.
- **Idempotency.** Unchanged effective content produces equal identity; original config edits cannot affect it.

### `ConnectionEndpoint::inspect_endpoint(&self) -> ConnectionEndpointView<'_>`

- **Intent.** Read the immutable effective endpoint and authentication references.
- **Where.** `src/domain/connection_identity.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns exact borrowed fields; cannot outlive the receiver, mutate it or construct an authority.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable read; receiver remains intact.

### `ConnectionIdentity::inspect_identity(&self) -> ConnectionIdentityView<'_>`

- **Intent.** Read the complete ordered identity without reconstructing it.
- **Where.** `src/domain/connection_identity.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns exact borrowed fields; cannot outlive the receiver, mutate it or construct an authority.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable read; receiver remains intact.

### `ResolvedConnection::inspect_resolution(&self) -> ResolvedConnectionView<'_>`

- **Intent.** Read the saved Server and revision binding of a resolution.
- **Where.** `src/domain/connection_identity.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns exact borrowed fields; cannot outlive the receiver, mutate it or construct an authority.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable read; receiver remains intact.

## Invariants

| ID    | Statement                                                                                                                                                                   | Role      | Proof                                                                          |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| INV-1 | Inventory reads return only actual committed SQLite snapshots and never perform SSH or turn storage failure into empty success.                                             | establish | Future test `configuration_inventory_real_store_failure`; first proof owner 01 |
| INV-2 | ConnectionIdentity includes every effective target/hop host, port, user and authentication context; metadata aliases cannot change equality and route resolution is finite. | establish | Future test `configuration_complete_identity_resolution`; first proof owner 01 |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Saved inventory and resolution: all steps; prerequisite inventory leg of direct compilation.

- In: `Store` ← external
- Out: `InventorySnapshot` → 03
- Out: `InventorySnapshot` → 05
- Out: `ResolvedConnection` → 04

External producer and gate details: ADR-0006:01 supplies actual Store::open/request and shared declarations. Saved rows are actual persisted external data, not existing application APIs. No other epic is assumed implemented.

## Steps

1. Write failing real-SQLite read/reopen/error tests for US-001-AC1 and US-010-AC5; implement spawn/inventory without connection work or fabricated empty state.
2. Write failing structural identity, alias-equality, complete per-hop authentication, missing/cyclic reference tests; implement resolve and the three immutable projections.
3. Preserve actual saved environment discriminators byte-for-byte. Classify exact E1 spellings without rewriting and expose nonmatching legacy metadata with a warning. It does not invalidate otherwise valid transport resolution. Report incompatible schema or unreadable values as StorageError; never fabricate missing/NULL data through the String observation.
4. Keep complete generated/master acceptance wired to 04/05 and downstream ADR-0001:02/03/04; this ticket proves the saved-input leg, not an existing transport.
5. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a saved Connection Identity, when its runtime configuration is generated, then the effective host, port, SSH user, complete Jump Server route and authentication context come from the saved inventory and referenced Auth Profile.
- [ ] `US-001-AC3`: Given two Server entries share a Connection Identity, when they are used, then configuration respects ADR-0001's single-master sharing boundary rather than treating inventory display names as separate authentication contexts.
- [ ] `US-001-AC4`: Given a different host, port, user, Jump Server route or authentication context is selected, when that identity is connected, then its configuration does not accidentally inherit the different identity's endpoint, route, key/agent reference or control socket.
- [ ] `US-010-AC5`: Given local storage or generated-configuration creation fails, when the workflow reports its outcome, then it identifies the affected inventory/Connection Identity, accurately states what was or was not saved, exposes the appropriate storage/configuration error and does not claim a usable configuration or successful connection.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test configuration_workflows configuration_inventory_real_store_failure
cargo test --test configuration_workflows configuration_complete_identity_resolution
rustfmt --edition 2024 --check src/domain/inventory.rs src/domain/connection_identity.rs src/application/configuration.rs src/ui/configuration.rs src/domain/mod.rs src/application/mod.rs src/ui/mod.rs tests/configuration_workflows.rs tests/configuration_gtk.rs src/main.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0002-generated-ssh-config/spec.md .scratch/adr-0002-generated-ssh-config/issues/00-overview.md .scratch/adr-0002-generated-ssh-config/issues/01-saved-inventory-resolution.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add recursive Include/re-import/diff, arbitrary OpenSSH evaluation, direct Store-vector writes, implicit Adoption or a second launch API.

## Comments

E1 is approved. Existing Server.environment_discriminator remains a legacy observation, not an open new-write enum. Configuration::resolve distinguishes legacy descriptive metadata from invalid effective connection fields. All new saves remain owned by03 and must validate through canonical ClosedEnvironment. No existing-row normalization or public field change is approved.
