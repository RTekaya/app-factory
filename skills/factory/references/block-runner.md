# Block runner: one block, a fresh context, a short result

The **block runner** builds exactly one block of `docs/plan.json`: develop → verify → review → docs → commit. It runs in a context of its own and returns only a short JSON result. The procedure (§ Procedure) is the same everywhere; three runners differ only in *where* it runs.

## Contract

- **Input**: the block ID, plus a task packet (`continuity.md`) with SKILL_DIR, the project root, and the resolved models for every role (`models.md`).
- **Output**: one JSON object matching `templates/block-result.schema.json`:
  ```json
  {"block":"B04","outcome":"done|blocked|quota","attempts":2,"models":["sonnet","opus"],
   "review_rounds":1,"commit":"<sha or null>","pushed":true,"summary":"<one line>",
   "next_block":"B05","missing_info":"","cost_usd":null}
  ```
- **Side effects**: one commit on the main branch (`feat(<ID>)` when done, `chore(<ID>)` when blocked). On a block, also a `wip/<ID>-attempt` branch. `HANDOFF.md`, `docs/handoffs/<ID>.md` and the plan status updated by the doc-keeper.
- **Never**: ask the user anything, write `.workflow/state.json`, or start another block.

## Runner A: nested subagent (Claude Code)

Launch the role `block-runner` (Claude Code: `app-factory:block-runner`) **in the foreground**, with the packet and `model` = the resolved `standard` model. It follows § Procedure and launches its own role subagents, also in the foreground. Its final message is the result JSON.

## Runner B: child headless session (any host with a CLI)

1. `mkdir -p .workflow/runs`, then start the child **in the background** (Claude Code: Bash with `run_in_background: true`), from the project root:
   ```
   claude -p "/factory --child next-block <ID>" --model <child model> --permission-mode <child mode> --output-format json --json-schema "$(cat <SKILL_DIR>/templates/block-result.schema.json)" > .workflow/runs/<ID>.out.json 2> .workflow/runs/<ID>.err.log
   ```
   - `<child mode>` is `preferences.child_permission_mode` (default `auto`).
   - Add `--max-budget-usd <n>` only if `preferences.budget.max_usd_per_child_session` is set.
   - On other hosts, use the headless command from `hosts.md`, and drop the flags the host does not have.
2. Wait for the completion notification. Do not poll in a tight loop. If the host cannot run a background command for an hour, do not use runner B; use C.
3. Read the result, in this order of preference:
   1. `.workflow/runs/<ID>.result.json` (the child writes it before it ends);
   2. `structured_output` in `.workflow/runs/<ID>.out.json`;
   3. the JSON block at the end of its `result` field.
   Take `cost_usd` from `total_cost_usd`.
4. If the output says the usage limit was reached (`is_error` true, with "limit" or "quota" in the text), the outcome is `quota`. If there is no result at all, the outcome is `blocked`, with the reason "child session ended without a result; see .workflow/runs/<ID>.err.log".

## Runner C: flat

You (the orchestrator) follow § Procedure yourself, launching each role as a subagent in the foreground. Without subagents, play each role in sequence from its role file. After the block, stop and tell the user to start a new session and run `/factory` for the next block: that is how C keeps a fresh context per block.

## Child mode (`/factory --child next-block <ID>`)

You are a child session started by runner B.
- Never take the lock, never write `.workflow/state.json`, never ask questions.
- Follow § Procedure for block `<ID>`.
- Before ending, write the result JSON to `.workflow/runs/<ID>.result.json`, then end your final message with the same JSON.

## Procedure

Track: `attempts` (developer runs), the models used, every verifier digest and reviewer finding.

### 1. Brief

Read the block in `docs/plan.json` (title, milestone, deps, complexity, notes) and its section in `docs/05-plan.md`. Build the developer's task packet:
- objective = the block's goal;
- read = `HANDOFF.md` first, then the architecture and design-system sections the block's notes point to, then the UX screens it implements;
- the acceptance criteria, copied verbatim;
- out of scope, copied verbatim;
- the decisions to respect, from `docs/DECISIONS.md`;
- the known pitfalls, from HANDOFF "Attention" and the block's notes (for example a `wip/` branch from an earlier attempt to inspect with `git diff HEAD..wip/<ID>-attempt`).

Models: the developer and the code-reviewer follow the complexity table in `models.md`.

### 2. Develop

Launch **developer** with the brief. If its report says `"status":"blocked"`, go to step 5 with its reason.

### 3. Verify (at most 2 fix rounds)

Launch **verifier** (tier `fast`). It runs § Gate and reports PASS or a digest of at most 15 lines.
- FAIL: send the digest to the developer. Resume the same developer if the host can; otherwise launch a new developer with the brief, the digest, and "your previous attempt is in the working tree". Then verify again. At most **2** fix rounds.

### 4. Review (at most 2 rounds)

Launch **code-reviewer** with the brief and the developer's report.
- `CHANGES_REQUESTED`: send the blocking items to the developer, then verify, then review again. At most **2** review rounds.

### 5. Escalate once

If verify or review still fails after its rounds, or the developer reported blocked: launch **developer-escalated** (tier `deep`, fresh context) once. Give it the brief, every digest and finding, and "a previous attempt is in the working tree: fix it or rework it". Then verify once and review once.

### 6a. Done path (verify PASS and review PASS)

1. Launch **doc-keeper** with: the block ID and title, outcome `done`, the final developer report (including `missing_info`), the reviewer's verdict and non-blocking notes, the attempts and the models used. It updates the docs, sets the block to `done` in `docs/plan.json`, and writes `HANDOFF.md` and `docs/handoffs/<ID>.md`.
2. `git add -A && git commit -m "feat(<ID>): <block title>"`.
3. Push, following `.workflow/preferences.json`:
   - `git_push: "always"`: if `git remote get-url origin` works, run `git push`. If there is no remote, `create_private_github_repo` is true, and `gh auth status` works, run `gh repo create <folder name> --private --source . --push`.
   - `git_push: "if_remote"`: push only if a remote exists.
   - `git_push: "never"`: skip.
   - If a push fails, do not retry with force. Report `pushed: false` and the reason in `summary`.
4. Result: `outcome: "done"`, `commit` = the new HEAD.

### 6b. Blocked path

1. Park the code: `git switch -c wip/<ID>-attempt`, `git add -A`, `git commit -m "wip(<ID>): blocked attempt"`, `git switch -` (the main branch is clean again). If the branch already exists, use `wip/<ID>-attempt-2`, and so on.
2. Launch **doc-keeper** with outcome `blocked`, the last digests and findings, the attempts, the models, and the wip branch name. It writes the blocker analysis in `HANDOFF.md` and sets the block to `blocked` in `docs/plan.json`.
3. `git add -A && git commit -m "chore(<ID>): blocked, see HANDOFF.md"`.
4. Result: `outcome: "blocked"`, and `summary` = the one-line reason.

## Gate

`.workflow/gate.json` (`templates/gate.json`):

```json
{ "configured": false,
  "builtin": ["frozen-docs", "secrets"],
  "commands": [ { "name": "lint", "run": "flutter analyze" } ],
  "timeout_minutes": 20 }
```

The verifier runs, in order:
1. **frozen-docs** (built in): list the changed files with `git diff --name-only HEAD` plus `git ls-files --others --exclude-standard`. FAIL if the list contains `docs/00-brief.md`, `docs/01-prd.md` or `docs/02-ux.md`, unless `.workflow/state.json` has `status: "changing"`.
2. **secrets** (built in): FAIL if a changed or new file matches `.env`, `.env.*`, `*.pem`, `*.key`, `*.p8`, `*.p12`, `*.jks`, `*.keystore`, `key.properties`, or `*service-account*.json`.
3. **configured**: if `configured` is false, FAIL with "quality gate not configured: block B01 must add the stack's commands to .workflow/gate.json". (B01 itself fixes this as part of its acceptance criteria.)
4. Each entry of `commands`, in order, non-interactively. Stop at the first failure.

Block B01 (walking skeleton) writes `commands` from architecture §8 and sets `configured: true`. After B01, changing `commands` is a deviation the reviewer must justify; removing or weakening a command is always BLOCKING.
