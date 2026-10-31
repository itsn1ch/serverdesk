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
