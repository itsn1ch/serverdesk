# 06: Discover routed target keys through permitted Jump Servers and render honest trust outcomes

**Status:** needs-info
**Blocked by:** 03: Review local trust evidence, exact fingerprints and Adoption drift without acceptance, 04: Gate reserved starts and regenerate atomic authoritative trust snapshots, 05: Commit explicitly confirmed trust changes with inseparable permanent audit facts.
**External blockers:** ADR-0006:01, ADR-0001:04, ADR-0004:08.
**Source:** `US-007-AC1`, `US-007-AC2`, `US-007-AC3`, `US-007-AC4`, `US-011-AC1`, `US-011-AC2`, `US-011-AC3`, `US-011-AC4`, `US-001-AC6`, `US-002-AC2`, `US-002-AC5`, `US-003-AC4`, `US-004-AC1`, `US-004-AC2`, `US-004-AC3`, `US-010-AC1`, `US-010-AC2`, `US-010-AC3`, `US-010-AC4`, `US-010-AC5`, `US-010-AC6`.
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

Use a real registered permitted Jump generation to discover a routed target without local fallback or inherited trust. Finish the actual accessible GTK trust-decision journey through canonical E4 public planning/decision (ADR-0004:08), reusing 03 semantic preparation, E4:05 admission/outcome stages and 05 commit; backend-only proof does not claim this UI outcome.

## Types

```rust
Trust // from ADR-0006:01
DiscoveryPath // from ADR-0006:01
ScanRequest // from ADR-0006:01
KeyObservation // from ADR-0006:01
TrustEvidence // from ADR-0006:01
TrustReview // from ADR-0006:01
TrustPending // from ADR-0006:01
TrustCompletion // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E2::Configuration // from ADR-0006:01
E2::ResolvedConnection // from ADR-0006:01
E1::StartReservation // from ADR-0006:01
E1::RouteAccess // from ADR-0006:01
E1::MasterGeneration // from ADR-0006:01
E4::OperationRequest // from ADR-0006:01
E4::OperationDecision // from ADR-0006:01
TrustChange // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. TrustEvidence and positive trust/prepared wrappers remain private; Store-created observations are immutable readable facts, not authority.

## Functions

No new public functions. Privately deepen `Trust::review` and optional-route façade behavior owned by 03; preserve `TrustGate::prepare` owned by 04 and introduce no second trust writer.

## Invariants

| ID    | Statement                                                                                                                                                                               | Role      | Proof                                                                          |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | preserve  | Future test `trust_authoritative_store_migration`; first proof owner 01        |
| INV-2 | Review, drift, discovery, connectivity and generated-file edits never accept, replace, delete or promote authoritative Host Keys.                                                       | preserve  | Future test `trust_readonly_review_provenance`; first proof owner 03           |
| INV-3 | Verified evidence requires exact independent complete SHA256 comparison; Adoption and TOFU retain their distinct provenance and cannot imply Verified.                                  | preserve  | Future test `trust_exact_fingerprint_evidence`; first proof owner 03           |
| INV-4 | NONE, MISMATCH and T3 role-ineligible accepted keys cannot produce TrustStart or authorize a master.                                                                                    | preserve  | Future test `trust_prestart_absent_mismatch_eligibility`; first proof owner 04 |
| INV-5 | Each positive start owns its original resolution/reservation and a fresh atomic SQLite-derived matching snapshot; failed regeneration never uses an old file.                           | preserve  | Future test `trust_atomic_snapshot_handback`; first proof owner 04             |
| INV-6 | Only explicit engine-admitted trust changes can commit, and each key delta/read-back plus its E4-authored event and ordinal commit atomically at expected state/tail without secrets.   | preserve  | Future test `trust_atomic_local_audit_commit`; first proof owner 05            |
| INV-7 | Routed discovery executes on the registered already-permitted Jump generation and never falls back locally or inherits target trust.                                                    | establish | Future test `trust_registered_jump_discovery`; first proof owner 06            |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | preserve  | Future test `trust_single_core_ports`; first proof owner 01                    |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Routed review plus final public GTK trust decision: private routed enhancement and canonical public Operation integration; no new public method or second mutation/trust writer.

- In: `TrustReview` ← 03
- In: `RouteAccess` ← external
- In: `OperationRequest` ← external
- In: `TrustCommitFacts` ← 05
- Out: `TrustReview` → final

External producers: ADR-0006:01 supplies declarations only; ADR-0001:04 produces actual registered RouteAccess/generations. ADR-0004:08 produces actual public plan/decide execution integration (planning ADR-0004:07), reusing the SAME ADR-0004:05 local stages and real E3:05 commit. Local 04 is the trust/prefix-gate prerequisite; local 05 supplies atomic key/audit facts. Some(routes) reassembly retains the same TrustCore. Unimplemented producers and the transitive file-policy 03 gate keep the whole public journey unavailable, never fakes a successful decision.

## Steps

1. Write failing actual registered Jump fixture discovery tests, with independently Verified preexisting Jump evidence and separately unresolved target evidence; use E1:04 current route capability.
2. Privately refine existing review/assemble implementation to use Some(routes) over SAME TrustCore, route preparation, generation-bound KeyScan and typed Consumer event receiver; no new public façade or duplicate method.
3. Assert keyscan executes on Jump, absent/mismatched/ineligible prefix blocks, failed target scan preserves prior decisions and target does not inherit Adopted/Verified from the route.
4. Join/stop routed scan consumers on cancellation, bind completions to target/route/generation and preserve stored trust; no local fallback with None or failed Some.
5. Exercise actual private GTK keyboard acceptance/reject/cancel/details, glib::MainContext loading/empty/drift/stale/error and affected target/route recovery presentation.
6. Through the actual E4:08 public plan/decide workflow, demonstrate plain/hashed Adoption, exact independent verification, warned TOFU, replacement/deletion, cancelled/dismissed decisions, storage failure and uncertain outcome. Observe real SQLite and permanent Operation details; no direct executor call or test-forged envelope substitutes for the user journey.
7. Run the complete add Server → explicit trust decision → actual connection → VTE terminal journey with independent Trust/Connection State and no promotion on authentication. Preserve all negative and keyboard/loading/stale/error criteria; approved local proof and joined-work lease release remain E4-owned.
8. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-007-AC1`: Given a registered target route through a Jump Server with an independently VERIFIED Host Key, when target Host Key discovery runs, then ssh-keyscan executes on that Jump Server rather than assuming the local workstation can discover the target directly.
- [ ] `US-007-AC2`: Given a Jump Server with no accepted Host Key or with a mismatched Host Key, when the route is requested, then it is not used for target discovery or connection until its own explicit trust acceptance or mismatch resolution completes.
- [ ] `US-007-AC3`: Given a target Host Key discovered through a trusted Jump Server, when the target trust workflow is shown, then it still requires Adoption, exact independent verification, or warned explicit TOFU and is not automatically VERIFIED or ADOPTED by the route.
- [ ] `US-007-AC4`: Given a failed target discovery on the Jump Server, when the result is shown, then the target remains unaccepted unless it already had a separate committed decision and the error identifies the affected target and route without claiming identity proof.
- [ ] `US-011-AC1`: Given discovery, Adoption lookup, verification, or trust persistence in progress, when I interact with the GTK application, then the main thread remains responsive and the affected workflow exposes loading rather than a fabricated successful result.
- [ ] `US-011-AC2`: Given no stored key, unresolved trust, observed drift, or a failed trust workflow, when the relevant view is opened, then it presents the corresponding empty/unverified, stale-external-evidence warning, or error state without confusing it with Server downtime.
- [ ] `US-011-AC3`: Given an error, when its details are shown, then the UI identifies what failed, the affected Server/key, whether a change may have occurred, available recovery actions, and the related Operation details where applicable.
- [ ] `US-011-AC4`: Given an explicit trust decision or warning, when I navigate using the keyboard, then the acceptance, rejection/cancel, and relevant details controls are reachable without requiring a pointer.
- [ ] `US-001-AC6`: Given an explicit Adoption, independent verification, or TOFU acceptance request, when its trust change is processed, then it uses the shared Operation workflow and retains a sanitized Audit Trail outcome rather than bypassing the engine as an unaudited direct store edit.
- [ ] `US-002-AC2`: Given a matching hashed-host entry, when I request Adoption, then lookup supports it and a successfully accepted Host Key is stored as ADOPTED rather than VERIFIED.
- [ ] `US-002-AC5`: Given an Adoption request that I cancel before acceptance, when the workflow ends, then no new trust acceptance is committed and the original user file is unchanged.
- [ ] `US-003-AC4`: Given a previously accepted ADOPTED or UNVERIFIED Host Key and a matching independent fingerprint, when explicit verification commits successfully, then the stored decision becomes VERIFIED with its verification evidence and time retained.
- [ ] `US-004-AC1`: Given a newly discovered Host Key without an independent trusted comparison, when I choose TOFU, then the workflow warns that identity has not been independently verified and requires explicit confirmation before acceptance.
- [ ] `US-004-AC2`: Given confirmed and successfully persisted TOFU acceptance, when the result is displayed or the application is reopened, then the key remains UNVERIFIED until an independent verification succeeds.
- [ ] `US-004-AC3`: Given a dismissed or rejected TOFU warning, when the workflow ends, then no TOFU acceptance is recorded and discovery alone does not establish trust.
- [ ] `US-010-AC1`: Given an existing accepted Host Key, when replacement or deletion is requested, then the affected Server/key and intended change are presented for an explicit decision through the shared Operation workflow rather than applied automatically by discovery or drift detection.
- [ ] `US-010-AC2`: Given an explicitly confirmed replacement, when the new decision completes, then the replacement follows one of the accepted acceptance paths and retains that path's Trust State; replacement alone does not mean VERIFIED.
- [ ] `US-010-AC3`: Given an explicitly confirmed deletion, when persistence and verification complete, then the deleted entry is no longer authoritative and a later connection without another eligible accepted entry is blocked rather than recreating trust automatically.
- [ ] `US-010-AC4`: Given a successful replacement or deletion, when Operation details are inspected, then the Audit Trail records the explicit change and verified outcome using the shared append-only operations/operation_events contract without secrets or file contents and without automatic audit deletion.
- [ ] `US-010-AC5`: Given failed persistence or insufficient verification evidence during replacement or deletion, when the result is displayed, then no successful change is claimed and the shared Operation outcome reflects the evidence rather than a dismissed dialog or attempted write.
- [ ] `US-010-AC6`: Given a previously recorded trust-change Operation, when later trust changes occur, then earlier audit evidence is not erased or rewritten to disguise the former decision.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.

```sh
cargo test --test trust_workflows trust_registered_jump_discovery
cargo test --test trust_gtk trust_routed_review_gtk_states
cargo test --test trust_gtk trust_public_decision_and_terminal_journey
rustfmt --edition 2024 --check src/domain/trust.rs src/application/trust.rs src/ui/trust.rs tests/trust_workflows.rs tests/trust_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/00-overview.md .scratch/adr-0003-host-key-store/issues/06-registered-routed-trust-review.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add automatic acceptance/promotion/synchronization, remote shell trust mutation, direct unaudited HostKey writes, fake capability constructors, or a second trust/audit owner.

## Comments

T3 is approved after the explicit correction 3, VERIFIED Jump only, overriding T2. L1 is separately approved. This ticket still remains needs-info because public workflow E4:08 depends on 07 and the unresolved file-policy 03. Neither policy approval nor backend 05 completes the full GTK journey.

| Accepted category               | Direct | Jump  |
| ------------------------------- | ------ | ----- |
| VERIFIED                        | Allow  | Allow |
| ADOPTED                         | Allow  | Block |
| Explicitly committed UNVERIFIED | Allow  | Block |
| MISMATCH                        | Block  | Block |
| NONE                            | Block  | Block |

Every registered hop must independently pass Jump membership before carrying discovery; the target uses Direct membership without inheriting hop evidence. Current matching accepted-key preflight, original binding and fresh atomic SQLite snapshot remain required. Role-ineligible accepted keys report HostKeyUntrusted; mismatch reports HostKeyMismatch. Discovery and dialog acknowledgement create no acceptance or permission.
