# Workflow state

`.workflow/state.json` holds where the workflow stands. Only the orchestrator (the main `/factory` session) writes it. Child sessions (runner B) and roles never write it; a child returns its result through `.workflow/runs/` (`block-runner.md`).

## Schema

The initial content is `templates/state.json`.

| Field | Type | Meaning |
|---|---|---|
| `schema` | int | Schema version of this file (1) |
| `factory_version` | string | App Factory version that bootstrapped the project |
| `host` | string | `claude-code`, `codex`, `gemini-cli`, `cursor`, `copilot`, or another host name |
| `capabilities` | object | `{subagents, nested_subagents, ask_choice, headless_cli, permission_rules, notifications}`, all booleans (`hosts.md`) |
| `runner` | string | Block mechanism: `A` (nested subagent), `B` (child headless session) or `C` (flat) (`block-runner.md`) |
| `phase` | string | `ideation`, `spec`, `build` or `done` |
| `status` | string | See the status table below |
| `flags` | object | `{auto_brainstorm, unattended, gates: []}`, saved from the arguments |
| `current_block` | string or null | Block being built |
| `blocked_block` | string or null | Block that stopped the build |
| `spec_version` | int | Approved spec version (0 before the first approval) |
| `spec_fix_rounds` | int | Automatic spec fix rounds used for the current draft (max 2) |
| `auto_retried` | string[] | Blocks that already had their one automatic retry |
| `blocks_this_run` | int | Blocks finished since the last `/factory` start (for `max_blocks_per_run`) |
| `cost_usd_total` | number | Sum of known costs (child sessions report them; otherwise 0) |
| `pending_question` | object or null | A question waiting for the user (`stops.md`) |
| `resume_after` | string or null | ISO time when a quota pause may end |
| `lock` | object or null | `{owner, since}` (see Lock) |
| `note` | string or null | One-line human-readable reason for the current status |
| `updated_at` | string | ISO-8601 UTC time of the last write |

## Status values

| phase | status | Meaning | Next action |
|---|---|---|---|
| ideation | `idle` | No brief yet | Brainstorm |
| spec | `drafting` | Spec pipeline running or interrupted | Run or resume `spec.md` |
| spec | `awaiting_approval` | Spec written and reviewed | `approve.md` § Spec |
| build | `next` | Ready for the next block | `build.md` |
| build | `in_progress` | A block is running (or a run died mid-block) | `build.md` § Preflight |
| build | `milestone_review` | A milestone just finished | `approve.md` § Milestone |
| build | `blocked` | A block or the build is stuck | `stops.md` § Blocked block |
| build | `paused_quota` | Subscription usage limit reached | `stops.md` § Quota pause |
| build | `changing` | A `/change` is being applied | `change.md` |
| build | `change_review` | A `/change` waits for the user's approval | `change.md` § Approve |
| done | `done` | Every block is done or skipped | Final summary |

## How to write

1. Read the file, change only the fields of this transition, set `updated_at` to the current UTC time, and write the whole file back. Never invent fields that are not in the schema.
2. Write the transition **before** starting the action it announces. Example: set `status=in_progress, current_block=B04` before launching block B04, not after.
3. Then append one line to the journal (below).

## Journal

`.workflow/journal.jsonl` is append-only. Write one JSON object per line and never rewrite earlier lines.

```json
{"at":"2026-10-12T14:03:11Z","step":"block","id":"B04","role":"block-runner","runner":"A","model":"sonnet","outcome":"done","summary":"Login screen + auth repository","handoff":"docs/handoffs/B04.md","cost_usd":null}
```

- `step` values: `bootstrap`, `brainstorm`, `spec`, `spec-fix`, `approve`, `block`, `milestone`, `stop`, `answer`, `change`, `quota`, `resume`.
- Write one line for each finished step, each stop, each answer, and each interrupted run discovered at startup (`outcome: "interrupted"`).
- When a milestone ends, also append a `milestone` line that sums the milestone: blocks, attempts, models used, known cost.

## Lock

Two orchestrators must never run on the same project at the same time.

- At start, if `lock` is null, write `lock = {owner: "<host>:<short random id>", since: <now>}` and keep the id for this session.
- If `lock` is set by another owner:
  - If `updated_at` is less than 2 hours old, another session is probably running: stop and say so. If the user confirms in this session that nothing else is running, take over the lock and record that answer in `docs/DECISIONS.md`.
  - If `updated_at` is older than 2 hours, treat the old run as dead: take the lock, append a `resume` journal line, and continue.
- Set `lock` back to null whenever you stop.

## Files kept in git, and files kept local

| Path | In git | Why |
|---|---|---|
| `.workflow/state.json`, `.workflow/journal.jsonl`, `.workflow/runs/` | no | Machine-local run state; changes after every commit |
| `.workflow/preferences.json`, `.workflow/models.json`, `.workflow/gate.json` | yes | Project configuration |
| `docs/plan.json` and every other file under `docs/` and `design/` | yes | The spec, the plan statuses and the handoffs |

To continue on another machine, run `/factory`: when `state.json` is missing but `docs/plan.json` exists, the bootstrap rebuilds the state from git (`bootstrap.md` § Rebuild).
