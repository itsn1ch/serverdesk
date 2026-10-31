# 02: Calculate rates only from measured counter continuity

**Status:** needs-info
**Blocked by:** 01: Collect actual observations with persisted validated settings.
**External blockers:** None beyond the local blocker chain.
**Source:** US-001-AC1, US-005-AC1, US-005-AC2, US-005-AC3, US-005-AC4, US-005-AC5, US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-006-AC5.
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

CPU percentage and network throughput become available only after comparable observed counters and measured elapsed time. First samples, discontinuities and unsupported arithmetic remain explicitly unavailable, with no invented replacement or gap sample.

No new public functions. This ticket privately deepens existing Monitoring::refresh/observe normalization and the fixed-loop record handling from 01. Consumer generation and actual records come from 01; counter comparison cannot create an extra connection or mutation authority.

## Types

```rust
MetricSample // from ADR-0006:01
MetricName // from ADR-0006:01
MetricUnit // from ADR-0006:01
MetricValue // from ADR-0006:01
MetricStatus // from ADR-0006:01
RawMetric // from ADR-0006:01
RawObservation // from ADR-0006:01
CounterValue // from ADR-0006:01
CounterBaseline // from ADR-0006:01
CollectorBuffer // from ADR-0006:01
Monitoring // from ADR-0006:01
Collector // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

No new public functions. Privately refine the existing boundaries named in Goal and Steps; do not duplicate an earlier ticket's introducing signature.

## Invariants

| ID    | Statement                                                                                                             | Role      | Proof                                                                         |
| ----- | --------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------- |
| INV-1 | At most one live Collector exists per Server and page navigation never duplicates it.                                 | preserve  | Future test `monitoring_actual_sources_and_single_collector`; first proof 01  |
| INV-2 | Each sample retains actual Server/metric/unit and local collection context; absent or failed reads never become zero. | preserve  | Future test `monitoring_sources_context_and_partial_failures`; first proof 01 |
| INV-3 | Only Monitoring can authorize validated settings writes; Store protocol cannot persist samples.                       | preserve  | Future test `monitoring_settings_sealed_persistence`; first proof 01          |
| INV-4 | Every CPU/network delta has comparable same-continuity counters and positive measured elapsed time.                   | establish | Future test `monitoring_counter_continuity`; first proof 02                   |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Refresh/collect continuity; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `Monitoring` ← 01
- In: `MetricSample` ← 01
- Out: `MetricSample` → 03

## Steps

1. Write failing pure normalization cases for first counters, actual zero, positive measured interval, invalid/non-finite arithmetic, CPU denominator, network per-interface context and changed configured cadence.
2. Implement checked private CPU/network normalization from comparable observed differences, never from configured interval alone; preserve units and separate per-metric/interface baseline timestamps; unrelated records cannot overwrite elapsed-time evidence.
3. Write and satisfy generation/boot/reset/decrease/drop/gap/reconnect cases; invalidate unsupported baselines and require new comparable observations, with no assumed wraparound/overflow rule.
4. Exercise actual remote reboot and new Collector generation in Incus to show no rate spans uncertain continuity, no missing-record sample is synthesized and legitimate evidenced zero remains valid.
5. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a selected Server with successful collection, when I open its Overview or Monitoring context, then hostname, OS, kernel, CPU, memory, disk, network throughput, uptime and service summary are shown from actual observations and the Server context is explicit.
- [ ] `US-005-AC1`: Given the first observation for CPU percentage or another delta-based metric, when it is presented, then its status is UNAVAILABLE rather than a fabricated zero or an immediately available rate.
- [ ] `US-005-AC2`: Given the first network-counter observation, when throughput is requested, then throughput is UNAVAILABLE until there is sufficient comparable counter and elapsed-time evidence.
- [ ] `US-005-AC3`: Given comparable successive observations and a valid measured interval, when a delta-based value is produced, then it is based on those observed differences and that interval rather than the configured interval being treated as proof of elapsed time.
- [ ] `US-005-AC4`: Given no comparable second sample or an invalid interval, when normalization would otherwise produce a delta, then no plausible replacement value is fabricated.
- [ ] `US-005-AC5`: Given a legitimate observed zero after sufficient valid evidence, when the metric is shown, then zero remains a valid observation and is not confused with the UNAVAILABLE or ERROR state.
- [ ] `US-006-AC1`: Given an observed counter reset or a remote reboot that invalidates a previous baseline, when the next delta would span that discontinuity, then it is not published as a valid rate from the old baseline.
- [ ] `US-006-AC2`: Given reconnection without evidence that old and new counters are comparable, when the first new counter observation arrives, then it does not manufacture a value spanning the disconnected period.
- [ ] `US-006-AC3`: Given a baseline invalidated by discontinuity, when insufficient new comparable observations exist, then affected delta metrics remain UNAVAILABLE until valid evidence is available again rather than being forced to zero.
- [ ] `US-006-AC4`: Given a missing sample or dropped record, when later data is handled, then no sample is inserted for the missing observation and no interpolation is presented as actual collected evidence.
- [ ] `US-006-AC5`: Given an ambiguous counter decrease or arithmetic condition whose accepted interpretation is not established, when a value would require assuming wraparound, reset magnitude or overflow behavior, then the workflow withholds the unsupported value rather than inventing that policy.
- [ ] Counter arithmetic is checked and finite; unsupported decrease, wraparound or overflow interpretation withholds the affected value rather than inventing a policy.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test monitoring_workflows monitoring_actual_sources_and_single_collector -- --exact
cargo test --test monitoring_workflows monitoring_sources_context_and_partial_failures -- --exact
cargo test --test monitoring_workflows monitoring_settings_sealed_persistence -- --exact
cargo test --test monitoring_workflows monitoring_counter_continuity -- --exact
rustfmt --edition 2024 --check src/domain/metrics.rs src/application/monitoring.rs src/infrastructure/collector/mod.rs tests/monitoring_workflows.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Earlier public function ownership stays in 01; 02–04 add private behavior only, and 05 owns quiesce. Do not create a second monitoring/refresh/retry API.
- E1 owns masters and authentication; E2 inventory identity, E3 trust, E4 administrative evidence/outcomes and E6 admission/storage remain foreign owner state.
- No persistent sample/history store, remote agent/daemon, autonomous Connect, mutation authority, fabricated zero, gap interpolation, inferred downtime, undocumented numeric tuning policy or unsupported benchmark claim.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
