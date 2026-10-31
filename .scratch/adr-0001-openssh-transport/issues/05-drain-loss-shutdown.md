# 05: Drain every descendant and integrate master loss and shutdown

**Status:** ready-for-agent
**Blocked by:** 02: Connect and authenticate one shared master with independent terminals, 03: Carry isolated typed SFTP Operation and Collector consumers, 04: Traverse registered Jump routes and discover through trusted prefixes.
**External blockers:** ADR-0006:02.
**Source:** US-012-AC1, US-012-AC2, US-012-AC3, US-012-AC4, US-012-AC5, US-013-AC1, US-013-AC2, US-013-AC3, US-013-AC5, US-014-AC1, US-014-AC2, US-014-AC3, US-014-AC4, US-014-AC5.
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Disconnect, master loss and exit stop all owned descendants before master close, update every alias, and leave unrelated connections usable. Transport never converts cleanup into verified administrative success or a replay request.

E6:02 admission is actual prerequisite, not launch implementation; E6:04 final desktop composition consumes this quiesce implementation later and is not a blocker. Later E4/E5 shutdown clients consume the joined transport result; this ticket must handle typed consumer termination without assigning their domain outcomes or depending cyclically on their final UI. Enabled administrative scenarios remain gated by the actual E4 per-type policy/issuer tickets; never forge authority for all-kind teardown tests.

## Types

```rust
ConnectionManager // from ADR-0006:01
Connection // from ADR-0006:01
Terminal // from ADR-0006:01
Consumer // from ADR-0006:01
ChannelEnd // from ADR-0006:01
TerminalBinding // from ADR-0006:01
TerminalLifecycleReport // from ADR-0006:01
ConnectCompletion // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `ConnectionManager::quiesce(&self) -> Result<Vec<ChannelEnd>, E6::Failure>`

- **Intent.** Join all owned transport before runtime closes storage and exits.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** Connection, Terminal, Consumer, pending continuations.
- **Writes.** Connection, Terminal, Consumer, ChannelEnd.
- **Pre.** Runtime closes new admission and has notified/quiesced E4/E5.
- **Post.** Stored drain waiters and stop/join workers leave mailbox alive; all descendants including late-returned child, VTE exit/reap and PTY release are joined before master closes; aliases cease Connected; no Collector remains.
- **Invariants.** INV-1, INV-2, INV-3, INV-6, INV-7, INV-8, INV-9, INV-10, INV-12.
- **Errors.** Unjoined resource or failed cleanup keeps ownership and reports incomplete shutdown; no success from cancellation alone.
- **Idempotency.** Concurrent calls join one global drain; completed call is no-op; no replay/reconnect after stopping.

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
| INV-10 | Every descendant is actually joined before its shared master is closed; other identities remain usable.                    | establish | Future test `transport_all_consumers_disconnect`; first proof 05                      |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan.                                                 | preserve  | Future test `transport_orphan_cleanup`; first proof 01                                |
| INV-12 | Routed traffic/discovery uses only the registered complete route with every Jump gated before use.                         | preserve  | Future test `transport_registered_jump_discovery`; first proof 04                     |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Disconnect/loss/shutdown; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `ConnectionPorts` ← 02
- In: `Consumer` ← 03
- In: `Connection` ← 04
- Out: `ChannelEnd` → final
- Out: `ConnectionView` → final

## Steps

1. Write failing all-kind descendant drain and unaffected-second-identity tests using actual VTE, SFTP, allowed structured/read channels and Collector; include late worker child and delayed PTY disposal.
2. Deepen the existing control owner privately: close shared identity admission, stop/join all registered resources, update aliases, and close master last. Missing acknowledgements leave disconnect incomplete.
3. Implement quiesce with stored waiters/owned join workers; continue servicing terminal reports, consumer termination and stale completion cleanup until every tracked resource is joined.
4. Exercise real master/control-check loss, deliberate Disconnect, normal exit, parent death and next-launch cleanup integration. Preserve E4 conservative remote effects/Unknown and E5 stop/no-reconnect; never replay mutation.
5. Complete actual keyboard/accessibility, real loading/error/stale and shared-alias presentation checks across loss/shutdown while another identity stays interactive.
6. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-012-AC1`: Given terminal, SFTP, Operation and Collector channels exist for a Connection Identity, when the Developer disconnects it, then the connection manager explicitly closes all descendant sessions before closing that master.
- [ ] `US-012-AC2`: Given multiple Server entries share that Connection Identity, when its master is disconnected, then every affected entry and consumer reflects the shared connection loss rather than retaining a false connected presentation.
- [ ] `US-012-AC3`: Given another Connection Identity remains active, when this identity is disconnected, then the other identity's master and sessions remain usable.
- [ ] `US-012-AC4`: Given Disconnect interrupts a transfer or Operation, when its result is displayed, then it retains the owning workflow's distinction between known no effects and possible remote effects and provides the relevant details rather than reporting success or safe cancellation merely because local processes ended.
- [ ] `US-012-AC5`: Given Disconnect completes, when process/channel ownership is inspected, then no descendant session remains owned by the disconnected master and no monitoring loop continues through that connection.
- [ ] `US-013-AC1`: Given a connected master exits or fails `ssh -O check`, when the connection manager observes the loss, then Connection State and affected consumer states cease claiming an active usable connection using only evidence-supported classification.
- [ ] `US-013-AC2`: Given a master is spawned, when its application parent dies, then the configured `PR_SET_PDEATHSIG` terminates that master rather than deliberately persisting it beyond the application.
- [ ] `US-013-AC3`: Given a normal application exit with active connections, when shutdown finishes, then owned sessions and masters are closed and Collectors stop.
- [ ] `US-013-AC5`: Given an interrupted Operation existed when transport or the application died, when the next connection becomes usable, then no transport-level replay occurs; Unknown Outcome and read-only Reconciliation are handled by ADR-0004 and ADR-0006.
- [ ] `US-014-AC1`: Given the relevant GTK connection, askpass or terminal surface, when it is used by keyboard, then available Connect, Disconnect, prompt confirmation/cancellation, terminal session and error-recovery controls can be reached and activated without a pointer.
- [ ] `US-014-AC2`: Given no Server, no open terminal or no connection has yet supplied data, when its surface is shown, then an appropriate empty or not-connected state is displayed rather than an indefinite spinner or invented connection result.
- [ ] `US-014-AC3`: Given connection, authentication, Capability detection or disconnection is in progress, when the Developer navigates or interacts with other surfaces, then GTK remains responsive and the affected workflow exposes its actual loading/progress state.
- [ ] `US-014-AC4`: Given an asynchronous result arrives, when visible widget state changes, then GTK owns presentation updates through channels scheduled to `glib::MainContext`; Tokio work never accesses GTK widgets directly.
- [ ] `US-014-AC5`: Given a prior observation survives a connection loss, when it is displayed, then its stale or disconnected status remains visible and an error does not turn old information into a current observation.
- [ ] No successful drain is reported before every actual registered child, including late spawn return and delayed VTE PTY release, joins; other identities remain interactive.
- [ ] Unknown remote effects remain E4-owned and explicit Disconnect never becomes safe rollback, verified cancellation, successful transfer or autonomous Collector reconnect.
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
cargo test --test transport_workflows transport_all_consumers_disconnect -- --exact
cargo test --test transport_workflows transport_orphan_cleanup -- --exact
cargo test --test transport_workflows transport_registered_jump_discovery -- --exact
cargo test --test transport_gtk transport_keyboard_loading_error_and_vte -- --exact --ignored
rustfmt --edition 2024 --check src/application/connection_manager.rs src/infrastructure/ssh/mod.rs src/ui/transport.rs tests/transport_workflows.rs tests/transport_gtk.rs tests/support/transport.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Public functions introduced by local tickets 01, 02, 03, 04; use them without adding duplicate APIs. Later private refinement of an earlier owned boundary is limited to this ticket's Goal.
- ADR-0002 inventory/import/generated policy, ADR-0003 trust eligibility/acceptance, ADR-0004 Verifier/Audit Trail/outcomes, ADR-0005 metric/backoff policy and ADR-0006 primary/admission/lease state remain with their canonical owners.
- Never add a second SSH library, arbitrary public launch/command API, privileged root shell, secret persistence, user-config runtime fallback, automatic mutation replay or orphan adoption.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.

## Comments

E1 environment, T3 trust eligibility, L1 local trust and S service policy decisions are approved in their canonical Designs. Readiness describes a policy-complete implementation assignment after its listed real producer prerequisites land, not an existing capability or runtime acceptance. No implementation or source acceptance criterion is checked by this update. File/transfer policy 03 remains independently unapproved; no positive file or download admission can be fabricated for downstream scenarios.
