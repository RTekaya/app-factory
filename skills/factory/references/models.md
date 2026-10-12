# Models: tiers, configuration, application

Roles never name a model: they ask for a **tier**. A configuration file maps each tier to a real model of the user's AI.

## Tiers

| Tier | Purpose | Roles (default) |
|---|---|---|
| `deep` | Decisions that are expensive to undo | brainstorm (main session), product-manager, ux-designer, architect, planner, spec-reviewer, code-reviewer, developer-escalated, diagnostician |
| `standard` | Volume and rule-following | developer, design-system, doc-keeper, block-runner, child sessions |
| `fast` | Mechanical work | verifier |

Per-block routing, from the block's `complexity` in `docs/plan.json`:

| Complexity | developer | code-reviewer |
|---|---|---|
| low | standard | standard |
| medium | standard | deep |
| high | deep | deep |

Escalation, for one extra attempt after the fix rounds (`block-runner.md`): use the `developer-escalated` role at tier `deep`. It has high reasoning effort and starts from a fresh context.

## Configuration

Resolution order (the first file found wins, field by field):
1. `.workflow/models.json` (this project)
2. `~/.app-factory/models.json` (this user, all projects)
3. The preset chosen at bootstrap

Format (`templates/models.json`):

```json
{
  "preset": "balanced",
  "tiers": { "deep": "opus", "standard": "sonnet", "fast": "haiku" },
  "roles": {},
  "child_session_model": "standard"
}
```

- `tiers`: model names as the host understands them. In Claude Code, use aliases (`opus`, `sonnet`, `haiku`): the subagent `model` parameter accepts only aliases. To pin an exact version, the user sets Claude Code's alias environment variables; see the README.
- `roles`: per-role tier overrides, e.g. `{"doc-keeper": "fast"}`.
- `child_session_model`: tier of the child session in runner B.

### Presets

| Preset | Effect |
|---|---|
| `max-quality` | every role `deep`, except the verifier (`fast`) |
| `balanced` (default) | the tier table above |
| `economy` | `deep` only for architect and spec-reviewer; code-reviewer `standard` up to medium complexity; doc-keeper `fast` |
| `custom` | the user types a model name for each tier |

For a host other than Claude Code, ask the user for the model names of their AI for each tier. Never guess model names.

## Applying the model

| Where the role runs | How |
|---|---|
| Subagent (runner A or C) | Pass the resolved model in the subagent call's `model` parameter, **every time**. The role file's `model:` is only a default for Claude Code |
| Child session (runner B) | Pass the CLI's model flag (`hosts.md`) |
| Main session (brainstorm, orchestration) | It runs on the model the user chose. Before the brainstorm, if the session's model is not the `deep` model, tell the user and suggest switching (Claude Code: `/model <deep model>`). Do not block if they decline |
| Host without per-role model choice | Everything runs on the session model; record it in the journal |

Record the model **actually used** in every journal line. At each milestone end, add to the milestone journal line, per tier: attempts, first-review pass rate, and known cost.

## Reconfigure

`/factory config`: show the current resolved configuration. Ask which preset (or custom names), and whether to save it for this project only or as the user default. Write the file and record the change in `docs/DECISIONS.md`.

If a milestone summary shows doc-keeper handoffs never had gaps, or a first-review pass rate above 80% at `medium`, suggest the matching `economy` override to the user once. Never change the configuration without their answer.
