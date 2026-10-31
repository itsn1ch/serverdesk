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
