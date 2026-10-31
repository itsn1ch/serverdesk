# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0004-verified-operations/spec.md`, section `# Design`.

## Files

```text
.scratch/
└── adr-0004-verified-operations/
    └── spec.md [01, 02, 03] Approved per-type lifecycle and evidence policies
src/
├── application/
│   ├── operations.rs [04, 05, 06, 07, 08, 09, 10, 11] Sole Operation workflow and owned-work tracking
│   └── mod.rs [04] Explicit Operation application module declaration
├── domain/
│   ├── operation.rs [04, 05, 07, 08, 09, 10] Intent, lifecycle, consent and sealed admission
│   ├── resource_observation.rs [06, 07, 09] Structured service and file evidence
│   └── mod.rs [04, 06] Explicit Operation domain module declarations
├── infrastructure/
│   ├── storage/
│   │   └── audit.rs [04, 05, 08, 10] Immutable audit append and ordered event folding
│   ├── systemd/
│   │   └── mod.rs [06, 07, 08, 10] Structured reads, exact sudo and approved service evidence
│   ├── sftp/
│   │   └── mod.rs [06, 09, 10] Separate payload, file preflight and destination evidence
│   └── mod.rs [04, 06] Explicit infrastructure module declarations
└── ui/
    ├── operations.rs [04, 05, 06, 07, 08, 09, 10, 11] GTK consent, progress, outcomes and Audit Trail
    └── mod.rs [04] Operation presentation module declaration
tests/
├── operations_audit.rs [04] Actual immutable audit and interrupted-state proof
├── operations_reads.rs [06] Actual structured Services and bounded file reads
├── operations_planning.rs [07] Target, privilege and revision-bound consent proof
├── operations_services.rs [08] All six verbs with approved per-type verification
├── operations_files.rs [09] Atomic promotion, external conflicts and transfer outcomes
├── operations_reconciliation.rs [10] Cancellation, read-only evidence and no replay
├── operations_gtk_shutdown.rs [11] Actual keyboard, errors and joined shutdown
└── operations_local_trust.rs [05] Local admission, write-ahead and completion ownership
```

## Architecture

```text
E6 Store -- actual audit [04] --> OperationEngine -- OperationHistory [04/11] --> GTK
E3 semantic prepare -- LocalTrustPlan [05] --> exact consent --> LocalTrustAdmitted [05]
E3 actual commit -- registered joined result [05 integration in E3:05] --> private finish --> E6 Release
E1 ConsumerAccess -- ResourceObservation [06] --> OperationEngine
Public request -- OperationPlan [07] --> confirmed exact ExecutionPermit [08] --> E1
DownloadAdmission / TransferDispatch [09] --> E1 SFTP
Runtime -- independent current Connection [10] --> read-only Reconciliation
OperationEngine -- OperationQuiesced [11] --> Runtime shutdown
```

Canonical declarations are introduced by ADR-0006:01 in the same owner modules, not duplicated or assumed existing. Store owns physical SQLite; each domain owner alone creates its sealed write requests. User choices and actual external observations never count as positive execution authority.

## Order

Human decisions01/02/03 may run in parallel.
Actual audit/recovery and early readable wrappers:04.
Local trust precursor:01 + 04 + E3:03/E6:03 →05 (no transport or E3 commit prerequisite).
Remote read branch:04 + E1:03 →06. It needs no service or file mutation policy, but awaits the transport's trust prerequisites.
Public plan/execution/files:05/06 +02/03 →07 →08 →09.
Cancellation/recovery/presentation:08/09 →10 →11.

- 01: Approve local trust lifecycle and evidence policy.
- 02: Approve six service Verifier and Reconciliation policies.
- 03: Approve per-type file and transfer evidence policies.
- 04: Inspect permanent audit and durably recover interrupted Operations.
- 05: Review and admit local trust Operations before effects.
- 06: Inspect Services and bounded file-opening evidence without mutation privilege.
- 07: Plan exact revision-bound changes and required consent.
- 08: Execute all six service verbs after write-ahead and verify outcomes.
- 09: Apply file mutations and independently admit read-only downloads.
- 10: Cancel and reconcile without replay or premature lease release.
- 11: Present actionable Operation states and quiesce owned work.

## In plain words

First, we must agree on how each kind of change can be proven to work. Meanwhile, we can build a permanent record and show interrupted work honestly. Trust changes can be reviewed against saved keys before a connection exists, with their consent and execution record kept together. You can inspect services and open a small text file without permission to change them. Approved service and file changes then show the exact target and impact before you confirm them, while downloading stays read-only on the server. Cancellation and connection loss never pretend a change was undone. Later investigation reads the current state without repeating the change. The screens remain usable while work finishes and the app shuts down safely.
