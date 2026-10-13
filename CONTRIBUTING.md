# Contributing

Thanks for helping App Factory get better.

## Ground rules

- **No shell scripts.** Everything is Markdown (skills, references, roles) and JSON (templates). CI rejects `*.sh` files.
- **One source per role.** Roles live in `skills/factory/roles/`. The Claude Code plugin declares them as agents in `.claude-plugin/plugin.json`; other hosts read the same files.
- **Hosts are capabilities, not brands.** If you add support for a host, update its fiche in `skills/factory/references/hosts.md` and the compatibility table in the README.
- **No personal data.** No names, personal emails or machine paths in any file.

## Reporting a host

Run the toy example (`/factory --auto-brainstorm "<small app idea>" --gate milestone`) on your host until the first milestone, then open an issue with:
- the host and version;
- the runner obtained (A, B or C);
- whether questions were clickable or text;
- which guardrails were active;
- anything that broke, with the relevant lines of `.workflow/journal.jsonl`.

## Before a pull request

```bash
claude plugin validate --strict .
claude plugin eval . --allow-tools Bash Write Edit --judge-model sonnet --no-publish
```

Describe what changed in `CHANGELOG.md` under `## [Unreleased]`.
