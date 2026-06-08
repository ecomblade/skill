# Ecomblade Skill

Codex skill for authenticating with Ecomblade connector APIs and running connector feature searches through the published CLI.

## Install

```bash
npx skills add ecomblade/skill
```

## What it does

- prefers the published `ecomblade` CLI
- supports OAuth login with PKCE and a localhost callback
- checks the current connector session with `whoami`
- supports consolidated connector feature commands with `--platform`
- supports local logout and remote revoke through the CLI
- keeps the human approval step in the Ecomblade frontend flow
- gives direct fetch guidance for already-authenticated feature calls

## Typical workflow

```bash
npx ecomblade whoami --json
npx ecomblade login
npx ecomblade whoami --json
npx ecomblade search-product --platform amazon --query "running shoes" --page 1 --json
npx ecomblade search-product --platform lazada --keyword "perfume" --page 1 --region lazada.sg --sort popularity --json
npx ecomblade logout --revoke --json
```

The canonical machine-readable version for the skill lives in [`skill.json`](./skill.json).

The [`agents/openai.yaml`](./agents/openai.yaml) file is OpenAI/Codex-specific UI metadata only.

The full skill definition lives in [SKILL.md](./SKILL.md).
