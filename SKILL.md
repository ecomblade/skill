---
name: ecomblade
description: Authenticate with Ecomblade connector APIs and run Amazon or Temu connector searches using the published Ecomblade CLI. Use when a user wants to log in, inspect the current connector session, search via public connector feature routes, or log out and revoke a saved connector session. Authentication uses OAuth with PKCE through the CLI; direct HTTP fallback is only for already-authenticated feature calls.
---

# Ecomblade Connectors

Use this skill when the task is to authenticate or manage a saved connector session for Ecomblade through the supported CLI surface.

## What this skill covers

- OAuth CLI login with PKCE and a temporary localhost callback
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

Do not fall back to direct REST authentication. The supported login flow needs a browser approval step and localhost OAuth callback handled by the CLI.

## Product assumptions

- The CLI is first-party and always targets `https://api.ecomblade.com`
- Authentication uses OAuth authorization code with PKCE
- The human approval flow happens in the Ecomblade frontend
- Local credentials are stored in `~/.ecomblade/config.json`

## Claude setup note

If this skill is being installed or used via Claude and direct API access is blocked, add the Ecomblade API domain to Claude's allowlist:

- Go to `Settings > Capabilities > Additional allowed domains`
- Add: `api.ecomblade.com`
- Enter the domain only, without `https`

## Recommended workflow

1. Check whether the user is already authenticated:
   - `npx ecomblade whoami --json`
2. If not authenticated:
   - `npx ecomblade login`
   - approve the OAuth request in the browser
   - wait for the CLI to receive the localhost callback and save the session
3. Confirm the connector session:
   - `npx ecomblade whoami --json`
4. Run connector feature queries when needed:
   - `npx ecomblade amazon search-product --query "running shoes" --page 1 --json`
   - `npx ecomblade amazon search-category --category-id 172282 --page 1 --json`
   - `npx ecomblade temu search-product --keyword "desk lamp" --page 1 --sort popularity --json`
   - `npx ecomblade temu category-product --category-path "home-kitchen/lighting" --page 1 --sort sales --json`
5. Revoke and clear the saved session when needed:
   - `npx ecomblade logout --revoke --json`

## Direct fetch feature routes

When CLI execution is not available but an access token already exists, use these authenticated GET endpoints:

- `/public/connectors/features/amazon/search-product?query=<query>&page=<page>&sort=<sort?>`
- `/public/connectors/features/amazon/search-category?categoryId=<categoryId>&page=<page>`
- `/public/connectors/features/temu/search-product?keyword=<keyword>&page=<page>&sort=<sort>`
- `/public/connectors/features/temu/category-product?categoryPath=<categoryPath>&page=<page>&sort=<sort>`

For authenticated calls, send:

- `Authorization: Bearer <access_token>`

If a feature call returns `expired_token` or `invalid_token`, use the CLI to refresh/re-authenticate before retrying.

## Output handling

- Prefer `--json` whenever the result needs to be parsed or used by another tool step
- Expect the CLI to preserve connector auth errors such as:
  - `access_denied`
  - `invalid_token`
  - `expired_token`
  - `unauthorized`
- For direct feature fetches, expect the backend envelope:
  - `success: true` with `data` on success
  - `success: false` with `error.code` and `error.message` on failure

## Non-goals

- Do not use this skill for browser automation beyond the user approval step
- Do not use removed device-auth endpoints; connector authentication is OAuth-only
