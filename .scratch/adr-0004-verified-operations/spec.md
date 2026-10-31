# Verified Operations with Fixed POSIX Scripts and Granular Sudo
Status: needs-info
Source: Accepted ADR-0004; connected ADR-0001, ADR-0002, ADR-0003 and ADR-0006; docs/spec.md §§3, 4.9, 6, 8, 9, 12–16 and 18; GLOSSARY.md.

## Problem Statement

A Developer managing personal Linux Servers needs administrative changes that can be authorized narrowly, attributed to an explicit Server and distinguished from merely successful command execution.
Running an entire script as root would defeat granular sudoers permissions.
SSH loss, an application crash or cancellation can leave a real remote change with an unknown result.
A successful exit code, a briefly active service or a disconnected client cannot independently prove that a requested change succeeded or did not occur.
The accepted architecture requires every structured administrative mutation to pass through one Operation engine, a type-specific Verifier and a permanent sanitized Audit Trail.
The sources do not yet define complete Verifier and Reconciliation policies for all required Operation types; this draft preserves that blocker rather than inventing success conditions.

## Solution

Provide structured service, file and Host Key trust workflows through the Operation engine rather than through arbitrary command construction.
Run fixed POSIX sh templates as the SSH login user, over the existing system OpenSSH ControlMaster, with typed arguments, one tested quoting function and machine-readable output.
Authorize only individual privileged commands and their exact arguments through noninteractive sudo.
Plan and classify each Operation, require explicit impact-aware consent for destructive changes, persist RUNNING before execution and withhold success until the operation-specific Verifier proves it.
Preserve Unknown Outcome whenever effects may have occurred without sufficient evidence.
Resolve uncertainty by read-only Reconciliation, never by automatic replay of a mutation.
Expose read-only service information independently of mutation privileges and explain detected Capability limitations.
Integrate file and trust mutations without adding privileged file editing or rebuilding the full file manager in this epic.

## User Stories

### Service information and execution authority

1. **US-001: Inspect services without depending on mutation privileges**
   - **Story:** As a Developer managing personal Linux Servers, I want structured service information even when I cannot change a service, so that missing administrative permissions do not hide useful observations.
   - **Basis:** Derived: ADR-0004 and docs/spec.md §§4.9 and 8.
   - **Acceptance criteria:**
     - **US-001-AC1:** Given a connected Server with the required Capability, when I open Services, then unit discovery uses `busctl --json=short` against `org.freedesktop.systemd1` without sudo rather than parsing a human-readable unit table.
     - **US-001-AC2:** Given a discovered unit, when I inspect it, then structured properties come from `systemctl show -p <properties> [--value]` and expose unit name, description, ActiveState, SubState, Result and MainPID, with restart count and memory shown when available.
     - **US-001-AC3:** Given access to a unit's journal, when I request recent logs, then the workflow obtains structured records with `journalctl -u <unit> -o json -n <N>` and displays them without human-table parsing.
     - **US-001-AC4:** Given an inspectable unit, when I request its dependencies, then the Services workflow exposes its dependencies rather than offering only mutation controls.
     - **US-001-AC5:** Given that `systemctl list-units` has no JSON mode, when the documented discovery and property mechanisms are available, then Services remains available.
     - **US-001-AC6:** Given unavailable systemd, unavailable required busctl JSON support or denied read access, when I inspect the affected feature, then it is unavailable with its detected reason rather than silently empty or implemented through a privileged fallback.

2. **US-002: Know the exact privilege required for a service change**
   - **Story:** As a Developer managing personal Linux Servers, I want permission checks for the precise change I am requesting, so that broad or unrelated privileges are not mistaken for authorization.
   - **Basis:** Derived: ADR-0004 and docs/spec.md §§4.9, 8.1 and 9.3.
   - **Acceptance criteria:**
     - **US-002-AC1:** Given a Server connecting, when Capability detection runs, then systemd and busctl JSON support are detected and sudo permissions are assessed per Operation and arguments rather than summarized as a single unrestricted-sudo flag.
     - **US-002-AC2:** Given a requested privileged change, when preflight checks authority, then `sudo -n -l` checks the exact absolute command path and exact arguments that the Operation would execute.
     - **US-002-AC3:** Given an authorized service mutation, when execution begins, then the script remains the login user and only the privileged command uses `sudo -n /usr/bin/systemctl <verb> <unit>`.
     - **US-002-AC4:** Given insufficient NOPASSWD permission, when I attempt that mutation, then it is blocked with the affected command's missing permission and no root shell, sudo password prompt, stored sudo password or D-Bus/polkit mutation fallback is attempted.
     - **US-002-AC5:** Given a missing privilege, when I view a suggested sudoers snippet, then it is guidance text only and the application neither installs it nor changes sudoers.
     - **US-002-AC6:** Given permissions differ between verbs or units, when I inspect available actions, then permission for one verb/unit does not authorize another, and permitted read-only service information remains usable independently.

3. **US-003: Pass values as data rather than executable shell text**
   - **Story:** As a Developer managing personal Linux Servers, I want structured inputs to remain data, so that a unit name or file argument cannot execute an unintended command.
   - **Basis:** Derived: ADR-0001, ADR-0004 and docs/spec.md §§6.1, 9.3 and 16.1.
   - **Acceptance criteria:**
     - **US-003-AC1:** Given a structured remote Operation, when it is dispatched, then it selects a fixed POSIX sh template sent through stdin to `sh -s -- <args>` rather than accepting arbitrary command text.
     - **US-003-AC2:** Given a supplied argument, when validation runs, then its accepted domain type is checked before dispatch, including systemd unit-name grammar for unit arguments, and invalid input produces ValidationError without executing the remote mutation.
     - **US-003-AC3:** Given valid values containing shell-significant characters, when they are encoded, then the single tested argument-quoting function preserves the literal values and no command interpolation occurs.
     - **US-003-AC4:** Given a completed script exchange, when the workflow interprets it, then it consumes machine-readable key/value or NUL-delimited records and meaningful exit codes, not a human-oriented table or a presumed success inferred from SSH stderr.
     - **US-003-AC5:** Given a file transfer accompanying an Operation, when file bytes are transferred, then payload uses its separate SFTP channel rather than the script's stdin or an interpolated command argument.

### Planning, evidence and outcomes

4. **US-004: Consent to the actual target and impact**
   - **Story:** As a Developer managing personal Linux Servers, I want to review the real target and consequences of a change, so that I do not unintentionally modify the wrong Server or an unmanaged resource.
   - **Basis:** Derived: docs/spec.md §§3.3, 6.7, 9.1, 9.6 and 18; GLOSSARY.md Managed Resource.
   - **Acceptance criteria:**
     - **US-004-AC1:** Given a structured request, when planning presents it, then its Server context, affected resource, action and classification are explicit rather than inferred from an unrelated currently selected Server.
     - **US-004-AC2:** Given the classification step, when the request is assessed, then it is classified as read-only, idempotent mutation, non-idempotent mutation or destructive mutation without treating every action as harmless because the transport command can be repeated.
     - **US-004-AC3:** Given a destructive mutation, when execution is requested, then it cannot proceed until I explicitly confirm the described impact.
     - **US-004-AC4:** Given an existing destination that will be overwritten, when I request the change, then overwrite confirmation is required before replacement.
     - **US-004-AC5:** Given a remote resource that is not a Managed Resource, when a workflow proposes changing it, then it requires my explicit consent rather than assuming application ownership.
     - **US-004-AC6:** Given a service restart quick action in Overview, when I invoke it, then it enters this same planned and verified Operation workflow for the explicitly shown Server rather than bypassing it.

5. **US-005: Record execution intent before any remote effect**
   - **Story:** As a Developer managing personal Linux Servers, I want durable evidence of an Operation before it can affect a Server, so that interruption cannot erase the fact that a change may have begun.
   - **Basis:** Derived: docs/spec.md §§9.2, 9.4, 12.3 and 13.
   - **Acceptance criteria:**
     - **US-005-AC1:** Given a new structured mutation, when it progresses, then its workflow exposes the applicable REQUESTED, VALIDATING, PLANNING, AWAITING_CONFIRMATION, RUNNING and VERIFYING stages rather than skipping directly from a click to success.
     - **US-005-AC2:** Given a validated and authorized plan, when execution is about to start, then the Operation's RUNNING state is committed in a SQLite transaction before remote execution begins.
     - **US-005-AC3:** Given that the write-ahead transaction fails, when execution is considered, then no remote mutation is started and a StorageError explains that execution was prevented.
     - **US-005-AC4:** Given a persisted Operation, when I inspect its record, then the accepted Operation metadata identifies its Server, resource, action, sanitized parameters, classification, status, timing, verification, sanitized error and correlation identifier as applicable to its stage.
     - **US-005-AC5:** Given an Operation requiring multiple related local persistence changes, when those changes are committed, then transactions and active foreign keys preserve their consistency and no storage value is silently truncated.

6. **US-006: Enable every required systemd verb only with its own evidence policy**
   - **Story:** As a Developer managing personal Linux Servers, I want each service action to have an explicit proof policy, so that superficially similar actions do not inherit an incorrect definition of success.
   - **Basis:** Derived: docs/spec.md §§8.2, 9.1 and 9.5.
   - **Acceptance criteria:**
     - **US-006-AC1:** Given a connected Server and a unit with the necessary detected Capability, exact permission and complete operation-specific evidence policy, when I open its service actions, then I can choose the supported start, stop, restart, reload, enable and disable actions by their actual names rather than being limited to restart or an arbitrary command field.
     - **US-006-AC2:** Given an available service action and any required confirmation, when I choose Start, Stop, Restart, Reload, Enable or Disable, then the Operation details identify the selected Server, unit and corresponding action, and execution requests exactly `systemctl start`, `systemctl stop`, `systemctl restart`, `systemctl reload`, `systemctl enable` or `systemctl disable` respectively for that unit through the authorized Operation workflow, without substituting another verb or adding an unrequested mutation.
     - **US-006-AC3:** Given any service action whose own Verifier or read-only Reconciliation predicate is missing, when I inspect or attempt that action, then it is unavailable with the missing evidence-policy reason even if connection and sudo permission are otherwise ready; another verb's policy does not enable it.
     - **US-006-AC4:** Given an action with an evidence policy that does not cover the selected unit's semantics, when I inspect or attempt that action, then it is unavailable with that unsupported-Capability reason rather than silently using another unit type's evidence, interpreting active state as enablement evidence or assuming disable also stops the unit.

7. **US-007: Distinguish an observed restart from a healthy restart**
   - **Story:** As a Developer managing personal Linux Servers, I want restart evidence to distinguish occurrence from subsequent health, so that a recycled or briefly active unit does not produce false success.
   - **Basis:** Derived: docs/spec.md §9.5.
   - **Preconditions:** The operation-specific restart policy required by US-006 has been completed for the relevant unit semantics.
   - **Acceptance criteria:**
     - **US-007-AC1:** Given a unit whose semantics support the documented evidence, when restart preflight runs, then it captures remote boot_id and the unit's ActiveEnterTimestampMonotonic and/or ExecMainStartTimestampMonotonic as applicable to those semantics.
     - **US-007-AC2:** Given the same boot_id before and after execution, when the applicable monotonic unit timestamp increases, then the documented evidence can establish that a restart occurred but does not alone establish healthy success.
     - **US-007-AC3:** Given occurrence evidence, when health is evaluated, then ActiveState, SubState and Result must satisfy the operation-specific expectations throughout the defined stabilization window before healthy success is reported.
     - **US-007-AC4:** Given a changed boot_id during restart, when the result is assessed, then the reboot is not automatically interpreted as either successful restart or failure.
     - **US-007-AC5:** Given unsupported timestamp semantics, missing evidence or an incomplete health/stabilization policy, when restart proof is requested, then the workflow does not invent timestamps, an expected state or a stabilization duration to declare success.
   - **Dependencies:** US-006.

8. **US-008: Receive an evidence-based final result**
   - **Story:** As a Developer managing personal Linux Servers, I want the final Operation result to reflect what is known about the requested effect, so that I can distinguish a proven outcome from uncertainty.
   - **Basis:** Derived: docs/spec.md §§9.2, 9.5, 14 and 18; GLOSSARY.md Verifier and Unknown Outcome.
   - **Acceptance criteria:**
     - **US-008-AC1:** Given a mutation that has executed, when the process exits, then the Operation enters VERIFYING rather than treating exit status alone as SUCCEEDED.
     - **US-008-AC2:** Given sufficient operation-specific proof of success, when verification finishes, then and only then is SUCCEEDED presented.
     - **US-008-AC3:** Given sufficient operation-specific proof of failure, when verification finishes, then FAILED is recorded with its available evidence rather than being confused with an unobserved remote outcome.
     - **US-008-AC4:** Given SSH loss after a mutation may have begun or otherwise insufficient evidence, when the outcome is assessed, then UNKNOWN is recorded with UnknownRemoteOutcome as appropriate, not an unproven FAILED, CANCELLED or SUCCEEDED.
     - **US-008-AC5:** Given a VerificationError, when the Verifier cannot establish a definite outcome, then the error preserves the Unknown Outcome boundary rather than converting inability to verify into proof that no remote effect occurred.

9. **US-009: Cancel without misrepresenting remote effects**
   - **Story:** As a Developer managing personal Linux Servers, I want cancellation to state whether a change may already have occurred, so that stopping my wait is not confused with undoing a remote action.
   - **Basis:** Derived: docs/spec.md §§6.6, 9.2 and 14; connected ADR-0006.
   - **Acceptance criteria:**
     - **US-009-AC1:** Given a request cancelled before any possible remote effect, when absence of effects is established, then the result can be CANCELLED.
     - **US-009-AC2:** Given cancellation after remote execution may have started, when absence of effects cannot be established, then the result is UNKNOWN rather than CANCELLED.
     - **US-009-AC3:** Given cancellation of an in-progress transfer, when only part of its work is known complete, then neither the transfer nor its enclosing mutation is shown as completed or successfully undone without verification.
     - **US-009-AC4:** Given pending cancellation, when I inspect the UI, then it distinguishes the request to stop from confirmed no-effect cancellation and does not promise rollback.

10. **US-010: Reconcile uncertainty without repeating the change**
    - **Story:** As a Developer managing personal Linux Servers, I want uncertain remote Operations investigated through reads after reconnect, so that recovery does not accidentally repeat a destructive or non-idempotent effect.
    - **Basis:** Derived: docs/spec.md §§9.4–9.5 and connected ADR-0006; GLOSSARY.md Reconciliation.
    - **Acceptance criteria:**
      - **US-010-AC1:** Given a previous RUNNING or VERIFYING Operation interrupted by application termination, when the startup recovery owned by the ADR-0006 epic processes it, then the Operation is UNKNOWN and marked for Reconciliation rather than resumed as if still executing.
      - **US-010-AC2:** Given an unresolved remote Operation and a subsequent connection to its Server, when Reconciliation runs, then it reads remote state through that Operation type's predicate without re-executing the mutation.
      - **US-010-AC3:** Given sufficient read-only evidence after reconnect, when Reconciliation completes, then the result may become SUCCEEDED or FAILED with the evidence retained.
      - **US-010-AC4:** Given still-insufficient evidence, when Reconciliation completes, then the result remains UNKNOWN and is still visibly unresolved.
      - **US-010-AC5:** Given any transport failure, verifier failure or Unknown Outcome, when normal handling or recovery continues, then no mutation is automatically retried, including an idempotent mutation.

### File, trust and audit integration

11. **US-011: Route file changes through the same safety boundary**
    - **Story:** As a Developer managing personal Linux Servers, I want file changes to share Operation safeguards without granting root file access, so that file management does not become an unverified privilege bypass.
    - **Basis:** Derived: docs/spec.md §§6, 9.1 and 9.5; GLOSSARY.md Managed Resource.
    - **Acceptance criteria:**
      - **US-011-AC1:** Given file write/edit, delete, rename, chmod or upload overwriting a file, when I initiate the change, then it passes through the Operation engine with its own Verifier and read-only Reconciliation policy before that mutation type is enabled.
      - **US-011-AC2:** Given an SSH user without write permission, when a file mutation is requested, then the file is read-only or the request is rejected with a clear reason; no privileged file-editing fallback is offered in MVP.
      - **US-011-AC3:** Given an edited file whose mtime, size or hash differs from the opening snapshot, when I save, then FileChangedExternally blocks the write and offers reload or explicit overwrite rather than silently replacing the external change.
      - **US-011-AC4:** Given a mutation involving a symlink or a path-traversal attempt, when its destination is validated, then it cannot silently redirect an approved change to an unintended resource.
      - **US-011-AC5:** Given another administrative file mutation exposed by the file manager, including file or directory creation, when its workflow is enabled, then it cannot bypass the same engine and per-type evidence prerequisite merely because it is not an overwrite.

12. **US-012: Verify transferred file effects independently of payload progress**
    - **Story:** As a Developer managing personal Linux Servers, I want transfer completion to mean verified destination state rather than bytes merely sent, so that partial uploads are not mistaken for complete changes.
    - **Basis:** Derived: docs/spec.md §§6.1, 6.4, 6.6 and 9.1.
    - **Acceptance criteria:**
      - **US-012-AC1:** Given an upload or file write, when the payload is applied, then data is written to a temporary file in the destination directory and promoted using `posix-rename@openssh.com` when that extension is available.
      - **US-012-AC2:** Given that atomic rename is unavailable, when non-atomic overwrite is proposed, then its warning and additional confirmation occur before overwrite rather than silently weakening atomicity.
      - **US-012-AC3:** Given a transferred file entering VERIFYING, when destination contents are assessed, then size is compared and an optional hash comparison is used when `sha256sum` is available; unsupported hashing is not represented as a successful hash check.
      - **US-012-AC4:** Given failure before or after rename, when completion is assessed, then partial transfer is never COMPLETED and the result distinguishes a proven failed change from a potentially applied change requiring UNKNOWN and Reconciliation.
      - **US-012-AC5:** Given a new upload or download, when the transfer is recorded, then its path, size and result are retained as metadata without file content; its transfer lifecycle distinguishes QUEUED, TRANSFERRING, VERIFYING, COMPLETED, FAILED, CANCELLED and UNKNOWN where applicable.

13. **US-013: Apply the Operation contract to Host Key trust changes**
    - **Story:** As a Developer managing personal Linux Servers, I want explicit trust changes to be verified and audited, so that altering which Server I trust cannot disappear outside the administrative workflow.
    - **Basis:** Derived: ADR-0003 and docs/spec.md §§4.6, 9.1, 9.5 and 13.
    - **Acceptance criteria:**
      - **US-013-AC1:** Given a requested Host Key trust change, when it is applied, then it participates in the Operation engine and sanitized Audit Trail rather than writing around them.
      - **US-013-AC2:** Given replacement or deletion of a trust entry, when I proceed, then explicit consent and the affected Server and Host Key identity are part of the request rather than automatic acceptance of observed drift.
      - **US-013-AC3:** Given a trust mutation type without its own verifiable persisted-state outcome and Reconciliation rule, when the action is considered for enablement, then it remains unavailable under the same evidence prerequisite; no remote shell or remote-reconnect prerequisite is fabricated for a local SQLite trust commit.
      - **US-013-AC4:** Given an accepted trust mutation, when its outcome is consumed by connection handling, then the ADR-0003 epic remains responsible for authoritative trust semantics and atomic generated known_hosts behavior; an Operation success does not independently bypass its connection gate.

14. **US-014: Keep a permanent sanitized Audit Trail**
    - **Story:** As a Developer managing personal Linux Servers, I want a durable Audit Trail of structured administrative changes, so that I can inspect their intent, evidence and uncertainty without leaking secrets or file contents.
    - **Basis:** Derived: docs/spec.md §§3.1, 9.8, 13 and 15; GLOSSARY.md Audit Trail.
    - **Acceptance criteria:**
      - **US-014-AC1:** Given any structured mutation in scope, when its lifecycle progresses, then its Operation and associated operation_events contribute to the Audit Trail; the accepted coverage target is all structured mutations, not terminal command capture.
      - **US-014-AC2:** Given an existing audit record or event, when later lifecycle information is recorded, then the append-only Audit Trail preserves prior evidence and is not automatically deleted in MVP.
      - **US-014-AC3:** Given arguments, remote errors or verification details containing sensitive material, when persistence occurs, then parameters, sanitized_error and sanitized_payload contain no secret, password, passphrase or file contents.
      - **US-014-AC4:** Given a terminal command typed directly by me, when it executes, then it remains outside the Operation engine and Audit Trail rather than being falsely claimed as audited structured administration.
      - **US-014-AC5:** Given a recent UNKNOWN Operation, when Overview shows recent Operations, then it remains included with its actual unresolved status rather than disappearing from recent activity.
      - **US-014-AC6:** Given a schema upgrade or database initialization, when operation storage is prepared, then versioned migrations, UUID identities and active foreign keys preserve accepted metadata relationships without introducing automatic audit retention or export behavior.

### Presentation and coordination

15. **US-015: Understand progress and actionable errors accessibly**
    - **Story:** As a Developer managing personal Linux Servers, I want usable progress and precise error explanations, so that I can act on a failure without guessing whether a change occurred.
    - **Basis:** Derived: docs/spec.md §§2.2, 3.4, 14 and 18.
    - **Acceptance criteria:**
      - **US-015-AC1:** Given a Services or Operation view still fetching data, when I open it, then it visibly shows loading rather than an invented empty or successful state.
      - **US-015-AC2:** Given a successful discovery with no units or an empty Operation history, when the relevant view is shown, then it presents a genuine empty state distinct from an error or pending request.
      - **US-015-AC3:** Given any workflow error, when the UI reports it, then it identifies what failed, the affected resource, whether change may have occurred, available actions and a link to Operation details where applicable.
      - **US-015-AC4:** Given permission, unsupported Capability, invalid input, execution, verification, external-file-change, storage or unknown-outcome failures, when displayed, then they retain their corresponding accepted error distinction rather than collapsing into a generic connection failure.
      - **US-015-AC5:** Given keyboard-only use or stale service observations after disconnect, when I navigate service details, action confirmation and Operation details, then controls remain keyboard-accessible and stale observations are visibly distinguished from current evidence.

16. **US-016: Preserve connection, resource and UI ownership boundaries**
    - **Story:** As a Developer managing personal Linux Servers, I want administrative work coordinated with the existing connection and resource owners, so that one change cannot freeze the application or silently conflict with another.
    - **Basis:** Derived: ADR-0001, ADR-0002, ADR-0006 and docs/spec.md §§9.7, 12 and 18.
    - **Acceptance criteria:**
      - **US-016-AC1:** Given a connected Server and an Operation, when remote work starts, then it consumes the ADR-0001 epic's existing system OpenSSH master under ADR-0002 generated configuration rather than establishing a competing SSH implementation or bypassing trust policy.
      - **US-016-AC2:** Given another conflicting Operation on the same Connection Identity and resource scope, including a Server alias with that identity, when the request is made, then the ADR-0006 lock owner rejects the conflict explicitly rather than silently queuing it; nonconflicting resources are not globally serialized by this epic.
      - **US-016-AC3:** Given resources that may have changed outside the application, when preflight reads run, then they assess current state without claiming local locks prevent external changes or acquiring a new remote lock mechanism.
      - **US-016-AC4:** Given Operation completion, failure or cancellation, when cleanup hands work back to the lock owner, then a pending stop is not represented as proof that owned remote work no longer conflicts; ADR-0006 owns safe lock release and startup cleanup.
      - **US-016-AC5:** Given long execution, verification, reconnect or an unreachable Server, when I use the application, then GTK remains responsive, Tokio performs remote work without accessing widgets and presentation updates arrive through channels scheduled on glib::MainContext with a single owner for each state.

## Implementation Decisions

- The accepted architecture is a future implementation contract, not a description of existing functionality.
- The application layer coordinates workflow and the Operation engine owns Operation state.
- Fixed POSIX sh templates, typed validation, one argument-quoting function and machine-readable records are the accepted remote execution mechanism.
- The script runs as the login user; only individual privileged commands may use noninteractive sudo after exact-command authorization.
- A general root shell, password sudo fallback and a second D-Bus/polkit mutation privilege model are excluded.
- Service discovery uses structured busctl output, properties use systemctl show and journal records use journalctl JSON.
- The lifecycle distinguishes execution, verification and semantic outcome; process completion is not outcome proof.
- Each enabled mutation type requires its own Verifier and read-only Reconciliation predicate.
- SQLite owns durable local state, Operation metadata and the append-only Audit Trail; RUNNING must be transactionally persisted before remote execution.
- The accepted operation_events and operations metadata are retained without secrets or file contents.
- File payload stays on SFTP, separately from operation-script stdin, and file mutations remain bounded by the SSH user's permissions.
- Trust mutation intent and evidence join the engine while the Host Key Store retains its accepted authority and generated-artifact responsibilities.
- Existing connection identity, generated configuration, trust gating and master ownership remain owned by the connected ADR epics.
- The single-instance/resource-lock epic owns lock arbitration, orphan cleanup and startup interrupted-state mapping.
- GTK owns presentation, the application layer owns workflow, and asynchronous work communicates through channels scheduled on glib::MainContext rather than accessing widgets from Tokio.

## Testing Decisions

- These are planned tests, not executed or passing checks; no behavioral harness or implementation is supplied by this specification task.
- The approved primary seam is the presentation-facing application workflow used by GTK, backed by actual system OpenSSH, SQLite and systemd rather than mocked SSH success or fabricated stderr classifications.
- Integration fixtures are isolated Incus/LXD Ubuntu 24.04 containers with systemd 255, sshd, a non-root user, granular sudoers and dummy services.
- Prove exact allowed command/argument execution and denied verb/unit variants through actual sudoers, observing real service state and absence of shell/password/polkit fallback.
- Exercise structured unit discovery, properties, dependencies and journal access, including unavailable busctl JSON, denied journal access and absence of a list-units JSON feature.
- Use narrow pure tests for typed validation, argument quoting, classification, lifecycle transitions and completed operation-specific Verifier predicates.
- Cover shell-significant values through observable literal argument effects, not checks of generated script formatting alone.
- For every enabled service verb, assert its approved postconditions and read-only Reconciliation behavior against real systemd; until those policies exist, assert the evidence gate and do not claim full verb acceptance.
- Exercise the documented restart evidence with unchanged boot identity, increasing timestamps, unsupported unit semantics, unhealthy results and remote reboot; health expectations and stabilization cases require the unresolved policy below.
- Inject SSH disconnect while mutating, cancellation before and after possible effects and application crash after write-ahead, then inspect persisted evidence and real remote state rather than assuming all disconnects are failures.
- Coordinate startup recovery and lock-conflict coverage with the ADR-0006 epic, including alias conflicts and no mutation replay during Reconciliation.
- Exercise file failure before and after rename, partial payload, optional hash availability, absent atomic-rename extension and external-edit conflicts through actual SFTP workflows.
- Restrict disk-full failure injection to an isolated container filesystem, such as a loop device inside that container, never the host filesystem.
- Inspect SQLite for transactional write-ahead, preserved append-only audit evidence, active foreign keys, versioned migration behavior and absence of secrets or file content.
- Test Host Key mutation integration through its real store and explicit confirmation, while leaving trust acceptance semantics to the ADR-0003 epic.
- Actual GTK smoke tests cover keyboard access, confirmation, loading/empty/error/stale states, Unknown Outcome visibility and a responsive main thread while remote work waits.
- The 100% structured-mutation audit coverage item is an acceptance/benchmark target to demonstrate, not a measured result of this document.

## Out of Scope

- Arbitrary command execution UI, terminal command capture, a root shell and terminal command verification.
- Password-based sudo, Secret Service storage for sudo and automatic sudoers installation.
- D-Bus/polkit mutations or disabling Services solely because human-table commands lack JSON.
- Privileged file editing, root configuration deployment, hosting validation/reload workflows and other Milestone 4 behavior.
- The full file browsing/editor feature set, except the explicitly described Operation integration, conflict, privilege, transfer and evidence boundaries.
- Designing new Verifier algorithms or choosing undocumented success states, timeout values or stabilization durations.
- Automatic mutation retry, rollback guarantees or replay of interrupted requests.
- Distributed or remote locks, multi-server operations, audit export and audit retention/deletion policy.
- Replacing the transport, config, Host Key Store, Collector or single-instance implementations owned by their separate ADR epics.

## Further Notes

### Publication blockers

- docs/spec.md §9.5 mandates a Verifier and Reconciliation predicate for every Operation type before implementation, but defines no complete policies for start, stop, reload, enable or disable.
- Restart has an occurrence example, not a complete supported-unit policy: applicable timestamp semantics, expected ActiveState/SubState/Result combinations, stabilization duration and the read-only Reconciliation predicate remain unspecified.
- File write/upload has destination-size and optional-hash evidence in §6.4, but complete per-type Reconciliation predicates and the remaining file mutation predicates, including delete, rename, chmod and creation, are not defined.
- Host Key changes must use the engine; the ADR-0003 epic supplies committed-store acceptance, replacement and deletion postconditions, but accepted sources do not give a complete per-type Reconciliation policy for interrupted local trust mutations or clarify how the remote-oriented lifecycle specializes to those local changes.
- The safe engine invariant and unavailable-action gate are fully observable today as requirements; they are not a substitute for delivering the complete six-verb service scope and all enabled mutation types.
- This draft is therefore needs-info rather than ready-for-agent; the missing accepted policies must be resolved before this whole epic can be published as implementable end to end.

### Cross-epic assumptions

- The ADR-0001 epic supplies an actual connected master and classifies transport failures without treating SSH exit 255 or stderr text as a complete outcome taxonomy.
- The ADR-0002 epic supplies isolated generated configuration; the ADR-0003 epic supplies the trust gate and owns trust acceptance semantics. Local trust changes commit to SQLite with audit under that epic's store contract; remote execution and reconnect are not prerequisites to committing a local trust change.
- The ADR-0006 epic owns single-instance startup sequencing, interrupted RUNNING/VERIFYING mapping, read-only reconnect scheduling and safe lock lifecycle; this epic owns each Operation's semantic evidence and outcome.
- The ADR-0005 epic's live service summary is observational and cannot replace an Operation-specific Verifier with unspecified Freshness or stabilization assumptions.
- No cross-epic story IDs are used as local implementation dependencies.

### Source and delivery caveats

- ADR-0004 establishes fixed scripts, granular noninteractive sudo and guidance-only sudoers snippets; §§8–9 supply service mechanisms, lifecycle and verification constraints.
- §§3, 4.9 and 6 supply explicit Server context, detected Capability and file integration; §§12–16 and 18 supply ownership, persistence, errors, targets, approved test environment and completion obligations.
- GLOSSARY.md governs Server, Connection Identity, Operation, Verifier, Reconciliation, Unknown Outcome, Audit Trail, Managed Resource and Capability terminology.
- The current repository is only a dependency-free Rust greeting program; none of the behaviors or test results above is claimed as implemented.
