# 03: Preserve Freshness and recover Collector channels honestly

**Status:** needs-info
**Blocked by:** 01: Collect actual observations with persisted validated settings, 02: Calculate rates only from measured counter continuity.
**External blockers:** None beyond the local blocker chain.
**Source:** US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4, US-007-AC5, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-008-AC5.
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Each metric exposes honest monotonic Freshness through errors, channel loss and recovery. Retry is exponential replacement-channel backoff under E1 ownership, never a new master, autonomous Connect or resurrection after deliberate Disconnect.

No new public functions. This ticket privately deepens the existing scheduler/feed/observe boundary from 01 and continuity normalizer from 02. A replacement E1 connection observation is actual authoritative input via a fresh dedicated ConnectionManager::observe subscription; not a fabricated resync or reuse of Runtime receiver.

## Types

```rust
Monitoring // from ADR-0006:01
Collector // from ADR-0006:01
CollectorPhase // from ADR-0006:01
MonitoringCompletion // from ADR-0006:01
MetricSample // from ADR-0006:01
MetricSeries // from ADR-0006:01
MetricStatus // from ADR-0006:01
Freshness // from ADR-0006:01
CounterValue // from ADR-0006:01
CounterBaseline // from ADR-0006:01
MonitoringChanged // from ADR-0006:01
Connection // from ADR-0006:01
ConnectionChanged // from ADR-0006:01
ChannelEnd // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

No new public functions. Privately refine the existing boundaries named in Goal and Steps; do not duplicate an earlier ticket's introducing signature.

## Invariants

| ID    | Statement                                                                                                                      | Role      | Proof                                                                         |
| ----- | ------------------------------------------------------------------------------------------------------------------------------ | --------- | ----------------------------------------------------------------------------- |
| INV-1 | At most one live Collector exists per Server and page navigation never duplicates it.                                          | preserve  | Future test `monitoring_actual_sources_and_single_collector`; first proof 01  |
| INV-2 | Each sample retains actual Server/metric/unit and local collection context; absent or failed reads never become zero.          | preserve  | Future test `monitoring_sources_context_and_partial_failures`; first proof 01 |
| INV-3 | Only Monitoring can authorize validated settings writes; Store protocol cannot persist samples.                                | preserve  | Future test `monitoring_settings_sealed_persistence`; first proof 01          |
| INV-4 | Every CPU/network delta has comparable same-continuity counters and positive measured elapsed time.                            | preserve  | Future test `monitoring_counter_continuity`; first proof 02                   |
| INV-5 | Freshness is per metric from local monotonic collection time; error/unavailable/gaps never become new successful observations. | establish | Future test `monitoring_monotonic_freshness`; first proof 03                  |
| INV-6 | Recovery retries only Collector channels with exponential backoff under current E1 admission, never master authentication.     | establish | Future test `monitoring_loss_backoff_and_disconnect`; first proof 03          |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Observe/recover; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `Monitoring` ← 01
- In: `MetricSample` ← 02
- Out: `Collector` → 04
- Out: `MetricSeries` → 04

## Steps

1. Write failing local-monotonic/per-metric Freshness and remote-clock-change tests, mixed-cadence partial read errors and stale retained-history redraw/reopen cases.
2. Deepen private normalization/events to retain original collected_at, keep Fresh/Stale/Unavailable/Error distinct, and refresh only the metric with a new actual observation.
3. Write and implement real channel/master-loss recovery: invalidate baselines, mark old values stale, back off exponentially and open replacement channels only under authoritative existing E1 binding. Do not assert undocumented initial/capped delay or retry count.
4. Handle deliberate Disconnect, stopped generation and connection-feed loss by invalidating retry tokens and blocking admission until real replacement observe snapshot; stream lag must resync rather than invent continuity.
5. Exercise gap return without fabricated zeros, inferred DOWN or relabeling unrelated values; never call Connect or bypass trust/configuration.
6. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-007-AC1`: Given a displayed metric, when I inspect it, then its Freshness is visible for that metric rather than implied only by a page-wide connected indicator.
- [ ] `US-007-AC2`: Given an adjustment to the remote clock, when Freshness is evaluated, then it is still based on elapsed local monotonic time since collection and is not made newer, older or negative by the remote adjustment.
- [ ] `US-007-AC3`: Given collection-state changes, when samples are exposed, then FRESH, STALE, UNAVAILABLE and ERROR remain distinct observable statuses rather than a single valid/invalid flag.
- [ ] `US-007-AC4`: Given a failed metric read, when its error is shown, then ERROR is never replaced by a numerical zero or a silent reuse of an earlier value as FRESH.
- [ ] `US-007-AC5`: Given a retained last-known value that is no longer current, when it remains visible, then it is identified as such with its real Freshness and does not become a new successful observation merely because the UI redraws or I reopen the page.
- [ ] `US-008-AC1`: Given existing collected data, when its Collector channel disconnects, then that data becomes STALE rather than continuing to be presented as FRESH.
- [ ] `US-008-AC2`: Given an unexpected collection-channel failure while the connection lifecycle permits recovery, when collection is retried, then reconnect uses exponential backoff rather than an uncontrolled immediate retry loop; no undocumented initial delay, cap or attempt limit is asserted here.
- [ ] `US-008-AC3`: Given an explicit Server Disconnect, when the connection owner closes child channels and its master, then the Collector closes with them and does not independently reconnect against that deliberate disconnect.
- [ ] `US-008-AC4`: Given an interval without collection because of loss, disconnection or application closure, when data is displayed after return, then that gap is not filled with samples, zero values or an inferred DOWN period.
- [ ] `US-008-AC5`: Given a new successful non-delta observation after reconnect, when it is shown, then its Freshness reflects that new collection; recovery of one metric does not relabel other old or unavailable metrics as current.
- [ ] A lost authoritative E1 observation feed blocks channel admission until an actual dedicated replacement subscription/snapshot is installed; Runtime retains its own independent receiver.
- [ ] No retry timer can resurrect a deliberate Disconnect or stopped-generation channel, and no metric failure labels a Server offline.
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
rustfmt --edition 2024 --check src/domain/metrics.rs src/application/monitoring.rs tests/monitoring_workflows.rs src/ui/monitoring.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Earlier public function ownership stays in 01; 02–04 add private behavior only, and 05 owns quiesce. Do not create a second monitoring/refresh/retry API.
- E1 owns masters and authentication; E2 inventory identity, E3 trust, E4 administrative evidence/outcomes and E6 admission/storage remain foreign owner state.
- No persistent sample/history store, remote agent/daemon, autonomous Connect, mutation authority, fabricated zero, gap interpolation, inferred downtime, undocumented numeric tuning policy or unsupported benchmark claim.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
