# 05: Present accessible monitoring states and join shutdown

**Status:** needs-info
**Blocked by:** 01: Collect actual observations with persisted validated settings, 04: Bound memory retention and Collector concurrency.
**External blockers:** ADR-0001:05.
**Source:** US-002-AC5, US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4, US-011-AC5, US-012-AC1, US-012-AC2, US-012-AC3, US-012-AC4.
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

The actual GTK monitoring surfaces keep Server context, keyboard controls, Freshness, pending reads and actionable failure states accessible while work waits. Runtime shutdown joins every Collector and invalidates recovery before transport stops, without monitoring claiming administrative evidence.

ADR-0001:05 supplies actual final all-consumer shutdown integration and master-loss semantics; Monitoring::quiesce here is the actual E5 runtime producer. Runtime final launch/shutdown ADR-0006:04 consumes this implementation later, not the other way around. The E4 read-only summary supplied in 01 never becomes execution/Verifier authority, even alongside an actual administrative Operation.

## Types

```rust
Monitoring // from ADR-0006:01
MonitoringMessage // from ADR-0006:01
MonitoringCompletion // from ADR-0006:01
Collector // from ADR-0006:01
CollectorPhase // from ADR-0006:01
MetricHistory // from ADR-0006:01
MetricSeries // from ADR-0006:01
MetricStatus // from ADR-0006:01
Freshness // from ADR-0006:01
MonitoringChanged // from ADR-0006:01
ChannelEnd // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `Monitoring::quiesce(&self) -> Result<(), E6::Failure>`

- **Intent.** Stop all collection and recovery before desktop shutdown.
- **Where.** `src/application/monitoring.rs`
- **Reads.** Collector, retry timers, pending consumer workers, ConsumerAccess.
- **Writes.** Collector phase and ephemeral scheduling/buffers.
- **Pre.** Runtime closes new admission.
- **Post.** Invalidates timers/generations, denies new collect/refresh, stops and joins tracked consumers including late worker returns via E1 stop. Owner continues handling completions until actual joins; Stopped only after no channel/recovery work remains.
- **Invariants.** INV-1, INV-2, INV-3, INV-4, INV-5, INV-6, INV-7, INV-8, INV-9.
- **Errors.** Unjoined work or failed cleanup reports incomplete shutdown, retains ownership; cancellation alone never claims stop.
- **Idempotency.** Concurrent calls join one drain; no future retry/reconnect; process exit discards history.

## Invariants

| ID    | Statement                                                                                                                      | Role      | Proof                                                                         |
| ----- | ------------------------------------------------------------------------------------------------------------------------------ | --------- | ----------------------------------------------------------------------------- |
| INV-1 | At most one live Collector exists per Server and page navigation never duplicates it.                                          | preserve  | Future test `monitoring_actual_sources_and_single_collector`; first proof 01  |
| INV-2 | Each sample retains actual Server/metric/unit and local collection context; absent or failed reads never become zero.          | preserve  | Future test `monitoring_sources_context_and_partial_failures`; first proof 01 |
| INV-3 | Only Monitoring can authorize validated settings writes; Store protocol cannot persist samples.                                | preserve  | Future test `monitoring_settings_sealed_persistence`; first proof 01          |
| INV-4 | Every CPU/network delta has comparable same-continuity counters and positive measured elapsed time.                            | preserve  | Future test `monitoring_counter_continuity`; first proof 02                   |
| INV-5 | Freshness is per metric from local monotonic collection time; error/unavailable/gaps never become new successful observations. | preserve  | Future test `monitoring_monotonic_freshness`; first proof 03                  |
| INV-6 | Recovery retries only Collector channels with exponential backoff under current E1 admission, never master authentication.     | preserve  | Future test `monitoring_loss_backoff_and_disconnect`; first proof 03          |
| INV-7 | Samples reside only in per-Server process-memory rings and temporal retention is at most one hour.                             | preserve  | Future test `monitoring_retention_and_no_disk_history`; first proof 04        |
| INV-8 | Active Collectors obey the effective positive configurable bound and never own terminal admission.                             | preserve  | Future test `monitoring_concurrency_and_terminal_interaction`; first proof 04 |
| INV-9 | GTK alone mutates widgets; monitoring stays read-only and stops all channels/recovery before quiescence success.               | establish | Future test `monitoring_accessible_states_and_shutdown`; first proof 05       |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Observe/present/stop; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `Monitoring` ← 01
- In: `MetricHistory` ← 04
- In: `MonitoringSettings` ← 04
- Out: `MetricHistory` → final
- Out: `Collector` → final

## Steps

1. Write failing actual GTK keyboard/navigation/refresh and per-metric Freshness/error tests while actual Collector reads wait; preserve stale history separately from pending new work and actual error.
2. Integrate channel events on glib::MainContext with explicit Server context, loading/genuine empty/unavailable/error/stale states, actionable read-only error details and actual Operation links only when related.
3. Implement quiesce with stored waiters, generation/timer invalidation and actual E1 channel joins, including late open completions. Mailbox continues processing joins; cancellation alone cannot satisfy stop.
4. Exercise explicit Disconnect, application exit/restart and mixed Services/SFTP/admin work through real boundaries; confirm no background Collector/reconnect, no restored sample history, and periodic service summary never supplies Verifier/Reconciliation evidence.
5. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-002-AC5`: Given application exit, when lifecycle cleanup runs, then the Collector and its owned collection work stop; no collection continues as a background monitoring daemon after the application closes.
- [ ] `US-011-AC1`: Given initial collection is in progress and no response has arrived, when I open the relevant view, then a loading state is shown instead of an invented zero series or a completed empty result.
- [ ] `US-011-AC2`: Given no samples have been collected or a successful query genuinely has no applicable data, when the view is shown, then it provides an empty or unavailable explanation appropriate to that condition rather than reporting a collection error that did not occur.
- [ ] `US-011-AC3`: Given a collection or unsupported-Capability error, when it is presented, then it identifies what failed, the affected Server or metric, that this read-only workflow did not itself perform an administrative change, and the available actions; Operation details are linked only when an actual related Operation exists.
- [ ] `US-011-AC4`: Given keyboard-only interaction, when I select the Server, navigate metrics or invoke refresh, then those controls and the meaningful Freshness/error information remain accessible without pointer-only interactions.
- [ ] `US-011-AC5`: Given a stale chart with new collection pending, when I view it, then stale retained observations, loading of new observations and actual errors remain distinguishable rather than removing the known history or disguising it as live data.
- [ ] `US-012-AC1`: Given slow collection or an unreachable Server, when I navigate or use other application controls, then the GTK main thread remains responsive while Tokio handles Collector and SSH work without accessing GTK widgets.
- [ ] `US-012-AC2`: Given a newly collected result, when presentation is updated, then it arrives through channels scheduled on glib::MainContext, with GTK owning presentation, the application layer owning scheduling and workflow, the connection manager owning the master and the Collector ring buffer owning live Metric Samples.
- [ ] `US-012-AC3`: Given Services, SFTP or an administrative Operation sharing the Server connection, when monitoring runs alongside them, then collection remains a separate read-only consumer rather than putting file payload into the Collector protocol or granting itself mutation authority.
- [ ] `US-012-AC4`: Given a service Operation awaiting verification, when a live service summary changes, then that summary alone does not substitute for the ADR-0004 epic's operation-specific Verifier, stabilization policy or Reconciliation evidence.
- [ ] Stopped is emitted only after actual E1 joins for active and late-returned Collector opens; incomplete joins leave quiescence incomplete and visible.
- [ ] All GTK presentation occurs on main context; monitoring has no mutation authority and never substitutes a service summary for approved Operation evidence.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test monitoring_workflows monitoring_actual_sources_and_single_collector -- --exact
cargo test --test monitoring_workflows monitoring_sources_context_and_partial_failures -- --exact
cargo test --test monitoring_workflows monitoring_settings_sealed_persistence -- --exact
cargo test --test monitoring_workflows monitoring_counter_continuity -- --exact
cargo test --test monitoring_workflows monitoring_monotonic_freshness -- --exact
cargo test --test monitoring_workflows monitoring_loss_backoff_and_disconnect -- --exact
cargo test --test monitoring_workflows monitoring_retention_and_no_disk_history -- --exact
cargo test --test monitoring_workflows monitoring_concurrency_and_terminal_interaction -- --exact
cargo test --test monitoring_workflows monitoring_accessible_states_and_shutdown -- --exact
cargo test --test monitoring_gtk monitoring_keyboard_states_and_interactivity -- --exact --ignored
rustfmt --edition 2024 --check src/application/monitoring.rs src/ui/monitoring.rs tests/monitoring_workflows.rs tests/monitoring_gtk.rs tests/support/monitoring.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Earlier public function ownership stays in 01; 02–04 add private behavior only, and 05 owns quiesce. Do not create a second monitoring/refresh/retry API.
- E1 owns masters and authentication; E2 inventory identity, E3 trust, E4 administrative evidence/outcomes and E6 admission/storage remain foreign owner state.
- No persistent sample/history store, remote agent/daemon, autonomous Connect, mutation authority, fabricated zero, gap interpolation, inferred downtime, undocumented numeric tuning policy or unsupported benchmark claim.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
