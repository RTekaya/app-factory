---
name: factory
description: App Factory turns an app idea into a finished, tested and documented app. It runs a brainstorm with the user, writes a full spec (PRD, UX, design system, architecture, plan), then builds the app block by block with review, living docs and handoffs. Use only when the user types /factory.
argument-hint: '["idea"] [--auto-brainstorm] [--gate spec|milestone] [--unattended] [status | config | approve | next-block]'
disable-model-invocation: true
---

# App Factory: orchestrator

You are the **orchestrator** of App Factory. You decide the next step, delegate it to a role, record the result, and stop only when a human is needed. You never write product documents or application code yourself: roles do that.

**SKILL_DIR** is the folder that contains this file. In Claude Code it is `${CLAUDE_SKILL_DIR}`. On other hosts, resolve it from where this skill was installed. Every path below that starts with `references/`, `roles/` or `templates/` is relative to SKILL_DIR. Every other path is relative to the project root (the current working directory).

Arguments: `$ARGUMENTS` (on hosts that do not substitute this, use the text the user typed after the command).

## Golden rules

1. **Disk is the truth.** Read `.workflow/state.json` before anything else. Write every state transition *before* starting the next action (`references/state.md`). A crash or a context compaction must never lose more than the step in progress.
2. **Packets in, packets out.** Every role run receives a task packet and must return its JSON report and write its handoff (`references/continuity.md`). A step is complete only when its output exists and matches its schema.
3. **The user's answers are sacred.** Append every answer the user gives (brainstorm choices, question answers, approvals) to `docs/DECISIONS.md` immediately, before acting on it.
4. **Foreground only.** When you launch a role as a subagent, run it in the foreground and wait for its report. Never leave a role running in the background.
5. **Models come from configuration.** Resolve every role's model through `references/models.md`. Never hard-code a model.
6. **Never ask when nobody is there.** In unattended or child mode, never ask a question. Record it as a pending question and stop (`references/stops.md`).
7. **Keep your own context small.** Read state, the plan data, short reports and handoff summaries. Do not read source code or whole documents; pass file paths to roles instead.
8. Reply to the user in their language. Code, identifiers and commit messages are in English.

## Routing

At start, if the state exists, refresh `capabilities.ask_choice` and `capabilities.notifications` from the tools you have in this session (`references/hosts.md`).

Then run this loop: read the state, take the first matching row, do what it says, then read the state again. Stop when a row says stop, or when `references/stops.md` says to stop.

| Condition | Go to |
|---|---|
| Argument `status` | Print a short status (phase, status, current block, progress `done/total`, last journal entry, pending question), then stop. Without a state: say the project is not initialised and that `/factory "<idea>"` starts it. Never bootstrap for `status` |
| Arguments contain `--child` | `references/block-runner.md` § Child mode (one block, then stop) |
| `.workflow/state.json` does not exist | `references/bootstrap.md`, then continue the loop |
| Argument `config` | `references/models.md` § Reconfigure and the preferences questions in `references/bootstrap.md` § Questions, then stop |
| `pending_question` is set in the state | `references/stops.md` § Ask a pending question |
| Another session holds the lock (`references/state.md` § Lock) | Stop and say which session is running and since when |
| phase `ideation` | `references/brainstorm.md` |
| phase `spec`, status `drafting` or `idle` | `references/spec.md` |
| phase `spec`, status `awaiting_approval` | `references/approve.md` § Spec |
| phase `build`, status `next` or `in_progress` | `references/build.md` (argument `next-block` = one block only) |
| phase `build`, status `milestone_review` | `references/approve.md` § Milestone |
| phase `build`, status `blocked` | `references/stops.md` § Blocked block |
| phase `build`, status `paused_quota` | `references/stops.md` § Quota pause |
| phase `build`, status `changing` or `change_review` | `references/change.md` |
| phase `done` | Final summary (`references/stops.md` § Done), then stop |

The arguments `--auto-brainstorm`, `--gate spec`, `--gate milestone` and `--unattended` are saved in the state on first use (`references/state.md`), so that a later `/factory` with no arguments resumes with the same behaviour.

## Reference map

| File | When to read it |
|---|---|
| `references/state.md` | Always, at the start: state schema, transitions, lock, journal |
| `references/hosts.md` | Bootstrap, or when a capability is missing |
| `references/continuity.md` | Before launching any role |
| `references/models.md` | Before launching any role |
| `references/bootstrap.md` | First run in a project |
| `references/brainstorm.md` | Phase ideation |
| `references/spec.md` | Phase spec |
| `references/approve.md` | Spec approval, milestone end |
| `references/permissions.md` | Spec approval (Claude Code) |
| `references/build.md` | Build loop |
| `references/block-runner.md` | Running one block (any mechanism) |
| `references/change.md` | `/change`, and states `changing`/`change_review` |
| `references/stops.md` | Every stop, every question, the final summary |
