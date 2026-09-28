# CDN 

## TL;DR

**CDN = Content Delivery Network.**

A network of servers spread around the world that keep **copies of your files close to your users**.

Without a CDN, a user in Tokyo asking for your logo waits for that image to travel from your server in Virginia — halfway around the planet, every single time.
With a CDN, the first Tokyo user pulls it across the ocean once; everyone after that gets it from a server sitting in Tokyo.

That's the whole idea. Everything else is detail.

---

## The problem it solves: distance costs time

Data travels through cables at roughly 2/3 the speed of light. That's fast, but not free:

| Route | Round trip (best case) |
|---|---|
| Same city | ~5 ms |
| Across a country | ~50 ms |
| Across the world | ~250-350 ms |

And a page doesn't make *one* request — it makes dozens (HTML, CSS, JS, fonts, images). Each one pays the distance tax.

```
NO CDN                                 WITH CDN

Tokyo user  ──────────────► Virginia   Tokyo user ──► Tokyo edge  (5 ms)  ✅
            ◄──────────────  server                      │
              ~300 ms, every request                     └──► Virginia (only the first time)
```

Your server is also doing all the work alone. 1 million users = 1 million requests hitting one machine.

---

## The analogy that makes it click

**A CDN is a chain of local warehouses.**

You (the origin) are the factory. Shipping every order from the factory to every customer worldwide is slow and expensive. So you stock the popular items in warehouses near your customers.

- Customer orders something **the local warehouse has** → ships today. → **cache HIT**
- Customer orders something **rare** → warehouse fetches it from the factory, ships it, *and keeps a copy* for the next person. → **cache MISS**

The factory still exists and is still the source of truth. It just stops being involved in every single order.

---

## The vocabulary (only 4 words you need)

| Term | Meaning |
|---|---|
| **Origin server** | Your actual server. The source of truth. |
| **Edge server / PoP** | The CDN's servers around the world holding copies. (PoP = Point of Presence) |
| **Cache HIT** | The edge already had the file. Fast, origin untouched. |
| **Cache MISS** | The edge didn't have it, so it fetched from origin and stored it. |

---

## How a request actually flows

```
1. Browser wants  https://cdn.myapp.com/logo.png

2. DNS lookup → the CDN answers with the IP of the *nearest* edge server
                (this routing is the CDN's real magic — anycast / geo-DNS)

3. Browser asks that edge server.

   ┌─ Edge has it (HIT)  ──────────────► sends it back.  Done in ~10 ms.
   │
   └─ Edge doesn't (MISS) ─► asks your origin ─► stores a copy ─► sends it back.
                                                  (next user gets a HIT)
```

So the first visitor in a region pays the slow price. Everyone after them doesn't.

You can see which happened — most CDNs add a response header:

```
x-cache: HIT
age: 431            ← seconds this copy has been sitting on the edge
```

---

## What belongs on a CDN

| Good fit (static) | Bad fit (dynamic / personal) |
|---|---|
| Images, video, fonts | Your logged-in dashboard |
| CSS and JS bundles | Shopping cart contents |
| PDFs, downloads | Bank balance, search results |
| Public API responses that rarely change | Anything with a user's private data |

**Rule of thumb:** if two different people should see the exact same bytes, cache it. If the response depends on *who* is asking, don't — or you'll serve Alice's data to Bob. That's a real and common bug.

---

## How you control the cache: `Cache-Control`

Your origin tells the CDN and browser how long a file stays fresh, using a response header:

```http
Cache-Control: public, max-age=31536000, immutable
```

| Directive | Meaning in plain English |
|---|---|
| `public` | Anyone (CDN included) may cache this |
| `private` | Only the user's browser may cache it — CDN must not |
| `max-age=60` | Fresh for 60 seconds |
| `s-maxage=300` | Same, but only for the CDN (overrides `max-age` there) |
| `no-store` | Never save this anywhere |
| `immutable` | This file will never change — don't even bother re-checking |

**TTL** (Time To Live) is just the nickname for that lifetime. When it expires, the edge re-checks with origin.

---

## The hard part: how do you update a cached file?

You pushed a new `style.css`, but edges around the world are still handing out the old one for the next 24 hours. Two ways out:

**1. Purge / invalidate (the manual way)**

Tell the CDN "drop `style.css` everywhere." Works, but it's slow to propagate, sometimes rate-limited, and easy to forget.

**2. Versioned filenames (the way everyone actually does it)**

Never change a file — change its *name*:

```
style.a3f9c1.css      ← content hash in the filename
style.7b2e40.css      ← new deploy, new hash, new URL = guaranteed new file
```

A URL that never changes content can be cached for a year safely. This is why build tools (Vite, Webpack, Next.js) put those random-looking hashes in your filenames — it's built for exactly this.

> Cache the hashed assets forever. Keep `index.html` on a short TTL, since it's the file that points to the new hashes.

---

## Free extras you get with a CDN

Speed is the headline, but usually not the only reason teams adopt one:

- **Origin protection** — most traffic never reaches your server, so it survives traffic spikes.
- **DDoS absorption** — a flood hits hundreds of edge servers instead of your one box.
- **HTTPS termination** — the TLS handshake happens nearby, saving another round trip or two.
- **Compression** — gzip/brotli applied at the edge automatically.
- **Image optimization** — many CDNs resize and convert to WebP/AVIF on the fly.
- **Bandwidth cost** — CDN egress is typically cheaper than your cloud provider's.

---

## Push vs Pull CDN

| | How it works | Use when |
|---|---|---|
| **Pull** (default, 99% of cases) | Edge fetches from origin on first miss, lazily | Normal websites and apps |
| **Push** | You upload files to the CDN yourself, ahead of time | Huge files, rare traffic (big video, installers) where a single miss is too expensive |

---

## When you *don't* need one

- Your users are all in one city/region and your server is there too. Distance is already ~0.
- Everything you serve is personalized and uncacheable.
- Small internal tool with 20 users — the complexity isn't worth it.

A CDN also adds a layer that can *cause* bugs (stale files, wrong region, caching something personal). Reach for it when distance or scale is genuinely hurting you.

---

## Gotchas

- **Caching a personalized response is a data leak.** If a response varies per user, mark it `private` / `no-store`, or you'll serve one user's page to another.
- **Purging is not instant.** Expect seconds to minutes across the world. Never rely on it for a time-critical fix — use versioned filenames.
- **Don't cache HTML for long.** It's the entry point that references everything else; a stale `index.html` freezes your whole deploy.
- **`Vary` matters.** If a response differs by `Accept-Encoding` or `Accept-Language`, say so with the `Vary` header, or the CDN will hand the wrong variant to someone.
- **A CDN doesn't fix a slow backend.** Cache misses and dynamic routes still go to origin at full speed — meaning your speed.
- **CDN outages are your outages.** You've added a dependency in front of everything. Know how to bypass it.
- **Check `x-cache` before celebrating.** A misconfigured `Cache-Control` means 100% MISS, and you've quietly added a hop instead of removing one.

---

## Common CDNs

Cloudflare · AWS CloudFront · Fastly · Akamai · Google Cloud CDN · Azure CDN · Bunny.net · Vercel/Netlify Edge (built into their hosting)

They differ in price, PoP count, and how programmable the edge is — but the core model above is identical across all of them.
