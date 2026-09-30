# Authentication and Authorization

## Brief

Two different questions, often confused:

- **Authentication (authn): "Who are you?"** Proving identity: password, fingerprint, Google login.
- **Authorization (authz): "What are you allowed to do?"** Checking permissions: can this user delete this post?

Authentication always comes first. You can't decide what someone may do until you know who they are.

The main ways to remember a logged-in user:

- **Session + cookie**: the server keeps a record; the browser holds a random ID.
- **JWT**: the server hands out a signed token that carries the user's info.
- **OAuth 2.0**: letting an app access your data on another service *without* giving it your password.
- **OpenID Connect / SSO**: "Log in with Google", built on top of OAuth.

---

## The analogy that makes it click

**Checking into a hotel.**

- At the front desk you show your **passport**. That's **authentication**: proving who you are.
- They give you a **key card**. That's your **session/token**: you don't show your passport at every door.
- The key card opens **your room and the gym, but not other rooms or the staff area**. That's **authorization**.

Two ways the key card can work:

- The card holds just a number, and the door checks the hotel's central system: **session-based**.
- The card itself is encoded with "Room 304, valid until Friday", signed by the hotel so it can't be forged: **JWT**.

And **OAuth** is like giving a **valet key** to a parking attendant: it can drive the car but can't open the trunk. Limited access, without handing over your master key.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Credentials** | What you prove identity with: password, OTP, biometric |
| **Session** | Server-side record that "this ID belongs to logged-in user 42" |
| **Cookie** | Small data the browser stores and auto-sends to the same site |
| **Token** | A string the client sends to prove it's authenticated |
| **JWT** | JSON Web Token: a signed, self-contained token |
| **Scope** | What an OAuth token is allowed to do (e.g. `read:email`) |
| **MFA** | Multi-factor auth: password + something else (OTP, phone) |
| **RBAC** | Role-Based Access Control: permissions by role (admin, editor, viewer) |

---

## Authn vs authz

| | Authentication | Authorization |
|---|---|---|
| **Question** | Who are you? | What can you do? |
| **Happens** | First | After authn |
| **Input** | Credentials | Identity + roles/permissions + the resource |
| **Fails with** | `401 Unauthorized` | `403 Forbidden` |
| **Example** | Logging in with email + password | Only the post's author can edit it |

(Yes, `401` is named "Unauthorized" but actually means "unauthenticated". Historical naming mistake.)

Common authorization models:

- **RBAC:** users get roles, roles get permissions. `admin` can delete anything.
- **Ownership checks:** you can edit only resources where `owner_id == your_id`.
- **ABAC (attribute-based):** rules on attributes, e.g. "managers can view reports from their own department during work hours".

**Always enforce authz on the server.** Hiding a "Delete" button in the UI is not security.

---

## Session + cookie auth (the classic way)

```
  BROWSER                            SERVER                     SESSION STORE
     │                                  │                         (Redis / DB)
  1. │── POST /login {email, pwd} ─────►│                               │
     │                                  │── check password hash         │
     │                                  │── create session ────────────►│ sess_9f2c → user 42
  2. │◄── Set-Cookie: sid=sess_9f2c ────│                               │
     │                                  │                               │
  3. │── GET /orders                    │                               │
     │   Cookie: sid=sess_9f2c ────────►│── lookup sess_9f2c ──────────►│
     │                                  │◄── user 42 ───────────────────│
  4. │◄── 200 OK (user 42's orders) ────│                               │
```

- The cookie holds only a **random, meaningless ID**. All real data is on the server.
- The browser sends the cookie **automatically** on every request to that site.
- **Logout** = delete the session from the store. Instantly effective everywhere.

Secure your session cookie:

```http
Set-Cookie: sid=sess_9f2c; HttpOnly; Secure; SameSite=Lax; Max-Age=86400
```

| Flag | Protects against |
|---|---|
| `HttpOnly` | JavaScript can't read it, which limits theft via XSS |
| `Secure` | Only sent over HTTPS |
| `SameSite` | Not sent on most cross-site requests, which limits CSRF |

**Pros:** easy revocation, tiny cookie, server has full control.
**Cons:** a lookup on every request; needs a shared session store to stay stateless across many servers; cookies are awkward for mobile apps and cross-domain APIs.

---

## JWT (JSON Web Token)

A JWT is a token that **carries its own data** and is **signed**, so the server can trust it without looking anything up.

### Structure

Three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcwMDAwMDAwMH0 . SflKxwRJSMeKKF2QT4fw...
└────── HEADER ─────┘   └──────────────────────── PAYLOAD ──────────────────────┘   └──── SIGNATURE ────┘
```

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload ("claims")
{ "sub": "42", "role": "admin", "iat": 1699996400, "exp": 1700000000 }
```

| Claim | Meaning |
|---|---|
| `sub` | Subject: the user ID |
| `iat` | Issued at (timestamp) |
| `exp` | Expires at (timestamp) |
| `iss` / `aud` | Who issued it / who it's meant for |

**The payload is encoded, NOT encrypted.** Anyone can decode and read it. Never put passwords or secrets in it.

### Signing

```
signature = sign( header + "." + payload, key )
```

- **HS256 (symmetric):** one shared secret signs and verifies. Simple, but every verifier holds the secret and could also create tokens.
- **RS256 / ES256 (asymmetric):** a **private key** signs (only the auth server has it); a **public key** verifies (any service can have it). Better for microservices.

If anyone changes even one character of the payload (say `"role": "user"` → `"admin"`), the signature no longer matches and the server rejects it.

### How it's used

```
1. POST /login  ──►  server verifies password, returns JWT
2. Client stores it, sends on each request:
      Authorization: Bearer eyJhbGciOi...
3. Any server verifies signature + exp  ──►  knows it's user 42. No DB lookup. ✅
```

### Pros and cons

| ✅ Pros | ❌ Cons |
|---|---|
| Stateless: no session store lookup | **Can't easily revoke** before expiry |
| Any service can verify it independently | Bigger than a session ID (sent on every request) |
| Works well across domains, mobile, microservices | Data inside can go stale (role changed, token still says admin) |
| | Easy to misuse (weak secrets, accepting `alg: none`, storing in localStorage exposed to XSS) |

### The revocation problem

With sessions, logout = delete the row. With JWT, the token stays valid until `exp` because **the server doesn't keep a list of tokens**. A stolen token keeps working.

Common fixes:

| Approach | How | Trade-off |
|---|---|---|
| **Short-lived access tokens** | `exp` = 5-15 minutes | Stolen tokens die fast |
| **+ Refresh tokens** | Long-lived token (stored server-side, revocable) used only to get new access tokens | Adds a refresh flow, but logout = revoke the refresh token |
| **Denylist** | Store revoked token IDs (`jti`) in Redis until they expire | Brings back a lookup, partially losing statelessness |
| **Token version** | Store `token_version` per user; bump it to invalidate all their tokens | Needs a lookup per request (can be cached) |

The standard answer: **short-lived access token + revocable refresh token.**

---

## OAuth 2.0

**OAuth 2.0 is about delegated authorization**: letting App X access *your* data on Service Y (e.g. a photo printing app reading your Google Photos) **without giving App X your Google password**.

### The four roles

| Role | Who | Example |
|---|---|---|
| **Resource owner** | You, the user | You |
| **Client** | The app wanting access | PrintMyPhotos app |
| **Authorization server** | Logs you in and issues tokens | Google's login/consent server |
| **Resource server** | Holds the data, accepts tokens | Google Photos API |

### Authorization Code flow (high level)

This is the standard flow for web and mobile apps.

```
 USER           CLIENT APP               AUTH SERVER (Google)       RESOURCE SERVER
  │                 │                          │                          │
1.│── "Connect Google Photos" ─►│              │                          │
  │                 │                          │                          │
2.│◄── redirect to Google login + consent ─────│                          │
  │    (?client_id=..&scope=photos.read&redirect_uri=..)                  │
  │                                            │                          │
3.│── logs in, clicks "Allow" ────────────────►│                          │
  │                                            │                          │
4.│◄── redirect back to app with ?code=XYZ ────│                          │
  │                 │                          │                          │
5.│                 │── code + client secret ─►│  (server-to-server)      │
  │                 │◄── access token ─────────│  (+ refresh token)       │
  │                 │                          │                          │
6.│                 │── GET /photos  Bearer <access token> ──────────────►│
  │                 │◄── your photos ─────────────────────────────────────│
```

Why the extra "code" step instead of handing over the token directly? The code travels through the browser (visible, less safe). The **real token is exchanged server-to-server**, with the client's secret, so it's never exposed in the URL.

For mobile apps and SPAs (which can't keep a secret), the same flow is used with **PKCE**: the app generates a one-time secret per login so a stolen code is useless to anyone else. PKCE is now recommended for all clients.

**Scopes** limit what the token can do: `photos.read` can't delete photos. That's the valet key.

---

## SSO and OpenID Connect (briefly)

**OAuth 2.0 alone is about access, not identity.** It says "this token can read photos", not "this is Asha".

**OpenID Connect (OIDC)** is a thin identity layer on top of OAuth 2.0. Alongside the access token, you get an **ID token** (a JWT) saying who the user is (`sub`, `email`, `name`). That's what powers **"Log in with Google / Apple / GitHub"**.

**SSO (Single Sign-On)** = log in once, get into many apps. Log into your company's identity provider (Okta, Google Workspace, Azure AD) and you're signed into Slack, Jira, and email without separate passwords. Built on **OIDC** (modern) or **SAML** (older, XML-based, common in enterprises).

```
             ┌─────────────────────┐
             │ Identity Provider   │   log in once
             │ (Okta / Google)     │ ◄───────────── User
             └──────────┬──────────┘
          trusted by    │
      ┌─────────────────┼─────────────────┐
      ▼                 ▼                 ▼
   Slack              Jira              Email       ← no separate logins
```

---

## Comparison

| | Session + cookie | JWT | OAuth 2.0 | OIDC / SSO |
|---|---|---|---|---|
| **Solves** | Remember a logged-in user | Remember a logged-in user, statelessly | Third-party access to a user's data | Identity via a trusted provider, one login for many apps |
| **Authn or authz?** | Authn (session) | Authn (carries identity; can carry roles) | **Authz** (delegated access) | **Authn** |
| **Server state** | Session store | None (unless denylist) | Auth server tracks grants/refresh tokens | Identity provider holds sessions |
| **Revocation** | ✅ Instant | ❌ Hard (use short expiry + refresh) | ✅ Revoke refresh token | ✅ At the provider |
| **Best for** | Classic web apps, same domain | APIs, mobile, microservices | "Let App X access my Google Drive" | "Log in with Google", enterprise SSO |

These aren't strictly competitors. A real system often combines them: users sign in via **OIDC**, your backend then issues its own **session cookie or short-lived JWT**, and internal services verify that JWT.

---

## Gotchas

- **Never store plain passwords.** Store a slow, salted hash (bcrypt, scrypt, Argon2).
- **JWT payload is readable by anyone.** Signed ≠ encrypted.
- **Always validate `exp`, `iss`, `aud`, and the algorithm** on JWTs. Reject `alg: none`.
- **Tokens in localStorage are exposed to XSS.** `HttpOnly` cookies are safer for browsers (then defend against CSRF).
- **OAuth is not login.** Use OIDC when you need to know *who* the user is.
- **Always HTTPS.** Any token or cookie over plain HTTP can be stolen.
- **Check authorization on every request**, for every resource, on the server. "Is this user allowed to see order 123?" is the most commonly forgotten check (it's called IDOR, Insecure Direct Object Reference).
