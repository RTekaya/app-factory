# Continuity: no information lost between sessions

A session or a role exists only through what it receives (its **task packet**) and what it leaves on disk (its **report** and **handoff**). Nothing that matters may live only in a conversation.

## The four memory files

| File | Content | Written by | Read by | Rule |
|---|---|---|---|---|
| `HANDOFF.md` (+ copy in `docs/handoffs/<ID>.md`) | Last block: next block, contracts, decisions, pitfalls, how to verify | doc-keeper | the next developer, first | ≤ 120 lines; overwritten each block; template `templates/HANDOFF.md` |
| `docs/handoffs/phase-<name>.md` | End of a phase (`brainstorm`, `spec`, `approval-v<n>`, `M<k>`): decisions, assumptions, rejected options and why, open questions | orchestrator | the next phase | Never overwritten; template `templates/phase-handoff.md` |
| `docs/DECISIONS.md` | Every answer from the user | orchestrator, **right after the answer** | spec roles, reviewer, diagnostician | Append-only; template `templates/DECISIONS.md` |
| `.workflow/journal.jsonl` | One line per step, role run, stop and answer | orchestrator | resume after a crash, diagnostician, cost summary | Append-only (`state.md` § Journal) |

## Task packet

Every role launch, whether a subagent, a child session, or yourself playing a role on a host without subagents, gets a packet in the format of `templates/task-packet.md`:

- **Role and mode**: e.g. `product-manager · full run` or `developer · fix round 2`.
- **Objective**: one or two sentences.
- **Read**: file paths, with the sections when relevant (`docs/04-architecture.md §3, §5, §10`). Never paste file contents.
- **Write**: the exact output files.
- **Decisions to respect**: the entries of `docs/DECISIONS.md` that concern this task, quoted in one line each, or "none".
- **Known pitfalls**: the "Attention" items of `HANDOFF.md` and anything from the journal that applies, or "none".
- **Out of scope**: what the role must not touch.
- **Report**: "End with the JSON report defined in your role file."
- **SKILL_DIR**: the absolute path, so the role can open `templates/` and `references/` files.

## Report

Each role file ends with a `## Report` section that defines a JSON object. The role's final message ends with that object in a ```json fenced block. Then:

1. Parse the JSON. If it is missing or a required field is absent, send the role **one** follow-up: "Your report is missing or malformed. Reply with only the JSON report." If it still fails, treat the step as failed.
2. Check that the files the report lists in `files_written` exist.
3. Only then mark the step complete in the state and the journal.

## Write before you act

- Before launching a step, write the state transition that announces it (`state.md`).
- After it ends, write the journal line, then the next transition.
- At startup, if the state says a step is in progress but no session holds it, the previous run died. Write a journal line with `outcome: "interrupted"`, then follow the resume rule of that phase (each reference has a Resume section).

## Handoff quality loop

The developer's report has a `missing_info` field ("what I needed at the start and did not find"). When it is not empty:

1. Pass it to the doc-keeper, which adds the missing facts to `HANDOFF.md`.
2. At the end of the milestone, list all `missing_info` items in `docs/handoffs/phase-M<k>.md` under "Handoff gaps". Future handoffs then cover them from the start.

## Phase handoff checklist

When a phase ends, write `docs/handoffs/phase-<name>.md` before changing the phase in the state. It must answer:

1. What was decided, and by whom (user or role)?
2. What was assumed without confirmation?
3. What was considered and rejected, and why?
4. What is still open, and who owns it?
5. What must the next phase read first?
