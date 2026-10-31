# 04: Generate and self-check isolated direct-start and attachment policy

**Status:** needs-info
**Blocked by:** 01: Read saved inventory and resolve complete Connection Identities.
**External blockers:** ADR-0006:01, ADR-0001:01, ADR-0003:04.
**Source:** `US-001-AC1`, `US-001-AC2`, `US-001-AC3`, `US-001-AC4`, `US-002-AC1`, `US-002-AC2`, `US-002-AC3`, `US-002-AC4`, `US-002-AC5`, `US-002-AC6`, `US-002-AC7`, `US-003-AC1`, `US-003-AC2`, `US-003-AC3`, `US-003-AC4`, `US-003-AC5`, `US-005-AC1`, `US-005-AC2`, `US-005-AC3`, `US-005-AC4`, `US-010-AC2`, `US-010-AC5`, `US-010-AC6`.
**Contract:** `.scratch/adr-0002-generated-ssh-config/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

A trust-gated original direct-start reservation yields the only self-checked isolated launch material, retaining its reservation and fresh trust result. Any generation/self-check/storage failure returns no launchable object; later E1 integration must exercise the actual master and every consumer.

## Types

```rust
CompiledConfiguration // from ADR-0006:01
CompiledLaunchView // from ADR-0006:01
ResolvedConnection // from ADR-0006:01
ConfigurationMessage // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E1::StartReservation // from ADR-0006:01
E1::MasterGeneration // from ADR-0006:01
E3::TrustStart // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. InventoryCommit and CompiledConfiguration fields/constructors remain private to their canonical owners.

## Functions

### `Configuration::compile(&self, resolved: ResolvedConnection, trust: E3::TrustStart, reservation: E1::StartReservation, prefixes: Vec<E1::MasterGeneration>) -> Result<CompiledConfiguration, E6::Failure>`

- **Intent.** Produce the only launchable generated configuration for one already-trust-gated reservation.
- **Where.** `src/application/configuration.rs`.
- **Pre.** Resolution precedes trust; trust and original reservation were returned together by PreparedStart; identity/start IDs agree; saved relevant revision remains current; prefixes exactly match the complete registered route.
- **Post.** Writes isolated generated start and fail-closed attachment aliases; forces StrictHostKeyChecking=yes, generated UserKnownHostsFile, GlobalKnownHostsFile=none, KnownHostsCommand=none, PermitLocalCommand=no, UpdateHostKeys=no, application-owned ControlMaster/ControlPath and ControlPersist=no. Runs ssh -G only on generated input, checks effective endpoint/authentication/route/security/control values, and returns a private wrapper retaining the original reservation/resolution/trust. No source directive map reaches OpenSSH.
- **Errors.** ConfigurationConflict, UnregisteredProxyRoute, ValidationError, generated-file/self-check failure or StorageError returns no launchable wrapper; no user-config, stale-file, unchecked or direct-route fallback. Errors expose affected identity and honest stale/change status.
- **Idempotency.** One reservation compiles once for one start; a new start needs new reservation and fresh trust snapshot. Crash residue is derived, never authoritative.

### `CompiledConfiguration::launch_view(&self) -> CompiledLaunchView<'_>`

- **Intent.** Let the transport launch worker inspect one sealed compilation while retaining its authority wrapper.
- **Where.** `src/application/configuration.rs`.
- **Pre.** A live Configuration-created CompiledConfiguration is held by the E1 private launch worker.
- **Post.** Borrows original reservation, resolution, trust, aliases and path; launching accepts the wrapper, never this view or arbitrary paths.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable borrowed read; does not duplicate a start.

## Invariants

| ID    | Statement                                                                                                                                                                   | Role      | Proof                                                                          |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| INV-1 | Inventory reads return only actual committed SQLite snapshots and never perform SSH or turn storage failure into empty success.                                             | preserve  | Future test `configuration_inventory_real_store_failure`; first proof owner 01 |
| INV-2 | ConnectionIdentity includes every effective target/hop host, port, user and authentication context; metadata aliases cannot change equality and route resolution is finite. | preserve  | Future test `configuration_complete_identity_resolution`; first proof owner 01 |
| INV-5 | A direct start/attachment is usable only from sealed CompiledConfiguration retaining its original reservation and fresh trust result under every forced security option.    | establish | Future test `configuration_direct_generated_policy`; first proof owner 04      |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Direct compilation and attachment: complete generation/validation leg; actual transport delivery is integrated by ADR-0001:02/03, not an external blocker creating a cycle.

- In: `ResolvedConnection` ← 01
- In: `TrustStart` ← external
- In: `StartReservation` ← external
- Out: `CompiledConfiguration` → 05
- Out: `CompiledConfiguration` → final

External producer and gate details: ADR-0006:01 declares shared data; ADR-0001:01 owns reservation/generation declaration/data prerequisites (actual reservations are later minted by ADR-0001:02). ADR-0003:04 is the actual TrustGate/PreparedStart producer; its eligibility policy is unresolved. Original reservation and resolution are handed back, never re-created. Actual E1:02/03 launch/consumer issuance is a downstream integration producer, not a hidden claim that it exists today.

## Steps

1. Write failing tests for the original reservation/resolution/trust handback, matching start/identity/digest and stale revision rejection; implement compile and launch_view using only private wrappers.
2. Write failing generated-only ssh -G checks for every forced option and effective endpoint/authentication/control binding; no source config evaluation is permitted.
3. Generate master-start and fail-closed attachment aliases; failure to attach cannot start an independent alternate connection.
4. Use actual OpenSSH/SQLite Incus fixtures and malicious user config/Include marker commands; exercise unknown/mismatched keys, conflicting sources and storage/generation failures without fallbacks.
5. Add application/GTK loading, affected-resource errors and visibly stale prior observations. Retain full master/terminal/SFTP/Operation/Collector integration journey in downstream ADR-0001:02/03, executing it once those real producers exist; no text-only test substitutes for it.
6. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a saved Connection Identity, when its runtime configuration is generated, then the effective host, port, SSH user, complete Jump Server route and authentication context come from the saved inventory and referenced Auth Profile.
- [ ] `US-001-AC2`: Given the Auth Profile selects an agent or key reference, when the configuration is generated, then the corresponding `IdentityAgent`/`IdentityFile` settings express those references without adding a password, passphrase or other secret to configuration or SQLite.
- [ ] `US-001-AC3`: Given two Server entries share a Connection Identity, when they are used, then configuration respects ADR-0001's single-master sharing boundary rather than treating inventory display names as separate authentication contexts.
- [ ] `US-001-AC4`: Given a different host, port, user, Jump Server route or authentication context is selected, when that identity is connected, then its configuration does not accidentally inherit the different identity's endpoint, route, key/agent reference or control socket.
- [ ] `US-002-AC1`: Given a master starts, when its effective configuration is inspected and exercised, then `StrictHostKeyChecking=yes` is enforced and an unaccepted or mismatched Host Key cannot be accepted implicitly by SSH.
- [ ] `US-002-AC2`: Given Host Key material has been generated by ADR-0003, when SSH checks Host Keys, then `UserKnownHostsFile` selects that generated file and `GlobalKnownHostsFile=none` excludes global Host Key files; user Host Key files are not runtime sources of trust.
- [ ] `US-002-AC3`: Given a connection starts, when its effective policy is exercised, then `KnownHostsCommand=none` and `PermitLocalCommand=no` prevent external Host Key command lookup and permitted local-command execution.
- [ ] `US-002-AC4`: Given a remote Server advertises additional Host Keys, when the connection is used, then `UpdateHostKeys=no` prevents SSH from silently changing the application's accepted Host Key material or Trust State.
- [ ] `US-002-AC5`: Given a Connection Identity is connected, when control options are resolved, then `ControlMaster` and `ControlPath` remain explicitly application-owned and identify the master/socket required by ADR-0001 rather than a master selected by user configuration.
- [ ] `US-002-AC6`: Given the application disconnects or terminates, when control persistence is evaluated, then `ControlPersist=no` is enforced and generated policy does not deliberately keep an independently persisting master alive.
- [ ] `US-002-AC7`: Given imported content requests different trust files, local commands, Host Key update behavior or control options, when the imported inventory is later connected, then the forced options above remain effective and the excluded source options do not override them.
- [ ] `US-003-AC1`: Given the application starts a master, when system OpenSSH is invoked, then `-F` selects generated configuration for that Connection Identity and `~/.ssh/config` is not read or evaluated at runtime.
- [ ] `US-003-AC2`: Given terminal, SFTP, Operation and Collector consumers use the connection, when they run, then they retain the generated configuration/control-master boundary rather than falling back to the user's SSH configuration or independently resolving the destination from it.
- [ ] `US-003-AC3`: Given the user's configuration contains an executable `Match exec`, `ProxyCommand`, `KnownHostsCommand` or `LocalCommand`, when application connection workflows run, then none of those source-file commands executes, whether it is in the directly named file or reachable through an Include tree.
- [ ] `US-003-AC4`: Given the original configuration is edited, removed or contains contradictory endpoint/authentication values after import, when the existing inventory snapshot is used for a connection, then those external changes do not alter its effective runtime configuration.
- [ ] `US-003-AC5`: Given a consumer cannot attach to the intended configured connection, when it reports failure, then it does not silently create an alternate connection governed by external configuration.
- [ ] `US-005-AC1`: Given the application uses `ssh -G`, when it performs a self-check, then its only configuration input is the application's generated configuration, never the original user configuration or an Include tree from it.
- [ ] `US-005-AC2`: Given a generated configuration self-check reveals a proxy route that is not registered in inventory, when the check completes, then connection is rejected with `UnregisteredProxyRoute` rather than treating `ssh -G` output as authorization to use that route.
- [ ] `US-005-AC3`: Given a configuration is imported, when its settings are inspected or resolved for import, then neither `ssh -G` nor another OpenSSH evaluation of the source file is used as the parser.
- [ ] `US-005-AC4`: Given generated configuration cannot be produced or its required checks fail, when connection is requested, then the workflow reports the configuration/storage failure and does not bypass the failure by reading the user's configuration or launching an unchecked fallback connection.
- [ ] `US-010-AC2`: Given source loading/parsing or configuration generation is in progress, when the Developer continues using the application, then the workflow exposes loading/progress without blocking GTK's main thread; presentation updates are delivered through `glib::MainContext` and Tokio never directly accesses GTK widgets.
- [ ] `US-010-AC5`: Given local storage or generated-configuration creation fails, when the workflow reports its outcome, then it identifies the affected inventory/Connection Identity, accurately states what was or was not saved, exposes the appropriate storage/configuration error and does not claim a usable configuration or successful connection.
- [ ] `US-010-AC6`: Given configuration failure affects an already presented workflow, when it is displayed, then the error identifies affected resources, whether changes may have occurred, available actions and applicable Operation details; any prior observation remains visibly stale rather than being presented as current success.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test configuration_workflows configuration_direct_generated_policy
cargo test --test configuration_workflows configuration_compilation_fail_closed
cargo test --test configuration_gtk configuration_generation_gtk_errors
rustfmt --edition 2024 --check src/domain/connection_identity.rs src/application/configuration.rs src/infrastructure/ssh/config.rs src/ui/configuration.rs src/infrastructure/ssh/mod.rs tests/configuration_workflows.rs tests/configuration_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0002-generated-ssh-config/spec.md .scratch/adr-0002-generated-ssh-config/issues/00-overview.md .scratch/adr-0002-generated-ssh-config/issues/04-isolated-direct-policy.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add recursive Include/re-import/diff, arbitrary OpenSSH evaluation, direct Store-vector writes, implicit Adoption or a second launch API.
