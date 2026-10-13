# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.1.0] - 2026-10-13

### Added
- `/factory`: brainstorm with a built-in grill, full spec pipeline with independent review, automatic or gated approval, and a block-by-block build with verify, review, living docs and handoffs.
- Block runners A (nested subagents), B (child headless session per block) and C (one block per session).
- Continuity: task packets, JSON reports, `HANDOFF.md`, phase handoffs, `docs/DECISIONS.md`, `.workflow/journal.jsonl`.
- Model tiers (`deep`, `standard`, `fast`), with presets and custom model names.
- Stops with a diagnosis and a multiple-choice question; `/factory answer`; quota pause.
- `/change`, including `--pre-approved`.
- Guardrails: frozen spec (native `deny` rules plus a portable gate check), secret and destructive-command rules.
- Eval suite (`claude plugin eval`) and CI.
