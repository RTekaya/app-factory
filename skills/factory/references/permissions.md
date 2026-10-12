# Agent permissions (Claude Code)

The architect proposes, in `docs/agent-permissions.json` (`{"allow": [...]}`), the commands the build needs: quality gate, build, code generation, dependency install, local emulators. At spec approval you add the safe ones to `.claude/settings.json` → `permissions.allow`. You never add anything else, and you never remove a `deny` rule here.

## Filter: accept a rule only if every condition holds

1. It has the form `Bash(<command> …)`, with a concrete command at the start (e.g. `Bash(flutter test *)`, `Bash(npm run lint)`).
2. It is not a blanket rule: reject `Bash`, `Bash(*)`, `Bash(* *)`, and any rule whose command part starts with `*`.
3. Its command is not one of these, alone or as the first word:
   `sudo`, `su`, `rm`, `rmdir`, `mv` on paths outside the project, `curl`, `wget`, `ssh`, `scp`, `rsync`, `nc`, `eval`, `exec`, `sh`, `bash`, `zsh`, `fish`, `dd`, `chmod`, `chown`, `mkfs`, `kill`, `pkill`, `launchctl`, `systemctl`, `crontab`.
4. It is not a destructive or publishing git or cloud action:
   `git push`, `git reset`, `git clean`, `git checkout --`, `git filter-branch`, `gh repo delete`, `gh release`, any `deploy`, `publish` or `release` subcommand, `firebase deploy`, `gcloud`, `aws`, `az`, `kubectl`, `terraform apply`, `eas submit`, `fastlane`, `npm publish`, `pod trunk`.
5. It does not grant reading secrets (`Read(.env…)`, keys, keystores).

Show the user both lists, "to add" and "rejected", each rule with a one-line reason for rejection. In automatic mode, add the accepted rules and record both lists in `docs/handoffs/phase-approval-v<n>.md`.

## Base rules installed at bootstrap

`templates/settings.json` contains the base `allow` rules (read-only git, commit, tag, branch, safe push) and the base `deny` rules (secrets, force push, hard reset, clean, `rm -rf`, deploy and publish commands). `deny` always wins over `allow` in Claude Code.
