# 03: Review local trust evidence, exact fingerprints and Adoption drift without acceptance

**Status:** ready-for-agent
**Blocked by:** 01: Open canonical stored-key evidence and distribute one trust core.
**External blockers:** ADR-0006:01, ADR-0002:01.
**Source:** `US-001-AC2`, `US-001-AC3`, `US-001-AC4`, `US-002-AC1`, `US-002-AC3`, `US-002-AC4`, `US-002-AC5`, `US-003-AC1`, `US-003-AC2`, `US-003-AC3`, `US-005-AC1`, `US-005-AC2`, `US-005-AC3`, `US-011-AC1`, `US-011-AC2`, `US-011-AC3`, `US-011-AC4`.
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Before transport exists, inspect actual stored trust, scan a direct endpoint, explicitly look up Adoption evidence including hashed hosts, compare complete independent fingerprints and warn about changed adopted evidence. Also produce an exact private expected-state trust intent for the later E4 local-admission stage, without committing acceptance or minting execution authority. A façade without RouteAccess never scans a routed target locally.

## Types

```rust
Trust // from ADR-0006:01
TrustQuery // from ADR-0006:01
TrustReview // from ADR-0006:01
TrustEvidence // from ADR-0006:01
TrustEvidenceView // from ADR-0006:01
TrustChange // from ADR-0006:01
PreparedTrustChange // from ADR-0006:01
PreparedTrustChangeView // from ADR-0006:01
TrustExecutor // from ADR-0006:01
ScanRequest // from ADR-0006:01
KeyObservation // from ADR-0006:01
DiscoveryPath // from ADR-0006:01
TrustPending // from ADR-0006:01
TrustCompletion // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E2::Configuration // from ADR-0006:01
E2::ResolvedConnection // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. TrustEvidence and positive trust/prepared wrappers remain private; Store-created observations are immutable readable facts, not authority.

## Functions

### `Trust::assemble(core: TrustCore, configuration: E2::Configuration, routes: Option<E1::RouteAccess>) -> Trust`

- **Intent.** Make real stored/local trust review available without requiring transport bootstrap.
- **Where.** `src/application/trust.rs`.
- **Pre.** Core and Configuration belong to the same primary; optional routes, when present, belong to that primary.
- **Post.** None supports stored inspection, explicit local Adoption lookup, direct discovery, exact comparison and drift review. None cannot serve routed discovery and cannot scan a routed target locally. A later façade with Some(routes) addresses the SAME core.
- **Errors.** Infallible composition; routed capability absence is reported by review without network fallback.
- **Idempotency.** Additional façades share the same core; composition never creates another trust authority.

### `Trust::review(&self, query: TrustQuery) -> Result<TrustReview, E6::Failure>`

- **Intent.** Present stored trust, discovery, independent comparison and drift evidence without committing acceptance.
- **Where.** `src/application/trust.rs`.
- **Pre.** External query identifies a resolvable saved Server; evidence matches its complete endpoint/route. Routed discovery requires Some(RouteAccess).
- **Post.** Resolves inventory FIRST; Inspect reads SQLite, explicit AdoptionLookup uses ssh-keygen -F including hashed entries, direct Discover uses actual keyscan, CompareFingerprint requires exact complete SHA256 equality, and CheckAdoptedDrift reads only the recorded Adoption source and warns without synchronization. Review distinguishes all five Trust States, stored provenance, accepted evidence and T3 role ineligibility, empty/loading/stale/error. No connectivity event promotes trust. Core launches owned scan/lookup workers and keeps servicing its mailbox; completions bind request, identity and revision before presentation.
- **Errors.** Read, discovery, lookup, comparison, unavailable-route or storage failure commits nothing; cancelled lookup/decision cannot accept. Routed requests with None return unavailable and never Local. Errors identify Server/key/route, possible changes, recovery and related Operation when applicable.
- **Idempotency.** Repeatable evidence queries may observe newer data; never automatic acceptance, retry mutation or external-file rewrite.

### `TrustEvidence::inspect_evidence(&self) -> TrustEvidenceView<'_>`

- **Intent.** Expose the exact candidate fingerprint and provenance for a keyboard-accessible trust decision without allowing evidence forgery.
- **Where.** `src/domain/trust.rs`.
- **Pre.** A live TrustCore-created TrustEvidence exists.
- **Post.** Returns exact borrowed candidate fields and optional comparison/provenance, not a mutable record or positive permission; UI can render complete fingerprint without accessing private constructors.
- **Errors.** Infallible; no I/O or allocation.
- **Idempotency.** Repeatable read; evidence remains sealed.

### `TrustExecutor::prepare_change(&self, operation_id: Uuid, resolved: E2::ResolvedConnection, change: TrustChange) -> Result<PreparedTrustChange, E6::Failure>`

- **Intent.** Translate an explicit reviewed trust intent into an exact expected-state change without applying it.
- **Where.** `src/application/trust.rs`.
- **Pre.** E4 supplies external Operation identity; evidence/endpoint/route match; expected authoritative key set is current; Verification has exact independent evidence, Adoption explicit lookup provenance, and TOFU retains warning/confirmation requirement.
- **Post.** Creates private operation/identity-bound expected and replacement sets with accepted fields and correct path/state/provenance; replacement alone never means Verified; deletion creates no substitute key. Does not decide consent, status or success.
- **Errors.** Validation, missing evidence, mismatch, stale expected keys or storage failure commits nothing. Only actual E4 L1 admission enables execution; semantic preparation never invents a remote script.
- **Idempotency.** Equivalent current state/input yields equivalent intent, not an execution permit; no acceptance on replay or cancellation.

## Invariants

| ID    | Statement                                                                                                                                                                               | Role      | Proof                                                                   |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------- |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | preserve  | Future test `trust_authoritative_store_migration`; first proof owner 01 |
| INV-2 | Review, drift, discovery, connectivity and generated-file edits never accept, replace, delete or promote authoritative Host Keys.                                                       | establish | Future test `trust_readonly_review_provenance`; first proof owner 03    |
| INV-3 | Verified evidence requires exact independent complete SHA256 comparison; Adoption and TOFU retain their distinct provenance and cannot imply Verified.                                  | establish | Future test `trust_exact_fingerprint_evidence`; first proof owner 03    |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | preserve  | Future test `trust_single_core_ports`; first proof owner 01             |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Bootstrap and review: remaining steps including exact read-only GTK presentation; no target master prerequisite.

- In: `TrustCore` ← 01
- In: `TrustExecutor` ← 01
- In: `TrustChange` ← external
- In: `Configuration` ← external
- In: `TrustQuery` ← external
- Out: `TrustEvidence` → 04
- Out: `PreparedTrustChange` → 05
- Out: `TrustReview` → 06
- Out: `TrustReview` → final

External producer and gate details: ADR-0006:01 supplies declarations; ADR-0002:01 actually produces Configuration::resolve and complete identities. None route capability is an explicit supported bootstrap input, not a dummy RouteAccess. Stored/local review is complete without E1. Routed Some capability and its actual scan path are 06.

## Steps

1. Write failing stored/local review tests using actual E2 resolution before every query; implement assemble with None and review without requiring E1 runtime capabilities.
2. Run actual ssh-keygen -F on explicitly selected plain/hashed fixture files only; test no-match, command failure and cancellation without rewriting user files or accepting a key.
3. Run real local keyscan and compare exact full independent SHA256, differing/partial values and failed/empty candidates; seal evidence and provide inspect_evidence borrowed display data.
4. Check recorded Adoption source entry digest for drift without replacing state/provenance, upgrading on login or doing background synchronization.
5. Implement owned workers, TrustPending identity/revision binding and TrustCompletion handling while mailbox remains serviced; late/cancelled reads cannot accept.
6. Write failing exact path/provenance/expected-state tests and implement prepare_change as pure semantic preparation, reusing PreparedTrustChange::prepared_trust_store_view introduced by ADR-0006:01. Use explicit external Operation identifiers; preparation alone cannot commit, grant a lease, bypass confirmation or produce an audit envelope. E4:05 later consumes this real producer.
7. Implement responsive private GTK loading/empty/unverified/provenance/drift/stale/error and complete keyboard details/cancel; use glib::MainContext, never widgets from Tokio. Trust-decision mutation controls remain explicitly unavailable until 05 and approved E4:05 local admission.
8. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-001-AC2`: Given a Server trust view, when its trust evidence is displayed, then VERIFIED, ADOPTED, UNVERIFIED, MISMATCH, and NONE remain distinct and the stored source and available verification evidence are inspectable rather than inferred from connectivity.
- [ ] `US-001-AC3`: Given an accepted Host Key and a failed or absent connection attempt, when Connection State changes or remains NOT_CONNECTED, then that observation does not itself upgrade, erase, or independently verify the Host Key's Trust State.
- [ ] `US-001-AC4`: Given a matching connection using an ADOPTED or UNVERIFIED key, when authentication or connection succeeds, then successful transport alone does not promote the key to VERIFIED.
- [ ] `US-002-AC1`: Given an inventory Server and an entry in the user's known_hosts, when I explicitly request Adoption, then ServerDesk looks up that host and port using ssh-keygen -F and presents the matching Host Key for the explicit trust workflow.
- [ ] `US-002-AC3`: Given no matching entry or a failed lookup, when Adoption is attempted, then no successful Adoption is claimed and the affected Server and available corrective action are shown without inventing a key.
- [ ] `US-002-AC4`: Given that I have not requested Adoption, when I add or connect to a Server, then the user's known_hosts is not silently used to accept or replace its Host Key.
- [ ] `US-002-AC5`: Given an Adoption request that I cancel before acceptance, when the workflow ends, then no new trust acceptance is committed and the original user file is unchanged.
- [ ] `US-003-AC1`: Given a SHA256 fingerprint I obtained from a trusted independent source such as the provider console or cloud-init, when I submit it for a discovered candidate Host Key, then ServerDesk compares it exactly with that key's ssh-keyscan-derived SHA256 fingerprint before accepting it as VERIFIED.
- [ ] `US-003-AC2`: Given that the supplied fingerprint differs from the discovered fingerprint, including only a partial match, when comparison completes, then verification does not succeed and the candidate is not stored as VERIFIED.
- [ ] `US-003-AC3`: Given that ssh-keyscan fails or yields no candidate matching the independent fingerprint, when verification is attempted, then no verified acceptance is committed and the discovery or comparison failure is explained.
- [ ] `US-005-AC1`: Given an ADOPTED Host Key whose corresponding user known_hosts entry has subsequently changed, when ServerDesk observes that drift, then the Server trust view warns about the changed external evidence.
- [ ] `US-005-AC2`: Given observed external drift, when the warning is shown or another connection is requested, then ServerDesk does not automatically replace the SQLite Host Key, its Trust State, or its provenance with the external entry.
- [ ] `US-005-AC3`: Given the drift warning, when I elect to change trust, then the new decision requires the explicit acceptance or replacement workflow rather than a background synchronization of the user file.
- [ ] `US-011-AC1`: Given discovery, Adoption lookup, verification, or trust persistence in progress, when I interact with the GTK application, then the main thread remains responsive and the affected workflow exposes loading rather than a fabricated successful result.
- [ ] `US-011-AC2`: Given no stored key, unresolved trust, observed drift, or a failed trust workflow, when the relevant view is opened, then it presents the corresponding empty/unverified, stale-external-evidence warning, or error state without confusing it with Server downtime.
- [ ] `US-011-AC3`: Given an error, when its details are shown, then the UI identifies what failed, the affected Server/key, whether a change may have occurred, available recovery actions, and the related Operation details where applicable.
- [ ] `US-011-AC4`: Given an explicit trust decision or warning, when I navigate using the keyboard, then the acceptance, rejection/cancel, and relevant details controls are reachable without requiring a pointer.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test trust_workflows trust_readonly_review_provenance
cargo test --test trust_workflows trust_exact_fingerprint_evidence
cargo test --test trust_workflows trust_prepare_change_is_nonmutating
cargo test --test trust_gtk trust_review_gtk_states
rustfmt --edition 2024 --check src/domain/trust.rs src/application/trust.rs src/ui/trust.rs src/ui/mod.rs tests/trust_workflows.rs tests/trust_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/00-overview.md .scratch/adr-0003-host-key-store/issues/03-readonly-evidence-review.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add automatic acceptance/promotion/synchronization, remote shell trust mutation, direct unaudited HostKey writes, fake capability constructors, or a second trust/audit owner.
