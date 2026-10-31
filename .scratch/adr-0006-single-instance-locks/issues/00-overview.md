# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0006-single-instance-locks/spec.md`, section `# Design`.

## Files

```text
src/
├── application/
│   ├── runtime.rs [01, 02, 03, 04] Primary ownership, admission, arbitration and shutdown
│   ├── operations.rs [01] Shared Operation declarations and pure Store projections
│   ├── connection_manager.rs [01] Shared transport handle and protocol declarations
│   ├── configuration.rs [01] Shared configuration declarations and InventoryCommit projection
│   ├── trust.rs [01] Shared trust declarations and prepared-change projection
│   ├── monitoring.rs [01] Shared monitoring declarations and settings projection
│   └── mod.rs [01] Explicit application module declarations
├── domain/
│   ├── error.rs [01] Actionable resource-specific failures and effect knowledge
│   ├── operation.rs [01] Shared intent, lifecycle and admission records
│   ├── resource_observation.rs [01] Shared service and file observation values
│   ├── connection.rs [01] Shared connection, Capability and Terminal observations
│   ├── connection_identity.rs [01] Shared structural identity declarations
│   ├── inventory.rs [01] Shared Server and Auth Profile records
│   ├── trust.rs [01] Shared Host Key and provenance records
│   ├── metrics.rs [01] Shared Metric Sample and Collector records
│   └── mod.rs [01] Explicit domain module declarations
├── infrastructure/
│   ├── storage/
│   │   └── mod.rs [01, 02, 04] Actual owned SQLite, migrations and typed transactions
│   ├── ssh/
│   │   ├── mod.rs [01] Shared private SSH adapter declarations
│   │   └── config.rs [01] Shared generated-configuration declarations
│   ├── collector/
│   │   └── mod.rs [01] Shared structured Collector record declarations
│   └── mod.rs [01] Explicit infrastructure module declarations
├── ui/
│   ├── runtime.rs [01, 02, 03, 04] GTK ownership, recovery, conflicts and shutdown presentation
│   ├── transport.rs [01] Shared private GTK transport bridge declarations
│   └── mod.rs [01] Explicit presentation module declarations
└── main.rs [01, 02, 04] Real process entry and final launch delegation
Cargo.toml [01, 04] Compatible native desktop, async and SQLite dependencies
tests/
├── runtime_primary_store.rs [01] Actual duplicate-launch and storage ownership proof
├── runtime_recovery_admission.rs [02] Actual orphan and durable recovery barriers
├── runtime_resource_locks.rs [03] Alias conflicts, independent scopes and joined release
└── runtime_desktop_lifecycle.rs [04] Actual GTK launch, reconnect feeds and shutdown
```

## Architecture

```text
GTK primary -- PrimaryOwnership [01] --> actual SQLite Store
Canonical E2/E3/E4/E5 sealed writes -- five pure projections [01] --> Store transactions
E1 CleanupReport + E4 RecoveryReport [02] --> Runtime admission
Runtime -- SAME-generation ReadyOwnership/Arc [02/04] --> actual E1/E5 owners
Operation workflow -- ConnectionIdentity + immutable ResourceScope [03] --> LockAuthority
E4 actual joined work -- WorkJoined [03/04] --> validated lease Release
E1 -- independent real Connection subscription [04] --> Runtime --> E4 read-only recovery
E5/E4 completed quiescence [04] --> E1 drained --> Store closed
```

Canonical declarations are introduced by ADR-0006:01 in the same owner modules, not duplicated or assumed existing. Store owns physical SQLite; each domain owner alone creates its sealed write requests. User choices and actual external observations never count as positive execution authority.

## Order

Shared declarations + five pure canonical Store projections + actual primary/storage:01.
Actual orphan/durable recovery admission:01 →02.
Identity/resource arbitration:02 →03.
Full desktop/separate reconnect feeds/shutdown:02/03 + real foreign owners →04.
01–03 need no mutation policy for their honest behavior. E1, T3, L1 and S are approved; 04 remains needs-info through unresolved file-policy 03 and its E3:06/E4:11 consumers.

- 01: Establish sole primary, compatible SQLite and preadmission runtime.
- 02: Admit only after actual orphan cleanup and durable Unknown recovery.
- 03: Arbitrate identity/resource conflicts and require joined-work release.
- 04: Launch the complete GTK desktop, reconcile reconnects and shut down safely.

## In plain words

First, one copy of the app becomes the owner and opens the real saved data. A second launch cannot create another owner or disturb ongoing work. Before connecting, the app clears its own leftover connections and records interrupted changes as unresolved. It prevents two of its changes from using the same resource at once without blocking unrelated work. Rejected changes never start later on their own. The final desktop listens for real reconnects and investigates uncertain changes by reading only. On exit, it waits for owned work to finish before closing connections and saved data. Recovery, errors and unresolved outcomes stay visible while the desktop remains usable.

## Comments

Policy decision completion does not issue ReadyOwnership, cleanup/recovery reports, trust snapshots, leases or runtime capabilities. L1 supplies an explicit local-only recovery exception after durable interrupted Unknown and actual worker/SQLite settlement; remote operations retain gated reconnect. File policy03 remains independent and all 360 source criteria remain required.
