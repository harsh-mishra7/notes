# The JavaScript Event Loop

How a single-threaded language runs a timer, three network requests and a scroll handler without blocking.

## Table of Contents
1. [The Problem](#the-problem)
2. [The Pieces](#the-pieces)
3. [One Turn of the Loop](#one-turn-of-the-loop)
4. [Macrotasks vs Microtasks](#macrotasks-vs-microtasks)
5. [Worked Examples](#worked-examples)
6. [async / await Is Just Microtasks](#asyncawait-is-just-microtasks)
7. [Rendering and the Frame Budget](#rendering-and-the-frame-budget)
8. [Node.js: The Same Idea, More Phases](#nodejs-the-same-idea-more-phases)
9. [Gotchas](#gotchas)
10. [Not Blocking the Loop](#not-blocking-the-loop)
11. [Summary](#summary)

---

## The Problem

JavaScript runs on **one thread**. One call stack, one thing happening at a time. Whatever is running holds the thread until it returns — there is no preemption, nothing can interrupt it.

```js
while (true) {}   // the tab is now dead. No clicks, no paint, no timers.
```

But a browser tab has to juggle timers, user input, network responses, animation and rendering — all at once, all on that one thread. The event loop is the scheduler that makes that work: **run one thing to completion, then pick up the next**.

The key word is *to completion*. A JS task is never paused halfway to run something else. That's why you never need locks or mutexes in JS — and also why one slow function freezes everything.

> The event loop is **not part of the JS language**. V8 (the engine) only gives you a heap and a call stack. The loop, the timers and the queues all live in the *host*: the browser, or Node, or Deno.

---

## The Pieces

| Piece | What it is | Who owns it |
|---|---|---|
| **Call stack** | Frames for the functions currently running. LIFO. | JS engine |
| **Heap** | Where objects live. | JS engine |
| **Web APIs / host APIs** | `setTimeout`, `fetch`, DOM events, disk I/O. Run **outside** the JS thread. | Host |
| **Task queue** (macrotask / callback queue) | Callbacks waiting for their turn. | Host |
| **Microtask queue** | Promise reactions. Higher priority, separate queue. | Host |
| **Event loop** | "Is the stack empty? Then take the next thing." | Host |

```
        ┌──────────────────────────────────────────┐
        │               Call Stack                 │
        │        (one thing at a time, LIFO)       │
        └──────────────────────────────────────────┘
                   ▲                   │
        run it     │                   │  hand off async work
                   │                   ▼
      ┌────────────┴───┐   ┌───────────────────────────┐
      │   Event Loop   │   │   Web APIs / host         │
      │ "stack empty   │   │   timers, fetch, DOM      │
      │     yet?"      │   │   events, I/O, workers    │
      └────────────────┘   └─────────────┬─────────────┘
            ▲       ▲                    │ done → queue a callback
            │       │                    ▼
   ┌────────┴───┐ ┌─┴──────────────────────────────────┐
   │ Microtasks │ │  Task queue (macrotasks)           │
   │ drain ALL  │ │  take exactly ONE per turn         │
   └────────────┘ └────────────────────────────────────┘
```

The important asymmetry, and basically the whole note in one line:

> **One macrotask per turn. All microtasks after it.**

---

## One Turn of the Loop

Each turn: take **one** task and run it to completion, drain **every** microtask, then — if this turn happens to be a frame — render.

Three things fall out of that:

1. **Your code is never interrupted.** A callback runs start to finish before anything else gets a turn.
2. **Microtasks jump the queue.** A promise queued inside a `setTimeout` runs before the *next* `setTimeout`.
3. **Rendering only happens between tasks.** If a task takes 500 ms, the page is frozen for 500 ms.

---

## Macrotasks vs Microtasks

### Macrotasks (tasks)
One per loop turn. Rendering can happen between them.

- `setTimeout`, `setInterval`
- `setImmediate` (Node)
- DOM events — `click`, `scroll`, `input`
- Network callbacks, I/O completion
- `MessageChannel` / `postMessage`

### Microtasks
Drained completely after each macrotask, and again whenever the stack empties.

- `.then()` / `.catch()` / `.finally()` callbacks
- Continuations after `await`
- `queueMicrotask(fn)`
- `MutationObserver` callbacks

| | Macrotask | Microtask |
|---|---|---|
| **How many per turn** | Exactly one | All of them — the queue is drained |
| **New ones queued while running** | Wait for a future turn | Run in the *same* drain |
| **Rendering in between** | Possible | Never |
| **Can starve the loop** | No | Yes |
| **Typical source** | `setTimeout`, events, I/O | Promises, `await` |

```
Task: click handler
  ├─ logs "a"
  ├─ queues promise → microtask
  └─ queues setTimeout → macrotask
  ↓ stack empties
Microtask drain: promise callback runs          ← before the timer
  ↓
(maybe render)
  ↓
Next task: the setTimeout callback
```

---

## Worked Examples

### 1. The classic

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');
```

```
1        ← synchronous
4        ← synchronous
3        ← microtask, drained as soon as the script finishes
2        ← macrotask, next turn
```

The whole `<script>` is itself a task. `3` beats `2` even though the timer was registered first, because the microtask queue is drained *before* the loop picks up the next task.

### 2. Microtasks queued by microtasks

```js
Promise.resolve().then(() => {
  console.log('A');
  Promise.resolve().then(() => console.log('B'));
});

setTimeout(() => console.log('C'), 0);
```

```
A
B      ← queued during the drain, still runs in this drain
C      ← only once the microtask queue is completely empty
```

### 3. Microtasks inside macrotasks

```js
setTimeout(() => {
  console.log('timer 1');
  Promise.resolve().then(() => console.log('  micro from timer 1'));
}, 0);

setTimeout(() => {
  console.log('timer 2');
  Promise.resolve().then(() => console.log('  micro from timer 2'));
}, 0);
```

```
timer 1
  micro from timer 1     ← drained before timer 2 gets its turn
timer 2
  micro from timer 2
```

### 4. The interview question

```js
console.log('script start');

setTimeout(() => console.log('setTimeout'), 0);

async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}
async function async2() {
  console.log('async2');
}

async1();

new Promise((resolve) => {
  console.log('promise constructor');
  resolve();
}).then(() => console.log('promise then'));

console.log('script end');
```

```
script start
async1 start          ← async functions run synchronously until the first await
async2
promise constructor   ← the executor is synchronous too
script end
async1 end            ← microtask: the continuation after await
promise then          ← microtask: queued after it
setTimeout            ← macrotask
```

Two traps live in here: **the body of an `async` function before the first `await` is synchronous**, and **the `new Promise` executor is synchronous** — only `.then` callbacks are deferred.

---

## async/await Is Just Microtasks

`await` is syntax over `.then()`. It splits a function in half: everything before it runs now, everything after it becomes a microtask.

```js
async function f() {
  console.log('before');
  await something;
  console.log('after');
}
```

is roughly:

```js
function f() {
  console.log('before');
  return Promise.resolve(something).then(() => {
    console.log('after');
  });
}
```

So `await` doesn't block the thread — it **returns the thread to the event loop** and registers the rest of the function as a continuation. Nothing is paused; the function literally stops and something else runs.

`await` on a non-promise still yields:

```js
console.log('1');
(async () => {
  console.log('2');
  await 42;              // not a promise, still defers the rest
  console.log('4');
})();
console.log('3');
// 1, 2, 3, 4
```

---

## Rendering and the Frame Budget

The browser wants to paint ~60 times a second — one frame every **16.7 ms**. It can only do that between tasks, because style, layout and paint need the main thread.

```
|--- task ---|-- microtasks --|-- rAF --|-- style/layout/paint --|--- task ---|
 <------------------------ ~16.7 ms budget ------------------------>
```

- `requestAnimationFrame(fn)` — runs `fn` just before the next paint. This is the correct place for animation, not `setTimeout(fn, 16)`.
- Anything over **50 ms** is a "long task" and is what users feel as jank.
- A background tab's timers are throttled to ~1/second, and `requestAnimationFrame` stops entirely.

```js
// ❌ drifts, fights the refresh rate
setInterval(() => el.style.left = (x += 1) + 'px', 16);

// ✅ synced to the browser's paint cycle
function step() {
  el.style.left = (x += 1) + 'px';
  requestAnimationFrame(step);
}
requestAnimationFrame(step);
```

---

## Node.js: The Same Idea, More Phases

Node uses libuv, and its "task queue" is actually several queues visited in a fixed order. Each trip through them is one **tick**:

```
   ┌─────────────────────────┐
┌─>│         timers          │  setTimeout, setInterval
│  ├─────────────────────────┤
│  │    pending callbacks    │  some deferred system/TCP errors
│  ├─────────────────────────┤
│  │     idle, prepare       │  internal
│  ├─────────────────────────┤      ┌───────────────┐
│  │          poll           │<─────┤  incoming I/O │  file reads, sockets
│  ├─────────────────────────┤      └───────────────┘
│  │          check          │  setImmediate
│  ├─────────────────────────┤
└──┤     close callbacks     │  socket.on('close')
   └─────────────────────────┘
```

Between **every** phase (and between every individual callback), Node drains two microtask queues, in this order:

1. `process.nextTick` queue — Node-specific, **highest priority of all**
2. Promise microtask queue

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
console.log('sync');
```

```
sync
nextTick
promise
timeout / immediate   ← order is NOT deterministic from the main module
```

That last pair depends on how long the process took to boot — if the loop starts after 1 ms has already elapsed, the timer is ready and fires first; otherwise `check` gets there first. **Inside an I/O callback it is deterministic**: `setImmediate` always wins, because `check` comes right after `poll`.

```js
fs.readFile('f.txt', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// immediate, then timeout — every time
```

Rule of thumb: `setImmediate` means "after the current poll phase"; `process.nextTick` means "right now, before anything else" — and is easy to abuse (see below).

---

## Gotchas

**`setTimeout(fn, 0)` is not 0 ms.** It means "queue this as a task, at the earliest after 0 ms". It runs after the current task *and* the whole microtask queue. On top of that, HTML clamps nested timers to a **4 ms minimum after 5 levels of nesting**, and background tabs get throttled to ~1 s.

**Timers are a floor, not a promise.** `setTimeout(fn, 100)` means "not before 100 ms". If the thread is busy at the 100 ms mark, the callback waits.

**Microtask starvation.** A microtask that queues another microtask never lets the loop continue. No timers, no events, no rendering — the tab hangs:

```js
function loop() { Promise.resolve().then(loop); }
loop();                 // ❌ freezes the page, forever

function loop() { setTimeout(loop, 0); }
loop();                 // ✅ macrotask — the loop breathes between turns
```

In Node the same trap is `process.nextTick` recursion.

**Blocking the thread with sync work.** `JSON.parse` on a 20 MB string, a `for` loop over a million items, `alert()`, synchronous XHR, `fs.readFileSync` in a server — all of it stops every other callback dead. Async does not mean parallel: `await` on a CPU-bound function is still CPU-bound.

**`await` in a loop serializes everything.**

```js
// ❌ 10 requests, one after another
for (const id of ids) results.push(await fetch(`/api/${id}`));

// ✅ all in flight at once
const results = await Promise.all(ids.map(id => fetch(`/api/${id}`)));
```

**Expecting `setTimeout` to preserve order across delays.** Timers fire by expiry time, not registration order, and equal delays fall back to insertion order.

**Assuming promises are async because they're promises.** The `new Promise` executor runs synchronously. Only the reactions are deferred.

**`for...of` over a loop of DOM updates, expecting intermediate paints.** The browser can't paint mid-task. Fifty style changes in one function produce exactly one paint — usually what you want, and the reason layout thrashing (`read → write → read → write`) is expensive for a different reason: forced synchronous layout.

**Errors in a microtask don't stop the drain.** An unhandled rejection is reported but the queue keeps going, which makes ordering bugs quiet.

---

## Not Blocking the Loop

**Chunk the work** so the loop can breathe between pieces:

```js
function processInChunks(items, fn, chunk = 100) {
  let i = 0;
  function run() {
    const end = Math.min(i + chunk, items.length);
    for (; i < end; i++) fn(items[i]);
    if (i < items.length) setTimeout(run, 0);   // yield to the loop
  }
  run();
}
```

**Yield explicitly** where supported — `await scheduler.yield()`, or `scheduler.postTask(fn, { priority: 'background' })`.

**Move CPU work off the thread entirely** — a Web Worker in the browser, `worker_threads` or a child process in Node. That's the only real parallelism JS has.

**Use the platform's async version.** `fs.promises.readFile` over `readFileSync`, `IntersectionObserver` over a scroll handler, streaming JSON over one giant `JSON.parse`.

**Measure it.** `performance.now()` around suspect code, the Performance panel's long-task bars, or `PerformanceObserver` on `longtask` entries.

---

## Summary

- JS is single-threaded; the event loop decides **what runs next**, never *at the same time*.
- Each turn: **one macrotask**, then **the entire microtask queue**, then maybe **a render**.
- Macrotasks = timers, events, I/O. Microtasks = promises, `await`, `queueMicrotask`.
- Microtasks always beat macrotasks — and can starve the loop if they queue each other forever.
- `await` doesn't pause anything; it hands the thread back and schedules the rest of the function as a microtask.
- Node is the same model with ordered phases, plus `process.nextTick` ahead of promises.
- Every performance problem here has the same shape: **something held the thread too long.**
