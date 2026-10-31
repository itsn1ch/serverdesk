# Single-Instance Ownership, Resource Locks, and Startup Recovery

Status: ready-for-agent
Source: ADR-0006; connected ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005; product specification §§4.3–4.10, 9, 12, 13, 14, 16, 18; GLOSSARY.md.

## Problem Statement

A Developer managing personal Linux Servers must not have two application processes racing over one SQLite database, application ControlMasters, or resource locks.
Different inventory aliases can refer to the same Connection Identity and therefore must not bypass conflict protection.
Global serialization would unnecessarily block independent resources, while silently queuing conflicting mutations would hide when a request actually takes effect.
Local locks cannot prevent another administrator, terminal session, or tool from changing remote state.
After a crash, a lost process, orphan master, or interrupted Operation is not proof that a mutation failed or never ran.
Startup must restore safe ownership without adopting old masters or replaying changes whose outcomes are unknown.

## Solution

Use GtkApplication single-instance ownership for the application process that owns SQLite, ControlMasters, and in-memory resource locks.
Arbitrate application Operations by the accepted pair of Connection Identity and resource scope, rejecting conflicting requests explicitly instead of queueing them.
Allow independently scoped Operations to proceed without a global mutation lock.
Use the Operation engine's preflight reads to detect external changes without claiming remote exclusivity or taking remote locks.
Retain locks while application-owned conflicting work is still active, including verification and cancellation cleanup.
Before new master ownership at startup, request exit for each leftover application socket and remove old sockets without adoption.
Convert interrupted RUNNING and VERIFYING Operations to UNKNOWN, retain their evidence, and perform read-only Reconciliation after the affected Server reconnects without replaying the mutation.
The testing approach is approved, but the acceptance criteria below are planned requirements rather than evidence of existing implementation or passing tests.

## User Stories

1. **US-001: Keep a single application owner**

   - **Story:** As a Developer managing personal Linux Servers, I want duplicate launches to share one application owner so that my database, ControlMasters, and resource locks are never independently owned by competing application processes.
   - **Basis:** Derived: ADR-0006, product §§9.7, 12.3.
   - **Acceptance criteria:**
     - **US-001-AC1:** Given an active ServerDesk GtkApplication instance, when I launch the application again, then no second independent application owner opens its own operational ownership of the same SQLite state, application ControlMasters, or lock registry.
     - **US-001-AC2:** Given simultaneous launch attempts, when application ownership is established, then only one owner performs application startup recovery and admits new managed connection or mutation work.
     - **US-001-AC3:** Given an active first instance with a locked resource, when a second launch occurs, then the duplicate launch does not clear that lock, start a competing master, or interrupt the first instance's running Operation.
     - **US-001-AC4:** Given that the previous application owner has exited, when a later launch becomes the owner, then it performs the startup recovery workflow rather than assuming the previous in-memory locks or master ownership survived.

2. **US-002: Make inventory aliases share resource conflict protection**

   - **Story:** As a Developer managing personal Linux Servers, I want aliases of the same connection to share resource locks so that changing the selected inventory entry cannot bypass an active Operation.
   - **Basis:** Derived: ADR-0006, product §§4.3, 9.7 and glossary Server, Connection Identity.
   - **Acceptance criteria:**
     - **US-002-AC1:** Given two Server entries with the same host, port, user, Jump Server route, and authentication context, when Operations target the same underlying resource scope through those entries, then they resolve to the same Connection Identity/resource-scope conflict rather than separate inventory-ID locks.
     - **US-002-AC2:** Given an admitted Operation with an active resource lock, when another Operation requests that same Connection Identity and resource scope, then the second request is explicitly rejected before conflicting mutation execution begins.
     - **US-002-AC3:** Given concurrent admission attempts for the same Connection Identity/resource-scope key, when arbitration completes, then at most one conflicting Operation is admitted and the other receives the explicit conflict result.
     - **US-002-AC4:** Given an inventory alias rename or a change of selected Server view without changing Connection Identity, when another request targets the already locked resource, then those presentation changes do not grant a second lock owner.

3. **US-003: Avoid globally serializing independent resources**

   - **Story:** As a Developer managing personal Linux Servers, I want independent Operations to proceed so that one busy resource does not unnecessarily block all administration.
   - **Basis:** Derived: ADR-0006 resource-scoped lock key and product §§9.7, 12.1.
   - **Acceptance criteria:**
     - **US-003-AC1:** Given an active Operation on one resource scope and another otherwise eligible Operation on an independent scope of the same Connection Identity, when the second request is made, then the first lock alone does not reject or defer it as a global per-connection mutation lock.
     - **US-003-AC2:** Given an active Operation for one Connection Identity, when another otherwise eligible Operation targets an independent Connection Identity/resource-scope key, then the first lock alone does not globally serialize the two.
     - **US-003-AC3:** Given independent eligible Operations with work still active, when both are admitted, then their externally observable execution can overlap without waiting for the unrelated Operation's completion solely because of resource locking.

4. **US-004: Reject conflicts visibly instead of queueing them**

   - **Story:** As a Developer managing personal Linux Servers, I want a conflicting request rejected immediately and explained so that it cannot run later after I have forgotten it.
   - **Basis:** Derived: ADR-0006, product §§9.7, 14.
   - **Acceptance criteria:**
     - **US-004-AC1:** Given a currently locked resource, when I request a conflicting Operation, then the UI explicitly identifies the affected Server/resource and the conflict instead of appearing to accept a future execution.
     - **US-004-AC2:** Given a rejected conflicting request, when the original lock is later released, then the rejected request does not automatically execute and no hidden mutation queue replays it.
     - **US-004-AC3:** Given a conflict rejection, when I inspect details, then the UI indicates that this rejected request has not performed its remote mutation and provides the related Operation details and available next actions where applicable.
     - **US-004-AC4:** Given that the conflicting Operation later completes, when I choose to attempt the action again, then it is a new explicit request subject to current validation, preflight, confirmation, and lock arbitration rather than a resumed hidden queue item.

5. **US-005: Respect external changes and resource ownership boundaries**

   - **Story:** As a Developer managing personal Linux Servers, I want preflight to detect external changes without pretending local locks own the remote system so that concurrent work outside ServerDesk is not silently overwritten.
   - **Basis:** Derived: ADR-0006, product §§9.3, 9.7, 18 and glossary Managed Resource.
   - **Acceptance criteria:**
     - **US-005-AC1:** Given a resource changed outside ServerDesk before its Operation preflight, when the operation-specific read detects changed assumptions, then that evidence is presented to the Operation workflow and the mutation does not silently proceed under stale assumptions.
     - **US-005-AC2:** Given a local resource lock held by ServerDesk, when another tool changes the remote resource, then ServerDesk does not claim that its local lock prevented the change or that the lock proves the remote state is unchanged.
     - **US-005-AC3:** Given an Operation using this lock system, when its remote behavior is observed, then no remote lock is introduced by this epic and the local lock does not establish a remote-exclusive ownership guarantee.
     - **US-005-AC4:** Given a remote resource that is not a Managed Resource, when an Operation is proposed, then holding or acquiring a local lock does not authorize changing it without the explicit consent required by the product ownership rule.
     - **US-005-AC5:** Given an external change after an earlier preflight observation, when final evidence is insufficient for the operation-specific Verifier, then the local lock is not substituted for verification and the shared engine does not report false success.

6. **US-006: Release locks only after owned conflicting work is finished**

   - **Story:** As a Developer managing personal Linux Servers, I want locks to cover active work and be cleaned up afterwards so that failures neither permit overlapping owned mutations nor leave resources permanently locked.
   - **Basis:** Derived: ADR-0006, product §§9.2, 9.5, 9.7, 12.3, 18.
   - **Acceptance criteria:**
     - **US-006-AC1:** Given an admitted mutation whose execution has finished but whose owned verification is still active, when a conflicting Operation is requested, then it remains rejected until the owned conflicting workflow has finished rather than being admitted merely because the command exited.
     - **US-006-AC2:** Given a completed and cleaned-up owned Operation, when a later explicit request targets the same scope, then no stale in-memory lock from that Operation prevents admission.
     - **US-006-AC3:** Given an execution error or transport failure with application-owned conflicting work still active, when the error is displayed, then error presentation alone does not release the lock prematurely.
     - **US-006-AC4:** Given an error path after all application-owned conflicting execution, verification, and cleanup have ended, when another explicit eligible request targets the scope, then an abandoned lock from the ended workflow does not block it.
     - **US-006-AC5:** Given an UNKNOWN outcome, when lock ownership is assessed, then UNKNOWN is neither treated as proof of no remote effects nor used by itself as a reason to retain a lock after all application-owned conflicting work has ended; subsequent action still uses current preflight and the shared Operation safety rules.

7. **US-007: Cancel without confusing intent, outcome, or lock release**

   - **Story:** As a Developer managing personal Linux Servers, I want cancellation to stop owned work safely without claiming that a remote effect was undone so that I can make informed decisions after an interrupted request.
   - **Basis:** Derived: product §§9.2, 9.4–9.7, 14, 18.
   - **Acceptance criteria:**
     - **US-007-AC1:** Given a cancellation request while owned work can still conflict, when cancellation is requested or acknowledged by the UI, then the lock is not released solely because the request was made or a progress surface closed.
     - **US-007-AC2:** Given cancellation before any remote effect and evidence confirming that no remote effect occurred, when the shared Operation workflow finishes, then CANCELLED may be reported and the lock is released after owned conflicting work is cleaned up.
     - **US-007-AC3:** Given cancellation after remote execution may have begun and insufficient evidence of its outcome, when cancellation finishes, then the Operation is UNKNOWN rather than CANCELLED or presumed FAILED, and the UI explains that a change may have occurred.
     - **US-007-AC4:** Given the cancelled workflow has no remaining application-owned conflicting work, when cleanup ends, then a subsequent explicit request is not blocked by an abandoned lock and is not an automatic retry of the cancelled request.

8. **US-008: Remove orphan masters and old sockets without adoption**

   - **Story:** As a Developer managing personal Linux Servers, I want a new application owner to clean up old transport ownership so that a previous process cannot silently supply my current connection.
   - **Basis:** Derived: ADR-0006 Consequences, product §§4.7, 9.4, 12.3, 16.3.
   - **Acceptance criteria:**
     - **US-008-AC1:** Given leftover application control sockets at startup, when the sole owner performs recovery, then each remaining socket receives an ssh -O exit request and the old socket is removed before it can be used for a new managed connection.
     - **US-008-AC2:** Given a still-responsive orphan application master, when startup finds its socket, then ServerDesk requests its exit and never adopts it as the new owner's active master.
     - **US-008-AC3:** Given an old socket without a live responsive master, when recovery handles it, then the stale socket is removed and is not interpreted as evidence of CONNECTED.
     - **US-008-AC4:** Given connect_on_startup on an affected Server, when startup recovery is in progress, then that setting does not bypass orphan/socket cleanup or cause a new managed master to coexist with an uncleared previous owner for that identity.
     - **US-008-AC5:** Given cleanup cannot complete, when startup presents the failure, then it identifies the affected connection/resource and available corrective action without claiming successful cleanup, adopting the old master, or admitting affected work on assumed ownership.
     - **US-008-AC6:** Given startup-owned cleanup, when application SSH resources are inspected, then cleanup is confined to application-owned runtime masters/sockets rather than treating unrelated user SSH connections as resources to adopt or terminate.

9. **US-009: Recover interrupted Operations as Unknown Outcome**

   - **Story:** As a Developer managing personal Linux Servers, I want crash-interrupted work recorded as unknown so that missing local execution does not hide a possible remote change.
   - **Basis:** Derived: ADR-0006 Consequences, product §§9.4, 9.8, 13.1–13.2, 14, 16.3.
   - **Acceptance criteria:**
     - **US-009-AC1:** Given an Operation durably left RUNNING by the previous process, when startup recovery reads it, then its recovered result is UNKNOWN and it is marked as needing Reconciliation instead of being treated as failed, cancelled, or successfully complete.
     - **US-009-AC2:** Given an Operation durably left VERIFYING by the previous process, when startup recovery reads it, then its recovered result is UNKNOWN and it is marked as needing Reconciliation even if the remote command had exited successfully.
     - **US-009-AC3:** Given recovery of an interrupted Operation, when its details are shown, then the original request, available verification evidence, and Audit Trail remain available and the UI says a remote change may have occurred.
     - **US-009-AC4:** Given interrupted Operations and enabled startup connection, when the new owner starts managed work, then startup recovery establishes their UNKNOWN/reconciliation-needed status rather than presenting the old RUNNING/VERIFYING records as current owned execution.
     - **US-009-AC5:** Given failure to persist required recovery changes, when startup recovery completes unsuccessfully, then the affected workflow reports StorageError and does not claim that the interrupted outcome has been safely recovered or reconciled.
     - **US-009-AC6:** Given a crash followed by more than one later application start, when the same unresolved Operation is inspected, then it is not erased, silently retried, or converted to success merely because no previous lock or local process remains.

10. **US-010: Reconcile by reading, never by replaying a mutation**

- **Story:** As a Developer managing personal Linux Servers, I want the application to determine an interrupted Operation's actual outcome from current remote evidence so that recovery does not perform the administrative change twice.
- **Basis:** Derived: product §§9.4–9.5, 14, 18 and glossary Reconciliation, Unknown Outcome, Verifier.
- **Acceptance criteria:**
  - **US-010-AC1:** Given an UNKNOWN interrupted Operation requiring Reconciliation, when the affected Server has not reconnected, then the record remains unresolved and startup does not replay its mutation to obtain an answer.
  - **US-010-AC2:** Given that the affected Server reconnects under the normal current trust/authentication policy, when Reconciliation runs, then it reads remote state using that Operation type's accepted Reconciliation predicate and performs no replay of the original mutation.
  - **US-010-AC3:** Given sufficient evidence meeting the operation-specific success predicate, when read-only Reconciliation completes, then the outcome may become SUCCEEDED and its evidence is available in Operation details.
  - **US-010-AC4:** Given sufficient evidence establishing the operation-specific failure predicate, when read-only Reconciliation completes, then the outcome may become FAILED without repeating the mutation.
  - **US-010-AC5:** Given unavailable, ambiguous, or insufficient evidence, including a failed reconnect or a remote reboot that invalidates evidence, when Reconciliation cannot determine the result, then the outcome remains UNKNOWN rather than fabricating success or failure.
  - **US-010-AC6:** Given an unresolved or reconciled Operation, when recovery finishes, then there is no automatic mutation retry; any later mutation requires a new explicit request through current validation, preflight, confirmation where required, and lock arbitration.
  - **US-010-AC7:** Given a proposed Operation type without its own Verifier and Reconciliation predicate, when its availability is evaluated, then startup recovery does not supply a generic success fallback and the shared engine's prerequisite for enabling that mutation remains in force.

11. **US-011: Present ownership and recovery without freezing GTK**

- **Story:** As a Developer managing personal Linux Servers, I want understandable recovery and conflict states while the desktop remains responsive so that I can inspect uncertain work instead of guessing what happened.
- **Basis:** Derived: product §§4.8, 12.2–12.3, 14, 18 and glossary Connection State, Unknown Outcome.
- **Acceptance criteria:**
  - **US-011-AC1:** Given startup cleanup, recovery persistence, preflight, or Reconciliation is running, when I interact with GTK, then the main thread remains responsive and affected work has a loading/recovery presentation rather than an unsupported success claim.
  - **US-011-AC2:** Given no interrupted Operations, when recovery-related results are shown, then an honest empty state is available; given unresolved Operations, then UNKNOWN and unavailable or stale evidence remain visible rather than being presented as Server downtime.
  - **US-011-AC3:** Given a cleanup failure, storage failure, conflict, or unresolved outcome, when the error is displayed, then it identifies what failed, the affected Server/resource, whether a change may have occurred, available actions, and related Operation details where applicable.
  - **US-011-AC4:** Given recovery details or a rejected-conflict surface, when I use keyboard navigation, then I can reach the details and available actions without a pointer or a forced automatic mutation retry.

## Implementation Decisions

- GtkApplication single-instance ownership will provide one application process responsible for SQLite, application ControlMasters, and the in-memory resource-lock registry.
- The application layer will own lifecycle and scheduling, the connection manager will own masters/sessions, the Operation engine will own Operation state and conflict admission, and SQLite will own persisted inventory, trust, and Audit Trail state.
- Resource locks will be in memory and keyed by the accepted pair of Connection Identity and resource scope; this epic adds no lock table, persisted lease, remote lock, or new schema.
- Connection Identity will retain the accepted host, port, user, Jump Server route, and authentication-context definition; inventory aliases will not create independent lock domains for the same identity and resource.
- Conflicts will be explicitly rejected rather than queued; unrelated resource scopes will not be serialized by a global mutation lock.
- External changes will be detected through the shared Operation preflight reads, not prevented or authorized by the existence of a local lock.
- Lock lifecycle will follow owned conflicting execution, verification, and cleanup rather than merely dialog lifetime, command exit, or cancellation intent.
- UNKNOWN will remain a statement about evidence of effects, not proof that a remote process stopped or that no remote change occurred.
- Startup will clean orphan masters and old application sockets without adoption, then establish interrupted RUNNING/VERIFYING Operations as UNKNOWN requiring Reconciliation before presenting fresh owned execution.
- Startup socket handling will use ssh -O exit and removal; normal master creation, child shutdown, parent-death handling, and healthy connection checks remain the ADR-0001 transport contract.
- Reconciliation will run read-only after reconnect and use the operation-specific predicates owned by ADR-0004; no recovery path will replay an administrative mutation automatically.
- SQLite recovery changes will follow the accepted transaction, foreign-key, versioned-migration, append-only Audit Trail, and secret-exclusion rules without adding fields beyond the accepted model.
- GTK will own widgets and presentation; Tokio/application work will communicate through channels scheduled to glib::MainContext without accessing GTK widgets from Tokio.

## Testing Decisions

The user approved the primary seam: presentation-facing application workflows, the same workflows invoked from GTK, backed by actual system OpenSSH and SQLite in isolated Incus/LXD Ubuntu 24.04 fixtures with systemd 255.
Use actual GTK smoke for duplicate-launch ownership, responsive startup/recovery surfaces, explicit conflict rejection, keyboard access, and unknown-outcome details.
Use narrow pure behavior tests for Connection Identity equivalence, resource-scope conflict admission, rejection rather than queueing, and lock lifecycle transitions.
These are planned tests; no authored harness exists and this documentation task runs no product tests.

- Start the packaged/application entry point twice while the first owner holds a live master and resource lock; observe one owner, no startup cleanup by the duplicate, no second master, and no independent mutation admission.
- Submit concurrent conflicting requests through two inventory aliases with the same Connection Identity and prove at most one executes against the fixture resource.
- Submit independent scopes on one identity and independent identities while work is deliberately still running; observe execution overlap rather than asserting only that two requests were accepted.
- Reject a conflict, release the original lock, and observe that the rejected mutation never runs without a fresh explicit request.
- Change a fixture resource externally before preflight and prove the shared workflow detects the changed evidence; separately demonstrate that the local lock is not a remote lock or an authorization grant.
- Exercise success, execution error, verification error, disconnect, and cancellation while owned work is active; prove conflicts remain blocked until owned conflicting work ends and that completed cleanup leaves no stale lock.
- Exercise cancellation before effects and cancellation after a mutation may have begun; preserve the distinction between confirmed CANCELLED and UNKNOWN without using lock release as evidence of no effect.
- Terminate the application during a real RUNNING mutation and during VERIFYING, then restart against its actual SQLite state; observe UNKNOWN, retained audit evidence, orphan cleanup, and no mutation replay.
- Include a responsive orphan master and a dead stale socket, observe exit requests and removal, and prove no recovered connection adopts the old master even with connect_on_startup enabled.
- Inject cleanup/storage failure and inspect the affected UI and inhibited ownership-dependent work rather than treating absence of a crash as successful recovery.
- Reconnect to a real fixture and exercise read-only Reconciliation outcomes SUCCEEDED, FAILED, and UNKNOWN using operation-specific predicates supplied by ADR-0004; observe that the original mutation is not issued again.
- Disconnect or reboot the fixture during a mutation and during recovery reads; verify uncertain evidence does not become false success or assumed failure.
- Inject disk-full only within an isolated container test filesystem and verify honest failure of recovery persistence without deleting the unresolved record or leaking secrets into error/audit payloads.
- Inspect permanent Operation evidence after repeated starts; no interrupted request or event is automatically removed, and no in-memory lock persistence is assumed across process death.

## Out of Scope

- Normal SSH connection/disconnect, ControlMaster creation, authentication, consumers, parent-death signaling, and Connection State derivation belong to ADR-0001.
- Generated runtime configuration and static SSH config import belong to ADR-0002; current Host Key eligibility and trust acceptance belong to ADR-0003.
- Fixed POSIX templates, argument quoting, privilege checks, mutation classification, confirmation rules, operation-specific Verifier internals, and concrete Reconciliation predicates belong to ADR-0004.
- Collector scheduling, buffers, sample Freshness, and normal Collector stop behavior belong to ADR-0005.
- Remote locks, distributed ownership, cross-application exclusion, globally serializing all resources, persisted lock leases, hidden mutation queues, and automatic mutation retry are not added.
- This epic does not define a hierarchical lock taxonomy, parent/child scope expansion, or a new canonicalization algorithm unsupported by the accepted sources.
- Terminal-entered commands remain outside the Operation engine and Audit Trail and are not presented as protected by application resource locks.
- Inventory removal does not delete a remote Server; this epic does not expand inventory management or remote resource ownership.
- Audit export/retention, multi-server operations, hosting management, and long-term availability monitoring remain outside MVP scope.

## Further Notes

1. **Cross-epic dependencies.** ADR-0001 supplies normal master/session lifecycle and real reconnect; ADR-0002 supplies isolated generated configuration; ADR-0003 supplies current Host Key gating before reconnect or a fresh master; ADR-0004 supplies write-ahead persistence, Operation outcomes, preflight, and operation-specific Verifier/Reconciliation evidence; ADR-0005 consumes the recovered connection but does not become a recovery owner. This epic owns single-instance admission, startup ordering, local lock arbitration, orphan non-adoption, and interrupted-state mapping.
2. **Resource-scope detail, not a new policy.** ADR-0006 and product §9.7 specify the lock key and rejection policy but do not publish a hierarchy or overlap algorithm. The committed criteria require the same underlying resource through the same identity, including inventory aliases, to conflict, and independently scoped Operations not to be globally serialized. Concrete resource types must supply their established operation-specific scope without inventing a universal lock taxonomy here.
3. **External-concurrency limit.** Preflight is an observation, not a remote lease. Changes after that observation remain possible; the operation-specific Verifier and UNKNOWN rules remain necessary. No claim of prevention or distributed atomicity is made.
4. **Cancellation limit.** Completion of local cleanup does not prove that an already-started remote mutation had no effect. The shared engine owns evidence-based CANCELLED versus UNKNOWN, while this epic owns preventing overlapping application-owned conflicting work and cleaning up in-memory locks.
5. **Nonblocking operational detail.** No numeric cleanup timeout, retry schedule, backoff, or remote-lock policy is selected by the sources. This draft does not invent one or promise that a particular external OpenSSH failure guarantees remote termination.
6. **Readiness.** No additional source-policy blocker was found for the committed ownership/recovery invariants. Publication readiness describes this specification, not implemented behavior; planned tests must later demonstrate the acceptance criteria. The accepted UNVERIFIED eligibility ambiguity is recorded in ADR-0003 rather than silently resolved by recovery.

# Design

## Usage

1. Real primary registration in the private Runtime GTK adapter opens Store and Runtime::begin; runtime.observe shows preadmission progress/failure, never fake Ready.
2. Actual cleanup/recover_interrupted reports feed runtime.admit. The same admitted generation is cheaply cloned through ReadyOwnership with shared Arc<RuntimeDirectory> for real E1/E5 composition, never re-registered or reminted.
3. src/main.rs calls Runtime-owned launch; Runtime and Monitoring each receive their own Connection subscription. E4 uses locks.arbitrate and sealed joined release; shutdown keeps Release/completion messages serviceable while workers join.

## Types

```rust
pub enum ErrorCategory {
    AuthenticationError, HostKeyMismatch, HostKeyUntrusted,
    UnregisteredProxyRoute, ConnectionTimeout, NetworkUnavailable,
    PermissionDenied, UnsupportedCapability, ValidationError,
    RemoteExecutionError, VerificationError, ConfigurationConflict,
    FileChangedExternally, StorageError, UnknownRemoteOutcome,
}
pub enum EffectKnowledge { NoChange, MayHaveChanged, ChangeVerified }
pub enum RecoveryAction {
    EditInventory, ReviewTrust, ReviewPolicy, ExplicitConnect, Disconnect,
    RetryRead, ReloadFile, RequestExplicitOverwrite, InspectOperation,
    CancelPendingWork, Close,
}
pub struct Failure {
    pub category: ErrorCategory,
    pub affected_resource: String,
    pub explanation: String,
    pub effects: EffectKnowledge,
    pub actions: Vec<RecoveryAction>,
    pub operation_id: Option<Uuid>,
}
pub enum Availability {
    Available,
    Unavailable {
        category: ErrorCategory,
        reason: String,
        required_decision: Option<String>,
    },
}
pub type Reply<T> = oneshot::Sender<Result<T, Failure>>;
pub struct Subscription<S, E> {
    pub snapshot: S,
    pub next_sequence: u64,
    pub events: mpsc::Receiver<E>,
}
pub enum Invocation {
    Desktop,
    Askpass(E1::AskpassRequest),
}
pub struct ApplicationPaths {
    pub database: PathBuf,
    pub runtime_directory: PathBuf,
    pub executable: PathBuf,
}
pub(crate) struct RuntimeDirectory {
    path: PathBuf,
    owner_uid: u32,
}
pub(crate) struct PrimaryOwnership {
    generation: Uuid,
    application_id: String,
}
pub(crate) struct StartupOwnership {
    generation: Uuid,
    runtime_directory: Arc<RuntimeDirectory>,
}
#[derive(Clone)]
pub(crate) struct ReadyOwnership {
    generation: Uuid,
    runtime_directory: Arc<RuntimeDirectory>,
}
pub enum StartupPhase {
    RegisteringOwner, OpeningStorage, CleaningTransport,
    RecoveringOperations, Ready, Blocked, Stopping, Stopped,
}
pub struct StartupSnapshot {
    pub phase: StartupPhase,
    pub failures: Vec<Failure>,
    pub unresolved_operations: Vec<Uuid>,
}
pub struct StartupChanged {
    pub sequence: u64,
    pub snapshot: StartupSnapshot,
}
pub struct Runtime { tx: mpsc::Sender<RuntimeMessage> }
pub(crate) struct LockAuthority { tx: mpsc::Sender<RuntimeMessage> }
pub(crate) struct ResourceLease {
    id: Uuid,
    owner_generation: Uuid,
    operation_id: Uuid,
    identity: E2::ConnectionIdentity,
    scopes: Vec<E4::ResourceScope>,
}
pub(crate) enum LockRequest {
    Acquire {
        operation_id: Uuid,
        identity: E2::ConnectionIdentity,
        scopes: Vec<E4::ResourceScope>,
    },
    Release {
        lease: ResourceLease,
        joined: E4::WorkJoined,
    },
}
pub(crate) enum LockResult {
    Acquired(ResourceLease),
    Released,
}
pub(crate) struct RuntimePorts {
    pub(crate) runtime: Runtime,
    pub(crate) locks: LockAuthority,
    pub(crate) startup: StartupOwnership,
}
pub struct DesktopServices {
    pub runtime: Runtime,
    pub configuration: E2::Configuration,
    pub trust: E3::Trust,
    pub connections: E1::ConnectionManager,
    pub operations: E4::OperationEngine,
    pub monitoring: E5::Monitoring,
}
pub(crate) struct LifecycleOwners {
    pub(crate) connections: E1::ConnectionManager,
    pub(crate) operations: E4::OperationEngine,
    pub(crate) monitoring: E5::Monitoring,
}
pub(crate) struct Store { tx: mpsc::Sender<StoreMessage> }
pub(crate) enum StoreRequest {
    Inventory(E2::InventoryStoreRequest),
    HostKeys(E3::TrustStoreRequest),
    Audit(E4::AuditStoreRequest),
    MonitoringSettings(E5::MonitoringStoreRequest),
    CommitLocalTrust {
        change: E3::PreparedTrustChange,
        audit: E4::LocalTrustEnvelope,
    },
}
pub(crate) enum StoreResult {
    Inventory(E2::InventoryStoreResult),
    HostKeys(Vec<E3::HostKey>),
    Audit(E4::AuditStoreResult),
    MonitoringSettings(E5::MonitoringStoreResult),
    LocalTrustCommitted(E3::TrustCommitObservation),
}
pub(crate) enum StoreMessage {
    Request { request: StoreRequest, reply: Reply<StoreResult> },
    Close { reply: Reply<()> },
}
pub(crate) enum RuntimeMessage {
    Observe { reply: Reply<Subscription<StartupSnapshot, StartupChanged>> },
    Lock { request: LockRequest, reply: Reply<LockResult> },
    Admit {
        cleanup: E1::CleanupReport,
        recovery: E4::RecoveryReport,
        reply: Reply<ReadyOwnership>,
    },
    InstallOwners { owners: LifecycleOwners, reply: Reply<()> },
    Shutdown { reply: Reply<StartupSnapshot> },
    ShutdownCompleted(ShutdownCompletion),
    OwnersSubscribed { result: Result<Subscription<E1::ConnectionView, E1::ConnectionChanged>, Failure> },
    ConnectionObserved(RuntimeConnectionUpdate),
    ReconciliationCompleted { result: Result<E4::ReconciliationBatch, Failure> },
}

pub fn launch(
    invocation: Invocation, paths: ApplicationPaths,
) -> Result<(), Failure>;

impl Store {
    pub(crate) async fn open(
        primary: &PrimaryOwnership,
        paths: &ApplicationPaths,
    ) -> Result<Store, Failure>;

    pub(crate) async fn request(
        &self, request: StoreRequest,
    ) -> Result<StoreResult, Failure>;

    pub(crate) async fn close(&self) -> Result<(), Failure>;
}
impl Runtime {
    pub(crate) fn begin(
        primary: PrimaryOwnership,
        store: Store,
        directory: RuntimeDirectory,
    ) -> RuntimePorts;

    pub(crate) async fn admit(
        &self,
        cleanup: E1::CleanupReport,
        recovery: E4::RecoveryReport,
    ) -> Result<ReadyOwnership, Failure>;

    pub(crate) async fn install_owners(
        &self, owners: LifecycleOwners,
    ) -> Result<(), Failure>;

    pub async fn observe(
        &self,
    ) -> Result<Subscription<StartupSnapshot, StartupChanged>, Failure>;

    pub async fn shutdown(&self) -> Result<StartupSnapshot, Failure>;
}
impl LockAuthority {
    pub(crate) async fn arbitrate(
        &self, request: LockRequest,
    ) -> Result<LockResult, Failure>;
}


pub(crate) struct RuntimeDirectoryView<'a> {
    pub(crate) path: &'a Path,
    pub(crate) owner_uid: u32,
}
pub(crate) struct OwnershipView<'a> {
    pub(crate) generation: Uuid,
    pub(crate) runtime_directory: &'a RuntimeDirectory,
}
pub(crate) struct ResourceLeaseView<'a> {
    pub(crate) id: Uuid,
    pub(crate) owner_generation: Uuid,
    pub(crate) operation_id: Uuid,
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) scopes: &'a [E4::ResourceScope],
}

impl RuntimeDirectory {
    pub(crate) fn inspect_runtime_directory(&self) -> RuntimeDirectoryView<'_>;
}
impl StartupOwnership {
    pub(crate) fn inspect_startup_ownership(&self) -> OwnershipView<'_>;
}
impl ReadyOwnership {
    pub(crate) fn inspect_ready_ownership(&self) -> OwnershipView<'_>;
}
impl ResourceLease {
    pub(crate) fn inspect_lease(&self) -> ResourceLeaseView<'_>;
}


pub(crate) enum ShutdownCompletion {
    MonitoringStopped { result: Result<(), Failure> },
    OperationsStopped { result: Result<E4::OperationQuiesced, Failure> },
    TransportStopped { result: Result<Vec<E1::ChannelEnd>, Failure> },
    StoreClosed { result: Result<(), Failure> },
}



pub(crate) enum RuntimeConnectionUpdate {
    Snapshot {
        view: E1::ConnectionView,
        next_sequence: u64,
    },
    Changed(E1::ConnectionChanged),
    StreamClosed(Failure),
}

// Canonical imported Store interfaces, defined in the named E2/E3/E4/E5 owner modules.
impl InventoryCommit {
    pub(crate) fn inventory_write_view(&self) -> InventoryWriteView<'_>;
}
impl SettingsCommit {
    pub(crate) fn settings_write_view(&self) -> SettingsWriteView<'_>;
}
impl AuditAppend {
    pub(crate) fn into_store_batch(self) -> AuditBatchData;
}
impl PreparedTrustChange {
    pub(crate) fn prepared_trust_store_view(&self) -> PreparedTrustChangeView<'_>;
}
impl LocalTrustEnvelope {
    pub(crate) fn local_trust_store_view(&self) -> LocalTrustEnvelopeView;
}
```

### Store and lifecycle boundaries

Store owns one operational SQLite connection and physical transactions, never semantic inventory/trust/Operation/settings decisions. Inventory commits arrive only as Configuration-created InventoryCommit; monitoring writes only as Monitoring-created SettingsCommit. Store reads their purpose-specific projections after accepting the sealed request. AuditAppend::into_store_batch and LocalTrustEnvelope::local_trust_store_view similarly expose data only after sealed authorization; arbitrary views and raw SQL are not Store inputs. Immutable Store-created TrustCommitObservation is an observation, not positive trust authority.

E4 alone allocates ordinals inside existing operation_events.sanitized_payload. Store atomically validates expected (event_id, ordinal) tail, UUID/content identity, ordinal uniqueness/contiguity and FK, with local trust changes/read-back and the exact paired audit ordinal in one transaction. Fold by validated ordinal, never SQL rowid, timestamps or UUID order. No audit UPDATE/DELETE, reset, truncation, new sequence column or retention stream exists. Failed/ambiguous writes force actual tail/history reads before a later append.

Runtime owns only current-generation in-memory arbitration and lifecycle. StartupOwnership and ResourceLease fields/constructors remain private and non-Clone. ReadyOwnership is Clone only for the SAME admitted generation and shares Arc<RuntimeDirectory> without copying its PathBuf; cloning cannot register a primary or mint a new generation. All borrowed projections remain non-authoritative. Primary capabilities come only from actual GTK registration. RuntimePorts contains handles plus existing startup authority; making that bundle readable does not expose a constructor for the authority. Every owner-loop handler launches workers then resumes its mailbox; only completion messages mutate lifecycle. Stopping rejects Acquire but continues Release until E4 acknowledges OperationQuiesced; E4 never hands back consumed WorkJoined. Runtime’s connection receiver is independently subscribed and cannot steal E5’s feed.

The common ADR-0006:01 cohort also implements exactly five infallible Store projections in their canonical modules: InventoryCommit::inventory_write_view, SettingsCommit::settings_write_view, AuditAppend::into_store_batch, PreparedTrustChange::prepared_trust_store_view and LocalTrustEnvelope::local_trust_store_view. They perform no I/O, authority construction, policy validation or domain write. This closes the buildable Store bootstrap without declaration-only method stubs or later-owner cycles; actual wrapper issuers remain later domain owner methods. Imported full function metadata below retains the invariant namespace of its canonical source epic.

## Contract

### Vocabulary

| Name                      | Kind   | Meaning                                                                                                   | Source of truth                                          | Writer            |
| ------------------------- | ------ | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------- |
| `ErrorCategory`           | enum   | Accepted domain failure categories                                                                        | `src/domain/error.rs` `ErrorCategory`                    | none (immutable)  |
| `EffectKnowledge`         | enum   | What evidence establishes about effects                                                                   | `src/domain/error.rs` `EffectKnowledge`                  | none (immutable)  |
| `RecoveryAction`          | enum   | Explicit available next action                                                                            | `src/domain/error.rs` `RecoveryAction`                   | none (immutable)  |
| `Failure`                 | value  | Sanitized actionable resource-specific error                                                              | `src/domain/error.rs` `Failure`                          | none (immutable)  |
| `Availability`            | enum   | Available or exact blocking reason                                                                        | `src/domain/error.rs` `Availability`                     | none (immutable)  |
| `Reply`                   | value  | Private asynchronous result plumbing                                                                      | `src/domain/error.rs` `Reply`                            | none (immutable)  |
| `Subscription`            | value  | Initial domain snapshot plus sequenced changes                                                            | `src/domain/error.rs` `Subscription`                     | none (immutable)  |
| `Invocation`              | enum   | Ordinary desktop or restricted askpass role                                                               | `src/application/runtime.rs` `Invocation`                | none (immutable)  |
| `ApplicationPaths`        | value  | Explicit local application locations                                                                      | `src/application/runtime.rs` `ApplicationPaths`          | none (immutable)  |
| `RuntimeDirectory`        | value  | Validated primary-owned runtime location                                                                  | `src/application/runtime.rs` `RuntimeDirectory`          | Runtime           |
| `PrimaryOwnership`        | value  | GtkApplication primary registration capability                                                            | `src/application/runtime.rs` `PrimaryOwnership`          | Runtime           |
| `StartupOwnership`        | value  | Current primary’s pre-admission capability                                                                | `src/application/runtime.rs` `StartupOwnership`          | Runtime           |
| `ReadyOwnership`          | value  | Current primary’s completed-safety capability                                                             | `src/application/runtime.rs` `ReadyOwnership`            | Runtime           |
| `StartupPhase`            | enum   | Actual ownership/recovery/shutdown lifecycle                                                              | `src/application/runtime.rs` `StartupPhase`              | none (immutable)  |
| `StartupSnapshot`         | value  | Phase, blockers and unresolved Operations                                                                 | `src/application/runtime.rs` `StartupSnapshot`           | none (immutable)  |
| `StartupChanged`          | event  | Runtime phase changed                                                                                     | `src/application/runtime.rs` `StartupChanged`            | Runtime           |
| `Runtime`                 | entity | Primary lifecycle/admission/lock owner                                                                    | `src/application/runtime.rs` `Runtime`                   | Runtime           |
| `LockAuthority`           | value  | Delegated resource arbitration capability                                                                 | `src/application/runtime.rs` `LockAuthority`             | none (immutable)  |
| `ResourceLease`           | entity | One admitted Operation’s in-memory scope ownership                                                        | `src/application/runtime.rs` `ResourceLease`             | Runtime           |
| `LockRequest`             | enum   | Atomic acquire or joined-work release                                                                     | `src/application/runtime.rs` `LockRequest`               | none (immutable)  |
| `LockResult`              | enum   | Acquired/released result                                                                                  | `src/application/runtime.rs` `LockResult`                | none (immutable)  |
| `RuntimePorts`            | value  | Early primary bootstrap capabilities                                                                      | `src/application/runtime.rs` `RuntimePorts`              | none (immutable)  |
| `DesktopServices`         | value  | Complete admitted desktop façade handles                                                                  | `src/application/runtime.rs` `DesktopServices`           | none (immutable)  |
| `LifecycleOwners`         | value  | Runtime shutdown participants                                                                             | `src/application/runtime.rs` `LifecycleOwners`           | none (immutable)  |
| `Store`                   | entity | Sole operational SQLite connection owner                                                                  | `src/infrastructure/storage/mod.rs` `Store`              | Store             |
| `StoreRequest`            | enum   | Private owner-authored persistence requests                                                               | `src/infrastructure/storage/mod.rs` `StoreRequest`       | none (immutable)  |
| `StoreResult`             | enum   | Private actual persisted-domain results                                                                   | `src/infrastructure/storage/mod.rs` `StoreResult`        | none (immutable)  |
| `StoreMessage`            | enum   | Private storage actor protocol                                                                            | `src/infrastructure/storage/mod.rs` `StoreMessage`       | none (immutable)  |
| `RuntimeMessage`          | enum   | Private runtime actor protocol                                                                            | `src/application/runtime.rs` `RuntimeMessage`            | none (immutable)  |
| `RuntimeDirectoryView`    | value  | Read-only validated runtime path/UID                                                                      | `src/application/runtime.rs` `RuntimeDirectoryView`      | none (immutable)  |
| `OwnershipView`           | value  | Read-only generation/directory binding                                                                    | `src/application/runtime.rs` `OwnershipView`             | none (immutable)  |
| `ResourceLeaseView`       | value  | Read-only actual lease binding                                                                            | `src/application/runtime.rs` `ResourceLeaseView`         | none (immutable)  |
| `ShutdownCompletion`      | enum   | Owned shutdown worker result                                                                              | `src/application/runtime.rs` `ShutdownCompletion`        | none (immutable)  |
| `RuntimeConnectionUpdate` | enum   | Runtime-owned Connection subscription input                                                               | `src/application/runtime.rs` `RuntimeConnectionUpdate`   | none (immutable)  |
| `ConnectionIdentity`      | value  | Foreign ConnectionIdentity boundary; declaration ADR-0006:01, actual producer remains canonical owner     | `src/domain/connection_identity.rs` `ConnectionIdentity` | Configuration     |
| `ResourceScope`           | value  | Foreign ResourceScope boundary; declaration ADR-0006:01, actual producer remains canonical owner          | `src/domain/operation.rs` `ResourceScope`                | OperationEngine   |
| `WorkJoined`              | value  | Foreign WorkJoined boundary; declaration ADR-0006:01, actual producer remains canonical owner             | `src/application/operations.rs` `WorkJoined`             | OperationEngine   |
| `CleanupReport`           | value  | Foreign CleanupReport boundary; declaration ADR-0006:01, actual producer remains canonical owner          | `src/infrastructure/ssh/mod.rs` `CleanupReport`          | ConnectionManager |
| `RecoveryReport`          | value  | Foreign RecoveryReport boundary; declaration ADR-0006:01, actual producer remains canonical owner         | `src/application/operations.rs` `RecoveryReport`         | OperationEngine   |
| `OperationQuiesced`       | value  | Foreign OperationQuiesced boundary; declaration ADR-0006:01, actual producer remains canonical owner      | `src/application/operations.rs` `OperationQuiesced`      | OperationEngine   |
| `ReconciliationBatch`     | value  | Foreign ReconciliationBatch boundary; declaration ADR-0006:01, actual producer remains canonical owner    | `src/application/operations.rs` `ReconciliationBatch`    | OperationEngine   |
| `ConnectionView`          | value  | Foreign ConnectionView boundary; declaration ADR-0006:01, actual producer remains canonical owner         | `src/domain/connection.rs` `ConnectionView`              | ConnectionManager |
| `ConnectionChanged`       | value  | Foreign ConnectionChanged boundary; declaration ADR-0006:01, actual producer remains canonical owner      | `src/domain/connection.rs` `ConnectionChanged`           | ConnectionManager |
| `Connection`              | value  | Foreign Connection boundary; declaration ADR-0006:01, actual producer remains canonical owner             | `src/domain/connection.rs` `Connection`                  | ConnectionManager |
| `InventoryCommit`         | value  | Foreign InventoryCommit boundary; declaration ADR-0006:01, actual producer remains canonical owner        | `src/application/configuration.rs` `InventoryCommit`     | Configuration     |
| `SettingsCommit`          | value  | Foreign SettingsCommit boundary; declaration ADR-0006:01, actual producer remains canonical owner         | `src/application/monitoring.rs` `SettingsCommit`         | Monitoring        |
| `AuditTail`               | value  | Foreign AuditTail boundary; declaration ADR-0006:01, actual producer remains canonical owner              | `src/domain/operation.rs` `AuditTail`                    | OperationEngine   |
| `LocalTrustEnvelope`      | value  | Foreign LocalTrustEnvelope boundary; declaration ADR-0006:01, actual producer remains canonical owner     | `src/application/operations.rs` `LocalTrustEnvelope`     | OperationEngine   |
| `PreparedTrustChange`     | value  | Foreign PreparedTrustChange boundary; declaration ADR-0006:01, actual producer remains canonical owner    | `src/application/trust.rs` `PreparedTrustChange`         | TrustCore         |
| `TrustCommitObservation`  | value  | Foreign TrustCommitObservation boundary; declaration ADR-0006:01, actual producer remains canonical owner | `src/application/trust.rs` `TrustCommitObservation`      | Store             |
| `InventoryWriteView`      | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/configuration.rs` `InventoryWriteView`  | Configuration     |
| `SettingsWriteView`       | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/monitoring.rs` `SettingsWriteView`      | Monitoring        |
| `AuditAppend`             | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/operations.rs` `AuditAppend`            | OperationEngine   |
| `AuditBatchData`          | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/operations.rs` `AuditBatchData`         | OperationEngine   |
| `PreparedTrustChangeView` | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/trust.rs` `PreparedTrustChangeView`     | TrustCore         |
| `LocalTrustEnvelopeView`  | value  | Imported immutable Store boundary; runtime issuance remains canonical owner                               | `src/application/operations.rs` `LocalTrustEnvelopeView` | OperationEngine   |

### Functions

### `launch(invocation: Invocation, paths: ApplicationPaths) -> Result<(), Failure>`

- **Intent.** Run or activate one responsive desktop owner with real safety processing before managed work.
- **Owner.** `src/application/runtime.rs` private GTK bootstrap.
- **Reads.** Invocation, paths, GtkApplication registration, actual runtime residue and Store.
- **Writes.** Runtime; delegates domain changes only to their semantic owners.
- **Preconditions.** Main thread; local required facilities accessible.
- **Postconditions.** Askpass only calls E1 helper mode. Secondary desktop only activates primary. Primary follows the bootstrap flow below, presents actual progress/failure, and exposes full managed services only after admission. Returns when lifecycle ends.
- **Invariants.** INV-1, INV-4, INV-5.
- **Errors.** Registration/storage/migration/cleanup/recovery failure blocks admission and presents specific affected resources; no orphan adoption or assumed empty audit.
- **Idempotency.** Duplicate launch cannot repeat primary startup, clear locks or open competing Store/master ownership.

### `Store::open(primary: &PrimaryOwnership, paths: &ApplicationPaths) -> Result<Store, Failure>`

- **Intent.** Open the actual compatible persisted state under sole ownership.
- **Owner.** `src/infrastructure/storage/mod.rs` Store.
- **Reads.** PrimaryOwnership, database and versioned migrations.
- **Writes.** Store and mechanical schema migration state.
- **Preconditions.** Current process is primary; database path is the intended application Store.
- **Postconditions.** Active foreign keys, UUID identities and versioned non-truncating migrations; real inventory, HostKey, settings and audit queries are available. Fresh database creates actual accepted tables, including operations/events.
- **Invariants.** INV-1.
- **Errors.** StorageError/migration incompatibility returns no operational Store, never substitutes in-memory empty state.
- **Idempotency.** Applied migrations are not re-applied; failed migration rolls back; no audit/key truncation.

### `Store::request(&self, request: StoreRequest) -> Result<StoreResult, Failure>`

- **Intent.** Execute one owner-authorized domain persistence transaction faithfully.
- **Owner.** `src/infrastructure/storage/mod.rs` Store.
- **Reads.** Request’s current inventory/key/audit/settings facts, AuditTail/payload ordinals and expected revisions/tails.
- **Writes.** Store; mechanically applies only the submitted sealed batch.
- **Preconditions.** Open primary Store; matching request/response variant; batch created by authorized semantic owner; FK and expected-state constraints hold.
- **Postconditions.** Read returns actual committed data. Inventory and settings writes accept only Configuration-created InventoryCommit and Monitoring-created SettingsCommit, never caller-built observation vectors. Store uses InventoryCommit::inventory_write_view and SettingsCommit::settings_write_view only after receiving the sealed requests. Inventory/settings/audit append uses all-or-nothing transaction. CommitLocalTrust compares expected keys/tail, applies E3 key changes, reads back and appends the E4 committed-change fact in one transaction. Audit UPDATE/DELETE are unavailable. Validate ordinal 1 for a first fact and contiguous following ordinals per Operation, exact event UUID/content and expected AuditTail inside the transaction; ordinal lives only in existing sanitized_payload. Fold by validated ordinal, never SQL rowid/time/UUID. Local trust returns its exact committed ordinal. Missing/duplicate/noncontiguous/conflicting positions roll back the entire request.
- **Invariants.** INV-1, INV-8.
- **Errors.** Validation/CAS/FK failure or missing, duplicate, noncontiguous or conflicting payload ordinals returns ConfigurationConflict where the submitted batch conflicts; malformed persisted facts, schema/read/write/migration failures return StorageError. Roll back the entire request with no silent truncation or secret/content persistence. Unknown delivery does not authorize blind mutation repeat; reread actual history/tail before another append.
- **Idempotency.** Reads repeat safely. Owner/event identities make matching append/local commit replay return retained result or conflict, never duplicate effects or consume a new ordinal. Failed/ambiguous append requires actual tail/history read, not an assumed rollback. Explicit Import semantics remain E2’s, not Store inference.

### `Store::close(&self) -> Result<(), Failure>`

- **Intent.** End physical persistence only after all semantic owners finish.
- **Owner.** `src/infrastructure/storage/mod.rs` Store.
- **Reads.** Pending Store requests.
- **Writes.** Store lifecycle.
- **Preconditions.** Runtime has stopped new admission and joined domain owners.
- **Postconditions.** Submitted transactions settle and connection closes; no pending successful transaction is falsely acknowledged.
- **Invariants.** INV-1.
- **Errors.** Close/transaction failure is explicit and does not certify shutdown success.
- **Idempotency.** Repeated close joins/returns completed close.

### `Runtime::begin(primary: PrimaryOwnership, store: Store, directory: RuntimeDirectory) -> RuntimePorts`

- **Intent.** Establish pre-admission lifecycle and resource arbitration capabilities.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** PrimaryOwnership, actual Store, validated RuntimeDirectory.
- **Writes.** Runtime.
- **Preconditions.** Current primary; directory owned/restricted to `0700`.
- **Postconditions.** Returns runtime/locks/startup capabilities; phase is pre-Ready; no ReadyOwnership or master admission exists.
- **Invariants.** INV-1.
- **Errors.** Infallible after prerequisites; real prerequisite failures occur before this constructor.
- **Idempotency.** Once per primary generation; does not inherit old in-memory leases.

### `Runtime::admit(&self, cleanup: E1::CleanupReport, recovery: E4::RecoveryReport) -> Result<ReadyOwnership, Failure>`

- **Intent.** Open managed admission only after actual current-generation safety results exist.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** CleanupReport, RecoveryReport.
- **Writes.** Runtime.
- **Preconditions.** Both reports match startup generation; cleanup has no unresolved socket/ownership failure; E4 recovery transactions actually committed.
- **Postconditions.** Ready is published; unresolved Unknown Operations remain visible; ReadyOwnership is minted for this generation only.
- **Invariants.** INV-1, INV-4.
- **Errors.** Invalid/incomplete report yields blocked ConfigurationConflict/StorageError with no Ready capability.
- **Idempotency.** Same successful admission returns same-generation authority; cannot admit from older reports.

### `Runtime::install_owners(&self, owners: LifecycleOwners) -> Result<(), Failure>`

- **Intent.** Install actual shutdown participants and Runtime’s own current-connection recovery feed.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** LifecycleOwners and a separately obtained ConnectionManager::observe subscription.
- **Writes.** Runtime participants/subscription and recovery scheduling state.
- **Preconditions.** Same current primary; cannot replace an installed different owner set.
- **Postconditions.** Worker calls its own connections.observe; OwnersSubscribed installs initial snapshot/sequence before success. Subsequent updates enter RuntimeConnectionUpdate. Runtime schedules one worker per actual Connected identity/generation and receives ReconciliationCompleted; E5 uses a separate receiver. On stream loss, re-observe and process a real replacement snapshot.
- **Invariants.** INV-1, INV-5, INV-7.
- **Errors.** Observe/owner conflict leaves installation incomplete and managed startup controls blocked; never silently lose recovery scheduling.
- **Idempotency.** Identical concurrent installs join; completed install reuses its feed and never consumes E5’s receiver.

### `Runtime::observe(&self) -> Result<Subscription<StartupSnapshot, StartupChanged>, Failure>`

- **Intent.** Explain actual recovery/admission/shutdown state to presentation.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** Runtime.
- **Writes.** None.
- **Preconditions.** Primary runtime exists.
- **Postconditions.** Returns real phase/blockers/unresolved IDs, not Server availability or fabricated recovery success.
- **Invariants.** INV-5.
- **Errors.** Closed owner yields explicit failure; prior blockers are retained in state.
- **Idempotency.** Read-only subscription; never retries mutations.

### `Runtime::shutdown(&self) -> Result<StartupSnapshot, Failure>`

- **Intent.** End every owned workflow without claiming that remote effects were undone.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** LifecycleOwners, ResourceLease, Store and current StartupPhase.
- **Writes.** Runtime lifecycle; delegates domain changes to their canonical owners.
- **Preconditions.** Current primary.
- **Postconditions.** Publish Stopping, close Acquire and reconnect scheduling, store waiters and launch E5/E4 quiescence workers; immediately resume mailbox. Release requests remain serviceable. Their successful completions trigger E1 quiescence, then Store close. Only owner-side ShutdownCompleted advances phases; actual completion yields Stopped.
- **Invariants.** INV-1, INV-2, INV-3, INV-4, INV-5, INV-6, INV-7.
- **Errors.** Failed/unjoined/persistence completion keeps shutdown incomplete with truthful ownership/effects; never publishes false Stopped.
- **Idempotency.** Concurrent callers join one shutdown; stale completion cannot advance another phase. Abrupt death uses next-start recovery, never mutation replay.

### `LockAuthority::arbitrate(&self, request: LockRequest) -> Result<LockResult, Failure>`

- **Intent.** Admit nonconflicting owned work immediately and release it only after joined completion.
- **Owner.** `src/application/runtime.rs` Runtime.
- **Reads.** ConnectionIdentity, ResourceScope, current leases, WorkJoined.
- **Writes.** ResourceLease and in-memory arbitration map.
- **Preconditions.** Acquire has current Ready admission and nonempty unique planner-produced scopes; Release matches lease/Operation/current owner and sealed joined-work proof; Release stays enabled during Stopping.
- **Postconditions.** Acquire atomically grants all keys or immediately rejects conflict with incumbent Operation details and NoChange; independent keys do not wait. Release removes only that lease’s keys after work is joined.
- **Invariants.** INV-2, INV-3.
- **Errors.** ConfigurationConflict identifies incumbent/resource and performs no mutation; invalid release changes no ownership. Rejected acquisition is never queued.
- **Idempotency.** Repeated matching release is harmless after validated completion. A new mutation after rejection/cancellation is a new explicit request with new preflight/confirmation.

### `RuntimeDirectory::inspect_runtime_directory(&self) -> RuntimeDirectoryView<'_>`

- **Intent.** Read the validated confined runtime path and owner UID.
- **Owner.** `src/application/runtime.rs` Runtime
- **Reads.** RuntimeDirectory
- **Writes.** None.
- **Preconditions.** A live owner-issued receiver exists.
- **Postconditions.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Invariants.** INV-1.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

### `StartupOwnership::inspect_startup_ownership(&self) -> OwnershipView<'_>`

- **Intent.** Bind actual preadmission safety work to the primary generation.
- **Owner.** `src/application/runtime.rs` Runtime
- **Reads.** StartupOwnership
- **Writes.** None.
- **Preconditions.** A live owner-issued receiver exists.
- **Postconditions.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Invariants.** INV-1.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

### `ReadyOwnership::inspect_ready_ownership(&self) -> OwnershipView<'_>`

- **Intent.** Read the generation whose startup safety checks completed.
- **Owner.** `src/application/runtime.rs` Runtime
- **Reads.** ReadyOwnership
- **Writes.** None.
- **Preconditions.** A live owner-issued receiver exists.
- **Postconditions.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Invariants.** INV-1, INV-4.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; same-generation Ready clones share the existing Arc directory; references cannot outlive receiver.

### `ResourceLease::inspect_lease(&self) -> ResourceLeaseView<'_>`

- **Intent.** Read the actual Operation, identity and scope lease binding.
- **Owner.** `src/application/runtime.rs` Runtime
- **Reads.** ResourceLease
- **Writes.** None.
- **Preconditions.** A live owner-issued receiver exists.
- **Postconditions.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Invariants.** INV-2, INV-3.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

### `InventoryCommit::inventory_write_view(&self) -> InventoryWriteView<'_>`

- **Intent:** Let Store read the validated inventory delta without allowing callers to mint a write request.
- **Owner:** `src/application/configuration.rs`.
- **Introducing ticket:** ADR-0006:01 implements only this infallible borrowed projection in the canonical Configuration module; it does not mint InventoryCommit or assume its later validation proof. E2:03 remains the sole validated runtime commit producer.
- **Reads:** InventoryCommit.
- **Writes:** None.
- **Preconditions:** Store received an owner-created InventoryCommit inside InventoryStoreRequest::Commit.
- **Postconditions:** Borrows exact expected revision, Servers and Auth Profiles; Store still requires the private wrapper, not a caller-created InventoryWriteView.
- **Invariants:** INV-3.
- **Errors:** Infallible; no I/O, mutation or authority allocation.
- **Idempotency:** Repeatable borrowed read; no duplicate commit is issued.

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

### `SettingsCommit::settings_write_view(&self) -> SettingsWriteView<'_>`

- **Intent.** Let Store persist validated settings without accepting arbitrary MonitoringSettings writes.
- **Owner.** `src/application/monitoring.rs`
- **Introducing ticket.** ADR-0006:01 implements this infallible projection in the canonical E5 module before Store::request; ADR-0005:01 alone creates actual validated SettingsCommit values through configure.
- **Reads.** SettingsCommit.
- **Writes.** None.
- **Preconditions.** Commit constructed only by Monitoring after configure validation.
- **Postconditions.** Borrows exact settings authorized for settings-only transaction; view does not mint commit and no Store entry accepts a view or arbitrary settings for SaveSettings.
- **Invariants.** INV-3.
- **Errors.** Infallible projection; transaction failures remain Store errors.
- **Idempotency.** Repeatable inspection of one immutable commit, no additional write or sample persistence.

### `AuditAppend::into_store_batch(self) -> AuditBatchData`

- **Intent.** Move the authorized immutable append batch into Store execution.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** AuditAppend
- **Writes.** None; moves existing data only.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Single consumption; cannot call twice on moved receiver.

- **Introducing ticket.** ADR-0006:01 implements this pure Store projection in the canonical owner module; no positive authority is issued.

### `LocalTrustEnvelope::local_trust_store_view(&self) -> LocalTrustEnvelopeView`

- **Intent.** Read the E4-authored atomic trust/audit admission.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** LocalTrustEnvelope
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0006:01 implements this pure Store projection in the canonical owner module; no positive authority is issued.

### Invariants

| ID    | Statement                                                                                                                             | Scope                                                                                                      | Enforced by                                                                    | Broken if                                                                   | Proof                                                                                  |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| INV-1 | Only the current GtkApplication primary opens operational Store and mints current-generation ownership capabilities.                  | Runtime, Store, PrimaryOwnership, StartupOwnership                                                         | Primary registration before initialization; private generation constructors    | Secondary or askpass opens operational state                                | First proof ADR-0006:01: `single_primary_real_store_and_preadmission`                  |
| INV-2 | Equal Connection Identity/resource-scope keys have at most one lease; rejected requests never queue.                                  | ResourceLease                                                                                              | Atomic multi-key in-memory acquisition; no deferred mutation queue             | Inventory UUID keys or partial acquisition bypass conflict                  | First proof ADR-0006:03: `concurrent_alias_conflict_and_independent_scopes`            |
| INV-3 | A lease is released only for matching Operation/generation after all owned conflicting work joins, regardless of UI or outcome label. | ResourceLease                                                                                              | E4-sealed WorkJoined, exact coverage/binding checks and consumption            | Cancel/exit/error/Unknown releases early or retains ended work forever      | First proof ADR-0006:03: `lease_requires_matching_joined_work`                         |
| INV-4 | Ready follows actual orphan non-adoption and durable interrupted Running/Verifying to Unknown recovery.                               | Runtime, ReadyOwnership                                                                                    | Same-generation real CleanupReport and RecoveryReport; fail-closed admission   | Missing table/storage error is empty or old socket is Connected             | First proof ADR-0006:02: `actual_orphans_and_durable_unknown_before_admit`             |
| INV-5 | GTK owns widgets and remains responsive with truthful ownership/recovery/shutdown states.                                             | Runtime                                                                                                    | Main-context domain delivery; workers cannot access widgets                    | Blocking work on main thread or fake empty/success                          | First proof ADR-0006:01: `runtime_gtk_preadmission_and_error_states`                   |
| INV-6 | Runtime services Release while shutdown awaits E4; lifecycle changes only on owner completion messages.                               | Runtime, ResourceLease                                                                                     | ShutdownCompletion workers and Release enabled during Stopping                 | Shutdown holds mailbox or state lock across E4 await                        | First proof ADR-0006:04: `shutdown_services_release_until_quiesced`                    |
| INV-7 | Runtime owns a separate real connection feed and schedules only identity-matching read-only recovery on actual usable generations.    | Runtime                                                                                                    | Independent observe, sequence/generation deduplication and E4 identity recheck | Consumes E5 receiver, loses reconnect or reconciles retargeted alias        | First proof ADR-0006:04: `runtime_separate_feed_reobserve_and_identity_reconciliation` |
| INV-8 | Store accepts only sealed canonical-owner writes; immutable projections expose exact data and never authorize a new write.            | Store, StoreRequest, InventoryCommit, SettingsCommit, AuditAppend, PreparedTrustChange, LocalTrustEnvelope | Private wrapper fields and five canonical infallible Store projections         | Public Server/settings vectors or read views authorize generic Store writes | First proof ADR-0006:01: `store_sealed_owner_requests_and_projection_privacy`          |

### Ordered flows

1. **Primary and preadmission (Usage 1/2; ticket 01).**
   1. External desktop invocation/application paths → private GTK primary-registration adapter. A duplicate activates the existing primary and cannot open Store or clear leases. Askpass is dispatched separately by final launch and never enters operational initialization.
   2. Primary validates confined runtime directory ownership and mode 0700; `RuntimeDirectory::inspect_runtime_directory` exposes the validated path. `Store::open` opens actual compatible SQLite with versioned migrations and active FKs, never an empty fallback.
   3. `Runtime::begin` consumes actual PrimaryOwnership/Store/directory → RuntimePorts with Runtime, LockAuthority and StartupOwnership. `StartupOwnership::inspect_startup_ownership` identifies this generation; no Ready capability exists. `Runtime::observe` drives actual preadmission loading/error presentation. Common declarations exist in their canonical six modules, but no future owner methods or capabilities are fabricated.
   4. Owner-authored external StoreRequest → `Store::request` executes actual reads or sealed transactions; InventoryCommit::inventory_write_view, SettingsCommit::settings_write_view, AuditAppend::into_store_batch, PreparedTrustChange::prepared_trust_store_view and LocalTrustEnvelope::local_trust_store_view expose their exact existing data inside this sealed boundary. InventoryCommit, SettingsCommit, AuditAppend and LocalTrustEnvelope retain their domain writers; immutable views cannot authorize writes.
2. **Actual recovery and admission (Usage 2; ticket 02).**
   1. Actual E2 Configuration::spawn and E3 TrustCore::spawn/TrustPorts::into_trust_ports consume Store; their actual handles → E4 OperationEngine::spawn without SSH.
   2. StartupOwnership → E1 cleanup_orphans, actual application sockets → CleanupReport::inspect_cleanup; each leftover receives exit/removal without adoption. Other user SSH resources are untouched. Failure blocks admission.
   3. Same StartupOwnership → E4 recover_interrupted reads real audit and durably appends Running/Verifying → Unknown while retaining evidence. RecoveryReport::inspect_recovery proves actual commits; missing table/failed read/failed commit is not empty.
   4. `Runtime::admit(cleanup,recovery)` verifies matching startup generation and completed actual safety work → ReadyOwnership; `ReadyOwnership::inspect_ready_ownership` binds later real owners. Unknown remains visible; Ready does not mean a remote mutation policy or Connect already exists.
3. **Resource arbitration (Usage 3; ticket 03).**
   1. External authorized owner supplies current ConnectionIdentity and planner-produced unique ResourceScope set to `LockAuthority::arbitrate(Acquire)`; atomic all-or-none current Ready check grants ResourceLease or immediately rejects conflict with incumbent details and NoChange. No hidden queue exists.
   2. `ResourceLease::inspect_lease` exposes exact generation/Operation/identity/scopes. Alias presentation cannot change the key; independent keys proceed without global serialization.
   3. Actual E4 cleanup owns its tracked consumer/task sets; after all work joins and conservative persistence settles it creates WorkJoined. `LockAuthority::arbitrate(Release)` consumes that proof and lease, checks exact binding, and removes only matching keys. Before E4 execution exists, ticket 03 exercises actual arbitration plus refusal of fabricated/mismatched proofs; it does not claim a real service mutation journey.
   4. Unknown is not no-effect proof and is not a reason to retain an ended workflow’s lease forever. Local ownership grants no remote exclusion, unmanaged-resource consent or Verifier proof.
4. **Full desktop/reconnect/shutdown (Usage 1/3; ticket 04).**
   1. `src/main.rs` calls the actual `launch` in `src/application/runtime.rs` on main thread: restricted askpass → E1 serve_askpass only; secondary desktop activates primary only. Primary invokes actual flows 1/2, with no new safety shortcut.
   2. Main-thread E1 install_transport_ui creates real authentication/VTE TransportUi; actual ReadyOwnership, Configuration and TrustGate → E1 ConnectionManager::spawn. E4 bind_transport receives actual ConsumerAccess. E3 Trust::assemble(core,configuration,Some(routes)) uses the SAME core previously inspectable with None.
   3. E5 Monitoring::spawn receives its own actual E1 Connection subscription and ConsumerAccess plus actual settings Store and E4 read-only summary capability. `Runtime::install_owners` starts a different observe worker; OwnersSubscribed establishes its initial snapshot/sequence before managed startup controls or connect_on_startup evaluation.
   4. Initial actual Connected snapshot and subsequent sequenced ConnectionChanged events → RuntimeConnectionUpdate. Each identity/generation schedules one owned E4 reconcile_connected worker, never an await inside owner handling. ReconciliationCompleted reports actual persisted outcomes/unavailable policies. Original identity must still match current target. Stream loss forces real re-observe/snapshot before current scheduling; duplicate events coalesce. No Connect or mutation replay is issued by recovery.
   5. External exit → `Runtime::shutdown` stores waiters, closes Acquire/reconnect scheduling, publishes Stopping, and launches E5/E4 quiescence workers. Mailbox continues servicing Release and completion messages while workers wait.
   6. MonitoringStopped and OperationsStopped (ordinary OperationQuiesced IDs after actual acknowledged E4 releases) → transport quiesce worker. TransportStopped after every descendant/master joins → `Store::close` worker. StoreClosed success → Stopped. Completion failure remains truthful incomplete shutdown; no live ownership is discarded.
   7. Abrupt termination instead preserves real append-only evidence; later primary repeats actual cleanup/recovery without adopting old masters or replaying mutations. Actual GTK journeys cover keyboard/loading/empty/error/stale/Unknown and responsiveness.

## Decision

- Select Sol’s per-owner actors, capability-based bootstrap and mechanical Store transactions.
- Reject per-identity worker processes because distributed ownership adds unnecessary failure boundaries.
- Reject one global reducer because it obscures semantic writers and serializes unrelated work.
- Add five pure canonical Store projections to the shared first slice without minting any future capability; accept explicit completion state for responsive shutdown.
- Share same-generation Ready directory by Arc; retain Terra’s read-only download and payload-ordinal grafts in E4.

## Open questions

What approved E2 environment vocabulary, E3 trust-use membership/local trust specialization and E4 per-type evidence policies will enable currently unavailable dependent mutations?
