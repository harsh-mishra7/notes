# Promises

A value you don't have yet, with a fixed set of rules for what happens when you finally get it — or don't.

## Table of Contents
1. [The Problem](#the-problem)
2. [What a Promise Actually Is](#what-a-promise-actually-is)
3. [Creating a Promise](#creating-a-promise)
4. [Consuming: then, catch, finally](#consuming-then-catch-finally)
5. [Chaining — The Three Rules](#chaining--the-three-rules)
6. [Error Handling](#error-handling)
7. [async / await](#async--await)
8. [Combinators](#combinators)
9. [Promises and the Microtask Queue](#promises-and-the-microtask-queue)
10. [Patterns](#patterns)
11. [Gotchas](#gotchas)
12. [Building One From Scratch](#building-one-from-scratch)
13. [Summary](#summary)

---

## The Problem

Before promises, async results came back through callbacks:

```js
getUser(id, (err, user) => {
  if (err) return handle(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handle(err);
    getPayment(orders[0].id, (err, payment) => {
      if (err) return handle(err);
      render(payment);           // four levels deep, three identical error checks
    });
  });
});
```

Two separate problems here, and only one of them is the indentation.

**Callback hell** is the cosmetic one — nesting grows with every step, error handling is copy-pasted at each level, and there's no way to say "do these three in parallel" without a counter variable.

**Inversion of control** is the real one. You hand your callback to `getUser` and *hope* it behaves: calls you back exactly once, doesn't call you synchronously sometimes and asynchronously other times, doesn't swallow your thrown errors, actually calls you at all. You have no guarantees, and a misbehaving library can't be fixed from the outside.

A promise flips that around. Instead of giving your continuation *to* the async function, the async function gives you **an object**, and you attach your continuation to it. Now the guarantees live in the object, and the language enforces them.

```js
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getPayment(orders[0].id))
  .then(render)
  .catch(handle);              // one error path for the whole chain
```

---

## What a Promise Actually Is

A promise is a **state machine with one transition**.

```
                   ┌──────────────────┐
                   │     pending      │
                   └────────┬─────────┘
                            │
             resolve(v)     │     reject(e)
          ┌─────────────────┴─────────────────┐
          ▼                                   ▼
  ┌───────────────┐                   ┌───────────────┐
  │   fulfilled   │                   │   rejected    │
  │   value = v   │                   │   reason = e  │
  └───────────────┘                   └───────────────┘
          └──────────── settled ──────────────┘
                     (permanent)
```

| State | Meaning |
|---|---|
| **pending** | No result yet. The initial state. |
| **fulfilled** | Succeeded, carries a **value**. |
| **rejected** | Failed, carries a **reason** (should be an `Error`). |
| **settled** | Not a state — shorthand for "fulfilled or rejected". |

The properties that make it useful:

- **It settles exactly once.** The second `resolve()` or `reject()` call is silently ignored. No "called my callback twice" bugs.
- **It's immutable once settled.** Nothing can change the value afterwards.
- **It's always asynchronous.** A `.then` callback *never* runs in the same tick, even on an already-settled promise. No "sometimes sync, sometimes async" — [Zalgo](https://blog.izs.me/2013/08/designing-apis-for-asynchrony/) stays in the box.
- **You can attach handlers whenever.** Before it settles, after it settled an hour ago — you still get the value. The result is stored, not broadcast-and-forgotten like an event.
- **It's a value.** You can return it, store it in an array, pass it to a function, `await` it twice.

> A promise represents a **result**, not a task. It does not give you a handle on the underlying work — there is no `.cancel()`. That's what `AbortController` is for.

---

## Creating a Promise

### From an existing value

```js
Promise.resolve(42);              // already fulfilled with 42
Promise.reject(new Error('nope')); // already rejected
```

`Promise.resolve(p)` on something that's already a promise returns it unchanged. On a **thenable** (any object with a `.then` method) it adopts it — that's how promise libraries interoperate.

### From a callback API — the executor

```js
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) reject(new RangeError('ms must be >= 0'));
    setTimeout(resolve, ms);
  });
}
```

The executor function runs **synchronously and immediately** — `new Promise` does not defer anything. Only the `.then` callbacks are deferred.

```js
console.log('a');
new Promise(res => { console.log('b'); res(); }).then(() => console.log('d'));
console.log('c');
// a, b, c, d
```

If the executor throws, the promise rejects with whatever was thrown — so a synchronous error inside it is handled, not lost.

### Promise.withResolvers (ES2024)

When you need to settle the promise from somewhere else entirely:

```js
const { promise, resolve, reject } = Promise.withResolvers();

socket.on('message', resolve);
socket.on('error', reject);

const first = await promise;
```

Before ES2024 this was the "deferred" pattern — hoisting `resolve`/`reject` out of the executor by hand.

---

## Consuming: then, catch, finally

```js
p.then(onFulfilled, onRejected)   // both optional
p.catch(onRejected)               // === p.then(undefined, onRejected)
p.finally(onSettled)              // runs either way, gets no argument
```

Every one of these **returns a new promise**. That's the entire mechanism behind chaining, and it's worth staring at:

```js
const a = fetch('/api');
const b = a.then(r => r.json());   // b is a DIFFERENT promise from a
```

`.finally` is for cleanup — hiding a spinner, closing a handle. It passes the value or reason straight through untouched:

```js
setLoading(true);
fetchData()
  .then(render)
  .catch(showError)
  .finally(() => setLoading(false));
```

Two caveats on `.finally`: its callback receives **no arguments** (it doesn't know how things went), and if it *throws* or returns a rejected promise, that rejection replaces whatever was passing through.

---

## Chaining — The Three Rules

Almost every promise bug comes from not internalising what a `.then` callback's **return value** does. There are exactly three cases:

```js
p.then(v => {
  return 5;                   // 1. plain value  → next .then gets 5
  return somePromise;         // 2. a promise    → chain WAITS, unwraps it
  throw new Error('x');       // 3. throw        → chain rejects, skips to .catch
});
```

| You return | The next `.then` receives |
|---|---|
| a plain value | that value |
| nothing (`undefined`) | `undefined` ← the classic bug |
| a promise | the promise's **resolved value**, after it settles |
| a rejected promise / `throw` | skipped — control jumps to the next `.catch` |

Rule 2 is the important one: **returning a promise flattens it**. Promises never nest — there is no `Promise<Promise<T>>`. This is what makes the flat chain possible.

```js
// ❌ nested — we're back in callback hell with extra steps
getUser(id).then(user => {
  getOrders(user.id).then(orders => {
    console.log(orders);       // and an error here escapes the outer .catch
  });
});

// ✅ flat — return it
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => console.log(orders))
  .catch(handle);
```

**Forgetting the `return` is the single most common promise bug.** With an arrow function's implicit return you're safe; the moment you add braces you have to say `return` yourself:

```js
.then(user => getOrders(user.id))          // ✅ implicit return
.then(user => { getOrders(user.id); })     // ❌ returns undefined, chain doesn't wait
```

### Chaining doesn't mean sequential *start*

A chain sequences what you *write* in it, but promises start executing the moment they're created — not when you `.then` them. This is **eager** evaluation, unlike a lazy observable or a Rust future:

```js
const a = fetch('/one');   // request is already in flight
const b = fetch('/two');   // this one too — they're parallel

await a;                   // we just wait for them here
await b;
```

---

## Error Handling

A rejection propagates down the chain until something handles it. `.then`'s fulfilment handlers are skipped along the way.

```js
step1()
  .then(step2)        // skipped if step1 rejected
  .then(step3)        // skipped
  .catch(handle);     // ← lands here
```

### Position matters

```js
// .catch at the end covers everything above it
fetchUser().then(render).catch(handle);

// a .catch in the middle RECOVERS — the chain continues fulfilled
fetchUser()
  .catch(() => DEFAULT_USER)   // returns a value → next .then runs
  .then(render);               // runs with DEFAULT_USER

// second argument only catches the promise ABOVE it, not render's own errors
fetchUser().then(render, handle);       // a throw inside render is unhandled
fetchUser().then(render).catch(handle); // ✅ catches both
```

### Rethrowing

A `.catch` that returns normally *swallows* the error and puts the chain back on the happy path. To log-and-continue-failing, rethrow:

```js
.catch(err => {
  logger.error(err);
  throw err;            // stays rejected for callers downstream
})
```

### fetch doesn't reject on 4xx/5xx

The single most-missed error case in the browser. `fetch` only rejects on *network* failure — DNS, offline, CORS, abort. A 404 is a perfectly successful HTTP round trip as far as it's concerned:

```js
const res = await fetch(url);
if (!res.ok) throw new Error(`HTTP ${res.status}`);   // you must do this yourself
return res.json();
```

### Unhandled rejections

A rejected promise with no handler is reported by the runtime — and in Node ≥15 it **crashes the process** by default.

```js
// browser
window.addEventListener('unhandledrejection', e => {
  report(e.reason);
  e.preventDefault();          // stops the console error
});

// node
process.on('unhandledRejection', reason => { report(reason); process.exit(1); });
```

Attaching a handler *later* in the same tick is fine — the check happens after the microtask queue drains. Attaching one in a `setTimeout` is too late.

---

## async / await

`async`/`await` is syntax over the same promises. Nothing new happens underneath.

- An `async` function **always returns a promise**, whatever you return inside it.
- A `throw` inside an `async` function becomes a rejection of that promise.
- `await p` pauses the function, hands the thread back, and resumes as a **microtask** when `p` settles.
- `await` on a non-promise still yields a microtask — it wraps the value.

```js
async function load(id) {
  const user   = await getUser(id);
  const orders = await getOrders(user.id);
  return orders;                       // → Promise<Orders>
}
// exactly equivalent to:
function load(id) {
  return getUser(id).then(user => getOrders(user.id));
}
```

Error handling goes back to ordinary `try/catch`:

```js
try {
  const data = await load(id);
  render(data);
} catch (err) {
  handle(err);          // network failures AND anything render-adjacent that throws
} finally {
  setLoading(false);
}
```

### When to still reach for `.then`

- **Combinators** — `Promise.all([...])` reads better than orchestrating with `await`.
- **Fire-and-forget with a handler** — `track(event).catch(noop);` at top level.
- **Not in an async function**, and top-level `await` isn't available (CommonJS, non-module scripts).

### await in a loop

The most expensive line in most codebases:

```js
// ❌ serial: 10 requests × 200 ms = 2 seconds
for (const id of ids) results.push(await fetch(`/api/${id}`));

// ✅ parallel: ~200 ms
const results = await Promise.all(ids.map(id => fetch(`/api/${id}`)));
```

Use the loop version deliberately when you need ordering, rate limiting, or when each step depends on the last. Otherwise it's just slow.

---

## Combinators

| Method | Settles when | Fulfils with | Rejects when |
|---|---|---|---|
| `Promise.all` | all fulfil, or one rejects | array of values (in order) | **first** rejection — fails fast |
| `Promise.allSettled` | all settle | array of `{status, value \| reason}` | never |
| `Promise.race` | first to **settle** | that value | if the first to settle rejected |
| `Promise.any` | first to **fulfil** | that value | only if **all** reject → `AggregateError` |

```js
// all — I need everything, and one failure means the whole thing is useless
const [user, settings, perms] = await Promise.all([
  getUser(id), getSettings(id), getPerms(id),
]);

// allSettled — independent work, partial success is fine
const results = await Promise.allSettled(urls.map(u => fetch(u)));
const ok = results.filter(r => r.status === 'fulfilled').map(r => r.value);

// race — usually a timeout
const data = await Promise.race([
  fetch(url),
  new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), 5000)),
]);

// any — first mirror that answers wins
const fastest = await Promise.any([fetch(m1), fetch(m2), fetch(m3)]);
```

Two things `Promise.all` does **not** do: it doesn't start anything (the promises were already running when you built the array), and it doesn't cancel the others when one rejects — they keep going, and their results are simply discarded. If one of *those* later rejects with nobody listening, you get an unhandled rejection. `allSettled` avoids that entirely.

Order of results in `all` and `allSettled` matches the **input array**, not completion order.

---

## Promises and the Microtask Queue

Promise reactions run on the **microtask queue**, which is drained completely after each macrotask and before any rendering. See [Event-loop.md](Event-loop.md) for the full picture.

The practical consequence:

```js
console.log('1');
setTimeout(() => console.log('2'), 0);          // macrotask
Promise.resolve().then(() => console.log('3')); // microtask
console.log('4');

// 1, 4, 3, 2   ← the promise beats the zero-delay timer, always
```

Each `await` or `.then` link costs one microtask tick, which is why this is a classic interview question:

```js
async function f() { console.log('f start'); await null; console.log('after await'); }
console.log('script start');
f();
Promise.resolve().then(() => console.log('then'));
console.log('script end');

// script start → f start → script end → after await → then
```

Microtasks can **starve the loop** — a promise chain that queues itself forever will freeze the page just as hard as a `while(true)`, because the drain never finishes.

---

## Patterns

### Promisify a callback API

```js
const promisify = fn => (...args) =>
  new Promise((resolve, reject) =>
    fn(...args, (err, val) => (err ? reject(err) : resolve(val))));

const readFile = promisify(fs.readFile);
```

Node ships this as `util.promisify`, and most core modules already have a `fs.promises`-style counterpart.

### Timeout

```js
const withTimeout = (promise, ms) => Promise.race([
  promise,
  new Promise((_, rej) =>
    setTimeout(() => rej(new Error(`timed out after ${ms}ms`)), ms)),
]);
```

Note the losing promise isn't cancelled — the work continues. For a real cancel, use `AbortSignal.timeout(ms)` with `fetch`.

### Retry with backoff

```js
async function retry(fn, attempts = 3, base = 300) {
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (err) {
      if (i === attempts - 1) throw err;
      await delay(base * 2 ** i + Math.random() * 100);   // jitter
    }
  }
}
```

### Concurrency limit

`Promise.all` over 500 URLs opens 500 connections. Cap it:

```js
async function pool(items, worker, limit = 5) {
  const results = [];
  const running = new Set();

  for (const item of items) {
    const p = Promise.resolve().then(() => worker(item));
    results.push(p);

    // track a never-rejecting twin, so one failure doesn't break the loop
    const slot = p.then(() => {}, () => {});
    running.add(slot);
    slot.then(() => running.delete(slot));

    if (running.size >= limit) await Promise.race(running);
  }
  return Promise.all(results);   // the real rejection surfaces here
}
```

Racing the raw promises instead of the twins looks simpler but isn't: a rejection would throw out of the loop, leaving the already-started promises in `results` with nobody handling them.

### Cancellation with AbortController

Promises have no cancel. The platform's answer is a separate signal object:

```js
const controller = new AbortController();
fetch(url, { signal: controller.signal })
  .catch(err => { if (err.name !== 'AbortError') throw err; });

controller.abort();       // request torn down, promise rejects with AbortError
```

### Sequential when you mean it

```js
// each step depends on the previous — a reduce over promises
const result = await steps.reduce(
  (chain, step) => chain.then(step),
  Promise.resolve(initial),
);
```

### Caching in-flight requests

Deduplicate concurrent callers by caching the **promise**, not the value:

```js
const cache = new Map();
function getUser(id) {
  if (!cache.has(id)) {
    cache.set(id, fetch(`/users/${id}`)
      .then(r => r.json())
      .catch(err => { cache.delete(id); throw err; }));  // don't cache failures
  }
  return cache.get(id);
}
```

---

## Gotchas

**Forgetting `return` in a `.then`.** The chain doesn't wait, the next handler gets `undefined`, and errors from the orphaned promise escape your `.catch`. Covered above because it's that common.

**`.then(fn())` instead of `.then(fn)`.** The first calls `fn` immediately and passes its return value — usually `undefined` — as the handler.

**Passing a method and losing `this`.** `p.then(obj.handle)` unbinds it. Use `p.then(v => obj.handle(v))` or `.bind(obj)`.

**`forEach` with async callbacks.** `forEach` ignores return values, so nothing is awaited and the loop finishes instantly:

```js
items.forEach(async item => { await save(item); });   // ❌ returns before any save
for (const item of items) await save(item);           // ✅ serial
await Promise.all(items.map(save));                   // ✅ parallel
```

**`await` inside a non-async callback.** `arr.map(x => await f(x))` is a syntax error; the callback itself needs `async`.

**Creating promises you never handle.** An async function called without `await` or `.catch` is an unhandled rejection waiting to happen. If it's genuinely fire-and-forget, say so: `void doThing().catch(report);`

**Rejecting with a non-Error.** `reject('failed')` gives you no stack trace. Always reject with an `Error`.

**`new Promise` around something that's already a promise.** The explicit-construction antipattern:

```js
// ❌
return new Promise((res, rej) => fetch(url).then(res).catch(rej));
// ✅
return fetch(url);
```

**Mixing `await` and `.then` in one expression.** It works, but pick one per function — mixed styles are where ordering bugs hide.

**Assuming `Promise.all` cancels siblings on failure.** It doesn't. They run to completion and their results are thrown away.

**Assuming a `.catch` in the middle stops the chain.** It recovers it. Everything after keeps running on the happy path unless you rethrow.

**A `try/catch` that doesn't `await`.** `try { return risky(); }` without `await` returns the promise before it can reject, so the `catch` never fires. `return await risky()` inside a `try` is the one place the "redundant await" lint rule is wrong.

---

## Building One From Scratch

A minimal Promises/A+-shaped implementation. Worth writing once — it makes the "then returns a new promise" rule concrete.

```js
class MyPromise {
  #state = 'pending';
  #value;
  #callbacks = [];

  constructor(executor) {
    const settle = (state, value) => {
      if (this.#state !== 'pending') return;      // settle exactly once
      this.#state = state;
      this.#value = value;
      queueMicrotask(() => this.#flush());        // always async
    };
    try {
      executor(v => settle('fulfilled', v), e => settle('rejected', e));
    } catch (err) {
      settle('rejected', err);                    // executor throws → reject
    }
  }

  #flush() {
    if (this.#state === 'pending') return;
    for (const cb of this.#callbacks) cb();
    this.#callbacks = [];
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const run = () => {
        const handler = this.#state === 'fulfilled' ? onFulfilled : onRejected;

        if (typeof handler !== 'function') {       // pass through
          return this.#state === 'fulfilled' ? resolve(this.#value) : reject(this.#value);
        }
        try {
          const result = handler(this.#value);
          // unwrap thenables — this is the flattening rule
          if (result && typeof result.then === 'function') result.then(resolve, reject);
          else resolve(result);
        } catch (err) {
          reject(err);                             // handler throws → reject
        }
      };

      if (this.#state === 'pending') this.#callbacks.push(run);
      else queueMicrotask(run);
    });
  }

  catch(onRejected)  { return this.then(undefined, onRejected); }
  finally(onSettled) {
    return this.then(
      v => { onSettled(); return v; },
      e => { onSettled(); throw e; },
    );
  }

  static resolve(v) { return v instanceof MyPromise ? v : new MyPromise(res => res(v)); }
  static reject(e)  { return new MyPromise((_, rej) => rej(e)); }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const values = [];
      let pending = 0, done = false;
      for (const [i, p] of [...promises].entries()) {
        pending++;
        MyPromise.resolve(p).then(v => {
          values[i] = v;                            // index, not push — order matters
          if (--pending === 0 && done) resolve(values);
        }, reject);                                 // first rejection wins
      }
      done = true;
      if (pending === 0) resolve(values);
    });
  }
}
```

The real spec has more: `resolve` adopting a thenable passed to it, cycle detection, and a `then` that must only be read once per object. But the shape above is what interviews are asking for.

---

## Summary

- A promise is a **one-way state machine**: pending → fulfilled *or* rejected, once, permanently.
- It solves callback hell (flat chains, one error path) and inversion of control (guarantees live in the object).
- `.then`/`.catch`/`.finally` each return a **new promise** — that's what makes chaining work.
- In a `.then` callback: **return a value** to pass it on, **return a promise** to wait for it, **throw** to reject. Forgetting the `return` is the classic bug.
- Promises never nest — returning one flattens it.
- `async`/`await` is the same machinery with `try/catch` syntax. An `async` function always returns a promise.
- Reactions run as **microtasks**, so they always beat `setTimeout(…, 0)`.
- `all` = all or nothing, `allSettled` = never rejects, `race` = first to settle, `any` = first to succeed.
- Promises are **eager** and **not cancellable** — they start on creation, and `AbortController` is the only way out.
- `fetch` doesn't reject on 4xx/5xx. Check `res.ok` yourself.
