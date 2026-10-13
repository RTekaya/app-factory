# Stops, questions and the end

The workflow stops only when a human is needed, or when a limit says so. Every stop that needs a decision ends with a **multiple-choice question**: 2 to 4 options, one recommended, and free text always allowed.

## How to stop

1. Write the state first: `status`, `note`, and `pending_question` when a decision is needed.
2. Append a `stop` journal line.
3. Notify (`hosts.md`): `⛔ <one-line reason>`, or `✅ …` at the end.
4. Release the lock (`lock = null`).
5. Final message, in at most 8 lines: what happened, what was tried, what you need, and how to resume (`/factory`).

## The diagnosis and the question

For a blocked block, a spec that is not ready, or a no-progress or deadlock stop:

1. Launch **diagnostician** (tier `deep`, read-only) with a packet that points at `HANDOFF.md`, `.workflow/journal.jsonl` (last 20 lines), the `note`, the review files, `docs/plan.json`, and the `wip/` branches. It returns:
   ```json
   {"summary":"≤ 3 lines","options":[{"label":"≤ 5 words","description":"…","action":"retry|retry_high|skip|change|edit_gate|pause|custom","details":"…","recommended":true}]}
   ```
2. Save it as `pending_question` in the state, so a dead session or an unattended run loses nothing.
3. **Interactive session** (`flags.unattended` false and you are the main session): ask now (§ Ask a pending question).
4. **Unattended or child**: stop. The next `/factory` asks it.

## Ask a pending question

1. Show the `summary` (3 lines).
2. Ask the question. Claude Code: `AskUserQuestion`, with the recommended option first and "(Recommended)" appended to its label. Elsewhere: a numbered text question (`hosts.md`).
3. Append the answer to `docs/DECISIONS.md`, and only then clear `pending_question`.
4. Apply the action:

| Action | Effect |
|---|---|
| `retry` | Block → `todo` in `docs/plan.json` (add the wip branch to its notes); commit `chore(<ID>): retry`; `status=next`. **Never merge a `wip/` branch into the main branch**: its code has not passed verify and review. The developer brings it back as uncommitted changes (§ Procedure step 1 of `block-runner.md`) |
| `retry_high` | Same as `retry`, and set the block's `complexity` to `high` |
| `skip` | Block → `skipped`; warn about the blocks that depend on it (they will wait or deadlock); commit; `status=next` |
| `change` | Tell the user to run `/change <description>`; stop |
| `edit_gate` | Apply the diagnostician's `details` to `.workflow/gate.json` (the user just approved it); commit `chore(gate): <what>`; then `retry` |
| `pause` | Keep the state, stop |
| `custom` / free text | Do what the user wrote if it fits the workflow; otherwise ask once more |

## Blocked block (phase `build`, status `blocked`)

1. If `blocked_block` is set and is **not** in `auto_retried`: retry automatically **once**, without asking. Add the block to `auto_retried`, apply `retry_high`, append a journal line, notify "B07 retried automatically on the deep tier", and go back to the build loop.
   **Exception**: skip the automatic retry when it cannot help. That is the case when the escalated attempt already ran on the `deep` tier **and** the HANDOFF analysis puts the cause outside the code: the environment or toolchain, a missing credential or service, or a spec or decision conflict. A retry would only repeat the failure. Say why in the journal line (`"auto-retry skipped: <cause>"`) and go to step 2.
2. Otherwise (second block, deadlock, no progress, dirty tree): run § The diagnosis and the question.

## Spec not ready

The review still says `NEEDS_FIXES` after 2 automatic fix rounds. The diagnostician proposes options such as: apply its suggested fixes, approve anyway (list the risks), answer the PRD's open questions, or edit the brief. Each option names the owners that would revise.

## Quota pause (`paused_quota`)

The subscription's usage limit was reached. This is not a failure.

1. Set `status=paused_quota`. Set `resume_after` to the reset time if the error gives one, otherwise to now + 1 hour. Keep `current_block`: the block will be retried from a clean tree.
2. Park any partial work: `git stash push -u -m "wip <ID> quota pause"`.
3. Append a `quota` journal line, notify "Usage limit reached; resumes after <time>", and stop.
4. On the next `/factory`, if the time has passed, set `status=next` and continue. If it is earlier and the session is interactive, tell the user the time. In unattended mode, wait (Claude Code: schedule a wake-up or background sleep until `resume_after`, when available) and then continue.

## Budget

Only when the user set a cap (`preferences.budget`):
- `max_usd_total` reached → stop with a question: raise the cap (the user types the amount), remove the cap, or pause.
- `max_blocks_per_run` reached → stop without a question: "N blocks done in this run; run /factory to continue".

## Done (phase `done`)

1. Write `docs/handoffs/phase-done.md`: what was built, every milestone, debt left, known cost, models used per tier, and how to run the app.
2. Commit it, notify `✅ <App>: all blocks done`, release the lock.
3. Final message: what to try first, where the docs are, and suggested next steps (QA on a device, release preparation).
