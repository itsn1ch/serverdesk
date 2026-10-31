# 05: Commit explicitly confirmed trust changes with inseparable permanent audit facts

**Status:** needs-info
**Blocked by:** 01: Open canonical stored-key evidence and distribute one trust core, 03: Review local trust evidence, exact fingerprints and Adoption drift without acceptance.
**External blockers:** ADR-0006:01, ADR-0004:01, ADR-0004:04, ADR-0004:05, ADR-0006:03.
**Source:** `US-001-AC1`, `US-001-AC2`, `US-001-AC3`, `US-001-AC4`, `US-001-AC5`, `US-001-AC6`, `US-002-AC1`, `US-002-AC2`, `US-002-AC3`, `US-002-AC4`, `US-002-AC5`, `US-003-AC1`, `US-003-AC2`, `US-003-AC3`, `US-003-AC4`, `US-004-AC1`, `US-004-AC2`, `US-004-AC3`, `US-005-AC1`, `US-005-AC2`, `US-005-AC3`, `US-009-AC1`, `US-009-AC2`, `US-009-AC3`, `US-009-AC4`, `US-009-AC5`, `US-009-AC6`, `US-010-AC1`, `US-010-AC2`, `US-010-AC3`, `US-010-AC4`, `US-010-AC5`, `US-010-AC6`, `US-011-AC1`, `US-011-AC2`, `US-011-AC3`, `US-011-AC4`.
**Contract:** `.scratch/adr-0003-host-key-store/spec.md`, section `# Design`. If this ticket and the contract disagree, the contract wins. Stop and report the conflict.

## Goal

After approved local Operation policy and actual E4:05 admission, commit the sealed prepared delta from 03 for explicit Adoption, exact verification, warned TOFU, replacement or deletion. This ticket also completes the private E4-owned commit-worker invocation integration in `src/application/operations.rs`, driving real admission → E3 commit → owned-work join → existing E4 finish handler → lease release; E4:05 alone cannot invoke the not-yet-introduced commit. Missing policy remains unavailable, and this refinement creates no second public workflow or ownership transfer.

## Types

```rust
PreparedTrustChange // from ADR-0006:01
PreparedTrustChangeView // from ADR-0006:01
TrustCommitObservation // from ADR-0006:01
TrustCommitFacts // from ADR-0006:01
TrustChange // from ADR-0006:01
AcceptancePath // from ADR-0006:01
TrustProvenance // from ADR-0006:01
HostKey // from ADR-0006:01
TrustPending // from ADR-0006:01
TrustCompletion // from ADR-0006:01
E6::Store // from ADR-0006:01
E6::Failure // from ADR-0006:01
E2::Configuration // from ADR-0006:01
E2::ResolvedConnection // from ADR-0006:01
E4::LocalTrustEnvelope // from ADR-0006:01
E4::LocalTrustAdmitted // from ADR-0006:01
E6::ResourceLease // from ADR-0006:01
```

These are exact canonical declarations from ADR-0006:01, not existing application APIs and not alternate definitions. This ticket owns only the runtime methods/producers listed below and its private behavior refinements; no type-only declaration mints a capability. Borrowed views cannot substitute for wrappers. TrustEvidence and positive trust/prepared wrappers remain private; Store-created observations are immutable readable facts, not authority.

## Functions

### `TrustExecutor::commit(&self, change: PreparedTrustChange, audit: E4::LocalTrustEnvelope) -> Result<TrustCommitObservation, E6::Failure>`

- **Intent.** Apply an explicitly engine-admitted local trust change with inseparable sanitized audit evidence.
- **Where.** `src/application/trust.rs`.
- **Pre.** Approved E4 local lifecycle/atomic outcome/Verifier/read-only Reconciliation policy exists; matching operation/revision/expected keys and contiguous expected audit tail; durable write-ahead and active Runtime lease exist. No approved LocalTrustApproval variant exists today.
- **Post.** Store transaction compares expected keys/tail, applies E3-authored delta, verifies read-back and appends E4-authored committed event ID/ordinal in sanitized_payload, all or none. Store creates plain immutable TrustCommitObservation; TrustCore forwards facts without minting verdict or connection permission. E4 keeps servicing stop/completion/lease mailboxes and owns final outcome.
- **Errors.** CAS, ordinal conflict, FK/validation/storage failure rolls back key change and paired event. Insufficient/ambiguous proof is not success; inspect retained event/state instead of blind repeat. Cancellation before commit has no acceptance; after possible commit follow approved local outcome policy.
- **Idempotency.** Same operation/event identity and exact content returns original facts or conflict, never another commit/ordinal. Crash leaves both trust+audit committed or neither; local read-only Reconciliation is policy-gated, not dependent on fake remote reconnect.

### `TrustCommitObservation::into_facts(self) -> TrustCommitFacts`

- **Intent.** Deliver actual Store-created committed/read-back facts to the Operation owner without inventing a verdict.
- **Where.** `src/application/trust.rs`.
- **Pre.** A Store-created immutable observation was returned by the authorized transaction.
- **Post.** Moves operation ID, before/after keys, committed event ID/ordinal and digest exactly; these are observations, not authority and not Succeeded.
- **Errors.** Infallible; no I/O.
- **Idempotency.** Single consumption; no effects or success promotion.

## Invariants

| ID    | Statement                                                                                                                                                                               | Role      | Proof                                                                   |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------- |
| INV-1 | Host Key reads and migrations use authoritative SQLite fields with UUID identities, enabled foreign keys and preserved public evidence; failures never fabricate NONE or truncate data. | preserve  | Future test `trust_authoritative_store_migration`; first proof owner 01 |
| INV-2 | Review, drift, discovery, connectivity and generated-file edits never accept, replace, delete or promote authoritative Host Keys.                                                       | preserve  | Future test `trust_readonly_review_provenance`; first proof owner 03    |
| INV-3 | Verified evidence requires exact independent complete SHA256 comparison; Adoption and TOFU retain their distinct provenance and cannot imply Verified.                                  | preserve  | Future test `trust_exact_fingerprint_evidence`; first proof owner 03    |
| INV-6 | Only explicit engine-admitted trust changes can commit, and each key delta/read-back plus its E4-authored event and ordinal commit atomically at expected state/tail without secrets.   | establish | Future test `trust_atomic_local_audit_commit`; first proof owner 05     |
| INV-8 | TrustPorts distributes core/gate/executor capabilities addressing exactly one TrustCore mailbox; façades do not create a second canonical trust writer.                                 | preserve  | Future test `trust_single_core_ports`; first proof owner 01             |

Only the establishing owner writes each first proof. Every preserved first proof is a local blocker ancestor. Later whole-flow proofs are explicitly downstream integration gates, not claims that an earlier ticket already proves a future surface.

## Flow

Explicit local change: consume 03's prepared semantics through real E4:05 admission, then this ticket's private integration in `src/application/operations.rs` registers the E4-owned worker and invokes E3 commit. The still-serviced E4 mailbox receives completion only after the actual commit future and owned-work join; its existing finish handler validates tracked work/state and applies approved outcome before releasing its retained lease. E4:05 supplies admission/finish/work-set validation, not a fabricated completed commit; public GTK delivery remains 06 after ADR-0004:08.

- In: `TrustExecutor` ← 01
- In: `PreparedTrustChange` ← 03
- In: `LocalTrustAdmitted` ← external
- In: `LocalTrustEnvelope` ← external
- Out: `TrustCommitFacts` → final
- Out: `TrustCommitFacts` → 06

External producer and gate details: ADR-0006:01 declares types only. ADR-0004:01 supplies the human local policy; ADR-0004:04 supplies actual durable audit substrate; ADR-0006:03 supplies actual resource arbitration. ADR-0004:05 implements prepare_local_trust using the real E3:03 semantic producer, then admit_local_trust with exact explicit decision/current preflight, lease and durable RUNNING. Its LocalTrustAdmitted::into_local_trust_admitted(self) -> (E3::PreparedTrustChange, LocalTrustEnvelope) moves the original prepared delta and actual E4-issued envelope to this commit. The real lease remains in E4's active-work map until its approved outcome and joined cleanup; no ResourceLease parameter is added to commit. These are the SAME canonical engine stages later public plan/decide reuse, not a second GTK mutation workflow or a declaration-as-issuer claim.

## Steps

1. First consume ADR-0004:01 approved local lifecycle/write-ahead/atomic outcome/Verifier/read-only Reconciliation policy and update contracts; no invented remote POSIX script, reconnect prerequisite or approximate success rule.
2. Reuse prepare_change introduced by 03 and PreparedTrustChange::prepared_trust_store_view introduced by ADR-0006:01; introduce or redefine neither method. Exercise actual E4:05 preparation/admission under approved local policy and reject stale assumptions, missing confirmation, lease conflict and failed durable RUNNING before any trust commit.
3. Implement commit and into_facts against real E6 Store and E4 admission/audit, plus the private E4-owned invocation integration in `src/application/operations.rs`. Register and retain the actual worker/lease binding, invoke commit using the admitted original delta/envelope, and return its result to the existing E4:05 finish handler only after actual commit-future completion and owned-work join. Compare expected keys/tail, append the E4-authored payload ordinal and verify read-back inside one transaction. Arbitrary caller Error is not joined proof; E4:05 must not claim it already called this later commit.
4. Exercise rollback at every transaction boundary, concurrent stale expected keys/tail, duplicate delivery/event IDs, storage/FK/migration failure and isolated-filesystem disk full; inspect real retained SQLite/audit state, never a mock success.
5. Test cancellation/rejection before effects commits nothing; after possible commit preserve honest uncertainty and approved local read-only Reconciliation. Earlier audit events remain immutable and secret-free.
6. Prove the transaction through the E4-owned application-stage harness using this ticket's real invocation integration: actual preparation/admission → E3 commit result → registered owned-work join → existing finish handler/work-set validation → approved outcome and lease release. Exercise delayed commit completion, failure/cancellation and pending joins while the E4 mailbox remains serviced. No caller-forged envelope, lease or completion substitutes for that chain. Final 06 after ADR-0004:08 owns the enabled public GTK journey; backend-only 05 does not claim it.
7. Run the future implementation Verify commands on the integrated affected implementation, not on these documentation drafts.

## Acceptance criteria

- [ ] `US-001-AC1`: Given a successfully committed trust decision, when the application is reopened, then the Host Key Store retains its hostname, port, key algorithm, public key, SHA256 fingerprint, Trust State, source, verification time where applicable, and creation time using the accepted host_keys fields.
- [ ] `US-001-AC2`: Given a Server trust view, when its trust evidence is displayed, then VERIFIED, ADOPTED, UNVERIFIED, MISMATCH, and NONE remain distinct and the stored source and available verification evidence are inspectable rather than inferred from connectivity.
- [ ] `US-001-AC3`: Given an accepted Host Key and a failed or absent connection attempt, when Connection State changes or remains NOT_CONNECTED, then that observation does not itself upgrade, erase, or independently verify the Host Key's Trust State.
- [ ] `US-001-AC4`: Given a matching connection using an ADOPTED or UNVERIFIED key, when authentication or connection succeeds, then successful transport alone does not promote the key to VERIFIED.
- [ ] `US-001-AC5`: Given two inventory Server entries with the same Connection Identity, when either entry uses stored trust, then both consult the authoritative SQLite Host Key Store rather than maintaining independent authoritative generated-file decisions.
- [ ] `US-001-AC6`: Given an explicit Adoption, independent verification, or TOFU acceptance request, when its trust change is processed, then it uses the shared Operation workflow and retains a sanitized Audit Trail outcome rather than bypassing the engine as an unaudited direct store edit.
- [ ] `US-002-AC1`: Given an inventory Server and an entry in the user's known_hosts, when I explicitly request Adoption, then ServerDesk looks up that host and port using ssh-keygen -F and presents the matching Host Key for the explicit trust workflow.
- [ ] `US-002-AC2`: Given a matching hashed-host entry, when I request Adoption, then lookup supports it and a successfully accepted Host Key is stored as ADOPTED rather than VERIFIED.
- [ ] `US-002-AC3`: Given no matching entry or a failed lookup, when Adoption is attempted, then no successful Adoption is claimed and the affected Server and available corrective action are shown without inventing a key.
- [ ] `US-002-AC4`: Given that I have not requested Adoption, when I add or connect to a Server, then the user's known_hosts is not silently used to accept or replace its Host Key.
- [ ] `US-002-AC5`: Given an Adoption request that I cancel before acceptance, when the workflow ends, then no new trust acceptance is committed and the original user file is unchanged.
- [ ] `US-003-AC1`: Given a SHA256 fingerprint I obtained from a trusted independent source such as the provider console or cloud-init, when I submit it for a discovered candidate Host Key, then ServerDesk compares it exactly with that key's ssh-keyscan-derived SHA256 fingerprint before accepting it as VERIFIED.
- [ ] `US-003-AC2`: Given that the supplied fingerprint differs from the discovered fingerprint, including only a partial match, when comparison completes, then verification does not succeed and the candidate is not stored as VERIFIED.
- [ ] `US-003-AC3`: Given that ssh-keyscan fails or yields no candidate matching the independent fingerprint, when verification is attempted, then no verified acceptance is committed and the discovery or comparison failure is explained.
- [ ] `US-003-AC4`: Given a previously accepted ADOPTED or UNVERIFIED Host Key and a matching independent fingerprint, when explicit verification commits successfully, then the stored decision becomes VERIFIED with its verification evidence and time retained.
- [ ] `US-004-AC1`: Given a newly discovered Host Key without an independent trusted comparison, when I choose TOFU, then the workflow warns that identity has not been independently verified and requires explicit confirmation before acceptance.
- [ ] `US-004-AC2`: Given confirmed and successfully persisted TOFU acceptance, when the result is displayed or the application is reopened, then the key remains UNVERIFIED until an independent verification succeeds.
- [ ] `US-004-AC3`: Given a dismissed or rejected TOFU warning, when the workflow ends, then no TOFU acceptance is recorded and discovery alone does not establish trust.
- [ ] `US-005-AC1`: Given an ADOPTED Host Key whose corresponding user known_hosts entry has subsequently changed, when ServerDesk observes that drift, then the Server trust view warns about the changed external evidence.
- [ ] `US-005-AC2`: Given observed external drift, when the warning is shown or another connection is requested, then ServerDesk does not automatically replace the SQLite Host Key, its Trust State, or its provenance with the external entry.
- [ ] `US-005-AC3`: Given the drift warning, when I elect to change trust, then the new decision requires the explicit acceptance or replacement workflow rather than a background synchronization of the user file.
- [ ] `US-009-AC1`: Given a trust workflow that changes multiple persisted facts, when a step fails before transaction commit, then the workflow does not expose a partially committed successful trust decision or authorize a master from that attempted decision.
- [ ] `US-009-AC2`: Given an unreadable or otherwise unavailable authoritative Host Key Store, when trust is needed for a master start, then the start is blocked rather than falling back to user known_hosts, generated-file edits, or assumed trust.
- [ ] `US-009-AC3`: Given a versioned SQLite migration required for the Host Key Store, when it succeeds, then existing Host Keys and evidence remain available without silent truncation.
- [ ] `US-009-AC4`: Given a newly created Host Key record and the active storage session, when their accepted storage invariants are inspected, then its primary key is a UUID and SQLite foreign-key enforcement is enabled without adding any relationship absent the accepted schema.
- [ ] `US-009-AC5`: Given trust workflows and their Audit Trail, when persisted data is inspected, then the accepted public key/evidence fields do not introduce stored passwords, passphrases, private-key material, or file contents into SQLite or sanitized audit payloads.
- [ ] `US-009-AC6`: Given a required Host Key Store migration that fails, when affected trust workflows are requested, then StorageError is reported rather than proceeding with incompatible state or claiming a successful migration.
- [ ] `US-010-AC1`: Given an existing accepted Host Key, when replacement or deletion is requested, then the affected Server/key and intended change are presented for an explicit decision through the shared Operation workflow rather than applied automatically by discovery or drift detection.
- [ ] `US-010-AC2`: Given an explicitly confirmed replacement, when the new decision completes, then the replacement follows one of the accepted acceptance paths and retains that path's Trust State; replacement alone does not mean VERIFIED.
- [ ] `US-010-AC3`: Given an explicitly confirmed deletion, when persistence and verification complete, then the deleted entry is no longer authoritative and a later connection without another eligible accepted entry is blocked rather than recreating trust automatically.
- [ ] `US-010-AC4`: Given a successful replacement or deletion, when Operation details are inspected, then the Audit Trail records the explicit change and verified outcome using the shared append-only operations/operation_events contract without secrets or file contents and without automatic audit deletion.
- [ ] `US-010-AC5`: Given failed persistence or insufficient verification evidence during replacement or deletion, when the result is displayed, then no successful change is claimed and the shared Operation outcome reflects the evidence rather than a dismissed dialog or attempted write.
- [ ] `US-010-AC6`: Given a previously recorded trust-change Operation, when later trust changes occur, then earlier audit evidence is not erased or rewritten to disguise the former decision.
- [ ] `US-011-AC1`: Given discovery, Adoption lookup, verification, or trust persistence in progress, when I interact with the GTK application, then the main thread remains responsive and the affected workflow exposes loading rather than a fabricated successful result.
- [ ] `US-011-AC2`: Given no stored key, unresolved trust, observed drift, or a failed trust workflow, when the relevant view is opened, then it presents the corresponding empty/unverified, stale-external-evidence warning, or error state without confusing it with Server downtime.
- [ ] `US-011-AC3`: Given an error, when its details are shown, then the UI identifies what failed, the affected Server/key, whether a change may have occurred, available recovery actions, and the related Operation details where applicable.
- [ ] `US-011-AC4`: Given an explicit trust decision or warning, when I navigate using the keyboard, then the acceptance, rejection/cancel, and relevant details controls are reachable without requiring a pointer.
- [ ] Every invariant this ticket establishes has its named first Proof passing against actual state; every invariant it preserves still has its Proof passing.
- [ ] Source criteria spanning a later runtime consumer remain present verbatim above and explicitly mapped to that named integration owner; do not replace actual OpenSSH/SQLite/GTK journeys with generated-text assertions or claim policy-blocked enabled behavior delivered.
- [ ] No stored passwords, passphrases, private-key material or file contents are introduced. Storage/schema failures are real failures, not empty data or reset/migration-success claims.

## Verify

Future implementation gates only: these targets/tests are to be authored by the numbered implementation slices; none exists or has passed by virtue of this ticket draft. The workflow suites provision isolated Incus/LXD Ubuntu 24.04 fixtures with actual system OpenSSH and SQLite; GTK suites exercise the private real application adapter. Real dependency order controls downstream journey execution. Disk-full injection is restricted to an isolated container filesystem; never fill the host filesystem. No greeting, wiring, wording or source-text-only test counts as behavior evidence.
The future `trust_owned_commit_worker_handoff` case must exercise the production private integration in `src/application/operations.rs`: E4:05 admission and finish handlers with the actual E3 commit future, tracked worker completion/join, durable settlement and lease release. A manually supplied failure/completion or a fake finish callback is not evidence of work ending.

```sh
cargo test --test trust_workflows trust_atomic_local_audit_commit
cargo test --test trust_workflows trust_explicit_accept_replace_delete
cargo test --test trust_workflows trust_owned_commit_worker_handoff
cargo test --test trust_gtk trust_change_gtk_outcomes
rustfmt --edition 2024 --check src/domain/trust.rs src/application/trust.rs src/application/operations.rs src/infrastructure/storage/host_keys.rs src/ui/trust.rs src/infrastructure/storage/mod.rs tests/trust_workflows.rs tests/trust_gtk.rs
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/00-overview.md .scratch/adr-0003-host-key-store/issues/05-atomic-audited-trust-change.md
```

## Do not touch

Other tickets' public method ownership, canonical shared declaration ownership ADR-0006:01, original spec text above `# Design`, unapproved human policy values, unrelated source/configuration and user SSH/known_hosts files. Do not add automatic acceptance/promotion/synchronization, remote shell trust mutation, direct unaudited HostKey writes, fake capability constructors, or a second trust/audit owner.

The `src/application/operations.rs` change is limited to private invocation/completion integration for the SAME E4-owned local workflow. Do not redefine E4:05's prepare/admit/finish public or crate-private function contracts, transfer its Operation/audit/lease state to E3, introduce a second mutation path, or claim E4:05 completed an E3 commit before this integration exists.
