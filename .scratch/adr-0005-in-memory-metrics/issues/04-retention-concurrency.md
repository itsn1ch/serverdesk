# 04: Bound memory retention and Collector concurrency

**Status:** ready-for-agent
**Blocked by:** 03: Preserve Freshness and recover Collector channels honestly.
**External blockers:** None beyond the local blocker chain.
**Source:** US-009-AC1, US-009-AC2, US-009-AC3, US-009-AC4, US-009-AC5, US-010-AC1, US-010-AC2, US-010-AC3, US-010-AC4, US-010-AC5.
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Live per-Server history has no more than one hour of memory-only retention, and active Collectors respect the effective configurable bound without starving real terminals. Settings persistence does not become a hidden sample or availability archive.

No new public functions. This ticket privately deepens CollectorBuffer history retention and existing configure/refresh scheduling. Saved effective settings and real raw observations originate in 01; latest normalized series/Collector state comes through 03. No fixed byte/sample cap, undocumented fairness policy or benchmark result is added.

## Types

```rust
Monitoring // from ADR-0006:01
MonitoringSettings // from ADR-0006:01
CollectionSchedule // from ADR-0006:01
Collector // from ADR-0006:01
CollectorBuffer // from ADR-0006:01
MetricHistory // from ADR-0006:01
MetricSeries // from ADR-0006:01
MetricSample // from ADR-0006:01
MonitoringStoreRequest // from ADR-0006:01
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
| INV-5 | Freshness is per metric from local monotonic collection time; error/unavailable/gaps never become new successful observations. | preserve  | Future test `monitoring_monotonic_freshness`; first proof 03                  |
| INV-6 | Recovery retries only Collector channels with exponential backoff under current E1 admission, never master authentication.     | preserve  | Future test `monitoring_loss_backoff_and_disconnect`; first proof 03          |
| INV-7 | Samples reside only in per-Server process-memory rings and temporal retention is at most one hour.                             | establish | Future test `monitoring_retention_and_no_disk_history`; first proof 04        |
| INV-8 | Active Collectors obey the effective positive configurable bound and never own terminal admission.                             | establish | Future test `monitoring_concurrency_and_terminal_interaction`; first proof 04 |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Refresh/collect retention and concurrency; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `Collector` ← 03
- In: `MetricSeries` ← 03
- Out: `MetricHistory` → 05
- Out: `MonitoringSettings` → 05

## Steps

1. Write failing clock-boundary ring retention tests for mixed intervals and disconnected Servers, plus actual SQLite/files/restart inspection proving no persisted samples or metric_samples table.
2. Implement per-Server memory rings and eviction on insertion/read/expiry, keeping temporal maximum one hour without invented sample-count/byte guarantees or restored history.
3. Write and implement actual >bound Collector scheduling, configurable default 10 and bound-reduction joins. Collector permits never gate terminals, SFTP or Operation admission; pending opens still count against reserved Collector slots.
4. Exercise actual VTE interaction while collection reads/backoff run at configured bound; bounded streaming tasks must yield executor time rather than monopolize it. State only observed interactivity/bounds, not an unmeasured latency/capacity benchmark.
5. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-009-AC1`: Given continuous collection, when history is retained for a Server, then its Metric Samples reside in an in-memory ring buffer and retention does not exceed one hour for that Server.
- [ ] `US-009-AC2`: Given multiple Servers and different configured intervals, when samples are retained, then the same maximum temporal retention applies per Server without inventing an undocumented fixed sample count or total byte guarantee.
- [ ] `US-009-AC3`: Given normal collection, errors, disconnect or application shutdown, when storage is inspected, then Metric Samples are not persisted in SQLite or a replacement disk history and no metric_samples table is introduced.
- [ ] `US-009-AC4`: Given application restart, when I open Monitoring before new samples arrive, then prior process-memory history is absent rather than restored as if collection had continued while closed.
- [ ] `US-009-AC5`: Given collection settings stored through the accepted settings mechanism, when they are retained, then settings persistence does not authorize persisting sample values, remote observations or a hidden long-term availability history.
- [ ] `US-010-AC1`: Given default settings, when concurrent Collectors are admitted, then the configured concurrency value defaults to 10 and is user-configurable rather than hard-coded as an immutable Server limit.
- [ ] `US-010-AC2`: Given a configured Collector concurrency bound and more eligible Servers than that bound, when scheduling occurs, then active collection respects the bound rather than creating an unlimited Collector per inventory entry.
- [ ] `US-010-AC3`: Given a Server not presently collecting because collection is not active for it, when I inspect that Server, then the UI does not present missing new samples as actual zero utilization or proven disconnection.
- [ ] `US-010-AC4`: Given interactive terminals while collection runs at the configured concurrency, when collection reads or reconnects, then polling does not starve the terminal sessions and the sessions remain interactive.
- [ ] `US-010-AC5`: Given product performance documentation, when these limits are described, then the default configurable concurrency and benchmark targets are distinguished from a measured guarantee of throughput, capacity or latency; no result is claimed without a benchmark.
- [ ] Pending reservations count against Collector concurrency; reducing the effective bound is not acknowledged until actual surplus channel joins.
- [ ] No SQLite metric_samples table, replacement disk history, raw observation/sample persistence, fixed sample-count/byte limit or unmeasured performance guarantee is introduced.
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
cargo test --test monitoring_gtk monitoring_keyboard_states_and_interactivity -- --exact --ignored
rustfmt --edition 2024 --check src/application/monitoring.rs tests/monitoring_workflows.rs tests/monitoring_gtk.rs tests/support/monitoring.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Earlier public function ownership stays in 01; 02–04 add private behavior only, and 05 owns quiesce. Do not create a second monitoring/refresh/retry API.
- E1 owns masters and authentication; E2 inventory identity, E3 trust, E4 administrative evidence/outcomes and E6 admission/storage remain foreign owner state.
- No persistent sample/history store, remote agent/daemon, autonomous Connect, mutation authority, fabricated zero, gap interpolation, inferred downtime, undocumented numeric tuning policy or unsupported benchmark claim.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.

## Comments

E1 environment, T3 trust eligibility, L1 local trust and S service policy decisions are approved in their canonical Designs. Readiness describes a policy-complete implementation assignment after its listed real producer prerequisites land, not an existing capability or runtime acceptance. No implementation or source acceptance criterion is checked by this update. File/transfer policy 03 remains independently unapproved; no positive file or download admission can be fabricated for downstream scenarios.
