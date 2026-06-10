---
name: ecomblade
description: Authenticate with Ecomblade connector APIs and run marketplace connector searches using the published Ecomblade CLI. Use when a user wants to log in, inspect the current connector session, search via public connector feature routes, or log out and revoke a saved connector session. Authentication uses OAuth with PKCE through the CLI; Claude Web uses login --web because localhost callbacks are unavailable.
---

# Ecomblade Connectors

Use this skill when the task is to authenticate or manage a saved connector session for Ecomblade through the supported CLI surface.

## What this skill covers

- OAuth CLI login with PKCE and a temporary localhost callback
- Claude Web compatible OAuth login with `login --web`
- Session inspection with `whoami`
- Local logout
- Remote revoke through `logout --revoke`
- Product search through the consolidated `search-product` route
- Category product search through the consolidated `category-product` route
- Marketplace selection with `--platform`
- JSON output for agentic workflows

## Preferred tool

Prefer the published CLI:

`npx ecomblade`

If the CLI repo is checked out locally for development, `node ./bin/ecomblade.js` from that repo is also acceptable.

Do not fall back to direct REST authentication. The supported login flow needs a browser approval step handled by the CLI. Use normal `login` for local terminals and `login --web` for Claude Web or sandboxes that cannot receive localhost callbacks.

## Product assumptions

- The CLI is first-party and targets `https://api.ecomblade.com` by default
- Authentication uses OAuth authorization code with PKCE
- The human approval flow happens in the Ecomblade frontend
- Local credentials are stored in `~/.ecomblade/config.json`

## Claude setup note

If this skill is being installed or used via Claude and direct API access is blocked, add the Ecomblade API domain to Claude's allowlist:

- Go to `Settings > Capabilities > Additional allowed domains`
- Add: `api.ecomblade.com`
- Enter the domain only, without `https`
- Then authenticate with: `npx ecomblade login --web`

## Recommended workflow

1. Check whether the user is already authenticated:
   - `npx ecomblade whoami --json`
2. If not authenticated:
   - `npx ecomblade login`
   - approve the OAuth request in the browser
   - wait for the CLI to receive the localhost callback and save the session
   - in Claude Web, use `npx ecomblade login --web` instead
3. Confirm the connector session:
   - `npx ecomblade whoami --json`
4. Run connector feature queries when needed:
   - `npx ecomblade search-product --platform amazon --query "running shoes" --page 1 --json`
   - `npx ecomblade search-product --platform lazada --keyword "perfume" --page 1 --region lazada.sg --sort popularity --json`
   - `npx ecomblade category-product --platform amazon --category-id 172282 --page 1 --json`
   - `npx ecomblade category-product --platform temu --category-path "home-kitchen/lighting" --page 1 --sort sales --json`
5. Revoke and clear the saved session when needed:
   - `npx ecomblade logout --revoke --json`

## CLI feature commands

Use only these two feature commands:

- `npx ecomblade search-product --platform <platform> [platform params] --json`
- `npx ecomblade category-product --platform <platform> [platform params] --json`

Search product platform params:

- `alibaba`: `--keyword <text> [--page <n>] [--page-size <n>]`
- `amazon`: `--query <text> --page <n> [--sort <sort>]`
  - `sort`: `relevanceblender`, `price-asc-rank`, `price-desc-rank`, `review-rank`, `date-desc-rank`, `exact-aware-popularity-rank`
- `lazada`: `--keyword <text> --page <n> --region <region> [--sort <sort>]`
  - `region`: `lazada.co.id`, `lazada.com.my`, `lazada.sg`, `lazada.com.ph`, `lazada.co.th`, `lazada.vn`
  - `sort`: `popularity`, `priceasc`, `pricedesc`, `ratingdesc`, `newest`
- `temu`: `--keyword <text> --page <n> --sort <sort>`
  - `sort`: `popularity`, `sales`, `most_recent`, `price_low_to_high`, `price_high_to_low`
- `tiktok`: `--keyword <text> --page <n> --sort <sort> --region <region>`
  - `sort`: `popularity`, `best_seller`, `newest`, `cheapest`
  - `region`: `sg`, `th`, `vn`, `id`, `my`, `ph`

Category product platform params:

- `alibaba`: `--category-ids <ids> --page <n> [--tab <tab>] [--delivery-id <id>] [--page-deduplicate-id <id>]`
  - `tab`: `launches`, `trends`
- `amazon`: `--category-id <id> --page <n>`
- `lazada`: `--category-path <path> --page <n> --region <region> [--sort <sort>]`
  - `region`: `lazada.co.id`, `lazada.com.my`, `lazada.sg`, `lazada.com.ph`, `lazada.co.th`, `lazada.vn`
  - `sort`: `popularity`, `priceasc`, `pricedesc`, `ratingdesc`, `newest`
- `temu`: `--category-path <path> --page <n> --sort <sort>`
  - `sort`: `popularity`, `sales`, `most_recent`, `price_low_to_high`, `price_high_to_low`
- `tiktok`: `--category <value> --page <n> --region <region>`
  - `region`: `sg`, `th`, `vn`, `id`, `my`, `ph`

## Direct fetch feature routes

When CLI execution is not available but an access token already exists, use these authenticated GET endpoints:

- `/public/connectors/features/search-product?platform=<platform>&...`
- `/public/connectors/features/category-product?platform=<platform>&...`

Use the same platform-specific query params listed above. For example:

- `/public/connectors/features/search-product?platform=lazada&keyword=perfume&page=1&sort=popularity&region=lazada.sg`
- `/public/connectors/features/category-product?platform=amazon&categoryId=172282&page=1`

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
