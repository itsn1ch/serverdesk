# 01: Approve local trust lifecycle and evidence policy

**Status:** ready-for-human
**Decision:** approved
**Blocked by:** None (can start immediately)
**External blockers:** None
**Source:** US-013-AC1, US-013-AC2, US-013-AC3, US-013-AC4
**Contract:** contract: n/a, approved human decision recorded in the appended Design; dependent runtime implementation remains pending.

## Goal

Approve the exact local trust lifecycle specialization, atomic key/audit commit evidence, proven success/failure/Unknown boundaries and read-only persisted-state Reconciliation for accept/replace/delete. State how interruption around write-ahead and inseparable commit is interpreted without any remote shell/reconnect prerequisite. The approved decision below supplies policy, not runtime implementation.

## Steps

1. Review the verbatim criteria below and record concrete supported cases, evidence inputs, predicates and uncertainty boundaries in the appended Design.
2. Resolve each relevant Open question with an explicit human decision; keep incomplete types unavailable.
3. Update the exact private approval cases and affected type/function/invariant/flow contracts and dependent tickets coherently. Do not invent generic policy registries, booleans or fallback predicates.
4. Review the accepted policy against source constraints and the future implementation cases; this decision ticket requires no completed runtime feature or product test.

## Acceptance criteria

Runtime context delivered by implementation 05/08, not required to complete this policy:

> `US-013-AC1`: Given a requested Host Key trust change, when it is applied, then it participates in the Operation engine and sanitized Audit Trail rather than writing around them.
> `US-013-AC2`: Given replacement or deletion of a trust entry, when I proceed, then explicit consent and the affected Server and Host Key identity are part of the request rather than automatic acceptance of observed drift.
> `US-013-AC3`: Given a trust mutation type without its own verifiable persisted-state outcome and Reconciliation rule, when the action is considered for enablement, then it remains unavailable under the same evidence prerequisite; no remote shell or remote-reconnect prerequisite is fabricated for a local SQLite trust commit.
> `US-013-AC4`: Given an accepted trust mutation, when its outcome is consumed by connection handling, then the ADR-0003 epic remains responsible for authoritative trust semantics and atomic generated known_hosts behavior; an Operation success does not independently bypass its connection gate.

- [x] A human approves exact local trust lifecycle, atomic persisted-state Verifier, success/failure/Unknown and read-only Reconciliation rules for accept/replace/delete, without a remote prerequisite.
- [x] The accepted decision names supported cases and explicit unavailable cases; no generic proof fallback or invented policy remains.
- [x] The appended Design and dependent implementation tickets incorporate the exact approved private cases, predicates, failure boundaries and first Proof owners.

## Verify

Future approval gate: human review of the complete predicates, uncertainty rules and updated contracts. After editing the appended contract, run `bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0004-verified-operations/spec.md` and the native ticket checker through the integrating owner. No product test result is claimed now.

## Do not touch

No application implementation, source acceptance text, unrelated ADR decisions, arbitrary commands, privileged editor, remote locks or numeric policies selected by an agent.

## Comments

The operator approved L1 on 2026-10-31, including the explicit local-only exception to broad remote-script and reconnect language. Canonical E4 Design specifies exactly Accept, Replace and Delete, per-case consent and atomic postconditions, attributed historical commit success independent from later current trust, no-commit failure, Unknown boundaries, actual worker closure, and read-only local Reconciliation. E3/E6 consumers retain exact sealed delta, read-back and inseparable event enforcement. Decision ACs are fulfilled; runtime context and first Proofs 04/05, E3:05 and 10 remain pending. Publication Status remains ready-for-human as the allowed decision-ticket role, not a runtime completion claim.

Documentation-only, tdd: n/a. Root mechanical gates and independent judgment govern final policy-unit acceptance. No application behavior is complete.
