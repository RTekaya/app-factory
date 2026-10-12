# Phase: build loop

You run the build one block at a time. Each block runs in a fresh context through the runner chosen at bootstrap (`block-runner.md`). Your context only receives each block's short JSON result.

## Preflight (each time you enter the build)

1. `status=in_progress` when you start: the previous run died mid-block (crash, compaction, killed session).
   - The working tree holds partial work: `git stash push -u -m "wip <current_block> interrupted run"`.
   - Append an `interrupted` journal line that mentions the stash, then set `status=next, current_block=null`.
   - Runner B: also look for `.workflow/runs/<current_block>.result.json`. If it exists with `outcome` set, the child finished but the parent died: apply that result (step 4 of the loop) instead of stashing.
2. `status=next` and `git status --porcelain` is not empty → set `status=blocked, note="dirty working tree before a block"`, then go to `stops.md`.
3. Reset `blocks_this_run=0` when this is a new `/factory` invocation.

## Loop

Repeat:

1. **Limits**:
   - `preferences.budget.max_blocks_per_run` reached → stop with a short summary; the next `/factory` continues.
   - `max_usd_total` reached → `stops.md` § Budget.
   - At `warn_at_percent` of a total cap, notify once.
2. **Pick the next block** from `docs/plan.json`: the first block with `status: "todo"` whose `deps` are all `done` or `skipped`.
   - None left (no `todo` and no `in_progress`) → set `phase=done, status=done`, and go to `stops.md` § Done.
   - `todo` blocks remain but none is ready → set `status=blocked, note="dependency deadlock: <ids>"`, then go to `stops.md`.
3. **Announce**: set `status=in_progress, current_block=<ID>`.
4. **Run the block** with the runner (`block-runner.md` § Runner A / B / C). Wait for its result JSON:
   ```json
   {"block":"B04","outcome":"done|blocked|quota","attempts":2,"models":["sonnet","opus"],
    "review_rounds":1,"commit":"<sha or null>","pushed":true,"summary":"…","next_block":"B05",
    "missing_info":"","cost_usd":null}
   ```
5. **Record**: write a `block` journal line, add `cost_usd` to `cost_usd_total` when it is known, and increase `blocks_this_run`.
6. **Branch on the outcome**:
   - `done`: if every block of its milestone is now `done` or `skipped`, run `git tag m<k>-done` (and push the tag if pushing is enabled and a remote exists), then set `status=milestone_review, current_block=null`. Otherwise set `status=next, current_block=null`. Tell the user in one line: `B04 Login screen: done · 2 attempts · sonnet→opus`.
   - `blocked`: set `status=blocked, blocked_block=<ID>, current_block=null, note="<ID> blocked: <reason>"`, then go to `stops.md` § Blocked block (it handles the one automatic retry).
   - `quota`: go to `stops.md` § Quota pause.
7. **No-progress guard**: if a run returns `done` but `git rev-parse HEAD` did not change, or two runs in a row return without changing the state or HEAD, set `status=blocked, note="no progress"` and go to `stops.md`.
8. Go back to step 1. Argument `next-block`: stop after one block.

## What you never do here

- Read source files, diffs or logs. That is the roles' job; you read results.
- Edit `docs/plan.json` statuses: the doc-keeper writes `done`/`blocked` in the block's commit. You only write the state.
- Skip the runner because "it is a small block": every block goes through verify and review.
