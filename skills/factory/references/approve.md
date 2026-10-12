# Approval: spec and milestones

Two moments can involve the user. Each is automatic unless the user asked for a gate (`flags.gates` contains `spec` or `milestone`), or a problem needs them. A `/factory approve` argument forces the interactive version.

## Spec (phase `spec`, status `awaiting_approval`)

### A. Decide whether a human is needed

- Review verdict `READY` and no `spec` gate → approve automatically (§ C).
- Review verdict `READY` with a `spec` gate → § B.
- Verdict `NEEDS_FIXES` after 2 fix rounds → `stops.md` § Spec not ready.

### B. With the user

1. Summarise `docs/handoffs/phase-spec.md` in at most 12 lines, and say where to read each document.
2. Ask: Approve (recommended) · Request changes (the user types feedback; go to `spec.md` § Revision run) · Pause (stop; `/factory` resumes here).
3. Record the answer in `docs/DECISIONS.md`.

### C. Apply the approval

1. **Agent permissions** (Claude Code only): filter `docs/agent-permissions.json` as described in `permissions.md`. In interactive mode, show the rules to add and the rules rejected, and ask Add (recommended) / Skip. In automatic mode, add the rules that pass the filter. Record what was added.
2. In `docs/01-prd.md` and `docs/02-ux.md`, set `Status: approved`. Do this **before** step 4, which freezes these files.
3. Create `CHANGELOG.md` (Keep a Changelog, with a `## [Unreleased]` section) and `docs/TECH_DEBT.md` if they do not exist. Fill the `## Stack & commands` part of the App Factory section in the instruction file with at most 10 lines from architecture §2 and §8.
4. **Freeze** the brief, the PRD and the UX spec:
   - Claude Code (`permission_rules`): add `Edit(docs/00-brief.md)`, `Edit(docs/01-prd.md)` and `Edit(docs/02-ux.md)` to `permissions.deny` in `.claude/settings.json`. These rules block every write to the three files (including Write and shell writes) and apply immediately.
   - Every host: the quality gate's built-in `frozen-docs` check now fails any block that touches them (`block-runner.md` § Gate).
5. `n = spec_version + 1`. Set `phase=build, status=next, spec_version=n, spec_fix_rounds=0, current_block=null, note=null`.
6. Write `docs/handoffs/phase-approval-v<n>.md` (short: what was approved, the permissions added, the gates).
7. `git add -A && git commit -m "docs(spec): approve spec v<n>"`, then `git tag spec-v<n>`.
8. Write an `approve` journal line. Notify "Spec v<n> approved: build starting" (`hosts.md`), then continue the routing loop.

## Milestone (phase `build`, status `milestone_review`)

1. Find the milestone that just finished in `docs/plan.json`, with its exit criteria and its blocks. Read their `docs/handoffs/<ID>.md` summaries (the first 15 lines of each are enough).
2. Write `docs/handoffs/phase-M<k>.md`: what is now demoable and how to run it, debt added (`docs/TECH_DEBT.md`), "Attention" items, handoff gaps (`continuity.md` § Handoff quality loop), and per-tier statistics (`models.md`).
3. Without a `milestone` gate: notify "M<k> done", set `status=next`, and continue.
4. With a `milestone` gate, or with `/factory approve`: summarise in at most 10 lines and ask Continue (recommended) · Request changes (→ `/change <request>`) · Pause. Record the answer.
   - Continue: `git tag m<k>-accepted`, set `status=next`, and continue.
   - Request changes: set `note`, stop, and tell the user to run `/change <what to change>`.
   - Pause: stop. The status stays `milestone_review`.
5. Commit the phase handoff (`docs(M<k>): milestone summary`) and write a `milestone` journal line.
