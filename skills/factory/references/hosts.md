# Hosts and capabilities

App Factory runs on any coding agent that can load a skill, read and write files, and run shell commands. It adapts to what the host can do. It reasons about **capabilities**, not brands.

## Detect capabilities (at bootstrap)

Check what *this* session can actually do, and record the result in `state.capabilities`:

| Capability | How to tell | Used for |
|---|---|---|
| `subagents` | You have a tool that launches a subagent with its own context (Claude Code: `Agent`/`Task`) | Roles in a fresh context |
| `nested_subagents` | A subagent can itself launch subagents. Claude Code: true (verified). Elsewhere: false unless the host's fiche below says verified | Runner A |
| `ask_choice` | You have a tool that asks the user a multiple-choice question (Claude Code: `AskUserQuestion`) | Clickable questions; otherwise numbered text questions |
| `headless_cli` | The host's CLI is on the PATH and has a non-interactive mode (fiche below; check with `command -v <cli>`) | Runner B, unattended runs |
| `permission_rules` | The host enforces declarative deny rules from a project file (Claude Code: `.claude/settings.json`) | Native guardrails (doc freeze, secrets) |
| `notifications` | You have a push notification tool (Claude Code: `PushNotification`) | Notify at stops and at the end |

Detection is done by checking, not by guessing:
- `headless_cli`: run `command -v claude codex gemini cursor-agent copilot` and keep the CLIs that exist. The capability is true if the CLI of *this* host is among them.
- `subagents`, `ask_choice` and `notifications`: look at the tools you actually have in this session.

**Re-detect at every start.** `ask_choice` and `notifications` depend on the session type: an interactive session has them, a headless `-p` session usually does not. At the start of every `/factory`, refresh these two fields from your current tools before routing. The other capabilities only change when the host is updated.

## Choose the runner

1. `subagents` and `nested_subagents` → **A**
2. otherwise `headless_cli` → **B**
3. otherwise → **C**

The user can force a runner with `/factory config`.

## Host fiches

"Verified" means tested end to end with App Factory. Unverified entries come from the host's documentation. Re-check the flags with `<cli> --help` before first use, and update this table through a pull request.

### Claude Code (verified, 2.1.296)

- Instruction file: `CLAUDE.md`. Permission file: `.claude/settings.json`.
- Subagents: yes, nested: yes. Subagents may run in the background by default: always pass `run_in_background: false`. After a block runner returns, check that its result JSON is complete; if a role it launched is still running, stop that role and rerun the block (`build.md` § Preflight) rather than continuing. Roles are the plugin's agents `app-factory:<role>` (when installed with skills.sh instead of the plugin, use a general-purpose subagent and pass the role file path in the packet).
- Choice questions: `AskUserQuestion`. Notifications: `PushNotification`.
- Headless child session:
  `claude -p "<prompt>" --model <model> --permission-mode <mode> --output-format json --json-schema '<schema>' [--max-budget-usd <n>]`
  The JSON result has `result`, `total_cost_usd`, `is_error` and `permission_denials`. Structured output goes in `structured_output` when `--json-schema` is used. If it is missing, parse the JSON block at the end of `result`.
- Model aliases: `opus`, `sonnet`, `haiku` (the subagent `model` parameter accepts aliases only; full model IDs work with `--model`).
- Deny rules: an `Edit(<path>)` deny rule blocks Edit, Write and shell writes to that file in every permission mode, including inside subagents. A rule added during a session applies immediately.
- Workspace trust: headless sessions (`claude -p`) **ignore the project's `permissions.allow` rules until the folder has been trusted once** in an interactive session (the trust dialog). `deny` rules still apply. At bootstrap in an interactive session, nothing to do. Unattended bootstrap: tell the user in the final message to open the folder once with `claude` and accept the trust dialog, so the build commands stop depending on the auto mode classifier.
- Editing `.claude/settings.json` from a session: allowed in `bypassPermissions`, and with the user's approval in interactive modes. In `auto` mode, the classifier may deny it as self-modification, even when a skill asks for it (seen at spec approval). Follow `approve.md` § Freeze for the fallback. Bootstrap merges usually pass because they only add base rules.

### Gemini CLI (unverified)

- Instruction file: `GEMINI.md` (also reads `AGENTS.md` when configured).
- Headless: `gemini -p "<prompt>" -m <model> --approval-mode yolo -o json`.
- Subagents: check the installed version. If there are none, use runner B (one `gemini -p` session per block) or C.

### Codex CLI (unverified)

- Instruction file: `AGENTS.md`.
- Headless: `codex exec "<prompt>" -m <model> --full-auto` (check `codex exec --help`).

### Cursor CLI (unverified)

- Instruction file: `AGENTS.md` or `.cursor/rules/`.
- Headless: `cursor-agent -p "<prompt>" --model <model>` (check `cursor-agent --help`).

### GitHub Copilot CLI (unverified)

- Instruction file: `AGENTS.md` or `.github/copilot-instructions.md`.
- Headless: `copilot -p "<prompt>" --model <model> --allow-all-tools` (check `copilot --help`).

## When a capability is missing

| Missing | Fallback |
|---|---|
| `subagents` | Play each role yourself, in sequence, by reading its `roles/<role>.md` file and following it strictly. Finish each role with its JSON report before starting the next. Prefer runner B for the build so that each block still gets a fresh session |
| `ask_choice` | Ask a numbered question in plain text: the problem in at most 3 lines, options `1.` to `4.` with the recommended one first and marked "(recommended)", and "or describe another option". Wait for the answer |
| `permission_rules` | The portable checks in the quality gate still apply (`block-runner.md` § Gate). Say once, at bootstrap, that the native guardrails are not available on this host |
| `notifications` | Put the message in your final reply |
| model choice per role | Everything runs on the session's model. Record the actual model in the journal |
