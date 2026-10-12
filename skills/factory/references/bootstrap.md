# Bootstrap (first `/factory` in a project)

Goal: prepare the project folder in one pass, without overwriting anything the user already has.

## 0. Rebuild instead?

If `.workflow/state.json` is missing but `docs/plan.json` exists, the project was started elsewhere (another machine, or a fresh clone). Go to § Rebuild.

## 1. Check the folder

1. Run `git rev-parse --show-toplevel`. If it fails, plan to run `git init`.
2. List the top level. If it already contains application source code but no `docs/00-brief.md`, warn the user that App Factory is designed to build a new app from an idea. Ask whether to continue here (the brainstorm and the architect will then have to take the existing code into account) or to stop. In unattended mode, stop with a pending question.

## 2. Detect the host

Follow `hosts.md` § Detect capabilities, then choose the runner. Keep the result; it goes into the state in step 4.

## 3. Questions

Ask everything in **one** question round (Claude Code: one `AskUserQuestion` call with up to 4 questions; elsewhere: one numbered message). Put the recommended option first. Skip a question when `~/.app-factory/preferences.json` or `~/.app-factory/models.json` already answers it: use those values and say so in one line.

1. **Models**: Balanced (recommended) · Max quality · Economy · Custom. On a host other than Claude Code, always ask for the model names of each tier (`models.md` § Presets).
2. **Budget**: No cap (recommended for subscriptions) · Total cap in USD · Cap per block session in USD. If the user picks a cap, ask for the amount.
3. **Git**: Push after each block and create a private GitHub repository if there is none (recommended; `git_push: "always"`, `create_private_github_repo: true`) · Push only if a remote already exists (`"if_remote"`) · Never push (`"never"`).
4. **Human gates**: None, fully automatic (recommended) · Pause after the spec · Pause at every milestone · Both.

`--unattended` without these files: use the recommended options, and list them in your final message so the user can change them with `/factory config`.

Record every answer in `docs/DECISIONS.md` (step 4 creates it), in the format of `templates/DECISIONS.md`.

## 4. Create the files

Never overwrite an existing file. When a file exists, merge as described below, or list it as "kept" in your summary.

| Create | From | Notes |
|---|---|---|
| `.workflow/state.json` | `templates/state.json` | Fill `factory_version` (from `plugin.json` or "unknown"), `host`, `capabilities`, `runner`, `flags` (from the arguments and the gates answer), `updated_at` |
| `.workflow/preferences.json` | `templates/preferences.json` | Answers 2 to 4 |
| `.workflow/models.json` | `templates/models.json` | Answer 1. If `~/.app-factory/models.json` does not exist, also save it there and say so |
| `.workflow/gate.json` | `templates/gate.json` | Only the built-in checks for now; block B01 adds the stack's commands |
| `docs/DECISIONS.md` | `templates/DECISIONS.md` | Then append the answers from step 3 |
| `docs/handoffs/`, `docs/adr/`, `docs/reviews/`, `design/` | — | Empty folders: add a `.gitkeep` |
| `.gitignore` | `templates/gitignore.txt` | Append the lines that are missing |
| Instruction file section | `templates/instructions-section.md` | Use the host's instruction file (`hosts.md`): `CLAUDE.md` for Claude Code, otherwise `AGENTS.md`. Insert or replace the block between `<!-- app-factory:start -->` and `<!-- app-factory:end -->`. Never touch text outside the markers |
| `.claude/settings.json` (Claude Code only) | `templates/settings.json` | Merge: add the missing `allow` and `deny` entries, and keep everything else. Show the user the entries you add. In runner B, also add `Bash(claude -p *)` to `allow` |

Then:
1. Run `git init` if needed.
2. `git add -A && git commit -m "chore(factory): bootstrap App Factory"`. If the folder already had unrelated uncommitted changes, do not commit them: add only the files you created or merged.
3. Write a `bootstrap` journal line.

## 5. Continue

Set `phase=ideation, status=idle` and go back to the routing loop (the brainstorm starts).

## Rebuild

`state.json` is local and not in git. To rebuild it on a new machine:

1. Detect the host and the runner (step 2).
2. Read `docs/plan.json`, the latest `spec-v*` and `m*-done` git tags, and `docs/DECISIONS.md`.
3. Derive the state:
   - no `docs/00-brief.md` → `ideation/idle`;
   - brief but no `spec-v*` tag → `spec/drafting`;
   - otherwise `build`, with status `next` if some block is `todo`, `done` if none is left, `blocked` with `blocked_block` if a block is `blocked`;
   - `spec_version` = the highest `spec-v<n>` tag.
4. Write `state.json`, a `resume` journal line, and tell the user what you derived. Keep the existing `.workflow/*.json` configuration files.
