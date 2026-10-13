# App Factory

**From an app idea to a finished, tested and documented app, with one command: `/factory`.**

App Factory is a multi-agent workflow for AI coding agents. It brainstorms the idea with you, writes a full spec (PRD, UX, design system, architecture, plan), then builds the app **block by block**. Each block runs in a fresh context: it is developed, run through a quality gate, reviewed by an independent agent, documented, and committed. A handoff file carries the knowledge to the next block. You step in only for the brainstorm and when the workflow genuinely needs a decision, which it presents as a multiple-choice question with a recommended option.

No shell scripts. It's all Markdown skills and roles, plus your agent's own tools.

## Install

**Claude Code** (plugin, updated with `/plugin update`):

```bash
claude plugin marketplace add RTekaya/app-factory && claude plugin install app-factory@app-factory
```

**Codex, Cursor, Gemini CLI, GitHub Copilot and other agents** (and Claude Code, if you prefer editable files):

```bash
npx skills@latest add RTekaya/app-factory
```

Then, in an empty folder:

```
/factory "a habit tracker web app: one screen, no account, data stays in the browser"
```

## How it works

```
idea ─▶ brainstorm + grill ─▶ PRD ─▶ UX ─▶ design system ∥ architecture ─▶ plan ─▶ spec review
                                                                                       │
        ┌──────────────────────────────────────────────────────────────────────────────┘
        ▼
   approval (auto, or your gate) ─▶ for each block: develop ─▶ verify ─▶ review ─▶ docs + handoff ─▶ commit
                                                 ▲   ≤2 fixes   │  ≤2 rounds │
                                                 └──────────────┴────────────┘
                                    still failing → 1 escalation → 1 auto-retry → diagnosis + question for you
```

| Phase | What happens | You |
|---|---|---|
| Brainstorm | The orchestrator challenges the idea, offers angles, cuts the MVP, then **grills** the draft brief for blind spots | You answer (or `--auto-brainstorm`) |
| Spec | Specialised roles write `docs/01-prd.md` … `05-plan.md`, `design/tokens.json`, ADRs. An independent reviewer checks traceability and consistency; up to 2 automatic fix rounds | Optional `--gate spec` |
| Approval | PRD, UX and brief are **frozen**; the spec is tagged `spec-vN` | Automatic unless gated |
| Build | One block at a time, each in a fresh context, until the plan is done | Optional `--gate milestone` |
| Stops | Blocked block, unready spec, budget, quota, or a `/change`: a diagnostician proposes 2–4 options | You pick one |

### Roles

| Role | Tier | Writes |
|---|---|---|
| product-manager | deep | `docs/01-prd.md` |
| ux-designer | deep | `docs/02-ux.md` |
| design-system | standard | `docs/03-design-system.md`, `design/tokens.json` |
| architect | deep | `docs/04-architecture.md`, ADRs, agent permissions |
| planner | deep | `docs/05-plan.md`, `docs/plan.json` |
| spec-reviewer | deep | `docs/reviews/spec-review.md` |
| block-runner | standard | nothing (orchestrates one block) |
| developer / developer-escalated | standard / deep | code and tests |
| verifier | fast | nothing (runs the quality gate) |
| code-reviewer | deep (standard for low complexity) | nothing (verdict) |
| doc-keeper | standard | living docs, `HANDOFF.md` |
| diagnostician | deep | nothing (options for you) |

Roles ask for a **tier** (`deep`, `standard`, `fast`), not a model. You map tiers to your AI's models: Claude defaults are Opus, Sonnet and Haiku.

### Nothing is lost between sessions

Every role receives a task packet and must return a JSON report and write its handoff before the workflow moves on. Four files carry the memory:

- `HANDOFF.md`: what the last block did, the contracts now available, pitfalls, and how to verify.
- `docs/handoffs/phase-*.md`: one handoff per phase and milestone, including what was rejected and why.
- `docs/DECISIONS.md`: every answer you gave, append-only.
- `.workflow/journal.jsonl`: one line per step, with the model actually used.

State lives in `.workflow/state.json`. A crash, a context compaction or a killed session costs at most the step in progress: run `/factory` again and it resumes.

### Fresh context per block

| Runner | How | When |
|---|---|---|
| A | A `block-runner` subagent launches the role subagents | Hosts with nested subagents (Claude Code) |
| B | A child headless session per block (`claude -p`, `codex exec`, `gemini -p` …) | Any host with a headless CLI; also used for unattended runs |
| C | One block per session | Last resort |

## Commands

| Command | Effect |
|---|---|
| `/factory "idea"` | Start: bootstrap, brainstorm with you, then everything else |
| `/factory` | Resume where it stopped (asks a pending question first) |
| `/factory --auto-brainstorm "idea"` | No interaction: the brainstorm decides alone and logs its assumptions |
| `/factory --gate spec` · `--gate milestone` | Add a pause after the spec / at each milestone |
| `/factory --unattended` | Never ask; questions are saved and asked at the next `/factory` |
| `/factory answer <choice>` | Answer a pending question without an interactive prompt |
| `/factory next-block` | Build one block only |
| `/factory status` · `config` | Show progress · change models, budget, git and gates |
| `/factory --runner A\|B\|C` | Force a block runner |
| `/change "<what and why>"` | Change the scope during the build (owners revise, review, your approval) |

Unattended overnight run (Claude Code):

```bash
claude -p "/factory --unattended" --permission-mode auto
```

## Configuration

Asked once, at the first `/factory`, and changeable with `/factory config`:

- **Models**: presets *Balanced* (default), *Max quality*, *Economy*, or *Custom* (your own model names per tier). Saved in `.workflow/models.json`, with a user default in `~/.app-factory/models.json`.
- **Budget**: no cap by default (subscriptions). Optional total cap, per-block-session cap, or blocks per run. A subscription usage limit pauses the build (`paused_quota`) instead of failing it.
- **Git**: push after each block and create a *private* GitHub repo if none exists (default), push only if a remote exists, or never.
- **Gates**: none (default), after the spec, at each milestone, or both.

## Guardrails

| Risk | Protection |
|---|---|
| Scope drift | Brief, PRD and UX are frozen after approval. The quality gate fails any block that touches them; in Claude Code a native `deny` rule also blocks every write. Only `/change` reopens them |
| Weakened tests | The reviewer treats any weakening of tests, lint or the gate as blocking |
| Secrets | `.gitignore` and `deny` rules for `.env`, keys and keystores; the gate fails if a secret file is in a diff |
| Destructive commands | `deny` rules for force push, hard reset, `git clean`, `rm -rf`, deploy and publish |
| Infinite loops | 2 fix rounds + 2 review rounds, 1 escalation, 1 auto-retry, then a question for you |
| Two runs at once | A lock in the state |

The guardrails are safety rails, not a sandbox. For long unattended runs, a container or VM is still the safest place.

## Compatibility

| Host | Status | Notes |
|---|---|---|
| Claude Code 2.1.296+ (CLI and Desktop) | ✅ Verified end to end | Runners A and B, clickable questions, native guardrails |
| Gemini CLI | 🧪 Not yet verified | Runner B via `gemini -p` |
| Codex CLI | 🧪 Not yet verified | Runner B via `codex exec` |
| Cursor CLI | 🧪 Not yet verified | |
| GitHub Copilot CLI | 🧪 Not yet verified | |

The workflow, documents and portable guardrails are the same everywhere. What varies is comfort (clickable vs numbered questions), cost (per-role models) and automation (runner A, B or C). Reports from other hosts are welcome; see [CONTRIBUTING.md](CONTRIBUTING.md).

## What it costs

Measured on a small web app (one screen; TypeScript, Vite, Preact), with the *Balanced* preset. Amounts are API-equivalent; on a subscription, they count against your usage limits:

| Step | Cost |
|---|---|
| Brainstorm + full spec + review (mostly Opus) | ≈ $20 |
| Milestone 1: walking skeleton + lint guardrails (2 blocks) | ≈ $4 |
| Milestone 2: domain, storage, store, UI, two vertical slices (8 blocks) | ≈ $14 |

The spec is the expensive part, and it is paid once. The *Economy* preset lowers it.

## Tips

- **Claude Code**: open the project once interactively and accept the folder trust dialog. Until then, headless sessions ignore the project's `allow` rules.
- In `auto` permission mode, Claude Code may refuse to let the session edit its own `.claude/settings.json`. App Factory then keeps the portable guardrails and tells you the rules to add.
- Run the brainstorm on your strongest model: everything downstream inherits its decisions.

## Contributing

Issues and pull requests are welcome; see [CONTRIBUTING.md](CONTRIBUTING.md). The eval suite runs with `claude plugin eval . --allow-tools Bash Write Edit --judge-model sonnet`.

## License

[MIT](LICENSE) © Zenith AI. Contact: contact@zenith-ai.dev
