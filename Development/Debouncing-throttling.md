# Debouncing & Throttling

Two ways to stop a function from running more often than it usefully can.

## Table of Contents
1. [The Problem](#the-problem)
2. [Debouncing](#debouncing)
3. [Throttling](#throttling)
4. [Side-by-Side](#side-by-side)
5. [Which One Do I Want?](#which-one-do-i-want)
6. [Implementations](#implementations)
7. [React](#react)
8. [Gotchas](#gotchas)
9. [Beyond the Browser](#beyond-the-browser)

---

## The Problem

Some events fire far faster than anything you'd want to do in response to them.

| Event | Rough fire rate |
|---|---|
| `mousemove` | 60–120 / second |
| `scroll` | 30–100 / second |
| `resize` | dozens per drag |
| `keyup` in a search box | as fast as someone types |

If each of those triggers an API call, a layout recalculation, or a re-render, you get a laggy page and a hammered server. Typing "headphones" into a search-as-you-type box is **10 keystrokes = 10 network requests**, and you only ever cared about the last one.

Debouncing and throttling both reduce that to something sane. They differ in **which** calls survive.

---

## Debouncing

> **Wait until the activity stops, then run once.**

Every call resets the timer. The function only fires after `delay` milliseconds of silence.

```
Keystrokes:  h  e  a  d  p        h  o  n  e  s
             |  |  |  |  |        |  |  |  |  |
Timer:       reset each time...   ...and again...
Fires:                     ✗                    ✓ (300ms after the last key)
```

**Mental model:** an elevator door. Every new person walking in restarts the "closing in 3 seconds" countdown. The lift only leaves once nobody else shows up.

**The trade:** you get exactly one execution with the final value, but you get it *late* — nothing happens while the user is still active.

### Typical uses
- Search-as-you-type / autocomplete
- Auto-saving a draft while someone writes
- Validating a form field after typing stops
- `resize` handlers that recompute layout (do it once, when the drag ends)

---

## Throttling

> **Run at most once every N milliseconds, no matter how many calls come in.**

The first call goes through, then a cooldown window opens. Calls during the window are dropped (or collapsed into one trailing call).

```
Scroll events: ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●
Throttle 200ms: ✓.......✓.......✓.......✓.......
```

**Mental model:** a machine-gun with a fire rate. Hold the trigger down all you want — it still fires at a fixed cadence.

**The trade:** you get regular updates during the activity (good for anything visual), but you deliberately throw work away.

### Typical uses
- Scroll position → sticky headers, progress bars, infinite scroll
- `mousemove` → drag-and-drop, canvas drawing, tooltips following the cursor
- Rate-limiting button clicks / form submissions
- Analytics or telemetry pings
- Game loops and animation

---

## Side-by-Side

| | Debounce | Throttle |
|---|---|---|
| **Rule** | Run once after silence | Run at most once per interval |
| **During a burst** | Nothing runs | Runs at a steady rate |
| **After a burst** | Runs once | Maybe one trailing call |
| **Timer on new call** | Reset | Ignored (window keeps running) |
| **Guarantees the last value** | Yes | Only with a trailing call |
| **Guarantees regular updates** | No | Yes |
| **Question it answers** | "What did they *end up* typing?" | "Where are they *right now*?" |

Same input, 10 events over 1 second, 300 ms setting:

```
Events:     ● ● ● ● ● ● ● ● ● ●
Debounce:                       ✓        (1 call, after the last event)
Throttle:   ✓     ✓     ✓     ✓          (≈4 calls, evenly spaced)
```

---

## Which One Do I Want?

Ask: **do I need intermediate values, or only the final one?**

- Only the final one → **debounce**
- I need to keep up during the activity → **throttle**

Quick lookup:

| Scenario | Pick | Why |
|---|---|---|
| Search box → API | Debounce (~300 ms) | Intermediate queries are wasted requests |
| Scroll progress bar | Throttle (~100 ms) | Must update *while* scrolling |
| Window resize → re-layout | Debounce (~250 ms) | The final size is the only one that matters |
| Drag an element | Throttle (~16 ms) | Needs to track the cursor smoothly |
| "Submit" button | Throttle (leading only) | Fire immediately, block double-clicks |
| Infinite scroll trigger | Throttle (~200 ms) | Check position periodically |
| Autosave a document | Debounce (~1000 ms) | Save when they pause, not every keystroke |

**Rule of thumb for delays:** 200–500 ms for typing (below ~200 ms feels like no debounce at all; above ~500 ms feels broken), ~16 ms (one frame) for anything animating, 100–200 ms for scroll.

---

## Implementations

### Debounce

```js
function debounce(fn, delay = 300) {
  let timer;

  function debounced(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  }

  debounced.cancel = () => clearTimeout(timer);
  return debounced;
}
```

Usage:

```js
const search = debounce((q) => fetch(`/api/search?q=${q}`), 300);
input.addEventListener('input', (e) => search(e.target.value));
```

### Debounce with a leading edge

Fire immediately on the first call, then stay quiet until things settle. Good for "do it now, but don't repeat".

```js
function debounce(fn, delay = 300, { leading = false, trailing = true } = {}) {
  let timer = null;

  return function (...args) {
    const callNow = leading && timer === null;
    clearTimeout(timer);

    timer = setTimeout(() => {
      timer = null;
      if (trailing && !callNow) fn.apply(this, args);
    }, delay);

    if (callNow) fn.apply(this, args);
  };
}
```

### Throttle (timestamp version)

Leading edge — fires immediately, then once per window.

```js
function throttle(fn, limit = 200) {
  let lastRun = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastRun >= limit) {
      lastRun = now;
      fn.apply(this, args);
    }
  };
}
```

Note the simple version **drops the final call**. If the user stops scrolling 10 ms into a window, that last position is never handled.

### Throttle with a trailing call

The version you usually actually want:

```js
function throttle(fn, limit = 200) {
  let lastRun = 0;
  let timer = null;

  return function (...args) {
    const now = Date.now();
    const remaining = limit - (now - lastRun);

    if (remaining <= 0) {
      clearTimeout(timer);
      timer = null;
      lastRun = now;
      fn.apply(this, args);
    } else if (!timer) {
      // schedule the last call of this window
      timer = setTimeout(() => {
        lastRun = Date.now();
        timer = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}
```

### `requestAnimationFrame` throttling

For anything that touches the DOM or paints, throttle to the frame rate instead of a fixed millisecond value — the browser tells you when it's ready.

```js
function rafThrottle(fn) {
  let queued = false;

  return function (...args) {
    if (queued) return;
    queued = true;
    requestAnimationFrame(() => {
      queued = false;
      fn.apply(this, args);
    });
  };
}

window.addEventListener('scroll', rafThrottle(updateProgressBar));
```

---

## React

### The classic bug

```jsx
// ❌ Broken: a new debounced function every render,
// so the timer is thrown away before it can fire.
function Search() {
  const [q, setQ] = useState('');
  const search = debounce((v) => fetchResults(v), 300);

  return <input onChange={(e) => search(e.target.value)} />;
}
```

### Debounced value hook

Usually cleaner than debouncing the callback — debounce the *value* and react to it.

```jsx
function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);   // reset on every change
  }, [value, delay]);

  return debounced;
}

function Search() {
  const [q, setQ] = useState('');
  const debouncedQ = useDebouncedValue(q, 300);

  useEffect(() => {
    if (debouncedQ) fetchResults(debouncedQ);
  }, [debouncedQ]);

  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}
```

The cleanup function *is* the reset — that's the whole debounce.

### Debounced callback hook

```jsx
function useDebouncedCallback(fn, delay = 300) {
  const fnRef = useRef(fn);
  fnRef.current = fn;                       // always call the latest closure

  const timer = useRef(null);

  const debounced = useCallback((...args) => {
    clearTimeout(timer.current);
    timer.current = setTimeout(() => fnRef.current(...args), delay);
  }, [delay]);

  useEffect(() => () => clearTimeout(timer.current), []);  // clear on unmount

  return debounced;
}
```

---

## Gotchas

**Losing `this` and the event object.** Use `fn.apply(this, args)`, not `fn(...args)`, if the wrapped function is a method. And with React's older pooled events (React &lt;17), read `e.target.value` *before* the delay — pull the value out, don't pass the event.

**Creating the wrapper inside a loop or a render.** A debounced function is only useful if it's the *same* function across calls — its timer lives in the closure. Create it once.

**Forgetting to cancel on unmount.** A pending timer that fires after the component is gone either wastes work or warns about setting state on an unmounted component.

**Debouncing a search box with no loading state.** The UI looks frozen for 300 ms. Show a spinner as soon as the input changes, not when the request starts.

**Out-of-order responses.** Debouncing reduces requests but doesn't order them. If request #1 (slow) resolves after request #2 (fast), you render stale results. Use `AbortController` to cancel in-flight requests, or check that the response matches the current query.

```js
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/search?q=${debouncedQ}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setResults)
    .catch(err => { if (err.name !== 'AbortError') throw err; });
  return () => controller.abort();
}, [debouncedQ]);
```

**Throttling something that needed the last value.** Leading-edge-only throttle silently drops the final event. If the end state matters (final scroll position, final cursor spot), use the trailing version.

**Reaching for a scroll throttle when the platform has a better answer.** `IntersectionObserver` for "is this element visible", `ResizeObserver` for element size, CSS `position: sticky` for sticky headers. No scroll listener needed at all.

**Debounce ≠ rate limiting.** Client-side throttling is a UX optimization, not a security control. Anyone can call your API directly. Rate-limit on the server too.

---

## Beyond the Browser

The same two shapes show up all over:

- **Backend** — debounce webhook processing so 50 rapid updates trigger one downstream job; throttle outbound calls to a third-party API to stay under its rate limit.
- **Databases / search indexing** — debounce reindexing after a burst of writes.
- **Hardware** — the term comes from *switch debouncing*: a physical button's contacts bounce for a few milliseconds and register as multiple presses, so the circuit ignores changes for a settling period.
- **Notifications** — debounce "you have new messages" so a chatty thread sends one digest instead of forty pings.

---

## Summary

- **Debounce** = *"stop bothering me until you're done."* One call, at the end, with the final value.
- **Throttle** = *"I'll listen, but only every N ms."* Regular calls, at a fixed rate, during the activity.
- Debounce is about **the last value**; throttle is about **keeping up**.
- Both are closures holding a timer. Everything else is edge cases: leading vs trailing, cancel, and cleanup.
