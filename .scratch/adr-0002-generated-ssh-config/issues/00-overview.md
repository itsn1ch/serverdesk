# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0002-generated-ssh-config/spec.md`, section `# Design`.

## Files

Paths are planned source changes, not implemented APIs. ADR-0006:01 introduces the common declarations and implements `InventoryCommit::inventory_write_view` in its canonical Configuration module; it cannot mint a validated commit. E2 tickets own the remaining listed methods, validated runtime producers and behavior. Shared module/Cargo declarations are merged with that producer, not replaced by greeting-only wiring. Human 02 changes the existing glossary/spec only after an explicit recorded decision.

```text
├── src/
│   ├── domain/
│   │   ├── inventory.rs [01, 03] Server and non-secret Auth Profile validation
│   │   ├── connection_identity.rs [01, 04, 05] Structural endpoint, route and authentication identity
│   │   └── mod.rs [01] Explicit domain module declarations
│   ├── application/
│   │   ├── configuration.rs [01, 03, 04, 05] Inventory, static Import and sealed compilation workflows
│   │   └── mod.rs [01] Explicit application module declarations
│   ├── infrastructure/
│   │   └── ssh/
│   │       ├── config.rs [04, 05] Generated policy and generated-only OpenSSH self-check
│   │       └── mod.rs [04] SSH configuration adapter declaration
│   ├── ui/
│   │   ├── configuration.rs [01, 03, 04, 05] GTK inventory inputs, exclusions and configuration errors
│   │   └── mod.rs [01] Configuration presentation module declaration
│   └── main.rs [01] Read-only inventory integration before final desktop composition
├── tests/
│   ├── configuration_workflows.rs [01, 03, 04, 05] Actual SQLite, static Import and isolated OpenSSH checks
│   └── configuration_gtk.rs [01, 03, 04, 05] Keyboard, loading, empty and error journeys
├── Cargo.toml [01] Compatible inventory workflow dependencies
├── GLOSSARY.md [02] Approved environment vocabulary
└── .scratch/
    └── adr-0002-generated-ssh-config/
        └── spec.md [02] Approved environment contract and affected ticket readiness
```

## Architecture

```text
E6 Store -- InventorySnapshot [01] --> Configuration -- ResolvedConnection [01] --> E3 trust gate
User source/drafts -- ConfigurationChange [03] --> Configuration -- InventoryCommit [03] --> E6 Store
E3 trust gate -- PreparedStart [04] --> E1 -- original resolution/reservation/TrustStart [04] --> Configuration
Configuration -- CompiledConfiguration [04,05] --> E1 private launch/attachments
Configuration -- ConfigurationResult/Failure [03,04,05] --> GTK configuration [03,04,05]
```

Configuration alone validates inventory and seals InventoryCommit; Store physically commits it but cannot accept arbitrary Server vectors. E1 receives only private compiled wrappers; views never grant launch authority. External inputs are saved SQLite rows, explicit user drafts/source files and earlier-epic owner capabilities.

## Order

Saved inventory: 01.
Environment policy: 02 → 03.
Static import: 01 → 03 → 05.
Direct generated policy: 01 → 04.
Registered route: 01 → 03 → 04 → 05 (03 and 04 may proceed in parallel after their real policy/external prerequisites).
Actual terminal/all-consumer/routed journeys: 04 → ADR-0001:02 → ADR-0001:03; 05 → ADR-0001:04. These are downstream integration consumers, not cyclic prerequisites.

- 01: Read saved inventory and resolve complete Connection Identities — local blockers: None; external blockers: ADR-0006:01; status: ready-for-agent.
- 03: Save reference-only settings and import an explicit static whitelist — local blockers: 01, 02; external blockers: ADR-0006:01; status: needs-info.
- 04: Generate and self-check isolated direct-start and attachment policy — local blockers: 01; external blockers: ADR-0006:01, ADR-0001:01, ADR-0003:04; status: needs-info.
- 05: Generate only registered complete Jump Server routes — local blockers: 01, 03, 04; external blockers: ADR-0006:01; status: needs-info.
- 02: Human policy decision — no local blockers; ready-for-human. It can run beside 01. Every later needs-info remains gated until its named policy is approved; a declaration is never a runtime capability.

## In plain words

First, read the saved connection settings without contacting a server. A person must settle the missing environment choices before new servers can be saved or imported. Import copies only supported settings, explains what it skipped, and never runs commands from the source file. The original files stay untouched. Connections then use a newly generated, checked set of settings that cannot borrow commands or trust decisions from your personal SSH files. Registered jump servers are added without hidden routes or a silent direct fallback. Errors, loading and saved results stay clear and usable from the keyboard.
