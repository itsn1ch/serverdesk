# 03: Save reference-only settings and import an explicit static whitelist

**Status:** ready-for-agent
**Blocked by:** 01: Read saved inventory and resolve complete Connection Identities, 02: Human policy decision.
**External blockers:** ADR-0006:01.
**Source:** `US-006-AC1`, `US-006-AC2`, `US-006-AC3`, `US-006-AC4`, `US-006-AC5`, `US-007-AC1`, `US-007-AC2`, `US-007-AC3`, `US-007-AC4`, `US-007-AC5`, `US-007-AC6`, `US-008-AC1`, `US-008-AC2`, `US-008-AC3`, `US-008-AC4`, `US-008-AC5`, `US-008-AC6`, `US-009-AC1`, `US-009-AC2`, `US-009-AC3`, `US-009-AC4`, `US-010-AC1`, `US-010-AC2`, `US-010-AC3`, `US-010-AC4`, `US-010-AC5`, `US-010-AC6`.
**Contract:** `.scratch/adr-0002-generated-ssh-config/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Implement approved E1 while saving non-secret references and explicitly importing supported settings into an independent durable inventory snapshot. Enforce exact closed environment validation and byte-preserving legacy reads; exclusions are not misreported as saved Servers.

## Types

```rust
ConfigurationChange // from ADR-0006:01
ConfigurationResult // from ADR-0006:01
ServerDraft // from ADR-0006:01
ClosedEnvironment // from ADR-0006:01
AuthProfileDraft // from ADR-0006:01
Server // from ADR-0006:01
AuthProfile // from ADR-0006:01
ImportReport // from ADR-0006:01
ImportExclusion // from ADR-0006:01
InventoryCommit // from ADR-0006:01
InventoryWriteView // from ADR-0006:01
InventoryStoreRequest // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. InventoryCommit and CompiledConfiguration fields/constructors remain private to their canonical owners.

## Functions

### `Configuration::change(&self, request: ConfigurationChange) -> Result<ConfigurationResult, E6::Failure>`

- **Intent.** Commit supported reference-only connection settings and explicit static imports without executing source behavior.
- **Where.** `src/application/configuration.rs`.
- **Pre.** Fields and existing references validate; routes are finite/registered; approved E1 ClosedEnvironment validation governs all new Server saves/imports.
- **Post.** Configuration alone creates a private InventoryCommit after validation; one Store transaction compares revision and persists supported data. Static Rust import accepts only HostName, Port, User, IdentityFile, IdentitiesOnly, ProxyJump and accepted key-algorithm/cipher settings; rejects dynamic/unsupported semantics with located reasons, never reads Include targets, preserves source files and known_hosts, creates no trust/SSH and defaults imported connect_on_startup to false. GTK results distinguish committed settings, exclusions, empty, loading, stale and failed states.
- **Errors.** ValidationError, UnregisteredProxyRoute, ConfigurationConflict, StorageError leaves rejected changes unsaved. A parse report cannot claim uncommitted Servers. Ambiguous delivery is reported honestly, not retried automatically.
- **Idempotency.** Same-ID/content saves do not duplicate. Each explicit import is a new snapshot import, not re-import/diff. Crash yields all-or-none transaction; inspect actual state after ambiguity.

## Invariants

| ID    | Statement                                                                                                                                                                   | Role      | Proof                                                                                       |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------- |
| INV-1 | Inventory reads return only actual committed SQLite snapshots and never perform SSH or turn storage failure into empty success.                                             | preserve  | Future test `configuration_inventory_real_store_failure`; first proof owner 01              |
| INV-2 | ConnectionIdentity includes every effective target/hop host, port, user and authentication context; metadata aliases cannot change equality and route resolution is finite. | preserve  | Future test `configuration_complete_identity_resolution`; first proof owner 01              |
| INV-3 | Only Configuration-created InventoryCommit can persist validated reference-only changes, atomically at its expected revision and without secrets.                           | establish | Future test `configuration_atomic_reference_commit`; first proof owner 03                   |
| INV-4 | Import never evaluates commands, reads Include targets, rewrites originals, synchronizes snapshots or accepts Host Keys.                                                    | establish | Future test `configuration_static_import_isolation`; first proof owner 03                   |
| INV-7 | Every newly saved environment maps to ClosedEnvironment; inventory preserves legacy bytes and never silently normalizes another row.                                        | establish | Future test `configuration_closed_environment_and_legacy_preservation`; first proof owner03 |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Save/import: all steps; registered-route import data for 05.

- In: `InventorySnapshot` ← 01
- In: `ConfigurationChange` ← external
- Out: `ImportReport` → 05
- Out: `ConfigurationResult` → final

External producer and gate details: ADR-0006:01 is the declaration and physical Store producer. Local 01 provides actual snapshots; human 02 has approved exact E1 environment policy. Runtime issuance of InventoryCommit is ONLY this ticket, not E6 declaration creation.

## Steps

1. First consume the approved 02 policy and update Design with exact variants/default/validation; do not choose values in code.
2. Write failing transaction/reference/secret/CAS tests and implement change creating private InventoryCommit. Reuse InventoryCommit::inventory_write_view introduced by ADR-0006:01; do not redefine its method or mint authority through a view.
3. Write failing static-parser tests for each whitelist category, unsupported values, mixed/empty results, Match bodies, Include refusal and command-like text. Use one whitelist definition, not a second parser or evaluator.
4. Implement registered ProxyJump inventory materialization only when representable; reject every opaque/unregistered route. Preserve original bytes on success, partial input, failure and cancellation.
5. Implement real private GTK import surface: keyboard controls, glib::MainContext updates, loading/empty/exclusion/recovery states; distinguish unsaved rejected entries from committed settings.
6. Exercise real SQLite rollback/unavailable storage and isolated container filesystem full; keep generation-specific US-010-AC2/5/6 regression wiring in 04 and actual connection honoring imported choices in ADR-0001 integration.
7. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-006-AC1`: Given the Developer explicitly invokes import for `~/.ssh/config`, when the source is read, then Rust statically parses the file without executing commands, opening SSH connections or invoking OpenSSH to evaluate the file.
- [ ] `US-006-AC2`: Given supported entries are imported successfully, when they are displayed, then they exist as saved inventory/Auth Profile settings rather than live references whose values depend on re-reading the original configuration at connection time.
- [ ] `US-006-AC3`: Given the original file contains a mixture of supported and unsupported content, when import reports its result, then the Developer can distinguish the settings imported from the entries/directives not imported and see the reason for each exclusion.
- [ ] `US-006-AC4`: Given a Server is saved through import without an explicit connection request, when import finishes, then it can remain disconnected; importing settings is not proof of connection or Host Key trust and does not bypass the default `connect_on_startup=false` behavior.
- [ ] `US-006-AC5`: Given source values contain text that resembles a command, when import and later connection run, then data cannot cause the importer to execute that text or emit excluded executable directives into generated configuration.
- [ ] `US-007-AC1`: Given statically supported HostName, Port and User settings, when import succeeds, then the saved connection settings and a subsequent actual connection use their imported endpoint and SSH user values.
- [ ] `US-007-AC2`: Given statically supported IdentityFile and IdentitiesOnly settings, when import succeeds and the saved configuration is used, then those settings are represented as non-secret key/authentication references and honored without importing key material or a passphrase into SQLite.
- [ ] `US-007-AC3`: Given a statically supported ProxyJump setting, when import succeeds, then it contributes the corresponding registered inventory Jump Server route under US-004 rather than a runtime dependency on a source-file alias.
- [ ] `US-007-AC4`: Given statically supported key-algorithm or cipher settings, when they are imported, then a subsequent generated configuration can use those allowed algorithm/cipher choices without permitting them to override the forced Host Key and local-execution policy.
- [ ] `US-007-AC5`: Given any directive outside HostName, Port, User, IdentityFile, IdentitiesOnly, ProxyJump and the accepted key-algorithm/cipher category, when import examines it, then it is excluded with a reason even if OpenSSH normally recognizes that directive.
- [ ] `US-007-AC6`: Given a source value is unsupported or cannot be represented in accepted inventory/Auth Profile settings, when import finishes, then that value is not presented as successfully imported or silently turned into an executable directive, and the exclusion is reported.
- [ ] `US-008-AC1`: Given a source contains a `Match` block, including `Match exec`, when import encounters it, then the block is reported as not imported with a reason, its condition is never evaluated and settings within that block are not imported as if they were unconditional values.
- [ ] `US-008-AC2`: Given a source contains `Include`, when import encounters it, then the directive is reported as not imported with a reason and the importer does not read, expand, recursively import or evaluate the referenced file tree.
- [ ] `US-008-AC3`: Given a source contains `ProxyCommand`, when import encounters it, then it is reported and skipped rather than executed or translated into a command-backed proxy route.
- [ ] `US-008-AC4`: Given a source contains `KnownHostsCommand` or `LocalCommand`, when import encounters it, then each is reported and skipped and neither command executes during import, checking or connection.
- [ ] `US-008-AC5`: Given a source contains other out-of-whitelist trust, control, agent or local-execution settings, when import encounters them, then they are reported and skipped rather than allowed to replace application-generated policy or Auth Profile choices.
- [ ] `US-008-AC6`: Given all apparent connection values are inside unsupported dynamic blocks or Include trees, when import completes, then the result accurately reports that no supported connection settings were imported rather than fabricating a resolved OpenSSH configuration.
- [ ] `US-009-AC1`: Given successful, partially supported or failed import input, when the workflow ends, then the original SSH configuration contents are unchanged and the application has not rewritten excluded directives or inserted generated settings into that file.
- [ ] `US-009-AC2`: Given a connection setting has been imported, when the original file later changes, then the saved inventory snapshot does not change automatically and the UI does not imply that it tracks those external edits.
- [ ] `US-009-AC3`: Given config Import has created a Server, when its Trust State is considered, then configuration import has neither read/adopted user `known_hosts` entries nor marked the Host Key `ADOPTED` or `VERIFIED`; those require the separate explicit ADR-0003 workflows.
- [ ] `US-009-AC4`: Given the source contains an Include reference or is subsequently modified, when this epic's workflows are used, then no recursive Include import, automatic re-import or re-import/diff workflow is exposed as implemented functionality.
- [ ] `US-010-AC1`: Given the import surface and a result containing unsupported directives, when used with the keyboard, then import controls, exclusions/reasons and available error-recovery actions are reachable without a pointer.
- [ ] `US-010-AC2`: Given source loading/parsing or configuration generation is in progress, when the Developer continues using the application, then the workflow exposes loading/progress without blocking GTK's main thread; presentation updates are delivered through `glib::MainContext` and Tokio never directly accesses GTK widgets.
- [ ] `US-010-AC3`: Given the selected source has no importable settings, when parsing completes, then the view presents an accurate empty result with exclusions where applicable rather than a success claim for nonexistent Servers or an indefinite loading state.
- [ ] `US-010-AC4`: Given the source cannot be read or contains a value that cannot be imported, when the failure is reported, then the UI identifies the affected source/entry, explains what failed and available actions, and does not claim the rejected settings were saved or that remote changes occurred.
- [ ] `US-010-AC5`: Given local storage or generated-configuration creation fails, when the workflow reports its outcome, then it identifies the affected inventory/Connection Identity, accurately states what was or was not saved, exposes the appropriate storage/configuration error and does not claim a usable configuration or successful connection.
- [ ] `US-010-AC6`: Given configuration failure affects an already presented workflow, when it is displayed, then the error identifies affected resources, whether changes may have occurred, available actions and applicable Operation details; any prior observation remains visibly stale rather than being presented as current success.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test configuration_workflows configuration_atomic_reference_commit
cargo test --test configuration_workflows configuration_static_import_isolation
cargo test --test configuration_gtk configuration_import_gtk_states
rustfmt --edition 2024 --check src/domain/inventory.rs src/application/configuration.rs src/ui/configuration.rs tests/configuration_workflows.rs tests/configuration_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0002-generated-ssh-config/spec.md .scratch/adr-0002-generated-ssh-config/issues/00-overview.md .scratch/adr-0002-generated-ssh-config/issues/03-static-reference-import.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add recursive Include/re-import/diff, arbitrary OpenSSH evaluation, direct Store-vector writes, implicit Adoption or a second launch API.

## Comments

E1 is approved in canonical E2 Design. ClosedEnvironment has exactly Unspecified, Development, Test, Staging and Production, serialized as unspecified, development, test, staging and production. The UI and static import default to Unspecified. ServerDraft keeps raw boundary input but every newly stored value must validate through ClosedEnvironment. Reject explicit null, empty, whitespace, aliases, wrong case and unknown strings without coercion. Preserve existing raw observations and require explicit valid selection when saving a legacy Server. No public signature or protocol omission rule changes. Establish INV-7 with configuration_closed_environment_and_legacy_preservation here; preserve existing inventory read proof in 01. Readiness means policy-complete assignment with declared prerequisites, not implemented or runtime accepted.
