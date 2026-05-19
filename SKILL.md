---
name: ecomblade
description: Authenticate with Ecomblade connector APIs and run Amazon or Temu connector searches using the published Ecomblade CLI. Use when a user wants to log in, inspect the current connector session, search via public connector feature routes, or log out and revoke a saved connector session. If the environment cannot run Node or `npx`, use authenticated fetch requests against the public connector auth and feature endpoints instead. Do not use browser automation beyond the human approval step.
---

# Ecomblade Connectors

Use this skill when the task is to authenticate or manage a saved connector session for Ecomblade through the supported CLI surface.

## What this skill covers

- Device-style CLI login
- Session inspection with `whoami`
- Local logout
- Remote revoke through `logout --revoke`
- Amazon product and category search through connector auth
- Temu product and category-product search through connector auth
- JSON output for agentic workflows

## Preferred tool

Prefer the published CLI:

`npx ecomblade`

If the CLI repo is checked out locally for development, `node ./bin/ecomblade.js` from that repo is also acceptable.

Do not fall back to direct REST calls unless the CLI is clearly blocked or the user explicitly asks for lower-level debugging.

## Fallback for LLMs without Node

If the environment can make HTTP requests but cannot run `npx ecomblade`, use direct fetches against `https://api.ecomblade.com`.

Auth flow:

1. Start device auth:
   - `POST /public/connectors/auth/device`
   - body:
     ```json
     {
       "client_name": "Ecomblade CLI",
       "machine_name": "chatgpt"
     }
     ```
2. Show the returned `verification_url` and `user_code` to the user for approval.
3. Poll:
   - `POST /public/connectors/auth/token`
   - body:
     ```json
     {
       "device_code": "<device_code>"
     }
     ```
4. Save `access_token`, `refresh_token`, `expires_in`, and `session_id`.
5. When the access token is near expiry, refresh:
   - `POST /public/connectors/auth/refresh`
   - body:
     ```json
     {
       "refresh_token": "<refresh_token>"
     }
     ```
6. For authenticated calls, send:
   - `Authorization: Bearer <access_token>`

If a feature call returns `expired_token` or `invalid_token`, refresh once and retry once.

## Product assumptions

- The CLI is first-party and always targets `https://api.ecomblade.com`
- The human approval flow happens in the Ecomblade frontend
- Local credentials are stored in `~/.ecomblade/config.json`

## Recommended workflow

1. Check whether the user is already authenticated:
   - `npx ecomblade whoami --json`
2. If not authenticated:
   - `npx ecomblade login`
3. For machine-driven approval handoff:
   - `npx ecomblade login --manual --json`
   - if a completion code is later available: `npx ecomblade login --device-code <code> --completion-code <code> --json`
4. Confirm the connector session:
   - `npx ecomblade whoami --json`
5. Run connector feature queries when needed:
   - `npx ecomblade amazon search-product --query "running shoes" --page 1 --json`
   - `npx ecomblade amazon search-category --category-id 172282 --page 1 --json`
   - `npx ecomblade temu search-product --keyword "desk lamp" --page 1 --sort popularity --json`
   - `npx ecomblade temu category-product --category-path "home-kitchen/lighting" --page 1 --sort sales --json`
6. Revoke and clear the saved session when needed:
   - `npx ecomblade logout --revoke --json`

## Direct fetch feature routes

When CLI execution is not available, use these authenticated GET endpoints:

- `/public/connectors/features/amazon/search-product?query=<query>&page=<page>&sort=<sort?>`
- `/public/connectors/features/amazon/search-category?categoryId=<categoryId>&page=<page>`
- `/public/connectors/features/temu/search-product?keyword=<keyword>&page=<page>&sort=<sort>`
- `/public/connectors/features/temu/category-product?categoryPath=<categoryPath>&page=<page>&sort=<sort>`

## Output handling

- Prefer `--json` whenever the result needs to be parsed or used by another tool step
- Expect the CLI to preserve connector auth errors such as:
  - `authorization_pending`
  - `slow_down`
  - `access_denied`
  - `invalid_token`
  - `expired_token`
- For direct fetches, expect the backend envelope:
  - `success: true` with `data` on success
  - `success: false` with `error.code` and `error.message` on failure

## Non-goals

- Do not use this skill for browser automation beyond the user approval step
