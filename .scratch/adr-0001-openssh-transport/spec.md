# System OpenSSH Transport and Connection Lifecycle

Status: ready-for-agent
Source: ADR-0001; ADR-0002, ADR-0003, ADR-0004 and ADR-0006 at their integration boundaries; product specification v2.1 §§2.1–2.4, 3.1, 4, 5, 6.1, 7.1, 7.4, 9, 12, 14, 16 and 18; GLOSSARY.md.

## Problem Statement

A Developer managing personal Linux Servers needs terminals, structured administration, file access and monitoring to use the same authenticated connection without implementing a second SSH security stack.

A process exit alone cannot establish whether authentication failed, a Host Key changed, a network failed or an administrative change occurred.

Independent consumer connections would duplicate authentication prompts, weaken lifecycle ownership and leave sessions or master processes running after the Developer disconnects or the application terminates.

The application must distinguish a last observed Connection State from a Server's actual availability and from its Host Key Trust State.

## Solution

Use the system OpenSSH executable for every SSH transport consumer, with exactly one application-owned ControlMaster for each active Connection Identity.

Reuse OpenSSH's ssh-agent, ProxyJump and FIDO capabilities rather than rebuilding SSH inside the process.

Use the generated configuration policy defined by ADR-0002 and the Host Key gate defined by ADR-0003 before a master can be used.

Expose normal connection, authentication, consumer and disconnection lifecycles through responsive GTK workflows.

Keep terminal commands outside the Operation engine and Audit Trail, while routing structured administrative mutations through the Operation engine and its Verifier.

This epic owns transport and ordinary lifecycle behavior, not the complete Files, Services or Monitoring products.

## User Stories

### Connection identity and startup

1. **US-001: Share one master for one Connection Identity**

   - **Story:** As a Developer managing personal Linux Servers, I want equivalent inventory entries to share their underlying SSH connection without sharing unrelated authentication or routes, so that I can reuse one authenticated connection without crossing security contexts.

   - **Basis:** Derived: product §§4.3, 4.7 and 12.3; glossary Connection Identity and Server.

   - **Acceptance criteria:**
     - **US-001-AC1:** Given two Server entries with the same host, port, SSH user, complete Jump Server route and authentication context, when both are connected or used concurrently, then their consumers use exactly one ControlMaster for that Connection Identity.
     - **US-001-AC2:** Given entries differing in host, port, SSH user, Jump Server route or authentication context, when they connect, then they do not reuse a master belonging to the different Connection Identity.
     - **US-001-AC3:** Given an established Connection Identity, when terminal, SFTP, Operation and Collector consumers are opened, then all use system OpenSSH through that identity's master rather than independent library-based SSH transports or separately authenticated replacement connections.
     - **US-001-AC4:** Given inventory is saved without a connection attempt, when it is subsequently displayed, then saving has not required a remote connection.

2. **US-002: Connect explicitly or by the saved startup preference**

   - **Story:** As a Developer managing personal Linux Servers, I want to control which connections start and see their actual progress, so that I can avoid unintended logins and understand when a connection is usable.

   - **Basis:** Derived: product §§4.1, 4.7, 4.8 and 12.3.

   - **Acceptance criteria:**
     - **US-002-AC1:** Given a saved Server with the default `connect_on_startup=false`, when the application starts, then no master is started for that Server solely because it is in inventory.
     - **US-002-AC2:** Given the Developer activates Connect, when configuration and trust prerequisites permit connection, then the connection manager starts the Connection Identity's master and the UI exposes `CONNECTING` until the attempt has an observed result.
     - **US-002-AC3:** Given a Server with `connect_on_startup=true`, when startup safety processing owned by ADR-0006 is complete, then its connection attempt uses the same configuration, trust and authentication gates as explicit Connect.
     - **US-002-AC4:** Given a master starts, when its control socket is created, then it resides within `$XDG_RUNTIME_DIR/serverdesk/` with that runtime directory restricted to permission `0700`.
     - **US-002-AC5:** Given the UI claims `CONNECTED`, when the claim is checked against the actual master process and `ssh -O check`, then it reflects a usable master rather than the mere fact that process creation was requested.

   - **Dependencies:** US-001.

3. **US-003: Apply configuration, route and Host Key gates before connection**

   - **Story:** As a Developer managing personal Linux Servers, I want every connection to follow my registered route and the application's Host Key decisions, so that I do not reach an unintended endpoint or bypass my trust decisions.

   - **Basis:** Derived: ADR-0002 and ADR-0003; product §§4.4, 4.6, 4.8 and 4.10.

   - **Acceptance criteria:**
     - **US-003-AC1:** Given a master is about to start, when its transport is launched, then it uses `-F` with generated configuration for its Connection Identity and the generated Host Key material supplied by ADR-0003, not the user's runtime SSH configuration.
     - **US-003-AC2:** Given the Host Key gate reports a mismatch or a Host Key not accepted for connection under ADR-0003, when Connect is requested, then the connection is blocked with `HOST_KEY_MISMATCH` or `HOST_KEY_UNTRUSTED` respectively and no terminal or administrative channel is admitted through that attempt.
     - **US-003-AC3:** Given a target uses a Jump Server, when connection is attempted, then the route must be registered in inventory and the Jump Server must pass the trust gate before it is used.
     - **US-003-AC4:** Given an unregistered proxy route is detected, when the connection workflow reports failure, then it exposes `UnregisteredProxyRoute` and does not silently use an alternate route or the user's SSH configuration.
     - **US-003-AC5:** Given a trusted Jump Server and an accepted target Host Key, when the target connection succeeds, then actual terminal or structured traffic reaches that target through the registered Jump Server using OpenSSH ProxyJump behavior.
     - **US-003-AC6:** Given the connection changes state, when the Server is displayed, then the Connection State remains distinguishable from its separate Trust State and connection success does not promote Host Key trust.

   - **Dependencies:** US-002.

### Authentication and observed failures

4. **US-004: Authenticate through agent, askpass and supported keys without storing secrets**

   - **Story:** As a Developer managing personal Linux Servers, I want to use my existing agent or key authentication and answer necessary prompts once for a shared master, so that I can use established authentication without storing secrets in the application.

   - **Basis:** Derived: ADR-0001; product §§1.3, 4.4, 4.5, 13.2 and 16.2; glossary Auth Profile.

   - **Acceptance criteria:**
     - **US-004-AC1:** Given a working ssh-agent exposed by `SSH_AUTH_SOCK`, including a GNOME Keyring agent, when the selected Auth Profile can authenticate through it, then the system OpenSSH master authenticates using that agent.
     - **US-004-AC2:** Given agent authentication is unavailable or insufficient and authentication requires the askpass fallback, when OpenSSH invokes `SSH_ASKPASS`, then it invokes the ServerDesk executable's askpass mode and presents a GTK prompt rather than requiring an embedded SSH implementation.
     - **US-004-AC3:** Given a prompt is accepted and the master authenticates, when additional consumers attach to that master, then the application does not initiate separate authentication or repeat the master-authentication prompt for each consumer.
     - **US-004-AC4:** Given a prompt is cancelled or authentication is rejected, when the attempt ends, then the application does not claim `CONNECTED`, retains no entered secret and shows the observed outcome with an available connection action.
     - **US-004-AC5:** Given the selected key and local system OpenSSH support FIDO authentication, when that key is used, then authentication remains on the system OpenSSH path and its required user interaction is not replaced by a stored application secret.
     - **US-004-AC6:** Given an Auth Profile is saved or an authentication attempt is inspected in persisted state and diagnostic details, when authentication completes or fails, then only authentication type and key/agent references are retained, never password, passphrase or authentication secret; Secret Service is not used for MVP SSH authentication.

   - **Dependencies:** US-002.

5. **US-005: Report only evidence-supported Connection States**

   - **Story:** As a Developer managing personal Linux Servers, I want connection failures to tell me what was observed without falsely claiming the Server is down or a specific cause is certain, so that I can choose an appropriate recovery action without acting on a fabricated diagnosis.

   - **Basis:** Derived: ADR-0001; product §§3.1, 3.4, 4.6–4.8, 14 and 18.

   - **Acceptance criteria:**
     - **US-005-AC1:** Given a connection has never been attempted, when it appears in inventory or fleet summaries, then it is `NOT_CONNECTED` and is not counted as a failed or offline Server.
     - **US-005-AC2:** Given an attempt is in progress, when GTK handles unrelated navigation or input, then `CONNECTING` remains visible without blocking the GTK main thread.
     - **US-005-AC3:** Given independently established authentication failure, trust rejection, timeout or network unreachability, when the workflow reports its result, then it uses the corresponding `AUTH_FAILED`, `HOST_KEY_MISMATCH`, `HOST_KEY_UNTRUSTED`, `TIMEOUT` or `UNREACHABLE` Connection State rather than conflating those outcomes.
     - **US-005-AC4:** Given SSH exits with code 255 and no independent evidence establishes a specific cause, when the attempt is classified, then the result is `UNKNOWN`; stderr text and exit 255 alone do not establish authentication failure, Host Key mismatch, timeout or network unreachability.
     - **US-005-AC5:** Given a Host Key mismatch is reported, when its evidence is inspected, then it comes from the Host Key preflight defined by ADR-0003 rather than parsing SSH stderr.
     - **US-005-AC6:** Given any failed connection attempt, when the UI presents the result, then it identifies the affected Server or Connection Identity, what failed, whether changes may have occurred, available actions and applicable Operation details, without claiming the remote Server is dead.

   - **Dependencies:** US-003, US-004.

6. **US-006: Detect Capabilities at connection without assuming administrative permission**

   - **Story:** As a Developer managing personal Linux Servers, I want to know which transport-dependent features are actually available on the connected Server, so that I can use supported features and understand unavailable ones before attempting them.

   - **Basis:** Derived: product §§4.9, 6.1, 8.1 and 14; glossary Capability; ADR-0004 for granular privilege checks.

   - **Acceptance criteria:**
     - **US-006-AC1:** Given a connection is established, when Capability detection runs, then observations cover OS/distribution, systemd, `busctl --json`, the SFTP subsystem, `posix-rename@openssh.com`, and `sha256sum` rather than assuming their presence.
     - **US-006-AC2:** Given an administrative Operation is evaluated for availability, when its permission is checked, then the Operation engine's exact-command-and-arguments `sudo -n -l` result is used rather than a blanket claim that the SSH user has sudo access.
     - **US-006-AC3:** Given a Capability is absent or permission is denied, when the corresponding feature is offered, then it is unavailable with the observed reason, and the application neither invents a successful detection result nor disables unrelated supported consumers.
     - **US-006-AC4:** Given the Server does not provide a human-oriented command's desired output format, when service Capabilities are evaluated, then the accepted machine-readable read mechanisms are considered and Services is not disabled solely because `systemctl list-units` lacks JSON.

   - **Dependencies:** US-002.

### Real transport consumers

7. **US-007: Open independent interactive terminal sessions through the master**

   - **Story:** As a Developer managing personal Linux Servers, I want interactive shells that reuse authentication while keeping each terminal's input, output and remote PTY independent, so that I can work in multiple shells without session interference.

   - **Basis:** Derived: product §§2.3, 2.4, 5.1, 5.2 and 5.4.

   - **Acceptance criteria:**
     - **US-007-AC1:** Given a usable Connection Identity, when a terminal is opened, then VTE spawns system `ssh -F <generated> -t <identity>` and the terminal attaches through that identity's ControlMaster with a remote interactive PTY.
     - **US-007-AC2:** Given an active terminal, when `vim`, `top` or `htop` is used, then interactive input and terminal output work and resizing the terminal updates the remote PTY dimensions.
     - **US-007-AC3:** Given multiple terminal sessions, including sessions for different Servers, when they exchange input and output, then each stream belongs to its own session and output does not mix between sessions.
     - **US-007-AC4:** Given an active terminal, when the Developer switches pages or the global Server selection, then that terminal remains active with its original Server context rather than being closed or retargeted.
     - **US-007-AC5:** Given a command is typed in the terminal, when it runs, then it is not represented as a structured Operation and its command text is not added to the application's Audit Trail.

   - **Dependencies:** US-001, US-003, US-004.

8. **US-008: Close or reconnect terminal sessions truthfully**

   - **Story:** As a Developer managing personal Linux Servers, I want terminal state and resource ownership to remain clear when a shell exits or its connection is lost, so that I can distinguish a new shell from an interrupted one and release resources deliberately.

   - **Basis:** Derived: product §§5.3, 5.4, 12.3 and 14.

   - **Acceptance criteria:**
     - **US-008-AC1:** Given a terminal is opened, when its lifecycle advances, then the visible state follows `CREATED → CONNECTING → ACTIVE` and ends in the observed `DISCONNECTED`, `EXITED` or `ERROR` state rather than remaining falsely active.
     - **US-008-AC2:** Given a terminal is `DISCONNECTED`, when the Developer reconnects, then a new shell is opened and the old tab remains marked `DISCONNECTED`; the application does not promise to resume the previous remote shell.
     - **US-008-AC3:** Given a terminal tab is closed, when its close workflow completes, then its owned session process, PTY and channel resources are released without closing unrelated tabs or destroying a still-used shared master.
     - **US-008-AC4:** Given a shell exits normally or its session fails while the master remains usable, when session state updates, then the application does not equate that session outcome with failure of every other session on the Connection Identity.

   - **Dependencies:** US-007.

9. **US-009: Carry SFTP over a separate subsystem channel**

   - **Story:** As a Developer managing personal Linux Servers, I want file traffic to use the authenticated connection without corrupting operation-script input or other sessions, so that I can transfer files safely while other work continues.

   - **Basis:** Derived: ADR-0001; product §§6.1, 6.2, 6.6 and 16.2.

   - **Acceptance criteria:**
     - **US-009-AC1:** Given a connected Server with an available SFTP subsystem, when a file transfer is requested, then `openssh-sftp-client` uses the OpenSSH `ssh -s <identity> sftp` subsystem channel through the existing master.
     - **US-009-AC2:** Given SFTP payload traffic and a structured Operation run concurrently, when both transmit data, then file payload travels on its own channel and cannot become script stdin or terminal input.
     - **US-009-AC3:** Given the SSH user's file permissions deny access or the SFTP subsystem is absent, when file transport is attempted, then the affected workflow reports the permission or Capability limitation without acquiring a privileged shell or silently switching transport.
     - **US-009-AC4:** Given a transfer channel is lost before completion, when transport termination is delivered to the file workflow, then it is not reported as a completed transfer; file verification and any Unknown Outcome are decided by the file/Operation workflow.

   - **Dependencies:** US-001, US-006.

10. **US-010: Carry structured Operations without weakening their outcome rules**

- **Story:** As a Developer managing personal Linux Servers, I want structured administration to use the shared SSH transport while preserving verified results and granular privileges, so that I can administer the Server without mistaking transport completion for verified success.

- **Basis:** Derived: ADR-0001 and ADR-0004; product §§9, 14 and 16.3.

- **Acceptance criteria:**
  - **US-010-AC1:** Given an Operation is admitted by its engine, when remote execution starts, then its fixed POSIX script is sent over a system OpenSSH channel through the existing master, with machine-readable results rather than parsed human-oriented command output.
  - **US-010-AC2:** Given the remote Operation runs, when transport starts its shell, then it runs as the SSH login user and does not substitute a privileged root shell for ADR-0004's granular `sudo -n` commands.
  - **US-010-AC3:** Given a channel exits successfully, when the engine is informed, then the transport result alone does not mark the Operation `SUCCEEDED`; its operation-specific Verifier still determines the outcome.
  - **US-010-AC4:** Given SSH disconnects, the Server reboots or the channel is cancelled after a remote change may have started, when the engine is informed, then transport loss is not translated into proof of no remote effects, automatic retry or verified cancellation; ADR-0004 decides Unknown Outcome and ADR-0006 coordinates recovery.

- **Dependencies:** US-001.

11. **US-011: Carry a long-lived Collector without starving interactive work**

- **Story:** As a Developer managing personal Linux Servers, I want monitoring to share the connection without repeatedly authenticating or degrading terminal interaction, so that I can observe the Server without sacrificing interactive access.

- **Basis:** Derived: product §§4.7, 7.1, 7.4 and 12.3; ADR-0005.

- **Acceptance criteria:**
  - **US-011-AC1:** Given a connected Server whose monitoring is active, when collection runs, then its one long-lived Collector channel uses the identity's master and emits structured records from the fixed loop rather than creating a fresh SSH login for each sample.
  - **US-011-AC2:** Given monitoring is active while terminal and other consumers are used, when collection is scheduled, then the configured Collector concurrency bound is respected and polling does not starve interactive terminal work.
  - **US-011-AC3:** Given the Collector channel or master is lost, when monitoring receives transport termination, then the existing data becomes `STALE` under ADR-0005 instead of being replaced by invented zero values or a claim that the Server is down.
  - **US-011-AC4:** Given collection recovery is attempted, when a new Collector channel is opened, then it remains subject to the current master's configuration, trust and authentication lifecycle; metric backoff and sample reset rules remain owned by ADR-0005.

- **Dependencies:** US-001.

### Shutdown, loss and presentation

12. **US-012: Disconnect every consumer before closing its master**

- **Story:** As a Developer managing personal Linux Servers, I want Disconnect to end the entire shared connection rather than leaving background channels running, so that I can deliberately stop every consumer of that Connection Identity.

- **Basis:** Derived: product §§4.3, 4.7, 5.3, 7.4, 9.2 and 12.3.

- **Acceptance criteria:**
  - **US-012-AC1:** Given terminal, SFTP, Operation and Collector channels exist for a Connection Identity, when the Developer disconnects it, then the connection manager explicitly closes all descendant sessions before closing that master.
  - **US-012-AC2:** Given multiple Server entries share that Connection Identity, when its master is disconnected, then every affected entry and consumer reflects the shared connection loss rather than retaining a false connected presentation.
  - **US-012-AC3:** Given another Connection Identity remains active, when this identity is disconnected, then the other identity's master and sessions remain usable.
  - **US-012-AC4:** Given Disconnect interrupts a transfer or Operation, when its result is displayed, then it retains the owning workflow's distinction between known no effects and possible remote effects and provides the relevant details rather than reporting success or safe cancellation merely because local processes ended.
  - **US-012-AC5:** Given Disconnect completes, when process/channel ownership is inspected, then no descendant session remains owned by the disconnected master and no monitoring loop continues through that connection.

- **Dependencies:** US-008, US-009, US-010, US-011.

13. **US-013: Detect master loss and terminate application-owned transport**

- **Story:** As a Developer managing personal Linux Servers, I want the application to stop claiming a usable connection when its master is lost and to avoid orphaned transport after application termination, so that I can rely on visible connection state and avoid unmanaged background connections.

- **Basis:** Derived: product §§4.7, 7.4, 12.3, 16.3 and 18; ADR-0006.

- **Acceptance criteria:**
  - **US-013-AC1:** Given a connected master exits or fails `ssh -O check`, when the connection manager observes the loss, then Connection State and affected consumer states cease claiming an active usable connection using only evidence-supported classification.
  - **US-013-AC2:** Given a master is spawned, when its application parent dies, then the configured `PR_SET_PDEATHSIG` terminates that master rather than deliberately persisting it beyond the application.
  - **US-013-AC3:** Given a normal application exit with active connections, when shutdown finishes, then owned sessions and masters are closed and Collectors stop.
  - **US-013-AC4:** Given crash residue is encountered on a later startup, when ADR-0006 performs cleanup, then the connection manager participates in exiting old masters and removing old sockets and never adopts an orphan as a new live connection.
  - **US-013-AC5:** Given an interrupted Operation existed when transport or the application died, when the next connection becomes usable, then no transport-level replay occurs; Unknown Outcome and read-only Reconciliation are handled by ADR-0004 and ADR-0006.

- **Dependencies:** US-005, US-012.

14. **US-014: Keep connection workflows accessible and responsive**

- **Story:** As a Developer managing personal Linux Servers, I want to connect, authenticate, inspect errors and disconnect with the keyboard while the desktop remains responsive, so that I can manage connections accessibly even while remote work is slow or failing.

- **Basis:** Derived: product §§2.1, 12.2, 12.3, 14 and 18.

- **Acceptance criteria:**
  - **US-014-AC1:** Given the relevant GTK connection, askpass or terminal surface, when it is used by keyboard, then available Connect, Disconnect, prompt confirmation/cancellation, terminal session and error-recovery controls can be reached and activated without a pointer.
  - **US-014-AC2:** Given no Server, no open terminal or no connection has yet supplied data, when its surface is shown, then an appropriate empty or not-connected state is displayed rather than an indefinite spinner or invented connection result.
  - **US-014-AC3:** Given connection, authentication, Capability detection or disconnection is in progress, when the Developer navigates or interacts with other surfaces, then GTK remains responsive and the affected workflow exposes its actual loading/progress state.
  - **US-014-AC4:** Given an asynchronous result arrives, when visible widget state changes, then GTK owns presentation updates through channels scheduled to `glib::MainContext`; Tokio work never accesses GTK widgets directly.
  - **US-014-AC5:** Given a prior observation survives a connection loss, when it is displayed, then its stale or disconnected status remains visible and an error does not turn old information into a current observation.

- **Dependencies:** US-005.

## Implementation Decisions

- Adopt the system OpenSSH process model in ADR-0001, not russh, libssh2 or a second in-process SSH implementation.
- The application connection manager will be the sole owner of masters and SSH sessions.
- The Connection Identity will include host, port, user, the complete Jump Server route and authentication context.
- One active Connection Identity will have exactly one ControlMaster, shared by every transport consumer.
- OpenSSH configuration and Host Key artifacts will be supplied by ADR-0002 and ADR-0003 before master startup.
- OpenSSH process state and `ssh -O check` will establish master usability.
- Master startup will use the accepted parent-death signal and restricted runtime socket directory.
- Authentication will prefer ssh-agent, use the application's GTK askpass mode as fallback and preserve the system OpenSSH path for supported FIDO keys.
- Auth Profiles will retain references rather than secrets; MVP SSH authentication will not use Secret Service.
- VTE will spawn an OpenSSH PTY session; terminal commands will remain outside the Operation engine and Audit Trail.
- `openssh-sftp-client` will use the OpenSSH SFTP subsystem through a channel separate from operation-script stdin.
- Structured Operations will consume the transport under ADR-0004's fixed-script, machine-readable and granular-privilege contract.
- Collector channels will consume the transport under ADR-0005's lifecycle, concurrency and Freshness contract.
- Ordinary Disconnect will explicitly close descendants before the master; startup orphan cleanup and interrupted-operation recovery will integrate with ADR-0006.
- SSH stderr will not be a machine protocol for classifying Host Key or other specific connection failures.
- GTK will own presentation, application services workflow, the connection manager transport, and the relevant domain owner each remaining state; Tokio will communicate with GTK through the main context.

## Testing Decisions

The user approved the common test seam: presentation-facing application workflows, backed by actual system OpenSSH and SQLite on isolated Incus/LXD Ubuntu 24.04 fixtures with systemd 255, plus real GTK smoke coverage and narrow pure-behavior tests.

These are planned tests, not executed or passing tests; the repository does not yet contain the authored application harness.

- Exercise real connection workflows with two inventory aliases sharing an identity and with variations of each identity component; observe actual master reuse and separation.
- Use actual sshd, non-root users, a Jump Server and agent/askpass authentication; cover available FIDO authentication with compatible fixture hardware/provider rather than mocked authentication success.
- Observe the askpass GTK surface, keyboard cancellation, no repeated consumer prompts and absence of secrets from persisted state and reported diagnostic details.
- Test Host Key acceptance and mismatch at the real ADR-0003 gate, registered and unregistered routes, and runtime configuration isolation with ADR-0002.
- Induce authentication failure, controlled timeout/network failure and ambiguous SSH failure; assert only evidence-supported Connection State classifications, never a parser's guess from stderr.
- Exercise actual VTE shells with interactive programs, PTY resize, multiple isolated streams, page/Server changes, shell exit, tab close and reconnect to a new shell.
- Carry real SFTP bytes, a fixed structured Operation and structured Collector records through the shared master; observe payload isolation and independence from terminal streams.
- Inject channel/master loss and remote reboot during consumer work; assert transport termination reaches each owner without false completed transfers, false Operation success, fabricated metrics or automatic mutation replay.
- Disconnect an identity with all consumer kinds and verify descendant termination before master closure while another identity remains usable.
- Terminate the application parent in the isolated fixture; observe master death and the ADR-0006 cleanup boundary on next launch rather than adopting leftovers.
- Smoke-test real GTK loading, empty, error, stale/disconnected and keyboard states while network work is blocked or failing; observe continuing UI responsiveness.
- Use pure tests only for domain identity/state invariants and evidence-based classification where these can be tested without pretending to exercise OpenSSH.

The product's concurrent-master and first-connect performance numbers are benchmark targets, not guarantees or results established by this specification.

## Out of Scope

- Replacing system OpenSSH with an SSH library or rebuilding its authentication, FIDO or proxy protocol implementation.
- Selecting new public APIs, types, process-launch interfaces or transport protocols.
- Complete inventory/group management, file editing, terminal layout/search/scrollback design, Services UI or Monitoring UI beyond the consumer and lifecycle observations specified here.
- Host Key acquisition, Adoption, Verification, TOFU, replacement or SQLite trust-store rules owned by ADR-0003.
- Static configuration import and generated security policy owned by ADR-0002.
- Operation planning, confirmation, Verifier predicates, Audit Trail semantics and granular sudo authorization owned by ADR-0004.
- Metric calculation, intervals, retention and Collector backoff policy owned by ADR-0005.
- Single-instance activation, resource-lock arbitration and crash-recovery sequencing owned by ADR-0006.
- tmux/screen session resumption, password/passphrase storage and Secret Service SSH authentication.
- Agents installed on remote Servers, non-MVP platforms, hosting-provider workflows or fleet mutation features.

## Further Notes

The architecture is an accepted future-state design, not existing application functionality; the current executable's greeting is not evidence that any transport behavior works.

Cross-epic integration is by existing responsibility: ADR-0002 supplies isolated configuration, ADR-0003 decides whether Host Key use is permitted, ADR-0004 owns mutation outcomes, ADR-0005 owns monitoring samples, and ADR-0006 owns startup ownership/recovery.

The accepted sources do not define numerical connection timeouts, shutdown deadlines or failure-detection polling intervals; none are invented here.

Supported FIDO behavior depends on the actual system OpenSSH and key provider; this epic does not promise compatibility beyond those capabilities.

The exact trust-state membership permitted by the Host Key gate must remain consistent with the ADR-0003 specification, particularly explicit TOFU remaining `UNVERIFIED`; this epic does not invent a second trust policy.

There is no independent specification decision blocker for transport: the transport contract delegates connection permission to the ADR-0003 Host Key gate. Resolving that epic's trust-policy ambiguity is a prerequisite for integrated implementation and real connection tests, not an additional transport policy decision or evidence that the prerequisite is implemented.

# Design

## Usage

1. **Bootstrap and connect:** E6 calls `cleanup_orphans(&startup, &directory).await?`, reads `report.inspect_cleanup()`, and admits only after actual cleanup and durable E4 recovery. On GTK main thread `install_transport_ui(&application, &terminal_dock)` returns the opaque capability consumed by `ConnectionManager::spawn(ready, configuration, trust_gate, ui)`. Then `connections.control(ConnectionControl::Connect { server_id }).await?` and `connections.observe().await?` show the real attempt.
2. **Use terminals and typed consumers:** `connections.control(ConnectionControl::Terminal(TerminalControl::Open { server_id })).await?`; the private VTE adapter reports actual lifecycle through its bound sink. E4/E5 call `ports.consumers.open(ConsumerTarget::Server { server_id }, request).await?.into_consumer_parts()`; routed E3 discovery uses only its issued Generation.
3. **Disconnect and exit:** `connections.control(ConnectionControl::Disconnect { server_id }).await?`; runtime later calls `connections.quiesce().await?`. Each owner receives joined termination, not fabricated remote outcomes.

E1–E6 qualify foreign canonical owners, not existing code. Shared record/enum/handle declarations are introduced by ADR-0006:01; only the owning method tickets create runtime capabilities. GTK widget-bearing bootstrap is private adapter API. All authority fields remain private; immutable launch/reply DTOs are readable by the actual GTK adapter but no authority-accepting entry takes a DTO/view. StartReservation, CompiledConfiguration and positive start authority are never Clone. MasterGeneration may clone its issued immutable binding.

## Types

```rust
pub enum ConnectionState {
    NotConnected, Connecting, Connected, AuthFailed, HostKeyMismatch,
    HostKeyUntrusted, Timeout, Unreachable, Unknown,
}
pub enum CapabilityKind {
    OperatingSystem, Systemd, BusctlJson, Sftp, PosixRename, Sha256sum,
}
pub struct Capability {
    pub kind: CapabilityKind,
    pub availability: E6::Availability,
    pub observations: Vec<(String, String)>,
    pub observed_at: Instant,
}
pub struct Connection {
    pub identity: E2::ConnectionIdentity,
    pub server_ids: Vec<Uuid>,
    pub generation: Option<u64>,
    pub state: ConnectionState,
    pub capabilities: Vec<Capability>,
    pub observed_at: Instant,
    pub failure: Option<E6::Failure>,
}
pub enum TerminalState {
    Created, Connecting, Active, Disconnected, Exited, Error,
}
pub struct Terminal {
    pub id: Uuid,
    pub server_id: Uuid,
    pub identity: E2::ConnectionIdentity,
    pub master_generation: Option<u64>,
    pub state: TerminalState,
    pub exit_code: Option<i32>,
    pub failure: Option<E6::Failure>,
}
pub enum TerminalControl {
    Open { server_id: Uuid },
    Reconnect { terminal_id: Uuid },
    Close { terminal_id: Uuid },
}
pub enum ConnectionControl {
    Connect { server_id: Uuid },
    Disconnect { server_id: Uuid },
    Terminal(TerminalControl),
}
pub enum ConnectionResult {
    Connection(Connection),
    Terminal(Terminal),
    Reconnected { previous: Terminal, replacement: Terminal },
    TerminalClosed { terminal_id: Uuid },
}
pub struct ConnectionView {
    pub connections: Vec<Connection>,
    pub terminals: Vec<Terminal>,
}
pub struct ConnectionChanged {
    pub sequence: u64,
    pub connection: Option<Connection>,
    pub terminal: Option<Terminal>,
    pub closed_terminal_id: Option<Uuid>,
}
pub struct ConnectionManager { tx: mpsc::Sender<ConnectionMessage> }
pub struct ConsumerAccess { tx: mpsc::Sender<ConnectionMessage> }
pub struct RouteAccess { tx: mpsc::Sender<ConnectionMessage> }
pub struct ConnectionPorts {
    pub manager: ConnectionManager,
    pub consumers: ConsumerAccess,
    pub routes: RouteAccess,
}
pub(crate) struct MasterGeneration {
    identity: E2::ConnectionIdentity,
    number: u64,
}
pub(crate) struct StartReservation {
    master: MasterGeneration,
    start_id: Uuid,
    control_path: PathBuf,
}
#[derive(Clone, Copy)]
pub(crate) enum DispatchKnowledge { NotDispatched, Dispatched }
pub(crate) enum ChannelEndReason {
    NormalExit { code: i32 },
    LocalStop,
    MasterLost,
    SpawnFailed(E6::Failure),
    TransportFailed(E6::Failure),
}
pub(crate) struct ChannelEnd {
    consumer_id: Uuid,
    master: MasterGeneration,
    dispatch: DispatchKnowledge,
    reason: ChannelEndReason,
}
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
pub(crate) enum ConsumerEvent {
    ResourcesObserved(E4::ResourceObservation),
    ExecutionObserved(E4::ExecutionObservation),
    TransferProgressed(E4::TransferProgress),
    CollectionObserved(E5::RawObservation),
    KeysObserved(E3::KeyObservation),
    Ended(ChannelEnd),
}
pub(crate) struct Consumer {
    id: Uuid,
    master: MasterGeneration,
    events: mpsc::Receiver<ConsumerEvent>,
}
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

pub(crate) struct AuthenticationDialog {
    pub(crate) attempt_id: Uuid,
    pub(crate) challenge_id: Uuid,
    pub(crate) server_id: Uuid,
    pub(crate) identity: E2::ConnectionIdentity,
    pub(crate) prompt: String,
    pub(crate) answer: oneshot::Sender<AuthenticationAnswer>,
}
pub(crate) enum AuthenticationAnswer {
    Accepted(zeroize::Zeroizing<Vec<u8>>),
    Cancelled,
}
pub(crate) struct AskpassRequest {
    pub(crate) owner_socket: PathBuf,
    pub(crate) attempt_id: Uuid,
    pub(crate) challenge_id: Uuid,
    pub(crate) nonce: [u8; 32],
    pub(crate) prompt: String,
}
pub(crate) struct TerminalBinding {
    pub(crate) terminal_id: Uuid,
    pub(crate) shell_generation: Uuid,
    pub(crate) master: MasterGeneration,
}
pub(crate) enum TerminalExit {
    ExitCode(i32),
    Signal(i32),
    Unknown,
}
pub(crate) enum TerminalUiEvent {
    ChildExited(TerminalExit),
    PtyReleased,
    SpawnFailed(E6::Failure),
}
pub(crate) struct TerminalLifecycleReport {
    binding: TerminalBinding,
    observed_at: Instant,
    event: TerminalUiEvent,
}
pub(crate) struct TerminalLifecycleSink {
    binding: TerminalBinding,
    tx: mpsc::UnboundedSender<ConnectionMessage>,
}
pub(crate) struct PrivateTerminalLaunch {
    pub(crate) binding: TerminalBinding,
    pub(crate) executable: PathBuf,
    pub(crate) arguments: Vec<OsString>,
    pub(crate) environment: Vec<(OsString, OsString)>,
}
pub(crate) struct VteSpawnReport {
    pub(crate) binding: TerminalBinding,
    pub(crate) process_id: u32,
    pub(crate) process_group: u32,
}
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

pub(crate) struct TransportUi {
    authentication: mpsc::Sender<AuthenticationDialog>,
    terminals: mpsc::Sender<TerminalUiCommand>,
}
pub(crate) struct CleanupReport {
    startup_generation: Uuid,
    visited_sockets: Vec<PathBuf>,
    removed_sockets: Vec<PathBuf>,
    unresolved: Vec<E6::Failure>,
}

pub(crate) enum ConsumerTarget {
    Server { server_id: Uuid },
    Generation(MasterGeneration),
}
pub(crate) struct MasterGenerationView<'a> {
    pub(crate) identity: &'a E2::ConnectionIdentity,
    pub(crate) number: u64,
}
pub(crate) struct StartReservationView<'a> {
    pub(crate) master: &'a MasterGeneration,
    pub(crate) start_id: Uuid,
    pub(crate) control_path: &'a Path,
}
pub(crate) struct ConsumerParts {
    pub(crate) id: Uuid,
    pub(crate) master: MasterGeneration,
    pub(crate) events: mpsc::Receiver<ConsumerEvent>,
}
pub(crate) struct ChannelEndView<'a> {
    pub(crate) consumer_id: Uuid,
    pub(crate) master: &'a MasterGeneration,
    pub(crate) dispatch: DispatchKnowledge,
    pub(crate) reason: &'a ChannelEndReason,
}
pub(crate) struct CleanupReportView<'a> {
    pub(crate) startup_generation: Uuid,
    pub(crate) visited_sockets: &'a [PathBuf],
    pub(crate) removed_sockets: &'a [PathBuf],
    pub(crate) unresolved: &'a [E6::Failure],
}

pub(crate) struct TrustWork {
    attempt_id: Uuid,
    resolved: E2::ResolvedConnection,
    reservation: StartReservation,
    prefixes: Vec<MasterGeneration>,
    discovery: E3::DiscoveryPath,
}
pub(crate) struct CompilableStart {
    attempt_id: Uuid,
    resolved: E2::ResolvedConnection,
    reservation: StartReservation,
    prefixes: Vec<MasterGeneration>,
    trust: E3::TrustStart,
}
pub(crate) struct MasterSpawnObservation {
    master: MasterGeneration,
    start_id: Uuid,
    process: Option<tokio::process::Child>,
    control_check: Result<Instant, E6::Failure>,
}
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

impl ConnectionManager {
    pub(crate) fn spawn(
        ready: E6::ReadyOwnership,
        configuration: E2::Configuration,
        trust: E3::TrustGate,
        ui: TransportUi,
    ) -> ConnectionPorts;

    pub async fn control(
        &self, request: ConnectionControl,
    ) -> Result<ConnectionResult, E6::Failure>;

    pub async fn observe(
        &self,
    ) -> Result<E6::Subscription<ConnectionView, ConnectionChanged>, E6::Failure>;

    pub(crate) async fn quiesce(
        &self,
    ) -> Result<Vec<ChannelEnd>, E6::Failure>;
}
impl ConsumerAccess {
    pub(crate) async fn open(
        &self,
        target: ConsumerTarget,
        request: ConsumerRequest,
    ) -> Result<Consumer, E6::Failure>;

    pub(crate) async fn stop(
        &self, consumer_id: Uuid,
    ) -> Result<ChannelEnd, E6::Failure>;
}
impl RouteAccess {
    pub(crate) async fn prepare_discovery(
        &self, resolved: E2::ResolvedConnection,
    ) -> Result<E3::DiscoveryPath, E6::Failure>;
}
pub(crate) async fn cleanup_orphans(
    startup: &E6::StartupOwnership,
    directory: &E6::RuntimeDirectory,
) -> Result<CleanupReport, E6::Failure>;

pub(crate) fn serve_askpass(
    request: AskpassRequest,
) -> Result<i32, E6::Failure>;
impl MasterGeneration {
    pub(crate) fn inspect_generation(&self) -> MasterGenerationView<'_>;
}
impl StartReservation {
    pub(crate) fn inspect_reservation(&self) -> StartReservationView<'_>;
}
impl Consumer {
    pub(crate) fn into_consumer_parts(self) -> ConsumerParts;
}
impl ChannelEnd {
    pub(crate) fn inspect_channel_end(&self) -> ChannelEndView<'_>;
}
impl CleanupReport {
    pub(crate) fn inspect_cleanup(&self) -> CleanupReportView<'_>;
}
impl TerminalLifecycleSink {
    pub(crate) fn report(&self, event: TerminalUiEvent) -> Result<(), E6::Failure>;
}
impl TransportUi {
    pub(crate) fn into_transport_ui_parts(self) -> (mpsc::Sender<AuthenticationDialog>, mpsc::Sender<TerminalUiCommand>);
}
// Private GTK adapter bootstrap, not a public application-service API.
pub(crate) fn install_transport_ui(application: &adw::Application, terminal_dock: &adw::TabView) -> TransportUi;
```

## Contract

### Vocabulary

| Name                      | Kind   | Meaning                                                                | Source of truth                                                   | Writer                                  |
| ------------------------- | ------ | ---------------------------------------------------------------------- | ----------------------------------------------------------------- | --------------------------------------- |
| `ConnectionState`         | enum   | Evidence-supported last connection observation, not remote health      | `src/domain/connection.rs` `ConnectionState`                      | `none (immutable)`                      |
| `CapabilityKind`          | enum   | Closed transport feature detection subjects                            | `src/domain/connection.rs` `CapabilityKind`                       | `none (immutable)`                      |
| `Capability`              | value  | Dated actual feature availability and reasons                          | `src/domain/connection.rs` `Capability`                           | `none (immutable)`                      |
| `Connection`              | entity | Shared identity-bound connection lifecycle                             | `src/domain/connection.rs` `Connection`                           | `src/domain/connection.rs`              |
| `TerminalState`           | enum   | Independent interactive shell lifecycle                                | `src/domain/connection.rs` `TerminalState`                        | `none (immutable)`                      |
| `Terminal`                | entity | Original-Server-bound shell observation                                | `src/domain/connection.rs` `Terminal`                             | `src/domain/connection.rs`              |
| `TerminalControl`         | enum   | Explicit shell open/reconnect/close intent                             | `src/domain/connection.rs` `TerminalControl`                      | `none (immutable)`                      |
| `ConnectionControl`       | enum   | Explicit connection or terminal action                                 | `src/domain/connection.rs` `ConnectionControl`                    | `none (immutable)`                      |
| `ConnectionResult`        | enum   | Observed control result                                                | `src/domain/connection.rs` `ConnectionResult`                     | `none (immutable)`                      |
| `ConnectionView`          | value  | Current connection and terminal snapshot                               | `src/domain/connection.rs` `ConnectionView`                       | `none (immutable)`                      |
| `ConnectionChanged`       | event  | Sequenced connection or terminal observation change                    | `src/domain/connection.rs` `ConnectionChanged`                    | `src/domain/connection.rs`              |
| `ConnectionManager`       | entity | Sole master/session owner capability                                   | `src/application/connection_manager.rs` `ConnectionManager`       | `src/application/connection_manager.rs` |
| `ConsumerAccess`          | value  | Restricted existing-generation channel capability                      | `src/application/connection_manager.rs` `ConsumerAccess`          | `none (immutable)`                      |
| `RouteAccess`             | value  | Registered-prefix discovery capability                                 | `src/application/connection_manager.rs` `RouteAccess`             | `none (immutable)`                      |
| `ConnectionPorts`         | value  | Manager-issued facade handles                                          | `src/application/connection_manager.rs` `ConnectionPorts`         | `none (immutable)`                      |
| `MasterGeneration`        | value  | Manager-issued immutable identity/generation authority                 | `src/application/connection_manager.rs` `MasterGeneration`        | `src/application/connection_manager.rs` |
| `StartReservation`        | value  | Unique movable start authority with original control path              | `src/application/connection_manager.rs` `StartReservation`        | `src/application/connection_manager.rs` |
| `DispatchKnowledge`       | enum   | Whether bytes were dispatched, never remote effect proof               | `src/application/connection_manager.rs` `DispatchKnowledge`       | `none (immutable)`                      |
| `ChannelEndReason`        | enum   | Observed joined channel termination cause                              | `src/application/connection_manager.rs` `ChannelEndReason`        | `none (immutable)`                      |
| `ChannelEnd`              | value  | Manager-issued actual joined termination evidence                      | `src/application/connection_manager.rs` `ChannelEnd`              | `src/application/connection_manager.rs` |
| `ConsumerRequest`         | enum   | Closed typed transport request without arbitrary commands              | `src/application/connection_manager.rs` `ConsumerRequest`         | `none (immutable)`                      |
| `ConsumerEvent`           | enum   | Closed typed resource/operation/transfer/collection/key observations   | `src/application/connection_manager.rs` `ConsumerEvent`           | `none (immutable)`                      |
| `Consumer`                | entity | Registered channel with unique event receiver                          | `src/application/connection_manager.rs` `Consumer`                | `src/application/connection_manager.rs` |
| `ConnectionMessage`       | enum   | Private owner command and worker-completion protocol                   | `src/application/connection_manager.rs` `ConnectionMessage`       | `none (immutable)`                      |
| `AuthenticationDialog`    | value  | One ephemeral prompt with one answer endpoint                          | `src/ui/transport.rs` `AuthenticationDialog`                      | `none (immutable)`                      |
| `AuthenticationAnswer`    | enum   | Zeroized accepted secret or cancellation                               | `src/ui/transport.rs` `AuthenticationAnswer`                      | `none (immutable)`                      |
| `AskpassRequest`          | value  | Attempt/challenge/nonce-bound helper invocation                        | `src/infrastructure/ssh/mod.rs` `AskpassRequest`                  | `none (immutable)`                      |
| `TerminalBinding`         | value  | Immutable terminal/shell/master generation binding                     | `src/ui/transport.rs` `TerminalBinding`                           | `none (immutable)`                      |
| `TerminalExit`            | enum   | Observed child exit code, signal or uncertainty                        | `src/ui/transport.rs` `TerminalExit`                              | `none (immutable)`                      |
| `TerminalUiEvent`         | enum   | Actual child exit, PTY release or spawn failure                        | `src/ui/transport.rs` `TerminalUiEvent`                           | `none (immutable)`                      |
| `TerminalLifecycleReport` | event  | Locally timed bound GTK lifecycle observation                          | `src/application/connection_manager.rs` `TerminalLifecycleReport` | `src/application/connection_manager.rs` |
| `TerminalLifecycleSink`   | value  | Privately issued binding-stamped lifecycle sender                      | `src/application/connection_manager.rs` `TerminalLifecycleSink`   | `src/application/connection_manager.rs` |
| `PrivateTerminalLaunch`   | value  | Manager-produced private GTK spawn DTO, not launch authority           | `src/ui/transport.rs` `PrivateTerminalLaunch`                     | `none (immutable)`                      |
| `VteSpawnReport`          | value  | Actual bound VTE child/process-group acknowledgement                   | `src/ui/transport.rs` `VteSpawnReport`                            | `none (immutable)`                      |
| `TerminalUiCommand`       | enum   | Private prompt-independent spawn/close channel protocol                | `src/ui/transport.rs` `TerminalUiCommand`                         | `none (immutable)`                      |
| `TransportUi`             | value  | Opaque installed GTK endpoint capability                               | `src/ui/transport.rs` `TransportUi`                               | `src/ui/transport.rs`                   |
| `CleanupReport`           | value  | Actual prior-primary transport cleanup facts                           | `src/infrastructure/ssh/mod.rs` `CleanupReport`                   | `none (immutable)`                      |
| `ConsumerTarget`          | enum   | Existing Server binding or manager-issued generation selector          | `src/application/connection_manager.rs` `ConsumerTarget`          | `none (immutable)`                      |
| `MasterGenerationView`    | value  | Borrowed immutable generation projection, not authority                | `src/application/connection_manager.rs` `MasterGenerationView`    | `none (immutable)`                      |
| `StartReservationView`    | value  | Borrowed exact original reservation projection                         | `src/application/connection_manager.rs` `StartReservationView`    | `none (immutable)`                      |
| `ConsumerParts`           | value  | Moved channel identity/master and unique typed event receiver          | `src/application/connection_manager.rs` `ConsumerParts`           | `none (immutable)`                      |
| `ChannelEndView`          | value  | Borrowed dispatch and joined-termination facts                         | `src/application/connection_manager.rs` `ChannelEndView`          | `none (immutable)`                      |
| `CleanupReportView`       | value  | Borrowed startup cleanup evidence                                      | `src/infrastructure/ssh/mod.rs` `CleanupReportView`               | `none (immutable)`                      |
| `TrustWork`               | value  | Owned route-completed inputs for trust worker                          | `src/application/connection_manager.rs` `TrustWork`               | `none (immutable)`                      |
| `CompilableStart`         | value  | Original reservation returned with accepted trust for compiler         | `src/application/connection_manager.rs` `CompilableStart`         | `none (immutable)`                      |
| `MasterSpawnObservation`  | value  | Actual child plus control-check outcome moved to owner                 | `src/application/connection_manager.rs` `MasterSpawnObservation`  | `none (immutable)`                      |
| `ConnectCompletion`       | enum   | Owned resolution/route/trust/compilation/spawn results                 | `src/application/connection_manager.rs` `ConnectCompletion`       | `none (immutable)`                      |
| `ReadyOwnership`          | value  | Current admitted primary capability                                    | `src/application/runtime.rs` `ReadyOwnership`                     | `src/application/runtime.rs`            |
| `StartupOwnership`        | value  | Current preadmission primary capability                                | `src/application/runtime.rs` `StartupOwnership`                   | `src/application/runtime.rs`            |
| `RuntimeDirectory`        | value  | Validated confined runtime directory                                   | `src/application/runtime.rs` `RuntimeDirectory`                   | `src/application/runtime.rs`            |
| `Failure`                 | value  | Structured actionable failure                                          | `src/domain/error.rs` `Failure`                                   | `none (immutable)`                      |
| `Configuration`           | entity | Saved inventory and compiler facade                                    | `src/application/configuration.rs` `Configuration`                | `src/application/configuration.rs`      |
| `ConnectionIdentity`      | value  | Complete structural target/route/authentication identity               | `src/domain/connection_identity.rs` `ConnectionIdentity`          | `none (immutable)`                      |
| `ResolvedConnection`      | value  | Validated inventory resolution                                         | `src/domain/connection_identity.rs` `ResolvedConnection`          | `none (immutable)`                      |
| `CompiledConfiguration`   | value  | Original reservation and trust-bound sealed compilation                | `src/application/configuration.rs` `CompiledConfiguration`        | `src/application/configuration.rs`      |
| `TrustGate`               | value  | Canonical trust-start gate handle                                      | `src/application/trust.rs` `TrustGate`                            | `none (immutable)`                      |
| `PreparedStart`           | value  | Original resolved/reservation values returned with trust               | `src/application/trust.rs` `PreparedStart`                        | `src/application/trust.rs`              |
| `TrustStart`              | value  | Canonical positive start/snapshot binding                              | `src/application/trust.rs` `TrustStart`                           | `src/application/trust.rs`              |
| `DiscoveryPath`           | enum   | Local or trusted-prefix discovery selection                            | `src/application/trust.rs` `DiscoveryPath`                        | `none (immutable)`                      |
| `ServiceRead`             | enum   | Closed structured read-only service selection                          | `src/domain/resource_observation.rs` `ServiceRead`                | `none (immutable)`                      |
| `RemoteDispatch`          | value  | Sealed fixed-script execution admission                                | `src/application/operations.rs` `RemoteDispatch`                  | `src/application/operations.rs`         |
| `ExecutionPermit`         | value  | E4-issued immutable plan/write-ahead/lease/expected-identity admission | `src/application/operations.rs` `ExecutionPermit`                 | `src/application/operations.rs`         |
| `ExecutionPermitView`     | value  | Borrowed exact admission including expected_identity                   | `src/application/operations.rs` `ExecutionPermitView`             | `none (immutable)`                      |
| `RemoteDispatchView`      | value  | Borrowed fixed-script request, preflight and sealed permit             | `src/application/operations.rs` `RemoteDispatchView`              | `none (immutable)`                      |
| `TransferDispatchView`    | value  | Borrowed separate upload/write selection and sealed permit             | `src/application/operations.rs` `TransferDispatchView`            | `none (immutable)`                      |
| `DownloadDispatchView`    | value  | Borrowed read-only download binding including expected_identity        | `src/application/operations.rs` `DownloadDispatchView`            | `none (immutable)`                      |
| `ReconciliationReadView`  | value  | Borrowed original Operation identity and evidence policy               | `src/application/operations.rs` `ReconciliationReadView`          | `none (immutable)`                      |
| `TransferDispatch`        | value  | Sealed separate upload/write admission                                 | `src/application/operations.rs` `TransferDispatch`                | `src/application/operations.rs`         |
| `DownloadDispatch`        | value  | Sealed read-only download admission                                    | `src/application/operations.rs` `DownloadDispatch`                | `src/application/operations.rs`         |
| `ReconciliationRead`      | value  | Identity-bound read-only evidence request                              | `src/application/operations.rs` `ReconciliationRead`              | `src/application/operations.rs`         |
| `CollectorSpec`           | value  | Fixed-loop schedule and Server/generation selection                    | `src/application/monitoring.rs` `CollectorSpec`                   | `none (immutable)`                      |
| `RawObservation`          | value  | Actual typed collection evidence                                       | `src/infrastructure/collector/mod.rs` `RawObservation`            | `none (immutable)`                      |
| `TextOpenRequest`         | value  | Bounded read-only text-open request                                    | `src/domain/resource_observation.rs` `TextOpenRequest`            | `none (immutable)`                      |
| `ResourceObservation`     | enum   | Actual typed service/text resource observations                        | `src/domain/resource_observation.rs` `ResourceObservation`        | `none (immutable)`                      |
| `ExecutionObservation`    | value  | Actual fixed-script execution evidence, not outcome                    | `src/application/operations.rs` `ExecutionObservation`            | `none (immutable)`                      |
| `TransferProgress`        | value  | Actual separate-payload transfer lifecycle evidence                    | `src/domain/operation.rs` `TransferProgress`                      | `none (immutable)`                      |
| `ScanRequest`             | value  | Read-only canonical endpoint key discovery input                       | `src/application/trust.rs` `ScanRequest`                          | `none (immutable)`                      |
| `KeyObservation`          | value  | Actual discovered key material, never accepted trust                   | `src/application/trust.rs` `KeyObservation`                       | `none (immutable)`                      |
| `Availability`            | enum   | Observed feature availability or exact blocking reason                 | `src/domain/error.rs` `Availability`                              | `none (immutable)`                      |
| `Reply`                   | value  | Private asynchronous result endpoint                                   | `src/domain/error.rs` `Reply`                                     | `none (immutable)`                      |
| `Subscription`            | value  | Snapshot and sequenced observation receiver                            | `src/domain/error.rs` `Subscription`                              | `none (immutable)`                      |

### Functions

### `cleanup_orphans(startup: &E6::StartupOwnership, directory: &E6::RuntimeDirectory) -> Result<CleanupReport, E6::Failure>`

- **Intent.** Establish actual old-transport cleanup before primary admission.
- **Owner.** `src/infrastructure/ssh/mod.rs`
- **Reads.** StartupOwnership, RuntimeDirectory, actual confined sockets.
- **Writes.** CleanupReport and old application-owned sockets.
- **Preconditions.** Current startup primary; directory UID, 0700 permissions and entry confinement validate.
- **Postconditions.** Enumerates actual residue, sends isolated system ssh -O exit to each application socket, waits for confirmed termination or conclusive staleness, removes only those sockets; never adopts a master.
- **Invariants.** INV-11.
- **Errors.** Unsafe entry, uncertain live ownership, failed exit or removal remains unresolved and prevents admission; unrelated sockets remain untouched.
- **Idempotency.** Repeated cleanup of absent/stale entries is harmless; crash requires a fresh actual scan.

### `CleanupReport::inspect_cleanup(&self) -> CleanupReportView<'_>`

- **Intent.** Expose actual cleanup facts to Runtime without granting transport authority.
- **Owner.** `src/infrastructure/ssh/mod.rs`
- **Reads.** CleanupReport.
- **Writes.** None.
- **Preconditions.** Report returned by cleanup_orphans.
- **Postconditions.** Borrows startup generation, visited/removed sockets and unresolved failures unchanged.
- **Invariants.** INV-11.
- **Errors.** Infallible projection; no filesystem work.
- **Idempotency.** Repeatable read; no admission or cleanup is implied.

### `ConnectionManager::spawn(ready: E6::ReadyOwnership, configuration: E2::Configuration, trust: E3::TrustGate, ui: TransportUi) -> ConnectionPorts`

- **Intent.** Give admitted workflows one transport lifecycle owner.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** ReadyOwnership, Configuration, TrustGate, TransportUi.
- **Writes.** ConnectionManager, ConnectionPorts.
- **Preconditions.** ReadyOwnership is from current primary after actual cleanup and durable recovery; one composition call.
- **Postconditions.** Consumes TransportUi via into_transport_ui_parts; installs one owner loop with separate command/lifecycle receivers, identity registry and pending workers; starts no master.
- **Invariants.** INV-1, INV-6, INV-7.
- **Errors.** Construction has no operational result; later worker failures are reported through owned requests/events.
- **Idempotency.** Only one manager is composed per admitted primary; handles may be cloned, not owner construction.

### `ConnectionManager::control(&self, request: ConnectionControl) -> Result<ConnectionResult, E6::Failure>`

- **Intent.** Apply explicit connection or terminal intent while retaining the original target.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** ConnectionControl, Connection, Terminal, E2::ResolvedConnection, E3::PreparedStart, E2::CompiledConfiguration.
- **Writes.** Connection, Terminal, MasterGeneration, StartReservation.
- **Preconditions.** Admission open; Server/terminal exists; terminal Open requires usable generation; Reconnect names disconnected shell.
- **Postconditions.** Resolution completion reserves/coalesces structural identity; owned route, trust and compile workers return original reservation through PreparedStart::into_prepared_start before sealed launch. Every completion validates attempt/start/generation. Created children are returned even on failed control checks and registered for cleanup. Connected requires live process and ssh -O check. Terminal uses fresh shell binding; exit/failure and PTY facts govern state/drain, not late spawn acknowledgement. Disconnect denies new children and joins descendants before closing master.
- **Invariants.** INV-1, INV-2, INV-3, INV-4, INV-5, INV-6, INV-7, INV-8.
- **Errors.** Configuration/storage/trust/route errors prevent start; unknown 255/stderr remains Unknown; cancellation never publishes Connected. Stale completions cannot revive state; returned children remain owned until joined. Incomplete cleanup never reports completed disconnect.
- **Idempotency.** Concurrent same-identity Connect joins one reservation; connected Connect reuses generation; repeated Close/Disconnect joins cleanup; each explicit Open/Reconnect creates a new shell, never resumes old shell or replays mutations.

### `ConnectionManager::observe(&self) -> Result<E6::Subscription<ConnectionView, ConnectionChanged>, E6::Failure>`

- **Intent.** Observe current connection and terminal truth without initiating transport.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** Connection, Terminal, E2 inventory.
- **Writes.** None.
- **Preconditions.** Manager exists in current primary.
- **Postconditions.** Snapshot plus sequenced changes starts at one owner observation boundary; never-attempted entries are NotConnected; trust remains separate; lag requires actual resubscription.
- **Invariants.** INV-1, INV-4, INV-5, INV-6.
- **Errors.** Closed owner or inventory/storage failure yields explicit failure, not a connection attempt or empty success.
- **Idempotency.** Resubscription returns current observations without new authentication, shell, or changed timestamps.

### `MasterGeneration::inspect_generation(&self) -> MasterGenerationView<'_>`

- **Intent.** Read an issued generation identity without issuing another authority.
- **Owner.** `src/application/connection_manager.rs`
- **Introducing ticket.** ADR-0001:01 supplies this pure getter before configuration/trust consumers; ADR-0001:02 alone issues actual runtime generations.
- **Reads.** MasterGeneration.
- **Writes.** None.
- **Preconditions.** Manager-issued immutable generation.
- **Postconditions.** Borrows exact identity and number; view cannot be accepted as launch or consumer authority.
- **Invariants.** INV-1, INV-2.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable; cloning issued generation never creates a new master.

### `StartReservation::inspect_reservation(&self) -> StartReservationView<'_>`

- **Intent.** Let configuration and trust inspect the original unique start binding.
- **Owner.** `src/application/connection_manager.rs`
- **Introducing ticket.** ADR-0001:01 supplies this pure getter before configuration/trust consumers; ADR-0001:02 alone issues actual runtime reservations.
- **Reads.** StartReservation.
- **Writes.** None.
- **Preconditions.** Owner-created live reservation.
- **Postconditions.** Borrows exact master/start ID/control path; original reservation moves through trust and compilation, is never reconstructed from view.
- **Invariants.** INV-1, INV-6, INV-7.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeated reads do not duplicate or consume reservation; no Clone for reservation.

### `serve_askpass(request: AskpassRequest) -> Result<i32, E6::Failure>`

- **Intent.** Relay one authorized OpenSSH challenge to the existing primary GTK prompt.
- **Owner.** `src/infrastructure/ssh/mod.rs`
- **Reads.** AskpassRequest, AuthenticationDialog, AuthenticationAnswer.
- **Writes.** Ephemeral IPC only.
- **Preconditions.** IPC UID/permissions, nonce, attempt and challenge match active request.
- **Postconditions.** Helper starts no primary or Store; accepted bytes are sent once to requesting OpenSSH stdout and zeroized, cancellation returns non-success; main-thread prompt remains responsive.
- **Invariants.** INV-5, INV-6, INV-7.
- **Errors.** Stale/replayed/invalid challenge, owner loss, prompt cancellation or IPC failure releases secret buffers and returns no secret; no diagnostic contains answer.
- **Idempotency.** Challenge is single-use, no cached answers or per-consumer authentication.

### `TerminalLifecycleSink::report(&self, event: TerminalUiEvent) -> Result<(), E6::Failure>`

- **Intent.** Deliver actual child and PTY observations for the sink-bound shell.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** TerminalBinding, TerminalUiEvent.
- **Writes.** TerminalLifecycleReport only.
- **Preconditions.** Installed GTK adapter reports observed event using privately issued sink.
- **Postconditions.** Stamps original terminal/shell/master plus local time and enqueues without blocking GTK; only manager mutates Terminal.
- **Invariants.** INV-3, INV-5, INV-6, INV-8.
- **Errors.** Closed owner is actionable failure; missing report never counts as child/PTY release.
- **Idempotency.** Duplicate reports set existing per-shell flags; old-shell report cannot satisfy replacement barrier.

### `install_transport_ui(application: &adw::Application, terminal_dock: &adw::TabView) -> TransportUi`

- **Intent.** Install real GTK authentication and VTE endpoints for transport.
- **Owner.** `src/ui/transport.rs`
- **Reads.** Primary GTK application, terminal dock.
- **Writes.** TransportUi and GTK adapter state.
- **Preconditions.** GTK main thread; one installation per primary/dock.
- **Postconditions.** Returns opaque privately constructed channel capability; adapters consume readable launch DTO, spawn actual VTE child, report spontaneous child exit, spawn failure and PTY disposal through bound sink. No Store, manager or SSH master starts during installation.
- **Invariants.** INV-3, INV-5, INV-6, INV-8.
- **Errors.** Later prompt/spawn failures return through declared channels; installation itself does no asynchronous transport.
- **Idempotency.** Install once and reuse capability; ordinary presentation never constructs sender capabilities.

### `TransportUi::into_transport_ui_parts(self) -> (mpsc::Sender<AuthenticationDialog>, mpsc::Sender<TerminalUiCommand>)`

- **Intent.** Move the installed adapter capability into manager-owned prompt and terminal endpoints.
- **Owner.** `src/ui/transport.rs`
- **Reads.** TransportUi.
- **Writes.** None.
- **Preconditions.** Capability was issued by install_transport_ui and is not previously consumed.
- **Postconditions.** Returns its exact two sender endpoints; consumes wrapper; no fabricated channel or GTK widget crosses to Tokio.
- **Invariants.** INV-5, INV-6.
- **Errors.** Infallible consuming decomposition.
- **Idempotency.** One consuming call; endpoints may be retained by sole manager, not reconstructed from launch DTOs.

### `ConsumerAccess::open(&self, target: ConsumerTarget, request: ConsumerRequest) -> Result<Consumer, E6::Failure>`

- **Intent.** Attach one typed consumer to an already-connected selected generation.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** ConsumerTarget, ConsumerRequest, current Connection and Server binding, E4 borrowed dispatch/permit/read views.
- **Writes.** Consumer.
- **Preconditions.** Existing generation Connected and admits children; relevant Capability available; sealed dispatch expected identity matches current binding.
- **Postconditions.** Server lookup or issued-generation validation, sealed expected_identity comparison and child reservation are atomic in one owner turn; starts only fail-closed attachment channel. Execute/Transfer read the respective dispatch's permit through inspect_execution_permit and compare its borrowed expected_identity; Download compares inspect_download_dispatch.expected_identity; Reconcile compares the original Operation connection_identity and fails if absent or different. These views are never admission inputs. Services/text use E4 records, operations fixed POSIX login-user script, transfers separate SFTP payload and downloads read-only admission, Collector fixed structured loop; E1 stamps each RawObservation collector_generation with the actual channel ConsumerParts.id and collected_at with local receipt Instant, never remote time. Exactly one typed Ended follows actual join; no independent login.
- **Invariants.** INV-1, INV-2, INV-4, INV-6, INV-7, INV-9.
- **Errors.** Disconnected/missing/stale/mismatched target, unavailable Capability, denied access or spawn failure yields no alternate transport, Connect, authentication or queue-for-connect. Transport exit is never mutation verification/transfer completion.
- **Idempotency.** Each accepted call opens one channel; ambiguous delivery is not automatically retried. Collector owner explicitly decides allowed replacement channel.

### `ConsumerAccess::stop(&self, consumer_id: Uuid) -> Result<ChannelEnd, E6::Failure>`

- **Intent.** Obtain actual joined termination for one owned typed channel.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** Consumer.
- **Writes.** Consumer, ChannelEnd.
- **Preconditions.** ID is registered or completed result retained.
- **Postconditions.** Releases owned process/channel resources and returns generation-bound dispatch and termination facts; no claim about remote rollback.
- **Invariants.** INV-2, INV-6, INV-9.
- **Errors.** Unknown ID changes nothing; failed cleanup retains ownership and returns actionable failure.
- **Idempotency.** Repeated requests join the same stop or return retained result, never reopen.

### `Consumer::into_consumer_parts(self) -> ConsumerParts`

- **Intent.** Transfer the unique typed event receiver with its consumer identity.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** Consumer.
- **Writes.** None.
- **Preconditions.** Manager returned Consumer; receiver not previously moved.
- **Postconditions.** Moves exact ID, issued master and unique receiver to E3/E4/E5 owner; no copy of live receiver or authority.
- **Invariants.** INV-2, INV-9.
- **Errors.** Infallible consuming decomposition.
- **Idempotency.** One consuming call; stream closure alone is not proof of joined termination.

### `ChannelEnd::inspect_channel_end(&self) -> ChannelEndView<'_>`

- **Intent.** Expose joined channel termination without granting execution or outcome authority.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** ChannelEnd.
- **Writes.** None.
- **Preconditions.** Manager-issued actual joined result.
- **Postconditions.** Borrows consumer/master/dispatch/reason unchanged; Dispatched says nothing about remote effects.
- **Invariants.** INV-2, INV-9.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable read; does not classify Operation or transfer outcome.

### `RouteAccess::prepare_discovery(&self, resolved: E2::ResolvedConnection) -> Result<E3::DiscoveryPath, E6::Failure>`

- **Intent.** Provide discovery through real registered and gated Jump prefixes.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** E2::ResolvedConnection, current prefix MasterGeneration.
- **Writes.** Connection for prerequisite prefixes.
- **Preconditions.** Route finite and registered in inventory.
- **Postconditions.** Direct target returns Local; routed target connects strictly shorter prefixes using ordinary control workers, returns last already-gated Jump generation; target trust is never inherited. Worker waits while manager continues servicing KeyScan/stop/askpass/completions.
- **Invariants.** INV-1, INV-6, INV-7, INV-12.
- **Errors.** Unregistered route, missing prefix trust, failed/lost prefix blocks discovery; never scans routed target locally or uses external alias.
- **Idempotency.** Reuses current prefix generations; repeatable read-only preparation does not accept target key.

### `ConnectionManager::quiesce(&self) -> Result<Vec<ChannelEnd>, E6::Failure>`

- **Intent.** Join all owned transport before runtime closes storage and exits.
- **Owner.** `src/application/connection_manager.rs`
- **Reads.** Connection, Terminal, Consumer, pending continuations.
- **Writes.** Connection, Terminal, Consumer, ChannelEnd.
- **Preconditions.** Runtime closes new admission and has notified/quiesced E4/E5.
- **Postconditions.** Stored drain waiters and stop/join workers leave mailbox alive; all descendants including late-returned child, VTE exit/reap and PTY release are joined before master closes; aliases cease Connected; no Collector remains.
- **Invariants.** INV-1, INV-2, INV-3, INV-6, INV-7, INV-8, INV-9, INV-10, INV-12.
- **Errors.** Unjoined resource or failed cleanup keeps ownership and reports incomplete shutdown; no success from cancellation alone.
- **Idempotency.** Concurrent calls join one global drain; completed call is no-op; no replay/reconnect after stopping.

### Invariants

Planned implementation proofs below have not been run.

| ID     | Statement                                                                                                                  | Scope                                      | Enforced by                                                                                                  | Broken if                                                                        | Proof                                                                 | First proof owner |
| ------ | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------- |
| INV-1  | At most one starting/live master exists for each structurally equal Connection Identity.                                   | ConnectionManager, Connection              | Owner-loop identity reservation before launching any worker                                                  | Concurrent aliases reserve separately                                            | Future test `transport_shared_identity_and_components`                | 02                |
| INV-2  | An attachment uses only its specified current master generation or fails without authentication.                           | MasterGeneration, Terminal, Consumer       | Private compiled launch plus ControlMaster=no, BatchMode, ProxyJump=none and deny-fallback attachment policy | Mux loss opens a new login                                                       | Future test `transport_mux_loss_no_fallback`                          | 02                |
| INV-3  | Closing a terminal requires actual child termination/reaping and PTY release for that shell.                               | Terminal                                   | Bound sink plus per-shell exit/release flags                                                                 | Close acknowledgement substitutes for resource join                              | Future test `transport_terminal_pty_drain`                            | 02                |
| INV-4  | Each specific Connection State and Capability has independent supporting evidence; 255/stderr alone is Unknown.            | Connection, Capability                     | Closed evidence classifier; live child plus ssh -O check for Connected                                       | Human stderr becomes a classifier or Capability assumed                          | Future test `transport_evidence_classification`                       | 02                |
| INV-5  | Every terminal has independent streams/PTY and immutable original Server context.                                          | Terminal                                   | Terminal ID plus shell/master binding; GTK-only VTE objects                                                  | Global selection retargets a shell                                               | Future test `transport_vte_independent_context`                       | 02                |
| INV-6  | Master/authentication start requires current primary admission, generated configuration, trust and GTK-owned presentation. | ConnectionManager, Connection, TransportUi | Private authority fields and wrapper-only launch worker; real askpass IPC                                    | Caller fabricates launch path or helper becomes primary                          | Future test `transport_admission_authentication_and_secret_exclusion` | 02                |
| INV-7  | Waiting route/trust/configuration/launch work never prevents servicing dependent manager messages.                         | ConnectionManager                          | Owned continuation inputs; pending waiters; owner-loop-only mutation                                         | Connect handler awaits trust while withholding mailbox                           | Future test `transport_continuation_cancellation`                     | 02                |
| INV-8  | Terminal state/drain uses exact shell/master binding and actual spontaneous child/PTY observations.                        | Terminal, TerminalLifecycleReport          | Privately issued sink and generation-validated independent completion flags                                  | Late spawn reply revives exited shell or stale report drains replacement         | Future test `transport_terminal_early_exit_and_stale_report`          | 02                |
| INV-9  | Typed consumer selection atomically uses an existing Connected binding; views cannot issue a generation/start.             | Consumer, ConsumerTarget                   | Owner lookup/reservation; private wrappers; sealed dispatch expected-identity match                          | Server open calls Connect, stale binding retargets or view accepted as authority | Future test `transport_consumers_existing_generation`                 | 03                |
| INV-10 | Every descendant is actually joined before its shared master is closed; other identities remain usable.                    | ConnectionManager, Consumer, Connection    | Registered child sets and drain barriers; live mailbox for late completions                                  | Cancellation or Close send counted as all-kind drain                             | Future test `transport_all_consumers_disconnect`                      | 05                |
| INV-11 | Startup cleanup never admits unresolved old transport or adopts an orphan.                                                 | CleanupReport                              | Validated confined directory and actual exit/staleness/removal observations                                  | Old socket accepted as new connection or unsafe entry deleted                    | Future test `transport_orphan_cleanup`                                | 01                |
| INV-12 | Routed traffic/discovery uses only the registered complete route with every Jump gated before use.                         | Connection, RouteAccess                    | Finite resolved route; strictly shorter-prefix connect workers; no local scan fallback                       | Target inherits Jump trust or hidden route is used                               | Future test `transport_registered_jump_discovery`                     | 04                |

### Ordered flows

**Bootstrap and connect**

1. External current preadmission E6 `StartupOwnership` and validated `RuntimeDirectory` → `cleanup_orphans` performs actual old-socket exit/removal → `CleanupReport::inspect_cleanup`; E6 combines this with actual E4 recovery before `Runtime::admit` produces ReadyOwnership. Empty successful cleanup means an actual empty scan, never a fake report.
2. External GTK primary/dock → `install_transport_ui` → opaque TransportUi. E6 passes it to `ConnectionManager::spawn`; that method consumes `TransportUi::into_transport_ui_parts` to install actual prompt and VTE senders. External E2 Configuration and E3 TrustGate are real earlier owner handles; construction starts no master.
3. External Connect → `ConnectionManager::control` stores waiter, starts resolution worker, immediately resumes mailbox. E2 `resolve` completion supplies ResolvedConnection; owner coalesces/reserves one identity generation. `MasterGeneration::inspect_generation` and `StartReservation::inspect_reservation` expose borrowed original bindings to canonical owners.
4. Owned route worker obtains Local for direct start or uses `RouteAccess::prepare_discovery` for registered prefixes. Prefix Connect uses strictly shorter routes and the same live mailbox; unregistered or ungated prefix fails closed. RoutePrepared carries owned TrustWork.
5. Owned trust worker calls E3 `TrustGate::prepare` with original reservation and resolved input. E3 returns PreparedStart; `PreparedStart::into_prepared_start` returns the SAME reservation with resolved identity/TrustStart. Worker combines retained prefixes into CompilableStart and sends TrustPrepared; failure never authorizes launch.
6. Owner starts E2 compile worker; ConfigurationCompiled returns sealed CompiledConfiguration containing that original reservation/trust binding. Private launch worker accepts ONLY this wrapper, reads `launch_view`, invokes foreground system OpenSSH with generated `-F`, application 0700 runtime control path, ControlPersist=no, parent-death signal and parent-race handling; its spawning supervisor remains alive for the application lifetime, not an ephemeral launch worker. Any created child returns through MasterSpawned even if control-check fails; owner registers/drains it.
7. Agent/key/FIDO remains system OpenSSH. External OpenSSH helper invocation → `serve_askpass` validates UID/socket/nonce/attempt/challenge → ephemeral AuthenticationDialog → GTK prompt; accepted bytes go only to helper stdout once and are zeroized. Cancellation and worker completions remain serviced concurrently.
8. Owner checks attempt ID/start ID/generation on every completion; stale/cancelled completion cannot publish Connected and cannot abandon returned child. Live master plus actual ssh -O check yields Connected; independent evidence alone establishes a specific failure, otherwise Unknown. Actual per-feature probes detect OS/distribution, systemd, busctl JSON, SFTP, rename extension and sha256sum; no sudo permission is inferred.
9. `ConnectionManager::observe` returns same-boundary snapshot/sequenced changes, including NotConnected, actual loading/failure and dated Capabilities. GTK main context alone renders state and actionable errors, separate from Trust State. Startup preference is scheduled by E6 only after safety admission through the identical control path.

**Terminals and typed consumers**

1. External terminal Open → control creates Created → Connecting with fresh TerminalBinding/shell generation and a privately issued TerminalLifecycleSink. GTK reads PrivateTerminalLaunch and VTE spawns actual `ssh -F <generated> -t <attachment-alias>`; no arbitrary external launch entry exists.
2. Actual VTE spawn acknowledgement may make Active only if still current and not already exited/failed. VTE child signal/exit/spawn failure and PTY disposal call `TerminalLifecycleSink::report` independently; early exit-before-spawn reply and old-shell reports cannot revive/drain replacement. Child exit on healthy master yields Exited/Error; master loss yields Disconnected.
3. Input/output/resize stay in per-shell PTY across navigation. Explicit Reconnect preserves old Disconnected tab and opens new shell; explicit Close waits for actual child reaping AND PTY release (or no-child spawn-failure evidence plus released PTY), never merely Close acknowledgement. Terminal text never enters Audit Trail.
4. External E4/E5 request or E3 routed scan → `ConsumerAccess::open` atomically checks existing Server binding or issued Generation and reserves child. Server target never calls Connect, waits for future connection or resolves user alias. Execute/Transfer read sealed permit expected_identity via inspect_execution_permit, Download reads inspect_download_dispatch.expected_identity, and Reconcile reads the original Operation connection_identity through inspect_reconciliation_read; the same owner turn compares it with the actual current connection before reserving/spawning, failing absent/mismatched identity. Readable views never substitute for the sealed request. Inventory edits cannot retarget an existing consumer.
5. `Consumer::into_consumer_parts` moves unique typed receiver/master/ID to owner. Fixed POSIX scripts run as login user with E4 granular commands; SFTP upload/download payload is a distinct subsystem channel; read-only Services and Collector use their canonical structured protocols. For each actual Collector record E1 stamps collector_generation = ConsumerParts.id and collected_at = local receipt Instant, never remote-provided time; replacement channel UUID creates a new collection epoch. Every attachment denies mux fallback/authentication. Missing Capability affects only that consumer.
6. `ConsumerAccess::stop` and natural termination produce actual joined ChannelEnd; `ChannelEnd::inspect_channel_end` exposes dispatch/reason. E4 alone decides Verifier/Unknown/transfer completion; E5 alone decides value/Freshness/backoff. Delivery of SSH exit is not proof of remote outcome.

**Disconnect, master loss and shutdown**

1. External Disconnect → control closes admission for that shared identity, stores waiter and launches stop/join work while processing lifecycle and continuation messages. Every alias observes loss; unrelated identities remain usable.
2. All VTE child/PTY and SFTP/script/Collector descendants join before master close. Late launch completions remain owned and must drain; failure reports incomplete cleanup. No local cancellation implies rollback/no effects.
3. Unexpected master exit/control-check failure invalidates generation and descendants, informs owners, and never replays mutation. E5 cannot resurrect deliberate Disconnect; E4/E6 retain Unknown/read-only recovery semantics.
4. E6 first closes admission and quiesces E4/E5, then calls `ConnectionManager::quiesce`; owner mailbox stays alive until all pending/active children and masters join. Successful result has no surviving owned sessions or Collectors. Crash uses actual parent-death protection and the next startup's cleanup flow, never orphan adoption.

## Decision

Sol actor facades keep transport lifecycle in one owner and GTK objects in its private adapter.
Private sealed launch/reservation and owned continuations close capability and mailbox deadlocks.
Terra read-only download separation is retained as a typed E4 admission; audit ordinal graft remains E4-owned.
Reject independent per-consumer transports and shared mutable repositories: they split authentication/lifecycle ownership.
We accept conservative Unknown and explicit drain acknowledgements for truthful evidence.

## Open questions

Which E3-approved Direct/Jump trust memberships, especially UNVERIFIED, permit use? ADR-0003:02 owns that decision; transport consumes it without choosing policy.
