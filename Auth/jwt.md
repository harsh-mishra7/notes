# JWT in Simple Terms

## TL;DR

A JWT is a **note the server writes about you, signs, and hands back to you to carry around**.

> "This is user 42, they're an admin, and this note stops being valid at 4:30pm."
> — signed, the server

You show that note on every request. The server checks its own signature, sees it's untampered, and believes it — **without looking anything up in a database**.

That's the whole idea. Everything else is detail.

---

## The one analogy that makes it click

| | How it works | Web equivalent |
|---|---|---|
| **Coat check ticket** | The ticket says `#42`. Meaningless by itself. The attendant looks up `#42` in their book to find your coat. | **Session cookie** — server keeps the truth |
| **Festival wristband** | The band has your name and date printed on it, with a seal that can't be faked. Staff just *read* it. No book. | **JWT** — the token carries the truth |

Lose the book (restart the server, add a second server) and coat checks break.
Wristbands keep working, because every guard can verify a seal on their own.

The catch, which is the same in both worlds: **you can't un-print a wristband.** Once it's on someone's wrist, it works until the date on it passes. More on that below.

---

## What a JWT actually looks like

Three chunks of base64url text, joined by dots:

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiJ9 . dBjftJeZ4CVP
└───── HEADER ──────┘ └────────────── PAYLOAD ─────────────┘ └─ SIGNATURE ─┘
```

Decode the first two and you get plain JSON:

```jsonc
// HEADER — how this token was signed
{ "alg": "HS256", "typ": "JWT" }

// PAYLOAD — the claims (the actual message)
{ "sub": "42", "role": "admin", "iat": 1789000000, "exp": 1789001800 }

// SIGNATURE — proof the server wrote it (raw bytes, not readable)
```

**The part everyone gets wrong:** base64 is *encoding*, not encryption. Paste any JWT into jwt.io and read it — no secret needed. A JWT is a **postcard with a wax seal**, not a sealed envelope.

→ Never put a password, API key, or anything private in a payload.

---

## How the signature works

The server computes:

```
signature = HMAC_SHA256( base64(header) + "." + base64(payload),  SECRET_KEY )
```

Then, on every incoming request, it recomputes that from the token it just received and compares.

**Why forging fails:**

```
Attacker edits the payload:   { "role": "user" }  →  { "role": "admin" }
Payload changes              →  the correct signature changes
Attacker can't compute the new one  →  they don't have SECRET_KEY
Server recomputes, compares, mismatch  →  401
```

So the signature buys you exactly one thing: **"this was issued by me and nobody edited it."**
It does *not* hide the contents, and it does *not* prove the person holding it is the person it describes — anyone who steals the token can use it (that's why they're called *bearer* tokens, and why HTTPS is mandatory).

---

## The full round trip

```
1. POST /login  { email, password }
        │
        ▼
2. Server checks the password, then builds and signs a token:
        { sub: 42, role: "admin", exp: now + 15min }  +  SECRET
        │
        ▼
3. Response:  { "access_token": "eyJhbGci..." }
        │
        ▼
4. Client stores it and sends it on every request:
        GET /api/orders
        Authorization: Bearer eyJhbGci...
        │
        ▼
5. Server: verify signature → check exp → read `sub`  →  done.
        No DB call. No session store. No Redis.
```

Step 5 is the entire payoff: **three servers behind a load balancer can all verify the same token, and none of them need to share anything but the key.**

---

## The standard claims

`sub`, `iat`, `exp` are the ones you'll actually use daily.

| Claim | Full name | Plain meaning |
|-------|-----------|---------------|
| `sub` | subject | **who this token is about** (usually the user ID) |
| `exp` | expiration | after this timestamp, reject it |
| `iat` | issued at | when it was created |
| `iss` | issuer | who made it (`https://auth.myapp.com`) |
| `aud` | audience | who it's *for* — reject tokens meant for another service |
| `nbf` | not before | don't accept it until this time |
| `jti` | JWT ID | unique ID, useful for denylists |

You can add your own (`role`, `plan`, `org_id`) — just remember every claim is **public** and **frozen until the token expires**. If someone gets demoted, their old token still says `admin`.

---

## HS256 vs RS256

| | HS256 | RS256 |
|---|---|---|
| Keys | One shared secret | Private key + public key |
| Signs | the secret | the **private** key |
| Verifies | the same secret | the **public** key |
| Analogy | A password both sides know | A wax seal — anyone can recognize it, only you can stamp it |
| Use when | One app signs and verifies its own tokens | Many services verify tokens they didn't issue |

Rule of thumb: **one service → HS256. Multiple services or third parties → RS256**, because with HS256 anyone who can *verify* can also *forge*.

---

## Where it gets awkward

**1. You can't take it back.**
Ban a user at 2:00 and their token still works until it expires. There's no "delete the session" — the server isn't holding anything to delete.

The standard fix is a **two-token setup**:

```text
Access token   — JWT, short lived (e.g., 15 mins), sent on every request, not revocable
Refresh token  — random string, long lived (e.g., weeks), stored in the DB. Only used
                 to get a new access token, and revocable (delete the row)
```

**How the two-token flow works:**
1. **Login:** Server verifies credentials, saves a root refresh token in the DB, and sends both tokens to the client.
2. **Normal usage:** Client sends purely the access token on API requests. Fast verification, zero DB lookups.
3. **Expiration:** After 15 minutes, the access token expires. The server starts returning `401 Unauthorized`.
4. **Refreshing:** Client catches the 401 and silently sends the refresh token to a `/refresh` endpoint.
5. **Validation:** Server looks up the refresh token in the DB. If it's valid and the user isn't banned, it replies with a brand new access token.
6. **Revocation (The Fix):** To ban a user or "log out everywhere," just delete their refresh token from the DB. When their current access token inevitably expires, they can't get a new one.

Worst case, a banned user has ~15 more minutes. That's the trade-off everyone accepts to keep everyday requests perfectly stateless.

**2. Where do you put it in a browser?**

| Storage | Risk |
|---------|------|
| `localStorage` | Any XSS on your page reads it and walks away with it |
| Normal cookie | Sent automatically → CSRF |
| **`HttpOnly` + `Secure` + `SameSite` cookie** | JS can't read it, CSRF mostly blocked → **usually the least bad** |

There is no perfect answer here. Pick the least bad one and don't pretend it's solved.

**3. It rides along on every request.** A payload stuffed with permissions can push your headers into kilobytes. Keep it small — ID, role, expiry.

---

## When *not* to reach for a JWT

- A plain server-rendered app with one backend → **a session cookie is simpler and safer.** "JWT for everything" is the single most common over-engineering trap in auth.
- You need instant logout / instant ban → sessions, or very short expiry.
- You're tempted to make a long-lived JWT (days/weeks) → that's a session with no off switch.

JWTs earn their keep when the token has to be verified by **something that can't reach your session store**: many services, other teams' services, mobile clients, serverless functions.

## Gotchas

- Base64 is not encryption. Anyone can read the payload. Never put secrets in it.
- The signature proves *who wrote it*, not *who's holding it*. Stolen token = full access. Always HTTPS.
- `alg` in the header is attacker-controlled input. Pin the algorithm server-side and never trust the token's own word for it (`alg: none` and RS256→HS256 confusion are both real attacks).
- Always check `exp` yourself if your library doesn't — a signature can be perfectly valid on a token from last year.
- Claims are a snapshot from login time, not live data. Permission changes don't reach existing tokens.
- A JWT is not "more secure" than a session. It's a different trade: statelessness in exchange for revocability.
