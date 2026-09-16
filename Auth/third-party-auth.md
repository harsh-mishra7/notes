# Third-Party Authentication in a Real App

> How "Sign in with Google" and services like Clerk actually work once you're the one wiring them up.
> Prerequisite: [auth.md](auth.md) §6 (OAuth 2) and §7 (OIDC) — this note is the practical layer on top.

## TL;DR

Logging a user in is **two separate jobs**, and every auth decision is really about who does which one:

| Job | When | Question |
|-----|------|----------|
| **Verification** | once, at login | *Is this really harsh@example.com?* |
| **Session** | every request after | *Is this request still from that person?* |

- **Rolling your own** = you do both.
- **"Sign in with Google"** = Google does job 1. **You still do job 2.**
- **Clerk / Auth0 / Supabase Auth** = they do both, and they store your users too.

The single most common misconception: *"if I use Google/Clerk, I don't need a users table."* You do. Always. More on that in §3.

---

## The three models

| | Verifies identity | Holds the session | Stores user rows | You build |
|---|---|---|---|---|
| **A. Roll your own** | you (bcrypt + DB) | you (cookie/JWT) | you | everything |
| **B. Direct OIDC** — talk to Google yourself | Google | **you** | you | the session + the long tail |
| **C. Managed provider** — Clerk, Auth0, Supabase | Clerk (+ Google behind it) | Clerk | Clerk (you mirror it) | almost nothing |

Model B is the one people mean by "third-party auth". Model C is what people actually ship when they don't want to think about auth again.

---

## 1. Model B — your app talks to Google directly

This is plain OIDC. Nothing between you and Google.

### One-time setup (before any code runs)

In the Google Cloud Console you register *your app* as an OAuth client. This is OAuth's other authentication — not "who is the user", but **"who is this app claiming to be Google's client?"**

You get back:

```
GOOGLE_CLIENT_ID      = 1234-abc.apps.googleusercontent.com   → public, ships to the browser
GOOGLE_CLIENT_SECRET  = GOCSPX-xxxxx                          → server only, NEVER in frontend code
```

and you register your exact **redirect URIs**:

```
http://localhost:3000/api/auth/callback/google      ← dev
https://quickclinic.com/api/auth/callback/google    ← prod
```

Google will refuse to redirect anywhere not on that list. That's what stops an attacker from starting a login with your `client_id` and having the `code` delivered to their own server.

### The runtime flow

```
 Browser                      Your backend                     Google
    │                              │                              │
    │  click "Sign in with Google" │                              │
    │─────────────────────────────▶│                              │
    │   302 to accounts.google.com │                              │
    │◀─────────────────────────────│                              │
    │      ?client_id&redirect_uri&scope=openid email profile      │
    │       &state=<random>&nonce=<random>                         │
    │─────────────────────────────────────────────────────────────▶│
    │                                                              │
    │            [ user logs in + consent screen ]                  │
    │                                                              │
    │◀─────────── 302 /callback?code=SplxlOB&state=<random> ────────│
    │─────────────────────────────▶│                              │
    │                              │  POST /token                 │
    │                              │  {code, client_id, secret}   │  ← server-to-server,
    │                              │─────────────────────────────▶│    over the internet,
    │                              │  {id_token, access_token}    │    never via the browser
    │                              │◀─────────────────────────────│
    │                              │                              │
    │                     [ verify id_token: signature/iss/aud/exp/nonce ]
    │                     [ upsert user WHERE (iss, sub) ]
    │                     [ mint YOUR session ]
    │                              │                              │
    │◀── 302 /dashboard  +  Set-Cookie: token=...; HttpOnly ──────│
    │                              │                              │
    │  every request after this: your cookie. Google is gone.      │
```

Note where the line stops: **Google is involved exactly once.** After the callback, your app is back to the same session mechanics it would have used with a password login.

### Code sketch (Next.js route handlers + Prisma)

```ts
// app/api/auth/google/route.ts  — step 1: send them to Google
export async function GET() {
  const state = crypto.randomUUID();
  const nonce = crypto.randomUUID();

  // stash both in short-lived HttpOnly cookies so the callback can compare
  cookies().set('oauth_state', state, { httpOnly: true, maxAge: 600, secure: true });
  cookies().set('oauth_nonce', nonce, { httpOnly: true, maxAge: 600, secure: true });

  const url = new URL('https://accounts.google.com/o/oauth2/v2/auth');
  url.searchParams.set('client_id', process.env.GOOGLE_CLIENT_ID!);
  url.searchParams.set('redirect_uri', `${process.env.APP_URL}/api/auth/callback/google`);
  url.searchParams.set('response_type', 'code');
  url.searchParams.set('scope', 'openid email profile');
  url.searchParams.set('state', state);
  url.searchParams.set('nonce', nonce);

  return NextResponse.redirect(url);
}
```

```ts
// app/api/auth/callback/google/route.ts  — step 2: they came back
export async function GET(req: NextRequest) {
  const code  = req.nextUrl.searchParams.get('code')!;
  const state = req.nextUrl.searchParams.get('state')!;

  if (state !== cookies().get('oauth_state')?.value) throw new Error('CSRF');

  // exchange the code — this is the only place client_secret is used
  const res = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      code,
      client_id: process.env.GOOGLE_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CLIENT_SECRET!,
      redirect_uri: `${process.env.APP_URL}/api/auth/callback/google`,
      grant_type: 'authorization_code',
    }),
  });
  const { id_token } = await res.json();

  // VERIFY — signature against Google's JWKS, plus iss / aud / exp / nonce.
  // Use a library (google-auth-library, jose). Never jwt.decode() and trust it.
  const claims = await verifyGoogleIdToken(id_token, {
    audience: process.env.GOOGLE_CLIENT_ID!,
    nonce: cookies().get('oauth_nonce')?.value,
  });

  if (!claims.email_verified) throw new Error('unverified email');

  const user = await prisma.user.upsert({
    where:  { provider_providerId: { provider: 'google', providerId: claims.sub } },
    update: { name: claims.name, profileImageUrl: claims.picture },
    create: {
      provider: 'google', providerId: claims.sub,
      email: claims.email, name: claims.name, emailVerified: true,
      role: 'PATIENT',
    },
  });

  // from here it's YOUR auth, identical to a password login
  const token = await signToken({ sub: user.id, role: user.role });
  cookies().set('token', token, { httpOnly: true, secure: true, sameSite: 'lax' });

  return NextResponse.redirect(new URL('/dashboard', req.url));
}
```

The last four lines are the point of the whole note: **after verification, third-party auth collapses back into the session model you already have.** Your middleware, your roles, your expiry — unchanged.

### What you still own in Model B

Google removed exactly one problem (password storage and verification). These remain yours:

- **Account linking** — same human signs in with Google *and* with email+password. Same email, two rows? Merge them? Require a confirmation step? (This is the genuinely hard one — see §4.)
- **Multiple providers** — add Apple and GitHub and you need a `accounts` table, not `provider` columns on `user`.
- **Roles, permissions, onboarding** — Google has no idea your app has doctors and patients.
- **Session lifecycle** — expiry, refresh, "log out everywhere", ban a user.
- **Everything for users who don't have Google.**

---

## 2. Model C — a managed auth provider (Clerk)

### What Clerk actually is

Three products in a trench coat:

1. **A hosted OIDC provider** — Clerk *is* the authorization server. It talks to Google, Apple, GitHub on your behalf, plus email/password, magic links, OTP, passkeys.
2. **Prebuilt UI** — `<SignIn />`, `<UserButton />`, the whole "manage your account" screen you'd otherwise spend three sprints on.
3. **A user database** — Clerk stores the user records, emails, MFA factors, organizations.

So Google is still doing job 1 behind the scenes, but **you never see Google**. You only ever talk to Clerk.

```
Without Clerk:   Your app ──OIDC──▶ Google
                          ──OIDC──▶ Apple          ← you implement each one
                          ──OIDC──▶ GitHub

With Clerk:      Your app ──────▶ Clerk ──OIDC──▶ Google / Apple / GitHub / email+OTP
                                    ▲
                            one integration, N providers
```

### The flow

```
 Browser                        Clerk                      Your backend
    │                             │                              │
    │  <SignIn /> component       │                              │
    │────────────────────────────▶│                              │
    │    [ Clerk runs the whole Google/OIDC dance itself ]        │
    │                             │                              │
    │◀── Set-Cookie: __session=<short-lived JWT>  ────────────────│
    │        (on YOUR domain, via clerk.yourapp.com CNAME)        │
    │                             │                              │
    │  GET /api/appointments  (cookie rides along)                │
    │────────────────────────────────────────────────────────────▶│
    │                             │       verify JWT signature    │
    │                             │◀─── fetch JWKS (cached) ──────│
    │                             │                              │
    │                        [ no network call to Clerk per request ]
    │◀───────────────────── 200 with data ────────────────────────│
```

### The bit that confuses everyone: whose cookie is it?

Clerk sets the session cookie, but on **your** domain (you point `clerk.yourapp.com` at them via CNAME in production). The cookie holds a **very short-lived JWT** — roughly a minute by default — and Clerk's frontend SDK silently refreshes it in the background using a longer-lived handshake.

That design is deliberate, and it's your `auth.md` §4 two-token strategy taken to the extreme:

- Token lifetime ~60s means **revocation is effectively instant** — ban a user, and their next refresh fails.
- Your backend can verify **networklessly** (signature check against cached JWKS), so no per-request call to Clerk, no dependency on their latency.

### How your backend verifies

```ts
// middleware.ts — Clerk's version of your current verifyToken() middleware
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isProtected = createRouteMatcher(['/doctor(.*)', '/patient(.*)', '/admin(.*)']);

export default clerkMiddleware(async (auth, req) => {
  if (isProtected(req)) await auth.protect();
});
```

```ts
// in a route handler / server component
import { auth } from '@clerk/nextjs/server';

const { userId, sessionClaims } = await auth();   // userId = "user_2abc..." (the JWT's `sub`)
if (!userId) return new Response('Unauthorized', { status: 401 });

// YOUR data is still keyed by YOUR user row
const me = await prisma.user.findUnique({ where: { clerkId: userId } });
```

Under the hood that's just: read cookie → verify JWT against Clerk's JWKS → hand you the claims. Same mechanics as your hand-rolled `verifyToken`, with the signing key owned by Clerk.

> API names differ across Clerk major versions (`authMiddleware` → `clerkMiddleware`, sync `auth()` → async). Check their docs for the version you install.

### Keeping your database in sync

Clerk holds the identity; **you still hold the domain data** — appointments, prescriptions, payments. Those need foreign keys to a row in *your* database. Two mechanisms, and you generally want both:

```
1. WEBHOOKS (push)         Clerk ──user.created──▶ POST /api/webhooks/clerk ──▶ INSERT users
                           Clerk ──user.updated──▶                          ──▶ UPDATE users
                           Clerk ──user.deleted──▶                          ──▶ soft-delete

2. LAZY UPSERT (pull)      first authed request for an unknown clerkId → create the row now
```

Webhooks are the happy path; the lazy upsert is the safety net for when a webhook is delayed, dropped, or fails during a deploy. Make both **idempotent** (upsert on `clerkId`) and **verify the webhook signature** (Clerk signs with Svix) — an unauthenticated `/api/webhooks/clerk` is a "create any user you like" endpoint.

### What you give up

- **Cost** — free tier, then per monthly-active-user. Fine at 1k users, a real line item at 500k.
- **Lock-in** — your users' credentials live in Clerk. Migrating out means an export + a forced password reset for password users (OAuth users are easier: same Google `sub`).
- **Their outage is your outage** — nobody logs in. (Existing sessions survive until their next refresh, which is ~60s away.)
- **Customisation ceiling** — prebuilt UI is a gift until the day you need a flow they don't support.
- **Data residency / compliance** — for a health app, "where do user records physically live" is a question you now have to answer about someone else's infrastructure.

---

## 3. You still need a users table

In every model. Non-negotiable.

```prisma
model User {
  id       String  @id @default(cuid())   // YOUR id — everything else FKs to this
  clerkId  String? @unique                // or: provider + providerId for direct OIDC
  email    String  @unique
  role     Role    @default(PATIENT)      // YOUR domain concept, not Clerk's
  // ...
  appointments Appointment[]
}
```

**Why not just use the Clerk/Google id everywhere as the FK?**

- You can't `JOIN` against a token claim, and you can't foreign-key to a value you don't store.
- Every list screen ("all appointments with patient names") becomes an N+1 of API calls to Clerk.
- Switching providers means rewriting every foreign key in the schema instead of one column.
- Provider ids are only unique *per provider* — add Apple login later and `sub` collisions become thinkable.

The pattern has a name: **shadow user** (or "mirror user"). The provider owns identity; you own a local row that identity points at.

---

## 4. Adding Google to an app that already has passwords

The migration people underestimate. Current state: `users.password` (bcrypt), login by email. Now you add Google, and a user who registered with `harsh@example.com` + password clicks "Sign in with Google" with the same address.

Three possible behaviours, in increasing order of correctness:

| Behaviour | Result |
|---|---|
| Blindly create a second row | Two accounts, split history. Support tickets forever. |
| Auto-link on matching email | Convenient — **but only safe if `email_verified` is true.** Otherwise a provider that lets anyone claim an unverified address hands over the existing account. |
| Link only after proving ownership | Ask them to sign in with the password once, then attach Google. Safest, one extra step. |

The schema change that makes this tractable — move identities out of `User` into their own table:

```prisma
model User {
  id       String    @id @default(cuid())
  email    String    @unique
  password String?                       // now nullable — Google users have none
  accounts Account[]
}

model Account {
  id         String @id @default(cuid())
  userId     String
  provider   String                      // "google" | "apple" | "credentials"
  providerId String                      // the id_token `sub`
  user       User   @relation(fields: [userId], references: [id])

  @@unique([provider, providerId])
}
```

One human, one `User`, N `Account` rows. This is exactly the shape NextAuth/Auth.js's Prisma adapter creates, and it's worth copying even if you hand-roll.

Corollary: **`password` must become nullable**, and every "change password" / "forgot password" path needs to handle a user who has none.

---

## 5. Frontend vs backend

The rule, in one line: **the frontend decides what to *render*; the backend decides what to *allow*.**

| | Frontend | Backend |
|---|---|---|
| `isSignedIn`, `user.role` | hide/show UI | ignore it — it's a hint, not a fact |
| Token verification | never meaningful | **always**, on every protected request |
| Trust level | zero (anyone can edit their own JS) | the only place a decision is real |

Hiding the admin button is UX. If `/api/admin/users` doesn't re-check the role server-side, the button being hidden means nothing — the endpoint is still one `curl` away.

Same caveat for Next.js middleware: it's a good gate, but run the real check in the route handler too. Middleware can be bypassed in ways route handlers cannot.

---

## 6. Picking one

| Option | Good when | Painful when |
|---|---|---|
| **Roll your own** | learning; full control; no per-user cost | you need MFA, magic links, passkeys, SSO — each is weeks |
| **Direct OIDC (DIY Google)** | one or two providers; you want the users in your DB; no vendor bill | 5 providers, account linking, org invites |
| **Auth.js / NextAuth** | Next.js; want DIY-ish with the boilerplate solved; self-hosted, free | you want prebuilt UI and an admin dashboard |
| **Clerk** | ship fast; great DX and UI; orgs/B2B built in | cost at scale; data lives with them |
| **Supabase / Firebase Auth** | already using the rest of the platform | you're not |
| **Auth0 / Cognito** | enterprise SSO, SAML, compliance paperwork | DX, pricing, and the learning curve |
| **Keycloak (self-hosted)** | must keep everything in-house | you now operate an identity server |

Rough heuristic: **side project or learning → DIY. Startup shipping fast → Clerk or Auth.js. Enterprise/regulated → Auth0/Keycloak.**

---

## Gotchas

- **`client_secret` in frontend code.** It's in the bundle, therefore public, therefore not a secret. Token exchange is server-side, always.
- **Loose redirect URIs.** Wildcards or open redirects on the callback hand the `code` to an attacker. Register exact URLs.
- **Trusting `email` without `email_verified`.** The account-takeover vector in §4.
- **No users table.** You'll rewrite the schema the first time you need a join.
- **Unverified webhooks.** `/api/webhooks/clerk` without signature verification = anyone can forge `user.created`.
- **Checking auth only in middleware.** Route handlers need their own check.
- **Assuming the provider's token *is* your session.** Google's `id_token` is a one-time login receipt; Clerk's session JWT is theirs to rotate. Your app's authorization state is yours.
- **`sub` is stable, `email` is not.** People change addresses; key on `(provider, sub)`.
- **Forgetting logout means two logouts.** Clearing your cookie doesn't end the Google/Clerk session — next click of "Sign in" may log them straight back in. Use the provider's `end_session_endpoint` / `signOut()` when you want a real logout.

## See also

- [auth.md](auth.md) — why any of this exists (Basic → sessions → OAuth → OIDC)
- [jwt.md](jwt.md) — the token format all of this rides on
