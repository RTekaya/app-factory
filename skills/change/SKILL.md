---
name: change
description: App Factory — change the scope or design of an app during its build (impact analysis, PRD/UX/architecture/plan revisions by their owners, review, then your approval). Use only when the user types /change.
argument-hint: "<what should change and why>"
disable-model-invocation: true
---

# /change

Change request: `$ARGUMENTS` (on hosts that do not substitute this, use the text the user typed after the command).

This command belongs to the App Factory workflow. Its procedure lives in the `factory` skill, next to this one:

1. Open `../factory/SKILL.md` (relative to this file's folder; in Claude Code: `${CLAUDE_SKILL_DIR}/../factory/SKILL.md`) and apply its Golden rules. Its SKILL_DIR is that `factory` folder.
2. Follow `../factory/references/change.md` with the change request above.
3. If `.workflow/state.json` does not exist, this project does not use App Factory yet: say so and suggest `/factory`.
