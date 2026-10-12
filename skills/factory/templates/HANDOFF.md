# Handoff: <BLOCK_ID> · <Block title>

| | |
|---|---|
| Outcome | done ✅ / blocked ⛔ |
| Date | YYYY-MM-DD |
| Milestone | M<n>: <k>/<total> blocks done |
| Attempts | <n> (models: <list>) |
| Review | PASS / CHANGES_REQUESTED → PASS after <n> rounds |
| Commit | <sha or "pending"> |

## Next block
<NEXT_ID>: <title> · deps satisfied: yes/no · complexity: <low|medium|high>
Read first: <architecture §, ADR, design-system component, SCR id>

## What changed
- <user-visible change>
- <technical change>

## Key files
- `<path>`: <role>

## Contracts now available
```
<real signatures: functions, classes, components + props, endpoints, routes>
```

## Decisions & deviations
- <decision> → <why> → <doc updated / ADR-NNNN>

## Attention (pitfalls, blockers, contradictions with PRD/UX)
- <item>

## Missing info reported by the developer
- <what the developer needed and did not find, now answered here, or "none">

## Debt logged
- <n> items added to docs/TECH_DEBT.md (<top one>)

## How to verify
- Quality gate: the commands in `.workflow/gate.json`
- Run the app / the screen touched: <command>
