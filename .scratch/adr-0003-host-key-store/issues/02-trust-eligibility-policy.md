# 02: Decide exact Direct and Jump eligibility of accepted Trust States

**Status:** ready-for-human
**Blocked by:** None (can start immediately).
**External blockers:** None.
**Source:** `US-006-AC1`, `US-007-AC2`; Further Notes item 1's unresolved use policy.
**Contract:** contract: n/a, human policy defining exact trust-use membership; no eligibility default is selected by this ticket draft.

## Goal

Define the exact accepted Trust State membership for Direct use and Jump use, explicitly including the treatment of an explicitly accepted UNVERIFIED key. Keep evidence strength separate from connection permission; do not let successful authentication or route position strengthen evidence.

## Steps

1. Record complete allowed/blocked membership for each Trust State under each TrustUse, especially confirmed TOFU/UNVERIFIED. Distinguish absent trust and mismatch from an accepted-but-not-independently-verified key. No permissive or restrictive membership is guessed by an agent.
2. Record how unavailable policy, unaccepted evidence and an ineligible accepted key are explained in gate/GTK outcomes without changing Trust State or conflating it with Server downtime.
3. Update only the appended Design in `.scratch/adr-0003-host-key-store/spec.md` and existing `GLOSSARY.md` if needed. Preserve the original source prefix. Propagate exact approved membership to 04/06 and downstream E1 policy consumers, keeping missing local trust Operation policy ADR-0004:01 separate.
4. Run the future document formatter check below after human approval; implementation and actual direct/Jump behavior stay in their numbered tickets.

## Acceptance criteria

Runtime context, delivered by tickets 04/06 rather than required to complete this policy decision:

> `US-006-AC1`: Given no accepted Host Key and Trust State NONE, when Connect or connect_on_startup would start a master, then connection is blocked as HostKeyUntrusted and the user is offered explicit trust acceptance instead of auto-acceptance.
>
> `US-007-AC2`: Given a Jump Server with no accepted Host Key or with a mismatched Host Key, when the route is requested, then it is not used for target discovery or connection until its own explicit trust acceptance or mismatch resolution completes.

- [ ] A complete Direct/Jump membership decision, including UNVERIFIED, is explicit and copied into the canonical Design without invented state promotions, runtime bypass settings or an implicit fallback.
- [ ] Until approved, 04 and transitively 06 stay needs-info; neither absence of a policy nor evidence discovery can issue positive TrustStart.
- [ ] This decision does not settle or bypass ADR-0004:01's separate local lifecycle/atomic-audit/Verifier/Reconciliation policy, and does not independently enable trust mutations.

## Verify

Future human-decision document gate only; no command ran during drafting and this is not implementation evidence.

```sh
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown GLOSSARY.md .scratch/adr-0003-host-key-store/spec.md .scratch/adr-0003-host-key-store/issues/02-trust-eligibility-policy.md .scratch/adr-0003-host-key-store/issues/04-prestart-atomic-snapshot.md .scratch/adr-0003-host-key-store/issues/06-registered-routed-trust-review.md
```

## Do not touch

Original source prefix, executable code, schema, automatic acceptance/promotion, user trust files, remote scripts or the separately owned E4 local trust Operation decision. No new rationale/design document or configurable permissive boolean.
