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
- supports Amazon and Temu connector feature routes through CLI commands
- supports local logout and remote revoke through the CLI
- keeps the human approval step in the Ecomblade frontend flow
- gives direct fetch guidance for already-authenticated feature calls

## Typical workflow

```bash
npx ecomblade whoami --json
npx ecomblade login
npx ecomblade whoami --json
npx ecomblade amazon search-product --query "running shoes" --page 1 --json
npx ecomblade logout --revoke --json
```

The canonical machine-readable version for the skill lives in [`skill.json`](./skill.json).

The [`agents/openai.yaml`](./agents/openai.yaml) file is OpenAI/Codex-specific UI metadata only.

The full skill definition lives in [SKILL.md](./SKILL.md).
