# 01: Collect actual observations with persisted validated settings

**Status:** needs-info
**Blocked by:** None (can start immediately once external blockers land).
**External blockers:** ADR-0006:01, ADR-0006:02, ADR-0001:03, ADR-0004:06, ADR-0002:01.
**Source:** US-001-AC1, US-001-AC2, US-001-AC3, US-001-AC4, US-001-AC5, US-002-AC1, US-002-AC2, US-002-AC3, US-002-AC4, US-003-AC1, US-003-AC2, US-003-AC3, US-003-AC4, US-003-AC5, US-004-AC1, US-004-AC2, US-004-AC3, US-004-AC4.
**Contract:** `.scratch/adr-0005-in-memory-metrics/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Overview and Monitoring can show actual non-delta Linux/system/service observations for the explicit Server, with documented cadences, real loading/empty/error states and validated persistent settings. CPU/network cards remain unavailable until their evidence calculation lands; this ticket supplies no pretend rate or historical data.

Actual external producers: ADR-0006:02 Runtime::admit → ReadyOwnership; ADR-0006:01 Store::open/request → actual SQLite/settings; ADR-0002:01 Configuration → saved Server identity; ADR-0001:02 ConnectionManager::observe → dedicated initial snapshot/sequence and subsequent events; ADR-0001:03 ConsumerAccess::open/Consumer::into_consumer_parts → actual existing-generation Collector channel. ADR-0004:06 supplies actual structured read-only ServiceRead::Summary semantics; no mutation permission is required or inferred. Runtime must supply E5 its own subscription, not consume E6 recovery receiver. Shared declarations are ADR-0006:01 only; runtime producers listed here must actually exist. Transitively unresolved E3 eligibility/local-trust policy keeps this implementation needs-info; no monitoring policy bypass is introduced.

## Types

```rust
CollectionSchedule // from ADR-0006:01
MonitoringSettings // from ADR-0006:01
RefreshKind // from ADR-0006:01
CollectorPhase // from ADR-0006:01
Collector // from ADR-0006:01
MetricName // from ADR-0006:01
MetricUnit // from ADR-0006:01
MetricValue // from ADR-0006:01
MetricStatus // from ADR-0006:01
MetricSample // from ADR-0006:01
Freshness // from ADR-0006:01
MetricSeries // from ADR-0006:01
MetricHistory // from ADR-0006:01
MonitoringChanged // from ADR-0006:01
Monitoring // from ADR-0006:01
CollectorSpec // from ADR-0006:01
RawMetric // from ADR-0006:01
RawObservation // from ADR-0006:01
CounterValue // from ADR-0006:01
CounterBaseline // from ADR-0006:01
CollectorBuffer // from ADR-0006:01
SettingsCommit // from ADR-0006:01
SettingsWriteView // from ADR-0006:01
MonitoringStoreRequest // from ADR-0006:01
MonitoringStoreResult // from ADR-0006:01
MonitoringMessage // from ADR-0006:01
MonitoringCompletion // from ADR-0006:01
ReadyOwnership // from ADR-0006:01
Store // from ADR-0006:01
Configuration // from ADR-0006:01
Server // from ADR-0006:01
ConsumerAccess // from ADR-0006:01
ConsumerTarget // from ADR-0006:01
Consumer // from ADR-0006:01
ConsumerParts // from ADR-0006:01
Connection // from ADR-0006:01
ConnectionChanged // from ADR-0006:01
ServiceRead // from ADR-0006:01
ServiceSummary // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

SettingsCommit::settings_write_view is a real earlier ADR-0006:01 projection in this canonical module, not owned or reimplemented by this ticket. Monitoring::configure here is the sole later runtime issuer of validated SettingsCommit values; early Store plumbing cannot mint one.

## Functions

### `Monitoring::spawn(ready: E6::ReadyOwnership, store: E6::Store, configuration: E2::Configuration, consumers: E1::ConsumerAccess, connection_events: E6::Subscription<E1::ConnectionView, E1::ConnectionChanged>) -> Monitoring`

- **Intent.** Establish admitted scheduling from actual persisted settings and a dedicated connection feed.
- **Where.** `src/application/monitoring.rs`
- **Reads.** ReadyOwnership, Store, Configuration, ConsumerAccess, connection subscription.
- **Writes.** Monitoring, Collector, CollectorBuffer.
- **Pre.** Current primary admission; actual E1 capability and separate E1 observation subscription exist.
- **Post.** Installs Server-keyed scheduler, settings-load worker and sequenced connection receiver; begins with empty in-process history. Actual connected eligible Servers may collect; initial settings loading/error is observable, never a fabricated successful default read. Defaults apply only to genuinely absent settings.
- **Invariants.** INV-1, INV-2, INV-3.
- **Errors.** Settings schema/read/validation failure is reported and collection admission stays blocked until resolved; channel/source failures remain explicit domain events.
- **Idempotency.** One scheduler per primary, never per page; no restored samples or extra master.

### `Monitoring::configure(&self, settings: MonitoringSettings) -> Result<MonitoringSettings, E6::Failure>`

- **Intent.** Save validated sampling settings and make the requested scheduling effective.
- **Where.** `src/application/monitoring.rs`
- **Reads.** MonitoringSettings, Collector, Store settings.
- **Writes.** SettingsCommit, effective settings, Collector scheduling.
- **Pre.** Positive representable intervals and concurrency; overrides unique and refer to saved Servers.
- **Post.** Serializes configuration changes, validates and constructs private SettingsCommit, joins surplus channels before saving/lowering effective bound; actual Store acknowledgement precedes effective-settings success. Defaults are resource 5s, service 15s, disk 30s, concurrency 10; only settings persist. Worker completions keep owner mailbox serviceable.
- **Invariants.** INV-1, INV-2, INV-3.
- **Errors.** Invalid input changes nothing; stop/join or Store failure leaves prior effective settings and explicit error, never claims lower bound active. Already stopped channels may be rescheduled under prior settings; no sample is restamped.
- **Idempotency.** Identical applied settings reuse state; pending identical change joins; crash reloads actual committed settings, never sample history.

### `Monitoring::refresh(&self, server_id: Uuid, kind: RefreshKind) -> Result<Collector, E6::Failure>`

- **Intent.** Request actual fresh observations for the explicit Server without independently connecting.
- **Where.** `src/application/monitoring.rs`
- **Reads.** Server, current Connection/Capabilities, MonitoringSettings, Collector.
- **Writes.** Collector, RawObservation, MetricSample, CollectorBuffer.
- **Pre.** Saved Server exists; admitted usable master and Collector slot required for channel open.
- **Post.** Reuses one Server Collector or reports WaitingForConnection/WaitingForSlot. A new CollectorSpec carries Server/read schedule; the actual E1 adapter stamps collector_generation with ConsumerParts.id and collected_at with local receipt Instant; E1 open uses ConsumerTarget::Server then into_consumer_parts. Fixed loop reads accepted sources at actual configured cadences and event-based system info; explicit refresh requests actual observations, not retimestamped old values.
- **Invariants.** INV-1, INV-2, INV-3.
- **Errors.** Unavailable source/Capability, denied read or lost connection yields per-metric unavailable/error or lifecycle waiting, no zero and no administrative change; sealed generation/identity mismatch never falls back to Connect.
- **Idempotency.** Overlapping pending reads coalesce; separate explicit requests may collect newer actual samples. Page changes never duplicate channel; after crash no automatic restoration of samples.

### `Monitoring::observe(&self, server_id: Uuid) -> Result<E6::Subscription<MetricHistory, MonitoringChanged>, E6::Failure>`

- **Intent.** Present actual Server history and sequenced changes with independently visible collection state.
- **Where.** `src/application/monitoring.rs`
- **Reads.** CollectorBuffer, Collector, monotonic clock.
- **Writes.** None; only expiry of ephemeral history.
- **Pre.** Saved Server exists; current process monotonic time.
- **Post.** Returns one-boundary snapshot/events, explicit pending state and real gaps. Samples retain Server/metric/unit/remote time/local collection time/status. Per-series Freshness derives from local monotonic age; retention never exceeds one hour. Lag yields ResyncRequired and an actual replacement snapshot, never invented continuity.
- **Invariants.** INV-1, INV-2.
- **Errors.** Invalid Server/closed owner returns explicit failure, not invented empty success. Source errors remain per metric.
- **Idempotency.** Redraw/reopen never changes collected_at or inserts a sample; observation only.

## Invariants

| ID    | Statement                                                                                                             | Role      | Proof                                                                         |
| ----- | --------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------- |
| INV-1 | At most one live Collector exists per Server and page navigation never duplicates it.                                 | establish | Future test `monitoring_actual_sources_and_single_collector`; first proof 01  |
| INV-2 | Each sample retains actual Server/metric/unit and local collection context; absent or failed reads never become zero. | establish | Future test `monitoring_sources_context_and_partial_failures`; first proof 01 |
| INV-3 | Only Monitoring can authorize validated settings writes; Store protocol cannot persist samples.                       | establish | Future test `monitoring_settings_sealed_persistence`; first proof 01          |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Configure and refresh/collect; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `ReadyOwnership` ← external
- In: `Store` ← external
- In: `Configuration` ← external
- In: `ConsumerAccess` ← external
- In: `ConnectionChanged` ← external
- In: `ServiceRead` ← external
- Out: `Monitoring` → 02
- Out: `Monitoring` → 03
- Out: `Monitoring` → 05
- Out: `MetricSample` → 02

## Steps

1. Write failing actual fixed-loop observation/cadence/Server-context tests against Incus Linux/systemd, actual SQLite settings and actual shared master, plus GTK initial loading/empty/partial-failure tests.
2. Implement Monitoring::spawn/configure/refresh/observe using the actual ready capability and dedicated E1 subscription. Reuse SettingsCommit::settings_write_view from ADR-0006:01. Construct SettingsCommit only after validation; generic Store accepts the sealed commit, not arbitrary settings or sample payload.
3. Implement one Server-keyed Collector, immutable Server/schedule CollectorSpec and E1-channel-UUID-stamped RawObservation, fixed accepted-source loop and canonical E4 ServiceRead::Summary. Open only ConsumerTarget::Server and consume the actual typed event receiver. Require collector_generation == ConsumerParts.id and local E1 receipt Instant; Monitoring never invents the collection epoch.
4. Schedule resource 5s/service 15s/disk 30s defaults, changed overrides and event-based system information/manual refresh; coalesce pending explicit refresh without restamping old data. Keep first CPU/network counters unavailable and do not render invented cards.
5. Keep the owner mailbox live during settings load/save/channel open/stop workers. Validate Server/generation/record context; publish actual source errors/unavailability without poisoning unrelated metrics. Confirm settings failure is not a fabricated default success.
6. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a selected Server with successful collection, when I open its Overview or Monitoring context, then hostname, OS, kernel, CPU, memory, disk, network throughput, uptime and service summary are shown from actual observations and the Server context is explicit.
- [ ] `US-001-AC2`: Given a Collector reading Linux resource observations, when records are obtained, then the accepted sources are `/proc/stat`, `/proc/meminfo`, `/proc/net/dev`, `/proc/loadavg`, `/proc/uptime`, filesystem statistics and `uname`, rather than fabricated substitutes for inaccessible data.
- [ ] `US-001-AC3`: Given available systemd read Capability, when the service summary is refreshed, then it derives from structured systemd D-Bus observations using the read-only service mechanisms owned by the ADR-0004 epic, not human-oriented table parsing or sudo mutation privileges.
- [ ] `US-001-AC4`: Given a metric source unavailable on this Server, when collection reports the affected metric, then its absence is distinguished from a legitimate observed zero and other successfully observed metrics remain usable.
- [ ] `US-001-AC5`: Given a fleet view and a Server never contacted, when summaries are displayed, then no metric or Collector gap is used to label it offline; Connection State and Trust State remain independently owned observations rather than inferred metric health.
- [ ] `US-002-AC1`: Given a connected Server whose collection is scheduled, when monitoring begins, then exactly one long-lived Collector channel for that Server runs a fixed loop emitting structured records over the existing system OpenSSH master.
- [ ] `US-002-AC2`: Given that Collector is already active, when I switch between Overview and Monitoring or revisit the Server, then the existing Collector is consumed rather than a second Collector for that Server being started.
- [ ] `US-002-AC3`: Given two Server entries with the same Connection Identity, when both have scheduled collection, then their per-Server data remains attributable to the correct Server and their channels use the identity's one connection-owner master rather than creating a second master.
- [ ] `US-002-AC4`: Given an unconnected Server or a trust/configuration gate that prevents connection, when Monitoring is opened, then collection cannot bypass that gate or create its own alternate authentication, Host Key acceptance or SSH configuration route.
- [ ] `US-003-AC1`: Given default collection settings, when CPU, memory and network observations are scheduled, then their default interval is 5 seconds.
- [ ] `US-003-AC2`: Given default collection settings, when service observations are scheduled, then their default interval is 15 seconds.
- [ ] `US-003-AC3`: Given default collection settings, when disk observations are scheduled, then their default interval is 30 seconds.
- [ ] `US-003-AC4`: Given a successful connection or my explicit system-information refresh, when system information is collected, then collection occurs at that event rather than implying it shares a periodic five-second default.
- [ ] `US-003-AC5`: Given configured collection intervals or the explicit refresh-metrics action for the selected Server, when scheduling uses those settings or that action, then the actual requested cadence or refresh applies to that Server's collection and does not imply that older samples became new observations.
- [ ] `US-004-AC1`: Given a collected Metric Sample, when the workflow exposes it, then it contains the accepted fields server_id, metric_name, value, unit, remote_observed_at, collected_at and status rather than a value detached from its context.
- [ ] `US-004-AC2`: Given samples for different metrics or Servers, when they are normalized and displayed, then their metric identity, Server identity and units remain distinguishable and a missing value does not become another metric's last observation.
- [ ] `US-004-AC3`: Given a remote observation timestamp, when I inspect sample timing, then remote_observed_at is informative only and is not used as the authoritative Freshness clock.
- [ ] `US-004-AC4`: Given local collection, when collected_at is assigned and later used, then it represents the local monotonic collection clock rather than a remote wall-clock value or a persisted timestamp reused after restart.
- [ ] Every RawObservation carries the matching Server plus collector_generation == actual E1 ConsumerParts.id and collected_at stamped from local E1 receipt Instant, never remote time; rejected stale/foreign records cannot create samples. Replacement channels always have a new epoch.
- [ ] Only Monitoring creates SettingsCommit after validation; generic Store cannot persist unvalidated settings or any raw/sample/history record. Actual SQLite read/save/schema failures surface honestly, not as fabricated default success.
- [ ] CPU/network delta values remain unavailable until 02 supplies comparable measured evidence; this slice does not render pretend data for an unimplemented card.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test monitoring_workflows monitoring_actual_sources_and_single_collector -- --exact
cargo test --test monitoring_workflows monitoring_sources_context_and_partial_failures -- --exact
cargo test --test monitoring_workflows monitoring_settings_sealed_persistence -- --exact
cargo test --test monitoring_gtk monitoring_keyboard_states_and_interactivity -- --exact --ignored
rustfmt --edition 2024 --check src/domain/metrics.rs src/application/monitoring.rs src/infrastructure/collector/mod.rs src/ui/monitoring.rs tests/monitoring_workflows.rs tests/monitoring_gtk.rs tests/support/monitoring.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Earlier public function ownership stays in 01; 02–04 add private behavior only, and 05 owns quiesce. Do not create a second monitoring/refresh/retry API.
- E1 owns masters and authentication; E2 inventory identity, E3 trust, E4 administrative evidence/outcomes and E6 admission/storage remain foreign owner state.
- No persistent sample/history store, remote agent/daemon, autonomous Connect, mutation authority, fabricated zero, gap interpolation, inferred downtime, undocumented numeric tuning policy or unsupported benchmark claim.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
