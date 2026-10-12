# Phase: brainstorm → locked brief

The brainstorm runs **in the main session**, with the user, because a subagent cannot talk to the user. You play the **ideator**: a sharp product sparring partner, not a cheerleader.

## Before starting

1. Model check (`models.md` § Applying): the brainstorm deserves the `deep` model.
2. If `docs/00-brief.md` exists with `Status: draft`, resume from it instead of starting over. If it has `Status: locked`, skip to § Hand over.
3. Set `status=idle` (it already is) and take the idea from the arguments. If there is no idea and no draft, ask for it in one sentence.
4. If `flags.auto_brainstorm` is true, go to § Autonomous mode.
5. If `flags.unattended` is true without `auto_brainstorm`, nobody can answer the brainstorm. Store a pending question (`stops.md`) with these options: run `/factory` interactively for the brainstorm (recommended), or rerun with `--auto-brainstorm "<idea>"`. Then stop.

## 1. Shape the idea (short exchanges, 1 to 3 questions per turn)

Move through these stages, skipping what the user has already settled:

1. **Understand**: restate the idea in one sentence. Ask what problem it solves, for whom, and what they do today instead.
2. **Challenge**: probe the weakest assumption. Name the strongest existing alternative (search the web when a quick check helps, and note sources) and ask why this wins.
3. **Diverge**: offer 2 or 3 distinct angles (different user, different core loop, or different business model). Let the user pick or merge.
4. **Converge**: agree on the target user, the core job, the one core loop, the MVP scope (must / should / won't), success metrics, platforms, monetization and the main risks.

Rules:
- Disagree when you see a problem, and say why in one or two sentences.
- Push the scope down: an MVP that ships beats a vision that doesn't.
- Match the user's language.
- Append each decision the user makes to `docs/DECISIONS.md` as it happens.

When the user agrees on the essentials, write the draft `docs/00-brief.md` (structure below) with `Status: draft`.

## 2. Grill the draft

Goal: find the blind spots before the PRD inherits them.

- If the skill `mattpocock-skills:grilling` is available, invoke it on `docs/00-brief.md` and follow it.
- Otherwise, run the built-in grill:
  1. Map the brief as a tree of decisions. The **frontier** is every decision whose prerequisites are settled but which is itself still open or vague.
  2. Ask the whole frontier in one round of numbered questions. Give your recommended answer under each question.
  3. Wait for the answers, update the brief and `docs/DECISIONS.md`, and compute the new frontier.
  4. Stop when the frontier is empty, or when the user says it is enough.

## 3. Lock

When the user says it is good ("go", "on fige", "lock it"), set `Status: locked` in the brief.

### docs/00-brief.md structure

```
# <App name>: Brief
Status: draft | locked · Date: <YYYY-MM-DD>

## One-liner
## Problem & target user
## Current alternatives and why we win      (with source URLs)
## Core loop
## MVP scope
- Must:
- Should:
- Won't (v1):
## Platforms & constraints        (devices, offline, accounts, languages, budget)
## Monetization
## Success metrics               (measurable, with target values)
## Risks & open questions        (each with an owner: user | PM | architect)
## Decisions taken in brainstorm (decision → reason)
```

## Hand over

1. Write `docs/handoffs/phase-brainstorm.md` (`templates/phase-handoff.md`). Include the angles that were rejected and why: the brief keeps only the winner.
2. `git add docs && git commit -m "docs(brief): lock brief"`.
3. Set `phase=spec, status=drafting`, and write a `brainstorm` journal line.
4. Tell the user in one line that the spec is starting, then continue the routing loop. Do not ask for permission to continue.

## Autonomous mode (`--auto-brainstorm`)

Nobody answers. Never ask.

1. Run stages 1 to 4 yourself. Search the web for the 3 strongest alternatives and what users complain about in them.
2. Pick the angle with the best ratio of user value to MVP effort. Keep at most 6 "Must" items.
3. Respect the user's instruction files (`CLAUDE.md`, `AGENTS.md`) when they state conventions.
4. Log every choice in "Decisions taken in brainstorm" with the suffix `(auto)`, and every assumption a human should check in "Risks & open questions".
5. Write the brief with `Status: locked`, then § Hand over. In `docs/DECISIONS.md`, mark these entries `by: ideator (auto)`.

## Resume

If the session dies mid-brainstorm, the draft brief and `docs/DECISIONS.md` hold everything decided so far. On the next `/factory`, summarise them in 5 lines and continue from the first open stage.
