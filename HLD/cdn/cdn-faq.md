# CDN — Common Questions

Companion to [cdn.md](cdn.md). That file covers what a CDN is and how it works. This one answers the questions that come up once you start using one, or get asked about one in an interview.

---

## Index

**Caching**
1. [Is a CDN just a cache?](#1-is-a-cdn-just-a-cache)
2. [What exactly is the "cache key"?](#2-what-exactly-is-the-cache-key)
3. [What happens when the TTL expires?](#3-what-happens-when-the-ttl-expires)
4. [Browser cache vs CDN cache: what's the difference?](#4-browser-cache-vs-cdn-cache-whats-the-difference)
5. [Why is my cache hit ratio so low?](#5-why-is-my-cache-hit-ratio-so-low)
6. [What if a million users request the same uncached file at once?](#6-what-if-a-million-users-request-the-same-uncached-file-at-once)

**Routing**

7. [How does DNS know which CDN server is closest to the user?](#7-how-does-dns-know-which-cdn-server-is-closest-to-the-user)
8. [What happens if the nearest edge server goes down?](#8-what-happens-if-the-nearest-edge-server-goes-down)

**Security and protected content**

9. [How is paid / protected content cached safely?](#9-how-is-paid--protected-content-cached-safely)
10. [Can a CDN cache pages for logged-in users?](#10-can-a-cdn-cache-pages-for-logged-in-users)
11. [How does HTTPS work if the CDN serves my domain?](#11-how-does-https-work-if-the-cdn-serves-my-domain)
12. [Can people skip the CDN and hit my origin directly?](#12-can-people-skip-the-cdn-and-hit-my-origin-directly)
13. [What are cache poisoning and cache deception?](#13-what-are-cache-poisoning-and-cache-deception)

**Beyond static files**

14. [Can a CDN cache API responses or POST requests?](#14-can-a-cdn-cache-api-responses-or-post-requests)
15. [How do Netflix and YouTube stream video through a CDN?](#15-how-do-netflix-and-youtube-stream-video-through-a-cdn)
16. [What is edge computing?](#16-what-is-edge-computing)
17. [Why would anyone use more than one CDN?](#17-why-would-anyone-use-more-than-one-cdn)

---

## Caching

### 1. Is a CDN just a cache?

Mostly, yes. A CDN is a **cache spread across many locations**, plus the routing that sends each user to the nearest copy.

Each edge server is a big HTTP cache:

- **Storage tiers.** The most popular files sit in RAM, less popular ones on SSD, and the rest are fetched from origin when needed.
- **Eviction.** Storage is finite, so rarely requested files get pushed out (usually some form of LRU, *least recently used*). A file with a 1-year TTL can still be evicted in a day if nobody asks for it.
- **Per location.** The Tokyo edge and the London edge have separate caches. A HIT in London tells you nothing about Tokyo.

The "cache" part is the easy bit. The hard parts are the global routing, the private network between PoPs, TLS, and doing all of it at huge scale.

---

### 2. What exactly is the "cache key"?

The cache key is the string the CDN uses to decide whether two requests are asking for the same thing. If the keys match, both requests get the same cached response.

By default it's roughly:

```
scheme + host + path + query string
https   cdn.myapp.com /img/logo.png ?v=3
```

What the key includes decides what gets shared:

| Include too little | Include too much |
|---|---|
| Different content under one key → **wrong content served** (e.g. English page sent to a French user) | Identical content under many keys → **low hit ratio**, origin gets hammered |

Common ways to tune it:

- **Ignore irrelevant query params.** `?utm_source=twitter` and `?utm_source=email` return the same image, so strip tracking params from the key.
- **Sort query params** so `?a=1&b=2` and `?b=2&a=1` count as the same key.
- **Add a header to the key** only when the response really differs by it (for example `Accept-Language`). The `Vary` header tells the CDN to do this.

---

### 3. What happens when the TTL expires?

The edge does **not** throw the file away. It marks the copy as **stale** and **revalidates** it with the origin:

```
Edge → Origin:  GET /style.css
                If-None-Match: "abc123"         ← the ETag it has

Origin → Edge:  304 Not Modified                ← tiny response, no body
                (edge resets the TTL and keeps serving its copy)

         — or —

Origin → Edge:  200 OK + new file + new ETag    ← file really changed
```

A `304` is cheap because the file isn't sent again. That's why every static response should carry an `ETag` or `Last-Modified` header.

Two directives make expiry less painful:

```http
Cache-Control: max-age=60, stale-while-revalidate=300, stale-if-error=86400
```

| Directive | Meaning |
|---|---|
| `stale-while-revalidate=300` | For 5 minutes after expiry, serve the stale copy **immediately** and refresh it in the background. The user never waits on origin. |
| `stale-if-error=86400` | If origin is down or returning 5xx, keep serving the stale copy for up to a day. The CDN keeps your site up during an outage. |

---

### 4. Browser cache vs CDN cache: what's the difference?

| | Browser cache | CDN cache |
|---|---|---|
| Lives on | The user's device | The CDN's edge servers |
| Shared by | One user | Every user in that region |
| You can purge it? | **No.** Once it's on the device, you can't reach it | Yes, via the CDN's purge API |
| Controlled by | `max-age`, `private` | `s-maxage` (falls back to `max-age`), `public` |

The practical consequence: **you can purge the CDN but not browsers.** So a common setup is:

```http
Cache-Control: public, max-age=60, s-maxage=86400
```

Browsers re-check every minute, while the CDN holds the file for a day. When you deploy, you purge the CDN, and every user sees the change within a minute.

(For hashed filenames none of this matters: cache them everywhere, forever. See `cdn.md`.)

---

### 5. Why is my cache hit ratio so low?

**Hit ratio** = hits ÷ total requests. Static assets should be at 90%+. The usual causes of a low ratio:

| Cause | Fix |
|---|---|
| Origin sends `Cache-Control: private` or `no-cache` by default (many frameworks do) | Set explicit headers for static routes |
| `Set-Cookie` on static responses. Many CDNs refuse to cache anything that sets a cookie | Don't send session cookies on asset routes |
| Random query strings (`?t=1695...` cache busters, tracking params) | Strip them from the cache key |
| `Vary: User-Agent` or `Vary: Cookie`, which creates a separate copy for every browser or user | Vary only on what really changes the response |
| Long-tail content: millions of files, each requested rarely (user uploads, old videos) | Tiered caching / origin shield (see Q6) |
| Too many PoPs, so traffic is split thin and each PoP rarely sees repeats | Tiered caching |
| TTL too short | Raise it and use versioned URLs for freshness |

Always test by requesting the same URL twice and checking the `x-cache` / `cf-cache-status` header.

---

### 6. What if a million users request the same uncached file at once?

This is a **cache stampede**, also called the **thundering herd**. A popular file expires, or a new one goes viral, and thousands of edge requests miss at the same moment. They all hit origin together and can take it down.

CDNs defend against it in layers:

**1. Request collapsing (coalescing).** Within one edge server, the first miss goes to origin and the others *wait* for that one response. 10,000 misses become 1 origin request.

**2. Tiered caching / origin shield.** Edges don't talk to origin directly. They go through a middle layer:

```
  Tokyo edge ─┐
  Seoul edge ─┼──► Regional shield (Singapore) ──► Origin
 Mumbai edge ─┘

  300 edges missing  →  a handful of shields asking  →  origin sees ~1 request
```

The shield is one big shared cache for many edges. This also fixes the long-tail hit-ratio problem, because a file that's rare at each edge is often popular across a whole region.

**3. `stale-while-revalidate`.** An expired file keeps being served while one background request refreshes it, so users don't all miss at once.

---

## Routing

### 7. How does DNS know which CDN server is closest to the user?

There are two main techniques. Most CDNs use one or a mix of both.

#### Approach A: GeoDNS (DNS-based routing)

The CDN runs the **authoritative DNS server** for your CDN hostname, and it gives **different answers to different askers**.

```
1. You set:   cdn.myapp.com  CNAME  myapp.cdnprovider.net

2. User in Tokyo → their ISP's DNS resolver → asks the CDN's DNS server
3. CDN's DNS sees the request came from a Tokyo resolver IP
4. Looks it up in a geo-IP database + live latency/load data
5. Answers:  "myapp.cdnprovider.net = 203.0.113.10"  (the Tokyo PoP)
6. A London user asking the same name gets a London IP
```

**The catch:** the CDN's DNS server never sees the *user's* IP. It sees the IP of the user's **DNS resolver**. That's usually fine, since your ISP's resolver is near you. But if you use `8.8.8.8` (Google) or `1.1.1.1`, the resolver could be in a different city or country, and you get routed to the wrong PoP.

**The fix is EDNS Client Subnet (ECS).** The resolver passes along a truncated piece of the user's IP (for example `49.36.12.0/24`) so the CDN can route by the user's real location. Some privacy-focused resolvers deliberately don't send it.

"Closest" also doesn't strictly mean closest on the map. The DNS decision weighs:

- measured **network latency** (the CDN constantly probes routes between networks and PoPs),
- PoP **load** (a full PoP gets skipped),
- PoP **health**,
- **cost** (some bandwidth is cheaper than other bandwidth).

Answers use a **short DNS TTL** (often 20–60 seconds) so the CDN can re-route users quickly.

Used by: Akamai, AWS CloudFront, most traditional CDNs.

#### Approach B: Anycast (network-based routing)

Here DNS gives **everyone the same IP address**, and the internet itself does the routing.

```
Every PoP worldwide announces the same IP block: 104.16.0.0/12

Tokyo user → packets to 104.16.x.x → routers pick the "shortest" path → Tokyo PoP
London user → same IP → routers pick their shortest path → London PoP
```

This works through **BGP**, the protocol internet routers use to share routes. When many locations announce the same IP, each router forwards packets along the path it thinks is shortest, which is usually the nearest PoP by network topology.

| | GeoDNS | Anycast |
|---|---|---|
| Who picks the PoP | CDN's DNS server | Internet routers (BGP) |
| Sees real user location? | Only via the resolver (or ECS) | Yes, routing starts from the user's network |
| Failover speed | Limited by DNS TTL (seconds to minutes, and some clients ignore TTL) | Fast. A dead PoP stops announcing and traffic reroutes |
| Fine-grained control | High (can route by load, cost, anything) | Lower. BGP "shortest" means fewest network hops, not lowest latency |
| DDoS | Attack goes to specific IPs | Attack is naturally spread across all PoPs |

Used by: Cloudflare, Google, Fastly, and the DNS root servers themselves.

**Interview one-liner:** *"Either the CDN's DNS returns a different IP per region based on the resolver's location (GeoDNS), or every PoP shares one IP and BGP routes each user to the nearest one (anycast)."*

---

### 8. What happens if the nearest edge server goes down?

- **A single server fails:** a load balancer inside the PoP sends traffic to the other servers there. Users don't notice.
- **The whole PoP fails:**
  - *Anycast:* the PoP stops announcing its routes, and within seconds BGP sends users to the next-nearest PoP.
  - *GeoDNS:* health checks remove that PoP from DNS answers. Users switch over once their cached DNS record expires, which is why CDN DNS TTLs are short.
- **The whole CDN fails** (it has happened to Cloudflare, Fastly and Akamai): your site is down unless you planned a bypass, such as a multi-CDN setup (see Q17) or a DNS switch straight to origin.

---

## Security and protected content

### 9. How is paid / protected content cached safely?

Take a paid course video: only subscribers should watch it, but you still want it cached at the edge.

The key point is that **caching and access control are separate steps**:

> The **file** is the same for every paying user, so it's safe to cache once and share.
> The **permission check** runs on every request at the edge, *before* the cached copy is served.

The CDN checks "is this person allowed?" on each request, then serves from cache. The origin isn't involved in either step.

#### Technique 1: Signed URLs (most common)

Your backend, which knows who is a subscriber, creates a URL that is only valid for a short time:

```
https://cdn.myapp.com/course/lesson1.mp4
    ?expires=1727600000
    &signature=Hx8f2k...      ← HMAC(secret, path + expires [+ user IP])
```

```
1. User clicks "Play"
2. Your backend checks: logged in? paid? ✅
3. Backend signs the URL with a secret key that only it and the CDN know
4. Browser requests the signed URL from the edge
5. Edge recomputes the signature:
     - wrong or tampered signature → 403
     - expired                     → 403
     - valid                       → serve from cache (or fetch on miss)
6. The cache key IGNORES the signature params, so every subscriber's
   request maps to the same cached copy
```

Step 6 is what makes caching work. Every user has a *different* URL but gets the *same* cached file.

If a signed URL gets shared on Reddit, it stops working when it expires, usually within minutes to hours. For tighter control, you can include the user's IP in the signature.

#### Technique 2: Signed cookies

This works like signed URLs, but the signature is in a cookie. It's useful when one permission covers **many files**. A video stream is hundreds of small segment files, and you don't want to sign each URL. One cookie saying "can access `/course/123/*` until 6 PM" covers all of them.

#### Technique 3: Token validation in edge code

For custom rules, a small program running at the edge (Cloudflare Workers, Lambda@Edge) validates a JWT, checks its claims, and then serves from cache. This works like the signed URL approach, with full flexibility.

#### Other layers used alongside these

- **Lock down the origin** so the files can't be downloaded around the CDN (see Q12).
- **Private buckets.** The S3 bucket isn't public, and only the CDN has read access (CloudFront calls this Origin Access Control).
- **Hotlink protection.** Reject requests whose `Referer` isn't your site, so others can't embed your images on theirs. This is easy to spoof, so it only raises the bar.
- **Encrypted streaming / DRM** for premium video (Widevine, FairPlay, PlayReady). The cached segments are **encrypted**, so anyone can download the bytes, but only an authorised player gets the decryption key from a licence server. The CDN caches ciphertext and never needs to know who paid.

#### What NOT to do

- **Don't** put the user's session token in the cache key. Each user then gets a private copy, and caching does nothing.
- **Don't** check auth only at origin and cache the response. The first user's request gets cached, and everyone after them gets it free with **no check at all**.

---

### 10. Can a CDN cache pages for logged-in users?

This is a different problem from Q9. There the *file* was the same for everyone. Here the *content* differs per user ("Hi Alice", Alice's orders).

**The whole personalised response can't be shared.** But you can cache most of the page:

1. **Split static and personal parts (the standard approach).** Cache the page shell (HTML, JS, CSS) for everyone. The browser then calls `/api/me` with `Cache-Control: private` for the personal data. Most SPAs work this way.
2. **Cache per segment, not per user.** If the page only differs by country, language, or "logged in vs logged out", put *that* in the cache key. That's a handful of copies instead of one per user.
3. **Edge Side Includes (ESI) / edge code.** The CDN caches the shared page and fills in a small personal fragment at the edge.

The rule from `cdn.md` still holds: if the response depends on *who* is asking, it must be `private` or `no-store`. Getting this wrong is how "I logged in and saw someone else's account" incidents happen.

---

### 11. How does HTTPS work if the CDN serves my domain?

A browser connecting to `cdn.myapp.com` expects a valid certificate for `cdn.myapp.com`, and it's the CDN's edge that answers. So **the CDN must hold a certificate for your domain**:

- **The CDN issues one for you** (most common): it proves domain control and gets a free certificate, often from Let's Encrypt, and renews it automatically.
- **You upload your own** certificate and private key.

This means the connection is split in two:

```
Browser ══TLS══► Edge (decrypts, reads, caches) ══TLS══► Origin
         cert for your domain                      separate connection
```

Two consequences:

- **The CDN sees your traffic in plaintext.** It has to, or it couldn't cache anything. You are trusting your CDN provider. (Banks that won't hand over private keys use setups like Cloudflare's *Keyless SSL*, where the key stays on their own servers and signs only the handshake.)
- **Encrypt the second connection too.** Some setups default to HTTPS from the browser but plain HTTP from edge to origin, which leaves the traffic exposed on that second connection. Use full or strict TLS to origin.

---

### 12. Can people skip the CDN and hit my origin directly?

Yes, if they find your origin's IP. It can leak through old DNS records, email headers, or scanners that search the internet for your certificate. Then attackers skip the DDoS protection, WAF and signed-URL checks entirely.

Lock it down:

- **Firewall:** origin accepts traffic **only from the CDN's IP ranges**, which every CDN publishes.
- **Shared secret header:** the CDN adds `X-Origin-Secret: <random>` and the origin rejects requests without it.
- **Mutual TLS:** the CDN proves who it is with a client certificate.
- **No public IP at all:** tunnels such as Cloudflare Tunnel, where the origin connects *out* to the CDN.
- For S3-style storage: a private bucket readable only by the CDN.

---

### 13. What are cache poisoning and cache deception?

Both are attacks that use a mismatch between **what the cache key considers** and **what the origin considers**.

**Cache poisoning: an attacker gets a harmful response cached for everyone.**

```
Attacker:  GET /home
           X-Forwarded-Host: evil.com     ← header NOT in the cache key

Origin:    uses that header to build links → page contains <script src="evil.com/...">
CDN:       caches it under key "/home"
Everyone:  gets the poisoned /home until the TTL expires
```

Fix: the origin must not let **unkeyed inputs** (headers the CDN ignores) change the response. Either add them to the cache key or ignore them at origin.

**Cache deception: the attacker tricks the CDN into caching a victim's private page.**

```
Attacker sends victim a link:  https://myapp.com/account/profile.css

CDN:     ".css? static, I'll cache it"
Origin:  ignores the fake suffix, serves the victim's /account page (with their data)
CDN:     caches it publicly
Attacker then opens the same URL → gets the victim's account page from cache
```

Fix: base caching on the origin's `Cache-Control` header, not on the file extension. Make the origin return 404 for paths that don't really exist, and always send `private` / `no-store` on account pages.

---

## Beyond static files

### 14. Can a CDN cache API responses or POST requests?

**GET API responses: yes, if they're the same for everyone.** Product catalogues, public leaderboards, config and exchange rates are good candidates. Even a **1–5 second TTL** helps a lot: with 10,000 requests per second, origin sees 1 request per second per edge instead of all 10,000. This is called **micro-caching**.

**POST / PUT / DELETE: no.** These change data and aren't safe to replay, so CDNs pass them straight to origin. The CDN still helps a little: TLS terminates nearby and the edge-to-origin connection is already open on the CDN's fast network, so even uncached requests often get faster.

(GraphQL sends everything as POST, which is why it's hard to cache at a CDN. The workaround is "persisted queries" sent as GET.)

---

### 15. How do Netflix and YouTube stream video through a CDN?

A 2-hour movie isn't sent as one file. It's cut into **small segments** (2–10 seconds each) using **HLS** or **DASH**:

```
movie.m3u8                  ← playlist (manifest): lists all segments
  1080p/seg_0001.ts
  1080p/seg_0002.ts
  ...
  720p/seg_0001.ts          ← the same segments at other qualities
```

- Each segment is a plain static file, which is ideal for a CDN.
- The player downloads segments one by one and **switches quality on the fly** based on your bandwidth. This is *adaptive bitrate streaming*.
- Popular titles' first segments are cached almost everywhere, so playback starts instantly.
- Paid content uses signed cookies and DRM encryption (see Q9).
- For live streams, the manifest has a very short TTL (it changes constantly), while the segments are immutable once written.

Netflix goes further with **Open Connect**: its own cache servers placed *inside ISPs' data centres*, pre-filled overnight with what each region is predicted to watch the next day. This is a push CDN at huge scale.

---

### 16. What is edge computing?

Running **your own code on the CDN's edge servers** instead of only serving cached files. Examples: Cloudflare Workers, Lambda@Edge / CloudFront Functions, Vercel Edge Functions, Fastly Compute.

Typical uses:

- Auth and token checks before serving from cache (Q9)
- Redirects, A/B test bucketing, geo-based routing
- Rewriting the cache key, adding security headers
- Personalising cached HTML (Q10)
- Resizing images on the fly

Limits: short CPU time, small memory, restricted APIs. And **your database is still in one region**, so edge code that queries it on every request just moves the slow round trip elsewhere. Edge computing works best for logic that doesn't need the database, or that uses edge storage (KV stores, globally replicated DBs).

---

### 17. Why would anyone use more than one CDN?

Large companies often run **multi-CDN** (for example Akamai + CloudFront + Fastly) for:

- **Resilience.** One CDN's outage doesn't take the site down.
- **Performance.** CDN A is best in Europe, CDN B in India. Route each region to the fastest one using real-user measurements.
- **Cost and negotiating power.** Shift traffic to whichever is cheaper this month.

The cost is complexity: a traffic-steering DNS layer in front, cache configs and purges kept in sync across providers, and several certificates and log formats to manage. It's not worth it until you're big enough that a CDN outage costs real money.

---

## Quick-fire recap

| Question | One-line answer |
|---|---|
| How is the nearest server picked? | GeoDNS (different IP per region) or anycast (same IP, BGP picks the path) |
| Why can public DNS resolvers mis-route? | The CDN sees the resolver's location, not yours. ECS fixes it |
| How is paid content cached? | Validate a signed URL/cookie at the edge, keep the signature out of the cache key |
| Can personalised pages be cached? | Cache the shell, fetch personal data separately with `private` |
| TTL expired, now what? | Revalidate with `ETag` → `304`, or serve stale while refreshing |
| Stampede protection? | Request collapsing + origin shield + `stale-while-revalidate` |
| Does the CDN see my HTTPS traffic? | Yes. It terminates TLS in order to cache |
| Protect the origin? | Allow only CDN IPs, a secret header, or a tunnel |
| Cache APIs? | Public GETs yes (even 1s helps), POST never |
| Video? | Split into small HLS/DASH segments, each cached like any static file |
