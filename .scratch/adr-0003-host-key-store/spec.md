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

# Design

## Usage

1. **Inspect/review before transport:** bootstrap calls `TrustCore::spawn(store).into_trust_ports()` and `Trust::assemble(core, configuration, None)`. GTK requests `trust.review(TrustQuery::Inspect { server_id }).await?`, direct Discover, explicit AdoptionLookup, CompareFingerprint or CheckAdoptedDrift; `candidate.inspect_evidence()` exposes readable evidence without a forgeable constructor. Later `Trust::assemble` with Some(routes) uses the SAME core for routed discovery, never a second owner.
2. **Prepare a master internally:** E1 resolves inventory FIRST and calls `gate.prepare(resolved, reservation, discovery).await?`. `prepared.into_prepared_start()` returns the original owned inputs and TrustStart; compilation reads `inspect_trust_start` only after accepted eligibility and atomic regeneration. Missing eligibility policy remains unavailable.
3. **Make an explicit audited decision:** GTK passes reviewed TrustChange to the canonical E4 Operation workflow. E3 `prepare_change` validates a sealed expected-state delta without committing, even before local admission is enabled. After human local-policy approval E4:05's `prepare_local_trust`/`admit_local_trust` stages create LocalTrustAdmitted with exact consent/current preflight, retained Runtime lease and durable RUNNING; `into_local_trust_admitted` returns the original prepared change and E4-issued envelope to the E3 commit caller. Store reads `prepared_trust_store_view`; `observation.into_facts()` supplies paired event/read-back facts for the approved E4 outcome and joined lease-release path.

## Types

```rust
pub enum TrustState { Verified, Adopted, Unverified, Mismatch, None }
pub enum TrustUse { Direct, Jump }
pub enum AcceptancePath { Adoption, Verification, Tofu }
pub enum TrustProvenance {
    Adopted {
        source: PathBuf,
        lookup_host: String,
        lookup_port: u16,
        source_entry_digest: [u8; 32],
    },
    IndependentlyCompared {
        source_description: String,
        independent_sha256: String,
    },
    Tofu,
}
pub struct HostKey {
    pub id: Uuid,
    pub hostname: String,
    pub port: u16,
    pub key_algorithm: String,
    pub public_key: String,
    pub fingerprint_sha256: String,
    pub trust_state: TrustState,
    pub source: TrustProvenance,
    pub verified_at: Option<SystemTime>,
    pub created_at: SystemTime,
}
pub(crate) struct ScanRequest {
    pub(crate) hostname: String,
    pub(crate) port: u16,
}
pub(crate) struct KeyObservation {
    pub(crate) hostname: String,
    pub(crate) port: u16,
    pub(crate) keys: Vec<(String, String, String)>,
    pub(crate) route_server_ids: Vec<Uuid>,
    pub(crate) collected_at: Instant,
}
pub(crate) enum DiscoveryPath {
    Local,
    Jump {
        generation: E1::MasterGeneration,
        consumers: E1::ConsumerAccess,
    },
}
pub struct TrustEvidence {
    endpoint: (String, u16),
    key_algorithm: String,
    public_key: String,
    fingerprint_sha256: String,
    route_server_ids: Vec<Uuid>,
    observed_at: Instant,
    provenance: Option<TrustProvenance>,
    exact_independent_match: Option<String>,
}
pub enum TrustQuery {
    Inspect { server_id: Uuid },
    Discover { server_id: Uuid },
    AdoptionLookup { server_id: Uuid, source: PathBuf },
    CompareFingerprint {
        server_id: Uuid,
        candidate: TrustEvidence,
        independent_sha256: String,
        independent_source: String,
    },
    CheckAdoptedDrift { server_id: Uuid },
}
pub struct TrustReview {
    pub server_id: Uuid,
    pub stored_keys: Vec<HostKey>,
    pub state: TrustState,
    pub candidates: Vec<TrustEvidence>,
    pub eligibility: Vec<(TrustUse, E6::Availability)>,
    pub adopted_evidence_changed: bool,
    pub observations: Vec<E6::Failure>,
}
pub enum TrustChange {
    Accept { evidence: TrustEvidence, path: AcceptancePath },
    Replace {
        expected: HostKey,
        evidence: TrustEvidence,
        path: AcceptancePath,
    },
    Delete { expected: HostKey },
}
pub struct Trust {
    core: TrustCore,
    configuration: E2::Configuration,
    routes: Option<E1::RouteAccess>,
}
pub(crate) struct TrustCore { tx: mpsc::Sender<TrustMessage> }
pub(crate) struct TrustGate { tx: mpsc::Sender<TrustMessage> }
pub(crate) struct TrustExecutor { tx: mpsc::Sender<TrustMessage> }
pub(crate) struct TrustPorts {
    core: TrustCore,
    gate: TrustGate,
    executor: TrustExecutor,
}
pub(crate) struct TrustPermission {
    start_id: Uuid,
    identity: E2::ConnectionIdentity,
    endpoint_uses: Vec<((String, u16), TrustUse, Uuid, TrustState)>,
    store_digest: [u8; 32],
}
pub(crate) struct TrustSnapshot {
    start_id: Uuid,
    known_hosts_path: PathBuf,
    store_digest: [u8; 32],
    generated_at: Instant,
}
pub(crate) struct TrustStart {
    permission: TrustPermission,
    snapshot: TrustSnapshot,
}
pub(crate) struct PreparedTrustChange {
    operation_id: Uuid,
    server_id: Uuid,
    identity: E2::ConnectionIdentity,
    endpoint: (String, u16),
    expected_keys: Vec<HostKey>,
    replacement_keys: Vec<HostKey>,
    acceptance_path: Option<AcceptancePath>,
    evidence_digest: [u8; 32],
}
pub(crate) struct TrustCommitObservation {
    pub(crate) operation_id: Uuid,
    pub(crate) before: Vec<HostKey>,
    pub(crate) after: Vec<HostKey>,
    pub(crate) committed_event_id: Uuid,
    pub(crate) committed_event_ordinal: u64,
    pub(crate) read_back_digest: [u8; 32],
}
pub(crate) enum TrustStoreRequest {
    Read { endpoints: Vec<(String, u16)> },
}
pub(crate) enum TrustPending {
    Review {
        server_id: Uuid,
        identity: E2::ConnectionIdentity,
        inventory_revision: [u8; 32],
        reply: E6::Reply<TrustReview>,
    },
    Start {
        start_id: Uuid,
        identity: E2::ConnectionIdentity,
        inventory_revision: [u8; 32],
        reply: E6::Reply<PreparedStart>,
    },
    Change {
        operation_id: Uuid,
        identity: E2::ConnectionIdentity,
        reply: E6::Reply<PreparedTrustChange>,
    },
    Commit {
        operation_id: Uuid,
        committed_event_id: Uuid,
        reply: E6::Reply<TrustCommitObservation>,
    },
}
pub(crate) enum TrustCompletion {
    Reviewed { request_id: Uuid, result: Result<TrustReview, E6::Failure> },
    Started { request_id: Uuid, result: Result<PreparedStart, E6::Failure> },
    ChangePrepared { request_id: Uuid, result: Result<PreparedTrustChange, E6::Failure> },
    Committed { request_id: Uuid, result: Result<TrustCommitObservation, E6::Failure> },
}
pub(crate) enum TrustMessage {
    Completed(TrustCompletion),
    Review {
        resolved: E2::ResolvedConnection,
        query: TrustQuery,
        discovery: Option<DiscoveryPath>,
        reply: E6::Reply<TrustReview>,
    },
    PrepareStart {
        resolved: E2::ResolvedConnection,
        reservation: E1::StartReservation,
        discovery: DiscoveryPath,
        reply: E6::Reply<PreparedStart>,
    },
    PrepareChange {
        operation_id: Uuid,
        resolved: E2::ResolvedConnection,
        change: TrustChange,
        reply: E6::Reply<PreparedTrustChange>,
    },
    Commit {
        change: PreparedTrustChange,
        audit: E4::LocalTrustEnvelope,
        reply: E6::Reply<TrustCommitObservation>,
    },
}
pub(crate) struct PreparedStart {
    resolved: E2::ResolvedConnection,
    reservation: E1::StartReservation,
    trust: TrustStart,
}
pub(crate) struct TrustStartView<'a> {
    pub(crate) start_id: Uuid,
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) endpoint_uses: &'a [((String, u16), TrustUse, Uuid, TrustState)],
    pub(crate) store_digest: &'a [u8; 32],
    pub(crate) known_hosts_path: &'a Path,
    pub(crate) generated_at: Instant,
}
pub(crate) struct PreparedTrustChangeView<'a> {
    pub(crate) operation_id: Uuid,
    pub(crate) server_id: Uuid,
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) endpoint: &'a (String, u16),
    pub(crate) expected_keys: &'a [HostKey],
    pub(crate) replacement_keys: &'a [HostKey],
    pub(crate) acceptance_path: Option<AcceptancePath>,
    pub(crate) evidence_digest: &'a [u8; 32],
}
pub(crate) struct TrustCommitFacts {
    pub(crate) operation_id: Uuid,
    pub(crate) before: Vec<HostKey>,
    pub(crate) after: Vec<HostKey>,
    pub(crate) committed_event_id: Uuid,
    pub(crate) committed_event_ordinal: u64,
    pub(crate) read_back_digest: [u8; 32],
}
pub struct TrustEvidenceView<'a> {
    pub endpoint: &'a (String, u16),
    pub key_algorithm: &'a str,
    pub public_key: &'a str,
    pub fingerprint_sha256: &'a str,
    pub route_server_ids: &'a [Uuid],
    pub observed_at: Instant,
    pub provenance: Option<&'a TrustProvenance>,
    pub exact_independent_match: Option<&'a str>,
}
impl TrustCore {
    pub(crate) fn spawn(store: E6::Store) -> TrustPorts;
}
impl Trust {
    pub(crate) fn assemble(
        core: TrustCore,
        configuration: E2::Configuration,
        routes: Option<E1::RouteAccess>,
    ) -> Trust;

    pub async fn review(
        &self, query: TrustQuery,
    ) -> Result<TrustReview, E6::Failure>;
}
impl TrustGate {
    pub(crate) async fn prepare(
        &self,
        resolved: E2::ResolvedConnection,
        reservation: E1::StartReservation,
        discovery: DiscoveryPath,
    ) -> Result<PreparedStart, E6::Failure>;
}
impl TrustExecutor {
    pub(crate) async fn prepare_change(
        &self,
        operation_id: Uuid,
        resolved: E2::ResolvedConnection,
        change: TrustChange,
    ) -> Result<PreparedTrustChange, E6::Failure>;

    pub(crate) async fn commit(
        &self,
        change: PreparedTrustChange,
        audit: E4::LocalTrustEnvelope,
    ) -> Result<TrustCommitObservation, E6::Failure>;
}
impl TrustPorts {
    pub(crate) fn into_trust_ports(self) -> (TrustCore, TrustGate, TrustExecutor);
}
impl PreparedStart {
    pub(crate) fn into_prepared_start(self) -> (E2::ResolvedConnection, E1::StartReservation, TrustStart);
}
impl TrustStart {
    pub(crate) fn inspect_trust_start(&self) -> TrustStartView<'_>;
}
impl PreparedTrustChange {
    pub(crate) fn prepared_trust_store_view(&self) -> PreparedTrustChangeView<'_>;
}
impl TrustCommitObservation {
    pub(crate) fn into_facts(self) -> TrustCommitFacts;
}
impl TrustEvidence {
    pub fn inspect_evidence(&self) -> TrustEvidenceView<'_>;
}
```

TrustEvidence, TrustStart, PreparedStart and PreparedTrustChange fields stay private; only borrowed views or consuming decomposition expose them. StartReservation/PreparedStart/TrustStart/prepared changes are not Clone. ScanRequest/KeyObservation are immutable inter-owner data, not trust authority. TrustCommitObservation is a plain Store-created pub(crate) observation and can be read/moved without a private E3 constructor; it cannot authorize trust or mark Operation Succeeded.

TrustCore alone writes trust semantics; E6 Store physically commits authorized requests and E4 alone authors audit/outcome facts. All three TrustPorts capabilities share one mailbox. Owned workers hold no mutable owner borrow across awaits; TrustPending keeps request/identity/revision/reply bindings and TrustCompletion returns owned results. Late/cancelled review/start completions cannot create current permission; a committed transaction's observation must still reach durable outcome handling even after caller cancellation. Core stays responsive to reads/completions and does not await a caller that needs its own mailbox. E1's attempt registry retires failed reservations using retained metadata, never forged wrappers. Canonical declarations come from ADR-0006:01, but actual handles/observations come only from their runtime owners. GTK uses main-context channel delivery; no Tokio worker touches GTK widgets.

## Contract

### Vocabulary

| Name                      | Kind   | Meaning                                                                                | Source of truth                                            | Writer                                                       |
| ------------------------- | ------ | -------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------ |
| `TrustState`              | enum   | Evidence strength/conflict, separate from connectivity                                 | `src/domain/trust.rs` `TrustState`                         | TrustCore                                                    |
| `TrustUse`                | enum   | Direct or Jump eligibility question                                                    | `src/domain/trust.rs` `TrustUse`                           | TrustCore                                                    |
| `AcceptancePath`          | enum   | Explicit accepted evidence path                                                        | `src/domain/trust.rs` `AcceptancePath`                     | TrustCore                                                    |
| `TrustProvenance`         | enum   | Retained origin/comparison evidence                                                    | `src/domain/trust.rs` `TrustProvenance`                    | TrustCore                                                    |
| `HostKey`                 | entity | Authoritative stored public key decision                                               | `src/domain/trust.rs` `HostKey`                            | TrustCore                                                    |
| `ScanRequest`             | value  | Fixed endpoint-only keyscan request                                                    | `src/application/trust.rs` `ScanRequest`                   | TrustCore                                                    |
| `KeyObservation`          | value  | Actual scan candidates and route                                                       | `src/application/trust.rs` `KeyObservation`                | `src/infrastructure/ssh/mod.rs` (immutable observation only) |
| `DiscoveryPath`           | enum   | Local discovery or authorized Jump generation                                          | `src/application/trust.rs` `DiscoveryPath`                 | TrustCore                                                    |
| `TrustEvidence`           | value  | Sealed discovered/adopted/compared evidence                                            | `src/domain/trust.rs` `TrustEvidence`                      | TrustCore                                                    |
| `TrustQuery`              | enum   | Read-only trust workflow request                                                       | `src/domain/trust.rs` `TrustQuery`                         | TrustCore                                                    |
| `TrustReview`             | value  | Stored decision, candidates, drift and eligibility                                     | `src/domain/trust.rs` `TrustReview`                        | TrustCore                                                    |
| `TrustChange`             | enum   | Explicit proposed acceptance/replacement/deletion                                      | `src/domain/trust.rs` `TrustChange`                        | TrustCore                                                    |
| `Trust`                   | value  | Desktop trust façade                                                                   | `src/application/trust.rs` `Trust`                         | TrustCore                                                    |
| `TrustCore`               | entity | Sole trust semantic owner                                                              | `src/application/trust.rs` `TrustCore`                     | TrustCore                                                    |
| `TrustGate`               | value  | Delegated master-start gate                                                            | `src/application/trust.rs` `TrustGate`                     | TrustCore                                                    |
| `TrustExecutor`           | value  | Delegated Operation-only trust executor                                                | `src/application/trust.rs` `TrustExecutor`                 | TrustCore                                                    |
| `TrustPorts`              | value  | Bootstrap handles to one trust actor                                                   | `src/application/trust.rs` `TrustPorts`                    | TrustCore                                                    |
| `TrustPermission`         | value  | Positive use/start/identity-bound decision                                             | `src/application/trust.rs` `TrustPermission`               | TrustCore                                                    |
| `TrustSnapshot`           | value  | Atomic SQLite-derived known_hosts generation                                           | `src/application/trust.rs` `TrustSnapshot`                 | TrustCore                                                    |
| `TrustStart`              | value  | One start’s matching permission and snapshot                                           | `src/application/trust.rs` `TrustStart`                    | TrustCore                                                    |
| `PreparedTrustChange`     | value  | Semantically validated expected/replacement key set                                    | `src/application/trust.rs` `PreparedTrustChange`           | TrustCore                                                    |
| `TrustCommitObservation`  | value  | Committed/read-back trust facts, not Operation verdict                                 | `src/application/trust.rs` `TrustCommitObservation`        | Store (committed observation only)                           |
| `TrustStoreRequest`       | enum   | Private authoritative trust reads                                                      | `src/application/trust.rs` `TrustStoreRequest`             | TrustCore                                                    |
| `TrustPending`            | enum   | Owner-retained request bindings and reply channels during owned I/O work               | `src/application/trust.rs` `TrustPending`                  | TrustCore                                                    |
| `TrustCompletion`         | enum   | Owned worker completion returned to the still-serviced trust mailbox                   | `src/application/trust.rs` `TrustCompletion`               | TrustCore                                                    |
| `TrustMessage`            | enum   | Private trust actor protocol                                                           | `src/application/trust.rs` `TrustMessage`                  | TrustCore                                                    |
| `PreparedStart`           | value  | Original owned start resolution/reservation plus matching TrustStart                   | `src/application/trust.rs` `PreparedStart`                 | TrustCore                                                    |
| `TrustStartView`          | value  | Borrowed permission and snapshot binding                                               | `src/application/trust.rs` `TrustStartView`                | TrustCore                                                    |
| `PreparedTrustChangeView` | value  | Borrowed exact expected and replacement Host Key sets                                  | `src/application/trust.rs` `PreparedTrustChangeView`       | TrustCore                                                    |
| `TrustCommitFacts`        | value  | Actual committed/read-back facts including paired event ordinal                        | `src/application/trust.rs` `TrustCommitFacts`              | TrustCore                                                    |
| `TrustEvidenceView`       | value  | Borrowed exact evidence fingerprint, origin and independent comparison                 | `src/domain/trust.rs` `TrustEvidenceView`                  | TrustCore                                                    |
| `Store`                   | entity | Sole physical primary SQLite owner                                                     | `src/infrastructure/storage/mod.rs` `Store`                | Store                                                        |
| `Failure`                 | value  | Classified application failure                                                         | `src/domain/error.rs` `Failure`                            | none (immutable)                                             |
| `Availability`            | enum   | Available or reasoned unavailable policy/capability                                    | `src/domain/error.rs` `Availability`                       | none (immutable)                                             |
| `Configuration`           | entity | Canonical saved-inventory workflow owner                                               | `src/application/configuration.rs` `Configuration`         | Configuration                                                |
| `ResolvedConnection`      | value  | Complete saved identity resolved before trust                                          | `src/domain/connection_identity.rs` `ResolvedConnection`   | Configuration                                                |
| `ConnectionIdentity`      | value  | Structural target and complete route                                                   | `src/domain/connection_identity.rs` `ConnectionIdentity`   | Configuration                                                |
| `StartReservation`        | value  | E1-issued single-start ownership                                                       | `src/application/connection_manager.rs` `StartReservation` | ConnectionManager                                            |
| `MasterGeneration`        | value  | E1-issued existing master reference                                                    | `src/application/connection_manager.rs` `MasterGeneration` | ConnectionManager                                            |
| `ConsumerAccess`          | value  | Capability for existing-generation consumers                                           | `src/application/connection_manager.rs` `ConsumerAccess`   | ConnectionManager                                            |
| `RouteAccess`             | value  | Capability for registered prefix discovery                                             | `src/application/connection_manager.rs` `RouteAccess`      | ConnectionManager                                            |
| `LocalTrustEnvelope`      | value  | Sealed approved local Operation/audit admission                                        | `src/application/operations.rs` `LocalTrustEnvelope`       | OperationEngine                                              |
| `AuditTail`               | value  | Expected last event identity and payload ordinal                                       | `src/domain/operation.rs` `AuditTail`                      | OperationEngine                                              |
| `LocalTrustPlan`          | value  | E4's sealed local prepared intent and exact confirmation revision                      | `src/application/operations.rs` `LocalTrustPlan`           | OperationEngine                                              |
| `LocalTrustAdmitted`      | value  | E4-issued original prepared change and local audit envelope, with lease retained by E4 | `src/application/operations.rs` `LocalTrustAdmitted`       | OperationEngine                                              |
| `OperationDecision`       | enum   | Explicit confirmation/rejection of E4's exact local plan revision                      | `src/domain/operation.rs` `OperationDecision`              | OperationEngine                                              |
| `OperationRequest`        | value  | Explicit typed user request consumed by canonical E4 planning                          | `src/domain/operation.rs` `OperationRequest`               | OperationEngine                                              |
| `Operation`               | entity | E4-owned durable administrative intent and observed outcome                            | `src/domain/operation.rs` `Operation`                      | OperationEngine                                              |
| `OperationEngine`         | entity | Sole local/remote lifecycle, audit and lease-owning coordinator                        | `src/application/operations.rs` `OperationEngine`          | OperationEngine                                              |

Foreign entities Store, Configuration, Operation and OperationEngine retain their ADR-0006, ADR-0002 and ADR-0004 invariants respectively; E3 defines no independent lifecycle or writer for them. Their canonical rows are references, never duplicate authorities.

### Functions

### `TrustCore::spawn(store: E6::Store) -> TrustPorts`

- **Intent:** Establish the one canonical trust owner before any transport exists.
- **Owner:** `src/application/trust.rs`.
- **Reads:** Store.
- **Writes:** TrustCore, TrustPorts.
- **Preconditions:** Compatible migrated primary Store exists.
- **Postconditions:** Returns core/gate/executor handles to one serviced mailbox; reads actual accepted host_keys fields through Store, preserving migrations/provenance without accepting or scanning anything.
- **Invariants:** INV-1, INV-8.
- **Errors:** Actual read/migration failures surface as StorageError on use, never fabricated NONE or silent truncation.
- **Idempotency:** One trust owner per primary; construction does not rewrite stored keys.

### `TrustPorts::into_trust_ports(self) -> (TrustCore, TrustGate, TrustExecutor)`

- **Intent:** Distribute the bootstrap handles of the same trust actor without a second writer.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustPorts.
- **Writes:** None.
- **Preconditions:** A live TrustPorts value from TrustCore::spawn exists.
- **Postconditions:** Moves exactly its existing core, gate and executor handles; no actor or HostKey is created.
- **Invariants:** INV-8.
- **Errors:** Infallible; no I/O.
- **Idempotency:** Single consuming call; cannot consume the same value twice.

### `Trust::assemble(core: TrustCore, configuration: E2::Configuration, routes: Option<E1::RouteAccess>) -> Trust`

- **Intent:** Make real stored/local trust review available without requiring transport bootstrap.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustCore, Configuration, RouteAccess.
- **Writes:** Trust.
- **Preconditions:** Core and Configuration belong to the same primary; optional routes, when present, belong to that primary.
- **Postconditions:** None supports stored inspection, explicit local Adoption lookup, direct discovery, exact comparison and drift review. None cannot serve routed discovery and cannot scan a routed target locally. A later façade with Some(routes) addresses the SAME core.
- **Invariants:** INV-1, INV-2, INV-8.
- **Errors:** Infallible composition; routed capability absence is reported by review without network fallback.
- **Idempotency:** Additional façades share the same core; composition never creates another trust authority.

### `Trust::review(&self, query: TrustQuery) -> Result<TrustReview, E6::Failure>`

- **Intent:** Present stored trust, discovery, independent comparison and drift evidence without committing acceptance.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustQuery, ResolvedConnection, HostKey, TrustEvidence, DiscoveryPath.
- **Writes:** TrustReview, TrustEvidence.
- **Preconditions:** External query identifies a resolvable saved Server; evidence matches its complete endpoint/route. Routed discovery requires Some(RouteAccess).
- **Postconditions:** Resolves inventory FIRST; Inspect reads SQLite, explicit AdoptionLookup uses ssh-keygen -F including hashed entries, direct Discover uses actual keyscan, CompareFingerprint requires exact complete SHA256 equality, and CheckAdoptedDrift reads only the recorded Adoption source and warns without synchronization. Review distinguishes all five Trust States, stored provenance, eligibility policy absence, empty/loading/stale/error. No connectivity event promotes trust. Core launches owned scan/lookup workers and keeps servicing its mailbox; completions bind request, identity and revision before presentation.
- **Invariants:** INV-1, INV-2, INV-3, INV-8; preserve INV-7 when routed refinement lands.
- **Errors:** Read, discovery, lookup, comparison, unavailable-route or storage failure commits nothing; cancelled lookup/decision cannot accept. Routed requests with None return unavailable and never Local. Errors identify Server/key/route, possible changes, recovery and related Operation when applicable.
- **Idempotency:** Repeatable evidence queries may observe newer data; never automatic acceptance, retry mutation or external-file rewrite.

### `TrustEvidence::inspect_evidence(&self) -> TrustEvidenceView<'_>`

- **Intent:** Expose the exact candidate fingerprint and provenance for a keyboard-accessible trust decision without allowing evidence forgery.
- **Owner:** `src/domain/trust.rs`.
- **Reads:** TrustEvidence.
- **Writes:** None.
- **Preconditions:** A live TrustCore-created TrustEvidence exists.
- **Postconditions:** Returns exact borrowed candidate fields and optional comparison/provenance, not a mutable record or positive permission; UI can render complete fingerprint without accessing private constructors.
- **Invariants:** INV-2, INV-3.
- **Errors:** Infallible; no I/O or allocation.
- **Idempotency:** Repeatable read; evidence remains sealed.

### `TrustGate::prepare(&self, resolved: E2::ResolvedConnection, reservation: E1::StartReservation, discovery: DiscoveryPath) -> Result<PreparedStart, E6::Failure>`

- **Intent:** Authorize exactly one reserved master start against current accepted trust and a fresh atomic snapshot.
- **Owner:** `src/application/trust.rs`.
- **Reads:** ResolvedConnection, StartReservation, DiscoveryPath, HostKey, TrustUse, TrustState.
- **Writes:** PreparedStart, TrustStart, TrustPermission, TrustSnapshot.
- **Preconditions:** Resolution occurs first; identity/reservation/discovery path agree; Jump path, if any, names an already-permitted registered current generation; Direct/Jump eligibility policy has been approved.
- **Postconditions:** Keyscan evidence is checked against accepted keys with ssh-keygen -F preflight before master launch. NONE, MISMATCH or unresolved eligibility produces no permission. On success atomically regenerates known_hosts from authoritative SQLite and returns PreparedStart containing the ORIGINAL resolved and reservation values plus matching private TrustStart/digest/start ID. Core delegates owned I/O workers and keeps its mailbox serviced.
- **Invariants:** INV-1, INV-2, INV-4, INV-5, INV-8.
- **Errors:** HostKeyUntrusted, HostKeyMismatch, unavailable policy, stale binding or storage/generation/atomic replacement failure yields no PreparedStart; preserves accepted key and forbids stale snapshot/user trust fallback. E1 retains attempt metadata to retire a failed consumed reservation without reconstructing authority. Classification is not human stderr or exit 255.
- **Idempotency:** Submit a reservation once. Success hands it back for one compilation; every startup/reconnect/new master requires a fresh reservation and regeneration. Generated-file edits never enter SQLite.

### `PreparedStart::into_prepared_start(self) -> (E2::ResolvedConnection, E1::StartReservation, TrustStart)`

- **Intent:** Return the original owned start inputs with their matching positive trust result.
- **Owner:** `src/application/trust.rs`.
- **Reads:** PreparedStart.
- **Writes:** None.
- **Preconditions:** A successful owner-created PreparedStart exists.
- **Postconditions:** Moves the original resolution and reservation together with exactly their TrustStart; no scalar reconstruction, clone or dropped ownership.
- **Invariants:** INV-4, INV-5.
- **Errors:** Infallible; no I/O.
- **Idempotency:** Single consumption; cannot authorize a second start.

### `TrustStart::inspect_trust_start(&self) -> TrustStartView<'_>`

- **Intent:** Supply compilation with the exact permission and authoritative snapshot binding.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustStart.
- **Writes:** None.
- **Preconditions:** A live TrustGate-created TrustStart exists.
- **Postconditions:** Borrows matching start ID, identity, endpoint uses, store digest, generated path and generation time; a constructed view cannot stand in for TrustStart.
- **Invariants:** INV-4, INV-5.
- **Errors:** Infallible; no I/O or allocation.
- **Idempotency:** Repeatable borrowed read; wrapper remains non-cloneable.

### `TrustExecutor::prepare_change(&self, operation_id: Uuid, resolved: E2::ResolvedConnection, change: TrustChange) -> Result<PreparedTrustChange, E6::Failure>`

- **Intent:** Translate an explicit reviewed trust intent into an exact expected-state change without applying it.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustChange, TrustEvidence, HostKey, ResolvedConnection.
- **Writes:** PreparedTrustChange.
- **Preconditions:** E4 supplies external Operation identity; evidence/endpoint/route match; expected authoritative key set is current; Verification has exact independent evidence, Adoption explicit lookup provenance, and TOFU retains warning/confirmation requirement.
- **Postconditions:** Creates private operation/identity-bound expected and replacement sets with accepted fields and correct path/state/provenance; replacement alone never means Verified; deletion creates no substitute key. Does not decide consent, status or success.
- **Invariants:** INV-1, INV-2, INV-3.
- **Errors:** Validation, missing evidence, mismatch, stale expected keys or storage failure commits nothing. Missing local policy prevents executable admission, not an invented remote script.
- **Idempotency:** Equivalent current state/input yields equivalent intent, not an execution permit; no acceptance on replay or cancellation.

### `PreparedTrustChange::prepared_trust_store_view(&self) -> PreparedTrustChangeView<'_>`

- **Intent:** Let local admission and Store inspect the exact trust-owner delta without accepting a forged projection.
- **Owner:** `src/application/trust.rs`.
- **Introducing ticket:** ADR-0006:01 implements only this infallible borrowed projection in the canonical trust module; it cannot mint PreparedTrustChange, grant execution or assume a later trust proof. E3:03 remains the sole semantic preparation producer.
- **Reads:** PreparedTrustChange.
- **Writes:** None.
- **Preconditions:** A live TrustCore-created PreparedTrustChange exists; a local admission reader may inspect it, while persistence separately requires the sealed E4 envelope.
- **Postconditions:** Borrows operation/Server/identity/endpoint, expected/replacement keys, path and evidence digest; neither E4 admission nor Store accepts a caller-created view instead of the original wrapper.
- **Invariants:** INV-1, INV-2, INV-3.
- **Errors:** Infallible; no I/O or allocation.
- **Idempotency:** Repeatable borrowed read; does not apply a mutation.

### `TrustExecutor::commit(&self, change: PreparedTrustChange, audit: E4::LocalTrustEnvelope) -> Result<TrustCommitObservation, E6::Failure>`

- **Intent:** Apply an explicitly engine-admitted local trust change with inseparable sanitized audit evidence.
- **Owner:** `src/application/trust.rs`.
- **Reads:** PreparedTrustChange, LocalTrustEnvelope, HostKey, AuditTail.
- **Writes:** HostKey; E4-authored committed-change fact under its sealed envelope; TrustCommitObservation.
- **Preconditions:** Approved E4 local lifecycle/atomic outcome/Verifier/read-only Reconciliation policy exists; matching operation/revision/expected keys and contiguous expected audit tail; durable write-ahead and active Runtime lease exist. No approved LocalTrustApproval variant exists today.
- **Postconditions:** Store transaction compares expected keys/tail, applies E3-authored delta, verifies read-back and appends E4-authored committed event ID/ordinal in sanitized_payload, all or none. Store creates plain immutable TrustCommitObservation; TrustCore forwards facts without minting verdict or connection permission. E4 keeps servicing stop/completion/lease mailboxes and owns final outcome.
- **Invariants:** INV-1, INV-2, INV-3, INV-6.
- **Errors:** CAS, ordinal conflict, FK/validation/storage failure rolls back key change and paired event. Insufficient/ambiguous proof is not success; inspect retained event/state instead of blind repeat. Cancellation before commit has no acceptance; after possible commit follow approved local outcome policy.
- **Idempotency:** Same operation/event identity and exact content returns original facts or conflict, never another commit/ordinal. Crash leaves both trust+audit committed or neither; local read-only Reconciliation is policy-gated, not dependent on fake remote reconnect.

### `TrustCommitObservation::into_facts(self) -> TrustCommitFacts`

- **Intent:** Deliver actual Store-created committed/read-back facts to the Operation owner without inventing a verdict.
- **Owner:** `src/application/trust.rs`.
- **Reads:** TrustCommitObservation.
- **Writes:** None.
- **Preconditions:** A Store-created immutable observation was returned by the authorized transaction.
- **Postconditions:** Moves operation ID, before/after keys, committed event ID/ordinal and digest exactly; these are observations, not authority and not Succeeded.
- **Invariants:** INV-6.
- **Errors:** Infallible; no I/O.
- **Idempotency:** Single consumption; no effects or success promotion.

### Invariants

| ID    | Statement                                                                                                                                                                               | Scope                                                  | Enforced by                                                         | Broken if                                                   | Proof                                                                     |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | HostKey, TrustCore                                     | Real versioned Store migrations/reads                               | Missing table/read failure is treated as no key             | First proof: 01; future test `trust_authoritative_store_migration`        |
| INV-2 | Review, drift, discovery, connectivity and generated-file edits never accept, replace, delete or promote authoritative Host Keys.                                                       | HostKey, TrustReview, TrustEvidence                    | Read-only query protocol and sole sealed executor mutation boundary | Discovery/login/drift becomes a direct write                | First proof: 03; future test `trust_readonly_review_provenance`           |
| INV-3 | Verified evidence requires exact independent complete SHA256 comparison; Adoption and TOFU retain their distinct provenance and cannot imply Verified.                                  | TrustEvidence, HostKey                                 | Private evidence constructor and exact equality                     | Partial match or successful login promotes evidence         | First proof: 03; future test `trust_exact_fingerprint_evidence`           |
| INV-4 | NONE, MISMATCH and unresolved Direct/Jump eligibility cannot produce TrustStart or authorize a master.                                                                                  | TrustPermission, TrustState, TrustStart                | Private gate constructor and explicit policy gate                   | UNVERIFIED gets an invented permissive membership           | First proof: 04; future test `trust_prestart_absent_mismatch_eligibility` |
| INV-5 | Each positive start owns its original resolution/reservation and a fresh atomic SQLite-derived matching snapshot; failed regeneration never uses an old file.                           | PreparedStart, TrustSnapshot, TrustStart               | Consuming handback and atomic per-start regeneration                | Reservation is reconstructed or prior snapshot reused       | First proof: 04; future test `trust_atomic_snapshot_handback`             |
| INV-6 | Only explicit engine-admitted trust changes can commit, and each key delta/read-back plus its E4-authored event and ordinal commit atomically at expected state/tail without secrets.   | PreparedTrustChange, HostKey, TrustCommitObservation   | Private prepared wrapper, sealed E4 envelope and Store transaction  | Generic vectors or half trust/audit commit bypass admission | First proof: 05; future test `trust_atomic_local_audit_commit`            |
| INV-7 | Routed discovery executes on the registered already-permitted Jump generation and never falls back locally or inherits target trust.                                                    | DiscoveryPath, TrustEvidence                           | E1 current-generation route capability and bound evidence           | Jump verification silently verifies target                  | First proof: 06; future test `trust_registered_jump_discovery`            |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | TrustCore, TrustPorts, TrustGate, TrustExecutor, Trust | Private capability constructors and consuming port decomposition    | Assembling routed façade creates new actor/state            | First proof: 01; future test `trust_single_core_ports`                    |

### Ordered flows

1. **Bootstrap and review (01 → 03):** external E6:01 compatible primary Store → `TrustCore::spawn` → `TrustPorts::into_trust_ports` moves three handles to SAME core → external E2:01 Configuration + optional routes None → `Trust::assemble`. External query/Server ID → `Trust::review` first resolves actual saved identity via E2:01, then reads actual stored keys, explicitly selected Adoption source with ssh-keygen -F (hashed entries included), direct keyscan, exact external independent fingerprint comparison, or recorded-source drift evidence. Core assigns request ID and TrustPending, starts owned I/O worker, immediately services mailbox; TrustCompletion binds request/identity/revision before reply. `TrustEvidence::inspect_evidence` gives GTK immutable fingerprint/provenance. No stage accepts trust; failed/cancelled requests cannot accept. GTK receives loading, actual empty/unresolved/provenance/drift/stale/error through glib::MainContext and remains keyboard usable.
2. **Master preparation (01 → 03 → 04):** human 02 supplies precise Direct/Jump eligibility; external E1:02 resolution plus original StartReservation and permitted DiscoveryPath → `TrustGate::prepare` retains owner attempt binding and runs owned scan/snapshot worker while servicing completions. NONE/mismatch/policy absence returns no positive output. Success compares actual keyscan via ssh-keygen -F preflight and atomically replaces known_hosts from SQLite, matching digest/start/identity → PreparedStart → `PreparedStart::into_prepared_start` returns original owned inputs plus TrustStart → `TrustStart::inspect_trust_start` allows E2:04 compile validation → external E1:02 master start. Startup/reconnect repeat regeneration with NEW reservations; old files never authorize fallback.
3. **Explicit local change (03 → external E4:05 → 05):** external reviewed TrustChange and E4 Operation identifier → E2:01 resolution → E3:03 `TrustExecutor::prepare_change` produces a sealed expected-state delta without acceptance; `PreparedTrustChange::prepared_trust_store_view` allows borrowed exact preflight. External E4:05 `OperationEngine::prepare_local_trust` invokes that same semantic preparation, validates the approved E4:01 local policy and retains LocalTrustPlan; `inspect_local_trust_plan` exposes its exact immutable confirmation revision. Missing policy is unavailable, not a remote script or approximate plan. External explicit OperationDecision → E4:05 `admit_local_trust` rechecks current assumptions, acquires E6:03 lease, durably records RUNNING using E4:04 audit, and issues LocalTrustAdmitted while retaining the real lease in its active-work map. E4:05 `into_local_trust_admitted` moves the ORIGINAL PreparedTrustChange and LocalTrustEnvelope → E3:05 private invocation integration in `src/application/operations.rs` registers the E4-owned commit worker and invokes `TrustExecutor::commit`, which delegates the owned Store transaction without blocking its mailbox → Store invokes E3 `prepared_trust_store_view` and E4 `local_trust_store_view`, compares keys/audit tail and contiguous payload ordinal, atomically commits keys + paired event/read-back → Store-created TrustCommitObservation → E4-owned commit worker integrated by E3:05 awaits the ACTUAL E3 commit future and local-work join → `OperationEngine::finish_local_trust(&self, operation_id: Uuid, result: Result<E3::TrustCommitObservation, E6::Failure>) -> Result<Operation, E6::Failure>` validates tracked IDs/tail and Store settlement, consumes observation through `into_facts`, applies only the approved local outcome and releases the retained lease after joined proof. An arbitrary caller error is NOT evidence that owned work ended. Cancellation before effects commits no acceptance; possible committed effects follow approved uncertainty/read-only Reconciliation, never blind replay. Later public E4 plan/decide reuse these SAME local stages, not a second GTK mutation workflow. E4:05 introduces admission, finish handling and work-set validation only; it cannot invoke the later E3 commit before this E3:05 integration exists. This private refinement transfers no E4 ownership and introduces no second public function or mutation workflow.
4. **Routed review (03 → 04 → 06):** external E1:04 registered RouteAccess → `Trust::assemble` with Some over SAME existing core; review resolves target first, obtains actual permitted already-connected prefix generation, opens E1:03 KeyScan consumer and consumes its typed event stream. Target keyscan executes on the trusted registered Jump Server, never silently locally. E1 target/generation validation and joined cleanup protect cancellation; errors identify target AND route. Target remains NONE/unaccepted unless its own previously committed decision exists; Jump trust supplies no target proof. `inspect_evidence` presents the result and explicit separate target acceptance choices; human 02, E4:01 and 05 remain their own gates.
5. **Final GTK trust-decision journey (03 → external E4:05 → 05 → external E4:08 → 06):** external explicit user acceptance/replacement/deletion request and reviewed evidence → actual public OperationEngine plan/decide integration (ADR-0004:07/08), reusing the SAME prepare/admit/finish stages above → E3 atomic commit/read-back → durable truthful Operation outcome and lease release → real GTK loading/confirmation/keyboard rejection/cancel/details/error/stale presentation → external explicit Connect through E1:02 gate/compile/master → actual VTE terminal. Backend-only 05 proof does not claim this public enabled UI; final 06 requires ADR-0004:08. Missing human policy keeps the relevant action needs-info/unavailable and never becomes successful acceptance.

## Decision

- Select Sol's sole trust actor with separate core/gate/executor capabilities and optional routed façade over the same owner.
- Integrate sealed borrowed projections, original-reservation handback and still-serviced owned-worker completions from repaired Sol.
- Graft Terra's explicit payload ordinal discipline into the inseparable E4-authored local trust/audit transaction; observations never become authority.
- Reject independent trust caches/known_hosts authority and Terra's whole worker-process shape; accept explicit policy gates rather than invent trust or local outcome semantics.

## Open questions

1. What exact accepted Trust State membership authorizes Direct and Jump use, especially explicitly accepted UNVERIFIED? Human ticket 02 owns this policy.
2. What exact local-only lifecycle, atomic outcome/audit rule, Verifier and read-only Reconciliation policy applies to trust Operations? External ADR-0004:01 owns the decision; no local mutation is enabled before it.
