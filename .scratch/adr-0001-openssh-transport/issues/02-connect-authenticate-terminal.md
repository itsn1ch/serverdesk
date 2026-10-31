# 02: Connect and authenticate one shared master with independent terminals

**Status:** needs-info
**Blocked by:** 01: Clean actual orphan transport before admission.
**External blockers:** ADR-0006:01, ADR-0006:02, ADR-0002:04, ADR-0003:04, ADR-0003:05.
**Source:** US-001-AC1, US-001-AC2, US-001-AC4, US-002-AC1, US-002-AC2, US-002-AC3, US-002-AC4, US-002-AC5, US-003-AC1, US-003-AC2, US-003-AC6, US-004-AC1, US-004-AC2, US-004-AC3, US-004-AC4, US-004-AC5, US-004-AC6, US-005-AC1, US-005-AC2, US-005-AC3, US-005-AC4, US-005-AC5, US-005-AC6, US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4, US-007-AC5, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-014-AC1, US-014-AC2, US-014-AC3, US-014-AC4, US-014-AC5.
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

The Developer can connect explicitly (or through the saved startup preference), authenticate through system OpenSSH, and use independent real VTE shells with truthful states. Direct start consumes actual configuration/trust/admission; it never invents policy or bypasses pending trust acceptance.

External producers: ADR-0006:02 Runtime::admit issues ReadyOwnership only after this epic 01 plus actual durable recovery. ADR-0002:01 supplies Configuration::resolve and ADR-0002:04 supplies actual compile/launch_view; ADR-0003:04 supplies TrustGate::prepare and PreparedStart::into_prepared_start. ADR-0003:05 supplies real confirmed trust acceptance when needed; accepted-key fixtures use actual SQLite, not forged TrustStart. TransportUi is external GTK bootstrap input produced by install_transport_ui owned IN THIS ticket and passed by E6 composition. Declaration availability from ADR-0006:01 does not issue any of these capabilities. Unresolved ADR-0003:02 eligibility and ADR-0004:01 local trust policy transitively gate integrated Connect; status stays needs-info, no alternative policy chosen.

## Types

```rust
ConnectionState // from ADR-0006:01
CapabilityKind // from ADR-0006:01
Capability // from ADR-0006:01
Connection // from ADR-0006:01
TerminalState // from ADR-0006:01
Terminal // from ADR-0006:01
TerminalControl // from ADR-0006:01
ConnectionControl // from ADR-0006:01
ConnectionResult // from ADR-0006:01
ConnectionView // from ADR-0006:01
ConnectionChanged // from ADR-0006:01
ConnectionManager // from ADR-0006:01
ConsumerAccess // from ADR-0006:01
RouteAccess // from ADR-0006:01
ConnectionPorts // from ADR-0006:01
MasterGeneration // from ADR-0006:01
StartReservation // from ADR-0006:01
DispatchKnowledge // from ADR-0006:01
ChannelEndReason // from ADR-0006:01
ChannelEnd // from ADR-0006:01
ConsumerRequest // from ADR-0006:01
ConsumerEvent // from ADR-0006:01
Consumer // from ADR-0006:01
ConnectionMessage // from ADR-0006:01
AuthenticationDialog // from ADR-0006:01
AuthenticationAnswer // from ADR-0006:01
AskpassRequest // from ADR-0006:01
TerminalBinding // from ADR-0006:01
TerminalExit // from ADR-0006:01
TerminalUiEvent // from ADR-0006:01
TerminalLifecycleReport // from ADR-0006:01
TerminalLifecycleSink // from ADR-0006:01
PrivateTerminalLaunch // from ADR-0006:01
VteSpawnReport // from ADR-0006:01
TerminalUiCommand // from ADR-0006:01
TransportUi // from ADR-0006:01
CleanupReport // from ADR-0006:01
MasterGenerationView // from ADR-0006:01
StartReservationView // from ADR-0006:01
CleanupReportView // from ADR-0006:01
TrustWork // from ADR-0006:01
CompilableStart // from ADR-0006:01
MasterSpawnObservation // from ADR-0006:01
ConnectCompletion // from ADR-0006:01
ReadyOwnership // from ADR-0006:01
Configuration // from ADR-0006:01
ResolvedConnection // from ADR-0006:01
CompiledConfiguration // from ADR-0006:01
TrustGate // from ADR-0006:01
PreparedStart // from ADR-0006:01
TrustStart // from ADR-0006:01
Failure // from ADR-0006:01
```

These are real foreign declaration prerequisites, not existing code. ADR-0006:01 defines each record/enum/handle in its canonical owner module without implementing the methods below or minting runtime authority. This ticket implements only its owned methods/behavior; the Contract Types block supplies exact fields and privacy. No caller-created borrowed view authorizes launch, execution, persistence or lease release.

## Functions

### `ConnectionManager::spawn(ready: E6::ReadyOwnership, configuration: E2::Configuration, trust: E3::TrustGate, ui: TransportUi) -> ConnectionPorts`

- **Intent.** Give admitted workflows one transport lifecycle owner.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** ReadyOwnership, Configuration, TrustGate, TransportUi.
- **Writes.** ConnectionManager, ConnectionPorts.
- **Pre.** ReadyOwnership is from current primary after actual cleanup and durable recovery; one composition call.
- **Post.** Consumes TransportUi via into_transport_ui_parts; installs one owner loop with separate command/lifecycle receivers, identity registry and pending workers; starts no master.
- **Invariants.** INV-1, INV-6, INV-7.
- **Errors.** Construction has no operational result; later worker failures are reported through owned requests/events.
- **Idempotency.** Only one manager is composed per admitted primary; handles may be cloned, not owner construction.

### `ConnectionManager::control(&self, request: ConnectionControl) -> Result<ConnectionResult, E6::Failure>`

- **Intent.** Apply explicit connection or terminal intent while retaining the original target.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** ConnectionControl, Connection, Terminal, E2::ResolvedConnection, E3::PreparedStart, E2::CompiledConfiguration.
- **Writes.** Connection, Terminal, MasterGeneration, StartReservation.
- **Pre.** Admission open; Server/terminal exists; terminal Open requires usable generation; Reconnect names disconnected shell.
- **Post.** Resolution completion reserves/coalesces structural identity; owned route, trust and compile workers return original reservation through PreparedStart::into_prepared_start before sealed launch. Every completion validates attempt/start/generation. Created children are returned even on failed control checks and registered for cleanup. Connected requires live process and ssh -O check. Terminal uses fresh shell binding; exit/failure and PTY facts govern state/drain, not late spawn acknowledgement. Disconnect denies new children and joins descendants before closing master.
- **Invariants.** INV-1, INV-2, INV-3, INV-4, INV-5, INV-6, INV-7, INV-8.
- **Errors.** Configuration/storage/trust/route errors prevent start; unknown 255/stderr remains Unknown; cancellation never publishes Connected. Stale completions cannot revive state; returned children remain owned until joined. Incomplete cleanup never reports completed disconnect.
- **Idempotency.** Concurrent same-identity Connect joins one reservation; connected Connect reuses generation; repeated Close/Disconnect joins cleanup; each explicit Open/Reconnect creates a new shell, never resumes old shell or replays mutations.

### `ConnectionManager::observe(&self) -> Result<E6::Subscription<ConnectionView, ConnectionChanged>, E6::Failure>`

- **Intent.** Observe current connection and terminal truth without initiating transport.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** Connection, Terminal, E2 inventory.
- **Writes.** None.
- **Pre.** Manager exists in current primary.
- **Post.** Snapshot plus sequenced changes starts at one owner observation boundary; never-attempted entries are NotConnected; trust remains separate; lag requires actual resubscription.
- **Invariants.** INV-1, INV-4, INV-5, INV-6.
- **Errors.** Closed owner or inventory/storage failure yields explicit failure, not a connection attempt or empty success.
- **Idempotency.** Resubscription returns current observations without new authentication, shell, or changed timestamps.

### `serve_askpass(request: AskpassRequest) -> Result<i32, E6::Failure>`

- **Intent.** Relay one authorized OpenSSH challenge to the existing primary GTK prompt.
- **Where.** `src/infrastructure/ssh/mod.rs`
- **Reads.** AskpassRequest, AuthenticationDialog, AuthenticationAnswer.
- **Writes.** Ephemeral IPC only.
- **Pre.** IPC UID/permissions, nonce, attempt and challenge match active request.
- **Post.** Helper starts no primary or Store; accepted bytes are sent once to requesting OpenSSH stdout and zeroized, cancellation returns non-success; main-thread prompt remains responsive.
- **Invariants.** INV-5, INV-6, INV-7.
- **Errors.** Stale/replayed/invalid challenge, owner loss, prompt cancellation or IPC failure releases secret buffers and returns no secret; no diagnostic contains answer.
- **Idempotency.** Challenge is single-use, no cached answers or per-consumer authentication.

### `TerminalLifecycleSink::report(&self, event: TerminalUiEvent) -> Result<(), E6::Failure>`

- **Intent.** Deliver actual child and PTY observations for the sink-bound shell.
- **Where.** `src/application/connection_manager.rs`
- **Reads.** TerminalBinding, TerminalUiEvent.
- **Writes.** TerminalLifecycleReport only.
- **Pre.** Installed GTK adapter reports observed event using privately issued sink.
- **Post.** Stamps original terminal/shell/master plus local time and enqueues without blocking GTK; only manager mutates Terminal.
- **Invariants.** INV-3, INV-5, INV-6, INV-8.
- **Errors.** Closed owner is actionable failure; missing report never counts as child/PTY release.
- **Idempotency.** Duplicate reports set existing per-shell flags; old-shell report cannot satisfy replacement barrier.

### `install_transport_ui(application: &adw::Application, terminal_dock: &adw::TabView) -> TransportUi`

- **Intent.** Install real GTK authentication and VTE endpoints for transport.
- **Where.** `src/ui/transport.rs`
- **Reads.** Primary GTK application, terminal dock.
- **Writes.** TransportUi and GTK adapter state.
- **Pre.** GTK main thread; one installation per primary/dock.
- **Post.** Returns opaque privately constructed channel capability; adapters consume readable launch DTO, spawn actual VTE child, report spontaneous child exit, spawn failure and PTY disposal through bound sink. No Store, manager or SSH master starts during installation.
- **Invariants.** INV-3, INV-5, INV-6, INV-8.
- **Errors.** Later prompt/spawn failures return through declared channels; installation itself does no asynchronous transport.
- **Idempotency.** Install once and reuse capability; ordinary presentation never constructs sender capabilities.

### `TransportUi::into_transport_ui_parts(self) -> (mpsc::Sender<AuthenticationDialog>, mpsc::Sender<TerminalUiCommand>)`

- **Intent.** Move the installed adapter capability into manager-owned prompt and terminal endpoints.
- **Where.** `src/ui/transport.rs`
- **Reads.** TransportUi.
- **Writes.** None.
- **Pre.** Capability was issued by install_transport_ui and is not previously consumed.
- **Post.** Returns its exact two sender endpoints; consumes wrapper; no fabricated channel or GTK widget crosses to Tokio.
- **Invariants.** INV-5, INV-6.
- **Errors.** Infallible consuming decomposition.
- **Idempotency.** One consuming call; endpoints may be retained by sole manager, not reconstructed from launch DTOs.

## Invariants

| ID     | Statement                                                                                                                  | Role      | Proof                                                                                 |
| ------ | -------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------- |
| INV-1  | At most one starting/live master exists for each structurally equal Connection Identity.                                   | establish | Future test `transport_shared_identity_and_components`; first proof 02                |
| INV-2  | An attachment uses only its specified current master generation or fails without authentication.                           | establish | Future test `transport_mux_loss_no_fallback`; first proof 02                          |
| INV-3  | Closing a terminal requires actual child termination/reaping and PTY release for that shell.                               | establish | Future test `transport_terminal_pty_drain`; first proof 02                            |
| INV-4  | Each specific Connection State and Capability has independent supporting evidence; 255/stderr alone is Unknown.            | establish | Future test `transport_evidence_classification`; first proof 02                       |
| INV-5  | Every terminal has independent streams/PTY and immutable original Server context.                                          | establish | Future test `transport_vte_independent_context`; first proof 02                       |
| INV-6  | Master/authentication start requires current primary admission, generated configuration, trust and GTK-owned presentation. | establish | Future test `transport_admission_authentication_and_secret_exclusion`; first proof 02 |
| INV-7  | Waiting route/trust/configuration/launch work never prevents servicing dependent manager messages.                         | establish | Future test `transport_continuation_cancellation`; first proof 02                     |
| INV-8  | Terminal state/drain uses exact shell/master binding and actual spontaneous child/PTY observations.                        | establish | Future test `transport_terminal_early_exit_and_stale_report`; first proof 02          |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan.                                                 | preserve  | Future test `transport_orphan_cleanup`; first proof 01                                |

Every preserved invariant depends on its first proof owner through the local blocker chain; its earlier test remains required. Proof names below are planned tests, not observed results.

## Flow

Bootstrap/connect and independent terminals; implement the matching ordered Contract steps in sequence. Foreign inputs below are external to this epic, with actual qualified producers identified in Goal and External blockers.

- In: `CleanupReport` ← 01
- In: `ReadyOwnership` ← external
- In: `Configuration` ← external
- In: `TrustGate` ← external
- In: `TransportUi` ← external
- Out: `ConnectionPorts` → 03
- Out: `ConnectionPorts` → 04
- Out: `ConnectionPorts` → 05

## Steps

1. Write failing direct Connect/alias-identity and actual GTK authentication/VTE workflow tests for the assigned source criteria before implementation. Seed actual SQLite accepted keys through the established schema; never simulate positive authority by constructing views.
2. Implement spawn, opaque UI issuance/decomposition and the owner-loop pending table. Resolve before identity reservation; run route/trust/compile/launch through owned continuations, consume E3 PreparedStart to retain original reservation, and return every spawned child on failure.
3. Implement actual system OpenSSH foreground master, restricted control path, application-lifetime spawning-supervisor parent-death/parent-race supervision and observed ssh -O check. Keep deny-fallback attachment material and specific evidence classification.
4. Implement agent/askpass/FIDO-supported path with UID/nonce/challenge authorization, GTK keyboard prompt cancellation and zeroized secret exclusion; helper never opens Store or becomes primary.
5. Implement per-shell VTE spawn/input/output/resize, immutable original Server, new-shell reconnect, spontaneous exit, early exit-before-spawn acknowledgement, stale generation rejection and exact child-plus-PTY close barrier.
6. Implement actual connection/capability observation and accessible loading/empty/failure/stale states on main context. Treat service availability separately from exact E4 sudo permissions.
7. Run the exact affected tests and changed-file formatter checks in Verify after implementation; report exercised results and any unresolved policy/fixture gate truthfully.

## Acceptance criteria

- [ ] `US-001-AC1`: Given two Server entries with the same host, port, SSH user, complete Jump Server route and authentication context, when both are connected or used concurrently, then their consumers use exactly one ControlMaster for that Connection Identity.
- [ ] `US-001-AC2`: Given entries differing in host, port, SSH user, Jump Server route or authentication context, when they connect, then they do not reuse a master belonging to the different Connection Identity.
- [ ] `US-001-AC4`: Given inventory is saved without a connection attempt, when it is subsequently displayed, then saving has not required a remote connection.
- [ ] `US-002-AC1`: Given a saved Server with the default `connect_on_startup=false`, when the application starts, then no master is started for that Server solely because it is in inventory.
- [ ] `US-002-AC2`: Given the Developer activates Connect, when configuration and trust prerequisites permit connection, then the connection manager starts the Connection Identity's master and the UI exposes `CONNECTING` until the attempt has an observed result.
- [ ] `US-002-AC3`: Given a Server with `connect_on_startup=true`, when startup safety processing owned by ADR-0006 is complete, then its connection attempt uses the same configuration, trust and authentication gates as explicit Connect.
- [ ] `US-002-AC4`: Given a master starts, when its control socket is created, then it resides within `$XDG_RUNTIME_DIR/serverdesk/` with that runtime directory restricted to permission `0700`.
- [ ] `US-002-AC5`: Given the UI claims `CONNECTED`, when the claim is checked against the actual master process and `ssh -O check`, then it reflects a usable master rather than the mere fact that process creation was requested.
- [ ] `US-003-AC1`: Given a master is about to start, when its transport is launched, then it uses `-F` with generated configuration for its Connection Identity and the generated Host Key material supplied by ADR-0003, not the user's runtime SSH configuration.
- [ ] `US-003-AC2`: Given the Host Key gate reports a mismatch or a Host Key not accepted for connection under ADR-0003, when Connect is requested, then the connection is blocked with `HOST_KEY_MISMATCH` or `HOST_KEY_UNTRUSTED` respectively and no terminal or administrative channel is admitted through that attempt.
- [ ] `US-003-AC6`: Given the connection changes state, when the Server is displayed, then the Connection State remains distinguishable from its separate Trust State and connection success does not promote Host Key trust.
- [ ] `US-004-AC1`: Given a working ssh-agent exposed by `SSH_AUTH_SOCK`, including a GNOME Keyring agent, when the selected Auth Profile can authenticate through it, then the system OpenSSH master authenticates using that agent.
- [ ] `US-004-AC2`: Given agent authentication is unavailable or insufficient and authentication requires the askpass fallback, when OpenSSH invokes `SSH_ASKPASS`, then it invokes the ServerDesk executable's askpass mode and presents a GTK prompt rather than requiring an embedded SSH implementation.
- [ ] `US-004-AC3`: Given a prompt is accepted and the master authenticates, when additional consumers attach to that master, then the application does not initiate separate authentication or repeat the master-authentication prompt for each consumer.
- [ ] `US-004-AC4`: Given a prompt is cancelled or authentication is rejected, when the attempt ends, then the application does not claim `CONNECTED`, retains no entered secret and shows the observed outcome with an available connection action.
- [ ] `US-004-AC5`: Given the selected key and local system OpenSSH support FIDO authentication, when that key is used, then authentication remains on the system OpenSSH path and its required user interaction is not replaced by a stored application secret.
- [ ] `US-004-AC6`: Given an Auth Profile is saved or an authentication attempt is inspected in persisted state and diagnostic details, when authentication completes or fails, then only authentication type and key/agent references are retained, never password, passphrase or authentication secret; Secret Service is not used for MVP SSH authentication.
- [ ] `US-005-AC1`: Given a connection has never been attempted, when it appears in inventory or fleet summaries, then it is `NOT_CONNECTED` and is not counted as a failed or offline Server.
- [ ] `US-005-AC2`: Given an attempt is in progress, when GTK handles unrelated navigation or input, then `CONNECTING` remains visible without blocking the GTK main thread.
- [ ] `US-005-AC3`: Given independently established authentication failure, trust rejection, timeout or network unreachability, when the workflow reports its result, then it uses the corresponding `AUTH_FAILED`, `HOST_KEY_MISMATCH`, `HOST_KEY_UNTRUSTED`, `TIMEOUT` or `UNREACHABLE` Connection State rather than conflating those outcomes.
- [ ] `US-005-AC4`: Given SSH exits with code 255 and no independent evidence establishes a specific cause, when the attempt is classified, then the result is `UNKNOWN`; stderr text and exit 255 alone do not establish authentication failure, Host Key mismatch, timeout or network unreachability.
- [ ] `US-005-AC5`: Given a Host Key mismatch is reported, when its evidence is inspected, then it comes from the Host Key preflight defined by ADR-0003 rather than parsing SSH stderr.
- [ ] `US-005-AC6`: Given any failed connection attempt, when the UI presents the result, then it identifies the affected Server or Connection Identity, what failed, whether changes may have occurred, available actions and applicable Operation details, without claiming the remote Server is dead.
- [ ] `US-006-AC1`: Given a connection is established, when Capability detection runs, then observations cover OS/distribution, systemd, `busctl --json`, the SFTP subsystem, `posix-rename@openssh.com`, and `sha256sum` rather than assuming their presence.
- [ ] `US-006-AC2`: Given an administrative Operation is evaluated for availability, when its permission is checked, then the Operation engine's exact-command-and-arguments `sudo -n -l` result is used rather than a blanket claim that the SSH user has sudo access.
- [ ] `US-006-AC3`: Given a Capability is absent or permission is denied, when the corresponding feature is offered, then it is unavailable with the observed reason, and the application neither invents a successful detection result nor disables unrelated supported consumers.
- [ ] `US-006-AC4`: Given the Server does not provide a human-oriented command's desired output format, when service Capabilities are evaluated, then the accepted machine-readable read mechanisms are considered and Services is not disabled solely because `systemctl list-units` lacks JSON.
- [ ] `US-007-AC1`: Given a usable Connection Identity, when a terminal is opened, then VTE spawns system `ssh -F <generated> -t <identity>` and the terminal attaches through that identity's ControlMaster with a remote interactive PTY.
- [ ] `US-007-AC2`: Given an active terminal, when `vim`, `top` or `htop` is used, then interactive input and terminal output work and resizing the terminal updates the remote PTY dimensions.
- [ ] `US-007-AC3`: Given multiple terminal sessions, including sessions for different Servers, when they exchange input and output, then each stream belongs to its own session and output does not mix between sessions.
- [ ] `US-007-AC4`: Given an active terminal, when the Developer switches pages or the global Server selection, then that terminal remains active with its original Server context rather than being closed or retargeted.
- [ ] `US-007-AC5`: Given a command is typed in the terminal, when it runs, then it is not represented as a structured Operation and its command text is not added to the application's Audit Trail.
- [ ] `US-008-AC1`: Given a terminal is opened, when its lifecycle advances, then the visible state follows `CREATED → CONNECTING → ACTIVE` and ends in the observed `DISCONNECTED`, `EXITED` or `ERROR` state rather than remaining falsely active.
- [ ] `US-008-AC2`: Given a terminal is `DISCONNECTED`, when the Developer reconnects, then a new shell is opened and the old tab remains marked `DISCONNECTED`; the application does not promise to resume the previous remote shell.
- [ ] `US-008-AC3`: Given a terminal tab is closed, when its close workflow completes, then its owned session process, PTY and channel resources are released without closing unrelated tabs or destroying a still-used shared master.
- [ ] `US-008-AC4`: Given a shell exits normally or its session fails while the master remains usable, when session state updates, then the application does not equate that session outcome with failure of every other session on the Connection Identity.
- [ ] `US-014-AC1`: Given the relevant GTK connection, askpass or terminal surface, when it is used by keyboard, then available Connect, Disconnect, prompt confirmation/cancellation, terminal session and error-recovery controls can be reached and activated without a pointer.
- [ ] `US-014-AC2`: Given no Server, no open terminal or no connection has yet supplied data, when its surface is shown, then an appropriate empty or not-connected state is displayed rather than an indefinite spinner or invented connection result.
- [ ] `US-014-AC3`: Given connection, authentication, Capability detection or disconnection is in progress, when the Developer navigates or interacts with other surfaces, then GTK remains responsive and the affected workflow exposes its actual loading/progress state.
- [ ] `US-014-AC4`: Given an asynchronous result arrives, when visible widget state changes, then GTK owns presentation updates through channels scheduled to `glib::MainContext`; Tokio work never accesses GTK widgets directly.
- [ ] `US-014-AC5`: Given a prior observation survives a connection loss, when it is displayed, then its stale or disconnected status remains visible and an error does not turn old information into a current observation.
- [ ] This direct-only slice rejects unsupported routed admission until 04 rather than installing fake successful route callbacks; final control contract is completed by that private route refinement.
- [ ] A late/stale/cancelled continuation always returns created children to actual cleanup and can never revive Connected or an exited terminal.
- [ ] Spontaneous child exit before spawn reply, delayed PTY release and stale old-shell reports cannot satisfy another shell drain barrier.
- [ ] Agent, prompt cancellation, supported FIDO interaction, empty/loading/error states and keyboard access are real GTK/OpenSSH fixture journeys, not greeting/wiring/text assertions.
- [ ] Every invariant this ticket establishes has its Proof passing.
- [ ] Every invariant this ticket preserves still has its Proof passing.

## Verify

These are exact future implementation gates to add and run, not existing/passing tests or a check executed while drafting. Run from the project root after implementing this ticket. The workflow tests provision isolated Incus/LXD Ubuntu 24.04/systemd 255 fixtures with actual sshd, non-root users, system OpenSSH and actual SQLite; GTK tests run the real installed GTK/VTE surface in a desktop test session. Narrow arithmetic/clock cases remain pure boundary tests, not replacements for those journeys. Any disk-full/read-only/schema-failure fixture uses only an isolated container filesystem; never fill or damage the host disk.

```sh
cargo test --test transport_workflows transport_shared_identity_and_components -- --exact
cargo test --test transport_workflows transport_mux_loss_no_fallback -- --exact
cargo test --test transport_workflows transport_terminal_pty_drain -- --exact
cargo test --test transport_workflows transport_evidence_classification -- --exact
cargo test --test transport_workflows transport_vte_independent_context -- --exact
cargo test --test transport_workflows transport_admission_authentication_and_secret_exclusion -- --exact
cargo test --test transport_workflows transport_continuation_cancellation -- --exact
cargo test --test transport_workflows transport_terminal_early_exit_and_stale_report -- --exact
cargo test --test transport_workflows transport_orphan_cleanup -- --exact
cargo test --test transport_gtk transport_keyboard_loading_error_and_vte -- --exact --ignored
rustfmt --edition 2024 --check src/domain/connection.rs src/application/connection_manager.rs src/infrastructure/ssh/mod.rs src/ui/transport.rs tests/transport_workflows.rs tests/transport_gtk.rs tests/support/transport.rs
```

Any changed Cargo manifest/dependency wiring is exercised by these affected test targets; no permanent greeting, source-text or wiring-only tests. No throughput, capacity or latency benchmark result is claimed.

## Do not touch

- Public functions introduced by local tickets 01, 03, 04, 05; use them without adding duplicate APIs. Later private refinement of an earlier owned boundary is limited to this ticket's Goal.
- ADR-0002 inventory/import/generated policy, ADR-0003 trust eligibility/acceptance, ADR-0004 Verifier/Audit Trail/outcomes, ADR-0005 metric/backoff policy and ADR-0006 primary/admission/lease state remain with their canonical owners.
- Never add a second SSH library, arbitrary public launch/command API, privileged root shell, secret persistence, user-config runtime fallback, automatic mutation replay or orphan adoption.
- Preserve the original source specification and accepted ADRs; this is implementation of the appended Design, not a license to change product policy.
