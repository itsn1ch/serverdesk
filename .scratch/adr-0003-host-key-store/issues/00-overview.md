# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`.

## Files

Paths are planned source changes, not implemented APIs. ADR-0006:01 introduces the common declarations and implements `PreparedTrustChange::prepared_trust_store_view` in its canonical trust module; it cannot mint prepared intent or execution admission. E3:03 owns semantic preparation, E3:05 owns commit and the real E4:05 completion path, and E3:06 completes public GTK criteria after ADR-0004:08. Shared module/Cargo declarations are merged with their true producers, never replaced by greeting-only wiring. Human 02 changes the existing glossary/spec only after an explicit recorded decision.

```text
├── src/
│   ├── domain/
│   │   ├── trust.rs [01, 03, 04, 05, 06] Host Key evidence, provenance and Trust State
│   │   └── mod.rs [01] Trust domain module declaration
│   ├── application/
│   │   ├── trust.rs [01, 03, 04, 05, 06] Evidence review, start gate and explicit trust executor
│   │   ├── operations.rs [05] Private E4-owned commit-worker invocation integration
│   │   └── mod.rs [01] Trust application module declaration
│   ├── infrastructure/
│   │   └── storage/
│   │       ├── host_keys.rs [01, 04, 05] Authoritative key reads, atomic snapshots and trust transactions
│   │       └── mod.rs [01, 04, 05] Trust storage adapter integration
│   └── ui/
│       ├── trust.rs [03, 04, 05, 06] GTK evidence, explicit decisions and actionable failures
│       └── mod.rs [03] Trust presentation module declaration
├── tests/
│   ├── trust_workflows.rs [01, 03, 04, 05, 06] Actual key discovery, SQLite, snapshot and atomicity proofs
│   └── trust_gtk.rs [03, 04, 05, 06] Keyboard, uncertainty and trust-decision journeys
├── Cargo.toml [01] Compatible trust workflow dependencies
├── GLOSSARY.md [02] Approved Direct and Jump trust-use terms
└── .scratch/
    └── adr-0003-host-key-store/
        └── spec.md [02] Approved eligibility contract and affected ticket readiness
```

## Architecture

```text
E6 Store -- HostKey [01] --> TrustCore -- TrustPorts [01] --> core/gate/executor
E2 Configuration -- ResolvedConnection [03] --> Trust façade -- query [03] --> SAME TrustCore
TrustCore -- TrustReview/TrustEvidence [03,06] --> GTK trust [03,06]
E1 original reservation + resolved identity -- prepare [04] --> TrustCore -- PreparedStart [04] --> E2 compile
E4 sealed LocalTrustEnvelope + prepared delta -- commit [05] --> E6 Store -- paired facts [05] --> E4
E1 RouteAccess -- optional Some [06] --> Trust façade -- KeyScan [06] --> permitted Jump generation
```

TrustCore is the sole trust semantic writer; three handles and both optional-route façades share its mailbox. E4 authors lifecycle/audit facts and E6 Store physically commits the sealed composition. External source files are inspected only on explicit Adoption/drift requests; no view or discovered key authorizes trust.

## Order

Core/bootstrap: 01.
Eligibility policy: 02 → 04.
Local evidence and pure expected-state preparation: 01 → 03.
Pre-start snapshots: 01 → 03 → 04.
Explicit local changes: 01 → 03 → external ADR-0004:05 → 05 (human ADR-0004:01, actual audit ADR-0004:04 and locks ADR-0006:03 are prerequisites).
Routed review and final GTK trust decisions: 03 → 04 → 05 → 06 (actual route ADR-0001:04 and public execution ADR-0004:08).
01 and human 02 may run in parallel. 04 and 05 can proceed independently once their separate policy/external prerequisites complete.
The actual E4:05 local-admission stages precede 05 and are reused later by public E4 plan/decide. E1:02 consumes completed trust commit/gate for the final trust-to-terminal journey; no reverse whole-engine/master dependency is added.

- 01: Open canonical stored-key evidence and distribute one trust core — local blockers: None; external blockers: ADR-0006:01; status: ready-for-agent.
- 03: Review local trust evidence, exact fingerprints and Adoption drift without acceptance — local blockers: 01; external blockers: ADR-0006:01, ADR-0002:01; status: ready-for-agent.
- 04: Gate reserved starts and regenerate atomic authoritative trust snapshots — local blockers: 01, 02, 03; external blockers: ADR-0006:01, ADR-0001:01; status: needs-info.
- 05: Commit explicitly confirmed trust changes with inseparable permanent audit facts — local blockers: 01, 03; external blockers: ADR-0006:01, ADR-0004:01, ADR-0004:04, ADR-0004:05, ADR-0006:03; status: needs-info.
- 06: Discover routed target keys and complete the public GTK trust-decision journey — local blockers: 03, 04, 05; external blockers: ADR-0006:01, ADR-0001:04, ADR-0004:08; status: needs-info.
- 02: Human policy decision — no local blockers; ready-for-human. It can run beside 01. Every later needs-info remains gated until its named policy is approved; a declaration is never a runtime capability.

## In plain words

First, keep accepted public keys and their evidence in the real database. You can inspect saved evidence, look up a key you already trusted elsewhere, compare an independent fingerprint, and notice later changes without accepting anything automatically. A person must decide exactly which accepted keys may be used for direct and jump connections. Every new connection then needs a fresh, safely replaced trust file, and missing or mismatched trust blocks it before it starts. Accepting, replacing or deleting trust also needs the separately approved local-operation rules, with the key change and permanent audit evidence saved together. A jump server can help discover a target key, but cannot prove who that target is. The screens show progress, uncertainty and failures honestly and work from the keyboard.
