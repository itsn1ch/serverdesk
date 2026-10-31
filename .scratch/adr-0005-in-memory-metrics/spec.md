# Live Metrics with Bounded In-Memory Retention
Status: ready-for-agent
Source: Accepted ADR-0005; connected ADR-0001, ADR-0002, ADR-0003 and ADR-0006; docs/spec.md §§3, 4.9, 6, 7, 8, 9, 12–16 and 18; GLOSSARY.md.

## Problem Statement

A Developer managing personal Linux Servers needs live observations with honest Freshness, not an apparent monitoring history that implies collection while the desktop application was closed.
Persisting five-second samples across many Servers would grow local storage while still leaving unavoidable gaps whenever the application or connection is absent.
Remote clock changes, a first counter sample, counter resets, reboot and connection loss can all make an attractive numerical chart misleading.
A collection failure must not become a fabricated zero, an inferred outage or proof that a service Operation succeeded.
Monitoring must remain useful without starving interactive terminals or blocking GTK.

## Solution

Use one long-lived Collector channel per connected Server, consuming the existing system OpenSSH ControlMaster and emitting structured records from a fixed loop script.
Use the accepted Linux and systemd sources with configurable sampling intervals and a configurable concurrent-Collector setting whose default is 10.
Keep Metric Samples only in per-Server memory ring buffers with at most one hour of retention.
Represent values and their statuses separately, using local monotonic collection time for Freshness and remote observation time for information only.
Require comparable observations for delta-based values and show unavailable or uncertain data honestly across startup, reconnect, counter reset and reboot.
On channel loss, mark existing data stale and use exponential reconnect backoff while respecting the connection owner's lifecycle; deliberate disconnect and application exit do not create a competing connection loop.
Expose actual current, stale, unavailable, error, loading and empty states in Overview and Monitoring without inventing historical availability.

## User Stories

### Observable data and collection

1. **US-001: See actual observations for an explicit Server**
   - **Story:** As a Developer managing personal Linux Servers, I want the selected Server's actual system observations, so that I can understand its current resources without confusing data between Servers.
   - **Basis:** Derived: docs/spec.md §§3, 4.9, 7.1 and 8.3; GLOSSARY.md Server and Metric Sample.
   - **Acceptance criteria:**
     - **US-001-AC1:** Given a selected Server with successful collection, when I open its Overview or Monitoring context, then hostname, OS, kernel, CPU, memory, disk, network throughput, uptime and service summary are shown from actual observations and the Server context is explicit.
     - **US-001-AC2:** Given a Collector reading Linux resource observations, when records are obtained, then the accepted sources are `/proc/stat`, `/proc/meminfo`, `/proc/net/dev`, `/proc/loadavg`, `/proc/uptime`, filesystem statistics and `uname`, rather than fabricated substitutes for inaccessible data.
     - **US-001-AC3:** Given available systemd read Capability, when the service summary is refreshed, then it derives from structured systemd D-Bus observations using the read-only service mechanisms owned by the ADR-0004 epic, not human-oriented table parsing or sudo mutation privileges.
     - **US-001-AC4:** Given a metric source unavailable on this Server, when collection reports the affected metric, then its absence is distinguished from a legitimate observed zero and other successfully observed metrics remain usable.
     - **US-001-AC5:** Given a fleet view and a Server never contacted, when summaries are displayed, then no metric or Collector gap is used to label it offline; Connection State and Trust State remain independently owned observations rather than inferred metric health.

2. **US-002: Use one Collector over the existing connection**
   - **Story:** As a Developer managing personal Linux Servers, I want collection to share the connection already established for my Server, so that visiting different pages does not create duplicate background collectors or a separate trust path.
   - **Basis:** Derived: ADR-0001, ADR-0002, ADR-0003 and docs/spec.md §§4.7, 7.1 and 12.3.
   - **Acceptance criteria:**
     - **US-002-AC1:** Given a connected Server whose collection is scheduled, when monitoring begins, then exactly one long-lived Collector channel for that Server runs a fixed loop emitting structured records over the existing system OpenSSH master.
     - **US-002-AC2:** Given that Collector is already active, when I switch between Overview and Monitoring or revisit the Server, then the existing Collector is consumed rather than a second Collector for that Server being started.
     - **US-002-AC3:** Given two Server entries with the same Connection Identity, when both have scheduled collection, then their per-Server data remains attributable to the correct Server and their channels use the identity's one connection-owner master rather than creating a second master.
     - **US-002-AC4:** Given an unconnected Server or a trust/configuration gate that prevents connection, when Monitoring is opened, then collection cannot bypass that gate or create its own alternate authentication, Host Key acceptance or SSH configuration route.
     - **US-002-AC5:** Given application exit, when lifecycle cleanup runs, then the Collector and its owned collection work stop; no collection continues as a background monitoring daemon after the application closes.

3. **US-003: Control the accepted sampling schedule**
   - **Story:** As a Developer managing personal Linux Servers, I want the documented sampling defaults to be configurable, so that collection frequency can fit my Servers without presenting every metric as sampled at the same rate.
   - **Basis:** Derived: docs/spec.md §§3.3 and 7.1.
   - **Acceptance criteria:**
     - **US-003-AC1:** Given default collection settings, when CPU, memory and network observations are scheduled, then their default interval is 5 seconds.
     - **US-003-AC2:** Given default collection settings, when service observations are scheduled, then their default interval is 15 seconds.
     - **US-003-AC3:** Given default collection settings, when disk observations are scheduled, then their default interval is 30 seconds.
     - **US-003-AC4:** Given a successful connection or my explicit system-information refresh, when system information is collected, then collection occurs at that event rather than implying it shares a periodic five-second default.
     - **US-003-AC5:** Given configured collection intervals or the explicit refresh-metrics action for the selected Server, when scheduling uses those settings or that action, then the actual requested cadence or refresh applies to that Server's collection and does not imply that older samples became new observations.

4. **US-004: Retain the accepted meaning of each Metric Sample**
   - **Story:** As a Developer managing personal Linux Servers, I want each displayed observation to retain its identity, unit and collection context, so that values are not mixed across Servers, metrics or clocks.
   - **Basis:** Derived: docs/spec.md §7.2; GLOSSARY.md Metric Sample and Freshness.
   - **Acceptance criteria:**
     - **US-004-AC1:** Given a collected Metric Sample, when the workflow exposes it, then it contains the accepted fields server_id, metric_name, value, unit, remote_observed_at, collected_at and status rather than a value detached from its context.
     - **US-004-AC2:** Given samples for different metrics or Servers, when they are normalized and displayed, then their metric identity, Server identity and units remain distinguishable and a missing value does not become another metric's last observation.
     - **US-004-AC3:** Given a remote observation timestamp, when I inspect sample timing, then remote_observed_at is informative only and is not used as the authoritative Freshness clock.
     - **US-004-AC4:** Given local collection, when collected_at is assigned and later used, then it represents the local monotonic collection clock rather than a remote wall-clock value or a persisted timestamp reused after restart.

### Honest values and Freshness

5. **US-005: Wait for evidence before displaying a delta**
   - **Story:** As a Developer managing personal Linux Servers, I want rate and percentage metrics calculated only from comparable samples, so that the first observation does not invent CPU usage or network throughput.
   - **Basis:** Derived: docs/spec.md §§7.4, 16.1 and 18.
   - **Acceptance criteria:**
     - **US-005-AC1:** Given the first observation for CPU percentage or another delta-based metric, when it is presented, then its status is UNAVAILABLE rather than a fabricated zero or an immediately available rate.
     - **US-005-AC2:** Given the first network-counter observation, when throughput is requested, then throughput is UNAVAILABLE until there is sufficient comparable counter and elapsed-time evidence.
     - **US-005-AC3:** Given comparable successive observations and a valid measured interval, when a delta-based value is produced, then it is based on those observed differences and that interval rather than the configured interval being treated as proof of elapsed time.
     - **US-005-AC4:** Given no comparable second sample or an invalid interval, when normalization would otherwise produce a delta, then no plausible replacement value is fabricated.
     - **US-005-AC5:** Given a legitimate observed zero after sufficient valid evidence, when the metric is shown, then zero remains a valid observation and is not confused with the UNAVAILABLE or ERROR state.

6. **US-006: Preserve uncertainty across counter discontinuities**
   - **Story:** As a Developer managing personal Linux Servers, I want resets, reboot and missing counter continuity to be visible as uncertainty, so that a chart cannot show an invented spike or negative rate across an invalid comparison.
   - **Basis:** Derived: docs/spec.md §§7.2–7.4, 16.1 and 18; the no-fabricated-values invariant applied to counter continuity.
   - **Acceptance criteria:**
     - **US-006-AC1:** Given an observed counter reset or a remote reboot that invalidates a previous baseline, when the next delta would span that discontinuity, then it is not published as a valid rate from the old baseline.
     - **US-006-AC2:** Given reconnection without evidence that old and new counters are comparable, when the first new counter observation arrives, then it does not manufacture a value spanning the disconnected period.
     - **US-006-AC3:** Given a baseline invalidated by discontinuity, when insufficient new comparable observations exist, then affected delta metrics remain UNAVAILABLE until valid evidence is available again rather than being forced to zero.
     - **US-006-AC4:** Given a missing sample or dropped record, when later data is handled, then no sample is inserted for the missing observation and no interpolation is presented as actual collected evidence.
     - **US-006-AC5:** Given an ambiguous counter decrease or arithmetic condition whose accepted interpretation is not established, when a value would require assuming wraparound, reset magnitude or overflow behavior, then the workflow withholds the unsupported value rather than inventing that policy.

7. **US-007: Distinguish Freshness from remote clock and error state**
   - **Story:** As a Developer managing personal Linux Servers, I want each metric's Freshness and collection state to remain explicit, so that old or failed observations are not mistaken for live data.
   - **Basis:** Derived: docs/spec.md §§3.4, 7.4 and 18; GLOSSARY.md Freshness.
   - **Acceptance criteria:**
     - **US-007-AC1:** Given a displayed metric, when I inspect it, then its Freshness is visible for that metric rather than implied only by a page-wide connected indicator.
     - **US-007-AC2:** Given an adjustment to the remote clock, when Freshness is evaluated, then it is still based on elapsed local monotonic time since collection and is not made newer, older or negative by the remote adjustment.
     - **US-007-AC3:** Given collection-state changes, when samples are exposed, then FRESH, STALE, UNAVAILABLE and ERROR remain distinct observable statuses rather than a single valid/invalid flag.
     - **US-007-AC4:** Given a failed metric read, when its error is shown, then ERROR is never replaced by a numerical zero or a silent reuse of an earlier value as FRESH.
     - **US-007-AC5:** Given a retained last-known value that is no longer current, when it remains visible, then it is identified as such with its real Freshness and does not become a new successful observation merely because the UI redraws or I reopen the page.

8. **US-008: Handle connection loss without inventing availability**
   - **Story:** As a Developer managing personal Linux Servers, I want collection loss to preserve last-known observations and reconnect carefully, so that missing data is not represented as remote downtime.
   - **Basis:** Derived: ADR-0001, ADR-0005 and docs/spec.md §§3.4, 4.7, 7.4 and 16.4.
   - **Acceptance criteria:**
     - **US-008-AC1:** Given existing collected data, when its Collector channel disconnects, then that data becomes STALE rather than continuing to be presented as FRESH.
     - **US-008-AC2:** Given an unexpected collection-channel failure while the connection lifecycle permits recovery, when collection is retried, then reconnect uses exponential backoff rather than an uncontrolled immediate retry loop; no undocumented initial delay, cap or attempt limit is asserted here.
     - **US-008-AC3:** Given an explicit Server Disconnect, when the connection owner closes child channels and its master, then the Collector closes with them and does not independently reconnect against that deliberate disconnect.
     - **US-008-AC4:** Given an interval without collection because of loss, disconnection or application closure, when data is displayed after return, then that gap is not filled with samples, zero values or an inferred DOWN period.
     - **US-008-AC5:** Given a new successful non-delta observation after reconnect, when it is shown, then its Freshness reflects that new collection; recovery of one metric does not relabel other old or unavailable metrics as current.

### Resource bounds and presentation

9. **US-009: Keep only the permitted live history**
   - **Story:** As a Developer managing personal Linux Servers, I want bounded live history held only in memory, so that the application does not grow a misleading persistent metrics database.
   - **Basis:** Derived: ADR-0005 and docs/spec.md §§7.3, 12.3 and 13.
   - **Acceptance criteria:**
     - **US-009-AC1:** Given continuous collection, when history is retained for a Server, then its Metric Samples reside in an in-memory ring buffer and retention does not exceed one hour for that Server.
     - **US-009-AC2:** Given multiple Servers and different configured intervals, when samples are retained, then the same maximum temporal retention applies per Server without inventing an undocumented fixed sample count or total byte guarantee.
     - **US-009-AC3:** Given normal collection, errors, disconnect or application shutdown, when storage is inspected, then Metric Samples are not persisted in SQLite or a replacement disk history and no metric_samples table is introduced.
     - **US-009-AC4:** Given application restart, when I open Monitoring before new samples arrive, then prior process-memory history is absent rather than restored as if collection had continued while closed.
     - **US-009-AC5:** Given collection settings stored through the accepted settings mechanism, when they are retained, then settings persistence does not authorize persisting sample values, remote observations or a hidden long-term availability history.

10. **US-010: Bound collection without starving interactive work**
    - **Story:** As a Developer managing personal Linux Servers, I want configurable bounded collection, so that monitoring many Servers does not consume unbounded resources or interfere with my terminal work.
    - **Basis:** Derived: docs/spec.md §§7.4, 12 and 15.
    - **Acceptance criteria:**
      - **US-010-AC1:** Given default settings, when concurrent Collectors are admitted, then the configured concurrency value defaults to 10 and is user-configurable rather than hard-coded as an immutable Server limit.
      - **US-010-AC2:** Given a configured Collector concurrency bound and more eligible Servers than that bound, when scheduling occurs, then active collection respects the bound rather than creating an unlimited Collector per inventory entry.
      - **US-010-AC3:** Given a Server not presently collecting because collection is not active for it, when I inspect that Server, then the UI does not present missing new samples as actual zero utilization or proven disconnection.
      - **US-010-AC4:** Given interactive terminals while collection runs at the configured concurrency, when collection reads or reconnects, then polling does not starve the terminal sessions and the sessions remain interactive.
      - **US-010-AC5:** Given product performance documentation, when these limits are described, then the default configurable concurrency and benchmark targets are distinguished from a measured guarantee of throughput, capacity or latency; no result is claimed without a benchmark.

11. **US-011: Understand loading, empty, stale and failed monitoring accessibly**
    - **Story:** As a Developer managing personal Linux Servers, I want monitoring views to explain their actual data state and available actions, so that an empty chart or a failure does not require guessing what happened.
    - **Basis:** Derived: docs/spec.md §§2.2, 3.3–3.4, 14 and 18.
    - **Acceptance criteria:**
      - **US-011-AC1:** Given initial collection is in progress and no response has arrived, when I open the relevant view, then a loading state is shown instead of an invented zero series or a completed empty result.
      - **US-011-AC2:** Given no samples have been collected or a successful query genuinely has no applicable data, when the view is shown, then it provides an empty or unavailable explanation appropriate to that condition rather than reporting a collection error that did not occur.
      - **US-011-AC3:** Given a collection or unsupported-Capability error, when it is presented, then it identifies what failed, the affected Server or metric, that this read-only workflow did not itself perform an administrative change, and the available actions; Operation details are linked only when an actual related Operation exists.
      - **US-011-AC4:** Given keyboard-only interaction, when I select the Server, navigate metrics or invoke refresh, then those controls and the meaningful Freshness/error information remain accessible without pointer-only interactions.
      - **US-011-AC5:** Given a stale chart with new collection pending, when I view it, then stale retained observations, loading of new observations and actual errors remain distinguishable rather than removing the known history or disguising it as live data.

12. **US-012: Keep state owners and administrative evidence separate**
    - **Story:** As a Developer managing personal Linux Servers, I want monitoring to remain responsive and observational, so that collecting statistics cannot silently mutate a Server or substitute for a verified administrative result.
    - **Basis:** Derived: docs/spec.md §§6.1, 8, 9, 12 and 18; connected ADR-0004 and ADR-0006.
    - **Acceptance criteria:**
      - **US-012-AC1:** Given slow collection or an unreachable Server, when I navigate or use other application controls, then the GTK main thread remains responsive while Tokio handles Collector and SSH work without accessing GTK widgets.
      - **US-012-AC2:** Given a newly collected result, when presentation is updated, then it arrives through channels scheduled on glib::MainContext, with GTK owning presentation, the application layer owning scheduling and workflow, the connection manager owning the master and the Collector ring buffer owning live Metric Samples.
      - **US-012-AC3:** Given Services, SFTP or an administrative Operation sharing the Server connection, when monitoring runs alongside them, then collection remains a separate read-only consumer rather than putting file payload into the Collector protocol or granting itself mutation authority.
      - **US-012-AC4:** Given a service Operation awaiting verification, when a live service summary changes, then that summary alone does not substitute for the ADR-0004 epic's operation-specific Verifier, stabilization policy or Reconciliation evidence.

## Implementation Decisions

- The accepted architecture is planned future behavior; a live Collector is not present in the current application.
- System OpenSSH and the connection manager's existing ControlMaster provide collection transport under generated configuration and the Host Key gate.
- One long-lived channel per Server runs a fixed loop with structured output; repeated view navigation is not a reason to create another Collector.
- Collection sources are the accepted Linux proc observations, filesystem statistics, uname and read-only systemd D-Bus observations.
- The default periodic intervals are 5 seconds for CPU, memory and network, 15 seconds for services and 30 seconds for disk; system information is collected on connection and manual refresh.
- Intervals and concurrent collection are configurable, with the accepted concurrency default of 10.
- Metric Sample retains server_id, metric_name, value, unit, remote_observed_at, collected_at and status.
- Local monotonic collection time determines Freshness; remote observation time is informational only.
- Delta-based metrics require comparable evidence, and the first sample is UNAVAILABLE.
- Discontinuities and unsupported arithmetic interpretations do not justify fabricated rates, zeros or gap-filled samples.
- Metric statuses remain FRESH, STALE, UNAVAILABLE and ERROR; a collection error is not numerical zero.
- Each Server's live ring buffer retains at most one hour in memory, without SQLite metric_samples or another persistent sample store.
- Settings and other accepted local data may remain persisted under their existing ownership; that does not relax the no-persistent-metrics decision.
- Channel loss makes prior data stale and collection recovery uses exponential backoff under the connection owner's lifecycle.
- The application layer owns scheduling and workflow, the connection manager owns master processes, the Collector buffer owns live samples and GTK owns presentation.
- Tokio performs asynchronous work and messages are scheduled through glib::MainContext rather than accessing GTK widgets.
- Collection is observational; administrative mutations and their proof remain the Operation engine's responsibility.

## Testing Decisions

- These are planned tests, not executed checks or evidence that monitoring exists; no implementation or test harness is added by this specification task.
- The approved primary seam is the presentation-facing workflow used by GTK, backed by real system OpenSSH and actual Linux/systemd sources in isolated Incus/LXD Ubuntu 24.04 containers with systemd 255.
- Use actual SQLite for the invariant that accepted local data can persist while metric_samples and sample values do not; do not substitute a mock repository that can never write metrics.
- Observe one Collector channel per Server and master reuse through repeated Overview/Monitoring navigation and same-identity Server aliases.
- Exercise each accepted source and default cadence, changed configured intervals, connect-time system information and explicit manual refresh without asserting an undocumented scheduling tolerance.
- Narrow pure tests cover sample normalization, unit identity, local-monotonic Freshness behavior, CPU/network delta calculation, unavailable first samples and invalid comparison evidence.
- Exercise remote clock changes without changing the basis of Freshness.
- Exercise real disconnect/reconnect and remote reboot, checking stale data, no downtime inference, no generated gap samples and no fabricated rate across a broken baseline.
- Supply counter-reset, ambiguous-decrease and missing-record cases at the normalization boundary, checking the no-fabricated-value invariant rather than asserting an undocumented overflow algorithm.
- Make required remote sources unavailable or unreadable and observe appropriate error/unavailable states without replacing them with zero or erasing other valid metrics.
- Exercise configured concurrency with more eligible Servers than the setting and active terminal sessions; check the bound and real interactive terminal behavior rather than just a mocked scheduler's call count.
- Test at-most-one-hour per-Server retention at the buffer/time boundary without requiring a literal one-hour wall-clock test and without inferring an unaccepted sample-count or byte limit.
- Exit and restart the application to demonstrate Collector shutdown and absence of restored metric history.
- Actual GTK smoke tests cover keyboard navigation, explicit Server context, refresh, per-metric Freshness, loading, genuine empty, error, unavailable and stale states while collection waits.
- Coordinate actual connection lifecycle cases with the ADR-0001 epic and single-owner cleanup with the ADR-0006 epic; no Collector-specific path may resurrect a deliberate disconnect.
- Performance targets, including the default concurrency context, need benchmarks before being reported as measured; this specification supplies no throughput or latency results.
- No metric failure-injection case requires touching the host filesystem; any storage failure fixture shared with the other epics must remain inside isolated container storage.

## Out of Scope

- Persistent Metric Samples, a SQLite metric_samples table, a disk cache of sample history and recovery of a previous process's in-memory chart history.
- Long-term availability monitoring, uptime-percentage guarantees, inferred downtime during gaps and Gatus integration, which remains Milestone 3 or an external tool's responsibility.
- Remote monitoring agents, a daemon that outlives the desktop application and a second SSH transport or trust implementation.
- Invented staleness thresholds, reconnect delay constants, retry caps, fairness algorithms, sample-count limits, byte budgets or arithmetic wraparound rules.
- A performance guarantee that a machine or network can sustain 10 Collectors, 20 masters or any target latency.
- Implementing the full terminal, file manager, service management, Host Key Store or Operation engine; monitoring only consumes their accepted boundaries.
- Using a periodic service summary as proof of an administrative mutation's success.
- Multi-server administrative operations, hosting monitoring extensions and automatic changes made in response to metric values.

## Further Notes

### Readiness and unspecified tuning

- The accepted nonpersistent metrics behavior has observable safety and lifecycle criteria without choosing new interfaces or new product scope.
- Freshness threshold values, initial/capped reconnect delays, cancellation timing, interval UI layout and scheduling fairness beyond the explicit no-terminal-starvation invariant are not specified numerically by the accepted sources.
- Those are implementation-tuning questions, not permission to invent defaults in this specification and not blockers to the stated no-fabricated-value or read-only behavior.
- Counter overflow/wraparound semantics and any policy for preserving a delta baseline across a gap are not specified; the acceptance boundary is to withhold values requiring unsupported assumptions rather than claim an unaccepted algorithm.
- No concrete publication blocker has been identified for the in-memory metrics epic itself; its implementation still depends on the connected lifecycle and trust contracts.

### Cross-epic assumptions

- The ADR-0001 epic owns master/channel lifecycle and Connection State; this epic requests collection recovery without replacing normal connection/disconnect ownership.
- The ADR-0002 and ADR-0003 epics provide generated configuration and a satisfied trust gate; unresolved trust policy must not be worked around by the Collector.
- The ADR-0004 epic provides structured read-only service observations and owns mutation evidence; its missing Verifier predicates do not justify fabricated monitoring values and do not block read-only Linux metrics.
- The ADR-0006 epic supplies the single owning application process and resource lifecycle; its exclusive mutation locks do not turn read-only collection into administrative work.
- SFTP, terminal and Collector channels share a master but not payload protocols or state ownership.
- No cross-epic story IDs are used as local implementation dependencies.

### Source coverage and delivery caveats

- ADR-0005 and §§7.3 and 13 prohibit persistent samples and retain a maximum one-hour in-memory history.
- §§3 and 7 specify actual observations, every sampling default, accepted sources, Metric Sample fields, Freshness states, local monotonic time, delta initialization, backoff, bounded concurrency and lifecycle.
- §§4.9 and 8 ground detected systemd Capability and structured read-only service integration; §§6 and 9 establish the separate payload and mutation-evidence boundaries rather than expanding file management scope.
- §§12–16 and 18 ground ownership, threading, persistence exclusions, actionable errors, benchmark caveats, real test fixtures and visible loading/empty/error/stale behavior.
- GLOSSARY.md supplies Server, Connection Identity, Connection State, Trust State, Metric Sample, Freshness, Collector and Capability terminology.
- The repository currently contains only a dependency-free Rust greeting program; no collection, GTK behavior, benchmarks or tests above are claimed as already implemented or passed.
