# 05: Generate only registered complete Jump Server routes

**Status:** needs-info
**Blocked by:** 01: Read saved inventory and resolve complete Connection Identities, 03: Save reference-only settings and import an explicit static whitelist, 04: Generate and self-check isolated direct-start and attachment policy.
**External blockers:** ADR-0006:01.
**Source:** `US-004-AC1`, `US-004-AC2`, `US-004-AC3`, `US-004-AC4`, `US-004-AC5`, `US-007-AC3`, `US-005-AC2`.
**Contract:** `.scratch/adr-0002-generated-ssh-config/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Saved and imported multi-hop routes generate exactly the registered Jump Servers with matching gated prefix generations. An unresolved alias, omitted hop, unexpected self-check proxy or failed Jump trust never degrades into a direct or external-config connection.

## Types

```rust
ConnectionIdentity // from ADR-0006:01
ResolvedConnection // from ADR-0006:01
CompiledConfiguration // from ADR-0006:01
ImportReport // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E1::StartReservation // from ADR-0006:01
E1::MasterGeneration // from ADR-0006:01
E3::TrustStart // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. InventoryCommit and CompiledConfiguration fields/constructors remain private to their canonical owners.

## Functions

No new public functions. Privately refine `Configuration::compile` owned by 04; do not duplicate its signature or add a second route-generation entry point.

## Invariants

| ID    | Statement                                                                                                                                                                   | Role      | Proof                                                                          |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| INV-1 | Inventory reads return only actual committed SQLite snapshots and never perform SSH or turn storage failure into empty success.                                             | preserve  | Future test `configuration_inventory_real_store_failure`; first proof owner 01 |
| INV-2 | ConnectionIdentity includes every effective target/hop host, port, user and authentication context; metadata aliases cannot change equality and route resolution is finite. | preserve  | Future test `configuration_complete_identity_resolution`; first proof owner 01 |
| INV-3 | Only Configuration-created InventoryCommit can persist validated reference-only changes, atomically at its expected revision and without secrets.                           | preserve  | Future test `configuration_atomic_reference_commit`; first proof owner 03      |
| INV-4 | Import never evaluates commands, reads Include targets, rewrites originals, synchronizes snapshots or accepts Host Keys.                                                    | preserve  | Future test `configuration_static_import_isolation`; first proof owner 03      |
| INV-5 | A direct start/attachment is usable only from sealed CompiledConfiguration retaining its original reservation and fresh trust result under every forced security option.    | preserve  | Future test `configuration_direct_generated_policy`; first proof owner 04      |
| INV-6 | Generated routed policy represents exactly the complete registered inventory route through matching gated prefix generations, never an external alias or direct fallback.   | establish | Future test `configuration_registered_route_policy`; first proof owner 05      |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Full registered route: all private route refinement steps; downstream ADR-0001:04 consumes this policy for actual routed transport.

- In: `InventorySnapshot` ← 01
- In: `ImportReport` ← 03
- In: `CompiledConfiguration` ← 04
- Out: `CompiledConfiguration` → final

External producer and gate details: ADR-0006:01 supplies declarations only. Actual runtime prefix generation/RouteAccess values are minted by downstream ADR-0001:04 over its earlier direct manager; the generator can validate supplied generation bindings without initiating Connect. E1:04 consumes this completed generator, so it is NOT an external blocker. The actual multi-hop acceptance journey is integration-owned there.

## Steps

1. Write failing direct/single/multi-hop route tests using accepted saved/imported inventory and the 04 generator; preserve exact compile signature and ownership.
2. Privately deepen compile to map each hop to the registered complete structural route and matching prefix generation; never resolve a source alias or emit command-backed proxies.
3. Use generated-only ssh -G to reject extra/missing/unregistered hops as UnregisteredProxyRoute, with no silent omission/direct fallback.
4. Retain trust gate for every prefix; add real actual route/destination integration acceptance to ADR-0001:04, which consumes this generator rather than blocking its implementation.
5. Show target/route-specific keyboard-accessible actionable errors through the same GTK path and preserve stale observations.
6. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-004-AC1`: Given a direct or multi-hop connection route is saved, when generated configuration represents it, then each Jump Server comes from registered inventory and the complete route contributes to Connection Identity.
- [ ] `US-004-AC2`: Given a ProxyJump value is imported, when it becomes usable connection configuration, then it is represented as a route through Jump Servers in inventory rather than an opaque user-config alias resolved at runtime.
- [ ] `US-004-AC3`: Given a route cannot be represented using registered Jump Servers, when Connect or the generated-configuration self-check encounters it, then it is rejected as `UnregisteredProxyRoute` rather than executed as an external command or silently omitted to make a direct connection.
- [ ] `US-004-AC4`: Given a registered route is otherwise usable but a Jump Server fails the ADR-0003 trust gate, when connection is requested, then configuration registration alone does not authorize its use and the connection remains blocked by that gate.
- [ ] `US-004-AC5`: Given a usable registered route, when actual connection traffic is exercised, then it reaches the configured target through the registered Jump Servers, with no extra route introduced by the user's SSH configuration.
- [ ] `US-007-AC3`: Given a statically supported ProxyJump setting, when import succeeds, then it contributes the corresponding registered inventory Jump Server route under US-004 rather than a runtime dependency on a source-file alias.
- [ ] `US-005-AC2`: Given a generated configuration self-check reveals a proxy route that is not registered in inventory, when the check completes, then connection is rejected with `UnregisteredProxyRoute` rather than treating `ssh -G` output as authorization to use that route.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test configuration_workflows configuration_registered_route_policy
cargo test --test configuration_workflows configuration_proxyjump_static_import
rustfmt --edition 2024 --check src/domain/connection_identity.rs src/application/configuration.rs src/infrastructure/ssh/config.rs src/ui/configuration.rs tests/configuration_workflows.rs tests/configuration_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0002-generated-ssh-config/spec.md .scratch/adr-0002-generated-ssh-config/issues/00-overview.md .scratch/adr-0002-generated-ssh-config/issues/05-registered-jump-policy.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add recursive Include/re-import/diff, arbitrary OpenSSH evaluation, direct Store-vector writes, implicit Adoption or a second launch API.
