# SQLite Host Key Store and Explicit Trust
Status: needs-info
Source: ADR-0003; connected ADR-0001, ADR-0002, ADR-0004, ADR-0006; product specification §§4.3–4.10, 9, 12, 13, 14, 16, 18; GLOSSARY.md.

## Problem Statement

A Developer managing personal Linux Servers needs to know which Host Key was accepted, on what evidence, and whether the currently discovered key still matches it.
OpenSSH known_hosts alone cannot express ServerDesk Trust State or distinguish independent verification from Adoption or TOFU.
Implicitly reading user trust files, accepting changed keys, or treating a successful connection as proof of identity would bypass that evidence.
A generated trust file must never become a competing source of truth or permit a connection after storage or generation failure.
The accepted sources leave the connection eligibility of explicitly accepted UNVERIFIED keys undefined; publication is blocked on the precise policy recorded in Further Notes.

## Solution

Use SQLite as the only authoritative Host Key Store, with explicit Adoption, independent SHA256 verification, and warned TOFU acceptance.
Generate the application known_hosts atomically from that store before every master start and supply only that trust file to system OpenSSH.
Keep Trust State independent from Connection State and detect key mismatch before master start rather than interpreting human stderr.
Discover a target behind a Jump Server by running ssh-keyscan on the already trusted Jump Server, without treating that network position as identity proof.
Make trust replacement and deletion explicit, verified, audited Operations using the shared Operation machinery.
Expose provenance, unresolved trust, drift, and storage errors without reporting an uncommitted or unverified change as successful.
The testing approach is approved, but the acceptance criteria below are planned requirements rather than evidence of existing implementation or passing tests.

## User Stories

1. **US-001: Preserve authoritative Host Key evidence**

   - **Story:** As a Developer managing personal Linux Servers, I want accepted Host Keys and their evidence retained independently of connection attempts so that later connections use the decision I actually made.
   - **Basis:** Derived: ADR-0003, product §§4.6, 4.8, 12.3, 13.1–13.2, and glossary Host Key, Trust State, Connection State.
   - **Acceptance criteria:**
     - **US-001-AC1:** Given a successfully committed trust decision, when the application is reopened, then the Host Key Store retains its hostname, port, key algorithm, public key, SHA256 fingerprint, Trust State, source, verification time where applicable, and creation time using the accepted host_keys fields.
     - **US-001-AC2:** Given a Server trust view, when its trust evidence is displayed, then VERIFIED, ADOPTED, UNVERIFIED, MISMATCH, and NONE remain distinct and the stored source and available verification evidence are inspectable rather than inferred from connectivity.
     - **US-001-AC3:** Given an accepted Host Key and a failed or absent connection attempt, when Connection State changes or remains NOT_CONNECTED, then that observation does not itself upgrade, erase, or independently verify the Host Key's Trust State.
     - **US-001-AC4:** Given a matching connection using an ADOPTED or UNVERIFIED key, when authentication or connection succeeds, then successful transport alone does not promote the key to VERIFIED.
     - **US-001-AC5:** Given two inventory Server entries with the same Connection Identity, when either entry uses stored trust, then both consult the authoritative SQLite Host Key Store rather than maintaining independent authoritative generated-file decisions.
     - **US-001-AC6:** Given an explicit Adoption, independent verification, or TOFU acceptance request, when its trust change is processed, then it uses the shared Operation workflow and retains a sanitized Audit Trail outcome rather than bypassing the engine as an unaudited direct store edit.

2. **US-002: Explicitly adopt a previously accepted Host Key**

   - **Story:** As a Developer managing personal Linux Servers, I want to explicitly adopt a Host Key I already accepted outside ServerDesk so that existing evidence is reused without silently importing trust.
   - **Basis:** Derived: ADR-0003, product §§4.6, 9.1, and glossary Adoption.
   - **Acceptance criteria:**
     - **US-002-AC1:** Given an inventory Server and an entry in the user's known_hosts, when I explicitly request Adoption, then ServerDesk looks up that host and port using ssh-keygen -F and presents the matching Host Key for the explicit trust workflow.
     - **US-002-AC2:** Given a matching hashed-host entry, when I request Adoption, then lookup supports it and a successfully accepted Host Key is stored as ADOPTED rather than VERIFIED.
     - **US-002-AC3:** Given no matching entry or a failed lookup, when Adoption is attempted, then no successful Adoption is claimed and the affected Server and available corrective action are shown without inventing a key.
     - **US-002-AC4:** Given that I have not requested Adoption, when I add or connect to a Server, then the user's known_hosts is not silently used to accept or replace its Host Key.
     - **US-002-AC5:** Given an Adoption request that I cancel before acceptance, when the workflow ends, then no new trust acceptance is committed and the original user file is unchanged.

3. **US-003: Verify against an exact independent fingerprint**

   - **Story:** As a Developer managing personal Linux Servers, I want to compare a discovered Host Key with an independently trusted SHA256 fingerprint so that VERIFIED means independent evidence matched exactly.
   - **Basis:** Derived: product §4.6 verification path and §§14, 18.
   - **Acceptance criteria:**
     - **US-003-AC1:** Given a SHA256 fingerprint I obtained from a trusted independent source such as the provider console or cloud-init, when I submit it for a discovered candidate Host Key, then ServerDesk compares it exactly with that key's ssh-keyscan-derived SHA256 fingerprint before accepting it as VERIFIED.
     - **US-003-AC2:** Given that the supplied fingerprint differs from the discovered fingerprint, including only a partial match, when comparison completes, then verification does not succeed and the candidate is not stored as VERIFIED.
     - **US-003-AC3:** Given that ssh-keyscan fails or yields no candidate matching the independent fingerprint, when verification is attempted, then no verified acceptance is committed and the discovery or comparison failure is explained.
     - **US-003-AC4:** Given a previously accepted ADOPTED or UNVERIFIED Host Key and a matching independent fingerprint, when explicit verification commits successfully, then the stored decision becomes VERIFIED with its verification evidence and time retained.

4. **US-004: Accept TOFU without overstating its evidence**

   - **Story:** As a Developer managing personal Linux Servers, I want a clearly warned TOFU decision so that accepting an unverified key is never mistaken for independently proving identity.
   - **Basis:** Derived: product §4.6 and glossary TOFU, Trust State.
   - **Acceptance criteria:**
     - **US-004-AC1:** Given a newly discovered Host Key without an independent trusted comparison, when I choose TOFU, then the workflow warns that identity has not been independently verified and requires explicit confirmation before acceptance.
     - **US-004-AC2:** Given confirmed and successfully persisted TOFU acceptance, when the result is displayed or the application is reopened, then the key remains UNVERIFIED until an independent verification succeeds.
     - **US-004-AC3:** Given a dismissed or rejected TOFU warning, when the workflow ends, then no TOFU acceptance is recorded and discovery alone does not establish trust.

5. **US-005: Notice drift in an adopted external entry**

   - **Story:** As a Developer managing personal Linux Servers, I want changes to the external entry from which I adopted trust to be visible, so that I can review changed evidence without silently changing my application trust decision.
   - **Basis:** Derived: ADR-0003 Consequences and product §4.6.
   - **Acceptance criteria:**
     - **US-005-AC1:** Given an ADOPTED Host Key whose corresponding user known_hosts entry has subsequently changed, when ServerDesk observes that drift, then the Server trust view warns about the changed external evidence.
     - **US-005-AC2:** Given observed external drift, when the warning is shown or another connection is requested, then ServerDesk does not automatically replace the SQLite Host Key, its Trust State, or its provenance with the external entry.
     - **US-005-AC3:** Given the drift warning, when I elect to change trust, then the new decision requires the explicit acceptance or replacement workflow rather than a background synchronization of the user file.

6. **US-006: Block absent trust and mismatched keys before transport**

   - **Story:** As a Developer managing personal Linux Servers, I want missing or mismatched trust to prevent a master from starting so that connection attempts cannot bypass Host Key policy.
   - **Basis:** Derived: ADR-0001, product §§4.6, 4.8, 4.10, 14.
   - **Acceptance criteria:**
     - **US-006-AC1:** Given no accepted Host Key and Trust State NONE, when Connect or connect_on_startup would start a master, then connection is blocked as HostKeyUntrusted and the user is offered explicit trust acceptance instead of auto-acceptance.
     - **US-006-AC2:** Given a stored Host Key and a different discovered Host Key, when the pre-start check runs using ssh-keygen -F against the keyscan result, then the mismatch blocks master start and is exposed as MISMATCH trust evidence and HostKeyMismatch/HOST_KEY_MISMATCH in the appropriate error and Connection State surfaces.
     - **US-006-AC3:** Given a key mismatch, when the connection is rejected, then the previously accepted key is not silently overwritten and no terminal, SFTP, Operation, or Collector starts through a newly accepted replacement.
     - **US-006-AC4:** Given a mismatch or trust failure, when the error is classified, then classification comes from the trust preflight rather than parsing human ssh stderr or assuming exit code 255 uniquely identifies the cause.

7. **US-007: Discover a target through a trusted Jump Server**

   - **Story:** As a Developer managing personal Linux Servers, I want the target Host Key discovered from the correct network route, so that I can assess its identity independently rather than inheriting the Jump Server's identity assurance.
   - **Basis:** Derived: product §§4.3, 4.4, 4.6, 4.10 and glossary Jump Server, Connection Identity.
   - **Acceptance criteria:**
     - **US-007-AC1:** Given a registered target route through a Jump Server with an independently VERIFIED Host Key, when target Host Key discovery runs, then ssh-keyscan executes on that Jump Server rather than assuming the local workstation can discover the target directly.
     - **US-007-AC2:** Given a Jump Server with no accepted Host Key or with a mismatched Host Key, when the route is requested, then it is not used for target discovery or connection until its own explicit trust acceptance or mismatch resolution completes.
     - **US-007-AC3:** Given a target Host Key discovered through a trusted Jump Server, when the target trust workflow is shown, then it still requires Adoption, exact independent verification, or warned explicit TOFU and is not automatically VERIFIED or ADOPTED by the route.
     - **US-007-AC4:** Given a failed target discovery on the Jump Server, when the result is shown, then the target remains unaccepted unless it already had a separate committed decision and the error identifies the affected target and route without claiming identity proof.

8. **US-008: Generate an authoritative trust snapshot before every master start**

   - **Story:** As a Developer managing personal Linux Servers, I want each master to start against an atomic snapshot of SQLite trust so that stale or manually edited trust files cannot override my decisions.
   - **Basis:** Derived: ADR-0003, ADR-0002, product §§4.4, 4.6–4.7, 14.
   - **Acceptance criteria:**
     - **US-008-AC1:** Given any requested new master start, including startup connection and reconnect, when trust prerequisites are satisfied, then the application's known_hosts is regenerated atomically from SQLite before that master is allowed to start.
     - **US-008-AC2:** Given a manually edited application-generated known_hosts, when the next master start is prepared, then regeneration replaces those edits with SQLite-derived content and does not import the edits into the Host Key Store.
     - **US-008-AC3:** Given a generated trust file ready for master start, when system OpenSSH is launched, then only the application-generated file is used for Host Key trust; user/global known_hosts and dynamic known-host commands cannot supply an alternative acceptance path.
     - **US-008-AC4:** Given generation or atomic replacement failure, including disk-full in an isolated test filesystem, when a master start is requested, then no partially generated file is treated as complete, no old snapshot is used as a fallback for that start, and the start fails closed with a storage/generation error.
     - **US-008-AC5:** Given a successful earlier connection and a later failed regeneration, when another master is requested, then that earlier success does not waive regeneration or authorize the later start.

9. **US-009: Fail closed when trust storage cannot commit**

   - **Story:** As a Developer managing personal Linux Servers, I want trust storage failures to leave an honest outcome so that a partially saved decision never authorizes access.
   - **Basis:** Derived: product §§13.1–13.2, 14, 16.2–16.3, 18.
   - **Acceptance criteria:**
     - **US-009-AC1:** Given a trust workflow that changes multiple persisted facts, when a step fails before transaction commit, then the workflow does not expose a partially committed successful trust decision or authorize a master from that attempted decision.
     - **US-009-AC2:** Given an unreadable or otherwise unavailable authoritative Host Key Store, when trust is needed for a master start, then the start is blocked rather than falling back to user known_hosts, generated-file edits, or assumed trust.
     - **US-009-AC3:** Given a versioned SQLite migration required for the Host Key Store, when it succeeds, then existing Host Keys and evidence remain available without silent truncation.
     - **US-009-AC4:** Given a newly created Host Key record and the active storage session, when their accepted storage invariants are inspected, then its primary key is a UUID and SQLite foreign-key enforcement is enabled without adding any relationship absent the accepted schema.
     - **US-009-AC5:** Given trust workflows and their Audit Trail, when persisted data is inspected, then the accepted public key/evidence fields do not introduce stored passwords, passphrases, private-key material, or file contents into SQLite or sanitized audit payloads.
     - **US-009-AC6:** Given a required Host Key Store migration that fails, when affected trust workflows are requested, then StorageError is reported rather than proceeding with incompatible state or claiming a successful migration.

10. **US-010: Explicitly replace or delete trust with an Audit Trail**

   - **Story:** As a Developer managing personal Linux Servers, I want trust replacement and deletion to be explicit audited decisions so that a mismatch cannot be resolved by an invisible overwrite.
   - **Basis:** Derived: product §§4.6, 9.1–9.2, 9.5–9.8, 13, 14, 18.
   - **Acceptance criteria:**
     - **US-010-AC1:** Given an existing accepted Host Key, when replacement or deletion is requested, then the affected Server/key and intended change are presented for an explicit decision through the shared Operation workflow rather than applied automatically by discovery or drift detection.
     - **US-010-AC2:** Given an explicitly confirmed replacement, when the new decision completes, then the replacement follows one of the accepted acceptance paths and retains that path's Trust State; replacement alone does not mean VERIFIED.
     - **US-010-AC3:** Given an explicitly confirmed deletion, when persistence and verification complete, then the deleted entry is no longer authoritative and a later connection without another eligible accepted entry is blocked rather than recreating trust automatically.
     - **US-010-AC4:** Given a successful replacement or deletion, when Operation details are inspected, then the Audit Trail records the explicit change and verified outcome using the shared append-only operations/operation_events contract without secrets or file contents and without automatic audit deletion.
     - **US-010-AC5:** Given failed persistence or insufficient verification evidence during replacement or deletion, when the result is displayed, then no successful change is claimed and the shared Operation outcome reflects the evidence rather than a dismissed dialog or attempted write.
     - **US-010-AC6:** Given a previously recorded trust-change Operation, when later trust changes occur, then earlier audit evidence is not erased or rewritten to disguise the former decision.

11. **US-011: Keep trust workflows usable and honest in GTK**

   - **Story:** As a Developer managing personal Linux Servers, I want responsive and accessible trust workflows so that I can decide safely even when discovery or persistence fails.
   - **Basis:** Derived: product §§12.2–12.3, 14, 18 and glossary Connection State, Unknown Outcome.
   - **Acceptance criteria:**
     - **US-011-AC1:** Given discovery, Adoption lookup, verification, or trust persistence in progress, when I interact with the GTK application, then the main thread remains responsive and the affected workflow exposes loading rather than a fabricated successful result.
     - **US-011-AC2:** Given no stored key, unresolved trust, observed drift, or a failed trust workflow, when the relevant view is opened, then it presents the corresponding empty/unverified, stale-external-evidence warning, or error state without confusing it with Server downtime.
     - **US-011-AC3:** Given an error, when its details are shown, then the UI identifies what failed, the affected Server/key, whether a change may have occurred, available recovery actions, and the related Operation details where applicable.
     - **US-011-AC4:** Given an explicit trust decision or warning, when I navigate using the keyboard, then the acceptance, rejection/cancel, and relevant details controls are reachable without requiring a pointer.

## Implementation Decisions

- SQLite will be the single owner of persisted Host Key truth; generated known_hosts will be derived runtime input, never an authority or import mechanism.
- The accepted host_keys fields are id, hostname, port, key_algorithm, public_key, fingerprint_sha256, trust_state, source, verified_at, and created_at; this epic adds no schema.
- UUID primary keys, enabled foreign keys, versioned migrations, transactions for multi-step mutations, and no silent truncation remain the accepted storage contract.
- Trust State will remain VERIFIED, ADOPTED, UNVERIFIED, MISMATCH, or NONE and will not be merged with the Connection State model.
- Adoption will use explicit ssh-keygen -F lookup, including hashed hosts; verification will compare the independent SHA256 fingerprint exactly against discovered evidence; warned TOFU will retain UNVERIFIED.
- System OpenSSH trust preflight will detect mismatch before master start; human stderr and exit code 255 will not serve as a unique Host Key diagnosis.
- The generated configuration policy owned by ADR-0002 will enforce StrictHostKeyChecking=yes, the generated UserKnownHostsFile, GlobalKnownHostsFile=none, KnownHostsCommand=none, and UpdateHostKeys=no; this epic will provide the authoritative trust snapshot and fail-closed gate.
- Atomic regeneration will be required for every master start, including reconnect and startup-triggered connection.
- A target behind a Jump Server will be discovered by ssh-keyscan on the trusted Jump Server and will undergo its own acceptance workflow.
- Explicit trust changes will use the shared Operation/Audit contract; a local SQLite trust change will not be misrepresented as a remote privileged shell mutation.
- The application layer will own workflows, the domain will own trust invariants, infrastructure will handle OpenSSH and SQLite, and GTK will own presentation state.
- Background work will run through the accepted Tokio/application boundaries, with UI delivery through channels scheduled to glib::MainContext and no GTK widget access from Tokio.
- No trusted-state allowlist is selected here because the accepted sources do not provide one; Further Notes item 1 must be resolved before publication readiness.

## Testing Decisions

The user approved the primary seam: presentation-facing application workflows, the same workflows invoked from GTK, backed by actual system OpenSSH and SQLite in isolated Incus/LXD Ubuntu 24.04 fixtures with systemd 255.
Actual GTK smoke will cover visible trust states, keyboard decisions, responsive failure handling, and the add Server → trust Host Key → connect → terminal journey.
Narrow pure behavior tests will cover exact fingerprint comparison and trust/state transitions where they have no external dependency.
These are planned tests; no authored harness exists and this documentation task runs no product tests.

- Exercise plain and hashed Adoption with real ssh-keygen, explicit-only reads, no match, cancellation, and post-Adoption external drift without automatic replacement.
- Exercise an independently matching SHA256 fingerprint, a differing fingerprint, a partial fingerprint, failed keyscan, and later independent verification of an UNVERIFIED key.
- Exercise warned explicit TOFU and durable UNVERIFIED status; direct/Jump connection eligibility assertions remain blocked until Further Notes item 1 is resolved.
- Exercise target discovery by running real ssh-keyscan on a trusted Jump Server and show that its returned target key still requires independent acceptance.
- Exercise mismatched keys before master start and prove no new terminal, SFTP, Operation, or Collector can bypass the rejected trust gate.
- Exercise generated-file tampering, regeneration before each new master, and atomic generation failure; inspect actual attempted master lifecycle rather than matching generated text alone.
- Exercise real SQLite transactions, foreign-key rejection, versioned migrations, replacement/deletion persistence, and durable sanitized Audit Trail outcomes.
- Inject disk-full and failed atomic replacement only on an isolated container test filesystem, never the host filesystem, and verify fail-closed master start plus no partial successful trust commit.
- Exercise storage-unavailable and migration-failure UI with no fallback to external or stale generated trust.
- Verify that rejected/dismissed dialogs, failed commits, successful authentication, and changed external files cannot independently produce VERIFIED.
- Inspect persisted results and error/audit payloads for secret exclusion without recording secret values in test reports.

## Out of Scope

- Normal ControlMaster ownership, authentication, terminal/SFTP/Collector transport, and disconnect lifecycle belong to ADR-0001.
- Generated SSH configuration isolation, the complete forced-options policy, static SSH config import, registered-route validation, and import exclusions belong to ADR-0002.
- General Operation execution, typed arguments, sudo policy, classification, and operation-specific remote Verifier/Reconciliation internals belong to ADR-0004.
- Startup orphan cleanup and interrupted-Operation recovery belong to ADR-0006.
- SSHFP/DNSSEC, automatic trust synchronization, automatic key replacement, and automatic promotion of TOFU to VERIFIED are not MVP behavior.
- Recursive user configuration import, config re-import/diff, audit export/retention, Secret Service, and privileged file editing are not added by this epic.
- Inventory removal must not delete or otherwise mutate the remote Server; inventory management itself is not expanded here.

## Further Notes

1. **Publication blocker — connection eligibility of UNVERIFIED and the meaning of trusted for a Jump Server.** Product §4.6 line 175 explicitly accepts TOFU but says its status remains UNVERIFIED until independent verification; lines 176–177 require a trusted Jump Server; §4.10 line 219 requires hosts not yet trusted to be blocked. ADR-0003 line 3 lists Trust States without an eligibility set, and glossary Trust State/TOFU defines evidence strength rather than connection permission. Product §3.1 line 98 highlights all states other than VERIFIED/ADOPTED but does not define a connection gate. Therefore the sources do not determine whether explicitly accepted UNVERIFIED permits a direct connection, a Jump Server connection, both, or neither. A policy decision for those two uses is required; this draft does not invent one. Concrete criteria cover absent/unaccepted keys, mismatch, explicit acceptance, and discovery through an independently VERIFIED Jump Server; the missing eligibility behavior must be resolved and receive concrete criteria before readiness.
2. **Cross-epic dependencies.** ADR-0001 consumes the trust gate and generated snapshot before normal master starts; ADR-0002 supplies runtime configuration isolation and registered routes; ADR-0004 supplies shared Operation/Audit machinery for trust changes; ADR-0006 sequences startup recovery before any new master and governs ownership. These are behavioral interfaces, not circular implementation-ticket ordering.
3. **Nonblocking detail.** The sources require a warning when adopted external evidence changes but do not prescribe its observation cadence; no polling interval or automatic synchronization is introduced.
4. **Shared Operation policy dependency.** Product §9.1 line 369 includes Host Key trust changes in the engine, while §9.3 line 382 describes every Operation as a remote POSIX script and §9.4 line 394 schedules Reconciliation after Server reconnect. Product §4.6 and ADR-0003 instead make these trust changes authoritative local SQLite changes. The sources do not supply the local-only lifecycle/Reconciliation specialization. This draft defines committed-store postconditions and audit integration, not an invented remote script or Operation Verifier algorithm. ADR-0004 owns resolution of that shared policy gap before enabling those mutation types; these store criteria alone do not claim it resolved.
5. **Evidence and readiness.** This is a specification of planned behavior, not an implemented Host Key Store; existing greeting-only startup proves none of these acceptance criteria. Readiness remains needs-info until item 1 is resolved, the shared policy dependency in item 4 is addressed.
