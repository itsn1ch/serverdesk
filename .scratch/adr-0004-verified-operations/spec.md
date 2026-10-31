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

# Design

## Usage

1. Read actual permanent history without SSH; after real E1 consumers exist, inspect Services and bounded text-opening independently of mutation permission.
2. Before transport exists, approved local trust planning uses prepare_local_trust, inspect_local_trust_plan and admit_local_trust; the registered commit worker completes through private finish_local_trust. Later public plan_operation/decide_operation delegate these exact stages for the same immutable target/impact/revision.
3. inspect_operations/observe show durable progress; cancel_operation requests stopping, reconcile_operation/reconcile_connected read only, and quiesce returns ordinary OperationQuiesced IDs only after joined work and acknowledged releases. Downloads have their own read-only admission and proof gate.

## Types

```rust
pub enum ServiceVerb { Start, Stop, Restart, Reload, Enable, Disable }
pub enum OperationClass {
    ReadOnly, IdempotentMutation, NonIdempotentMutation, DestructiveMutation,
}
pub enum OperationStatus {
    Requested, Validating, Planning, AwaitingConfirmation,
    Running, Verifying, Succeeded, Failed, Unknown, Cancelled,
}
pub enum Confirmation {
    RequestedChange, DestructiveImpact, Overwrite, UnmanagedResource,
    NonAtomicOverwrite, TofuWarning, TrustReplacement, TrustDeletion,
}
pub struct ResourceScope {
    pub(crate) resource_type: String,
    pub(crate) resource_id: String,
}
pub struct FileVersion {
    pub requested_path: String,
    pub resolved_path: String,
    pub size: u64,
    pub mtime_seconds: i64,
    pub mtime_nanoseconds: u32,
    pub sha256: [u8; 32],
    pub observed_at: Instant,
}
pub struct TextDocument {
    pub server_id: Uuid,
    pub path: String,
    pub utf8: Arc<str>,
    pub version: FileVersion,
}
pub struct TextOpenRequest {
    pub server_id: Uuid,
    pub path: String,
}
pub enum FileChange {
    CreateFile { path: String },
    CreateDirectory { path: String },
    WriteText { opened: TextDocument, replacement: Arc<str> },
    Delete { path: String },
    Rename { source: String, destination: String },
    Chmod { path: String, mode: u32 },
    Upload { local_source: PathBuf, destination: String },
}
pub enum AdministrativeChange {
    Service { verb: ServiceVerb, unit: String },
    File(FileChange),
    Download(DownloadRequest),
    Trust(E3::TrustChange),
}
pub struct OperationRequest {
    pub server_id: Uuid,
    pub change: AdministrativeChange,
}
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
pub struct JournalRecord {
    pub observed_at: Option<SystemTime>,
    pub priority: Option<u8>,
    pub message: String,
}
pub struct ServiceSummary {
    pub unit_states: Vec<(String, String, String)>,
    pub observed_at: Instant,
}
pub enum ServiceRead {
    Units { server_id: Uuid },
    Unit { server_id: Uuid, unit: String },
    Dependencies { server_id: Uuid, unit: String },
    Journal { server_id: Uuid, unit: String, limit: u32 },
    Summary { server_id: Uuid },
}
pub enum ResourceQuery {
    Services(ServiceRead),
    OpenText(TextOpenRequest),
}
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
pub struct ServiceEvidence {
    pub remote_boot_id: Uuid,
    pub active_enter_monotonic_usec: Option<u64>,
    pub exec_main_start_monotonic_usec: Option<u64>,
    pub unit: UnitInfo,
    pub observed_at: Instant,
}
pub struct VerificationEvidence {
    pub policy_reference: Option<String>,
    pub facts: Vec<(String, String)>,
    pub service_observations: Vec<ServiceEvidence>,
    pub file_observations: Vec<FileVersion>,
    pub observed_at: SystemTime,
}
pub enum TransferState {
    Queued, Transferring, Verifying, Completed, Failed, Cancelled, Unknown,
}
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
pub struct TransferProgress {
    pub transfer_id: Uuid,
    pub bytes_transferred: u64,
    pub expected_bytes: Option<u64>,
}
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
pub struct OperationPlan {
    pub operation: Operation,
    pub revision: Uuid,
    pub scopes: Vec<ResourceScope>,
    pub impact: String,
    pub required_confirmations: Vec<Confirmation>,
    pub availability: E6::Availability,
}
pub enum PlanResult {
    Confirmable(OperationPlan),
    Blocked {
        operation: Operation,
        required_decisions: Vec<String>,
        failure: E6::Failure,
    },
}
pub enum OperationDecision {
    Confirm { revision: Uuid, acknowledged: Vec<Confirmation> },
    Reject { revision: Uuid },
}
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
pub struct OperationRecorded {
    pub id: Uuid,
    pub operation_id: Uuid,
    pub ordinal: u64,
    pub fact: OperationFact,
    pub created_at: SystemTime,
}
pub enum OperationQuery {
    Recent { limit: u32 },
    Detail { operation_id: Uuid },
    NeedingReconciliation,
    Interrupted,
}
pub struct OperationHistory {
    pub operations: Vec<Operation>,
    pub events: Vec<OperationRecorded>,
}
pub struct OperationChanged {
    pub sequence: u64,
    pub operation: Operation,
    pub event: OperationRecorded,
}
pub struct OperationEngine { tx: mpsc::Sender<OperationMessage> }

pub(crate) enum EnabledMutation {}
pub(crate) enum LocalTrustApproval {}

pub(crate) struct ExecutionPermit {
    expected_identity: E2::ConnectionIdentity,
    operation_id: Uuid,
    plan_revision: Uuid,
    write_ahead: AuditTail,
    lease_id: Uuid,
    enabled: EnabledMutation,
}
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
pub(crate) struct RemoteDispatch {
    request: OperationRequest,
    preflight: VerificationEvidence,
    permit: ExecutionPermit,
}
pub(crate) struct TransferDispatch {
    transfer_id: Uuid,
    request: OperationRequest,
    expected_source_size: u64,
    expected_source_sha256: Option<[u8; 32]>,
    destination_version: Option<FileVersion>,
    use_atomic_rename: bool,
    permit: ExecutionPermit,
}
pub(crate) struct ReconciliationRead {
    operation: Operation,
    policy_reference: String,
}
pub(crate) struct ExecutionObservation {
    pub(crate) operation_id: Uuid,
    pub(crate) dispatch: E1::DispatchKnowledge,
    pub(crate) exit_code: Option<i32>,
    pub(crate) evidence: VerificationEvidence,
    pub(crate) failure: Option<E6::Failure>,
}
pub(crate) struct WorkJoined {
    operation_id: Uuid,
    lease_id: Uuid,
    closed_consumer_ids: Vec<Uuid>,
    joined_task_ids: Vec<Uuid>,
}
pub(crate) struct RecoveryReport {
    startup_generation: Uuid,
    interrupted_operation_ids: Vec<Uuid>,
    unresolved_operation_ids: Vec<Uuid>,
    committed_event_ids: Vec<Uuid>,
}
pub(crate) struct AuditAppend {
    new_intents: Vec<OperationIntent>,
    events: Vec<OperationRecorded>,
    expected_tails: Vec<(Uuid, Option<AuditTail>)>,
}
pub(crate) enum AuditStoreRequest {
    Read(OperationQuery),
    Append(AuditAppend),
}
pub(crate) enum AuditStoreResult {
    History(OperationHistory),
    Appended { event_ids: Vec<Uuid>, history: OperationHistory },
}
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

impl OperationEngine {
    pub(crate) fn spawn(
        store: E6::Store,
        configuration: E2::Configuration,
        trust: E3::TrustExecutor,
        locks: E6::LockAuthority,
    ) -> OperationEngine;

    pub(crate) async fn bind_transport(
        &self, consumers: E1::ConsumerAccess,
    ) -> Result<(), E6::Failure>;

    pub async fn inspect_resources(
        &self, query: ResourceQuery,
    ) -> Result<ResourceObservation, E6::Failure>;

    pub async fn plan_operation(
        &self, request: OperationRequest,
    ) -> Result<PlanResult, E6::Failure>;

    pub async fn decide_operation(
        &self, operation_id: Uuid, decision: OperationDecision,
    ) -> Result<Operation, E6::Failure>;

    pub async fn inspect_operations(
        &self, query: OperationQuery,
    ) -> Result<OperationHistory, E6::Failure>;

    pub async fn cancel_operation(
        &self, operation_id: Uuid,
    ) -> Result<Operation, E6::Failure>;

    pub async fn reconcile_operation(
        &self, operation_id: Uuid,
    ) -> Result<Operation, E6::Failure>;

    pub async fn observe(
        &self,
    ) -> Result<E6::Subscription<OperationHistory, OperationChanged>, E6::Failure>;

    pub(crate) async fn recover_interrupted(
        &self, startup: &E6::StartupOwnership,
    ) -> Result<RecoveryReport, E6::Failure>;

    pub(crate) async fn quiesce(
        &self,
    ) -> Result<OperationQuiesced, E6::Failure>;
}


pub(crate) struct ExecutionPermitView<'a> {
    pub(crate) expected_identity: &'a E2::ConnectionIdentity,
    pub(crate) operation_id: Uuid,
    pub(crate) plan_revision: Uuid,
    pub(crate) write_ahead: AuditTail,
    pub(crate) lease_id: Uuid,
}
pub(crate) struct RemoteDispatchView<'a> {
    pub(crate) request: &'a OperationRequest,
    pub(crate) preflight: &'a VerificationEvidence,
    pub(crate) permit: &'a ExecutionPermit,
}
pub(crate) struct TransferDispatchView<'a> {
    pub(crate) transfer_id: Uuid,
    pub(crate) request: &'a OperationRequest,
    pub(crate) expected_source_size: u64,
    pub(crate) expected_source_sha256: Option<[u8; 32]>,
    pub(crate) destination_version: Option<&'a FileVersion>,
    pub(crate) use_atomic_rename: bool,
    pub(crate) permit: &'a ExecutionPermit,
}
pub(crate) struct ReconciliationReadView<'a> {
    pub(crate) operation: &'a Operation,
    pub(crate) policy_reference: &'a str,
}
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
pub(crate) struct AuditBatchData {
    pub(crate) new_intents: Vec<OperationIntent>,
    pub(crate) events: Vec<OperationRecorded>,
    pub(crate) expected_tails: Vec<(Uuid, Option<AuditTail>)>,
}
pub(crate) struct WorkJoinedView<'a> {
    pub(crate) operation_id: Uuid,
    pub(crate) lease_id: Uuid,
    pub(crate) closed_consumer_ids: &'a [Uuid],
    pub(crate) joined_task_ids: &'a [Uuid],
}
pub(crate) struct RecoveryReportView<'a> {
    pub(crate) startup_generation: Uuid,
    pub(crate) interrupted_operation_ids: &'a [Uuid],
    pub(crate) unresolved_operation_ids: &'a [Uuid],
    pub(crate) committed_event_ids: &'a [Uuid],
}

impl ExecutionPermit {
    pub(crate) fn inspect_execution_permit(&self) -> ExecutionPermitView<'_>;
}
impl RemoteDispatch {
    pub(crate) fn inspect_remote_dispatch(&self) -> RemoteDispatchView<'_>;
}
impl TransferDispatch {
    pub(crate) fn inspect_transfer_dispatch(&self) -> TransferDispatchView<'_>;
}
impl ReconciliationRead {
    pub(crate) fn inspect_reconciliation_read(&self) -> ReconciliationReadView<'_>;
}
impl LocalTrustEnvelope {
    pub(crate) fn local_trust_store_view(&self) -> LocalTrustEnvelopeView;
}
impl AuditAppend {
    pub(crate) fn into_store_batch(self) -> AuditBatchData;
}
impl WorkJoined {
    pub(crate) fn inspect_joined_work(&self) -> WorkJoinedView<'_>;
}
impl RecoveryReport {
    pub(crate) fn inspect_recovery(&self) -> RecoveryReportView<'_>;
}


pub(crate) struct OperationQuiesced {
    pub(crate) operation_ids: Vec<Uuid>,
    pub(crate) released_lease_ids: Vec<Uuid>,
}
pub(crate) struct OwnedWorkJoined {
    operation_id: Uuid,
    consumer_ends: Vec<E1::ChannelEnd>,
    local_task_results: Vec<(Uuid, Result<(), E6::Failure>)>,
}
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



// E4 domain input; OperationRequest still supplies the explicit server_id.
pub struct DownloadRequest {
    pub source: String,
    pub local_destination: PathBuf,
}

// No approved proof cases are supplied today.
pub(crate) enum DownloadProofApproval {}

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
pub(crate) struct DownloadDispatch {
    transfer_id: Uuid,
    admission: DownloadAdmission,
}
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
impl DownloadDispatch {
    pub(crate) fn inspect_download_dispatch(&self) -> DownloadDispatchView<'_>;
}



// E4; Copy immutable identity of an already committed event position.
#[derive(Clone, Copy)]
pub(crate) struct AuditTail {
    pub(crate) event_id: Uuid,
    pub(crate) ordinal: u64,
}



pub(crate) struct ReconciliationBatch {
    pub(crate) identity: E2::ConnectionIdentity,
    pub(crate) generation: u64,
    pub(crate) outcomes: Vec<Operation>,
    pub(crate) unavailable: Vec<E6::Failure>,
}

impl OperationEngine {
    pub(crate) async fn reconcile_connected(
        &self,
        connection: E1::Connection,
    ) -> Result<ReconciliationBatch, E6::Failure>;
}

pub(crate) struct LocalTrustPlan {
    plan: OperationPlan,
    prepared: E3::PreparedTrustChange,
}
pub(crate) struct LocalTrustAdmitted {
    prepared: E3::PreparedTrustChange,
    audit: LocalTrustEnvelope,
}
impl OperationEngine {
    pub(crate) async fn prepare_local_trust(
        &self, request: OperationRequest,
    ) -> Result<LocalTrustPlan, E6::Failure>;
    pub(crate) async fn admit_local_trust(
        &self, plan: LocalTrustPlan, decision: OperationDecision,
    ) -> Result<LocalTrustAdmitted, E6::Failure>;
}
impl LocalTrustPlan {
    pub(crate) fn inspect_local_trust_plan(&self) -> &OperationPlan;
}
impl LocalTrustAdmitted {
    pub(crate) fn into_local_trust_admitted(
        self,
    ) -> (E3::PreparedTrustChange, LocalTrustEnvelope);
}

// Restricted implementation lives in private application::operations::local_trust.
impl OperationEngine {
    pub(in crate::application::operations) async fn finish_local_trust(
        &self, operation_id: Uuid,
        result: Result<E3::TrustCommitObservation, E6::Failure>,
    ) -> Result<Operation, E6::Failure>;
}
```

### Persistence and actor ownership

OperationEngine is the sole ordinal allocator and semantic Operation/Transfer writer. `operations` holds immutable accepted intent; `operation_events.sanitized_payload` encodes ordinal plus fact, and event_type retains its accepted meaning. There is no new column/table, SQL rowid chronology, mutable current-state audit row or second persistence stream. Each Operation starts at ordinal 1; subsequent batches carry the actual committed AuditTail and contiguous successors. Store validates all affected tails, ordinal uniqueness/contiguity, event UUIDs and FKs in the same transaction; matching duplicate identity/content returns retained facts, conflicting duplicates roll back. Folding orders validated per-Operation ordinals, never timestamps, UUIDs or physical retrieval order. One unresolved append reservation per Operation prevents gaps without globally serializing independent Operations; ambiguous writes force actual history/tail read. UPDATE/DELETE are denied for intent and event tables. Sanitization excludes passwords, passphrases, secrets, file bytes, text buffers and raw remote output before E4 creates a sealed AuditAppend.

Every authority wrapper keeps private fields and is non-Clone; views are immutable observations, not write authority. AuditAppend and LocalTrustEnvelope enter Store sealed; their projections are not accepted as Store requests. ExecutionObservation may be infrastructure-created facts but never proof of outcome. Empty EnabledMutation, LocalTrustApproval and DownloadProofApproval are deliberate unavailable policy gates, not implementation stubs; only human-approved per-type contract changes can supply supported constructors. No generic Verifier registry or permissive boolean is introduced.

Owner handlers retain pending replies and expected work IDs, launch owned workers with immutable inputs and resume mailbox service; workers return completion messages and never mutate owner state. Lease state is Held → ReleasePending → Released. Stop/join errors with any unjoined owned work create no WorkJoined. Settled joins and outcome persistence precede the release worker; OperationQuiesced is emitted only after actual release acknowledgements and never returns already-consumed WorkJoined values. No actor-state lock or SQLite transaction spans a remote await.

ResourceScope is immutable pub(crate) equality-key data, never authority: only the actual planner selects its operation-specific key, without new hierarchy or canonicalization. ExecutionPermit and DownloadAdmission privately retain expected_identity; borrowed views expose that exact reference so E1 atomically compares the selected current binding before consumer reservation.

## Contract

### Vocabulary

| Name                     | Kind   | Meaning                                                                                               | Source of truth                                             | Writer            |
| ------------------------ | ------ | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ----------------- |
| `ServiceVerb`            | enum   | The six required systemd mutation verbs                                                               | `src/domain/operation.rs` `ServiceVerb`                     | none (immutable)  |
| `OperationClass`         | enum   | Accepted effect/repetition classification                                                             | `src/domain/operation.rs` `OperationClass`                  | none (immutable)  |
| `OperationStatus`        | enum   | Request/execution/evidence lifecycle                                                                  | `src/domain/operation.rs` `OperationStatus`                 | none (immutable)  |
| `Confirmation`           | enum   | Revision-bound impact acknowledgement                                                                 | `src/domain/operation.rs` `Confirmation`                    | none (immutable)  |
| `ResourceScope`          | value  | Operation-specific local equality key                                                                 | `src/domain/operation.rs` `ResourceScope`                   | none (immutable)  |
| `FileVersion`            | value  | Opening/preflight identity, mtime, size and content hash                                              | `src/domain/resource_observation.rs` `FileVersion`          | none (immutable)  |
| `TextDocument`           | value  | Bounded in-memory UTF-8 opening and evidence                                                          | `src/domain/resource_observation.rs` `TextDocument`         | none (immutable)  |
| `TextOpenRequest`        | value  | Explicit Server/path read                                                                             | `src/domain/resource_observation.rs` `TextOpenRequest`      | none (immutable)  |
| `FileChange`             | enum   | Required bounded file/transfer intents                                                                | `src/domain/operation.rs` `FileChange`                      | none (immutable)  |
| `AdministrativeChange`   | enum   | Service/file/trust structured intent                                                                  | `src/domain/operation.rs` `AdministrativeChange`            | none (immutable)  |
| `OperationRequest`       | value  | Explicit Server and change                                                                            | `src/domain/operation.rs` `OperationRequest`                | none (immutable)  |
| `UnitInfo`               | value  | Structured service details and per-action availability                                                | `src/domain/resource_observation.rs` `UnitInfo`             | none (immutable)  |
| `JournalRecord`          | value  | Read-only journal observation                                                                         | `src/domain/resource_observation.rs` `JournalRecord`        | none (immutable)  |
| `ServiceSummary`         | value  | Structured observational unit summary                                                                 | `src/domain/resource_observation.rs` `ServiceSummary`       | none (immutable)  |
| `ServiceRead`            | enum   | Required service read requests                                                                        | `src/domain/resource_observation.rs` `ServiceRead`          | none (immutable)  |
| `ResourceQuery`          | enum   | Bounded service/text-opening reads                                                                    | `src/domain/resource_observation.rs` `ResourceQuery`        | none (immutable)  |
| `ResourceObservation`    | enum   | Explicit-context read result                                                                          | `src/domain/resource_observation.rs` `ResourceObservation`  | none (immutable)  |
| `ServiceEvidence`        | value  | Actual boot/timestamp/state observations                                                              | `src/domain/resource_observation.rs` `ServiceEvidence`      | none (immutable)  |
| `VerificationEvidence`   | value  | Sanitized retained facts, not generic proof of success                                                | `src/domain/operation.rs` `VerificationEvidence`            | none (immutable)  |
| `TransferState`          | enum   | Evidence-qualified transfer lifecycle                                                                 | `src/domain/operation.rs` `TransferState`                   | none (immutable)  |
| `Transfer`               | entity | One payload workflow and metadata                                                                     | `src/domain/operation.rs` `Transfer`                        | OperationEngine   |
| `TransferProgress`       | value  | Actual byte progress observed                                                                         | `src/domain/operation.rs` `TransferProgress`                | none (immutable)  |
| `OperationIntent`        | value  | Immutable request metadata                                                                            | `src/domain/operation.rs` `OperationIntent`                 | OperationEngine   |
| `Operation`              | entity | One structured request’s folded lifecycle                                                             | `src/domain/operation.rs` `Operation`                       | OperationEngine   |
| `OperationPlan`          | value  | Immutable target/impact/scopes/revision                                                               | `src/domain/operation.rs` `OperationPlan`                   | OperationEngine   |
| `PlanResult`             | enum   | Confirmable plan or explicit blocked request                                                          | `src/domain/operation.rs` `PlanResult`                      | none (immutable)  |
| `OperationDecision`      | enum   | Exact-revision confirmation/rejection                                                                 | `src/domain/operation.rs` `OperationDecision`               | none (immutable)  |
| `OperationFact`          | enum   | Append-only lifecycle/evidence facts                                                                  | `src/domain/operation.rs` `OperationFact`                   | none (immutable)  |
| `OperationRecorded`      | event  | A permanently recorded Operation fact                                                                 | `src/domain/operation.rs` `OperationRecorded`               | OperationEngine   |
| `OperationQuery`         | enum   | Audit queries                                                                                         | `src/domain/operation.rs` `OperationQuery`                  | none (immutable)  |
| `OperationHistory`       | value  | Folded Operations and retained facts                                                                  | `src/domain/operation.rs` `OperationHistory`                | none (immutable)  |
| `OperationChanged`       | event  | A durable Operation observation changed                                                               | `src/domain/operation.rs` `OperationChanged`                | OperationEngine   |
| `OperationEngine`        | entity | Sole Operation/Transfer semantic writer                                                               | `src/application/operations.rs` `OperationEngine`           | OperationEngine   |
| `EnabledMutation`        | enum   | No approved mutation cases yet                                                                        | `src/application/operations.rs` `EnabledMutation`           | OperationEngine   |
| `LocalTrustApproval`     | enum   | No approved local specialization yet                                                                  | `src/application/operations.rs` `LocalTrustApproval`        | OperationEngine   |
| `ExecutionPermit`        | value  | Single-dispatch confirmed/write-ahead/lease capability                                                | `src/application/operations.rs` `ExecutionPermit`           | OperationEngine   |
| `LocalTrustEnvelope`     | value  | E4-authorized local admission and paired audit identity                                               | `src/application/operations.rs` `LocalTrustEnvelope`        | OperationEngine   |
| `RemoteDispatch`         | value  | Typed admitted remote execution                                                                       | `src/application/operations.rs` `RemoteDispatch`            | OperationEngine   |
| `TransferDispatch`       | value  | Typed admitted separate-payload execution                                                             | `src/application/operations.rs` `TransferDispatch`          | OperationEngine   |
| `ReconciliationRead`     | value  | Approved read-only evidence request                                                                   | `src/application/operations.rs` `ReconciliationRead`        | OperationEngine   |
| `ExecutionObservation`   | value  | Transport/process evidence, not semantic verdict                                                      | `src/application/operations.rs` `ExecutionObservation`      | OperationEngine   |
| `WorkJoined`             | value  | Sealed proof that this workflow’s owned work ended                                                    | `src/application/operations.rs` `WorkJoined`                | OperationEngine   |
| `RecoveryReport`         | value  | Actually committed interrupted-state processing                                                       | `src/application/operations.rs` `RecoveryReport`            | OperationEngine   |
| `AuditAppend`            | value  | Sealed append batch with expected tails                                                               | `src/application/operations.rs` `AuditAppend`               | OperationEngine   |
| `AuditStoreRequest`      | enum   | Private audit read/append requests                                                                    | `src/application/operations.rs` `AuditStoreRequest`         | OperationEngine   |
| `AuditStoreResult`       | enum   | Private durable audit results                                                                         | `src/application/operations.rs` `AuditStoreResult`          | OperationEngine   |
| `OperationMessage`       | enum   | Private actor protocol                                                                                | `src/application/operations.rs` `OperationMessage`          | OperationEngine   |
| `ExecutionPermitView`    | value  | Read-only dispatch admission binding                                                                  | `src/application/operations.rs` `ExecutionPermitView`       | OperationEngine   |
| `RemoteDispatchView`     | value  | Read-only admitted remote request                                                                     | `src/application/operations.rs` `RemoteDispatchView`        | OperationEngine   |
| `TransferDispatchView`   | value  | Read-only admitted upload/write payload                                                               | `src/application/operations.rs` `TransferDispatchView`      | OperationEngine   |
| `ReconciliationReadView` | value  | Read-only evidence request                                                                            | `src/application/operations.rs` `ReconciliationReadView`    | OperationEngine   |
| `LocalTrustEnvelopeView` | value  | Read-only atomic audit admission                                                                      | `src/application/operations.rs` `LocalTrustEnvelopeView`    | OperationEngine   |
| `AuditBatchData`         | value  | Consumed authorized persistence batch                                                                 | `src/application/operations.rs` `AuditBatchData`            | OperationEngine   |
| `WorkJoinedView`         | value  | Read-only lease-release evidence                                                                      | `src/application/operations.rs` `WorkJoinedView`            | OperationEngine   |
| `RecoveryReportView`     | value  | Read-only durable startup recovery result                                                             | `src/application/operations.rs` `RecoveryReportView`        | OperationEngine   |
| `OperationQuiesced`      | value  | Completed work/lease-release IDs                                                                      | `src/application/operations.rs` `OperationQuiesced`         | OperationEngine   |
| `OwnedWorkJoined`        | value  | Worker join results pending owner validation                                                          | `src/application/operations.rs` `OwnedWorkJoined`           | OperationEngine   |
| `OperationCompletion`    | enum   | Stop/release continuation                                                                             | `src/application/operations.rs` `OperationCompletion`       | OperationEngine   |
| `DownloadRequest`        | value  | Explicit remote-read/local-destination intent                                                         | `src/domain/operation.rs` `DownloadRequest`                 | none (immutable)  |
| `DownloadProofApproval`  | enum   | Closed unavailable download-proof gate                                                                | `src/application/operations.rs` `DownloadProofApproval`     | OperationEngine   |
| `DownloadAdmission`      | value  | Sealed read-only transfer admission                                                                   | `src/application/operations.rs` `DownloadAdmission`         | OperationEngine   |
| `DownloadDispatch`       | value  | Admitted separate download payload                                                                    | `src/application/operations.rs` `DownloadDispatch`          | OperationEngine   |
| `DownloadDispatchView`   | value  | Read-only exact download binding                                                                      | `src/application/operations.rs` `DownloadDispatchView`      | OperationEngine   |
| `AuditTail`              | value  | Committed event UUID and payload ordinal pair                                                         | `src/domain/operation.rs` `AuditTail`                       | OperationEngine   |
| `ReconciliationBatch`    | value  | Persisted results from one connection-triggered read batch                                            | `src/application/operations.rs` `ReconciliationBatch`       | OperationEngine   |
| `Store`                  | value  | Foreign Store boundary; declaration ADR-0006:01, runtime producer in its owning epic                  | `src/infrastructure/storage/mod.rs` `Store`                 | Store             |
| `Configuration`          | value  | Foreign Configuration boundary; declaration ADR-0006:01, runtime producer in its owning epic          | `src/application/configuration.rs` `Configuration`          | Configuration     |
| `ConnectionIdentity`     | value  | Foreign ConnectionIdentity boundary; declaration ADR-0006:01, runtime producer in its owning epic     | `src/domain/connection_identity.rs` `ConnectionIdentity`    | Configuration     |
| `ResolvedConnection`     | value  | Foreign ResolvedConnection boundary; declaration ADR-0006:01, runtime producer in its owning epic     | `src/application/configuration.rs` `ResolvedConnection`     | Configuration     |
| `TrustExecutor`          | value  | Foreign TrustExecutor boundary; declaration ADR-0006:01, runtime producer in its owning epic          | `src/application/trust.rs` `TrustExecutor`                  | TrustCore         |
| `TrustChange`            | enum   | Foreign TrustChange boundary; declaration ADR-0006:01, runtime producer in its owning epic            | `src/domain/trust.rs` `TrustChange`                         | TrustCore         |
| `PreparedTrustChange`    | value  | Foreign PreparedTrustChange boundary; declaration ADR-0006:01, runtime producer in its owning epic    | `src/application/trust.rs` `PreparedTrustChange`            | TrustCore         |
| `TrustCommitObservation` | value  | Foreign TrustCommitObservation boundary; declaration ADR-0006:01, runtime producer in its owning epic | `src/application/trust.rs` `TrustCommitObservation`         | Store             |
| `ConsumerAccess`         | value  | Foreign ConsumerAccess boundary; declaration ADR-0006:01, runtime producer in its owning epic         | `src/application/connection_manager.rs` `ConsumerAccess`    | ConnectionManager |
| `ConsumerTarget`         | enum   | Foreign ConsumerTarget boundary; declaration ADR-0006:01, runtime producer in its owning epic         | `src/application/connection_manager.rs` `ConsumerTarget`    | ConnectionManager |
| `Consumer`               | value  | Foreign Consumer boundary; declaration ADR-0006:01, runtime producer in its owning epic               | `src/application/connection_manager.rs` `Consumer`          | ConnectionManager |
| `ChannelEnd`             | value  | Foreign ChannelEnd boundary; declaration ADR-0006:01, runtime producer in its owning epic             | `src/application/connection_manager.rs` `ChannelEnd`        | ConnectionManager |
| `Connection`             | value  | Foreign Connection boundary; declaration ADR-0006:01, runtime producer in its owning epic             | `src/domain/connection.rs` `Connection`                     | ConnectionManager |
| `Capability`             | value  | Foreign Capability boundary; declaration ADR-0006:01, runtime producer in its owning epic             | `src/domain/connection.rs` `Capability`                     | ConnectionManager |
| `DispatchKnowledge`      | enum   | Foreign DispatchKnowledge boundary; declaration ADR-0006:01, runtime producer in its owning epic      | `src/application/connection_manager.rs` `DispatchKnowledge` | ConnectionManager |
| `Failure`                | value  | Foreign Failure boundary; declaration ADR-0006:01, runtime producer in its owning epic                | `src/domain/error.rs` `Failure`                             | none (immutable)  |
| `Availability`           | enum   | Foreign Availability boundary; declaration ADR-0006:01, runtime producer in its owning epic           | `src/domain/error.rs` `Availability`                        | none (immutable)  |
| `Subscription`           | value  | Foreign Subscription boundary; declaration ADR-0006:01, runtime producer in its owning epic           | `src/domain/error.rs` `Subscription`                        | none (immutable)  |
| `LockAuthority`          | value  | Foreign LockAuthority boundary; declaration ADR-0006:01, runtime producer in its owning epic          | `src/application/runtime.rs` `LockAuthority`                | Runtime           |
| `ResourceLease`          | value  | Foreign ResourceLease boundary; declaration ADR-0006:01, runtime producer in its owning epic          | `src/application/runtime.rs` `ResourceLease`                | Runtime           |
| `StartupOwnership`       | value  | Foreign StartupOwnership boundary; declaration ADR-0006:01, runtime producer in its owning epic       | `src/application/runtime.rs` `StartupOwnership`             | Runtime           |
| `LocalTrustPlan`         | value  | Sealed exact local trust review revision with original E3 prepared delta                              | `src/application/operations.rs` `LocalTrustPlan`            | OperationEngine   |
| `LocalTrustAdmitted`     | value  | One local trust delta paired with E4 write-ahead admission; lease retained by engine                  | `src/application/operations.rs` `LocalTrustAdmitted`        | OperationEngine   |

### Functions

### `OperationEngine::spawn(store: E6::Store, configuration: E2::Configuration, trust: E3::TrustExecutor, locks: E6::LockAuthority) -> OperationEngine`

- **Intent.** Establish an actual audit/recovery and policy-gated Operation owner without requiring SSH bootstrap.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** Store, Configuration, TrustExecutor, LockAuthority.
- **Writes.** OperationEngine.
- **Preconditions.** Primary Store contains real compatible operations/events tables.
- **Postconditions.** Audit queries and startup recovery work immediately; remote methods explicitly lack transport until bound; no mutation or fabricated empty-store shortcut.
- **Invariants.** INV-1, INV-5, INV-8, INV-10.
- **Errors.** Subsequent real Store failures remain StorageError.
- **Idempotency.** Once per primary; construction neither clears audit nor resumes work.
- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::bind_transport(&self, consumers: E1::ConsumerAccess) -> Result<(), E6::Failure>`

- **Intent.** Make real remote reads available after the connection owner exists.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** ConsumerAccess.
- **Writes.** OperationEngine’s transport capability.
- **Preconditions.** Same primary generation; not previously bound to a different owner.
- **Postconditions.** Remote requests use this E1 capability; no connection or mutation starts during binding.
- **Invariants.** INV-1.
- **Errors.** ConfigurationConflict for different/replaced binding leaves original intact.
- **Idempotency.** Rebinding the same capability is harmless; never creates a second transport.
- **Introducing ticket.** ADR-0004:06.

### `OperationEngine::inspect_resources(&self, query: ResourceQuery) -> Result<ResourceObservation, E6::Failure>`

- **Intent.** Read required service details or file-opening evidence independently of mutation privileges.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** ResourceQuery, E1 Connection/Capability.
- **Writes.** None administrative.
- **Preconditions.** Typed/bounded input, bound transport, current usable master and relevant read Capability.
- **Postconditions.** Structured service mechanisms only; text is UTF-8 ≤1 MiB with mtime/size/hash snapshot; returned context stays explicit. Action availability is per verb/unit, exact permission and policy.
- **Invariants.** INV-1, INV-13.
- **Errors.** Validation, permission, unavailable Capability/policy or transport failure is not empty/success; no privileged fallback.
- **Idempotency.** Repeats read current state without mutation; reads can remain usable while all mutations are unavailable.
- **Introducing ticket.** ADR-0004:06.

### `OperationEngine::plan_operation(&self, request: OperationRequest) -> Result<PlanResult, E6::Failure>`

- **Intent.** Present an executable change only when its exact target, authority and evidence policy are complete.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** OperationRequest, E2 resolution, current resources, exact privilege and per-type policy; E3 prepared trust semantics where applicable.
- **Writes.** Operation, OperationRecorded.
- **Preconditions.** Explicit user intent and Server context; valid typed arguments.
- **Postconditions.** Local Trust requests delegate the same prepare_local_trust stage and retain its sealed plan; no parallel trust policy/workflow. Records Requested and validation honestly. Missing policy records a blocked/no-effect outcome and required decisions, never a confirmable dispatch. Complete applicable policy is required before Planning/AwaitingConfirmation and immutable revision/impact/scopes/acknowledgements.
- **Invariants.** INV-1, INV-5, INV-8, INV-10.
- **Errors.** Storage failure returns no false durable request; validation/permission/external-change/Capability/policy failures perform no mutation and retain any already-recorded facts.
- **Idempotency.** Each explicit new request creates one Operation; redelivery is not a new intent. Crash before dispatch cannot execute an unconfirmed plan.
- **Introducing ticket.** ADR-0004:07.

### `OperationEngine::decide_operation(&self, operation_id: Uuid, decision: OperationDecision) -> Result<Operation, E6::Failure>`

- **Intent.** Consume or reject the exact reviewed plan without duplicate dispatch.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** OperationPlan, decision, fresh assumptions/permission, E6 lease, per-type policy.
- **Writes.** Operation, OperationRecorded, Transfer.
- **Preconditions.** Matching AwaitingConfirmation revision; all required acknowledgements; admission open; complete applicable policy; atomic identity/scope lease acquisition succeeds.
- **Postconditions.** Reject before effects records Cancelled. Confirm rechecks assumptions and authority, commits Running, then permits one dispatch; execution enters Verifying; only approved proof yields Succeeded/Failed, otherwise Unknown. Local trust delegates the same admit_local_trust stage, consumes LocalTrustAdmitted into its original delta/envelope, invokes actual E3 commit, and settles approved local outcome through the canonical completion path; no second trust workflow.
- **Invariants.** INV-1, INV-2, INV-4, INV-5, INV-8, INV-10, INV-11.
- **Errors.** Stale revision/assumptions, conflict, missing acknowledgement/policy/permission or failed write-ahead causes no dispatch. After possible effects, errors preserve uncertainty; failed final persistence never claims a durable outcome.
- **Idempotency.** Consumed revision cannot dispatch again; duplicate returns recorded result/conflict. Crash after Running is recovered Unknown, never retried.
- **Introducing ticket.** ADR-0004:08.

### `OperationEngine::inspect_operations(&self, query: OperationQuery) -> Result<OperationHistory, E6::Failure>`

- **Intent.** Show permanent intent, progress, evidence and unresolved outcomes.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** OperationIntent, OperationRecorded, Transfer metadata.
- **Writes.** None.
- **Preconditions.** Valid bounded query where a limit applies.
- **Postconditions.** Folds actual facts; Recent includes Unknown; empty is distinct from storage failure.
- **Invariants.** INV-5, INV-8.
- **Errors.** ValidationError/StorageError changes no audit or remote state.
- **Idempotency.** Read-only across repeated calls/restarts.
- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::cancel_operation(&self, operation_id: Uuid) -> Result<Operation, E6::Failure>`

- **Intent.** Stop owned work without confusing a stop request with absence of effects.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** Operation, dispatch/effect evidence, owned work.
- **Writes.** Operation, Transfer, OperationRecorded.
- **Preconditions.** Operation exists.
- **Postconditions.** Records cancellation request, prevents further dispatch, stops relevant consumers and joins execution/verification/cleanup. Cancelled requires proof of no effects; possible effects with insufficient evidence become Unknown; lease lasts until all owned conflicting work ends.
- **Invariants.** INV-2, INV-12, INV-5.
- **Errors.** Cleanup/storage/verification failure preserves active ownership and uncertainty; dialog dismissal is not completion.
- **Idempotency.** Repeated calls join cancellation or return recorded disposition; never dispatch/retry/rollback.
- **Introducing ticket.** ADR-0004:10.

### `OperationEngine::reconcile_operation(&self, operation_id: Uuid) -> Result<Operation, E6::Failure>`

- **Intent.** Investigate Unknown using the original type’s approved read-only predicate.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** Unknown Operation, retained evidence, current read-only observations, approved predicate.
- **Writes.** Operation, OperationRecorded.
- **Preconditions.** Unknown; complete applicable Reconciliation policy; remote case has current gated connection. Local trust has its separately approved local rule, not a remote reconnect requirement.
- **Postconditions.** Appends proved Succeeded/Failed or still Unknown; holds appropriate lease through owned work; executes no mutation or rollback.
- **Invariants.** INV-1, INV-12, INV-5.
- **Errors.** Missing policy, lost connection, reboot-invalidated/insufficient evidence leaves Unknown and explains why; storage failure does not claim recorded resolution.
- **Idempotency.** Repeated reads may append new evidence, never repeat the change.
- **Introducing ticket.** ADR-0004:10.

### `OperationEngine::observe(&self) -> Result<E6::Subscription<OperationHistory, OperationChanged>, E6::Failure>`

- **Intent.** Deliver durable Operation progress to responsive presentation.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** OperationHistory.
- **Writes.** None.
- **Preconditions.** Primary actor exists.
- **Postconditions.** Snapshot and subsequent events share an observation boundary; committed state only is described as durable.
- **Invariants.** INV-5.
- **Errors.** Store/owner failure is explicit; no false empty history.
- **Idempotency.** Subscription never requests mutation or replay.
- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::recover_interrupted(&self, startup: &E6::StartupOwnership) -> Result<RecoveryReport, E6::Failure>`

- **Intent.** Durably expose prior Running/Verifying work as unresolved before new admission.
- **Owner.** `src/application/operations.rs` OperationEngine.
- **Reads.** StartupOwnership and actual folded audit.
- **Writes.** Operation, OperationRecorded.
- **Preconditions.** Current primary holds startup; remote dispatch remains closed.
- **Postconditions.** Appends interruption facts for every Running/Verifying Operation, yielding Unknown/reconciliation-needed with original intent/evidence retained; returns actual committed IDs.
- **Invariants.** INV-3, INV-5, INV-8, INV-10.
- **Errors.** Read/append/migration/storage failure returns StorageError and no recovery-complete report.
- **Idempotency.** Already Unknown remains unresolved without another interruption conversion; repeated starts never erase/retry it.
- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::quiesce(&self) -> Result<OperationQuiesced, E6::Failure>`

- **Intent.** End administrative ownership and release its actual leases before transport shutdown.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** Operation, Transfer, expected work sets, ResourceLease, termination and persistence results.
- **Writes.** Operation, Transfer, OperationRecorded; E6 alone removes leases.
- **Preconditions.** New Runtime admission is closed.
- **Postconditions.** Stored waiters and stop/join workers keep the mailbox serviced. The owner validates complete consumer/task coverage, settles conservative outcomes, creates WorkJoined, moves it with ResourceLease into a release worker, and waits for LeaseReleased. OperationQuiesced contains ordinary completed IDs only after acknowledgements.
- **Invariants.** INV-2, INV-5, INV-10, INV-11, INV-12.
- **Errors.** Unjoined work, persistence failure or rejected/incomplete release leaves quiescence incomplete and identifies the Operation/resource; no forged joined proof.
- **Idempotency.** Calls join one pending quiescence; stale/duplicate completions cannot release another lease.

- **Introducing ticket.** ADR-0004:11.

### `OperationEngine::reconcile_connected(&self, connection: E1::Connection) -> Result<ReconciliationBatch, E6::Failure>`

- **Intent.** Investigate eligible unresolved remote work after a current gated connection becomes usable.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** Connected Connection, actual NeedingReconciliation history, immutable recorded identity and approved per-type read policy.
- **Writes.** Operation and OperationRecorded through the existing read-only Reconciliation workflow.
- **Preconditions.** Observation is Connected with a generation; E1 rechecks current admission and original identity when each consumer opens.
- **Postconditions.** Select only matching recorded identity/Server remote Operations; schedule the same reconcile_operation behavior through workers while servicing mailbox. Local trust is excluded; missing policies stay Unknown in unavailable. No Connect or mutation occurs.
- **Invariants.** INV-1, INV-5, INV-8, INV-10, INV-12.
- **Errors.** Stale/lost/retargeted connection prevents reads and retains Unknown; Store failure never claims durable resolution.
- **Idempotency.** Coalesce the same Operation/generation; new connected generations or explicit reads can collect evidence but never replay a mutation.

- **Introducing ticket.** ADR-0004:10.

### `RemoteDispatch::inspect_remote_dispatch(&self) -> RemoteDispatchView<'_>`

- **Intent.** Read the exact admitted typed remote request without issuing authority.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** RemoteDispatch
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `TransferDispatch::inspect_transfer_dispatch(&self) -> TransferDispatchView<'_>`

- **Intent.** Read the admitted upload/write payload binding.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** TransferDispatch
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `ReconciliationRead::inspect_reconciliation_read(&self) -> ReconciliationReadView<'_>`

- **Intent.** Read the original Operation and approved read-only policy reference.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** ReconciliationRead
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

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

### `WorkJoined::inspect_joined_work(&self) -> WorkJoinedView<'_>`

- **Intent.** Read the actual lease/Operation/joined-work IDs for E6 release validation.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** WorkJoined
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `RecoveryReport::inspect_recovery(&self) -> RecoveryReportView<'_>`

- **Intent.** Read the actually committed startup recovery result.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** RecoveryReport
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `DownloadDispatch::inspect_download_dispatch(&self) -> DownloadDispatchView<'_>`

- **Intent.** Read the specifically admitted read-only download without receiving mutation authority.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** DownloadDispatch
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact immutable expected ConnectionIdentity plus existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::prepare_local_trust(&self, request: OperationRequest) -> Result<LocalTrustPlan, E6::Failure>`

- **Intent.** Review the exact stored-state local trust change without requiring remote transport.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** OperationRequest, Configuration resolution and E3 PreparedTrustChange.
- **Writes.** Operation, OperationRecorded, LocalTrustPlan.
- **Preconditions.** Explicit Trust request; accepted E4:01 local specialization and E3 semantic validation; no remote prerequisites.
- **Postconditions.** Use actual stored keys/preconditions and E3 prepare_change to persist Requested/Validating/Planning/AwaitingConfirmation and return sealed exact plan/prepared delta. Present Server/Host Key identity, impact and all applicable acknowledgements. Missing policy blocks without admission.
- **Invariants.** INV-1, INV-5, INV-8, INV-10.
- **Errors.** Validation, unsupported policy, stored-key/revision conflict or Store failure admits no change; preserve already durable facts.
- **Idempotency.** One explicit request creates one local Operation; no blind redelivery or remote action.

- **Introducing ticket.** ADR-0004:05.

### `LocalTrustPlan::inspect_local_trust_plan(&self) -> &OperationPlan`

- **Intent.** Present the sealed local trust plan without exposing its authority constructor.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** LocalTrustPlan.
- **Writes.** None.
- **Preconditions.** A live engine-created LocalTrustPlan exists.
- **Postconditions.** Borrow the exact plan; no prepared delta/envelope is fabricated or cloned.
- **Invariants.** INV-1, INV-5.
- **Errors.** Infallible borrowed read.
- **Idempotency.** Repeatable, receiver remains owned and borrowed view cannot outlive it.

- **Introducing ticket.** ADR-0004:05.

### `OperationEngine::admit_local_trust(&self, plan: LocalTrustPlan, decision: OperationDecision) -> Result<LocalTrustAdmitted, E6::Failure>`

- **Intent.** Admit one exact confirmed local trust change after real lease and durable write-ahead.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** LocalTrustPlan, OperationDecision, fresh stored preconditions, ResourceLease and actual AuditTail.
- **Writes.** Operation, OperationRecorded, LocalTrustAdmitted, LocalTrustEnvelope; engine retains active lease.
- **Preconditions.** Approved local trust policy, current exact revision, complete consent and Ready lock admission; same current stored preconditions.
- **Postconditions.** Acquire the real identity/scopes lease, commit Running before key effects, reserve exact next committed-change event UUID/ordinal and return original PreparedTrustChange paired with LocalTrustEnvelope. Engine retains the lease/active local-work record; no trust write happens here.
- **Invariants.** INV-1, INV-2, INV-5, INV-8, INV-10.
- **Errors.** Reject/missing consent/stale revision/key conflict/lease conflict/write-ahead failure returns no envelope and no key change; settle any acquired ownership honestly.
- **Idempotency.** A revision is consumed at most once; ambiguous delivery requires actual audit read, never duplicate admission.

- **Introducing ticket.** ADR-0004:05.

### `LocalTrustAdmitted::into_local_trust_admitted(self) -> (E3::PreparedTrustChange, LocalTrustEnvelope)`

- **Intent.** Move the original prepared local delta and its paired admission to the trust executor.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** LocalTrustAdmitted.
- **Writes.** None; moves existing data.
- **Preconditions.** A live engine-created admitted value exists.
- **Postconditions.** Move the exact PreparedTrustChange and LocalTrustEnvelope once; engine retains its actual lease until settled completion.
- **Invariants.** INV-2, INV-8.
- **Errors.** Infallible; no authority allocation.
- **Idempotency.** Single consumption; the moved receiver cannot be used twice.

- **Introducing ticket.** ADR-0004:05.

### `ExecutionPermit::inspect_execution_permit(&self) -> ExecutionPermitView<'_>`

- **Intent.** Read the admitted operation/revision/write-ahead/lease binding.
- **Owner.** `src/application/operations.rs` OperationEngine
- **Reads.** ExecutionPermit
- **Writes.** None.
- **Preconditions.** A live owner-created receiver exists.
- **Postconditions.** Returns the exact immutable expected ConnectionIdentity plus existing fields; no authority-accepting boundary accepts the view instead of its wrapper. Borrowed fields cannot outlive the receiver.
- **Invariants.** INV-5, INV-8, INV-14.
- **Errors.** Infallible; no I/O or new authority allocation.
- **Idempotency.** Repeatable pure inspection.

- **Introducing ticket.** ADR-0004:04.

### `OperationEngine::finish_local_trust(&self, operation_id: Uuid, result: Result<E3::TrustCommitObservation, E6::Failure>) -> Result<Operation, E6::Failure>`

- **Intent.** Settle one registered local trust commit after actual worker join and release its retained lease.
- **Owner.** `src/application/operations.rs` private local_trust child workflow
- **Reads.** Registered operation/worker/work-set identity, actual E3 commit result, current persisted audit/tail and retained ResourceLease.
- **Writes.** Operation, OperationRecorded and engine local ownership; delegates removal to E6.
- **Preconditions.** Only the registered E4-owned local commit worker may invoke after the actual commit future/local work join; operation ID matches retained active admission.
- **Postconditions.** Validate actual tracked complete work set and Store settlement; evaluate approved local persisted-state proof. Sufficient evidence yields Succeeded/Failed; insufficient evidence yields actual Unknown, never assumed no effects. After settled conservative persistence, create WorkJoined and release retained lease through the still-serviced mailbox; no local active lease is abandoned.
- **Invariants.** INV-1, INV-2, INV-5, INV-8, INV-10, INV-15.
- **Errors.** Caller-supplied Error is not proof of join. Missing/mismatched worker, unjoined work or unsettled Store returns incomplete ownership without release; ambiguous commit is reread from actual Store and stays Unknown unless proved.
- **Idempotency.** Duplicate matching completion returns retained disposition; cannot release another lease or execute/commit again.

- **Introducing ticket.** ADR-0004:05.

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

| ID     | Statement                                                                                                                                           | Scope                                                                                           | Enforced by                                                                    | Broken if                                                           | Proof                                                                          |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| INV-1  | No mutation dispatches without its own complete applicable Verifier/Reconciliation/authority policy.                                                | OperationEngine, Operation                                                                      | Closed unavailable gates until approved per-type contract                      | Exit-zero/generic policy enables a missing type                     | First proof ADR-0004:04: `missing_type_policy_blocks_every_action`             |
| INV-2  | One confirmed revision, active lease and committed Running fact precede one mutation dispatch.                                                      | Operation, ExecutionPermit, LocalTrustEnvelope                                                  | Private non-cloneable permit and transactional expected-tail write-ahead       | UI bypass or ambiguous dispatch is retried                          | First proof ADR-0004:05: `confirmed_revision_write_ahead_single_dispatch`      |
| INV-3  | Prior Running or Verifying Operations become durable Unknown with retained evidence before startup admission.                                       | Operation, RecoveryReport                                                                       | Actual audit query and atomic interruption append                              | Missing table/error is treated as empty or old execution resumes    | First proof ADR-0004:04: `interrupted_audit_recovery_is_durable_unknown`       |
| INV-4  | Remote mutation execution uses fixed typed literal login-user scripts and exact granular sudo only.                                                 | RemoteDispatch, OperationEngine                                                                 | Private fixed templates, one encoder and exact command/argument checks         | Interpolation, sudo shell/password or polkit fallback               | First proof ADR-0004:08: `literal_arguments_and_exact_sudo_authority`          |
| INV-5  | Intent and facts are permanent sanitized append-only records; current state is derived.                                                             | Operation, OperationRecorded, Transfer, OperationEngine                                         | Immutable rows, database guards and sanitized deterministic fold               | Prior evidence is overwritten or file/secret content persisted      | First proof ADR-0004:04: `audit_is_permanent_append_only_and_sanitized`        |
| INV-6  | File mutation cannot silently change target, overwrite consent, privilege or opening assumptions.                                                   | FileVersion, TextDocument, Transfer                                                             | Fresh per-type path/effect checks and revision-bound consent                   | Lock or stale lstat substitutes for safety                          | First proof ADR-0004:09: `file_external_version_and_path_promotion_safety`     |
| INV-7  | GTK alone owns widgets and presents honest loading, empty, stale, error and progress states.                                                        | OperationEngine                                                                                 | Main-context channel presentation                                              | Worker touches widgets or stale becomes current                     | First proof ADR-0004:11: `operations_gtk_keyboard_states_and_responsiveness`   |
| INV-8  | Per-Operation audit ordinals start at 1 and remain contiguous independently of physical database order.                                             | OperationRecorded, AuditAppend, AuditTail                                                       | Sole E4 allocator; transactional expected-tail and payload-ordinal checks      | Rowid/time/UUID orders lifecycle or ambiguous append skips position | First proof ADR-0004:04: `audit_payload_ordinals_tail_conflict_and_reload`     |
| INV-9  | Downloads use read-only typed admission, never mutation admission, and require their own complete proof policy.                                     | DownloadAdmission, DownloadDispatch, Transfer                                                   | Private separate gate and transport variant                                    | Removing mutation gate enables unverified completion                | First proof ADR-0004:09: `download_read_only_admission_and_missing_proof`      |
| INV-10 | Pending external work never prevents the Operation owner from processing its mailbox and completion messages.                                       | OperationEngine, OperationCompletion                                                            | Stored waiters, owned immutable-input workers and owner-only state transitions | Handler awaits a worker that needs its mailbox                      | First proof ADR-0004:04: `operation_mailbox_progress_during_pending_work`      |
| INV-11 | Insufficient effect evidence yields Unknown; successful process termination alone never establishes success.                                        | Operation, Transfer                                                                             | Evidence-qualified transitions and per-type proof                              | SSH exit, lock release or missing proof becomes semantic verdict    | First proof ADR-0004:08: `service_effect_evidence_unknown_boundary`            |
| INV-12 | Cancellation retains leases until all tracked work joins; Reconciliation performs reads only and never replays a mutation.                          | Operation, Transfer, WorkJoined                                                                 | Tracked ID coverage, sealed joined proof and read-only request type            | Stop intent becomes joined proof or reconnection repeats a mutation | First proof ADR-0004:10: `cancel_reconcile_joined_work_and_no_replay`          |
| INV-13 | Read observations retain explicit Server and opening/current evidence without requiring mutation permission.                                        | ResourceObservation, TextDocument, FileVersion                                                  | Typed bounded read protocol and actual SFTP opening hash                       | Denied mutation hides reads or stale opening evidence is invented   | First proof ADR-0004:06: `services_and_text_reads_are_independent_and_bounded` |
| INV-14 | Read projections and consuming decomposition reveal exact existing data but cannot mint positive authority.                                         | ExecutionPermit, LocalTrustEnvelope, AuditAppend, WorkJoined, RecoveryReport, DownloadAdmission | Private wrapper fields and authority-accepting sealed inputs only              | A view or observation substitutes for wrapper authority             | First proof ADR-0004:04: `operation_projection_privacy_and_exact_binding`      |
| INV-15 | Only registered actual local commit work completion and settled persisted evidence can finish a local trust outcome and release its retained lease. | LocalTrustAdmitted, Operation, WorkJoined                                                       | Private finisher, worker/work-set binding and actual Store read-back           | Caller Error or DTO is treated as joined/no-effect proof            | First proof ADR-0004:05: `local_trust_commit_joined_outcome_and_release`       |

### Ordered flows

1. **History and startup recovery (Usage 1/3; tickets 04).**
   1. External current primary supplies actual E6 Store and LockAuthority (ADR-0006:01), E2 Configuration (ADR-0002:01) and E3 TrustExecutor (ADR-0003:01). `OperationEngine::spawn` creates one audit owner without SSH. Constructor LockAuthority cannot acquire until ADR-0006:03 implements arbitration.
   2. External bounded OperationQuery → `inspect_operations` reads real intents/events; `observe` establishes the snapshot/event boundary. A successful actual empty query is empty; read/missing-table/migration failure is StorageError.
   3. External StartupOwnership from Runtime::begin → `recover_interrupted` → atomic interrupted Running/Verifying → Unknown facts → `RecoveryReport::inspect_recovery` → E6 Runtime::admit. No remote call is required.
   4. Every append reserves the next per-Operation positions, sanitizes facts, and creates AuditAppend; `AuditAppend::into_store_batch` moves it only after Store accepts the sealed request. Store validates tails, contiguous payload ordinals, UUID uniqueness and FK atomically. Ambiguous completion triggers actual tail/history read before another append.
2. **Local trust review/admission/completion (Usage 2; ticket 05, before remote reads).**
   1. External explicit Trust OperationRequest and actual stored Configuration/E3 semantic preconditions from ADR-0003:03 → `prepare_local_trust` → sealed LocalTrustPlan.
   2. `LocalTrustPlan::inspect_local_trust_plan` exposes exact Server/Host Key/action/impact/revision. External exact OperationDecision supplies all required acknowledgements.
   3. `admit_local_trust` consumes that exact plan, rechecks actual stored preconditions and acquires E6:03 ResourceLease; durable Running precedes LocalTrustAdmitted, while engine retains active lease/work identity. No SSH or E3 commit capability is needed to establish truthful admission.
   4. `LocalTrustAdmitted::into_local_trust_admitted` returns the ORIGINAL PreparedTrustChange plus LocalTrustEnvelope; actual E3:05 commit worker later executes the inseparable SQLite key/read-back/audit transaction. Store uses the five pure projections introduced in ADR-0006:01, including `LocalTrustEnvelope::local_trust_store_view`, only after accepting sealed requests.
   5. Registered E4-owned worker joins actual local work and reports its real commit result → private `finish_local_trust`. Match operation/worker/work sets and actual persisted tail/read-back; an arbitrary Error is never joined proof. Approved local proof establishes Succeeded/Failed, insufficient evidence Unknown; settled conservative persistence precedes WorkJoined and acknowledged real lease release. No fabricated completed mutation is claimed before actual E3:05 exists.
   6. Later public `plan_operation`/`decide_operation` delegate these SAME canonical stages. User-facing mutation UI remains delivered only by public workflow integration in08 and final presentation11.
3. **Read resources (Usage 1; ticket 06).**
   1. Actual E1 ConsumerAccess from ADR-0001:03 → `bind_transport`; external ResourceQuery → `inspect_resources` validates unit/path/limit and opens an existing ConsumerTarget::Server. No Connect or mutation permission prerequisite.
   2. Units/Summary use busctl JSON, Unit/Dependencies use systemctl show, Journal uses journalctl JSON. OpenText uses separate login-user SFTP, actual UTF-8 ≤1 MiB read and mtime/size/hash FileVersion. Capability/read failure is not empty.
   3. Consumer::into_consumer_parts supplies the real receiver; typed observations return to owner via workers while mailbox remains serviced. Presentation later labels stale evidence after disconnect.
4. **Plan and dispatch (Usage 2; tickets 07→08→09).**
   1. External explicit OperationRequest → `plan_operation` → Requested → Validating. Real E2 resolution/current preflight/exact sudo permission and E3 prepare_change provide assumptions. Missing approved per-type policy yields Blocked and no-effect facts, never a confirmable approximate plan.
   2. Approved policy → Planning → immutable OperationPlan revision/impact/scopes → AwaitingConfirmation. Destructive, overwrite, unmanaged, trust replacement/deletion and non-atomic conditions require the exact applicable acknowledgements.
   3. External OperationDecision → `decide_operation`; Reject before effects records Cancelled. Confirm rechecks current assumptions/permission and acquires all E6 identity/scopes atomically; conflict never queues. Changed file version offers reload or a new explicitly reviewed overwrite revision.
   4. Commit Running before effects → ExecutionPermit → `ExecutionPermit::inspect_execution_permit`; `RemoteDispatch::inspect_remote_dispatch` carries a typed fixed-script request over E1 existing-master transport. Only exact commands use sudo -n; the script remains login user. Start/Stop/Restart/Reload/Enable/Disable retain their own applicability and proof policies.
   5. For approved local trust: E3 prepare_change → LocalTrustEnvelope → `LocalTrustEnvelope::local_trust_store_view` only inside Store’s sealed CommitLocalTrust transaction; compare keys/tail, apply keys, read back and append exact committed-event ordinal together. E3 TrustCommitObservation::into_facts supplies actual facts; E4 evaluates approved local proof without SSH/reconnect.
   6. For approved file writes/uploads: `TransferDispatch::inspect_transfer_dispatch` feeds separate SFTP payload, destination-directory temporary file and approved atomic/non-atomic promotion. Current path/symlink/version checks cannot be replaced by locks.
   7. Download is AdministrativeChange::Download, classified ReadOnly: its own approved proof and consent create DownloadAdmission, never ExecutionPermit or EnabledMutation. `DownloadDispatch::inspect_download_dispatch` supplies separate read-only SFTP and explicit local destination. No remote mutation script or administrative blanket lease is granted.
   8. Queued → Transferring → Verifying → Completed only with approved destination evidence; retain size and optional actual hash, metadata only. Service process exit → Verifying; approved type-specific proof yields Succeeded/Failed, otherwise Unknown. Restart occurrence is distinct from health and stabilization.
5. **Cancellation, Reconciliation and shutdown (Usage 3; tickets 10→11).**
   1. External operation ID → `cancel_operation` records stop intent; prevents further dispatch, starts stop/join workers and keeps mailbox active. Confirmed no-effect permits Cancelled; possible effects without proof yields Unknown.
   2. External operation ID → `reconcile_operation`; or Runtime’s independent current gated Connection feed → `reconcile_connected` → identity-matching Unknown remote Operations through the same read-only predicate. `ReconciliationRead::inspect_reconciliation_read` provides only approved reads. Local trust is not sent through remote reconnect.
   3. Coalesce duplicate Operation/generation reads. Original stored identity must match current E1 target; an alias retarget never reconciles on another machine. Missing/invalidated evidence keeps Unknown; no mutation is retried, including idempotent requests.
   4. Owner validates every tracked consumer termination via ChannelEnd::inspect_channel_end and local task join against expected ID sets. Only settled conservative persistence plus empty work sets creates WorkJoined; `WorkJoined::inspect_joined_work` lets E6 validate Release. E4 moves WorkJoined and ResourceLease to the release worker and consumes LeaseReleased completion.
   5. Runtime closes admission → `quiesce` stores waiters and dispatches workers, never awaits them inside owner handling. Complete work joins, durable outcomes and acknowledged releases → ordinary OperationQuiesced IDs. Only then may E6 stop transport and close Store.
   6. `inspect_operations`/`observe` feed main-context GTK updates for loading/empty/progress/error/Unknown; terminal commands remain outside the Audit Trail.

## Decision

- Select Sol’s deep Operation actor and private typed execution; reject global workflow reducers that obscure semantic writers.
- Graft Terra’s read-only download separation and explicit payload ordinals.
- Introduce actual local trust review/admission/completion before remote reads, reusing the same stages in later public workflows.
- Keep immutable audit folding, private identity-bound authority and owner-authored Store requests; accept explicit worker continuations for responsiveness.
- No Monitoring summary, exit status or local lease substitutes for type-specific proof.

## Open questions

1. What local trust lifecycle, atomic persisted-state proof, outcome and read-only Reconciliation rules are approved?
2. What complete Verifier/Reconciliation policy applies to each service verb and supported unit semantics, including restart timestamps, healthy states and stabilization?
3. What complete per-type file/transfer proof, Reconciliation, local overwrite and path-race rules are approved?
