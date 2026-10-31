# 03: Approve per-type file and transfer evidence policies

**Status:** needs-info
**Blocked by:** None (can start immediately)
**External blockers:** None
**Source:** US-011-AC1, US-011-AC2, US-011-AC3, US-011-AC4, US-011-AC5, US-012-AC1, US-012-AC2, US-012-AC3, US-012-AC4, US-012-AC5
**Contract:** contract: n/a, human approval of missing policy; update only appended Design before dependent implementation.

## Goal

Approve separate complete policies for file/directory creation, write/edit, deletion, rename, chmod and upload, including path/symlink races, exact failure/Unknown and read-only Reconciliation. Separately approve read-only download source/local-destination proof, overwrite, cancellation/partial output and Reconciliation; its approval grants no remote mutation authority. No implementation policy is chosen by this ticket draft.

## Steps

1. Review the verbatim criteria below and record concrete supported cases, evidence inputs, predicates and uncertainty boundaries in the appended Design.
2. Resolve each relevant Open question with an explicit human decision; keep incomplete types unavailable.
3. Update the exact private approval cases and affected type/function/invariant/flow contracts and dependent tickets coherently. Do not invent generic policy registries, booleans or fallback predicates.
4. Review the accepted policy against source constraints and the future implementation cases; this decision ticket requires no completed runtime feature or product test.

## Acceptance criteria

Runtime context delivered by implementation 09/10, not required to complete this policy:

> `US-011-AC1`: Given file write/edit, delete, rename, chmod or upload overwriting a file, when I initiate the change, then it passes through the Operation engine with its own Verifier and read-only Reconciliation policy before that mutation type is enabled.
> `US-011-AC2`: Given an SSH user without write permission, when a file mutation is requested, then the file is read-only or the request is rejected with a clear reason; no privileged file-editing fallback is offered in MVP.
> `US-011-AC3`: Given an edited file whose mtime, size or hash differs from the opening snapshot, when I save, then FileChangedExternally blocks the write and offers reload or explicit overwrite rather than silently replacing the external change.
> `US-011-AC4`: Given a mutation involving a symlink or a path-traversal attempt, when its destination is validated, then it cannot silently redirect an approved change to an unintended resource.
> `US-011-AC5`: Given another administrative file mutation exposed by the file manager, including file or directory creation, when its workflow is enabled, then it cannot bypass the same engine and per-type evidence prerequisite merely because it is not an overwrite.
> `US-012-AC1`: Given an upload or file write, when the payload is applied, then data is written to a temporary file in the destination directory and promoted using `posix-rename@openssh.com` when that extension is available.
> `US-012-AC2`: Given that atomic rename is unavailable, when non-atomic overwrite is proposed, then its warning and additional confirmation occur before overwrite rather than silently weakening atomicity.
> `US-012-AC3`: Given a transferred file entering VERIFYING, when destination contents are assessed, then size is compared and an optional hash comparison is used when `sha256sum` is available; unsupported hashing is not represented as a successful hash check.
> `US-012-AC4`: Given failure before or after rename, when completion is assessed, then partial transfer is never COMPLETED and the result distinguishes a proven failed change from a potentially applied change requiring UNKNOWN and Reconciliation.
> `US-012-AC5`: Given a new upload or download, when the transfer is recorded, then its path, size and result are retained as metadata without file content; its transfer lifecycle distinguishes QUEUED, TRANSFERRING, VERIFYING, COMPLETED, FAILED, CANCELLED and UNKNOWN where applicable.

- [ ] A human approves separate complete proof, read-only Reconciliation and path/symlink-race rules for every enabled file/create/delete/rename/chmod/write/upload type.
- [ ] Read-only download receives its own explicit source/local-destination proof, overwrite, partial-output and cancellation policy without granting remote mutation authority.
- [ ] Approved supported/unavailable cases and uncertainty boundaries are propagated into the appended Design and dependent implementation tickets; no runtime journey is required to finish this policy decision.

## Verify

Future approval gate: human review of the complete predicates, uncertainty rules and updated contracts. After editing the appended contract, run `bun node_modules/prettier/bin/prettier.cjs --check --ignore-unknown .scratch/adr-0004-verified-operations/spec.md` and the native ticket checker through the integrating owner. No product test result is claimed now.

## Do not touch

No application implementation, source acceptance text, unrelated ADR decisions, arbitrary commands, privileged editor, remote locks or numeric policies selected by an agent.

## Comments

On 2026-10-31 the operator rejected cooperative-only candidate F and retained the full published path/symlink race safety. No file mutation or download proof policy is approved. Repeated metadata reads, hashes, REALPATH, final rechecks, posix-rename, consent and local locks cannot supply atomic no-follow confinement or conditional replacement. No helper, new privilege mechanism or remote lock is authorized. This ticket and transitively blocked whole-ticket 07/08/09/10/11, E3:06 and E6:04 remain needs-info. EnabledMutation contains only the six approved S service cases; DownloadProofApproval stays empty. All source criteria remain required, not waived or passed.

The operator subsequently authorized bounded design and isolated proof of a specified one-shot login-user capability-aware SFTP endpoint over system OpenSSH. This supersedes the earlier no-helper clause only for design work, not deployment or feature enablement. Persistent agents, root editors and remote locks remain excluded.

The operator selected only the reviewed objects as the namespace-mutation subjects. A replacement object at the same name is not authorized. Cooperative-only F and directory-entry semantics remain rejected. Standard unlinkat and pathname rename do not enforce an expected reviewed-object identity; helper permission alone does not resolve that mechanism gap or verified staging-object promotion. No atomic content-version CAS or coherent snapshot requirement is added.

These two inputs are recorded, but this ticket's three decision acceptance criteria remain unchecked. Complete per-type proof, read-only Reconciliation, safe enforcement and download policy still require resolution before approval or implementation. No file mutation or download authority is issued.
