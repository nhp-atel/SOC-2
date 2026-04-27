# OAuth 2.0 — Hands-On Guide for Engineers (with Microsoft Graph)

**Goal:** Understand OAuth 2.0 deeply enough to (a) explain it confidently in an interview and (b) implement a real OAuth 2.0 flow yourself in Postman against Microsoft Graph.

**Time:** ~25 min reading + ~25 min hands-on.

**Prereqs:**
- Postman installed
- A free Microsoft account (personal `outlook.com`/`hotmail.com` works, or any work/school Microsoft account)
- Read the OAuth 1.0 guide first — many concepts (consumer/provider/resource owner, tokens, scopes) carry over.

**Why Microsoft Graph?** Same OAuth 2.0 as GitHub/Google/Slack — but it's the exact auth flow used by Logic Apps, Power Automate, and Azure connectors, so it doubles as interview prep for Microsoft-shop roles.

---

## 1. The 30-Second Pitch (memorize this)

> **OAuth 2.0 is a delegated authorization framework. It lets a user grant a third-party app limited, scoped, revocable access to their data on a service — without sharing credentials. It does this by issuing the app a short-lived bearer token over HTTPS, instead of OAuth 1.0's per-request signing.**

Same problem as OAuth 1.0 — but the implementation is simpler:
- No per-request signing → tokens are just opaque strings
- Multiple **grant types** for different client situations (web apps, mobile apps, server-to-server, IoT)
- Relies on **HTTPS** for transport security (no HMAC needed)
- Adds **refresh tokens** so users don't have to re-authorize daily

OAuth 2.0 is what every modern API uses: Microsoft Graph, Google APIs, GitHub, Slack, Stripe, Salesforce, AWS Cognito, Okta.

---

## 2. Vocabulary Cheatsheet

OAuth 2.0 renamed almost everything from 1.0. Memorize the new names:

| OAuth 2.0 Term | What it is | OAuth 1.0 equivalent |
|---------------|-----------|----------------------|
| **Resource Owner** | The user who owns the data | Same |
| **Client** | The third-party app | Consumer |
| **Authorization Server** | The server that issues tokens (e.g. `login.microsoftonline.com`) | Service Provider (auth side) |
| **Resource Server** | The API serving the protected data (e.g. `graph.microsoft.com`) | Service Provider (data side) |
| **Client ID** | Public identifier for your app | Consumer Key |
| **Client Secret** | Private credential for your app (server-side apps only) | Consumer Secret |
| **Access Token** | Short-lived bearer token, used in `Authorization: Bearer <token>` header | Access Token (but no signing) |
| **Refresh Token** | Long-lived token used to get new access tokens without user interaction | *(no equivalent in 1.0)* |
| **Scope** | Specific permission requested (e.g. `User.Read`, `Mail.Read`) | Loosely parallel to `scope=read/write` |
| **Authorization Code** | One-time code returned in browser redirect, exchanged for a token | Verifier |
| **State** | Random value to prevent CSRF on the redirect | *(no direct equivalent)* |

**Key conceptual split (NEW in 2.0):** the **Authorization Server** (issues tokens) and the **Resource Server** (serves data) are now distinct concepts. Sometimes one company runs both (Microsoft, Google), but the protocol treats them separately.

---

## 3. The Four Grant Types (when to use each)

OAuth 2.0's biggest improvement over 1.0 is **multiple grant types** for different client situations. You must be able to name all four in an interview.

### 3a. Authorization Code Grant ⭐ (the most common, what we'll implement)
**Use when:** A web app with a backend server is acting on behalf of a logged-in user.
**Flow:** User → browser → auth server → user logs in & consents → redirect with one-time `code` → backend exchanges `code` + `client_secret` for access token.
**Why it's the default:** The access token never touches the browser; only the short-lived `code` does. The actual token exchange happens server-to-server.

### 3b. Authorization Code with PKCE
**Use when:** Single-page apps (React, Vue), mobile apps, native desktop apps — clients that **can't safely store a client secret**.
**Flow:** Same as Authorization Code, but the client generates a random `code_verifier`, hashes it to a `code_challenge`, sends the challenge with the auth request, and proves possession of the verifier when exchanging the code.
**Why:** Without PKCE, an attacker who intercepts the redirect URL can steal the code and exchange it for a token. PKCE binds the code to the original client.
**Modern recommendation:** Use PKCE *even* for server-side apps. It's now considered best practice everywhere.

### 3c. Client Credentials Grant
**Use when:** Server-to-server, no user involved (e.g. cron job pulls data, microservice talks to another microservice).
**Flow:** Client just sends `client_id` + `client_secret` directly to the token endpoint, gets back an access token.
**Why simple:** No browser, no user, no consent screen. The app is acting as itself, not on behalf of a user.

### 3d. Device Authorization Grant (Device Code)
**Use when:** Devices without a browser or keyboard — smart TVs, CLI tools, IoT devices.
**Flow:** Device shows user a short code (e.g. `WDJB-MJHT`) and a URL. User opens the URL on their phone, enters the code, logs in. Device polls the auth server until the user finishes.
**Real example:** `gh auth login` on the command line, Netflix on your TV.

### Deprecated / don't use
- **Implicit Grant** — was for SPAs before PKCE existed. Returned access token directly in URL fragment. Insecure, deprecated as of OAuth 2.1.
- **Resource Owner Password Credentials** — user gives username/password directly to the app. Defeats the entire point of OAuth. Deprecated.

---

## 4. The Authorization Code Flow (the one we'll implement)

```
   ┌──────────┐              ┌──────────────┐         ┌────────────────────┐    ┌──────────────┐
   │   USER   │              │ YOUR APP     │         │ AUTHORIZATION      │    │ RESOURCE     │
   │ (you)    │              │ (Postman)    │         │ SERVER (login.MS)  │    │ SERVER       │
   └────┬─────┘              └──────┬───────┘         └─────────┬──────────┘    │ (graph.MS)   │
        │                           │                            │              └──────┬───────┘
        │                           │                            │                     │
        │                           │  1. Open browser to        │                     │
        │  ◄──── auth URL ──────────│     /authorize?...         │                     │
        │                           │                            │                     │
        │  2. Browser → /authorize ────────────────────────────► │                     │
        │                                                        │                     │
        │  3. Login + consent                                   │                     │
        │ ─────────────────────────────────────────────────────► │                     │
        │                                                        │                     │
        │  ◄──── 4. Redirect to callback?code=XYZ ─────────────  │                     │
        │                                                        │                     │
        │  5. Postman captures code from redirect                                      │
        │ ────────────────────────► │                            │                     │
        │                           │                            │                     │
        │                           │  6. POST /token            │                     │
        │                           │     code=XYZ               │                     │
        │                           │     client_id, secret      │                     │
        │                           │ ─────────────────────────► │                     │
        │                           │                            │                     │
        │                           │ ◄──── access_token + ──── │                     │
        │                           │       refresh_token        │                     │
        │                           │                            │                     │
        │                           │  7. GET /me                                      │
        │                           │     Authorization:                               │
        │                           │     Bearer <token>                               │
        │                           │ ──────────────────────────────────────────────► │
        │                           │                                                  │
        │                           │  ◄──── your profile JSON ─────────────────────  │
        │                           │                                                  │
```

**Compare to OAuth 1.0:** the ceremony is similar (browser dance, then exchange a short-lived code for a long-lived token), but no signing — the token is just an opaque string sent over HTTPS.

---

## 5. Hands-On: Authorization Code Flow with Microsoft Graph

### Step 0 — Register your app in Azure AD

1. Open https://portal.azure.com and sign in (any Microsoft account works — personal or work)
2. In the search bar at top, type **"App registrations"** → click it
3. Click **"+ New registration"**
4. Fill in:
   - **Name:** `OAuth Learning`
   - **Supported account types:** select **"Accounts in any organizational directory (Any Microsoft Entra ID tenant - Multitenant) and personal Microsoft accounts"** (this is the most permissive — works for any sign-in)
   - **Redirect URI:**
     - Platform: **Web**
     - URL: `https://oauth.pstmn.io/v1/callback` *(this is Postman's official OAuth callback — Postman captures the redirect here)*
5. Click **Register**

You'll land on the app's overview page. **Copy and save these two values:**
- **Application (client) ID** — your `client_id`
- **Directory (tenant) ID** — keep this handy too

Now create the client secret:

6. In the left sidebar, click **Certificates & secrets**
7. Under **Client secrets**, click **+ New client secret**
8. Description: `Postman testing`, Expires: `90 days` (or shorter), click **Add**
9. **Immediately copy the `Value` column** (NOT `Secret ID` — you want the long random string under `Value`). This is your `client_secret`.

> ⚠️ Azure shows the secret value **only once**. If you navigate away without copying, you have to delete it and create a new one.

Now configure permissions:

10. In the left sidebar, click **API permissions**
11. You'll see `User.Read` is already there by default — that's enough for our test (it's the permission to read the signed-in user's basic profile). No changes needed.

> 💡 If we wanted email or calendar access, we'd add `Mail.Read` or `Calendars.Read` here. The user gets to see the full list on the consent screen.

You should now have:
- **Client ID** (a UUID like `12345678-1234-1234-1234-123456789abc`)
- **Client Secret** (a long random string with letters, digits, and symbols)
- **Tenant ID** (another UUID, optional for our flow but good to have)

### Step 1 — Set up a Postman environment

1. Click the **Environments** tab in Postman → **+** → name it `MS Graph OAuth2`
2. Add these variables:

| Variable | Initial Value | Type |
|----------|--------------|------|
| `client_id` | *(paste your Application ID)* | default |
| `client_secret` | *(paste your client secret VALUE)* | secret |
| `tenant_id` | `common` | default |
| `auth_url` | `https://login.microsoftonline.com/{{tenant_id}}/oauth2/v2.0/authorize` | default |
| `token_url` | `https://login.microsoftonline.com/{{tenant_id}}/oauth2/v2.0/token` | default |
| `scope` | `User.Read offline_access` | default |
| `access_token` | *(leave blank)* | default |

> **Why `tenant_id = common`?** The `common` endpoint accepts any Microsoft account (personal or work). If you want to lock to a specific organization, replace with your tenant UUID.
> **Why `offline_access` in scope?** This is what tells Microsoft to issue a **refresh token** alongside the access token. Without it, you only get a short-lived access token and have to re-authorize when it expires.

3. **Save** the environment and **select it** in the top-right dropdown.

### Step 2 — Configure OAuth 2.0 in Postman

1. Create a new request:
   - **Method:** `GET`
   - **URL:** `https://graph.microsoft.com/v1.0/me`
2. Click the **Authorization** tab
3. **Type:** `OAuth 2.0`
4. Under **Configure New Token**, fill in:

| Field | Value |
|-------|-------|
| Token Name | `MS Graph Token` |
| Grant Type | `Authorization Code` |
| Callback URL | `https://oauth.pstmn.io/v1/callback` |
| Authorize using browser | ✅ **check this box** |
| Auth URL | `{{auth_url}}` |
| Access Token URL | `{{token_url}}` |
| Client ID | `{{client_id}}` |
| Client Secret | `{{client_secret}}` |
| Scope | `{{scope}}` |
| State | *(leave blank — Postman generates one)* |
| Client Authentication | `Send client credentials in body` |

> **What is `State`?** A random value sent with the auth request and echoed back in the callback. The client checks they match — if they don't, the redirect was forged (CSRF protection). Postman generates and validates this for you.

### Step 3 — Get an access token (the browser flow)

1. Scroll to the bottom of the Authorization tab → click **"Get New Access Token"**
2. Your browser opens → Microsoft login page appears
3. Sign in with your Microsoft account
4. **Consent screen** appears: *"OAuth Learning is requesting permission to: Sign you in and read your profile, Maintain access to data you have given it access to"* → click **Accept**
5. Browser redirects to `https://oauth.pstmn.io/v1/callback?code=...&state=...`
6. The Postman callback page captures the code, exchanges it for a token, and tells you "Authentication complete. You may close this window."
7. Back in Postman, a **Manage Access Tokens** dialog opens showing your access token + refresh token + expires_in. Click **Use Token**.

> **What just happened in slow motion:**
> - Postman built a URL: `{{auth_url}}?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...`
> - Browser hit it → Microsoft showed login → you authenticated → Microsoft redirected back to Postman's callback with `?code=ONE_TIME_CODE`
> - Postman extracted that code and made a **server-side POST to `{{token_url}}`** with the code, client_id, and client_secret to exchange it for the actual access token
> - The code-for-token exchange happens out of the browser, so the access token never appears in any URL or browser history

### Step 4 — Call Microsoft Graph

1. The Authorization tab now shows the new token applied to your request
2. Click **Send**
3. **Expected response (200 OK):** JSON with your Microsoft profile:

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
  "displayName": "Nimesh Patel",
  "mail": "ankita.patel@codeticks.com",
  "userPrincipalName": "ankita.patel@codeticks.com",
  "id": "abc-def-..."
}
```

🎉 **You just completed the OAuth 2.0 Authorization Code flow.**

### Step 5 — Inspect the Authorization header

Click the **Headers** tab on your request → expand `Authorization`. You'll see:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImt...
```

That's it. **No signing, no nonce, no timestamp.** Just the bearer token in plain text. The entire security model rests on:
1. HTTPS protecting the token in transit
2. The token expiring in ~1 hour
3. The user being able to revoke it from their Microsoft account

This simplicity is OAuth 2.0's blessing and its curse — anyone who steals the bearer token *is you* until it expires.

### Bonus: Decode the token (it's a JWT)

Microsoft access tokens are **JWTs** (JSON Web Tokens). Copy the token, paste it at https://jwt.ms (Microsoft's official JWT decoder — never paste real production tokens into third-party tools).

You'll see three sections separated by dots:
- **Header** — `{"alg":"RS256","typ":"JWT","kid":"..."}`  → which algorithm signed it
- **Payload** — the actual claims: `aud` (audience), `iss` (issuer), `exp` (expiry), `scp` (scopes granted), `oid` (your user ID), etc.
- **Signature** — RS256 signature of header + payload, verifiable with Microsoft's public key

Most APIs use opaque tokens (random strings); Microsoft and Auth0 use JWTs because resource servers can verify them locally without calling the auth server on every request.

### Bonus: Use the refresh token

When the access token expires (~1 hour), instead of re-running the browser flow, Postman can silently use the refresh token to get a new access token:

1. In the Authorization tab → **Manage Access Tokens** → click your token → click **Refresh**
2. Postman POSTs to `{{token_url}}` with `grant_type=refresh_token` and your refresh token → gets back a fresh access token
3. No browser needed, no user interaction. This is what makes "stay logged in" work in real apps.

---

## 6. Mechanics You Should Be Able to Name

### Bearer Token
An opaque (or JWT) string sent in the `Authorization: Bearer <token>` header. **The token IS the credential** — possession alone is sufficient. No signing, no proof of identity beyond holding the string.
**Why it exists:** Simple. Stateless from the resource server's perspective (especially for JWTs).
**Trade-off:** Theft = instant compromise until expiry. Mitigated with short lifetimes + refresh tokens + HTTPS.

### Refresh Token
A long-lived token (days to months) issued alongside the access token. Used to silently get new access tokens without re-prompting the user.
**Why it exists:** Lets access tokens be short-lived (≤1 hour) — so a stolen access token has limited damage — without forcing users to log in every hour.
**Important:** Refresh tokens are NEVER sent to resource servers. They go only to the auth server's token endpoint.

### Scope
A space-separated list of permissions the app is requesting (`User.Read Mail.Read`). The user sees these on the consent screen and can deny specific ones.
**Why it exists:** Principle of least privilege. The app should ask for only what it needs. Scopes scope the access token — if the app asked for `User.Read`, the token can't be used to call mail endpoints.

### State
A random value sent in the auth request and echoed back in the callback. Client verifies they match.
**Why it exists:** **CSRF protection.** Without it, an attacker could trick a logged-in user into hitting a callback URL with the attacker's auth code, linking the victim's session to the attacker's account.

### PKCE (Proof Key for Code Exchange)
The client generates a random `code_verifier`, hashes it to `code_challenge`, sends the challenge with the auth request, sends the verifier with the token exchange.
**Why it exists:** Without it, a malicious app on the same device (or a network attacker) could intercept the auth code from the redirect and exchange it for tokens. PKCE binds the code to the client that started the flow.

### Authorization Code
A one-time, short-lived code (~10 min) returned in the redirect after user consent. Exchanged at the token endpoint for an access token.
**Why it exists:** So the actual access token never touches the browser/URL/history. Codes are useless without the client_secret (or PKCE verifier).

### Why everything goes in the Authorization header
Same reasons as OAuth 1.0: clean URLs, separates auth from data, body stays free for payload.

---

## 7. OAuth 1.0 vs 2.0 — The Comparison Table

| Aspect | OAuth 1.0a | OAuth 2.0 |
|--------|-----------|-----------|
| Year | 2009 | 2012 |
| Token format | Token + secret pair | Bearer token (string) |
| Per-request signing | HMAC-SHA1 over base string | None |
| Transport security | Worked over plain HTTP (signing) | Requires HTTPS |
| Replay protection | Nonce + timestamp | Token expiry (no replay protection on token itself) |
| Grant types | One (3-legged) | Four (Auth Code, PKCE, Client Credentials, Device) |
| Refresh tokens | None | Yes |
| Mobile/SPA friendly | No (secret can't be hidden in client) | Yes (PKCE, public clients) |
| Implementation difficulty | High (signing bugs everywhere) | Low (string concatenation) |
| Risk if HTTPS fails | Low (request still signed) | High (token sniffable, immediately reusable) |
| Used by today | Trello, Discogs, Flickr (legacy) | Everyone else |

**The trade-off in one sentence:** OAuth 2.0 traded protocol-level security for HTTPS-level security to gain simplicity and flexibility. The bet was right because HTTPS became universal.

---

## 8. Interview Q&A (drill these)

**Q: What's the difference between authentication and authorization?**
> Authentication is "who are you." Authorization is "what are you allowed to do." OAuth 2.0 is purely an **authorization** framework — it tells a resource server "this token holder is allowed to do X" but doesn't standardize how to identify the user. OpenID Connect (OIDC) is the layer built on top of OAuth 2.0 that adds standardized authentication.

**Q: Walk me through the Authorization Code flow.**
> The user clicks "Sign in with X" in the app. App redirects browser to the authorization server's `/authorize` endpoint with client_id, scope, redirect_uri, and a state value. User logs in and grants consent. Auth server redirects back to the app's redirect URI with a one-time code. The app's backend exchanges the code plus client_secret at the `/token` endpoint for an access token (and optionally a refresh token). The app then calls the resource server with `Authorization: Bearer <token>`.

**Q: When would you use Client Credentials instead of Authorization Code?**
> Server-to-server with no user involved. Examples: a backend job calling another internal API, a Logic App or Power Automate flow running unattended, a microservice fetching data from another microservice. There's no user consent because the app is acting as itself.

**Q: Why PKCE?**
> Because public clients (mobile apps, SPAs) can't store a client_secret safely. Without a secret, an attacker who intercepts the redirect URL can exchange the auth code for a token. PKCE solves this: the client generates a random verifier, hashes it to a challenge, sends the challenge with the auth request, and proves possession of the verifier when exchanging the code. The auth server only honors the exchange if the verifier hashes to the original challenge.

**Q: What's the purpose of the `state` parameter?**
> CSRF protection. The client generates a random state, sends it with the auth request, and verifies the same value comes back in the callback. Without it, an attacker could trick a logged-in user into completing an OAuth flow against the attacker's account, hijacking the victim's session in the app.

**Q: What's the purpose of a refresh token?**
> Lets the app get a new access token without user interaction when the old one expires. This allows access tokens to be short-lived (limiting damage if stolen) while keeping the user logged in for long periods. Refresh tokens are never sent to resource servers — only to the auth server's token endpoint.

**Q: Why is OAuth 2.0 simpler than OAuth 1.0 but considered less secure by default?**
> 2.0 dropped per-request signing in favor of bearer tokens over HTTPS. Simpler to implement, but the token IS the credential — anyone holding it is you. 1.0 required signing every request with HMAC-SHA1, so even an intercepted request couldn't be replayed or tampered with. The industry bet HTTPS would become universal, and that bet paid off.

**Q: What's a JWT and how does it relate to OAuth?**
> JWT is a token format — JSON claims signed with a key, base64-encoded into three dot-separated sections (header.payload.signature). OAuth 2.0 doesn't mandate JWTs (tokens can be opaque random strings), but many providers (Microsoft, Auth0, Okta) use JWTs because resource servers can verify them locally with the auth server's public key, no network call needed per request.

**Q: What's the difference between the Authorization Server and the Resource Server?**
> Authorization Server issues tokens (e.g. `login.microsoftonline.com`). Resource Server hosts the protected data and validates tokens (e.g. `graph.microsoft.com`). Sometimes one company runs both, but architecturally they're separate. This separation is new in OAuth 2.0; OAuth 1.0 conflated them as "Service Provider."

**Q: What is a scope?**
> A specific permission the app is requesting (e.g. `User.Read`, `Mail.Read`, `Files.ReadWrite`). The user sees scopes on the consent screen and can deny granular permissions. The issued access token is then constrained to those scopes — calling an out-of-scope endpoint returns 403.

**Q: What happens when an access token expires?**
> The resource server returns 401. If the app has a refresh token, it silently POSTs to the auth server's token endpoint with `grant_type=refresh_token` and gets a fresh access token. If not, the user has to go through the full auth flow again.

**Q: What's OpenID Connect?**
> OIDC is an authentication layer built on OAuth 2.0. It adds an `id_token` (a JWT containing user identity claims like `sub`, `name`, `email`) and a standardized `/userinfo` endpoint. "Sign in with Google" / "Sign in with Microsoft" is OIDC, not raw OAuth. Rule of thumb: **OAuth 2.0 = authorization, OIDC = authentication on top of OAuth 2.0**.

---

## 9. Quick Recap (the version you say in 60 seconds)

> "OAuth 2.0 is a delegated authorization framework — it lets a user grant a third-party app limited, scoped, revocable access to their data without sharing credentials. It defines four grant types for different client situations: Authorization Code for web apps with backends, Authorization Code with PKCE for mobile and single-page apps, Client Credentials for server-to-server with no user, and Device Code for browserless devices. The most common is Authorization Code: the user authenticates and consents in the browser, the auth server redirects back with a one-time code, and the app's backend exchanges that code plus its client secret for an access token. The token is a bearer token sent in the Authorization header — possession alone is sufficient, so the model relies on HTTPS, short token lifetimes, and refresh tokens. OpenID Connect is the authentication layer built on top of OAuth 2.0, adding an ID token with user identity claims."

---

## 10. Real-World Context (interview color)

**Where you'll hit OAuth 2.0 in enterprise work:**
- **Microsoft Graph / Azure AD** — every Microsoft 365 integration, Logic Apps connectors, Power Automate, custom enterprise apps
- **Salesforce, ServiceNow, Workday** — most enterprise SaaS uses OAuth 2.0 for API access
- **Okta, Auth0, AWS Cognito** — managed identity providers that handle the OAuth 2.0 flows for you
- **GitHub Apps, Slack apps, Stripe Connect** — third-party platform integrations all use OAuth 2.0
- **API Gateway patterns** — bearer token validation at the edge before hitting backend services

**In the context of Logic Apps / Power Automate (UPS Healthcare relevance):**
- Connectors authenticate to APIs using OAuth 2.0 under the hood — usually Authorization Code flow for user-context connectors and Client Credentials for unattended flows
- "Custom connector" setup in Power Platform often involves filling in the same fields you just used in Postman: auth URL, token URL, client ID, client secret, scope
- Service principals in Azure (apps registered without a human user) use Client Credentials grant

---

## Done
You now know more than the average developer about both OAuth 1.0a and 2.0 — and crucially, you've actually executed both flows yourself. Drill the Q&A sections aloud a few times before your interview.
