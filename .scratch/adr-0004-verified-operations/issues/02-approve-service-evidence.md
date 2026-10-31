# 02: Approve six service Verifier and Reconciliation policies

**Status:** ready-for-human
**Decision:** approved
**Blocked by:** None (can start immediately)
**External blockers:** None
**Source:** US-006-AC1, US-006-AC2, US-006-AC3, US-006-AC4, US-007-AC1, US-007-AC2, US-007-AC3, US-007-AC4, US-007-AC5, US-008-AC1, US-008-AC2, US-008-AC3, US-008-AC4, US-008-AC5
**Contract:** contract: n/a, approved human decision recorded in the appended Design; dependent runtime implementation remains pending.

## Goal

Approve distinct Verifier and read-only Reconciliation predicates for Start, Stop, Restart, Reload, Enable and Disable, with supported unit semantics and explicit unsupported cases. For restart decide applicable boot/timestamp evidence, expected ActiveState/SubState/Result and stabilization window; occurrence is not health. Do not infer enablement from active state or assume disable stops a unit. The approved decision below supplies policy, not runtime implementation.

## Steps

1. Review the verbatim criteria below and record concrete supported cases, evidence inputs, predicates and uncertainty boundaries in the appended Design.
2. Resolve each relevant Open question with an explicit human decision; keep incomplete types unavailable.
3. Update the exact private approval cases and affected type/function/invariant/flow contracts and dependent tickets coherently. Do not invent generic policy registries, booleans or fallback predicates.
4. Review the accepted policy against source constraints and the future implementation cases; this decision ticket requires no completed runtime feature or product test.

## Acceptance criteria

Runtime context delivered by implementation 07/08/10, not required to complete this policy:

> `US-006-AC1`: Given a connected Server and a unit with the necessary detected Capability, exact permission and complete operation-specific evidence policy, when I open its service actions, then I can choose the supported start, stop, restart, reload, enable and disable actions by their actual names rather than being limited to restart or an arbitrary command field.
> `US-006-AC2`: Given an available service action and any required confirmation, when I choose Start, Stop, Restart, Reload, Enable or Disable, then the Operation details identify the selected Server, unit and corresponding action, and execution requests exactly `systemctl start`, `systemctl stop`, `systemctl restart`, `systemctl reload`, `systemctl enable` or `systemctl disable` respectively for that unit through the authorized Operation workflow, without substituting another verb or adding an unrequested mutation.
> `US-006-AC3`: Given any service action whose own Verifier or read-only Reconciliation predicate is missing, when I inspect or attempt that action, then it is unavailable with the missing evidence-policy reason even if connection and sudo permission are otherwise ready; another verb's policy does not enable it.
> `US-006-AC4`: Given an action with an evidence policy that does not cover the selected unit's semantics, when I inspect or attempt that action, then it is unavailable with that unsupported-Capability reason rather than silently using another unit type's evidence, interpreting active state as enablement evidence or assuming disable also stops the unit.
> `US-007-AC1`: Given a unit whose semantics support the documented evidence, when restart preflight runs, then it captures remote boot_id and the unit's ActiveEnterTimestampMonotonic and/or ExecMainStartTimestampMonotonic as applicable to those semantics.
> `US-007-AC2`: Given the same boot_id before and after execution, when the applicable monotonic unit timestamp increases, then the documented evidence can establish that a restart occurred but does not alone establish healthy success.
> `US-007-AC3`: Given occurrence evidence, when health is evaluated, then ActiveState, SubState and Result must satisfy the operation-specific expectations throughout the defined stabilization window before healthy success is reported.
> `US-007-AC4`: Given a changed boot_id during restart, when the result is assessed, then the reboot is not automatically interpreted as either successful restart or failure.
> `US-007-AC5`: Given unsupported timestamp semantics, missing evidence or an incomplete health/stabilization policy, when restart proof is requested, then the workflow does not invent timestamps, an expected state or a stabilization duration to declare success.
> `US-008-AC1`: Given a mutation that has executed, when the process exits, then the Operation enters VERIFYING rather than treating exit status alone as SUCCEEDED.
> `US-008-AC2`: Given sufficient operation-specific proof of success, when verification finishes, then and only then is SUCCEEDED presented.
> `US-008-AC3`: Given sufficient operation-specific proof of failure, when verification finishes, then FAILED is recorded with its available evidence rather than being confused with an unobserved remote outcome.
> `US-008-AC4`: Given SSH loss after a mutation may have begun or otherwise insufficient evidence, when the outcome is assessed, then UNKNOWN is recorded with UnknownRemoteOutcome as appropriate, not an unproven FAILED, CANCELLED or SUCCEEDED.
> `US-008-AC5`: Given a VerificationError, when the Verifier cannot establish a definite outcome, then the error preserves the Unknown Outcome boundary rather than converting inability to verify into proof that no remote effect occurred.

- [x] A human approves separate complete Verifier and read-only Reconciliation predicates for Start, Stop, Restart, Reload, Enable and Disable and their supported unit semantics.
- [x] The accepted restart decision fixes applicable timestamps/boot identity, healthy ActiveState/SubState/Result and stabilization policy while distinguishing occurrence from health and preserving uncertainty.
- [x] Supported and unavailable cases, errors and evidence boundaries are propagated into the appended Design and dependent implementation tickets; no runtime feature is a prerequisite to finishing this decision.

## Verify

Future approval gate: human review of the complete predicates, uncertainty rules and updated contracts. After editing the appended contract, run `bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0004-verified-operations/spec.md` and the native ticket checker through the integrating owner. No product test result is claimed now.

## Do not touch

No application implementation, source acceptance text, unrelated ADR decisions, arbitrary commands, privileged editor, remote locks or numeric policies selected by an agent.

## Comments

The operator approved S on 2026-10-31. Canonical E4 Design specifies each of the six exact verbs, the complete supported-unit matrix, systemd-observable 10-second stabilization with observations at 1-second boundaries including both endpoints, no contrary intervening evidence, and later-window read-only Reconciliation. This is not mathematical continuity or application health. Exact technical properties, reload synchrony, job evidence, privileges and timestamps still require detection and proof. EnabledMutation gains exactly six service cases, no file case. Decision ACs are fulfilled; runtime context and first Proofs 08/10 remain pending. Publication Status remains ready-for-human as the allowed decision-ticket role, not a runtime completion claim.

Documentation-only, tdd: n/a. Root mechanical gates and independent judgment govern final policy-unit acceptance. No application behavior is complete.
