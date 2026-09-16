# Authentication & Authorization

## TL;DR

Every auth scheme is an answer to the question **"who are you, and how do I keep asking without being annoying or insecure?"** The history is a chain of fixes: send the password every time → stop sending it, keep a server-side session → stop sharing the password with third-party apps → simplify that mess → stop storing session state on the server.

Two words that get mixed up constantly:

- **Authentication (AuthN)** — who are you? (login)
- **Authorization (AuthZ)** — what are you allowed to do? (permissions)

OAuth is *authorization*. JWT is a *token format*. Neither is "login" by itself.

---

## The timeline at a glance

| Year | What arrived | Problem it solved | Problem it left behind |
|------|-------------|-------------------|------------------------|
| 1996 | **HTTP Basic Auth** (RFC 1945/2617) | A standard way to prove identity over HTTP | Password sent on *every* request; no logout; no UI control |
| ~1995–2000 | **Session + Cookie auth** | Password sent once; server remembers you | Server-side state; scaling pain; CSRF |
| 2007–2010 | **OAuth 1.0 / 1.0a** (RFC 5849) | Third-party apps stop asking for your password | Request signing was brutal to implement |
| 2012 | **OAuth 2.0** (RFC 6749) | Dropped signatures, leaned on TLS; simple + flexible | It's a *framework*, not a protocol — many wrong ways to use it |
| 2014 | **OpenID Connect** | OAuth 2 said nothing about *identity*; everyone invented their own login | One more spec to get wrong; `id_token` vs `access_token` confusion |
| 2015 | **JWT** (RFC 7519) | Self-contained, stateless, verifiable token | Can't revoke easily; people put secrets in it; alg confusion |

```
1996 ──────── 1997 ──────── 2007 ─── 2010 ─── 2012 ─── 2014 ─── 2015 ──────── 2019 ───▶
  │             │             │        │        │        │        │             │
Basic        Session       OAuth 1   OAuth    OAuth 2   OIDC     JWT         WebAuthn
 Auth        Cookies         .0       1.0a    RFC 6749  (authN   RFC 7519    / Passkeys
                                    RFC 5849            on top
                                                       of OAuth2)
```

---

## 1. HTTP Basic Auth (1996)

The original. Built straight into the HTTP spec.

**How it works**

```http
# 1. Client asks for a protected resource
GET /admin HTTP/1.1

# 2. Server says "nope, authenticate"
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Admin Area"

# 3. Client resends with credentials on EVERY request
GET /admin HTTP/1.1
Authorization: Basic aGFyc2g6c3VwZXJzZWNyZXQ=
                     └── base64("harsh:supersecret")
```

**Key point:** `base64` is *encoding*, not encryption. Anyone who sees the header has the password in plaintext. Over HTTP (no TLS) this is equivalent to shouting your password.

**Why it wasn't enough**

- Password travels on every single request → more chances to leak
- No real logout (the browser caches credentials until you close it)
- No custom login page — you get the browser's ugly native popup
- Server must be able to compare the raw password → limits how you store it
- No concept of "session expiry", "remember me", or roles

**Still used today for:** server-to-server APIs, internal tooling, `curl` scripts — always over HTTPS.

---

## 2. Session-Based Auth (cookies, late 1990s)

Cookies arrived in Netscape (1994, later RFC 6265) and gave the web the missing piece: **the browser can carry a small value automatically on every request.**

Now you send the password *once*.

**How it works**

```
1. POST /login  { email, password }
2. Server verifies against the DB (password is hashed — bcrypt/argon2)
3. Server creates a session record:
      sessions["a7f3c9..."] = { user_id: 42, expires: ... }   ← server memory / Redis / DB
4. Server responds:
      Set-Cookie: session_id=a7f3c9...; HttpOnly; Secure; SameSite=Lax
5. Browser attaches that cookie automatically on every later request
6. Server looks up the session ID → knows who you are
```

The cookie is an **opaque pointer**, not data. It means nothing on its own — all the truth lives on the server.

**What this unlocked**

- Real logout → just delete the server-side session
- Custom login pages, "remember me", expiry, forced logout everywhere
- Password never travels again after step 1

**The flags that matter**

| Flag | What it does |
|------|-------------|
| `HttpOnly` | JavaScript can't read the cookie → blocks XSS token theft |
| `Secure` | Only sent over HTTPS |
| `SameSite=Lax/Strict` | Browser won't send it on cross-site requests → blocks most CSRF |

**Why it wasn't enough**

- **Stateful.** Every request hits session storage. With many servers you need sticky sessions or a shared Redis.
- **CSRF.** Since the browser sends cookies *automatically*, another site can trigger authenticated requests. Needs CSRF tokens / `SameSite`.
- **Doesn't work well for non-browser clients** — mobile apps and APIs don't want a cookie jar.
- **The real blocker:** it can't answer *"let App X read my Gmail contacts."* The only way to do that back then was to give App X your Gmail password. That practice had a name: **the password anti-pattern.**

---

## 3. JWT (2015, RFC 7519)

Not an auth protocol — a **token format**. It answers: *can the token carry its own proof, so nobody has to look it up?*

**Structure — three base64url parts joined by dots**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiI0MiIsImV4cCI6MTc...  . SflKxwRJSMeKKF2QT4
└────────── HEADER ──────────────────┘ └────────── PAYLOAD ─────────┘ └──── SIGNATURE ────┘
   { "alg": "HS256", "typ": "JWT" }      { "sub": "42",                HMAC-SHA256(
                                           "role": "admin",             b64(header) + "." + b64(payload),
                                           "exp": 1789...,              secret
                                           "iat": 1789... }           )
```

**Critical:** the payload is *encoded*, **not encrypted**. Anyone can paste a JWT into jwt.io and read it. The signature only guarantees it *wasn't tampered with*. **Never put secrets in a JWT.**

**Standard claims:** `iss` (issuer), `sub` (subject/user), `aud` (audience), `exp` (expiry), `iat` (issued at), `nbf` (not before), `jti` (token ID).

**The trade it makes**

```
Session:  cookie → [lookup in Redis] → user            stateful, revocable, extra hop
JWT:      token  → [verify signature] → user           stateless, no lookup, hard to revoke
```

The server only needs the key. No database hit. That's why it fits microservices, mobile apps, and horizontally scaled backends.

**Why it isn't a silver bullet**

- **Revocation is the hard part.** A valid JWT works until `exp`, even if you ban the user. Workarounds: short-lived access tokens (5–15 min) + a refresh token you *can* revoke server-side, or a denylist (which... reintroduces state).
- **Where do you store it in a browser?** `localStorage` → readable by XSS. Cookie → back to CSRF. There's no clean answer; a `HttpOnly` cookie + `SameSite` is usually the least bad.
- **`alg: none` attack.** Early libraries accepted a token whose header said "no signature". Always pin the expected algorithm server-side.
- **Algorithm confusion.** Attacker flips `RS256` → `HS256` and signs with the *public* key as the HMAC secret. Same fix: never trust `alg` from the token.
- **Bloat.** A fat JWT rides on every request header.

**HS256 vs RS256**

- `HS256` — one shared secret, signs and verifies. Fine when one service does both.
- `RS256` — private key signs, public key verifies. Use when many services verify tokens they didn't issue (that's what JWKS endpoints are for).

---

## 4. The Two-Token Strategy (Access & Refresh Tokens)

When using stateless tokens like JWTs, **revocation is the hard part**. A valid JWT works until its `exp` time, even if you ban the user mid-session. There's no server storage to delete!

To mitigate this without completely breaking the stateless benefits, the industry uses a **two-token setup**:

```text
Access token   — (e.g. JWT), short lived (e.g., 15 mins), sent on every request, NOT revocable.
Refresh token  — Random opaque string, long lived (e.g., weeks), stored in the DB, ONLY used to get new access tokens, IS revocable (delete the row).
```

### The Workflow

1. **Login:** Server verifies credentials, saves a root refresh token in the DB, and sends both tokens to the client.
2. **Normal usage:** Client sends purely the access token on API requests. Fast verification, zero DB lookups.
3. **Expiration:** After ~15 minutes, the access token expires. The server starts returning `401 Unauthorized`.
4. **Refreshing:** Client catches the 401 and silently sends the refresh token to a `/refresh` endpoint.
5. **Validation:** Server looks up the refresh token in the DB. If it's valid and the user isn't banned, it replies with a brand new access token.
6. **Revocation (The Fix):** To ban a user or "log out everywhere," just delete their refresh token from the DB. When their current access token inevitably expires, they can't get a new one.

Worst case, a banned user has ~15 more minutes. That's the trade-off everyone accepts to keep everyday requests perfectly stateless.

---

## 5. OAuth 1.0 → 1.0a (2007–2010)

Born at Twitter (2006–07) precisely to kill the password anti-pattern.

**The core idea: delegated authorization.** Three parties instead of two.

```
   You (Resource Owner)
        │  "yes, allow it"
        ▼
   Twitter (Provider) ──── issues a limited token ───▶ PrintMyTweets.com (Consumer)
        ▲                                                       │
        └────────── token used instead of your password ─────────┘
```

The third-party app gets a **token** that is:
- scoped (read tweets, not delete account)
- revocable independently of your password
- not your password

**How it worked mechanically**

Every request had to be **cryptographically signed** by the client:

1. Collect all params + HTTP method + URL
2. Sort and normalize them into a canonical string
3. HMAC-SHA1 it with `consumer_secret & token_secret`
4. Attach `oauth_signature`, `oauth_nonce`, `oauth_timestamp`, ...

This meant no TLS was strictly required — the signature proved integrity. Clever, but:

**Why it wasn't enough**

- Signing was **notoriously painful** — one wrong character in the normalization and you got a useless `401`. Half of Stack Overflow in 2009 was OAuth 1 signature debugging.
- Every client needed a crypto library.
- Didn't fit mobile apps, TVs, CLI tools, or JS in the browser (you can't keep a `consumer_secret` secret in a public client).
- **1.0a exists because 1.0 had a session-fixation flaw** (2009) — the fix added `oauth_verifier` to bind the authorization to the request that started it.

---

## 6. OAuth 2.0 (2012, RFC 6749)

The pragmatic rewrite. The big decision: **stop signing requests, just mandate TLS and use bearer tokens.**

> *Bearer token* = whoever bears it, uses it. Like cash. No signature, no proof of possession — so TLS is non-negotiable.

**The roles**

| Role | Example |
|------|---------|
| Resource Owner | You |
| Client | The third-party app |
| Authorization Server | Google's login/consent screen |
| Resource Server | The Google API holding your data |

**Authorization Code flow (the main one)**

```
1. App redirects you to:
     https://auth.example.com/authorize
       ?response_type=code&client_id=abc&redirect_uri=...&scope=read:profile&state=xyz

2. You log in and click "Allow"

3. Redirect back:  https://app.com/callback?code=SplxlOB&state=xyz
                                            └── short-lived, single-use

4. App's BACKEND exchanges the code (server-to-server, with client_secret):
     POST /token  { code, client_id, client_secret, redirect_uri }
   →  { access_token, refresh_token, expires_in: 3600 }

5. App calls the API:
     Authorization: Bearer <access_token>
```

Why the two-step dance (code → token)? So the **access token never travels through the browser/URL bar**, where it would land in history, logs, and `Referer` headers.

**Key concepts**

- **Access token** — short-lived (minutes/hours), used on every API call
- **Refresh token** — long-lived, used only to get new access tokens
- **Scopes** — `read:profile`, `write:repo` — granular permission
- **`state`** — random value echoed back; protects against CSRF on the callback
- **PKCE** (RFC 7636, 2015) — replaces `client_secret` for mobile/SPA public clients; now recommended for *all* clients

**Grant types**

| Grant | Use for |
|-------|---------|
| Authorization Code (+ PKCE) | Web apps, SPAs, mobile — **the default** |
| Client Credentials | Machine-to-machine, no user involved |
| Device Code | TVs, CLIs, anything without a browser |
| ~~Implicit~~ | Deprecated — token in URL fragment, leaky |
| ~~Password (ROPC)~~ | Deprecated — reintroduces the password anti-pattern |

**Why it wasn't the end of the story**

- It's a **framework, not a protocol**. Two "OAuth 2 compliant" servers can be incompatible. Eran Hammer, the lead editor, [resigned and called it "the road to hell"](https://gist.github.com/nunoarruda/f0a8f6ad2ab5c26982c7).
- It says **nothing about authentication.** An access token tells you *the app may call this API* — not *who the user is*. Everyone bolted on their own "get user info" endpoint anyway.
  - → **OpenID Connect (2014)** fixed exactly this — see section 7 below.
- The token itself was still usually opaque → the resource server had to call back to the auth server (*token introspection*) on every request. Stateful again.

---

## 7. OpenID Connect (2014)

The missing half of OAuth 2. OAuth 2 answers *"may this app call this API?"* — it never answers *"who is sitting at the keyboard?"*. Apps hacked around that by calling some vendor-specific `/me` endpoint and treating "the access token worked" as proof of login. That isn't authentication, and it's exploitable (any token stolen from any other app for the same API also "works").

OIDC is **a thin, standard authN layer bolted on top of OAuth 2** — same authorize/token endpoints, same redirect dance. It adds one token and one contract.

**What actually changes in the flow: one scope.**

```
GET https://accounts.google.com/o/oauth2/v2/auth
  ?response_type=code
  &client_id=abc
  &redirect_uri=https://app.com/callback
  &scope=openid email profile     ← "openid" = this is OIDC, also give me an id_token
  &state=xyz                      ← CSRF protection on the callback (OAuth 2)
  &nonce=n-0S6_WzA2Mj             ← binds the id_token to THIS login request (OIDC)
```

The token exchange then returns one extra field:

```json
{
  "access_token":  "ya29.a0Af...",    // for calling APIs    (OAuth 2, opaque to you)
  "id_token":      "eyJhbGciOiJSUzI...", // for knowing WHO  (OIDC, a JWT you read)
  "refresh_token": "1//0gLm...",
  "expires_in": 3600
}
```

### The `id_token`

Always a **JWT**, normally `RS256`-signed, and it is addressed **to your app** — not to an API. Decoded:

```json
{
  "iss": "https://accounts.google.com",      // who issued it
  "sub": "110248495921238473",               // stable unique user id ← key YOUR users on this
  "aud": "abc.apps.googleusercontent.com",   // YOUR client_id
  "exp": 1789003600, "iat": 1789000000,
  "nonce": "n-0S6_WzA2Mj",
  "email": "harsh@example.com",
  "email_verified": true,
  "name": "Harsh Mishra",
  "picture": "https://lh3.googleusercontent.com/..."
}
```

**Validate it — don't just decode it.** The whole security of "Sign in with X" is these five checks:

| Check | Why |
|-------|-----|
| Signature, against the provider's **JWKS** (`jwks_uri`) | Otherwise anyone can forge an identity |
| `iss` == the expected issuer | Stops a different provider vouching for the user |
| `aud` == your `client_id` | Stops a token minted for *another app* logging in here |
| `exp` / `iat` still fresh | Replay of an old login |
| `nonce` == the one you sent | Ties this token to the login you actually started |

### Two tokens, two jobs

| | `id_token` | `access_token` |
|---|---|---|
| Audience | **your app** | **the resource server (API)** |
| Format | always a JWT | opaque or JWT — none of your business |
| Answers | *who is the user?* | *what is this app allowed to do?* |
| You should | verify it, then create a session | forward it as `Authorization: Bearer ...` |
| You should **not** | send it to an API | treat it as proof of identity |

That last row is the classic OIDC bug in both directions.

### What else OIDC standardizes

- **Discovery** — `GET /.well-known/openid-configuration` returns every endpoint, the supported scopes, and the `jwks_uri`. This is why an OIDC client library can talk to Google, Auth0, Keycloak, and Okta with only an issuer URL as config.
- **`/userinfo`** — call it with the access token to get claims that weren't crammed into the `id_token`.
- **Standard scopes** — `openid` (required), `profile`, `email`, `address`, `phone`.
- **Authentication controls** — `prompt=login` (force a fresh login), `max_age`, `acr_values` (demand MFA), `login_hint`.
- **Logout** — RP-initiated logout at `end_session_endpoint`, plus front/back-channel logout so sibling apps get told.

### After a successful login

An `id_token` is a **login receipt, not a session**. The normal move is: validate it once, then issue your *own* session cookie or your own tokens — see sections 2 and 4. Don't keep re-sending the provider's `id_token` on every request.

### Gotchas

- **Don't key accounts on `email`.** People change emails and providers reuse them. `sub` is the stable id — but it's only unique *per issuer*, and some providers issue *pairwise* `sub`s (a different value per client), so store `(iss, sub)`.
- **Check `email_verified`.** Without it, a sloppy provider that lets anyone claim an unverified email hands you an account takeover.
- **`state` ≠ `nonce`.** `state` protects the callback from CSRF; `nonce` protects the `id_token` from replay. You need both.
- **"Sign in with Google / Apple / Microsoft" is OIDC.** "Login with GitHub / Facebook" is *raw OAuth 2* plus a bespoke `/me` call — same UX, no `id_token`, so you must validate identity differently.

---

## Where it stands now

- **Sessions never died.** For a plain server-rendered web app, a `HttpOnly` cookie session is still the simplest and safest default. "JWT for everything" is a common over-engineering trap.
- **OAuth 2.1** (draft) folds in the accumulated best practice: PKCE mandatory, implicit and password grants removed.
- **Passkeys / WebAuthn** (W3C Rec, 2019; mainstream ~2023) are the real next step — public-key auth bound to your device, no shared secret to phish or breach. The end of passwords, not just of sending them.
- **DPoP** (RFC 9449, 2023) adds proof-of-possession to bearer tokens — an echo of what OAuth 1's signatures were doing, now done properly.

## Gotchas

- Base64 ≠ encryption. Applies to Basic Auth *and* JWT payloads.
- OAuth ≠ login. If you want identity, you want **OpenID Connect**.
- A JWT is not "more secure" than a session — it's a different trade (statelessness for revocability).
- `alg` in a JWT header is attacker-controlled input. Pin it.
- Cookies are sent automatically → that convenience *is* CSRF.
- `id_token` is for your app, `access_token` is for the API. Swapping them is the classic OIDC bug.
- Access tokens should be short-lived. Refresh tokens are the revocable part.
