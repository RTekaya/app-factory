<!-- app-factory:start -->
## App Factory

This project is built with the App Factory workflow (`/factory`).

- **State**: `.workflow/state.json`. Read it before acting; only the `/factory` orchestrator writes it.
- **Source of truth**, when documents disagree: PRD > UX > architecture > design system > plan > code. Report contradictions; never resolve them silently.
- **Frozen during the build**: `docs/00-brief.md`, `docs/01-prd.md`, `docs/02-ux.md`. They change only through `/change`.
- **Building or fixing code**: read `HANDOFF.md` first, then only the doc sections you need.
- **Quality gate**: `.workflow/gate.json`, run by the verifier. Never weaken tests, lint rules or the gate to get green.
- **Decisions**: `docs/DECISIONS.md` is append-only. Never contradict a recorded decision.
- Code, identifiers and commit messages in English. Docs in the language of `docs/00-brief.md`.
- Secrets never go into the repository, the docs or prompts.

### Stack & commands
<!-- Filled at spec approval from docs/04-architecture.md §2 and §8. -->
<!-- app-factory:end -->
