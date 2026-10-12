# Phase: spec

You delegate every document to its owning role and never write the specs yourself. Read the roles' JSON reports and `docs/reviews/spec-review.md`, not the full documents.

Preconditions: `docs/00-brief.md` exists with `Status: locked`. If it does not, set `phase=ideation` and go back to the routing loop.

## 1. Full run

Launch each role with a task packet (`continuity.md`) and its resolved model (`models.md`). Wait for each step before starting the steps that depend on it.

| Step | Role | Writes | Depends on |
|---|---|---|---|
| 1 | product-manager | `docs/01-prd.md` | brief |
| 2 | ux-designer | `docs/02-ux.md` | 1 |
| 3a | design-system | `docs/03-design-system.md`, `design/tokens.json` | 2 |
| 3b | architect | `docs/04-architecture.md`, `docs/adr/`, `docs/agent-permissions.json` | 2 (run 3a and 3b in parallel when subagents allow it, both in the foreground) |
| 4 | design-system, "implementation notes" mode | `docs/03-design-system.md` §5 | 3b, only if the architect's report says the stack changes how tokens map to the framework |
| 5 | planner | `docs/05-plan.md`, `docs/plan.json` | 3a, 3b |
| 6 | spec-reviewer | `docs/reviews/spec-review.md` | 5 |

Every packet's "Decisions to respect" quotes the relevant entries of `docs/DECISIONS.md`. Record each finished step in the journal.

## 2. Automatic fix rounds (at most 2 in total)

If the review verdict is `NEEDS_FIXES` and `spec_fix_rounds < 2`:

1. Increase `spec_fix_rounds` in the state.
2. Group the blocking issues by owner. Send each owner its issues ("revise: fix these blocking issues, change nothing else"). Go upstream first (PRD → UX → design system/architecture → plan); independent owners can run in parallel.
3. Run the spec-reviewer again in "delta review" mode.
4. Repeat while `NEEDS_FIXES` and rounds remain.

## 3. Revision run (user feedback)

When the user gives feedback on a spec that is still in phase `spec` (for example after a `--gate spec` pause), work out which document it first affects (brief < PRD < UX < design system/architecture < plan). Send that owner the feedback in "revise" mode, then every downstream owner with a one-line summary of what changed upstream. Finish with a "delta review". Record the feedback in `docs/DECISIONS.md` first.

## 4. Save and hand over

1. `git add docs design && git commit -m "docs(spec): draft v<spec_version + 1>"`.
2. Write `docs/handoffs/phase-spec.md`: the documents and their one-line summaries (from the reports), the PRD's blocking open questions, the assumptions that matter most, the review verdict and the remaining issues, the plan (milestones, block count, critical path, high-complexity blocks), and the permissions proposed by the architect.
3. Set `status=awaiting_approval` and write a `spec` journal line.
4. Continue the routing loop (`approve.md` § Spec).

## Resume

If the run died mid-spec (`status=drafting` with some documents already written), read the journal to see which steps finished. Restart at the first step whose report is missing. A document that exists without a journal line is a partial write: rerun its owner in "revise: complete this document" mode.
