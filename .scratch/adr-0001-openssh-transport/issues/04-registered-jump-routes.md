# 04: Traverse registered Jump routes and discover through trusted prefixes

**Status:** needs-info
**Blocked by:** 02: Connect and authenticate one shared master with independent terminals, 03: Carry isolated typed SFTP Operation and Collector consumers.
**External blockers:** ADR-0002:05, ADR-0003:04.
**Source:** US-001-AC1, US-001-AC2, US-003-AC3, US-003-AC4, US-003-AC5.
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Registered multi-hop Jump routes reach the real target using only individually gated prefixes. Routed discovery uses the existing trusted Jump generation and never scans locally or transfers trust from Jump to target.

External producer ADR-0002:05 refines actual Configuration::compile with the complete registered route; ADR-0002:01 resolves inventory identities. ADR-0003:04 supplies the actual trust gate over the same canonical trust core. Consumer from 03 is the actual typed scanner channel used by routed discovery, not a new login. E3:06 consumes this route capability later and must not be an upstream blocker. ADR-0003:02 trust eligibility is still a human gate, not policy supplied here.

## Types

```rust
RouteAccess // from ADR-0006:01
Connection // from ADR-0006:01
MasterGeneration // from ADR-0006:01
ConsumerTarget // from ADR-0006:01
ConsumerRequest // from ADR-0006:01
Consumer // from ADR-0006:01
ConsumerParts // from ADR-0006:01
ConnectCompletion // from ADR-0006:01
TrustWork // from ADR-0006:01
ResolvedConnection // from ADR-0006:01
DiscoveryPath // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `RouteAccess::prepare_discovery(&self, resolved: E2::ResolvedConnection) -> Result<E3::DiscoveryPath, E6::Failure>`

- **Intent.** Provide discovery through real registered and gated Jump prefixes.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** E2::ResolvedConnection, current prefix MasterGeneration.
- **Writes.** Connection for prerequisite prefixes.
- **Pre.** Route finite and registered in inventory.
- **Post.** Direct target returns Local; routed target connects strictly shorter prefixes using ordinary control workers, returns last already-gated Jump generation; target trust is never inherited. Worker waits while manager continues servicing KeyScan/stop/askpass/completions.
- **Invariants.** INV-1, INV-6, INV-7, INV-12.
- **Errors.** Unregistered route, missing prefix trust, failed/lost prefix blocks discovery; never scans routed target locally or uses external alias.
- **Idempotency.** Reuses current prefix generations; repeatable read-only preparation does not accept target key.

## Invariants

| ID     | Statement                                                                                                                  | Role      | Proof                                                                                 |
| ------ | -------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------- |
| INV-1  | At most one starting/live master exists for each structurally equal Connection Identity.                                   | preserve  | Future test `transport_shared_identity_and_components`; first proof 02                |
| INV-2  | An attachment uses only its specified current master generation or fails without authentication.                           | preserve  | Future test `transport_mux_loss_no_fallback`; first proof 02                          |
| INV-3  | Closing a terminal requires actual child termination/reaping and PTY release for that shell.                               | preserve  | Future test `transport_terminal_pty_drain`; first proof 02                            |
| INV-4  | Each specific Connection State and Capability has independent supporting evidence; 255/stderr alone is Unknown.            | preserve  | Future test `transport_evidence_classification`; first proof 02                       |
| INV-5  | Every terminal has independent streams/PTY and immutable original Server context.                                          | preserve  | Future test `transport_vte_independent_context`; first proof 02                       |
| INV-6  | Master/authentication start requires current primary admission, generated configuration, trust and GTK-owned presentation. | preserve  | Future test `transport_admission_authentication_and_secret_exclusion`; first proof 02 |
| INV-7  | Waiting route/trust/configuration/launch work never prevents servicing dependent manager messages.                         | preserve  | Future test `transport_continuation_cancellation`; first proof 02                     |
| INV-8  | Terminal state/drain uses exact shell/master binding and actual spontaneous child/PTY observations.                        | preserve  | Future test `transport_terminal_early_exit_and_stale_report`; first proof 02          |
| INV-9  | Typed consumer selection atomically uses an existing Connected binding; views cannot issue a generation/start.             | preserve  | Future test `transport_consumers_existing_generation`; first proof 03                 |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan.                                                 | preserve  | Future test `transport_orphan_cleanup`; first proof 01                                |
| INV-12 | Routed traffic/discovery uses only the registered complete route with every Jump gated before use.                         | establish | Future test `transport_registered_jump_discovery`; first proof 04                     |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Registered route and trusted-prefix discovery; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `ConnectionPorts` ← 02
- In: `Consumer` ← 03
- In: `ResolvedConnection` ← external
- Out: `Connection` → 05

## Steps

1. Write failing actual two-hop registered-route target traffic and routed-key discovery cases, including unregistered route, ungated Jump, lost prefix and separately untrusted target.
2. Implement prepare_discovery as an owned worker using finite resolved inventory route and ordinary strictly shorter-prefix Connect; mailbox must continue serving dependent KeyScan, askpass and stop.
3. Use E2 generated prefix/target policy and E3 issued trust/snapshot, and return only Local or the actual last gated Jump generation. Preserve no local fallback, no external user route, no target-trust inheritance.
4. Demonstrate terminal or structured traffic reaches actual routed target and routed discovery can consume the issued prefix generation; keep E3:06 presentation refinement downstream, not a blocker here.
5. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-001-AC1`: Given two Server entries with the same host, port, SSH user, complete Jump Server route and authentication context, when both are connected or used concurrently, then their consumers use exactly one ControlMaster for that Connection Identity.
- [ ] `US-001-AC2`: Given entries differing in host, port, SSH user, Jump Server route or authentication context, when they connect, then they do not reuse a master belonging to the different Connection Identity.
- [ ] `US-003-AC3`: Given a target uses a Jump Server, when connection is attempted, then the route must be registered in inventory and the Jump Server must pass the trust gate before it is used.
- [ ] `US-003-AC4`: Given an unregistered proxy route is detected, when the connection workflow reports failure, then it exposes `UnregisteredProxyRoute` and does not silently use an alternate route or the user's SSH configuration.
- [ ] `US-003-AC5`: Given a trusted Jump Server and an accepted target Host Key, when the target connection succeeds, then actual terminal or structured traffic reaches that target through the registered Jump Server using OpenSSH ProxyJump behavior.
- [ ] All Jump prefixes are registered and gated independently; no routed discovery falls back to local scan, ~/.ssh/config or implicit proxy command.
- [ ] Pending routed scan/Connect cancellation is serviced by the live manager mailbox, with no owner-state lock or SQLite transaction across remote await.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test transport_workflows transport_shared_identity_and_components -- --exact
cargo test --test transport_workflows transport_mux_loss_no_fallback -- --exact
cargo test --test transport_workflows transport_terminal_pty_drain -- --exact
cargo test --test transport_workflows transport_evidence_classification -- --exact
cargo test --test transport_workflows transport_vte_independent_context -- --exact
cargo test --test transport_workflows transport_admission_authentication_and_secret_exclusion -- --exact
cargo test --test transport_workflows transport_continuation_cancellation -- --exact
cargo test --test transport_workflows transport_terminal_early_exit_and_stale_report -- --exact
cargo test --test transport_workflows transport_consumers_existing_generation -- --exact
cargo test --test transport_workflows transport_orphan_cleanup -- --exact
cargo test --test transport_workflows transport_registered_jump_discovery -- --exact
rustfmt --edition 2024 --check src/application/connection_manager.rs src/infrastructure/ssh/mod.rs tests/transport_workflows.rs tests/support/transport.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Public functions introduced by local tickets 01, 02, 03, 05; use them without adding duplicate APIs. Later private refinement of an earlier owned boundary is limited to this ticket's Goal.
- ADR-0002 inventory/import/generated policy, ADR-0003 trust eligibility/acceptance, ADR-0004 Verifier/Audit Trail/outcomes, ADR-0005 metric/backoff policy and ADR-0006 primary/admission/lease state remain with their canonical owners.
- Never add a second SSH library, arbitrary public launch/command API, privileged root shell, secret persistence, user-config runtime fallback, automatic mutation replay or orphan adoption.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
