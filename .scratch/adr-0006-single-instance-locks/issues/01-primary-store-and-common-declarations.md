# 01: Establish sole primary, compatible SQLite and preadmission runtime

**Status:** ready-for-agent
**Blocked by:** None (can start immediately)
**External blockers:** None
**Source:** US-001-AC1, US-001-AC2, US-011-AC1, US-011-AC3, US-011-AC4
**Contract:** `.scratch/adr-0006-single-instance-locks/spec.md`, section `# Design`; the five imported Store projections retain their canonical E2/E3/E4/E5 full contracts and invariant namespaces. Contract wins; stop and report a conflict.

## Goal

The real sole primary opens compatible SQLite and presents truthful preadmission status. Introduce every shared record/enum/handle declaration in its canonical owner module and implement the five pure Store projections, without minting future domain capabilities or pretending Ready exists.

## Types

```rust
// ADR-0001 canonical declaration cohort; no positive runtime issuance here
// src/domain/connection.rs
pub enum ConnectionState {
    NotConnected, Connecting, Connected, AuthFailed, HostKeyMismatch,
    HostKeyUntrusted, Timeout, Unreachable, Unknown,
}

// src/domain/connection.rs
pub enum CapabilityKind {
    OperatingSystem, Systemd, BusctlJson, Sftp, PosixRename, Sha256sum,
}

// src/domain/connection.rs
pub struct Capability {
    pub kind: CapabilityKind,
    pub availability: E6::Availability,
    pub observations: Vec<(String, String)>,
    pub observed_at: Instant,
}

// src/domain/connection.rs
pub struct Connection {
    pub identity: E2::ConnectionIdentity,
    pub server_ids: Vec<Uuid>,
    pub generation: Option<u64>,
    pub state: ConnectionState,
    pub capabilities: Vec<Capability>,
    pub observed_at: Instant,
    pub failure: Option<E6::Failure>,
}

// src/domain/connection.rs
pub enum TerminalState {
    Created, Connecting, Active, Disconnected, Exited, Error,
}

// src/domain/connection.rs
pub struct Terminal {
    pub id: Uuid,
    pub server_id: Uuid,
    pub identity: E2::ConnectionIdentity,
    pub master_generation: Option<u64>,
    pub state: TerminalState,
    pub exit_code: Option<i32>,
    pub failure: Option<E6::Failure>,
}

// src/domain/connection.rs
pub enum TerminalControl {
    Open { server_id: Uuid },
    Reconnect { terminal_id: Uuid },
    Close { terminal_id: Uuid },
}

// src/domain/connection.rs
pub enum ConnectionControl {
    Connect { server_id: Uuid },
    Disconnect { server_id: Uuid },
    Terminal(TerminalControl),
}

// src/domain/connection.rs
pub enum ConnectionResult {
    Connection(Connection),
    Terminal(Terminal),
    Reconnected { previous: Terminal, replacement: Terminal },
    TerminalClosed { terminal_id: Uuid },
}

// src/domain/connection.rs
pub struct ConnectionView {
    pub connections: Vec<Connection>,
    pub terminals: Vec<Terminal>,
}

// src/domain/connection.rs
pub struct ConnectionChanged {
    pub sequence: u64,
    pub connection: Option<Connection>,
    pub terminal: Option<Terminal>,
    pub closed_terminal_id: Option<Uuid>,
}

// src/application/connection_manager.rs
pub struct ConnectionManager { tx: mpsc::Sender<ConnectionMessage> }

// src/application/connection_manager.rs
pub struct ConsumerAccess { tx: mpsc::Sender<ConnectionMessage> }

// src/application/connection_manager.rs
pub struct RouteAccess { tx: mpsc::Sender<ConnectionMessage> }

// src/application/connection_manager.rs
pub struct ConnectionPorts {
    pub manager: ConnectionManager,
    pub consumers: ConsumerAccess,
    pub routes: RouteAccess,
}

// src/application/connection_manager.rs
pub(crate) struct MasterGeneration {
    identity: E2::ConnectionIdentity,
    number: u64,
}

// src/application/connection_manager.rs
pub(crate) struct StartReservation {
    master: MasterGeneration,
    start_id: Uuid,
    control_path: PathBuf,
}

// src/application/connection_manager.rs
#[derive(Clone, Copy)]
pub(crate) enum DispatchKnowledge { NotDispatched, Dispatched }

// src/application/connection_manager.rs
pub(crate) enum ChannelEndReason {
    NormalExit { code: i32 },
    LocalStop,
    MasterLost,
    SpawnFailed(E6::Failure),
    TransportFailed(E6::Failure),
}

// src/application/connection_manager.rs
pub(crate) struct ChannelEnd {
    consumer_id: Uuid,
    master: MasterGeneration,
    dispatch: DispatchKnowledge,
    reason: ChannelEndReason,
}

// src/application/connection_manager.rs
pub(crate) enum ConsumerRequest {
    Services(E4::ServiceRead),
    OpenText(E4::TextOpenRequest),
    Execute(E4::RemoteDispatch),
    Transfer(E4::TransferDispatch),
    Download(E4::DownloadDispatch),
    Reconcile(E4::ReconciliationRead),
    Collect(E5::CollectorSpec),
    KeyScan(E3::ScanRequest),
}

// src/application/connection_manager.rs
pub(crate) enum ConsumerEvent {
    ResourcesObserved(E4::ResourceObservation),
    ExecutionObserved(E4::ExecutionObservation),
    TransferProgressed(E4::TransferProgress),
    CollectionObserved(E5::RawObservation),
    KeysObserved(E3::KeyObservation),
    Ended(ChannelEnd),
}

// src/application/connection_manager.rs
pub(crate) struct Consumer {
    id: Uuid,
    master: MasterGeneration,
    events: mpsc::Receiver<ConsumerEvent>,
}

// src/application/connection_manager.rs
pub(crate) enum ConnectionMessage {
    Control {
        request: ConnectionControl,
        reply: E6::Reply<ConnectionResult>,
    },
    Observe { reply: E6::Reply<E6::Subscription<ConnectionView, ConnectionChanged>> },
    OpenConsumer {
        target: ConsumerTarget,
        request: ConsumerRequest,
        reply: E6::Reply<Consumer>,
    },
    StopConsumer {
        consumer_id: Uuid,
        reply: E6::Reply<ChannelEnd>,
    },
    PrepareDiscovery {
        resolved: E2::ResolvedConnection,
        reply: E6::Reply<E3::DiscoveryPath>,
    },
    Quiesce { reply: E6::Reply<Vec<ChannelEnd>> },
    ConnectCompleted(ConnectCompletion),
    TerminalLifecycle(TerminalLifecycleReport),
}

// src/ui/transport.rs
pub(crate) struct AuthenticationDialog {
    pub(crate) attempt_id: Uuid,
    pub(crate) challenge_id: Uuid,
    pub(crate) server_id: Uuid,
    pub(crate) identity: E2::ConnectionIdentity,
    pub(crate) prompt: String,
    pub(crate) answer: oneshot::Sender<AuthenticationAnswer>,
}

// src/ui/transport.rs
pub(crate) enum AuthenticationAnswer {
    Accepted(zeroize::Zeroizing<Vec<u8>>),
    Cancelled,
}

// src/infrastructure/ssh/mod.rs
pub(crate) struct AskpassRequest {
    pub(crate) owner_socket: PathBuf,
    pub(crate) attempt_id: Uuid,
    pub(crate) challenge_id: Uuid,
    pub(crate) nonce: [u8; 32],
    pub(crate) prompt: String,
}

// src/ui/transport.rs
pub(crate) struct TerminalBinding {
    pub(crate) terminal_id: Uuid,
    pub(crate) shell_generation: Uuid,
    pub(crate) master: MasterGeneration,
}

// src/ui/transport.rs
pub(crate) enum TerminalExit {
    ExitCode(i32),
    Signal(i32),
    Unknown,
}

// src/ui/transport.rs
pub(crate) enum TerminalUiEvent {
    ChildExited(TerminalExit),
    PtyReleased,
    SpawnFailed(E6::Failure),
}

// src/application/connection_manager.rs
pub(crate) struct TerminalLifecycleReport {
    binding: TerminalBinding,
    observed_at: Instant,
    event: TerminalUiEvent,
}

// src/application/connection_manager.rs
pub(crate) struct TerminalLifecycleSink {
    binding: TerminalBinding,
    tx: mpsc::UnboundedSender<ConnectionMessage>,
}

// src/ui/transport.rs
pub(crate) struct PrivateTerminalLaunch {
    pub(crate) binding: TerminalBinding,
    pub(crate) executable: PathBuf,
    pub(crate) arguments: Vec<OsString>,
    pub(crate) environment: Vec<(OsString, OsString)>,
}

// src/ui/transport.rs
pub(crate) struct VteSpawnReport {
    pub(crate) binding: TerminalBinding,
    pub(crate) process_id: u32,
    pub(crate) process_group: u32,
}

// src/ui/transport.rs
pub(crate) enum TerminalUiCommand {
    Spawn {
        launch: PrivateTerminalLaunch,
        events: TerminalLifecycleSink,
        reply: E6::Reply<VteSpawnReport>,
    },
    Close {
        terminal_id: Uuid,
        reply: E6::Reply<()>,
    },
}

// src/ui/transport.rs
pub(crate) struct TransportUi {
    authentication: mpsc::Sender<AuthenticationDialog>,
    terminals: mpsc::Sender<TerminalUiCommand>,
}

// src/infrastructure/ssh/mod.rs
pub(crate) struct CleanupReport {
    startup_generation: Uuid,
    visited_sockets: Vec<PathBuf>,
    removed_sockets: Vec<PathBuf>,
    unresolved: Vec<E6::Failure>,
}

// src/application/connection_manager.rs
pub(crate) enum ConsumerTarget {
    Server { server_id: Uuid },
    Generation(MasterGeneration),
}

// src/application/connection_manager.rs
pub(crate) struct MasterGenerationView<'a> {
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) number: u64,
}

// src/application/connection_manager.rs
pub(crate) struct StartReservationView<'a> {
    pub(crate) master: &'a MasterGeneration,
    pub(crate) start_id: Uuid,
    pub(crate) control_path: &'a Path,
}

// src/application/connection_manager.rs
pub(crate) struct ConsumerParts {
    pub(crate) id: Uuid,
    pub(crate) master: MasterGeneration,
    pub(crate) events: mpsc::Receiver<ConsumerEvent>,
}

// src/application/connection_manager.rs
pub(crate) struct ChannelEndView<'a> {
    pub(crate) consumer_id: Uuid,
    pub(crate) master: &'a MasterGeneration,
    pub(crate) dispatch: DispatchKnowledge,
    pub(crate) reason: &'a ChannelEndReason,
}

// src/infrastructure/ssh/mod.rs
pub(crate) struct CleanupReportView<'a> {
    pub(crate) startup_generation: Uuid,
    pub(crate) visited_sockets: &'a [PathBuf],
    pub(crate) removed_sockets: &'a [PathBuf],
    pub(crate) unresolved: &'a [E6::Failure],
}

// src/application/connection_manager.rs
pub(crate) struct TrustWork {
    attempt_id: Uuid,
    resolved: E2::ResolvedConnection,
    reservation: StartReservation,
    prefixes: Vec<MasterGeneration>,
    discovery: E3::DiscoveryPath,
}

// src/application/connection_manager.rs
pub(crate) struct CompilableStart {
    attempt_id: Uuid,
    resolved: E2::ResolvedConnection,
    reservation: StartReservation,
    prefixes: Vec<MasterGeneration>,
    trust: E3::TrustStart,
}

// src/application/connection_manager.rs
pub(crate) struct MasterSpawnObservation {
    master: MasterGeneration,
    start_id: Uuid,
    process: Option<tokio::process::Child>,
    control_check: Result<Instant, E6::Failure>,
}

// src/application/connection_manager.rs
pub(crate) enum ConnectCompletion {
    Resolved {
        request_id: Uuid,
        result: Result<E2::ResolvedConnection, E6::Failure>,
    },
    RoutePrepared {
        attempt_id: Uuid,
        result: Result<TrustWork, E6::Failure>,
    },
    TrustPrepared {
        attempt_id: Uuid,
        result: Result<CompilableStart, E6::Failure>,
    },
    ConfigurationCompiled {
        attempt_id: Uuid,
        result: Result<E2::CompiledConfiguration, E6::Failure>,
    },
    MasterSpawned {
        attempt_id: Uuid,
        observation: MasterSpawnObservation,
    },
}

// ADR-0002 canonical declaration cohort; no positive runtime issuance here
// src/domain/inventory.rs
pub enum AuthMethod {
    Agent { socket_reference: Option<PathBuf> },
    Key {
        identity_files: Vec<PathBuf>,
        identities_only: bool,
        agent_reference: Option<PathBuf>,
    },
}

// src/domain/inventory.rs
pub enum NegotiationClass { KeyAlgorithm, Cipher }

// src/domain/inventory.rs
pub struct NegotiationChoice {
    directive: String,
    category: NegotiationClass,
    values: Vec<String>,
}

// src/domain/inventory.rs
pub struct AuthProfile {
    pub id: Uuid,
    pub name: String,
    pub method: AuthMethod,
    pub negotiation: Vec<NegotiationChoice>,
    pub created_at: SystemTime,
}

// src/domain/inventory.rs
pub struct AuthProfileDraft {
    pub id: Uuid,
    pub name: String,
    pub method: AuthMethod,
    pub negotiation_directives: Vec<(String, String)>,
}

// src/domain/inventory.rs
pub enum ClosedEnvironment { Unspecified, Development, Test, Staging, Production }
// src/domain/inventory.rs
pub struct Server {
    pub id: Uuid,
    pub name: String,
    pub hostname: String,
    pub ssh_port: u16,
    pub ssh_username: String,
    pub auth_profile_id: Uuid,
    pub jump_server_id: Option<Uuid>,
    pub group_id: Option<Uuid>,
    pub environment_discriminator: String,
    pub tags: Vec<String>,
    pub connect_on_startup: bool,
    pub created_at: SystemTime,
    pub updated_at: SystemTime,
}

// src/domain/inventory.rs
pub struct ServerDraft {
    pub id: Uuid,
    pub name: String,
    pub hostname: String,
    pub ssh_port: u32,
    pub ssh_username: String,
    pub auth_profile_id: Uuid,
    pub jump_server_id: Option<Uuid>,
    pub group_id: Option<Uuid>,
    pub environment_discriminator: String,
    pub tags: Vec<String>,
    pub connect_on_startup: bool,
}

// src/domain/connection_identity.rs
pub struct AuthenticationContext {
    agent_reference: Option<PathBuf>,
    identity_files: Vec<PathBuf>,
    identities_only: bool,
    negotiation: Vec<NegotiationChoice>,
}

// src/domain/connection_identity.rs
pub struct ConnectionEndpoint {
    hostname: String,
    port: u16,
    ssh_username: String,
    authentication: AuthenticationContext,
}

// src/domain/connection_identity.rs
pub struct ConnectionIdentity {
    target: ConnectionEndpoint,
    jump_route: Vec<ConnectionEndpoint>,
}

// src/domain/inventory.rs
pub struct InventorySnapshot {
    pub servers: Vec<Server>,
    pub auth_profiles: Vec<AuthProfile>,
    pub revision: [u8; 32],
}

// src/domain/inventory.rs
pub struct ImportExclusion {
    pub source: PathBuf,
    pub line: usize,
    pub entry: Option<String>,
    pub directive: String,
    pub reason: String,
}

// src/domain/inventory.rs
pub struct ImportReport {
    pub saved_servers: Vec<Server>,
    pub saved_auth_profiles: Vec<AuthProfile>,
    pub imported_settings: Vec<(Uuid, Vec<String>)>,
    pub exclusions: Vec<ImportExclusion>,
}

// src/domain/inventory.rs
pub enum ConfigurationChange {
    SaveServer(ServerDraft),
    SaveAuthProfile(AuthProfileDraft),
    Import { source: PathBuf },
}

// src/domain/inventory.rs
pub enum ConfigurationResult {
    ServerSaved(Server),
    AuthProfileSaved(AuthProfile),
    Imported(ImportReport),
}

// src/application/configuration.rs
pub struct Configuration { tx: mpsc::Sender<ConfigurationMessage> }

// src/domain/connection_identity.rs
pub(crate) struct ResolvedConnection {
    server_id: Uuid,
    identity: ConnectionIdentity,
    route_server_ids: Vec<Uuid>,
    inventory_revision: [u8; 32],
}

// src/application/configuration.rs
pub(crate) struct CompiledConfiguration {
    reservation: E1::StartReservation,
    resolved: ResolvedConnection,
    configuration_path: PathBuf,
    master_alias: String,
    attachment_alias: String,
    prefix_aliases: Vec<(E1::MasterGeneration, String)>,
    trust: E3::TrustStart,
}

// src/application/configuration.rs
pub(crate) struct InventoryCommit {
    expected_revision: [u8; 32],
    servers: Vec<Server>,
    auth_profiles: Vec<AuthProfile>,
}

// src/application/configuration.rs
pub(crate) struct InventoryWriteView<'a> {
    pub(crate) expected_revision: &'a [u8; 32],
    pub(crate) servers: &'a [Server],
    pub(crate) auth_profiles: &'a [AuthProfile],
}

// src/application/configuration.rs
pub(crate) enum InventoryStoreRequest {
    Read,
    Commit(InventoryCommit),
}

// src/application/configuration.rs
pub(crate) enum InventoryStoreResult {
    Snapshot(InventorySnapshot),
    Committed(InventorySnapshot),
}

// src/application/configuration.rs
pub(crate) enum ConfigurationMessage {
    Inventory { reply: E6::Reply<InventorySnapshot> },
    Change {
        request: ConfigurationChange,
        reply: E6::Reply<ConfigurationResult>,
    },
    Resolve {
        server_id: Uuid,
        reply: E6::Reply<ResolvedConnection>,
    },
    Compile {
        resolved: ResolvedConnection,
        trust: E3::TrustStart,
        reservation: E1::StartReservation,
        prefixes: Vec<E1::MasterGeneration>,
        reply: E6::Reply<CompiledConfiguration>,
    },
}

// src/domain/connection_identity.rs
pub(crate) struct ConnectionEndpointView<'a> {
    pub(crate) hostname: &'a str,
    pub(crate) port: u16,
    pub(crate) ssh_username: &'a str,
    pub(crate) authentication: &'a AuthenticationContext,
}

// src/domain/connection_identity.rs
pub(crate) struct ConnectionIdentityView<'a> {
    pub(crate) target: &'a ConnectionEndpoint,
    pub(crate) jump_route: &'a [ConnectionEndpoint],
}

// src/domain/connection_identity.rs
pub(crate) struct ResolvedConnectionView<'a> {
    pub(crate) server_id: Uuid,
    pub(crate) identity: &'a ConnectionIdentity,
    pub(crate) route_server_ids: &'a [Uuid],
    pub(crate) inventory_revision: &'a [u8; 32],
}

// src/application/configuration.rs
pub(crate) struct CompiledLaunchView<'a> {
    pub(crate) reservation: &'a E1::StartReservation,
    pub(crate) resolved: &'a ResolvedConnection,
    pub(crate) configuration_path: &'a Path,
    pub(crate) master_alias: &'a str,
    pub(crate) attachment_alias: &'a str,
    pub(crate) prefix_aliases: &'a [(E1::MasterGeneration, String)],
    pub(crate) trust: &'a E3::TrustStart,
}

// ADR-0003 canonical declaration cohort; no positive runtime issuance here
// src/domain/trust.rs
pub enum TrustState { Verified, Adopted, Unverified, Mismatch, None }

// src/domain/trust.rs
pub enum TrustUse { Direct, Jump }

// src/domain/trust.rs
pub enum AcceptancePath { Adoption, Verification, Tofu }

// src/domain/trust.rs
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

// src/domain/trust.rs
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

// src/application/trust.rs
pub(crate) struct ScanRequest {
    pub(crate) hostname: String,
    pub(crate) port: u16,
}

// src/application/trust.rs
pub(crate) struct KeyObservation {
    pub(crate) hostname: String,
    pub(crate) port: u16,
    pub(crate) keys: Vec<(String, String, String)>,
    pub(crate) route_server_ids: Vec<Uuid>,
    pub(crate) collected_at: Instant,
}

// src/application/trust.rs
pub(crate) enum DiscoveryPath {
    Local,
    Jump {
        generation: E1::MasterGeneration,
        consumers: E1::ConsumerAccess,
    },
}

// src/domain/trust.rs
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

// src/domain/trust.rs
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

// src/domain/trust.rs
pub struct TrustReview {
    pub server_id: Uuid,
    pub stored_keys: Vec<HostKey>,
    pub state: TrustState,
    pub candidates: Vec<TrustEvidence>,
    pub eligibility: Vec<(TrustUse, E6::Availability)>,
    pub adopted_evidence_changed: bool,
    pub observations: Vec<E6::Failure>,
}

// src/domain/trust.rs
pub enum TrustChange {
    Accept { evidence: TrustEvidence, path: AcceptancePath },
    Replace {
        expected: HostKey,
        evidence: TrustEvidence,
        path: AcceptancePath,
    },
    Delete { expected: HostKey },
}

// src/application/trust.rs
pub struct Trust {
    core: TrustCore,
    configuration: E2::Configuration,
    routes: Option<E1::RouteAccess>,
}

// src/application/trust.rs
pub(crate) struct TrustCore { tx: mpsc::Sender<TrustMessage> }

// src/application/trust.rs
pub(crate) struct TrustGate { tx: mpsc::Sender<TrustMessage> }

// src/application/trust.rs
pub(crate) struct TrustExecutor { tx: mpsc::Sender<TrustMessage> }

// src/application/trust.rs
pub(crate) struct TrustPorts {
    core: TrustCore,
    gate: TrustGate,
    executor: TrustExecutor,
}

// src/application/trust.rs
pub(crate) struct TrustPermission {
    start_id: Uuid,
    identity: E2::ConnectionIdentity,
    endpoint_uses: Vec<((String, u16), TrustUse, Uuid, TrustState)>,
    store_digest: [u8; 32],
}

// src/application/trust.rs
pub(crate) struct TrustSnapshot {
    start_id: Uuid,
    known_hosts_path: PathBuf,
    store_digest: [u8; 32],
    generated_at: Instant,
}

// src/application/trust.rs
pub(crate) struct TrustStart {
    permission: TrustPermission,
    snapshot: TrustSnapshot,
}

// src/application/trust.rs
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

// src/application/trust.rs
pub(crate) struct TrustCommitObservation {
    pub(crate) operation_id: Uuid,
    pub(crate) before: Vec<HostKey>,
    pub(crate) after: Vec<HostKey>,
    pub(crate) committed_event_id: Uuid,
    pub(crate) committed_event_ordinal: u64,
    pub(crate) read_back_digest: [u8; 32],
}

// src/application/trust.rs
pub(crate) enum TrustStoreRequest {
    Read { endpoints: Vec<(String, u16)> },
}

// src/application/trust.rs
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

// src/application/trust.rs
pub(crate) enum TrustCompletion {
    Reviewed { request_id: Uuid, result: Result<TrustReview, E6::Failure> },
    Started { request_id: Uuid, result: Result<PreparedStart, E6::Failure> },
    ChangePrepared { request_id: Uuid, result: Result<PreparedTrustChange, E6::Failure> },
    Committed { request_id: Uuid, result: Result<TrustCommitObservation, E6::Failure> },
}

// src/application/trust.rs
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

// src/application/trust.rs
pub(crate) struct PreparedStart {
    resolved: E2::ResolvedConnection,
    reservation: E1::StartReservation,
    trust: TrustStart,
}

// src/application/trust.rs
pub(crate) struct TrustStartView<'a> {
    pub(crate) start_id: Uuid,
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) endpoint_uses: &'a [((String, u16), TrustUse, Uuid, TrustState)],
    pub(crate) store_digest: &'a [u8; 32],
    pub(crate) known_hosts_path: &'a Path,
    pub(crate) generated_at: Instant,
}

// src/application/trust.rs
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

// src/application/trust.rs
pub(crate) struct TrustCommitFacts {
    pub(crate) operation_id: Uuid,
    pub(crate) before: Vec<HostKey>,
    pub(crate) after: Vec<HostKey>,
    pub(crate) committed_event_id: Uuid,
    pub(crate) committed_event_ordinal: u64,
    pub(crate) read_back_digest: [u8; 32],
}

// src/domain/trust.rs
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

// ADR-0004 canonical declaration cohort; no positive runtime issuance here
// src/domain/operation.rs
pub enum ServiceVerb { Start, Stop, Restart, Reload, Enable, Disable }

// src/domain/operation.rs
pub enum OperationClass {
    ReadOnly, IdempotentMutation, NonIdempotentMutation, DestructiveMutation,
}

// src/domain/operation.rs
pub enum OperationStatus {
    Requested, Validating, Planning, AwaitingConfirmation,
    Running, Verifying, Succeeded, Failed, Unknown, Cancelled,
}

// src/domain/operation.rs
pub enum Confirmation {
    RequestedChange, DestructiveImpact, Overwrite, UnmanagedResource,
    NonAtomicOverwrite, TofuWarning, TrustReplacement, TrustDeletion,
}

// src/domain/operation.rs
pub struct ResourceScope {
    pub(crate) resource_type: String,
    pub(crate) resource_id: String,
}

// src/domain/resource_observation.rs
pub struct FileVersion {
    pub requested_path: String,
    pub resolved_path: String,
    pub size: u64,
    pub mtime_seconds: i64,
    pub mtime_nanoseconds: u32,
    pub sha256: [u8; 32],
    pub observed_at: Instant,
}

// src/domain/resource_observation.rs
pub struct TextDocument {
    pub server_id: Uuid,
    pub path: String,
    pub utf8: Arc<str>,
    pub version: FileVersion,
}

// src/domain/resource_observation.rs
pub struct TextOpenRequest {
    pub server_id: Uuid,
    pub path: String,
}

// src/domain/operation.rs
pub enum FileChange {
    CreateFile { path: String },
    CreateDirectory { path: String },
    WriteText { opened: TextDocument, replacement: Arc<str> },
    Delete { path: String },
    Rename { source: String, destination: String },
    Chmod { path: String, mode: u32 },
    Upload { local_source: PathBuf, destination: String },
}

// src/domain/operation.rs
pub enum AdministrativeChange {
    Service { verb: ServiceVerb, unit: String },
    File(FileChange),
    Download(DownloadRequest),
    Trust(E3::TrustChange),
}

// src/domain/operation.rs
pub struct OperationRequest {
    pub server_id: Uuid,
    pub change: AdministrativeChange,
}

// src/domain/resource_observation.rs
pub struct UnitInfo {
    pub name: String,
    pub description: String,
    pub active_state: String,
    pub sub_state: String,
    pub result: String,
    pub main_pid: u32,
    pub restart_count: Option<u64>,
    pub memory_bytes: Option<u64>,
    pub dependencies: Vec<String>,
    pub actions: Vec<(ServiceVerb, E6::Availability)>,
}

// src/domain/resource_observation.rs
pub struct JournalRecord {
    pub observed_at: Option<SystemTime>,
    pub priority: Option<u8>,
    pub message: String,
}

// src/domain/resource_observation.rs
pub struct ServiceSummary {
    pub unit_states: Vec<(String, String, String)>,
    pub observed_at: Instant,
}

// src/domain/resource_observation.rs
pub enum ServiceRead {
    Units { server_id: Uuid },
    Unit { server_id: Uuid, unit: String },
    Dependencies { server_id: Uuid, unit: String },
    Journal { server_id: Uuid, unit: String, limit: u32 },
    Summary { server_id: Uuid },
}

// src/domain/resource_observation.rs
pub enum ResourceQuery {
    Services(ServiceRead),
    OpenText(TextOpenRequest),
}

// src/domain/resource_observation.rs
pub enum ResourceObservation {
    Units { server_id: Uuid, units: Vec<UnitInfo>, observed_at: Instant },
    Unit { server_id: Uuid, unit: UnitInfo, observed_at: Instant },
    Dependencies {
        server_id: Uuid, unit: String, units: Vec<String>, observed_at: Instant,
    },
    Journal {
        server_id: Uuid, unit: String,
        records: Vec<JournalRecord>, observed_at: Instant,
    },
    Summary { server_id: Uuid, summary: ServiceSummary },
    Text(TextDocument),
}

// src/domain/resource_observation.rs
pub struct ServiceEvidence {
    pub remote_boot_id: Uuid,
    pub active_enter_monotonic_usec: Option<u64>,
    pub exec_main_start_monotonic_usec: Option<u64>,
    pub unit: UnitInfo,
    pub observed_at: Instant,
}

// src/domain/operation.rs
pub struct VerificationEvidence {
    pub policy_reference: Option<String>,
    pub facts: Vec<(String, String)>,
    pub service_observations: Vec<ServiceEvidence>,
    pub file_observations: Vec<FileVersion>,
    pub observed_at: SystemTime,
}

// src/domain/operation.rs
pub enum TransferState {
    Queued, Transferring, Verifying, Completed, Failed, Cancelled, Unknown,
}

// src/domain/operation.rs
pub struct Transfer {
    pub id: Uuid,
    pub operation_id: Uuid,
    pub server_id: Uuid,
    pub source_path: String,
    pub destination_path: String,
    pub expected_bytes: Option<u64>,
    pub transferred_bytes: u64,
    pub state: TransferState,
    pub failure: Option<E6::Failure>,
}

// src/domain/operation.rs
pub struct TransferProgress {
    pub transfer_id: Uuid,
    pub bytes_transferred: u64,
    pub expected_bytes: Option<u64>,
}

// src/domain/operation.rs
pub struct OperationIntent {
    pub id: Uuid,
    pub server_id: Uuid,
    pub resource_type: String,
    pub resource_id: String,
    pub action: String,
    pub sanitized_parameters: Vec<(String, String)>,
    pub requested_at: SystemTime,
    pub correlation_id: Uuid,
}

// src/domain/operation.rs
pub struct Operation {
    pub intent: OperationIntent,
    pub connection_identity: Option<E2::ConnectionIdentity>,
    pub classification: Option<OperationClass>,
    pub status: OperationStatus,
    pub started_at: Option<SystemTime>,
    pub finished_at: Option<SystemTime>,
    pub verification: Option<VerificationEvidence>,
    pub failure: Option<E6::Failure>,
    pub reconciliation_needed: bool,
    pub cancellation_requested: bool,
    pub transfer: Option<Transfer>,
}

// src/domain/operation.rs
pub struct OperationPlan {
    pub operation: Operation,
    pub revision: Uuid,
    pub scopes: Vec<ResourceScope>,
    pub impact: String,
    pub required_confirmations: Vec<Confirmation>,
    pub availability: E6::Availability,
}

// src/domain/operation.rs
pub enum PlanResult {
    Confirmable(OperationPlan),
    Blocked {
        operation: Operation,
        required_decisions: Vec<String>,
        failure: E6::Failure,
    },
}

// src/domain/operation.rs
pub enum OperationDecision {
    Confirm { revision: Uuid, acknowledged: Vec<Confirmation> },
    Reject { revision: Uuid },
}

// src/domain/operation.rs
pub enum OperationFact {
    Requested,
    StageEntered {
        status: OperationStatus,
        classification: Option<OperationClass>,
        started_at: Option<SystemTime>,
        finished_at: Option<SystemTime>,
    },
    PlanRecorded { plan: OperationPlan },
    ConfirmationRecorded { revision: Uuid, acknowledged: Vec<Confirmation> },
    CancellationRequested,
    TransferChanged { transfer: Transfer },
    EvidenceRecorded { evidence: VerificationEvidence },
    LocalTrustCommitted { evidence_digest: [u8; 32], postcondition_digest: [u8; 32] },
    OutcomeEstablished {
        status: OperationStatus,
        evidence: Option<VerificationEvidence>,
        failure: Option<E6::Failure>,
        reconciliation_needed: bool,
    },
    InterruptionRecorded { previous_status: OperationStatus },
    ReconciliationRecorded {
        status: OperationStatus,
        evidence: Option<VerificationEvidence>,
        failure: Option<E6::Failure>,
    },
}

// src/domain/operation.rs
pub struct OperationRecorded {
    pub id: Uuid,
    pub operation_id: Uuid,
    pub ordinal: u64,
    pub fact: OperationFact,
    pub created_at: SystemTime,
}

// src/domain/operation.rs
pub enum OperationQuery {
    Recent { limit: u32 },
    Detail { operation_id: Uuid },
    NeedingReconciliation,
    Interrupted,
}

// src/domain/operation.rs
pub struct OperationHistory {
    pub operations: Vec<Operation>,
    pub events: Vec<OperationRecorded>,
}

// src/domain/operation.rs
pub struct OperationChanged {
    pub sequence: u64,
    pub operation: Operation,
    pub event: OperationRecorded,
}

// src/application/operations.rs
pub struct OperationEngine { tx: mpsc::Sender<OperationMessage> }

// src/application/operations.rs
pub(crate) enum EnabledMutation {
    ServiceStart, ServiceStop, ServiceRestart, ServiceReload, ServiceEnable, ServiceDisable,
}

// src/application/operations.rs
pub(crate) enum LocalTrustApproval { Accept, Replace, Delete }

// src/application/operations.rs
pub(crate) struct ExecutionPermit {
    expected_identity: E2::ConnectionIdentity,
    operation_id: Uuid,
    plan_revision: Uuid,
    write_ahead: AuditTail,
    lease_id: Uuid,
    enabled: EnabledMutation,
}

// src/application/operations.rs
pub(crate) struct LocalTrustEnvelope {
    operation_id: Uuid,
    plan_revision: Uuid,
    expected_audit_tail: AuditTail,
    write_ahead: AuditTail,
    lease_id: Uuid,
    committed_event_id: Uuid,
    committed_event_ordinal: u64,
    committed_at: SystemTime,
    approval: LocalTrustApproval,
}

// src/application/operations.rs
pub(crate) struct RemoteDispatch {
    request: OperationRequest,
    preflight: VerificationEvidence,
    permit: ExecutionPermit,
}

// src/application/operations.rs
pub(crate) struct TransferDispatch {
    transfer_id: Uuid,
    request: OperationRequest,
    expected_source_size: u64,
    expected_source_sha256: Option<[u8; 32]>,
    destination_version: Option<FileVersion>,
    use_atomic_rename: bool,
    permit: ExecutionPermit,
}

// src/application/operations.rs
pub(crate) struct ReconciliationRead {
    operation: Operation,
    policy_reference: String,
}

// src/application/operations.rs
pub(crate) struct ExecutionObservation {
    pub(crate) operation_id: Uuid,
    pub(crate) dispatch: E1::DispatchKnowledge,
    pub(crate) exit_code: Option<i32>,
    pub(crate) evidence: VerificationEvidence,
    pub(crate) failure: Option<E6::Failure>,
}

// src/application/operations.rs
pub(crate) struct WorkJoined {
    operation_id: Uuid,
    lease_id: Uuid,
    closed_consumer_ids: Vec<Uuid>,
    joined_task_ids: Vec<Uuid>,
}

// src/application/operations.rs
pub(crate) struct RecoveryReport {
    startup_generation: Uuid,
    interrupted_operation_ids: Vec<Uuid>,
    unresolved_operation_ids: Vec<Uuid>,
    committed_event_ids: Vec<Uuid>,
}

// src/application/operations.rs
pub(crate) struct AuditAppend {
    new_intents: Vec<OperationIntent>,
    events: Vec<OperationRecorded>,
    expected_tails: Vec<(Uuid, Option<AuditTail>)>,
}

// src/application/operations.rs
pub(crate) enum AuditStoreRequest {
    Read(OperationQuery),
    Append(AuditAppend),
}

// src/application/operations.rs
pub(crate) enum AuditStoreResult {
    History(OperationHistory),
    Appended { event_ids: Vec<Uuid>, history: OperationHistory },
}

// src/application/operations.rs
pub(crate) enum OperationMessage {
    InspectResources { query: ResourceQuery, reply: E6::Reply<ResourceObservation> },
    Plan { request: OperationRequest, reply: E6::Reply<PlanResult> },
    Decide {
        operation_id: Uuid, decision: OperationDecision,
        reply: E6::Reply<Operation>,
    },
    InspectOperations { query: OperationQuery, reply: E6::Reply<OperationHistory> },
    Cancel { operation_id: Uuid, reply: E6::Reply<Operation> },
    Reconcile { operation_id: Uuid, reply: E6::Reply<Operation> },
    Observe { reply: E6::Reply<E6::Subscription<OperationHistory, OperationChanged>> },
    BindTransport { consumers: E1::ConsumerAccess, reply: E6::Reply<()> },
    Recover {
        startup_generation: Uuid,
        reply: E6::Reply<RecoveryReport>,
    },
    Quiesce { reply: E6::Reply<OperationQuiesced> },
    Completed(OperationCompletion),
    FinishLocalTrust { operation_id: Uuid, result: Result<E3::TrustCommitObservation, E6::Failure>, reply: E6::Reply<Operation> },
    PrepareLocalTrust { request: OperationRequest, reply: E6::Reply<LocalTrustPlan> },
    AdmitLocalTrust { plan: LocalTrustPlan, decision: OperationDecision, reply: E6::Reply<LocalTrustAdmitted> },
    ReconcileConnected { connection: E1::Connection, reply: E6::Reply<ReconciliationBatch> },
}

// src/application/operations.rs
pub(crate) struct ExecutionPermitView<'a> {
    pub(crate) expected_identity: &'a E2::ConnectionIdentity,
    pub(crate) operation_id: Uuid,
    pub(crate) plan_revision: Uuid,
    pub(crate) write_ahead: AuditTail,
    pub(crate) lease_id: Uuid,
}

// src/application/operations.rs
pub(crate) struct RemoteDispatchView<'a> {
    pub(crate) request: &'a OperationRequest,
    pub(crate) preflight: &'a VerificationEvidence,
    pub(crate) permit: &'a ExecutionPermit,
}

// src/application/operations.rs
pub(crate) struct TransferDispatchView<'a> {
    pub(crate) transfer_id: Uuid,
    pub(crate) request: &'a OperationRequest,
    pub(crate) expected_source_size: u64,
    pub(crate) expected_source_sha256: Option<[u8; 32]>,
    pub(crate) destination_version: Option<&'a FileVersion>,
    pub(crate) use_atomic_rename: bool,
    pub(crate) permit: &'a ExecutionPermit,
}

// src/application/operations.rs
pub(crate) struct ReconciliationReadView<'a> {
    pub(crate) operation: &'a Operation,
    pub(crate) policy_reference: &'a str,
}

// src/application/operations.rs
pub(crate) struct LocalTrustEnvelopeView {
    pub(crate) operation_id: Uuid,
    pub(crate) plan_revision: Uuid,
    pub(crate) expected_audit_tail: AuditTail,
    pub(crate) write_ahead: AuditTail,
    pub(crate) lease_id: Uuid,
    pub(crate) committed_event_id: Uuid,
    pub(crate) committed_event_ordinal: u64,
    pub(crate) committed_at: SystemTime,
}

// src/application/operations.rs
pub(crate) struct AuditBatchData {
    pub(crate) new_intents: Vec<OperationIntent>,
    pub(crate) events: Vec<OperationRecorded>,
    pub(crate) expected_tails: Vec<(Uuid, Option<AuditTail>)>,
}

// src/application/operations.rs
pub(crate) struct WorkJoinedView<'a> {
    pub(crate) operation_id: Uuid,
    pub(crate) lease_id: Uuid,
    pub(crate) closed_consumer_ids: &'a [Uuid],
    pub(crate) joined_task_ids: &'a [Uuid],
}

// src/application/operations.rs
pub(crate) struct RecoveryReportView<'a> {
    pub(crate) startup_generation: Uuid,
    pub(crate) interrupted_operation_ids: &'a [Uuid],
    pub(crate) unresolved_operation_ids: &'a [Uuid],
    pub(crate) committed_event_ids: &'a [Uuid],
}

// src/application/operations.rs
pub(crate) struct OperationQuiesced {
    pub(crate) operation_ids: Vec<Uuid>,
    pub(crate) released_lease_ids: Vec<Uuid>,
}

// src/application/operations.rs
pub(crate) struct OwnedWorkJoined {
    operation_id: Uuid,
    consumer_ends: Vec<E1::ChannelEnd>,
    local_task_results: Vec<(Uuid, Result<(), E6::Failure>)>,
}

// src/application/operations.rs
pub(crate) enum OperationCompletion {
    LocalTrustCommitJoined { operation_id: Uuid, worker_id: Uuid, result: Result<E3::TrustCommitObservation, E6::Failure> },
    QuiesceWorkJoined {
        operation_id: Uuid,
        result: Result<OwnedWorkJoined, E6::Failure>,
    },
    LeaseReleased {
        operation_id: Uuid,
        lease_id: Uuid,
        result: Result<(), E6::Failure>,
    },
}

// src/domain/operation.rs
pub struct DownloadRequest {
    pub source: String,
    pub local_destination: PathBuf,
}

// src/application/operations.rs
pub(crate) enum DownloadProofApproval {}

// src/application/operations.rs
pub(crate) struct DownloadAdmission {
    expected_identity: E2::ConnectionIdentity,
    operation_id: Uuid,
    server_id: Uuid,
    plan_revision: Uuid,
    source: String,
    local_destination: PathBuf,
    expected_source_bytes: u64,
    expected_source_sha256: Option<[u8; 32]>,
    acknowledged: Vec<Confirmation>,
    write_ahead: AuditTail,
    proof: DownloadProofApproval,
}

// src/application/operations.rs
pub(crate) struct DownloadDispatch {
    transfer_id: Uuid,
    admission: DownloadAdmission,
}

// src/application/operations.rs
pub(crate) struct DownloadDispatchView<'a> {
    pub(crate) expected_identity: &'a E2::ConnectionIdentity,
    pub(crate) transfer_id: Uuid,
    pub(crate) operation_id: Uuid,
    pub(crate) server_id: Uuid,
    pub(crate) plan_revision: Uuid,
    pub(crate) source: &'a str,
    pub(crate) local_destination: &'a Path,
    pub(crate) expected_source_bytes: u64,
    pub(crate) expected_source_sha256: Option<[u8; 32]>,
    pub(crate) acknowledged: &'a [Confirmation],
    pub(crate) write_ahead: AuditTail,
}

// src/domain/operation.rs
#[derive(Clone, Copy)]
pub(crate) struct AuditTail {
    pub(crate) event_id: Uuid,
    pub(crate) ordinal: u64,
}

// src/application/operations.rs
pub(crate) struct ReconciliationBatch {
    pub(crate) identity: E2::ConnectionIdentity,
    pub(crate) generation: u64,
    pub(crate) outcomes: Vec<Operation>,
    pub(crate) unavailable: Vec<E6::Failure>,
}

// src/application/operations.rs
pub(crate) struct LocalTrustPlan {
    plan: OperationPlan,
    prepared: E3::PreparedTrustChange,
}

// src/application/operations.rs
pub(crate) struct LocalTrustAdmitted {
    prepared: E3::PreparedTrustChange,
    audit: LocalTrustEnvelope,
}

// ADR-0005 canonical declaration cohort; no positive runtime issuance here
// src/domain/metrics.rs
pub struct CollectionSchedule {
    pub resource_interval: Duration,
    pub service_interval: Duration,
    pub disk_interval: Duration,
}

// src/domain/metrics.rs
pub struct MonitoringSettings {
    pub default_schedule: CollectionSchedule,
    pub per_server: Vec<(Uuid, CollectionSchedule)>,
    pub concurrent_collectors: usize,
}

// src/domain/metrics.rs
pub enum RefreshKind { Metrics, SystemInformation, All }

// src/domain/metrics.rs
pub enum CollectorPhase {
    WaitingForConnection, WaitingForSlot, Collecting, BackingOff, Stopped,
}

// src/domain/metrics.rs
pub struct Collector {
    pub server_id: Uuid,
    pub phase: CollectorPhase,
    pub master_generation: Option<u64>,
    pub consumer_id: Option<Uuid>,
    pub backoff_attempt: u32,
    pub next_attempt_at: Option<Instant>,
    pub failure: Option<E6::Failure>,
}

// src/domain/metrics.rs
pub enum MetricName {
    Hostname, OperatingSystem, Kernel, CpuUsage, LoadAverage,
    MemoryTotal, MemoryAvailable,
    DiskTotal { filesystem: String },
    DiskAvailable { filesystem: String },
    NetworkReceive { interface: String },
    NetworkTransmit { interface: String },
    Uptime, Services,
}

// src/domain/metrics.rs
pub enum MetricUnit {
    Percent, Bytes, BytesPerSecond, Seconds, Count, Text, None,
}

// src/domain/metrics.rs
pub enum MetricValue {
    Number(f64),
    Text(String),
    ServiceSummary(E4::ServiceSummary),
}

// src/domain/metrics.rs
pub enum MetricStatus {
    Fresh { value: MetricValue },
    Stale { value: MetricValue },
    Unavailable { reason: String },
    Error { failure: E6::Failure },
}

// src/domain/metrics.rs
pub struct MetricSample {
    pub server_id: Uuid,
    pub metric_name: MetricName,
    pub unit: MetricUnit,
    pub remote_observed_at: Option<SystemTime>,
    pub collected_at: Instant,
    pub status: MetricStatus,
}

// src/domain/metrics.rs
pub struct Freshness {
    pub elapsed_since_collection: Duration,
}

// src/domain/metrics.rs
pub struct MetricSeries {
    pub metric_name: MetricName,
    pub unit: MetricUnit,
    pub samples: Vec<MetricSample>,
    pub freshness: Option<Freshness>,
}

// src/domain/metrics.rs
pub struct MetricHistory {
    pub server_id: Uuid,
    pub collector: Collector,
    pub series: Vec<MetricSeries>,
    pub collection_pending: bool,
    pub observed_at: Instant,
}

// src/domain/metrics.rs
pub enum MonitoringChanged {
    CollectorChanged { sequence: u64, collector: Collector },
    SampleCollected { sequence: u64, sample: MetricSample },
    SamplesBecameStale { sequence: u64, server_id: Uuid, names: Vec<MetricName> },
    HistoryExpired { sequence: u64, server_id: Uuid, before: Instant },
    ResyncRequired { sequence: u64, server_id: Uuid },
}

// src/application/monitoring.rs
pub struct Monitoring { tx: mpsc::Sender<MonitoringMessage> }

// src/application/monitoring.rs
pub(crate) struct CollectorSpec {
    pub(crate) server_id: Uuid,
    pub(crate) schedule: CollectionSchedule,
    pub(crate) services: E4::ServiceRead,
    pub(crate) collect_system_information: bool,
}

// src/infrastructure/collector/mod.rs
pub(crate) enum RawMetric {
    CpuCounters { total: u64, idle: u64 },
    NetworkCounters { interface: String, received: u64, transmitted: u64 },
    Number { name: MetricName, value: f64, unit: MetricUnit },
    Text { name: MetricName, value: String },
    Services(E4::ServiceSummary),
    Unavailable { name: MetricName, reason: String },
    Error { name: MetricName, failure: E6::Failure },
}

// src/infrastructure/collector/mod.rs
pub(crate) struct RawObservation {
    pub(crate) server_id: Uuid,
    pub(crate) collector_generation: Uuid,
    pub(crate) record_number: u64,
    pub(crate) remote_boot_id: Option<Uuid>,
    pub(crate) remote_observed_at: Option<SystemTime>,
    pub(crate) collected_at: Instant,
    pub(crate) observation: RawMetric,
}

// src/application/monitoring.rs
pub(crate) enum CounterValue {
    Cpu { total: u64, idle: u64 },
    Network { received: u64, transmitted: u64 },
}

// src/application/monitoring.rs
pub(crate) struct CounterBaseline {
    metric_name: MetricName,
    collector_generation: Uuid,
    record_number: u64,
    remote_boot_id: Option<Uuid>,
    collected_at: Instant,
    counters: CounterValue,
}

// src/application/monitoring.rs
pub(crate) struct CollectorBuffer {
    server_id: Uuid,
    samples: VecDeque<MetricSample>,
    last_record_number: Option<u64>,
    baselines: Vec<CounterBaseline>,
}

// src/application/monitoring.rs
pub(crate) struct SettingsCommit {
    settings: MonitoringSettings,
}

// src/application/monitoring.rs
pub(crate) struct SettingsWriteView<'a> {
    pub(crate) settings: &'a MonitoringSettings,
}

// src/application/monitoring.rs
pub(crate) enum MonitoringStoreRequest {
    ReadSettings,
    SaveSettings(SettingsCommit),
}

// src/application/monitoring.rs
pub(crate) enum MonitoringStoreResult {
    Settings(MonitoringSettings),
    Saved(MonitoringSettings),
}

// src/application/monitoring.rs
pub(crate) enum MonitoringMessage {
    Configure { settings: MonitoringSettings, reply: E6::Reply<MonitoringSettings> },
    Refresh { server_id: Uuid, kind: RefreshKind, reply: E6::Reply<Collector> },
    Observe {
        server_id: Uuid,
        reply: E6::Reply<E6::Subscription<MetricHistory, MonitoringChanged>>,
    },
    ConnectionChanged { change: E1::ConnectionChanged },
    ConnectionSnapshot {
        subscription: E6::Subscription<E1::ConnectionView, E1::ConnectionChanged>,
    },
    Completed(MonitoringCompletion),
    Quiesce { reply: E6::Reply<()> },
}

// src/application/monitoring.rs
pub(crate) enum MonitoringCompletion {
    SettingsLoaded { result: Result<MonitoringSettings, E6::Failure> },
    SettingsApplied {
        request_id: Uuid,
        result: Result<MonitoringSettings, E6::Failure>,
    },
    ConsumerOpened {
        server_id: Uuid,
        open_request_id: Uuid,
        result: Result<E1::Consumer, E6::Failure>,
    },
    RecordReceived(RawObservation),
    ConsumerJoined {
        server_id: Uuid,
        consumer_id: Uuid,
        result: Result<E1::ChannelEnd, E6::Failure>,
    },
    ConnectionFeedLost { failure: E6::Failure },
}

// ADR-0006 canonical declaration cohort; no positive runtime issuance here
// src/domain/error.rs
pub enum ErrorCategory {
    AuthenticationError, HostKeyMismatch, HostKeyUntrusted,
    UnregisteredProxyRoute, ConnectionTimeout, NetworkUnavailable,
    PermissionDenied, UnsupportedCapability, ValidationError,
    RemoteExecutionError, VerificationError, ConfigurationConflict,
    FileChangedExternally, StorageError, UnknownRemoteOutcome,
}

// src/domain/error.rs
pub enum EffectKnowledge { NoChange, MayHaveChanged, ChangeVerified }

// src/domain/error.rs
pub enum RecoveryAction {
    EditInventory, ReviewTrust, ReviewPolicy, ExplicitConnect, Disconnect,
    RetryRead, ReloadFile, RequestExplicitOverwrite, InspectOperation,
    CancelPendingWork, Close,
}

// src/domain/error.rs
pub struct Failure {
    pub category: ErrorCategory,
    pub affected_resource: String,
    pub explanation: String,
    pub effects: EffectKnowledge,
    pub actions: Vec<RecoveryAction>,
    pub operation_id: Option<Uuid>,
}

// src/domain/error.rs
pub enum Availability {
    Available,
    Unavailable {
        category: ErrorCategory,
        reason: String,
        required_decision: Option<String>,
    },
}

// src/domain/error.rs
pub type Reply<T> = oneshot::Sender<Result<T, Failure>>;

// src/domain/error.rs
pub struct Subscription<S, E> {
    pub snapshot: S,
    pub next_sequence: u64,
    pub events: mpsc::Receiver<E>,
}

// src/application/runtime.rs
pub enum Invocation {
    Desktop,
    Askpass(E1::AskpassRequest),
}

// src/application/runtime.rs
pub struct ApplicationPaths {
    pub database: PathBuf,
    pub runtime_directory: PathBuf,
    pub executable: PathBuf,
}

// src/application/runtime.rs
pub(crate) struct RuntimeDirectory {
    path: PathBuf,
    owner_uid: u32,
}

// src/application/runtime.rs
pub(crate) struct PrimaryOwnership {
    generation: Uuid,
    application_id: String,
}

// src/application/runtime.rs
pub(crate) struct StartupOwnership {
    generation: Uuid,
    runtime_directory: Arc<RuntimeDirectory>,
}

// src/application/runtime.rs
#[derive(Clone)]
pub(crate) struct ReadyOwnership {
    generation: Uuid,
    runtime_directory: Arc<RuntimeDirectory>,
}

// src/application/runtime.rs
pub enum StartupPhase {
    RegisteringOwner, OpeningStorage, CleaningTransport,
    RecoveringOperations, Ready, Blocked, Stopping, Stopped,
}

// src/application/runtime.rs
pub struct StartupSnapshot {
    pub phase: StartupPhase,
    pub failures: Vec<Failure>,
    pub unresolved_operations: Vec<Uuid>,
}

// src/application/runtime.rs
pub struct StartupChanged {
    pub sequence: u64,
    pub snapshot: StartupSnapshot,
}

// src/application/runtime.rs
pub struct Runtime { tx: mpsc::Sender<RuntimeMessage> }

// src/application/runtime.rs
pub(crate) struct LockAuthority { tx: mpsc::Sender<RuntimeMessage> }

// src/application/runtime.rs
pub(crate) struct ResourceLease {
    id: Uuid,
    owner_generation: Uuid,
    operation_id: Uuid,
    identity: E2::ConnectionIdentity,
    scopes: Vec<E4::ResourceScope>,
}

// src/application/runtime.rs
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

// src/application/runtime.rs
pub(crate) enum LockResult {
    Acquired(ResourceLease),
    Released,
}

// src/application/runtime.rs
pub(crate) struct RuntimePorts {
    pub(crate) runtime: Runtime,
    pub(crate) locks: LockAuthority,
    pub(crate) startup: StartupOwnership,
}

// src/application/runtime.rs
pub struct DesktopServices {
    pub runtime: Runtime,
    pub configuration: E2::Configuration,
    pub trust: E3::Trust,
    pub connections: E1::ConnectionManager,
    pub operations: E4::OperationEngine,
    pub monitoring: E5::Monitoring,
}

// src/application/runtime.rs
pub(crate) struct LifecycleOwners {
    pub(crate) connections: E1::ConnectionManager,
    pub(crate) operations: E4::OperationEngine,
    pub(crate) monitoring: E5::Monitoring,
}

// src/infrastructure/storage/mod.rs
pub(crate) struct Store { tx: mpsc::Sender<StoreMessage> }

// src/infrastructure/storage/mod.rs
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

// src/infrastructure/storage/mod.rs
pub(crate) enum StoreResult {
    Inventory(E2::InventoryStoreResult),
    HostKeys(Vec<E3::HostKey>),
    Audit(E4::AuditStoreResult),
    MonitoringSettings(E5::MonitoringStoreResult),
    LocalTrustCommitted(E3::TrustCommitObservation),
}

// src/infrastructure/storage/mod.rs
pub(crate) enum StoreMessage {
    Request { request: StoreRequest, reply: Reply<StoreResult> },
    Close { reply: Reply<()> },
}

// src/application/runtime.rs
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

// src/application/runtime.rs
pub(crate) struct RuntimeDirectoryView<'a> {
    pub(crate) path: &'a Path,
    pub(crate) owner_uid: u32,
}

// src/application/runtime.rs
pub(crate) struct OwnershipView<'a> {
    pub(crate) generation: Uuid,
    pub(crate) runtime_directory: &'a RuntimeDirectory,
}

// src/application/runtime.rs
pub(crate) struct ResourceLeaseView<'a> {
    pub(crate) id: Uuid,
    pub(crate) owner_generation: Uuid,
    pub(crate) operation_id: Uuid,
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) scopes: &'a [E4::ResourceScope],
}

// src/application/runtime.rs
pub(crate) enum ShutdownCompletion {
    MonitoringStopped { result: Result<(), Failure> },
    OperationsStopped { result: Result<E4::OperationQuiesced, Failure> },
    TransportStopped { result: Result<Vec<E1::ChannelEnd>, Failure> },
    StoreClosed { result: Result<(), Failure> },
}

// src/application/runtime.rs
pub(crate) enum RuntimeConnectionUpdate {
    Snapshot {
        view: E1::ConnectionView,
        next_sequence: u64,
    },
    Changed(E1::ConnectionChanged),
    StreamClosed(Failure),
}
```

These are exact data declarations from the final six Design Types blocks, excluding their impls/function signatures. This ticket additionally implements only InventoryCommit::inventory_write_view, SettingsCommit::settings_write_view, AuditAppend::into_store_batch, PreparedTrustChange::prepared_trust_store_view and LocalTrustEnvelope::local_trust_store_view from the exact Functions below. Their data definitions stay in their canonical Configuration, Trust, Operation and Monitoring modules. No validation, policy, actor method or positive authority constructor is implemented by the cohort. Actual issuers remain E2:01 Configuration/resolution; E3:01 TrustCore/ports; E4:04 audit/recovery and readable execution/evidence projections; E1:01 cleanup/generation/reservation views; E6:02 Ready; E6:03 leases; E3:03 prepared local trust delta; E4:05 local admission/private finish handler; E3:05 actual local commit-worker integration; E4:08 remote execution; E4:09 download; E4:10 remote joined completion; E5:01 Collector/SettingsCommit. Shared declarations never stand in for these issuers.

## Functions

### `Store::open(primary: &PrimaryOwnership, paths: &ApplicationPaths) -> Result<Store, Failure>`

- **Intent.** Open the actual compatible persisted state under sole ownership.
- **Where.** `src/infrastructure/storage/mod.rs`.
- **Pre.** Current process is primary; database path is the intended application Store.
- **Post.** Active foreign keys, UUID identities and versioned non-truncating migrations; real inventory, HostKey, settings and audit queries are available. Fresh database creates actual accepted tables, including operations/events.
- **Errors.** StorageError/migration incompatibility returns no operational Store, never substitutes in-memory empty state.
- **Idempotency.** Applied migrations are not re-applied; failed migration rolls back; no audit/key truncation.

### `Store::request(&self, request: StoreRequest) -> Result<StoreResult, Failure>`

- **Intent.** Execute one owner-authorized domain persistence transaction faithfully.
- **Where.** `src/infrastructure/storage/mod.rs`.
- **Pre.** Open primary Store; matching request/response variant; batch created by authorized semantic owner; FK and expected-state constraints hold.
- **Post.** Read returns actual committed data. Inventory and settings writes accept only Configuration-created InventoryCommit and Monitoring-created SettingsCommit, never caller-built observation vectors. Store uses InventoryCommit::inventory_write_view and SettingsCommit::settings_write_view only after receiving the sealed requests. Inventory/settings/audit append uses all-or-nothing transaction. CommitLocalTrust compares expected keys/tail, applies E3 key changes, reads back and appends the E4 committed-change fact in one transaction. Audit UPDATE/DELETE are unavailable. Validate ordinal 1 for a first fact and contiguous following ordinals per Operation, exact event UUID/content and expected AuditTail inside the transaction; ordinal lives only in existing sanitized_payload. Fold by validated ordinal, never SQL rowid/time/UUID. Local trust returns its exact committed ordinal. Missing/duplicate/noncontiguous/conflicting positions roll back the entire request. L1 additionally requires complete intended-set read-back before the exact reserved LocalTrustCommitted event, with original Operation/revision/identity/lease and plan-digest bindings. AuditAppend rejects that event outside CommitLocalTrust. A settled attributable paired event proves historical commit even after later audited changes; current rows alone do not. No extra stream, schema or attestation is introduced.
- **Errors.** Validation/CAS/FK failure or missing, duplicate, noncontiguous or conflicting payload ordinals returns ConfigurationConflict where the submitted batch conflicts; malformed persisted facts, schema/read/write/migration failures return StorageError. Roll back the entire request with no silent truncation or secret/content persistence. Unknown delivery does not authorize blind mutation repeat; reread actual history/tail before another append.
- **Idempotency.** Reads repeat safely. Owner/event identities make matching append/local commit replay return retained result or conflict, never duplicate effects or consume a new ordinal. Failed/ambiguous append requires actual tail/history read, not an assumed rollback. Explicit Import semantics remain E2’s, not Store inference.

### `Store::close(&self) -> Result<(), Failure>`

- **Intent.** End physical persistence only after all semantic owners finish.
- **Where.** `src/infrastructure/storage/mod.rs`.
- **Pre.** Runtime has stopped new admission and joined domain owners.
- **Post.** Submitted transactions settle and connection closes; no pending successful transaction is falsely acknowledged.
- **Errors.** Close/transaction failure is explicit and does not certify shutdown success.
- **Idempotency.** Repeated close joins/returns completed close.

### `Runtime::begin(primary: PrimaryOwnership, store: Store, directory: RuntimeDirectory) -> RuntimePorts`

- **Intent.** Establish pre-admission lifecycle and resource arbitration capabilities.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Current primary; directory owned/restricted to `0700`.
- **Post.** Returns runtime/locks/startup capabilities; phase is pre-Ready; no ReadyOwnership or master admission exists.
- **Errors.** Infallible after prerequisites; real prerequisite failures occur before this constructor.
- **Idempotency.** Once per primary generation; does not inherit old in-memory leases.

### `Runtime::observe(&self) -> Result<Subscription<StartupSnapshot, StartupChanged>, Failure>`

- **Intent.** Explain actual recovery/admission/shutdown state to presentation.
- **Where.** `src/application/runtime.rs`.
- **Pre.** Primary runtime exists.
- **Post.** Returns real phase/blockers/unresolved IDs, not Server availability or fabricated recovery success.
- **Errors.** Closed owner yields explicit failure; prior blockers are retained in state.
- **Idempotency.** Read-only subscription; never retries mutations.

### `RuntimeDirectory::inspect_runtime_directory(&self) -> RuntimeDirectoryView<'_>`

- **Intent.** Read the validated confined runtime path and owner UID.
- **Where.** `src/application/runtime.rs`.
- **Pre.** A live owner-issued receiver exists.
- **Post.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

### `StartupOwnership::inspect_startup_ownership(&self) -> OwnershipView<'_>`

- **Intent.** Bind actual preadmission safety work to the primary generation.
- **Where.** `src/application/runtime.rs`.
- **Pre.** A live owner-issued receiver exists.
- **Post.** Returns exact borrowed fields without minting a wrapper. No authority-accepting entry accepts the view instead of the wrapper.
- **Errors.** Infallible; no I/O and no replacement authority allocation.
- **Idempotency.** Repeatable pure read; references cannot outlive receiver.

### `InventoryCommit::inventory_write_view(&self) -> InventoryWriteView<'_>`

- **Intent.** Let Store read the validated inventory delta without allowing callers to mint a write request.
- **Where.** `src/application/configuration.rs`.
- **Pre.** Store received an owner-created InventoryCommit inside InventoryStoreRequest::Commit.
- **Post.** Borrows exact expected revision, Servers and Auth Profiles; Store still requires the private wrapper, not a caller-created InventoryWriteView.
- **Errors.** Infallible; no I/O, mutation or authority allocation.
- **Idempotency.** Repeatable borrowed read; no duplicate commit is issued.

### `PreparedTrustChange::prepared_trust_store_view(&self) -> PreparedTrustChangeView<'_>`

- **Intent.** Let local admission and Store inspect the exact trust-owner delta without accepting a forged projection.
- **Where.** `src/application/trust.rs`.
- **Pre.** A live TrustCore-created PreparedTrustChange exists; a local admission reader may inspect it, while persistence separately requires the sealed E4 envelope.
- **Post.** Borrows operation/Server/identity/endpoint, expected/replacement keys, path and evidence digest; neither E4 admission nor Store accepts a caller-created view instead of the original wrapper.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable borrowed read; does not apply a mutation.

### `SettingsCommit::settings_write_view(&self) -> SettingsWriteView<'_>`

- **Intent.** Let Store persist validated settings without accepting arbitrary MonitoringSettings writes.
- **Where.** `src/application/monitoring.rs`.
- **Pre.** Commit constructed only by Monitoring after configure validation.
- **Post.** Borrows exact settings authorized for settings-only transaction; view does not mint commit and no Store entry accepts a view or arbitrary settings for SaveSettings.
- **Errors.** Infallible projection; transaction failures remain Store errors.
- **Idempotency.** Repeatable inspection of one immutable commit, no additional write or sample persistence.

### `AuditAppend::into_store_batch(self) -> AuditBatchData`

- **Intent.** Move the authorized immutable append batch into Store execution.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Single consumption; cannot call twice on moved receiver.

### `LocalTrustEnvelope::local_trust_store_view(&self) -> LocalTrustEnvelopeView`

- **Intent.** Read the E4-authored atomic trust/audit admission.
- **Where.** `src/application/operations.rs`.
- **Pre.** A live owner-created receiver exists.
- **Post.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

## Invariants

| ID    | Statement                                                                                                                  | Role      | Proof                                                                |
| ----- | -------------------------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------- |
| INV-1 | Only the current GtkApplication primary opens operational Store and mints current-generation ownership capabilities.       | establish | `single_primary_real_store_and_preadmission`; first proof 01         |
| INV-5 | GTK owns widgets and remains responsive with truthful ownership/recovery/shutdown states.                                  | establish | `runtime_gtk_preadmission_and_error_states`; first proof 01          |
| INV-8 | Store accepts only sealed canonical-owner writes; immutable projections expose exact data and never authorize a new write. | establish | `store_sealed_owner_requests_and_projection_privacy`; first proof 01 |

## Flow

Design flow 1: actual primary → compatible Store → preadmission Runtime; five imported pure projections precede their Store request uses.

- In: `PrimaryOwnership` ← external
- Out: `StartupOwnership` → 02
- Out: `Store` → 02
- Out: `LockAuthority` → 03
- Out: `Runtime` → 04
- Out: `Store` → 04
- In: `ApplicationPaths` ← external
- Out: `StartupSnapshot` → final

The actual private primary GTK adapter in application/runtime.rs creates PrimaryOwnership only after real registration and validates RuntimeDirectory. Filesystem/session bus/ApplicationPaths are external inputs, not assumed owner services. The same ticket supplies the mechanical Store projections before request branches use them.

## Steps

1. Write the failing behavior proof for Introduce the exact common data/enum/channel-handle declarations below in their canonical modules, and implement ONLY the five exact pure Store projections in Functions. No owner constructors, late actor methods, fake Ready/backends or capability issuance is included; each actual runtime producer remains its domain ticket. Then implement until that proof passes.
2. Write the failing behavior proof for Implement actual GTK primary registration and storage/application path adapter for this preadmission slice, not the final full launch function. Duplicate launches activate the sole owner; neither duplicates nor restricted askpass can open operational Store or independently own locks/masters. Then implement until that proof passes.
3. Write the failing behavior proof for Open actual SQLite at the intended path, apply accepted versioned migrations with active foreign keys and UUID relationships, and return actual schema/read failures. A fresh database creates real inventory/key/settings/audit tables; an incompatible/missing/failed store never becomes assumed empty. Then implement until that proof passes.
4. Write the failing behavior proof for Implement Store requests as actual single-connection mechanical transactions using the five canonical infallible projections already implemented by this same slice. Only Configuration-created InventoryCommit, Monitoring-created SettingsCommit, E4 AuditAppend and LocalTrustEnvelope plus E3 PreparedTrustChange authorize their respective closed requests; arbitrary views or vectors cannot authorize a write. Then implement until that proof passes.
5. Write the failing behavior proof for For audit, validate expected tails and contiguous ordinals encoded only in existing sanitized_payload, event UUID/content uniqueness and FK atomically; preserve append-only evidence and return exact committed local-trust ordinal/read-back facts. Failed/ambiguous writes require actual tail/history reread before another append. Then implement until that proof passes.
6. Write the failing behavior proof for Runtime::begin yields real current-generation preadmission state and StartupOwnership; observe shows actual loading/error/blocked state while GTK stays responsive. No Connect or Ready is exposed. Implement close to settle transactions honestly, without claiming the final owner shutdown protocol already exists. Then implement until that proof passes.
7. Run future Verify commands and preserved earlier invariant Proofs.

## Acceptance criteria

- [ ] `US-001-AC1`: Given an active ServerDesk GtkApplication instance, when I launch the application again, then no second independent application owner opens its own operational ownership of the same SQLite state, application ControlMasters, or lock registry.
- [ ] `US-001-AC2`: Given simultaneous launch attempts, when application ownership is established, then only one owner performs application startup recovery and admits new managed connection or mutation work.
- [ ] `US-011-AC1`: Given startup cleanup, recovery persistence, preflight, or Reconciliation is running, when I interact with GTK, then the main thread remains responsive and affected work has a loading/recovery presentation rather than an unsupported success claim.
- [ ] `US-011-AC3`: Given a cleanup failure, storage failure, conflict, or unresolved outcome, when the error is displayed, then it identifies what failed, the affected Server/resource, whether a change may have occurred, available actions, and related Operation details where applicable.
- [ ] `US-011-AC4`: Given recovery details or a rejected-conflict surface, when I use keyboard navigation, then I can reach the details and available actions without a pointer or a forced automatic mutation retry.
- [ ] Establish each invariant Proof listed here and preserve earlier Proofs. Full runtime integration criteria are exercised again by04 with actual domain issuers, not forged early capabilities.
- [ ] No declaration-only owner, no-op backend, empty Store fallback or unsupported Ready/completed-outcome claim is introduced.

## Verify

Future implementation gates: author these exact affected tests/harnesses here; none are claimed present or passing now.

```sh
cargo test --test runtime_primary_store
rustfmt --edition 2024 --check src/domain/operation.rs src/domain/resource_observation.rs src/application/operations.rs src/domain/error.rs src/application/runtime.rs src/infrastructure/storage/mod.rs src/domain/connection.rs src/application/connection_manager.rs src/ui/transport.rs src/infrastructure/ssh/mod.rs src/application/configuration.rs src/domain/connection_identity.rs src/infrastructure/ssh/config.rs src/application/trust.rs src/application/monitoring.rs src/infrastructure/collector/mod.rs src/domain/inventory.rs src/domain/trust.rs src/domain/metrics.rs src/main.rs src/ui/runtime.rs src/domain/mod.rs src/application/mod.rs src/infrastructure/mod.rs src/ui/mod.rs tests/runtime_primary_store.rs
```

Run actual GTK primary/duplicate registration under a session bus and real SQLite migration/transaction/FK/read/failure/close journeys. Exercise sealed-write/privacy boundaries and actual append-only/payload-ordinal guards, not source-text/greeting assertions. Incompatible/missing/denied Store is not empty. Actual GTK surfaces must remain responsive and keyboard accessible; errors/Unknown/stale state remain honest. Disk-full only on an isolated container filesystem/loop device, never the host. Run earlier preserved invariant Proofs as affected regressions.

## Do not touch

No foreign semantic writer transfer, future owner constructor, generic proof fallback, remote/distributed lock, persisted lease, new scope hierarchy, hidden mutation queue, automatic mutation replay, schema/audit reset/deletion, secret/file-content persistence or permanent greeting/wiring/text tests. Only04 owns final production launch composition.

## Comments

E1, T3, L1 and S were approved on 2026-10-31. ClosedEnvironment is declared only in canonical src/domain/inventory.rs. The six EnabledMutation service cases and three LocalTrustApproval cases match E4 Design exactly. DownloadProofApproval stays empty. This declaration cohort creates no positive domain capability. Store must enforce L1's exact sealed bindings, atomic complete-set read-back and paired event; the ordinary AuditAppend path rejects free-standing LocalTrustCommitted. Actual E2/E3/E4 producers and their first Proofs remain pending. No source acceptance criterion or runtime checkbox is passed by policy approval.
