# Change: modify the scope during the build

The brief, the PRD and the UX spec are frozen during the build. `/change <request>` is the only sanctioned way to change them. You delegate every edit to the document's owner.

## 0. Preflight

1. Phase `spec` → this is not a change but a spec revision: go to `spec.md` § Revision run.
2. Status `in_progress` → a block is running. Say so and stop; retry after it ends.
3. Empty request → ask what should change and why.
4. Record the request in `docs/DECISIONS.md`.
5. Set `status=changing`.
6. **Unfreeze** (Claude Code): remove the three `Edit(docs/0x-…)` rules from `permissions.deny` in `.claude/settings.json`. The gate's `frozen-docs` check allows these files while the status is `changing`.

## 1. Impact analysis

Read the index of `docs/plan.json` and the headings of each document (not the full texts). Decide:
- which documents the change touches;
- whether it affects `done` blocks. Those get follow-up blocks, never rewrites.

## 2. Revisions, upstream first

Send each affected owner the request in "revise" mode, with a one-line note of what changed upstream:

1. **product-manager**: always, if behaviour or scope changes. It adds a Revisions entry.
2. **ux-designer**: if screens, flows or states change.
3. **architect** and/or **design-system**: if the structure, data model, contracts, stack, tokens or components change. Run them in parallel when both apply.
4. **planner**: always. It adds new or modified `todo` blocks, follow-up blocks for `done` work, and marks dropped work `skipped`.
5. **spec-reviewer**: "delta review".

## 3. Hand over

1. `git add docs design && git commit -m "docs(change): <short summary>"`.
2. Set `status=change_review` and write a `change` journal line.
3. Continue with § Approve.

## Approve (status `change_review`)

A change the user asked for is approved by the user. There is no automatic approval, even when unattended. Unattended: store the question as `pending_question` and stop.

1. Show, in at most 12 lines: the requirement IDs added, changed or deprecated; the screens touched; the ADRs; the blocks added, changed or skipped; the review verdict.
2. Ask: Approve (recommended if the review is READY) · Revise (the user types feedback; go back to § 2 with it) · Reject (revert the docs with `git revert` of the change commit).
3. Record the answer in `docs/DECISIONS.md`.
4. Approve: re-freeze (add the three `deny` rules back), `n = spec_version + 1`, commit `docs(change): approve spec v<n>`, `git tag spec-v<n>`, set `spec_version=n, status=next`. Write `docs/handoffs/phase-approval-v<n>.md`.
5. Reject: revert, re-freeze, set `status=next`.
