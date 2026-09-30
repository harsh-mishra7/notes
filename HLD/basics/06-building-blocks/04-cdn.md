# CDN (Content Delivery Network)

## Brief

**A CDN is a network of servers spread around the world that keep copies of your content close to your users.**

Instead of every user fetching files from your one server (the **origin**), they fetch from a nearby CDN server (an **edge**). Faster for users, far less load on you.

This is the short building-block overview. The main idea: **a CDN is a geographically distributed cache for your static content.**

---

## The analogy that makes it click

**A CDN is a chain of local warehouses.**

Your origin server is the factory. Shipping every order from the factory to customers worldwide is slow. So you keep popular items stocked in warehouses near your customers.

- The local warehouse **has it** → delivered fast → **cache HIT**
- It **doesn't** → warehouse orders it from the factory, delivers it, and keeps a copy for the next customer → **cache MISS**

The factory is still the source of truth. It just stops being involved in every order.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Origin** | Your real server (or object storage bucket). The source of truth. |
| **Edge server** | A CDN server near users that holds cached copies |
| **PoP** (Point of Presence) | A CDN location — a data center with many edge servers (e.g. "Mumbai PoP") |
| **Cache HIT** | Edge already had the file — served instantly, origin untouched |
| **Cache MISS** | Edge didn't have it — fetched from origin, stored, then served |
| **TTL** | How long the edge keeps a copy before re-checking with origin |

---

## Why it exists: distance costs time

Data travels through fiber at roughly 2/3 the speed of light. A round trip across the world is ~250-300 ms, and a page makes dozens of requests.

```
WITHOUT CDN                                WITH CDN

User in Mumbai ────────────► Origin        User in Mumbai ──► Mumbai PoP (~10 ms) ✅
               ◄────────────  (Virginia)                        │
               ~250 ms per request                              └─► Origin (first miss only)
```

Benefits:
- **Lower latency** — content comes from a nearby city.
- **Less origin load** — most requests never reach your servers.
- **Handles spikes & DDoS** — traffic is spread over hundreds of PoPs.
- **Cheaper bandwidth** — CDN egress is often cheaper than cloud egress.

---

## How a request flows

```
1. Browser requests  https://cdn.myapp.com/logo.png

2. DNS / anycast routes the browser to the NEAREST PoP

3. Edge server checks its cache
      ├─ HIT  ──────────────────────────► return file  (fast)
      └─ MISS ──► fetch from origin ──► store copy ──► return file
                                         (next user in this region gets a HIT)
```

Responses usually show what happened in a header like `x-cache: HIT` or `x-cache: MISS`.

---

## Push vs Pull CDN

| | **Pull CDN** | **Push CDN** |
|---|---|---|
| How content gets there | Edge fetches from origin on the **first miss** (lazy) | You **upload** content to the CDN ahead of time |
| Setup | Easy — just point the CDN at your origin | You manage uploads and updates |
| First request | Slower (a miss) | Already there |
| Storage | Only popular content ends up cached | Everything you push is stored |
| Best for | Most websites and apps (the default) | Large files, low traffic, or content you know will be needed (big videos, game patches) |

```
PULL:  User ──► Edge ──(miss)──► Origin          (edge pulls when asked)
PUSH:  You  ──► upload ──► CDN storage ──► Edge  (you push before anyone asks)
```

---

## What to put on a CDN

| ✅ Good fit (same for everyone) | ❌ Bad fit (personal / changes per request) |
|---|---|
| Images, videos, audio | Logged-in dashboards |
| CSS and JavaScript bundles | Shopping cart, account balance |
| Fonts, icons | Search results for a specific user |
| PDFs, downloads, installers | Anything with private data |
| Public API responses that rarely change | Real-time data (live scores, stock prices) |

**Rule of thumb:** if two different users should get the **exact same bytes**, it can go on a CDN. If the response depends on **who is asking**, don't cache it at the CDN — or you risk showing one user's data to another.

Modern CDNs can also accelerate dynamic content (optimized routes back to origin, edge compute), but in HLD interviews "CDN = static content" is the core idea.

---

## TTL and invalidation

Your origin controls how long edges keep a file, using the `Cache-Control` header:

```http
Cache-Control: public, max-age=86400        ← edges and browsers may cache for 1 day
Cache-Control: private, no-store            ← never cache at the CDN
```

The problem: you deploy a new `app.js`, but edges keep serving the old one until the TTL expires.

| Approach | How it works | Notes |
|---|---|---|
| **Wait for TTL** | Let old copies expire | Fine for short TTLs |
| **Purge / invalidate** | Tell the CDN to drop specific files everywhere | Takes seconds to minutes; may be rate-limited or cost money |
| **Versioned filenames** | `app.3f9a1c.js` → new deploy = new name | ✅ Best practice. Cache forever; new content = new URL. |

```
Deploy 1:  index.html → <script src="app.3f9a1c.js">   (cached for 1 year)
Deploy 2:  index.html → <script src="app.8b2e40.js">   (brand new URL, instantly fresh)

Keep index.html itself on a SHORT TTL — it's what points to the new filenames.
```

---

## Common CDNs

Cloudflare · AWS CloudFront · Akamai · Fastly · Google Cloud CDN · Azure Front Door/CDN · Bunny.net
