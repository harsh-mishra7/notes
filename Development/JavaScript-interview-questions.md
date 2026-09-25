# Top 20 JavaScript Interview Questions

The ones that actually come up, with the answer an interviewer is listening for — not just the definition.

## Table of Contents
1. [var vs let vs const](#1-var-vs-let-vs-const)
2. [Hoisting and the Temporal Dead Zone](#2-hoisting-and-the-temporal-dead-zone)
3. [Closures](#3-closures)
4. [How does `this` work?](#4-how-does-this-work)
5. [call, apply, bind](#5-call-apply-bind)
6. [Arrow functions vs regular functions](#6-arrow-functions-vs-regular-functions)
7. [`==` vs `===` and type coercion](#7--vs--and-type-coercion)
8. [The event loop: microtasks vs macrotasks](#8-the-event-loop-microtasks-vs-macrotasks)
9. [Promises](#9-promises)
10. [async / await](#10-async--await)
11. [Prototypes and prototypal inheritance](#11-prototypes-and-prototypal-inheritance)
12. [Pass by value vs pass by reference](#12-pass-by-value-vs-pass-by-reference)
13. [Shallow copy vs deep copy](#13-shallow-copy-vs-deep-copy)
14. [null vs undefined vs NaN](#14-null-vs-undefined-vs-nan)
15. [Event bubbling, capturing, and delegation](#15-event-bubbling-capturing-and-delegation)
16. [Debouncing vs throttling](#16-debouncing-vs-throttling)
17. [map / filter / reduce / forEach](#17-map--filter--reduce--foreach)
18. [Spread, rest, and destructuring](#18-spread-rest-and-destructuring)
19. [Higher-order functions, currying, memoization](#19-higher-order-functions-currying-memoization)
20. [localStorage vs sessionStorage vs cookies](#20-localstorage-vs-sessionstorage-vs-cookies)

[Rapid-fire output puzzles](#rapid-fire-output-puzzles)

---

## 1. var vs let vs const

| | `var` | `let` | `const` |
|---|---|---|---|
| **Scope** | Function | Block | Block |
| **Hoisted** | Yes, initialized as `undefined` | Yes, but in the TDZ | Yes, but in the TDZ |
| **Redeclare in same scope** | Yes | No | No |
| **Reassign** | Yes | Yes | No |
| **Attaches to `window`** | Yes (at top level) | No | No |

```js
function scopes() {
  if (true) {
    var a = 1;
    let b = 2;
  }
  console.log(a); // 1   — var leaked out of the block
  console.log(b); // ReferenceError
}
```

**`const` does not mean immutable.** It means the *binding* can't be reassigned. The object it points at is still mutable:

```js
const user = { name: 'Ada' };
user.name = 'Grace';        // fine
user = {};                  // TypeError: Assignment to constant variable
```

Use `Object.freeze(user)` for a (shallow) immutable object.

**What they're testing:** whether you understand block scope. The classic follow-up:

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));  // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i));  // 0 1 2
```

`var` has one binding shared by all three callbacks. `let` creates a fresh binding per iteration.

**Rule:** `const` by default, `let` when you must reassign, `var` never.

---

## 2. Hoisting and the Temporal Dead Zone

**Hoisting** = declarations are registered when the scope is created, before any code runs. Only the *declaration* moves, not the assignment.

```js
console.log(x);      // undefined  — declared, not yet assigned
var x = 5;

console.log(y);      // ReferenceError: Cannot access 'y' before initialization
let y = 5;

greet();             // "hi" — function declarations are fully hoisted
function greet() { console.log('hi'); }

greetToo();          // TypeError: greetToo is not a function
var greetToo = function () {};   // only the var is hoisted
```

**Temporal Dead Zone (TDZ):** the window between entering a scope and the `let`/`const` declaration being executed. The variable exists but touching it throws. This is a *feature* — it turns a silent `undefined` bug into a loud error.

**Order of precedence:** function declarations are hoisted above `var` declarations in the same scope.

---

## 3. Closures

> A closure is a function that keeps access to the variables of the scope it was **created** in, even after that scope has returned.

```js
function counter() {
  let count = 0;                  // stays alive because the inner fn references it
  return {
    inc: () => ++count,
    get: () => count,
  };
}

const c = counter();
c.inc(); c.inc();
console.log(c.get());   // 2
console.log(c.count);   // undefined — genuinely private
```

**Why it matters in practice:**
- Private state (the module pattern, the example above)
- Every `debounce` / `throttle` / memoize implementation — the timer or cache lives in the closure
- Function factories: `const double = multiplier(2)`
- React hooks are closures; "stale closure" bugs are the #1 hooks gotcha

**Classic interview task — fix this:**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);   // 3 3 3
}

// Fix A: let
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 100);

// Fix B: IIFE capturing a copy (the pre-ES6 answer)
for (var i = 0; i < 3; i++) {
  (function (j) { setTimeout(() => console.log(j), 100); })(i);
}
```

**Downside:** closed-over variables can't be garbage collected while the closure lives. Long-lived closures holding big objects = memory leaks.

---

## 4. How does `this` work?

`this` is decided by **how a function is called**, not where it's defined. Check the rules in order:

1. **`new`** → `this` is the newly created object
2. **Explicit** (`call` / `apply` / `bind`) → whatever you passed
3. **Method call** `obj.fn()` → `obj` (whatever is left of the dot)
4. **Plain call** `fn()` → `undefined` in strict mode / modules, `globalThis` otherwise
5. **Arrow function** → no own `this`; it uses the enclosing scope's

```js
const user = {
  name: 'Ada',
  greet() { return `I am ${this.name}`; },
};

user.greet();                  // "I am Ada"

const detached = user.greet;
detached();                    // "I am undefined" — lost the receiver

detached.call(user);           // "I am Ada"
```

**The most-asked trap:**

```js
const obj = {
  name: 'Ada',
  regular() { setTimeout(function () { console.log(this.name); }, 0); },  // undefined
  arrow()   { setTimeout(() => console.log(this.name), 0); },             // "Ada"
};
```

The callback is invoked by the timer, not by `obj`, so rule 4 applies. The arrow function has no `this` of its own and reaches out to `arrow()`'s scope.

---

## 5. call, apply, bind

All three set `this` explicitly.

```js
function intro(greeting, punct) {
  return `${greeting}, I am ${this.name}${punct}`;
}

const user = { name: 'Ada' };

intro.call(user, 'Hello', '!');     // invokes now, args listed
intro.apply(user, ['Hello', '!']);  // invokes now, args as an array
const bound = intro.bind(user);     // returns a NEW function, doesn't invoke
bound('Hello', '!');
```

| | Invokes immediately | Arguments | Returns |
|---|---|---|---|
| `call` | Yes | Comma-separated | Function's result |
| `apply` | Yes | Array | Function's result |
| `bind` | No | Comma-separated (partial application allowed) | New bound function |

**Mnemonic:** **A** for **A**rray (apply), **C** for **C**omma (call), **B** for **B**ound-later (bind).

`bind` can also pre-fill arguments: `const hi = intro.bind(user, 'Hi')` → `hi('!')`.

A bound function **cannot be re-bound** — the first bind wins.

---

## 6. Arrow functions vs regular functions

| | Regular | Arrow |
|---|---|---|
| **Own `this`** | Yes (call-site) | No (lexical) |
| **`arguments` object** | Yes | No (use rest `...args`) |
| **Usable with `new`** | Yes | No |
| **Has `prototype`** | Yes | No |
| **Hoisted** | Declarations are | No |
| **Usable as a generator** | Yes | No |
| **Implicit return** | No | Yes, with a concise body |

```js
const add = (a, b) => a + b;              // implicit return
const makeUser = (name) => ({ name });    // object literal needs parens
```

**Don't use an arrow function for:**
- Object methods that need `this` → `const o = { n: 1, get: () => this.n }` is broken
- Prototype methods, constructors, class fields that rely on the instance via `new`
- DOM handlers where you want `this` to be the element

**Do use one for:** callbacks, array methods, anything nested inside a method that needs the outer `this`.

---

## 7. `==` vs `===` and type coercion

- `===` (strict): compares type **and** value. No coercion.
- `==` (loose): coerces the operands to a common type first.

```js
1 == '1'        // true
1 === '1'       // false
null == undefined   // true   (special-cased)
null === undefined  // false
NaN == NaN      // false — NaN is not equal to anything, use Number.isNaN()
[] == false     // true   ([] → '' → 0, false → 0)
'' == 0         // true
```

**The rule:** always use `===`. The one accepted use of `==` is `x == null`, which is a neat check for "null **or** undefined".

**Falsy values — memorise all eight:** `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is truthy, including `[]`, `{}`, `'0'`, and `'false'`.

**`typeof` quirks they love to ask:**

```js
typeof null          // "object"  — a famous, unfixable bug
typeof []            // "object"  — use Array.isArray()
typeof NaN           // "number"
typeof function(){}  // "function"
typeof undeclared    // "undefined" — doesn't throw
```

---

## 8. The event loop: microtasks vs macrotasks

JavaScript is **single-threaded** with one call stack. Anything async is handed to the host (browser/Node), which queues the callback back when it's ready.

**The loop:** run the call stack to empty → drain the **entire microtask queue** → take **one** macrotask → repeat (render in between, in browsers).

| Microtasks (higher priority) | Macrotasks |
|---|---|
| Promise `.then` / `.catch` / `.finally` | `setTimeout`, `setInterval` |
| `await` continuations | `setImmediate` (Node) |
| `queueMicrotask` | I/O, UI events |
| `MutationObserver` | `requestAnimationFrame`* |

The canonical question:

```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');

// 1, 4, 3, 2
```

Sync code first (`1`, `4`), then **all** microtasks (`3`), then the macrotask (`2`). `setTimeout(fn, 0)` does not mean "now" — it means "after the current stack and every pending microtask".

**Follow-up:** an infinite chain of microtasks starves the macrotask queue and freezes the page.

---

## 9. Promises

A Promise is an object representing a value that isn't available yet. It has three states: **pending → fulfilled** or **pending → rejected**. Once settled, it never changes.

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve('done'), 100);
});

p.then(v => v.toUpperCase())
 .then(v => console.log(v))
 .catch(err => console.error(err))
 .finally(() => console.log('cleanup'));
```

Each `.then` returns a **new** promise — that's what makes chaining work and what fixes callback hell (nesting becomes a flat chain).

**Combinators — know all four:**

| | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | All fulfil (array of results) | Any one rejects — fail fast |
| `Promise.allSettled` | All settle (`{status, value/reason}[]`) | Never |
| `Promise.race` | First to **settle**, either way | If the first to settle rejects |
| `Promise.any` | First to **fulfil** | Only if all reject (`AggregateError`) |

```js
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);  // parallel
```

**Gotcha:** a rejected promise with no `.catch` produces an unhandled rejection. And `.catch` only catches errors *above* it in the chain.

---

## 10. async / await

Syntax sugar over promises. An `async` function always returns a promise; `await` pauses that function until the promise settles, without blocking the thread.

```js
async function load() {
  try {
    const res = await fetch('/api/user');
    if (!res.ok) throw new Error(res.status);   // fetch only rejects on network failure
    return await res.json();
  } catch (err) {
    console.error(err);
    throw err;
  } finally {
    setLoading(false);
  }
}
```

**The performance mistake interviewers look for:**

```js
// ❌ Sequential — 2 seconds
const user = await fetchUser();     // 1s
const posts = await fetchPosts();   // 1s

// ✅ Parallel — 1 second
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
```

Only `await` in sequence when the second call genuinely needs the first result.

**Other points:** `await` in a `forEach` doesn't work (use `for...of` for sequential, or `map` + `Promise.all` for parallel); top-level `await` works in ES modules only.

---

## 11. Prototypes and prototypal inheritance

Every object has a hidden link (`[[Prototype]]`, reachable via `Object.getPrototypeOf` or the legacy `__proto__`) to another object. Miss a property lookup, and JS walks up that chain until it hits `null`.

```
myArray → Array.prototype → Object.prototype → null
```

```js
function Person(name) { this.name = name; }
Person.prototype.greet = function () { return `Hi, ${this.name}`; };

const p = new Person('Ada');
p.greet();                        // found on the prototype
p.hasOwnProperty('greet');        // false
Object.getPrototypeOf(p) === Person.prototype;  // true
```

`greet` lives in **one** place and is shared by every instance — that's the memory win over defining methods inside the constructor.

**`class` is syntax sugar over exactly this**, not a different inheritance model:

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}
class Dog extends Animal {
  speak() { return `${super.speak()} — woof`; }
}
```

**What `new` actually does:** creates `{}`, links its prototype to `Fn.prototype`, calls `Fn` with `this` = that object, returns it (unless the constructor returns its own object).

---

## 12. Pass by value vs pass by reference

JavaScript is **always pass by value** — but for objects, the value being copied is a *reference*.

```js
let a = 1;
function change(x) { x = 2; }
change(a);
console.log(a);          // 1 — primitives are copied

const obj = { n: 1 };
function mutate(o) { o.n = 2; }      // mutates the shared object
function reassign(o) { o = { n: 3 }; }  // rebinds the local copy only
mutate(obj);   console.log(obj.n);   // 2
reassign(obj); console.log(obj.n);   // 2, not 3
```

**Primitives** (string, number, boolean, null, undefined, symbol, bigint) are copied by value and are immutable. **Objects** (including arrays and functions) are copied by reference.

```js
const a = [1, 2];
const b = a;
b.push(3);
console.log(a);      // [1, 2, 3] — same array
console.log(a === b);// true
```

This is the root of most "why did my React state not update / update everywhere" bugs.

---

## 13. Shallow copy vs deep copy

A **shallow** copy duplicates the top level; nested objects are still shared. A **deep** copy duplicates everything.

```js
const original = { name: 'Ada', address: { city: 'London' } };

const shallow = { ...original };            // or Object.assign({}, original)
shallow.address.city = 'Paris';
console.log(original.address.city);         // "Paris" — leaked!

const deep = structuredClone(original);     // modern, built-in
deep.address.city = 'Berlin';
console.log(original.address.city);         // "Paris" — safe
```

| Method | Depth | Notes |
|---|---|---|
| `{...obj}` / `Object.assign` | Shallow | Fastest, most common |
| `arr.slice()` / `[...arr]` | Shallow | Arrays |
| `structuredClone(obj)` | Deep | Built-in; handles Date, Map, Set, cycles. **Drops functions** |
| `JSON.parse(JSON.stringify(obj))` | Deep | Loses functions, `undefined`, Symbols, Dates→strings, throws on cycles |
| `_.cloneDeep` (lodash) | Deep | Handles everything, extra dependency |

Reach for `structuredClone` first; the JSON trick is the old answer and worth naming as a trap.

---

## 14. null vs undefined vs NaN

| | Meaning | Set by | `typeof` |
|---|---|---|---|
| `undefined` | Not assigned yet | The engine | `"undefined"` |
| `null` | Intentionally empty | You | `"object"` (legacy bug) |
| `NaN` | An invalid number result | Failed math | `"number"` |

```js
let a;                         // undefined
let b = null;                  // deliberately no value
0 / 0;                         // NaN
Number('abc');                 // NaN

null == undefined              // true
null === undefined             // false
Number.isNaN(NaN)              // true  — prefer over the global isNaN
```

**Default parameters and `??` only trigger on `undefined`** (and `null`, for `??`), not on other falsy values:

```js
function f(x = 10) { return x; }
f(undefined);   // 10
f(null);        // null
f(0);           // 0

0 || 'fallback'    // "fallback"   — || catches all falsy
0 ?? 'fallback'    // 0            — ?? only catches null/undefined
```

Optional chaining pairs with it: `user?.address?.city ?? 'unknown'`.

---

## 15. Event bubbling, capturing, and delegation

An event travels in three phases: **capture** (window → target), **target**, then **bubble** (target → window). Handlers run in the bubble phase by default; pass `{ capture: true }` for the other one.

```js
parent.addEventListener('click', handler);                  // bubble
parent.addEventListener('click', handler, { capture: true });  // capture
```

- `e.stopPropagation()` — stop travelling further up/down
- `e.preventDefault()` — cancel the default browser action (form submit, link nav)
- `e.target` — where it actually happened; `e.currentTarget` — where the listener sits

**Event delegation:** put one listener on a common ancestor instead of one per child.

```js
document.querySelector('#list').addEventListener('click', (e) => {
  const item = e.target.closest('li');
  if (!item) return;
  console.log('clicked', item.dataset.id);
});
```

**Why it matters:** one listener instead of hundreds (memory), and it works for elements added to the DOM *later* — no re-binding. React has always used a form of this internally.

---

## 16. Debouncing vs throttling

- **Debounce** — wait until the calls stop, then run once. Every call resets the timer.
- **Throttle** — run at most once per N ms, however many calls arrive.

```js
const debounce = (fn, ms = 300) => {
  let t;
  return (...a) => { clearTimeout(t); t = setTimeout(() => fn(...a), ms); };
};

const throttle = (fn, ms = 200) => {
  let last = 0;
  return (...a) => {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn(...a); }
  };
};
```

Debounce a search box (you only want the final query). Throttle a scroll handler (you need updates *while* it scrolls).

→ Full write-up with leading/trailing edges, React hooks and gotchas: [Debouncing-throttling.md](Debouncing-throttling.md)

---

## 17. map / filter / reduce / forEach

| Method | Returns | Use it for |
|---|---|---|
| `forEach` | `undefined` | Side effects; cannot be chained or broken out of |
| `map` | New array, same length | Transform every item |
| `filter` | New array, ≤ length | Keep items matching a predicate |
| `reduce` | Any single value | Fold the array into one thing |
| `find` / `findIndex` | First match / its index | Lookup, stops early |
| `some` / `every` | Boolean | "Any?" / "All?", stop early |

```js
const orders = [{ total: 20 }, { total: 30 }, { total: 50 }];

const revenue = orders.reduce((sum, o) => sum + o.total, 0);   // 100
```

Always pass the initial value to `reduce` — without it, the first element becomes the accumulator and an empty array throws.

`reduce` can implement the others; a group-by is the usual live-coding ask:

```js
const groupBy = (arr, key) =>
  arr.reduce((acc, item) => {
    (acc[item[key]] ||= []).push(item);
    return acc;
  }, {});
```

**Mutating vs non-mutating — the classic follow-up:** `splice`, `sort`, `reverse`, `push`, `pop`, `shift`, `unshift`, `fill` mutate. `slice`, `map`, `filter`, `concat`, `toSorted`, `toReversed` return a new array. `slice(1, 3)` copies; `splice(1, 3)` cuts three items out in place.

---

## 18. Spread, rest, and destructuring

Same `...` token, opposite jobs: **spread expands**, **rest collects**.

```js
// spread — expand
const merged = { ...defaults, ...overrides };   // later keys win
const copy   = [...arr];
Math.max(...numbers);

// rest — collect
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); }
const { id, ...others } = user;                 // pull one key out, keep the rest
```

Destructuring, with defaults, renaming and nesting:

```js
const { name: userName = 'anon', address: { city } = {} } = user;
const [first, , third = 0] = arr;
[a, b] = [b, a];                                 // swap
```

**Remember:** spread is a **shallow** copy (see #13), and it's the everyday way to update state immutably:

```js
setUser(prev => ({ ...prev, name: 'Grace' }));
```

---

## 19. Higher-order functions, currying, memoization

A **higher-order function** takes a function as an argument, returns one, or both. `map`, `filter`, `setTimeout`, `debounce` — all HOFs.

**Currying** turns `f(a, b, c)` into `f(a)(b)(c)`, so you can fix arguments early:

```js
const multiply = (a) => (b) => a * b;
const double = multiply(2);
double(5);    // 10
```

Generic version — a common whiteboard question:

```js
function curry(fn) {
  return function curried(...args) {
    return args.length >= fn.length
      ? fn.apply(this, args)
      : (...more) => curried.apply(this, [...args, ...more]);
  };
}
```

**Memoization** caches results by argument — a HOF built on a closure:

```js
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
}
```

Only safe for **pure** functions (same input → same output, no side effects), and watch the cache growing without bound.

---

## 20. localStorage vs sessionStorage vs cookies

| | localStorage | sessionStorage | Cookies |
|---|---|---|---|
| **Capacity** | ~5–10 MB | ~5–10 MB | ~4 KB |
| **Expires** | Never (until cleared) | When the tab closes | Set by `Expires` / `Max-Age` |
| **Sent to server** | No | No | Yes, on every request |
| **Scope** | Origin, all tabs | That tab only | Origin + path |
| **API** | Sync, string only | Sync, string only | `document.cookie` string |

```js
localStorage.setItem('theme', JSON.stringify({ mode: 'dark' }));
const theme = JSON.parse(localStorage.getItem('theme') ?? 'null');
```

**Points that score:** all three are readable by JavaScript, so **never store a JWT or anything secret in localStorage** — an XSS bug reads it instantly. Auth tokens belong in an `HttpOnly`, `Secure`, `SameSite` cookie, which JS cannot touch. Storage APIs are synchronous and block the main thread, so keep the payloads small; use IndexedDB for anything large or structured.

---

## Rapid-fire output puzzles

```js
console.log(0.1 + 0.2 === 0.3);   // false — IEEE-754 floats. Compare with an epsilon.
console.log(typeof NaN);          // "number"
console.log([] + {});             // "[object Object]"
console.log([1,2,3] + [4,5]);     // "1,2,34,5"  — both stringify
console.log([10, 9, 1].sort());   // [1, 10, 9] — default sort is lexicographic
console.log('5' - 3, '5' + 3);    // 2, "53"   — minus coerces, plus concatenates
```

```js
// Sequence — combines the event loop and async/await
async function f() {
  console.log('1');
  await null;
  console.log('2');
}
console.log('0'); f(); console.log('3');
// 0, 1, 3, 2   — everything after the first await is a microtask
```

```js
// this + arrow, one more time
const o = {
  n: 42,
  a: () => this.n,        // undefined — no own this
  b() { return this.n; }, // 42
};
```

**Other things worth having a one-liner ready for:** `use strict`, IIFEs, generators (`function*`), `Map` vs object, `Set` for deduping, `WeakMap` for private/GC-friendly data, ES modules vs CommonJS, `Object.freeze`, symbols, and the difference between `defer` and `async` on a `<script>` tag.
