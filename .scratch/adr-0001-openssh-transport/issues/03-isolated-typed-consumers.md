# 03: Carry isolated typed SFTP Operation and Collector consumers

**Status:** needs-info
**Blocked by:** 02: Connect and authenticate one shared master with independent terminals.
**External blockers:** ADR-0006:01, ADR-0004:04.
**Source:** US-001-AC3, US-009-AC1, US-009-AC2, US-009-AC3, US-009-AC4, US-010-AC1, US-010-AC2, US-010-AC3, US-010-AC4, US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4.
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Existing connected Servers carry separate typed read-only, SFTP, fixed-script and Collector channels without another login or payload mixing. Transport reports actual joined termination and dispatch only; enabled administrative execution remains gated by the canonical E4 policies and admissions.

External declarations: ADR-0006:01 introduces canonical E3/E4/E5 protocols only. ADR-0004:04 introduces readable sealed dispatch projections without minting execution/download authority. Runtime requests come from actual E4/E5 owners as they land; their canonical modules, not E1, issue E4 admissions and CollectorSpec. E4:08/09/10 remain downstream policy-enabled execution/transfer/Reconciliation integration, not fabricated existing producers and not a cyclic blocker of transport. Missing policy/authority remains unavailable: never construct ExecutionPermit/DownloadAdmission in an E1 fixture. Read-only actual consumers and admissible Collector records exercise the real common transport now; enabled mutation criteria below remain explicit guarded owner-integration requirements, not a claim they were executed.

## Types

```rust
ConsumerAccess // from ADR-0006:01
ConsumerTarget // from ADR-0006:01
ConsumerRequest // from ADR-0006:01
ConsumerEvent // from ADR-0006:01
Consumer // from ADR-0006:01
ConsumerParts // from ADR-0006:01
ChannelEnd // from ADR-0006:01
ChannelEndView // from ADR-0006:01
DispatchKnowledge // from ADR-0006:01
ChannelEndReason // from ADR-0006:01
MasterGeneration // from ADR-0006:01
RemoteDispatch // from ADR-0006:01
TransferDispatch // from ADR-0006:01
DownloadDispatch // from ADR-0006:01
ReconciliationRead // from ADR-0006:01
ExecutionPermit // from ADR-0006:01
ExecutionPermitView // from ADR-0006:01
RemoteDispatchView // from ADR-0006:01
TransferDispatchView // from ADR-0006:01
DownloadDispatchView // from ADR-0006:01
ReconciliationReadView // from ADR-0006:01
CollectorSpec // from ADR-0006:01
RawObservation // from ADR-0006:01
ServiceRead // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `ConsumerAccess::open(&self, target: ConsumerTarget, request: ConsumerRequest) -> Result<Consumer, E6::Failure>`

- **Intent.** Attach one typed consumer to an already-connected selected generation.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** ConsumerTarget, ConsumerRequest, current Connection and Server binding, E4 borrowed dispatch/permit/read views.
- **Writes.** Consumer.
- **Pre.** Existing generation Connected and admits children; relevant Capability available; sealed dispatch expected identity matches current binding.
- **Post.** Server lookup or issued-generation validation, sealed expected_identity comparison and child reservation are atomic in one owner turn; starts only fail-closed attachment channel. Execute/Transfer read the respective dispatch's permit through inspect_execution_permit and compare its borrowed expected_identity; Download compares inspect_download_dispatch.expected_identity; Reconcile compares the original Operation connection_identity and fails if absent or different. These views are never admission inputs. Services/text use E4 records, operations fixed POSIX login-user script, transfers separate SFTP payload and downloads read-only admission, Collector fixed structured loop; E1 stamps each RawObservation collector_generation with the actual channel ConsumerParts.id and collected_at with local receipt Instant, never remote time. Exactly one typed Ended follows actual join; no independent login.
- **Invariants.** INV-1, INV-2, INV-4, INV-6, INV-7, INV-9.
- **Errors.** Disconnected/missing/stale/mismatched target, unavailable Capability, denied access or spawn failure yields no alternate transport, Connect, authentication or queue-for-connect. Transport exit is never mutation verification/transfer completion.
- **Idempotency.** Each accepted call opens one channel; ambiguous delivery is not automatically retried. Collector owner explicitly decides allowed replacement channel.

### `ConsumerAccess::stop(&self, consumer_id: Uuid) -> Result<ChannelEnd, E6::Failure>`

- **Intent.** Obtain actual joined termination for one owned typed channel.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** Consumer.
- **Writes.** Consumer, ChannelEnd.
- **Pre.** ID is registered or completed result retained.
- **Post.** Releases owned process/channel resources and returns generation-bound dispatch and termination facts; no claim about remote rollback.
- **Invariants.** INV-2, INV-6, INV-9.
- **Errors.** Unknown ID changes nothing; failed cleanup retains ownership and returns actionable failure.
- **Idempotency.** Repeated requests join the same stop or return retained result, never reopen.

### `Consumer::into_consumer_parts(self) -> ConsumerParts`

- **Intent.** Transfer the unique typed event receiver with its consumer identity.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** Consumer.
- **Writes.** None.
- **Pre.** Manager returned Consumer; receiver not previously moved.
- **Post.** Moves exact ID, issued master and unique receiver to E3/E4/E5 owner; no copy of live receiver or authority.
- **Invariants.** INV-2, INV-9.
- **Errors.** Infallible consuming decomposition.
- **Idempotency.** One consuming call; stream closure alone is not proof of joined termination.

### `ChannelEnd::inspect_channel_end(&self) -> ChannelEndView<'_>`

- **Intent.** Expose joined channel termination without granting execution or outcome authority.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** ChannelEnd.
- **Writes.** None.
- **Pre.** Manager-issued actual joined result.
- **Post.** Borrows consumer/master/dispatch/reason unchanged; Dispatched says nothing about remote effects.
- **Invariants.** INV-2, INV-9.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable read; does not classify Operation or transfer outcome.

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
| INV-9  | Typed consumer selection atomically uses an existing Connected binding; views cannot issue a generation/start.             | establish | Future test `transport_consumers_existing_generation`; first proof 03                 |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan.                                                 | preserve  | Future test `transport_orphan_cleanup`; first proof 01                                |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Typed consumers; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `ConnectionPorts` ← 02
- In: `ConsumerRequest` ← external
- Out: `Consumer` → 04
- Out: `Consumer` → 05

## Steps

1. Write failing real shared-master read/SFTP/Collector isolation and existing-binding admission tests; kill master between check and spawn and observe no fresh authentication.
2. Implement open/stop and consuming/borrowed interfaces against owner registry; reserve atomically from ConsumerTarget, enforce expected identity for sealed E4 dispatches, and bind every running child to the selected generation.
3. Implement separate system ssh SFTP subsystem, fixed login-user POSIX-script and fixed structured Collector/read adapters. Use canonical E3/E4/E5 immutable records; never expose arbitrary argv, scripts, file payload on stdin or GTK objects to application callers.
   Stamp every Collector record with the actual E1-issued ConsumerParts.id as collection epoch and local receipt Instant as collected_at. A replacement channel gets a new UUID; remote record fields cannot choose either authority-bound timing/context field.
4. Exercise real read-only/script/Collector and SFTP channels with permission/Capability failures and joined termination. E4 owns execution/download admission and later policy-enabled mutation/verification scenarios; do not forge those wrappers to make a fixture execute.
5. Deliver precise ChannelEnd and typed stream observations to each owner. Preserve E4 Unknown/Verifier and E5 Freshness/backoff ownership; copy every policy-dependent source criterion as a required later owner integration gate, not as delivered mutation behavior.
6. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-001-AC3`: Given an established Connection Identity, when terminal, SFTP, Operation and Collector consumers are opened, then all use system OpenSSH through that identity's master rather than independent library-based SSH transports or separately authenticated replacement connections.
- [ ] `US-009-AC1`: Given a connected Server with an available SFTP subsystem, when a file transfer is requested, then `openssh-sftp-client` uses the OpenSSH `ssh -s <identity> sftp` subsystem channel through the existing master.
- [ ] `US-009-AC2`: Given SFTP payload traffic and a structured Operation run concurrently, when both transmit data, then file payload travels on its own channel and cannot become script stdin or terminal input.
- [ ] `US-009-AC3`: Given the SSH user's file permissions deny access or the SFTP subsystem is absent, when file transport is attempted, then the affected workflow reports the permission or Capability limitation without acquiring a privileged shell or silently switching transport.
- [ ] `US-009-AC4`: Given a transfer channel is lost before completion, when transport termination is delivered to the file workflow, then it is not reported as a completed transfer; file verification and any Unknown Outcome are decided by the file/Operation workflow.
- [ ] `US-010-AC1`: Given an Operation is admitted by its engine, when remote execution starts, then its fixed POSIX script is sent over a system OpenSSH channel through the existing master, with machine-readable results rather than parsed human-oriented command output.
- [ ] `US-010-AC2`: Given the remote Operation runs, when transport starts its shell, then it runs as the SSH login user and does not substitute a privileged root shell for ADR-0004's granular `sudo -n` commands.
- [ ] `US-010-AC3`: Given a channel exits successfully, when the engine is informed, then the transport result alone does not mark the Operation `SUCCEEDED`; its operation-specific Verifier still determines the outcome.
- [ ] `US-010-AC4`: Given SSH disconnects, the Server reboots or the channel is cancelled after a remote change may have started, when the engine is informed, then transport loss is not translated into proof of no remote effects, automatic retry or verified cancellation; ADR-0004 decides Unknown Outcome and ADR-0006 coordinates recovery.
- [ ] `US-011-AC1`: Given a connected Server whose monitoring is active, when collection runs, then its one long-lived Collector channel uses the identity's master and emits structured records from the fixed loop rather than creating a fresh SSH login for each sample.
- [ ] `US-011-AC2`: Given monitoring is active while terminal and other consumers are used, when collection is scheduled, then the configured Collector concurrency bound is respected and polling does not starve interactive terminal work.
- [ ] `US-011-AC3`: Given the Collector channel or master is lost, when monitoring receives transport termination, then the existing data becomes `STALE` under ADR-0005 instead of being replaced by invented zero values or a claim that the Server is down.
- [ ] `US-011-AC4`: Given collection recovery is attempted, when a new Collector channel is opened, then it remains subject to the current master's configuration, trust and authentication lifecycle; metric backoff and sample reset rules remain owned by ADR-0005.
- [ ] Disconnected Server-target open causes zero Connect/login/prompt attempts; changing inventory cannot retarget a running consumer or reconcile an old identity on a new Server target.
- [ ] Sealed read-only download cannot execute remote mutation, while absent approved proof/admission remains unavailable; no E1 fixture fabricates a positive E4 permit.
- [ ] The source criteria requiring an enabled administrative issuer remain policy-gated owner integration acceptance, not a claim that missing E4 Verifier/transfer policies are implemented here.
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
rustfmt --edition 2024 --check src/application/connection_manager.rs src/infrastructure/ssh/mod.rs tests/transport_workflows.rs tests/support/transport.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Public functions introduced by local tickets 01, 02, 04, 05; use them without adding duplicate APIs. Later private refinement of an earlier owned boundary is limited to this ticket's Goal.
- ADR-0002 inventory/import/generated policy, ADR-0003 trust eligibility/acceptance, ADR-0004 Verifier/Audit Trail/outcomes, ADR-0005 metric/backoff policy and ADR-0006 primary/admission/lease state remain with their canonical owners.
- Never add a second SSH library, arbitrary public launch/command API, privileged root shell, secret persistence, user-config runtime fallback, automatic mutation replay or orphan adoption.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
