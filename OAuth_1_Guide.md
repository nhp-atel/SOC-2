# OAuth 1.0a — Hands-On Guide for Engineers

**Goal:** Understand OAuth 1.0a deeply enough to (a) explain it confidently in an interview and (b) implement a real OAuth 1.0a flow yourself in Postman.

**Time:** ~20 min reading + ~20 min hands-on in Postman.

**Prereqs:** Postman installed, a free Trello account (we'll use Trello as our test API since it still supports OAuth 1.0a in 2026).

---

## 1. The 30-Second Pitch (memorize this)

> **OAuth is a protocol that lets a user grant a third-party application limited access to their data on another service — without ever sharing their password with that third party.**

Before OAuth, if you wanted "App X" to read your Gmail, you'd hand App X your Gmail password. Bad: App X now has full access forever, can change your password, can read everything, and you have to trust it not to leak the password.

OAuth solves this by introducing a **token**: the user authorizes the app once via the real service's login page, and the service hands the app a *scoped, revocable token*. The app uses the token instead of the password.

**OAuth 1.0a (2009)** was the first widely-adopted version. It signs every request with a cryptographic signature so requests can't be tampered with — even over plain HTTP. **OAuth 2.0 (2012)** simplified this by relying on HTTPS for transport security and dropping the per-request signing.

---

## 2. Vocabulary Cheatsheet

You **must** be able to name these in an interview. OAuth 1.0a uses three roles:

| Role | What it is | Trello example |
|------|-----------|----------------|
| **Resource Owner** | The human user who owns the data | You |
| **Consumer** (a.k.a. Client) | The third-party app asking for access | Your Postman setup |
| **Service Provider** | The service hosting the data and issuing tokens | Trello |

And four credentials:

| Credential | Who has it | Purpose |
|-----------|-----------|---------|
| **Consumer Key** | Public, identifies your app | Like a username for your app |
| **Consumer Secret** | Private, never sent on the wire | Used to sign requests (proves your app is the real one) |
| **Access Token** | Issued after user authorizes | Identifies the user-app pair |
| **Token Secret** | Private, paired with the access token | Used along with consumer secret to sign requests |

**Key insight:** Two secrets are combined into the signing key (`consumer_secret&token_secret`). This means a leaked access token alone is useless without the consumer secret — and vice versa.

---

## 3. The "3-Legged" Flow

OAuth 1.0a is called **3-legged** because three parties participate: the user, the app, and the service.

```
   ┌──────────┐                    ┌──────────────┐                    ┌──────────────┐
   │   USER   │                    │  YOUR APP    │                    │   TRELLO     │
   │ (you)    │                    │  (Postman)   │                    │ (provider)   │
   └────┬─────┘                    └──────┬───────┘                    └──────┬───────┘
        │                                 │                                   │
        │                                 │  1. POST /OAuthGetRequestToken    │
        │                                 │ ────────────────────────────────► │
        │                                 │                                   │
        │                                 │  ◄──── request_token + secret ─── │
        │                                 │                                   │
        │  2. Open browser to Trello      │                                   │
        │  ◄────── authorize URL ──────── │                                   │
        │                                                                     │
        │  3. Log in & click "Allow"                                          │
        │ ──────────────────────────────────────────────────────────────────► │
        │                                                                     │
        │  ◄──── verifier code shown on page ──────────────────────────────── │
        │                                 │                                   │
        │  4. Paste verifier into Postman │                                   │
        │ ──────────────────────────────► │                                   │
        │                                 │  5. POST /OAuthGetAccessToken     │
        │                                 │       (with verifier)             │
        │                                 │ ────────────────────────────────► │
        │                                 │                                   │
        │                                 │  ◄──── access_token + secret ──── │
        │                                 │                                   │
        │                                 │  6. GET /members/me               │
        │                                 │       (signed with access token)  │
        │                                 │ ────────────────────────────────► │
        │                                 │                                   │
        │                                 │  ◄──── your Trello profile ────── │
        │                                 │                                   │
```

**Three legs:**
1. **Leg 1:** App ↔ Provider — get a temporary request token
2. **Leg 2:** User ↔ Provider — user authorizes in browser, gets a verifier
3. **Leg 3:** App ↔ Provider — exchange request token + verifier for an access token

After leg 3, the app holds an access token and can make signed API calls on behalf of the user.

---

## 4. Hands-On: Implement the Full Flow in Postman

### Step 0 — Get your Trello consumer credentials

1. Sign in to Trello at https://trello.com
2. Go to https://trello.com/power-ups/admin
3. Click **"New"** → fill in any name (e.g. "OAuth Learning"), workspace = your personal workspace, leave iframe URL blank
4. Once created, click your new Power-Up → **"API key"** tab in the left sidebar
5. Click **"Generate a new API key"**
6. Copy the **API Key** (this is your **Consumer Key**)
7. Click **"Generate a new secret"** next to OAuth secret → copy the **OAuth Secret** (this is your **Consumer Secret**)

Keep these two values handy. **Never commit them to git.**

### Step 1 — Set up a Postman environment

In Postman:

1. Click the **Environments** tab on the left → **+** → name it `Trello OAuth1`
2. Add these variables (set both Initial and Current values; the secret one should be type "secret"):

| Variable | Initial Value |
|----------|--------------|
| `consumer_key` | *(paste your Trello API Key)* |
| `consumer_secret` | *(paste your Trello OAuth Secret)* |
| `request_token` | *(leave blank — we'll fill in later)* |
| `request_token_secret` | *(leave blank)* |
| `access_token` | *(leave blank)* |
| `access_token_secret` | *(leave blank)* |

3. **Save** the environment, then **select it** from the environment dropdown in the top right.

### Step 2 — Leg 1: Get a request token

Create a new request in Postman:

- **Method:** `POST`
- **URL:** `https://trello.com/1/OAuthGetRequestToken`

Configure the **Authorization** tab:
- **Type:** `OAuth 1.0`
- **Add authorization data to:** `Request Headers`
- **Signature Method:** `HMAC-SHA1`
- **Consumer Key:** `{{consumer_key}}`
- **Consumer Secret:** `{{consumer_secret}}`
- **Token:** *(leave blank)*
- **Token Secret:** *(leave blank)*
- **Callback URL:** `oob` *(stands for "out-of-band" — Trello will show the verifier in the browser instead of redirecting)*
- **Timestamp / Nonce:** *(leave blank — Postman generates them)*
- **Version:** `1.0`
- **Realm:** *(leave blank)*

Click **Send**.

**Expected response (200 OK):**
```
oauth_token=ABC123...&oauth_token_secret=XYZ789...&oauth_callback_confirmed=true
```

This is **form-urlencoded**, not JSON. Copy the two values:
- Paste the `oauth_token` value into the `request_token` environment variable
- Paste the `oauth_token_secret` value into the `request_token_secret` variable

> **What just happened:** Your app proved its identity (consumer key + signature) and got a *temporary* token. This token has not been authorized by any user yet.

### Step 3 — Leg 2: User authorization in browser

Open this URL in your browser (replace `YOUR_REQUEST_TOKEN` with the value you just copied):

```
https://trello.com/1/OAuthAuthorizeToken?oauth_token=YOUR_REQUEST_TOKEN&name=OAuthLearning&scope=read&expiration=1day
```

You'll see a Trello page asking: *"OAuthLearning would like to access your Trello account. Allow?"*

Click **Allow**.

Trello will display a **verifier code** (a short string like `abcd1234`). Copy it.

> **What just happened:** Trello confirmed *you* (the resource owner) authorized this specific request token. The verifier is proof that the user actually clicked Allow.

### Step 4 — Leg 3: Exchange for access token

Create another POST request in Postman:

- **Method:** `POST`
- **URL:** `https://trello.com/1/OAuthGetAccessToken`

**Authorization** tab:
- **Type:** `OAuth 1.0`
- **Consumer Key:** `{{consumer_key}}`
- **Consumer Secret:** `{{consumer_secret}}`
- **Token:** `{{request_token}}`
- **Token Secret:** `{{request_token_secret}}`
- **Verifier:** *(paste the verifier code from the browser)*
- **Signature Method:** `HMAC-SHA1`

Click **Send**.

**Expected response (200 OK):**
```
oauth_token=NEW_ACCESS_TOKEN&oauth_token_secret=NEW_SECRET
```

Copy these into `access_token` and `access_token_secret` in your environment.

> **What just happened:** Your app traded the request token + verifier for a *long-lived* access token. The request token is now dead.

### Step 5 — Make a signed API call

Create one more request:

- **Method:** `GET`
- **URL:** `https://api.trello.com/1/members/me`

**Authorization** tab:
- **Type:** `OAuth 1.0`
- **Consumer Key:** `{{consumer_key}}`
- **Consumer Secret:** `{{consumer_secret}}`
- **Token:** `{{access_token}}`
- **Token Secret:** `{{access_token_secret}}`
- **Signature Method:** `HMAC-SHA1`

Click **Send**.

**Expected response (200 OK):** A JSON object with your Trello profile (id, username, email, boards, etc.).

🎉 **You just completed a full OAuth 1.0a flow.**

### Bonus: Inspect the Authorization header

Click the **Headers** tab of your last request → expand the auto-generated `Authorization` header. You'll see something like:

```
OAuth oauth_consumer_key="abc123",
      oauth_token="def456",
      oauth_signature_method="HMAC-SHA1",
      oauth_timestamp="1745701234",
      oauth_nonce="kllo9940pd9333jh",
      oauth_version="1.0",
      oauth_signature="tR3%2BTy81lMeYAr%2Fl5%2F..."
```

**Every field here matters.** This is what we cover next.

---

## 5. Mechanics You Should Be Able to Name

You don't need to compute these — but you should know what they are and why they exist.

### Signature Base String
A canonical string built from: HTTP method + URL + all parameters (sorted alphabetically). Example:
```
POST&https%3A%2F%2Ftrello.com%2F1%2FOAuthGetRequestToken&oauth_consumer_key%3Dabc...
```
**Why it exists:** Both client and server compute this same string independently and check signatures match. If anything in the request was tampered with mid-flight (parameter, URL, method), the signatures won't match.

### HMAC-SHA1 Signing
The signature base string is hashed with HMAC-SHA1 using a key built as: `consumer_secret&token_secret` (URL-encoded, joined by `&`). The resulting hash is the `oauth_signature`.
**Why HMAC:** It's a **keyed** hash — only someone holding the secrets can produce a valid signature. Verifies *both* integrity and authenticity in one step.

### Nonce
A random string, unique per request (e.g. `kllo9940pd9333jh`).
**Why it exists:** **Replay protection.** Even if an attacker intercepts a signed request, they can't resend it — the server tracks recently-used nonces and rejects duplicates.

### Timestamp
Unix epoch seconds when the request was made.
**Why it exists:** Limits how long a captured request stays "fresh." Combined with nonce: the server only needs to remember nonces from the last few minutes (e.g. 5 min window), not forever. Replays older than the window are rejected based on timestamp alone.

### Why everything goes in the Authorization header
You *can* put OAuth params in the query string or the body, but the header is preferred because:
1. It keeps URLs clean (no signature in your logs/browser history)
2. It separates auth from application data
3. The body stays available for actual payload (e.g. JSON)

---

## 6. Why OAuth 2.0 Replaced It

Three reasons, in interview-ready form:

1. **Signing is hard.** OAuth 1.0 client libraries had constant interop bugs around URL encoding, parameter sorting, and base string construction. OAuth 2.0 dropped signing entirely and just requires HTTPS — *the network handles integrity, not the protocol*.
2. **Mobile and SPAs needed flexibility.** OAuth 1.0 has one flow. OAuth 2.0 defines multiple **grant types** (authorization code, client credentials, device code, etc.) for different client types — server apps, mobile apps, IoT devices, machine-to-machine.
3. **Bearer tokens are simpler.** OAuth 2.0 access tokens are just opaque strings sent as `Authorization: Bearer <token>`. No signing, no nonces, no timestamps. The trade-off: *the token is the credential* — if someone steals it, they're you. (That's why OAuth 2.0 absolutely requires HTTPS, and why short-lived tokens + refresh tokens became standard.)

---

## 7. Interview Q&A (drill these)

**Q: What problem does OAuth solve?**
> Delegated access. It lets a user grant a third-party app limited access to their data on a service without sharing their password with the third party.

**Q: What does "3-legged" mean?**
> Three parties participate: the user (resource owner), the app (consumer), and the service (provider). The flow has three legs: app gets a request token, user authorizes in browser, app exchanges for access token.

**Q: What's a nonce and why do you need one?**
> A nonce is a unique random string per request. It prevents replay attacks — even if an attacker captures a valid signed request, they can't resend it because the server tracks used nonces and rejects duplicates.

**Q: Why does OAuth 1.0 also have a timestamp if it has a nonce?**
> Timestamp lets the server expire nonces. Without it, the server would have to remember every nonce forever. With a timestamp + reasonable window (e.g. 5 minutes), the server only tracks nonces from that recent window and rejects anything older outright.

**Q: How is the OAuth 1.0 signature computed?**
> Build a canonical signature base string: HTTP method + URL + all parameters sorted alphabetically. Then HMAC-SHA1-sign it with a key made from `consumer_secret&token_secret`. The resulting hash goes in the `oauth_signature` header.

**Q: Why is OAuth 1.0 considered more secure-by-default than 2.0?**
> Because every request is cryptographically signed, OAuth 1.0 can run safely over plain HTTP — tampering is detectable and tokens can't be replayed. OAuth 2.0 relies entirely on HTTPS for transport security; if HTTPS fails or is misconfigured, bearer tokens are sniffable and immediately reusable. The trade-off was complexity vs. simplicity — the industry chose simplicity.

**Q: Why did the industry move to OAuth 2.0 then?**
> Three reasons: signing was a constant source of interop bugs, OAuth 1.0 only defined one flow (couldn't accommodate mobile, SPAs, M2M, IoT), and HTTPS had become near-universal — making per-request signing redundant in practice.

**Q: Difference between consumer secret and token secret?**
> Consumer secret identifies the app — it's issued when you register the app. Token secret identifies the user-app authorization — it's issued after the user clicks "Allow." Both are combined to form the signing key, so leaking one alone doesn't compromise the other.

**Q: What is "oob" in the callback URL?**
> Out-of-band. Used when the app can't receive a redirect (e.g. CLI tools, native apps without a deep link). The provider displays the verifier code on screen and the user pastes it into the app manually.

---

## 8. Quick Recap (the version you say in 60 seconds)

> "OAuth 1.0 is a delegated authorization protocol. The user authorizes a third-party app to access their data on a service, without sharing their password. It's called 3-legged because three parties are involved — user, app, service — and the flow has three legs: the app gets a temporary request token, the user authorizes it in the browser and receives a verifier, then the app exchanges the request token plus verifier for a long-lived access token. Every API call is then signed using HMAC-SHA1 over a canonical signature base string, with a nonce and timestamp for replay protection. OAuth 2.0 later replaced it by relying on HTTPS for transport security and using bearer tokens instead of signing — simpler to implement but more dependent on TLS being correctly configured."

---

## Next Up
Once this clicks, we'll do the OAuth 2.0 guide — same hands-on style, with Microsoft Graph or GitHub as the test API. You'll see how the four 2.0 grant types map to different client situations, and why the authorization code + PKCE flow is the modern default.
