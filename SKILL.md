---
name: ecomblade
description: Authenticate with Ecomblade connector APIs and run marketplace connector searches using the published Ecomblade CLI or server-side pure fetch OAuth fallback. Use when a user wants to log in, inspect the current connector session, search via public connector feature routes, or log out and revoke a saved connector session. Prefer the CLI when available; in Claude Web use bash_tool with curl or node for HTTP because browser widgets cannot reach api.ecomblade.com.
---

# Ecomblade Connectors

Use this skill to authenticate and run marketplace connector searches through the Ecomblade CLI or pure fetch OAuth fallback.

## CLI invocation

Always prefer the published CLI:

```
npx ecomblade
```

If the CLI repo is checked out locally for development, `node ./bin/ecomblade.js` is also acceptable.

If `npx` is unavailable, use the pure fetch OAuth workflow below. Do not invent a password login, device-code flow, or token shortcut — the fallback is still OAuth authorization code with PKCE with a human approval step in the Ecomblade frontend.

**Claude Web note:** The browser widget iframe is CSP-sandboxed and cannot reach `api.ecomblade.com`. Never use a browser-side fetch widget for OAuth in Claude Web. Run all OAuth steps through `bash_tool` using `curl` or `node` instead.

## Product assumptions

- Targets `https://api.ecomblade.com` by default
- Authentication uses OAuth authorization code with PKCE; human approval happens in the Ecomblade frontend
- Local credentials stored in `~/.ecomblade/config.json`
- Pure fetch auth uses `client_id=ecomblade-cli` and `redirect_uri=https://api.ecomblade.com/public/connectors/auth/oauth/web-complete`

## Claude setup note

If API access is blocked, add `api.ecomblade.com` to Claude's allowlist at `Settings > Capabilities > Additional allowed domains` (domain only, no `https`). Then authenticate with `npx ecomblade login --web`, or use the pure fetch workflow if `npx` is unavailable.

Note: this affects server-side network egress only. Browser-side fetch from a widget may still fail due to CSP even with the domain allowed — always use `bash_tool` for HTTP in Claude Web.

## Recommended workflow

1. `npx ecomblade whoami --json`
2. If not authenticated:
   - Standard: `npx ecomblade login` → approve in browser → CLI saves session
   - Claude Web: `npx ecomblade login --web`
   - No `npx`: use the pure fetch OAuth workflow below
3. Confirm: `npx ecomblade whoami --json`
4. Run feature queries (see CLI feature commands below)
5. To revoke: `npx ecomblade logout --revoke --json`

## CLI feature commands

```
npx ecomblade search-product   --platform <platform> [platform params] --json
npx ecomblade category-product --platform <platform> [platform params] --json
```

**search-product params by platform:**

| Platform | Required | Optional |
|----------|----------|---------|
| `alibaba` | `--keyword <text>` | `--page <n>`, `--page-size <n>` |
| `amazon` | `--query <text>`, `--page <n>` | `--sort relevanceblender\|price-asc-rank\|price-desc-rank\|review-rank\|date-desc-rank\|exact-aware-popularity-rank` |
| `lazada` | `--keyword <text>`, `--page <n>`, `--region <region>` | `--sort popularity\|priceasc\|pricedesc\|ratingdesc\|newest` |
| `temu` | `--keyword <text>`, `--page <n>`, `--sort <sort>` | sort: `popularity\|sales\|most_recent\|price_low_to_high\|price_high_to_low` |
| `tiktok` | `--keyword <text>`, `--page <n>`, `--sort <sort>`, `--region <region>` | sort: `popularity\|best_seller\|newest\|cheapest` |

Lazada `--region`: `lazada.co.id`, `lazada.com.my`, `lazada.sg`, `lazada.com.ph`, `lazada.co.th`, `lazada.vn`  
TikTok `--region`: `sg`, `th`, `vn`, `id`, `my`, `ph`

**category-product params by platform:**

| Platform | Required | Optional |
|----------|----------|---------|
| `alibaba` | `--category-ids <ids>`, `--page <n>` | `--tab launches\|trends`, `--delivery-id <id>`, `--page-deduplicate-id <id>` |
| `amazon` | `--category-id <id>`, `--page <n>` | |
| `lazada` | `--category-path <path>`, `--page <n>`, `--region <region>` | `--sort popularity\|priceasc\|pricedesc\|ratingdesc\|newest` |
| `temu` | `--category-path <path>`, `--page <n>`, `--sort <sort>` | sort: `popularity\|sales\|most_recent\|price_low_to_high\|price_high_to_low` |
| `tiktok` | `--category <value>`, `--page <n>`, `--region <region>` | |

## Direct fetch feature routes

When the CLI is unavailable but an access token exists, use these authenticated GET endpoints:

```
GET /public/connectors/features/search-product?platform=<platform>&...
GET /public/connectors/features/category-product?platform=<platform>&...
```

Use the same platform params as the CLI. Send `Authorization: Bearer <access_token>`. If the response returns `expired_token` or `invalid_token`, refresh via `/public/connectors/auth/refresh` (pure fetch) or re-authenticate via CLI.

## Pure fetch OAuth workflow

Use in Claude Web or any environment that can reach `https://api.ecomblade.com` but cannot run `npx`. In Claude Web, execute every step via `bash_tool` (`curl` or `node`) — not browser-side JavaScript.

**Constants:**

| | |
|-|-|
| API base | `https://api.ecomblade.com` |
| Client ID | `ecomblade-cli` |
| Redirect URI | `https://api.ecomblade.com/public/connectors/auth/oauth/web-complete` |
| Authorize | `GET /public/connectors/auth/oauth/authorize` |
| Completion | `GET /public/connectors/auth/oauth/completion` |
| Token | `POST /public/connectors/auth/oauth/token` |
| Refresh | `POST /public/connectors/auth/refresh` |
| Session | `GET /public/connectors/auth/me` |
| Revoke | `POST /public/connectors/auth/revoke` |

**Login steps:**

1. Generate `state` and `code_verifier` (random base64url strings); compute `code_challenge` = base64url(SHA-256(`code_verifier`))
2. `GET /authorize` with `client_id`, `redirect_uri`, `response_type=code`, `state`, `code_challenge`, `code_challenge_method=S256`; follow with `redirect: "manual"`
3. Read the `Location` header — this is the Ecomblade approval URL; extract `oauth_request_id` from it
4. Ask the user to open the approval URL and approve access
5. Poll `GET /completion?request_id=<id>&state=<state>` until `data.redirect_to` appears (wait `data.retry_after` or 2 s between polls); stop on `error=access_denied`
6. Verify returned `state` matches; extract `code` from `redirect_to`
7. `POST /token` with `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, `code_verifier`
8. Store `access_token`, `refresh_token`, `expires_in`, `session_id`, `token_type`, `user_display_name`

**JavaScript example (run via `bash_tool`/Node, not in a browser widget):**

```js
const API_BASE_URL = "https://api.ecomblade.com";
const CLIENT_ID = "ecomblade-cli";
const REDIRECT_URI =
  "https://api.ecomblade.com/public/connectors/auth/oauth/web-complete";

const base64url = (bytes) => {
  const binary = Array.from(bytes, (byte) => String.fromCharCode(byte)).join("");
  const base64 =
    typeof btoa === "function"
      ? btoa(binary)
      : Buffer.from(bytes).toString("base64");
  return base64.replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/g, "");
};

const randomToken = (bytes = 32) =>
  base64url(crypto.getRandomValues(new Uint8Array(bytes)));

const sha256Base64url = async (value) => {
  const bytes = new TextEncoder().encode(value);
  const digest = await crypto.subtle.digest("SHA-256", bytes);
  return base64url(new Uint8Array(digest));
};

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function loginWithFetch() {
  const state = randomToken();
  const codeVerifier = randomToken(48);
  const codeChallenge = await sha256Base64url(codeVerifier);

  const authorizeUrl = new URL("/public/connectors/auth/oauth/authorize", API_BASE_URL);
  authorizeUrl.searchParams.set("client_id", CLIENT_ID);
  authorizeUrl.searchParams.set("redirect_uri", REDIRECT_URI);
  authorizeUrl.searchParams.set("response_type", "code");
  authorizeUrl.searchParams.set("state", state);
  authorizeUrl.searchParams.set("code_challenge", codeChallenge);
  authorizeUrl.searchParams.set("code_challenge_method", "S256");

  const authorizeResponse = await fetch(authorizeUrl, { redirect: "manual" });
  const approvalUrl = authorizeResponse.headers.get("location");
  if (!approvalUrl) throw new Error("Missing Ecomblade approval URL");

  const requestId = new URL(approvalUrl).searchParams.get("oauth_request_id");
  if (!requestId) throw new Error("Missing OAuth request ID");

  console.log(`Open this URL and approve access: ${approvalUrl}`);

  let redirectTo;
  for (let attempt = 0; attempt < 300; attempt += 1) {
    const completionUrl = new URL("/public/connectors/auth/oauth/completion", API_BASE_URL);
    completionUrl.searchParams.set("request_id", requestId);
    completionUrl.searchParams.set("state", state);

    const completionResponse = await fetch(completionUrl);
    const completion = await completionResponse.json();
    if (!completion.success) {
      throw new Error(completion.error?.message || "OAuth completion failed");
    }
    if (completion.data.redirect_to) {
      redirectTo = completion.data.redirect_to;
      break;
    }
    await sleep((completion.data.retry_after || 2) * 1000);
  }

  if (!redirectTo) throw new Error("Timed out waiting for approval");

  const callbackUrl = new URL(redirectTo);
  if (callbackUrl.searchParams.get("state") !== state) throw new Error("OAuth state mismatch");

  const denied = callbackUrl.searchParams.get("error");
  if (denied) throw new Error(`OAuth authorization failed: ${denied}`);

  const code = callbackUrl.searchParams.get("code");
  if (!code) throw new Error("Missing authorization code");

  const tokenResponse = await fetch(
    new URL("/public/connectors/auth/oauth/token", API_BASE_URL),
    {
      method: "POST",
      headers: { accept: "application/json", "content-type": "application/json" },
      body: JSON.stringify({
        grant_type: "authorization_code",
        code,
        redirect_uri: REDIRECT_URI,
        client_id: CLIENT_ID,
        code_verifier: codeVerifier,
      }),
    },
  );

  const token = await tokenResponse.json();
  if (!tokenResponse.ok || !token.access_token) {
    throw new Error(token.error?.message || "OAuth token exchange failed");
  }
  return token;
}
```

**Pure fetch session calls:**

- Inspect: `GET /public/connectors/auth/me` — `Authorization: Bearer <access_token>`
- Refresh: `POST /public/connectors/auth/refresh` — `{ "refresh_token": "<refresh_token>" }` (preserve original if response omits a new one)
- Revoke: `POST /public/connectors/auth/revoke` — `Authorization: Bearer <access_token>`, `{ "session_id": "<session_id>" }`

Feature calls use the same endpoints and params from [Direct fetch feature routes](#direct-fetch-feature-routes). Always send `Authorization: Bearer <access_token>`.

## Output format

CLI: use `--json` whenever output will be parsed or used by a subsequent step. Auth errors surfaced: `access_denied`, `invalid_token`, `expired_token`, `unauthorized`.

Direct fetch envelope:
- Success: `{ success: true, data: { ... } }`
- Failure: `{ success: false, error: { code, message } }`

## Non-goals

- No browser automation beyond the user approval step during OAuth login.
- Do not use removed device-auth endpoints; connector authentication is OAuth-only.
