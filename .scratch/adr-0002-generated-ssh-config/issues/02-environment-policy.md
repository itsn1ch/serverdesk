# 02: Define the saved Server environment vocabulary, default and validation

**Status:** ready-for-human
**Blocked by:** None (can start immediately).
**External blockers:** None.
**Source:** `US-001-AC1`; input prerequisite derived from product §4.1's saved Server environment field, not a new feature.
**Contract:** contract: n/a, human product policy decision supplying the unresolved environment vocabulary; implementation remains in 03.

## Goal

Record the exact closed environment enum variants, the default for newly saved/imported Servers, and validation of supplied or previously stored values. Preserve actual existing discriminators without silently claiming arbitrary strings belong to an approved enum.

## Steps

1. Decide and record the exact variant spellings and meanings, the new/imported default, and the validation/rejection behavior for missing or unrecognized values. Do not infer these from common environment names or choose them in implementation.
2. Update only the appended Design in `.scratch/adr-0002-generated-ssh-config/spec.md` and the existing `GLOSSARY.md` if the decision adds a glossary term. Preserve original source text byte-for-byte and do not create another design document.
3. Update 03's copied type/contract/acceptance input and its readiness after the decision is explicit; keep independent trust policy dependencies visible. No source/SQLite code or runtime settings are introduced by this decision ticket.
4. Run the future document formatter check below after the human-approved change.

## Acceptance criteria

Runtime context, delivered by ticket 04 rather than required to complete this policy decision:

> `US-001-AC1`: Given a saved Connection Identity, when its runtime configuration is generated, then the effective host, port, SSH user, complete Jump Server route and authentication context come from the saved inventory and referenced Auth Profile.

- [ ] Exact closed variants, new/imported default, missing/invalid-value handling and preserved existing-row interpretation are recorded unambiguously; no environment values are invented by an agent.
- [ ] Until the decision exists, 03 remains needs-info and new Server save/import is unavailable with a precise reason; an empty string or raw discriminator is not a permissive fallback.
- [ ] 01 can still read existing saved values without certifying undefined enum membership; this decision does not require SSH or expand full inventory/group management.

## Verify

Future human-decision document gate only; no command ran during drafting and this is not an implementation test.

```sh
bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown GLOSSARY.md .scratch/adr-0002-generated-ssh-config/spec.md .scratch/adr-0002-generated-ssh-config/issues/02-environment-policy.md .scratch/adr-0002-generated-ssh-config/issues/03-static-reference-import.md
```

## Do not touch

Original source spec prefix, executable code, schema, source/user SSH files, source trust eligibility, or any Operation policy. Do not convert the missing policy into a generic configurable string/boolean escape hatch or create a new rationale document.
