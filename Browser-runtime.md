# How a Web Page Actually Runs

The browser is the program. Your JavaScript is a guest that gets called back.

> Companion to [The JavaScript Event Loop](Event-loop.md). That note explains the loop — the queues, the ordering, the drain. This one explains what *feeds* it: who owns the threads, how a page boots, and how a click in the real world turns into a function call.

## Table of Contents
1. [The Mental Model](#the-mental-model)
2. [The Browser Is a Small Operating System](#the-browser-is-a-small-operating-system)
3. [From URL to Pixels](#from-url-to-pixels)
4. [How Scripts Load](#how-scripts-load)
5. [The Lifecycle Events](#the-lifecycle-events)
6. [The DOM Is the API Boundary](#the-dom-is-the-api-boundary)
7. [How a Click Reaches Your Code](#how-a-click-reaches-your-code)
8. [The Rendering Pipeline](#the-rendering-pipeline)
9. [Single Page Apps Invert the Model](#single-page-apps-invert-the-model)
10. [Gotchas](#gotchas)
11. [Summary](#summary)

---

## The Mental Model

The intuitive model of a web page goes something like this:

> The HTML renders, the JS runs, then event listeners sit there listening for things to happen.

Everything in that sentence is *almost* right, and the two places it's wrong are the two places all the confusion lives.

**Wrong thing #1: nothing in JavaScript is listening.** Your code is not polling. It cannot be — JS has one thread, and a polling loop would hold it forever:

```js
while (true) { checkForClicks() }   // the tab is now dead
```

The listening is done by the **browser**, in native code, on threads you don't control. `addEventListener` is a registration: you hand a function pointer to a C++ DOM node and your stack unwinds. The main thread goes idle.

**Wrong thing #2: rendering and script execution are interleaved, not sequential.** A `<script>` tag stops the HTML parser dead. Your JS can — and usually does — run before the page has ever been painted, with a DOM that is only half-built.

The accurate model has two phases:

```
[ PARSE + INITIAL RUN ]   a few hundred ms, once
         ↓
[ IDLE / REACTIVE ]       minutes or hours — 99% of the page's life
   call stack empty, main thread parked,
   woken only when the event loop hands it a task
```

Phase 2 is the whole game. **After startup, every line of JS that ever executes is a callback.** A click handler, a `setTimeout`, a `fetch().then`, a `requestAnimationFrame`. There is no `main()`, no outer loop you wrote. The browser drives you.

---

## The Browser Is a Small Operating System

"The browser is single-threaded" is shorthand for "**your JavaScript** gets one thread." The browser itself is aggressively multi-process and multi-threaded.

```
┌─ Browser process ────────── UI, tabs, address bar, navigation, OS input
│
├─ Network process ────────── DNS, TCP, TLS, HTTP, cache
│
├─ GPU process ────────────── rasterization, drawing to the screen
│
└─ Renderer process ───────── one per site (sandboxed)
   │
   ├── Main thread  ◄── YOUR JS lives here. Also: DOM, CSSOM,
   │                    style, layout, paint, and the event loop.
   ├── Compositor thread ── input hit-testing, scrolling, layer compositing
   ├── Raster threads ────── paint records → bitmaps
   └── Worker threads ────── Web Workers (real parallel JS)
```

Two consequences worth internalising:

**Your JS shares its thread with rendering.** Style, layout and paint all need the main thread. So a slow `for` loop doesn't just block other JS — it blocks the *pixels*. That's why a frozen tab is frozen visually, not just functionally.

**The waiting happens elsewhere.** `fetch` doesn't block because the network process does the socket work. `setTimeout` doesn't block because a timer thread tracks expiry. Both just queue a callback when they're done. This is the entire trick behind "single-threaded but non-blocking."

> A Web Worker is the only way to get real parallel JS — and it gets its *own* event loop, its own stack, and **no DOM access**. It talks to the main thread by message passing only.

---

## From URL to Pixels

```
1. Navigation     URL → browser process → network process
2. Fetch          DNS → TCP → TLS → HTTP GET → response headers
3. Renderer       pick/spawn a renderer process for that site
4. Parse          bytes → chars → tokens → nodes → DOM tree
5. Style          CSS → CSSOM, then matched against the DOM
6. Layout         geometry: where every box is, how big
7. Paint          draw order → paint records
8. Raster         records → bitmap tiles
9. Composite      tiles → GPU → screen
```

Steps 4–9 are not one-shot. They run again, partially, every time something changes for the rest of the page's life.

### Parsing is incremental and interruptible

The parser builds the DOM top-down and streams: it starts painting before the HTML has finished downloading. But two things interrupt it:

| Resource | Effect |
|---|---|
| `<link rel="stylesheet">` | **Render-blocking.** Parsing continues, but nothing paints until CSS is in. |
| `<script>` (plain) | **Parser-blocking.** DOM construction halts entirely until it downloads *and* runs. |

CSS blocks rendering because painting with un-styled content would flash. JS blocks parsing because `document.write` exists and any script might restructure the document underneath the parser.

There's a non-obvious interaction between them:

> A `<script>` will not execute until all **preceding stylesheets** have loaded — because the script might call `getComputedStyle()` and expect a real answer. So a slow CSS file can delay your JS.

### The preload scanner

While the main parser is stuck on a script, a secondary lightweight parser races ahead through the raw HTML looking for `src` and `href`, and starts those downloads early. It's why moving scripts around often changes less than you'd expect — the *fetches* were already parallel. What it can't fix is the **execution** block.

---

## How Scripts Load

This table is the practical payoff of everything above:

| Tag | Download | Execute | Order kept | Blocks parser |
|---|---|---|---|---|
| `<script>` | blocks parser | immediately | yes | **yes** |
| `<script defer>` | parallel | after parsing, before `DOMContentLoaded` | yes | no |
| `<script async>` | parallel | the instant it lands | **no** | on execute |
| `<script type="module">` | parallel | like `defer` | yes | no |
| `<script type="module" async>` | parallel | the instant it lands | no | on execute |

```html
<head>
  <script src="a.js"></script>          <!-- ❌ stalls the whole page -->
  <script defer src="b.js"></script>    <!-- ✅ default choice -->
  <script async src="analytics.js"></script>  <!-- ✅ independent, order-free -->
</head>
```

- **`defer`** is the right default for app code: parallel download, ordered execution, DOM guaranteed complete.
- **`async`** is for scripts with zero dependencies on anything else — analytics, error reporters. Execution order between async scripts is a race.
- **Modules** (`type="module"`) are deferred by default, run in strict mode, and have their own scope — no accidental globals.

The old "put scripts at the bottom of `<body>`" advice was a workaround for a world without `defer`. `defer` does it better, because the download starts earlier.

---

## The Lifecycle Events

```
  parsing starts
        │
        ├── readyState = "loading"
        │
  HTML fully parsed + deferred scripts executed
        │
        ├── readyState = "interactive"
        └── ► DOMContentLoaded        ← DOM is safe to touch
        │
  images, stylesheets, iframes, fonts finish
        │
        ├── readyState = "complete"
        └── ► load                    ← everything is in
        │
        ...page lives here for minutes or hours...
        │
        ├── ► visibilitychange        ← tab hidden/shown
        ├── ► pagehide                ← navigating away (bfcache-safe)
        └── ► beforeunload            ← last chance to warn
```

| Event | Fires when | Use it for |
|---|---|---|
| `DOMContentLoaded` | DOM parsed, deferred scripts run | initialising your app |
| `load` | + all subresources | measuring images, final layout math |
| `visibilitychange` | tab backgrounded/foregrounded | pausing polling, video, animation |
| `pagehide` | leaving the page | saving state, `sendBeacon` |
| `beforeunload` | leaving the page | "you have unsaved changes" prompt |

`DOMContentLoaded` is what jQuery's `$(document).ready()` wrapped. With `defer` you rarely need it — your script already runs at the right moment.

**The trap**: if your script runs *after* the event already fired, your listener never fires. Guard it:

```js
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', init);
} else {
  init();                   // already parsed — just go
}
```

`beforeunload` is unreliable by design (it blocks the **bfcache**, the back/forward instant-restore cache) and browsers ignore custom messages. Prefer `visibilitychange` → `hidden` for saving state.

---

## The DOM Is the API Boundary

The DOM is not a JavaScript data structure. It's a **native C++ tree** that the browser exposes to JS through wrapper objects. Every property access crosses a language boundary.

```
   JavaScript                    │                    Browser (C++)
 ─────────────────────────────── │ ───────────────────────────────────
   el.textContent = 'hi'    ─────┼────►  mutate node, mark dirty
   el.offsetHeight          ◄────┼─────  force layout, return a number
   addEventListener(fn)     ─────┼────►  store fn on the node
                            ◄────┼─────  later: invoke fn as a task
```

This boundary explains a lot of performance folklore:

- **DOM reads and writes are not free.** They're cross-boundary calls, and some of them (`offsetHeight`, `getBoundingClientRect`) force the browser to compute layout *synchronously* before it can answer.
- **Batching works.** Fifty style changes inside one function produce exactly one layout and one paint, because the browser can't render mid-task anyway.
- **This is what virtual DOMs are for.** React diffs a plain-JS tree (cheap) and issues the minimum number of real DOM writes (expensive).

The globals you actually have:

| Object | What it is |
|---|---|
| `window` | the global object *and* the browser tab/frame — `globalThis` in a browser |
| `document` | the root of the DOM tree |
| `navigator` | info about the browser/device |
| `location` | the URL, and a way to change it |
| `history` | the session stack — the thing SPAs manipulate |

---

## How a Click Reaches Your Code

This is the full path from a physical mouse to your callback:

```
1. OS         hardware interrupt → OS input event
2. Browser    browser process receives it, routes to the right tab
              process
3. Compositor compositor thread hit-tests: which element is under
      thread   those coordinates? Can this be handled without JS?
                 ├─ yes (e.g. a plain scroll) → scroll immediately,
                 │                              main thread untouched
                 └─ no  → post a task to the main thread
4. Event      main thread picks up the task (see Event-loop.md),
      loop     builds the Event object, computes the path
5. Dispatch   capture → target → bubble
6. Your fn    finally called, with `event` as the argument
```

### Capture, target, bubble

An event doesn't just fire on the element. It travels the whole path from the document root down to the target and back up:

```
   document
      │  ① CAPTURE phase — root → target
      ▼
    <div>
      │
      ▼
   <button>   ② TARGET
      │
      ▲  ③ BUBBLE phase — target → root
    <div>
      ▲
   document
```

```js
// default: fires on the way UP (bubble)
el.addEventListener('click', fn);

// fires on the way DOWN (capture)
el.addEventListener('click', fn, { capture: true });
```

| Method | Effect |
|---|---|
| `e.stopPropagation()` | stop travelling further along the path |
| `e.stopImmediatePropagation()` | also skip other listeners on *this* element |
| `e.preventDefault()` | cancel the browser's built-in behaviour (link nav, form submit) |
| `e.target` | what was actually clicked |
| `e.currentTarget` | the element whose listener is running |

Not everything bubbles: `focus`, `blur`, `load`, `scroll` (on elements) don't. `focusin` / `focusout` are the bubbling versions of the first two.

### Event delegation

Bubbling is what makes **one listener** able to serve a thousand elements:

```js
// ❌ 1000 listeners, and new rows get none
document.querySelectorAll('.row').forEach(r =>
  r.addEventListener('click', handle));

// ✅ one listener, works for rows added later
document.querySelector('#table').addEventListener('click', (e) => {
  const row = e.target.closest('.row');
  if (row) handle(row);
});
```

This is exactly what React does — it attaches a small set of listeners at the root container and routes events to your components itself. A React `onClick` is **not** a real DOM listener on that element.

### Passive listeners

Step 3 above has a fast path: if the compositor knows no JS wants to cancel a scroll, it scrolls on its own thread without waiting for the main thread at all. But a listener *might* call `preventDefault()`, so the browser has to ask first — unless you promise it won't:

```js
// ✅ "I will never preventDefault" → compositor scrolls immediately
el.addEventListener('touchmove', fn, { passive: true });
```

Chrome now defaults `touchstart` / `touchmove` / `wheel` on `window` and `document` to passive for exactly this reason. A non-passive scroll listener that does slow work is one of the most common sources of scroll jank.

---

## The Rendering Pipeline

Once the page is live, every visual change re-runs some suffix of this:

```
  JS/CSS  →  Style  →  Layout  →  Paint  →  Composite
                         │          │          │
   change width ─────────┘          │          │   all three
   change color ────────────────────┘          │   two
   change transform/opacity ───────────────────┘   one — and off-thread
```

| You change | Runs | Cost |
|---|---|---|
| `width`, `top`, `margin`, `font-size` | layout → paint → composite | 🔴 expensive |
| `color`, `background`, `box-shadow` | paint → composite | 🟠 medium |
| `transform`, `opacity` | composite only | 🟢 cheap — can run on the compositor thread |

Which is the real reason animation advice says "use `transform`, not `top`": a `transform` animation can run entirely on the compositor thread and **keeps animating even while the main thread is busy**. A `top` animation needs layout, needs the main thread, and dies the moment your JS blocks.

### Layout thrashing

Reads force the browser to flush pending layout so it can give you a truthful number. Interleaving reads and writes makes it flush over and over:

```js
// ❌ forced synchronous layout, once per iteration
for (const el of els) {
  el.style.height = el.offsetHeight + 10 + 'px';   // read, write, read, write...
}

// ✅ batch: all reads, then all writes
const heights = els.map(el => el.offsetHeight);     // read phase
els.forEach((el, i) => el.style.height = heights[i] + 10 + 'px');  // write phase
```

The properties that trigger this: `offsetTop/Left/Width/Height`, `clientTop/Left/Width/Height`, `scrollTop/Left/Width/Height`, `getBoundingClientRect()`, `getComputedStyle()`.

### The frame budget

The browser wants a frame every **16.7 ms**. Rendering only happens between tasks — so this is the same constraint as the event loop's, seen from the other side:

```
|--- task ---|-- microtasks --|-- rAF --|-- style/layout/paint --|--- task ---|
 <------------------------ ~16.7 ms budget ------------------------>
```

`requestAnimationFrame(fn)` runs `fn` in that gap, right before paint. It's where visual updates belong.

---

## Single Page Apps Invert the Model

The intuitive model — HTML document with some script tags — is an accurate description of a 2010 website. A modern SPA turns it inside out:

```html
<body>
  <div id="root"></div>
  <script type="module" src="/bundle.js"></script>
</body>
```

The HTML is a husk. The JS bundle **builds the entire DOM at runtime**. The primitives are identical — DOM nodes, listeners, the event loop — but everything is authored in JS instead of shipped as markup.

**Navigation is intercepted**, not performed:

```js
// change the URL without a server round-trip or a page reload
history.pushState({ page: 'about' }, '', '/about');
render('/about');

// the back button fires this instead of a navigation
window.addEventListener('popstate', (e) => render(location.pathname));
```

A client-side router is: intercept link clicks → `preventDefault()` → `pushState` → swap what's rendered. The document never reloads, so **JS state survives navigation** — which is the entire point, and also why SPAs leak memory in ways multi-page sites can't.

The trade-offs, briefly:

| | Multi-page | SPA |
|---|---|---|
| First paint | fast — HTML is ready | slow — wait for bundle, then build DOM |
| Navigation | full reload, state lost | instant, state kept |
| SEO | trivial | needs SSR/prerender |
| Memory | reset every page | accumulates |

**SSR + hydration** is the compromise: render the HTML on the server so the first paint is real content, then ship the same JS to the client to attach listeners to the already-existing DOM. Hydration is the phase where "the markup is there but nothing is clickable yet."

---

## Gotchas

**Script runs before the element exists.**

```html
<script>
  document.querySelector('#btn').addEventListener(...)  // ❌ null
</script>
<button id="btn">Click</button>
```

Use `defer`, or move the script below, or wait for `DOMContentLoaded`.

**`innerHTML` destroys listeners.** Replacing markup discards the old nodes and their registered handlers along with them. The new nodes have none. This is the single strongest argument for event delegation.

```js
container.innerHTML = '<button id="b">hi</button>';  // any old listeners: gone
```

**`load` when you meant `DOMContentLoaded`.** `load` waits for every image and font. On an image-heavy page that's seconds of your app sitting idle for no reason.

**Assuming `e.target` is the element you bound to.** With delegation it's whatever was actually clicked — often a child `<span>` or `<svg>` inside your button. `e.currentTarget` is the bound element; `e.target.closest(sel)` is usually what you want.

**Adding listeners in a loop without removing them.** Detached DOM nodes still referenced by a closure won't be collected. In SPAs this is the classic leak — a component mounts, subscribes to `window.resize`, unmounts, never unsubscribes.

```js
const onResize = () => { /* ... */ };
window.addEventListener('resize', onResize);
// cleanup — the same function reference is required
window.removeEventListener('resize', onResize);
```

`{ once: true }` and `AbortSignal` are the tidier options:

```js
const ctrl = new AbortController();
window.addEventListener('resize', onResize, { signal: ctrl.signal });
ctrl.abort();                      // removes it — and any others on that signal
```

**Heavy work in a `scroll`, `resize` or `mousemove` handler.** These fire far more often than you can render. Throttle them, or better, use the purpose-built observers: `IntersectionObserver` for "is it visible", `ResizeObserver` for "did it change size". Both run off the main thread's hot path.

**Expecting intermediate paints inside a function.** The browser cannot render mid-task. A loop that sets `el.textContent` 100 times shows only the last value.

**Animating `top`/`left` instead of `transform`.** Forces layout every frame, on the main thread, where your JS can starve it.

**Forgetting the compositor can't help you if the listener isn't passive.** A non-passive `wheel` handler means every scroll waits on the main thread.

---

## Summary

- The **browser** is the program; your JS is a guest it calls back. Nothing in JS is polling or listening.
- Your JS gets **one thread**, shared with style, layout and paint. Everything else — network, timers, rasterisation, input hit-testing — runs on other threads or other processes.
- Page load is **interleaved**, not sequential: CSS blocks rendering, a plain `<script>` blocks parsing, and a script waits on preceding stylesheets. `defer` is the sane default.
- After startup the page is purely **reactive**. Every line of JS that runs is a callback delivered by the event loop.
- `addEventListener` hands a function to a native DOM node. The browser stores it, hit-tests input on the compositor thread, and queues a task when it matches.
- Events travel **capture → target → bubble**, which is what makes event delegation — and React's synthetic event system — possible.
- Visual changes cost **layout → paint → composite**, in that order. `transform` and `opacity` skip to the last step and can run off the main thread.
- SPAs use the same primitives with the polarity flipped: JS builds the DOM, `history.pushState` fakes navigation, and state survives — along with the leaks.
- Every symptom in this note and in [Event-loop.md](Event-loop.md) reduces to the same sentence: **something held the main thread too long.**
