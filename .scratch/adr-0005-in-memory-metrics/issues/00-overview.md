# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`.

## Files

These files describe planned implementation. At planning base `5b62b8d`, the executable only prints a greeting. ADR-0006:01 introduces shared declarations in their canonical owner modules; these tickets implement the workflows without redefining foreign records.

```
src/
├── application/
│   └── monitoring.rs [01, 02, 03, 04, 05] settings authority, scheduler, channel recovery and memory rings
├── domain/
│   └── metrics.rs [01, 02, 03] Metric Sample, status, units and continuity normalization
├── infrastructure/
│   └── collector/
│       └── mod.rs [01, 02] fixed accepted-source loop and actual structured records
└── ui/
    └── monitoring.rs [01, 03, 05] GTK-only explicit Server/Freshness and actionable state presentation
tests/
├── monitoring_gtk.rs [01, 04, 05] future actual accessible GTK and terminal interaction journeys
├── monitoring_workflows.rs [01, 02, 03, 04, 05] future actual Linux/OpenSSH/SQLite and pure clock/counter proofs
└── support/
    └── monitoring.rs [01, 04, 05] isolated actual monitoring fixtures without fake observations
```

## Architecture

```
E6 -- ReadyOwnership/Store --> Monitoring [01]
E1 -- ConnectionChanged/ConsumerAccess --> Monitoring -- CollectorSpec --> E1 shared master [01]
E4 read semantics -- ServiceRead/ServiceSummary --> fixed Collector loop [01]
Collector loop -- RawObservation --> normalization -- MetricSample --> memory CollectorBuffer [01,02,04]
Monitoring -- SettingsCommit/SettingsWriteView --> E6 Store [01]
Monitoring -- MetricHistory/MonitoringChanged/Freshness --> GTK [01,03,05]
E6 stopping -- Monitoring::quiesce --> E1 joined ChannelEnd --> Stopped [05]
```

Monitoring owns validated settings/scheduling; each Collector owns one memory buffer, E1 owns masters and GTK owns widgets.
Accepted-source normalization validates timing/continuity; Store receives only sealed settings, never samples or raw records.
ADR-0006:01 supplies common declarations and pure Store projections, not E5 runtime authority; actual E1 channels, dedicated connection feed and E4 structured reads are external prerequisites.

## Order

- Configure and actual refresh/collect: external ADR-0001:03 / ADR-0004:06 / ADR-0006:02 → 01.
- Counter evidence then Freshness/recovery: 01 → 02 → 03.
- Retention/concurrency then presentation/shutdown: 03 → 04 → 05.
- No local work tickets run in parallel; all refine the same owner state and preserve the earlier proof chain.

- 01: Collect actual observations with persisted validated settings; local blockers none.
- 02: Calculate rates only from measured counter continuity; local blockers 01.
- 03: Preserve Freshness and recover Collector channels honestly; local blockers 01, 02.
- 04: Bound memory retention and Collector concurrency; local blockers 03.
- 05: Present accessible monitoring states and join shutdown; local blockers 01, 04.

## In plain words

You see the selected Server's real readings, collected through the connection you already opened. You can change how often they are collected or ask for a fresh reading without creating another login. CPU and network rates appear only after enough trustworthy measurements, not as made-up zeros. Each reading shows how current it is, and old readings stay visibly old when collection fails. Short live history stays only in memory and disappears when the app closes. Collection is limited so terminals remain usable, with no claim that a performance target has already been measured. You can navigate, refresh and understand errors with the keyboard while the app keeps responding. Disconnect and exit stop collection rather than quietly reconnecting in the background.

## Comments

E1, T3, L1 and S decisions are approved. Numbered implementation assignments are ready-for-agent after their actual listed producer prerequisites, not runtime-complete. T3 permits accepted VERIFIED, ADOPTED and explicitly committed UNVERIFIED for Direct use, VERIFIED only for every Jump role, and blocks NONE/MISMATCH in both. Fresh preflight, original binding and atomic SQLite snapshot remain required. File-policy 03 stays independently unresolved; its downstream enabled scenarios cannot be claimed delivered by read-only transport or monitoring.
