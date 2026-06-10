---
name: ecomblade
description: Authenticate with Ecomblade connector APIs and run marketplace connector searches using the published Ecomblade CLI or pure fetch OAuth fallback. Use when a user wants to log in, inspect the current connector session, search via public connector feature routes, or log out and revoke a saved connector session. Prefer the CLI when available; use pure fetch in Claude Web or environments where npx/local callbacks are unavailable.
---

# Ecomblade Connectors

Use this skill when the task is to authenticate or manage a saved connector session for Ecomblade through the supported CLI surface or pure HTTP fetch fallback.

## What this skill covers

- OAuth CLI login with PKCE and a temporary localhost callback
- Claude Web compatible OAuth login with `login --web`
- Pure fetch OAuth with PKCE for environments that cannot run `npx`
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

If `npx` or the CLI is unavailable, use the pure fetch OAuth workflow below. Do not invent a password login, device-code flow, or token shortcut. The supported fallback is still OAuth authorization code with PKCE and a human approval step in the Ecomblade frontend.

## Product assumptions

- The CLI is first-party and targets `https://api.ecomblade.com` by default
- Authentication uses OAuth authorization code with PKCE
- The human approval flow happens in the Ecomblade frontend
- Local credentials are stored in `~/.ecomblade/config.json`
- Pure fetch auth uses `client_id=ecomblade-cli`
- Pure fetch auth uses `redirect_uri=https://api.ecomblade.com/public/connectors/auth/oauth/web-complete`

## Claude setup note

If this skill is being installed or used via Claude and direct API access is blocked, add the Ecomblade API domain to Claude's allowlist:

- Go to `Settings > Capabilities > Additional allowed domains`
- Add: `api.ecomblade.com`
- Enter the domain only, without `https`
- Then authenticate with `npx ecomblade login --web`, or use the pure fetch workflow if `npx` is unavailable

## Recommended workflow

1. Check whether the user is already authenticated:
   - `npx ecomblade whoami --json`
2. If not authenticated:
   - `npx ecomblade login`
   - approve the OAuth request in the browser
   - wait for the CLI to receive the localhost callback and save the session
   - in Claude Web, use `npx ecomblade login --web` instead
   - if `npx` is unavailable, use the pure fetch OAuth workflow below
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

If a feature call returns `expired_token` or `invalid_token`, refresh the access token with `/public/connectors/auth/refresh` when using pure fetch, or use the CLI to refresh/re-authenticate when using CLI mode.

## Pure fetch OAuth workflow

Use this workflow in Claude Web or any environment that can make HTTPS requests to `api.ecomblade.com` but cannot run `npx ecomblade` reliably.

Constants:

- API base URL: `https://api.ecomblade.com`
- Client ID: `ecomblade-cli`
- Redirect URI: `https://api.ecomblade.com/public/connectors/auth/oauth/web-complete`
- Authorization endpoint: `/public/connectors/auth/oauth/authorize`
- Completion endpoint: `/public/connectors/auth/oauth/completion`
- Token endpoint: `/public/connectors/auth/oauth/token`
- Refresh endpoint: `/public/connectors/auth/refresh`
- Session endpoint: `/public/connectors/auth/me`
- Revoke endpoint: `/public/connectors/auth/revoke`

Pure fetch login steps:

1. Generate:
   - `state`: random base64url string
   - `code_verifier`: random base64url string with enough entropy
   - `code_challenge`: base64url SHA-256 digest of `code_verifier`
2. Build the authorization URL:
   - `client_id=ecomblade-cli`
   - `redirect_uri=https://api.ecomblade.com/public/connectors/auth/oauth/web-complete`
   - `response_type=code`
   - `state=<state>`
   - `code_challenge=<code_challenge>`
   - `code_challenge_method=S256`
3. `GET` the authorization URL with `redirect: "manual"`.
4. Read the `Location` response header. This is the Ecomblade approval URL.
5. Extract `oauth_request_id` from the approval URL.
6. Ask the user to open the approval URL and approve access.
7. Poll:
   - `GET /public/connectors/auth/oauth/completion?request_id=<oauth_request_id>&state=<state>`
   - If `data.status` is `pending`, wait `data.retry_after` seconds or 2 seconds, then poll again.
   - If `data.redirect_to` is present, parse it as a URL.
   - If `redirect_to` contains `error=access_denied`, stop and report denial.
   - If `redirect_to` contains `code`, continue.
8. Verify the returned `state` matches the original state.
9. Exchange the authorization code:
   - `POST /public/connectors/auth/oauth/token`
   - JSON body:
     - `grant_type: "authorization_code"`
     - `code: <code>`
     - `redirect_uri: "https://api.ecomblade.com/public/connectors/auth/oauth/web-complete"`
     - `client_id: "ecomblade-cli"`
     - `code_verifier: <code_verifier>`
10. Store the returned:
    - `access_token`
    - `refresh_token`
    - `expires_in`
    - `session_id`
    - `token_type`
    - `user_display_name`

Pure fetch JavaScript example:

```js
const API_BASE_URL = 'https://api.ecomblade.com'
const CLIENT_ID = 'ecomblade-cli'
const REDIRECT_URI =
  'https://api.ecomblade.com/public/connectors/auth/oauth/web-complete'

const base64url = (bytes) => {
  const binary = Array.from(bytes, (byte) => String.fromCharCode(byte)).join('')
  const base64 =
    typeof btoa === 'function'
      ? btoa(binary)
      : Buffer.from(bytes).toString('base64')

  return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '')
}

const randomToken = (bytes = 32) => base64url(crypto.getRandomValues(new Uint8Array(bytes)))

const sha256Base64url = async (value) => {
  const bytes = new TextEncoder().encode(value)
  const digest = await crypto.subtle.digest('SHA-256', bytes)
  return base64url(new Uint8Array(digest))
}

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms))

async function loginWithFetch() {
  const state = randomToken()
  const codeVerifier = randomToken(48)
  const codeChallenge = await sha256Base64url(codeVerifier)

  const authorizeUrl = new URL('/public/connectors/auth/oauth/authorize', API_BASE_URL)
  authorizeUrl.searchParams.set('client_id', CLIENT_ID)
  authorizeUrl.searchParams.set('redirect_uri', REDIRECT_URI)
  authorizeUrl.searchParams.set('response_type', 'code')
  authorizeUrl.searchParams.set('state', state)
  authorizeUrl.searchParams.set('code_challenge', codeChallenge)
  authorizeUrl.searchParams.set('code_challenge_method', 'S256')

  const authorizeResponse = await fetch(authorizeUrl, { redirect: 'manual' })
  const approvalUrl = authorizeResponse.headers.get('location')
  if (!approvalUrl) throw new Error('Missing Ecomblade approval URL')

  const requestId = new URL(approvalUrl).searchParams.get('oauth_request_id')
  if (!requestId) throw new Error('Missing OAuth request ID')

  console.log(`Open this URL and approve access: ${approvalUrl}`)

  let redirectTo
  for (let attempt = 0; attempt < 300; attempt += 1) {
    const completionUrl = new URL('/public/connectors/auth/oauth/completion', API_BASE_URL)
    completionUrl.searchParams.set('request_id', requestId)
    completionUrl.searchParams.set('state', state)

    const completionResponse = await fetch(completionUrl)
    const completion = await completionResponse.json()
    if (!completion.success) {
      throw new Error(completion.error?.message || 'OAuth completion failed')
    }

    if (completion.data.redirect_to) {
      redirectTo = completion.data.redirect_to
      break
    }

    await sleep((completion.data.retry_after || 2) * 1000)
  }

  if (!redirectTo) throw new Error('Timed out waiting for approval')

  const callbackUrl = new URL(redirectTo)
  if (callbackUrl.searchParams.get('state') !== state) {
    throw new Error('OAuth state mismatch')
  }

  const denied = callbackUrl.searchParams.get('error')
  if (denied) throw new Error(`OAuth authorization failed: ${denied}`)

  const code = callbackUrl.searchParams.get('code')
  if (!code) throw new Error('Missing authorization code')

  const tokenResponse = await fetch(
    new URL('/public/connectors/auth/oauth/token', API_BASE_URL),
    {
      method: 'POST',
      headers: {
        accept: 'application/json',
        'content-type': 'application/json',
      },
      body: JSON.stringify({
        grant_type: 'authorization_code',
        code,
        redirect_uri: REDIRECT_URI,
        client_id: CLIENT_ID,
        code_verifier: codeVerifier,
      }),
    }
  )

  const token = await tokenResponse.json()
  if (!tokenResponse.ok || !token.access_token) {
    throw new Error(token.error?.message || 'OAuth token exchange failed')
  }

  return token
}
```

Pure fetch session calls:

- Inspect session:
  - `GET /public/connectors/auth/me`
  - Header: `Authorization: Bearer <access_token>`
- Refresh access token:
  - `POST /public/connectors/auth/refresh`
  - JSON body: `{ "refresh_token": "<refresh_token>" }`
  - Preserve the original refresh token if the response does not return a new one.
- Revoke session:
  - `POST /public/connectors/auth/revoke`
  - Header: `Authorization: Bearer <access_token>`
  - JSON body: `{ "session_id": "<session_id>" }`

Pure fetch feature calls use the same endpoints and query parameters from Direct fetch feature routes. Always send `Authorization: Bearer <access_token>`.

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
