# Top 10 Node.js Interview Questions

The ones that actually come up, with the answer an interviewer is listening for — not just the definition.

## Table of Contents
1. [What is Node.js, and what makes it "non-blocking"?](#1-what-is-nodejs-and-what-makes-it-non-blocking)
2. [Is Node.js really single-threaded?](#2-is-nodejs-really-single-threaded)
3. [The Node event loop: phases, nextTick vs setImmediate](#3-the-node-event-loop-phases-nexttick-vs-setimmediate)
4. [What blocks the event loop, and how do you fix it?](#4-what-blocks-the-event-loop-and-how-do-you-fix-it)
5. [Streams, buffers, and backpressure](#5-streams-buffers-and-backpressure)
6. [CommonJS vs ES Modules](#6-commonjs-vs-es-modules)
7. [Error handling in Node](#7-error-handling-in-node)
8. [Middleware and the Express request lifecycle](#8-middleware-and-the-express-request-lifecycle)
9. [Scaling Node: cluster, child_process, worker_threads](#9-scaling-node-cluster-child_process-worker_threads)
10. [Memory leaks and performance debugging](#10-memory-leaks-and-performance-debugging)

[Rapid-fire questions](#rapid-fire-questions)

---

## 1. What is Node.js, and what makes it "non-blocking"?

Node is a **runtime**, not a framework and not a language: Google's **V8** engine to execute JavaScript, plus **libuv** (a C library) for the event loop, the thread pool and cross-platform async I/O, plus a standard library (`fs`, `http`, `net`, `crypto`, …) and the module system.

"Non-blocking" is about **who waits**. A classic thread-per-request server parks a whole OS thread on a slow query. Node hands the I/O to the kernel (`epoll` on Linux, `kqueue` on macOS, IOCP on Windows), returns immediately, and the thread goes on to serve other requests. When the kernel says the data is ready, the event loop runs your callback.

```js
// Blocking — the whole process stops here
const data = fs.readFileSync('big.json');
console.log('read');

// Non-blocking — the loop is free while the disk works
fs.readFile('big.json', (err, data) => console.log('read'));
console.log('this prints first');
```

**Points that score:** this is why Node is excellent for **I/O-bound** work (APIs, proxies, real-time, streaming) and poor for **CPU-bound** work (image processing, big crypto, heavy JSON transforms) — the latter occupies the one thread that everyone else is queued behind. Concurrency here is *interleaving*, not parallelism.

---

## 2. Is Node.js really single-threaded?

**Your JavaScript is single-threaded. The process is not.** A Node process has:

| | Runs on | Examples |
|---|---|---|
| **Main thread** | One V8 isolate, one event loop | All your JS, all callbacks |
| **libuv thread pool** | 4 threads by default (`UV_THREADPOOL_SIZE`, max 1024) | `fs.*`, `dns.lookup`, `crypto.pbkdf2`, `zlib` |
| **Kernel async I/O** | No thread at all | TCP/HTTP sockets, `dns.resolve` |
| **Worker threads** | Separate V8 isolates you create | `worker_threads` for CPU work |

The common misconception is that *all* async work uses the thread pool. Network I/O does not — it is event-driven in the kernel, which is why one Node process can hold tens of thousands of sockets. But four parallel `pbkdf2` calls will saturate the pool and a fifth waits.

```js
// All four land on the thread pool; the 5th queues behind them
for (let i = 0; i < 5; i++) {
  crypto.pbkdf2('pw', 'salt', 1e6, 64, 'sha512', () => console.log(i, Date.now()));
}
```

---

## 3. The Node event loop: phases, nextTick vs setImmediate

Each turn ("tick") of the loop walks through phases in order:

1. **timers** — expired `setTimeout` / `setInterval` callbacks
2. **pending callbacks** — some deferred system callbacks (e.g. TCP errors)
3. **idle / prepare** — internal
4. **poll** — wait for and run I/O callbacks (this is where the loop blocks when idle)
5. **check** — `setImmediate` callbacks
6. **close callbacks** — `socket.on('close')` and friends

**Between every phase** (and after each individual callback) Node drains two microtask queues, in this order:

1. `process.nextTick` queue
2. Promise / `queueMicrotask` queue

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');

// sync → nextTick → promise → timeout → immediate
```

**Points that score:** `setTimeout(fn, 0)` is really "at least 1 ms, in the next timers phase"; `setImmediate` means "end of this turn, in the check phase". Both microtask queues starve the loop if you recurse in them — an infinite `process.nextTick` chain never lets I/O run again, whereas an infinite `setImmediate` chain still yields between turns. Inside an I/O callback, `setImmediate` always fires before `setTimeout(…, 0)`; at the top level the order is non-deterministic, because it depends on how long the loop took to start.

See [Event-loop.md](Event-loop.md) for the browser version of the same machinery.

---

## 4. What blocks the event loop, and how do you fix it?

Anything synchronous and slow: `readFileSync` in a request handler, `JSON.parse` on a 50 MB payload, `bcrypt.hashSync`, a regex with catastrophic backtracking, a `for` loop over a million rows, `console.log` to a slow pipe.

While it runs, **every** other request is frozen — the symptom is p99 latency spiking for unrelated endpoints.

Fixes, in order of preference:

- Use the async API (`fs.promises`, `crypto.pbkdf2`, `bcrypt.hash`) so the work goes to the thread pool.
- Move real CPU work to a **worker thread** or a separate service/queue.
- **Chunk** unavoidable loops with `setImmediate` so the loop breathes between batches.
- **Stream** instead of buffering (see below).

```js
function processInChunks(rows, done, i = 0) {
  const end = Math.min(i + 1000, rows.length);
  for (; i < end; i++) transform(rows[i]);
  if (i < rows.length) setImmediate(() => processInChunks(rows, done, i));
  else done();
}
```

Measure it with `--prof`, `clinic doctor`, or `perf_hooks.monitorEventLoopDelay()` — event loop lag is the single most useful Node metric to alert on.

---

## 5. Streams, buffers, and backpressure

A **Buffer** is a fixed-length chunk of raw bytes outside the V8 heap — Node's way of holding binary data. A **stream** processes data in chunks as it arrives, instead of loading it all into memory.

Four types: **Readable**, **Writable**, **Duplex** (a TCP socket), **Transform** (`zlib.createGzip()`).

```js
// Buffers the entire file in memory — dies on a 2 GB file
const data = await fs.promises.readFile('big.csv');
res.end(data);

// Constant memory, starts sending immediately
pipeline(
  fs.createReadStream('big.csv'),
  zlib.createGzip(),
  res,
  err => err && console.error(err)
);
```

**Backpressure** is the answer they're waiting for. `writable.write()` returns `false` when its internal buffer is past `highWaterMark`; if you ignore that and keep writing, memory grows until the process dies. You either wait for the `'drain'` event, or let `pipe`/`pipeline` handle it for you. Prefer **`stream.pipeline`** over `.pipe()` — it propagates errors and destroys every stream in the chain, which `.pipe()` does not.

---

## 6. CommonJS vs ES Modules

| | CommonJS | ES Modules |
|---|---|---|
| **Syntax** | `require` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous, at runtime | Asynchronous, statically analysed |
| **Enabled by** | Default, or `.cjs` | `"type": "module"`, or `.mjs` |
| **Bindings** | A copy of the value | A live binding to the export |
| **Dynamic path** | `require(someVar)` works | Needs `await import(someVar)` |
| **Top-level await** | No | Yes |
| **Tree-shaking** | Hard | Yes |
| **`__dirname`** | Available | Use `import.meta.dirname` |

An ESM file can `import` a CJS file; a CJS file cannot `require` an ESM one (use dynamic `import()`). Note that recent Node versions do allow `require()` of an ESM module that has no top-level `await`, so check the version before making a hard claim.

**The caching point:** `require` caches by resolved filename, so a module's top-level code runs **once** per process and every importer shares the same object. That's the idiomatic singleton in Node (a DB pool, a config object) — and also why mutating another module's exports is such a nasty source of bugs.

```js
// db.js — one pool for the whole process, no matter how many files require it
const pool = new Pool(process.env.DATABASE_URL);
module.exports = pool;
```

---

## 7. Error handling in Node

Four channels, and mixing them up is the most common real-world Node bug:

```js
// 1. Callbacks — error is the first argument
fs.readFile(p, (err, data) => { if (err) return cb(err); /* … */ });

// 2. Promises / async-await — try/catch
try { await getUser(id); } catch (err) { /* … */ }

// 3. EventEmitters — the 'error' event. Unhandled, it CRASHES the process
stream.on('error', err => console.error(err));

// 4. Sync throws — plain try/catch
```

**The trap:** `try/catch` cannot catch an error thrown from an async callback, because the surrounding frame is long gone.

```js
try {
  setTimeout(() => { throw new Error('boom'); }, 0);   // uncaught, process dies
} catch (e) { /* never runs */ }
```

Process-level safety nets:

```js
process.on('unhandledRejection', err => { log(err); process.exit(1); });
process.on('uncaughtException',  err => { log(err); process.exit(1); });
```

Use them to **log and exit**, not to keep running — after an uncaught exception the process state is unknown. Let a supervisor (`pm2`, systemd, Kubernetes) restart it, and drain in-flight requests first for a graceful shutdown.

**Points that score:** distinguish *operational* errors (network timeout, bad input, 404 — handle and respond) from *programmer* errors (`undefined is not a function` — crash and fix). Always `throw new Error()`, never a string, so you keep a stack. In Express 4, async handler errors must be passed to `next(err)` or wrapped; Express 5 forwards rejected promises automatically.

---

## 8. Middleware and the Express request lifecycle

Middleware is a function `(req, res, next)` that sits in an ordered chain. Each one can respond, mutate `req`/`res`, or call `next()` to pass control along. **Order is everything** — this is usually the real question.

```js
app.use(express.json());                       // 1. parse the body
app.use(morgan('dev'));                        // 2. log
app.use('/api', rateLimiter);                  // 3. path-scoped
app.get('/me', auth, (req, res) => res.json(req.user));   // 4. route-level
app.use((req, res) => res.status(404).send('Not found')); // 5. no route matched
app.use((err, req, res, next) => {             // 6. error handler — 4 args
  res.status(err.status ?? 500).json({ error: err.message });
});
```

Things interviewers probe: an error-handling middleware is identified by its **four** arguments and must be registered **last**; forgetting `next()` hangs the request forever; `express.json()` must come before any handler that reads `req.body`; and a route that already sent a response must not call `next()`.

---

## 9. Scaling Node: cluster, child_process, worker_threads

| | Use for | Memory | Communication |
|---|---|---|---|
| **`cluster`** | Using all CPU cores for one HTTP server | Separate process each | IPC, messages |
| **`child_process`** | Running another program (`ffmpeg`, a shell script) | Separate process | stdio, IPC |
| **`worker_threads`** | CPU-bound JS inside one process | Shared process, separate heaps | `postMessage`, `SharedArrayBuffer` |

```js
if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
  cluster.on('exit', () => cluster.fork());        // restart dead workers
} else {
  app.listen(3000);   // all workers share the port; the OS/primary balances
}
```

**Points that score:** workers do not share memory, so anything stateful must move out of the process — sessions to Redis, in-memory caches to Redis, WebSocket rooms to a pub/sub adapter, timers/cron to a single scheduler or a job queue. In containers, one process per container plus a horizontal autoscaler is usually preferred over `cluster` inside the container. `worker_threads` are for CPU, not for I/O — spawning a thread for a database call makes things slower, not faster.

---

## 10. Memory leaks and performance debugging

Node's heap is capped (`--max-old-space-size`), so a leak ends in an OOM crash. The usual sources:

- A module-level array or `Map` that only ever grows — a home-made cache with no eviction. Use an LRU with a size bound, or `WeakMap`/`WeakRef` for object-keyed data.
- Listeners added per request and never removed (`emitter.on` inside a handler) — the *"MaxListenersExceededWarning"* is the tell.
- Closures capturing big objects, kept alive by a long-lived timer or a pending promise.
- Unbounded concurrency: `Promise.all` over 100k items holds 100k results in memory. Batch it, or use a concurrency limiter.

How to find one, in order: watch RSS and heap over time (`process.memoryUsage()`, `/metrics`); take two **heap snapshots** under load with `--inspect` + Chrome DevTools; compare and look at what grew. For CPU, use the flame graph from `node --cpu-prof` or `clinic flame`. For latency, `monitorEventLoopDelay()`.

**Quick wins worth naming:** `gzip`/`brotli` compression, HTTP keep-alive on outbound agents, connection **pooling** instead of a connection per request, a cache in front of hot reads, and pagination instead of `SELECT *`.

---

## Rapid-fire questions

- **`package.json` vs `package-lock.json`** — declared ranges vs the exact resolved tree. Commit both; use `npm ci` in CI for a reproducible install.
- **`dependencies` vs `devDependencies`** — shipped at runtime vs needed only to build and test.
- **`npm` vs `npx`** — install vs execute a package binary without installing it globally.
- **Globals in CJS** — `process`, `Buffer`, `__dirname`, `__filename`, `require`, `module`, `exports`. `__dirname` is not available in ESM.
- **`process.env`** — config in, secrets out of git. Validate it at boot and fail fast on a missing variable.
- **`EventEmitter`** — Node's pub/sub base class: `on`, `once`, `emit`, `off`. Streams, `http.Server` and `process` are all emitters.
- **`path.join` vs `path.resolve`** — join concatenates segments; resolve produces an absolute path, restarting at any absolute segment.
- **`fs.promises`** — the async API without the callbacks. Use it over the `*Sync` variants everywhere but startup scripts.
- **`process.exit()`** — kills the process immediately, truncating pending writes. Prefer letting the loop drain, or use `exitCode`.
- **Sequential vs parallel awaits** — two `await`s in a row are serial; `Promise.all([a, b])` overlaps them. Classic follow-up: `Promise.all` vs `allSettled` vs `race` vs `any`.
- **Security basics** — `helmet`, CORS configured explicitly, rate limiting, parameterized queries, `bcrypt`/`argon2` for passwords, `HttpOnly` + `Secure` + `SameSite` cookies for tokens, and `npm audit` in CI.
