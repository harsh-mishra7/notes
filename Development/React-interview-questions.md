# Top 10 React Interview Questions

The ones that actually come up, with the answer an interviewer is listening for — not just the definition.

## Table of Contents
1. [The Virtual DOM and reconciliation](#1-the-virtual-dom-and-reconciliation)
2. [Props vs state (and why `setState` looks async)](#2-props-vs-state-and-why-setstate-looks-async)
3. [What are hooks, and the Rules of Hooks](#3-what-are-hooks-and-the-rules-of-hooks)
4. [`useEffect`: dependencies, cleanup, and the usual bugs](#4-useeffect-dependencies-cleanup-and-the-usual-bugs)
5. [Keys in lists](#5-keys-in-lists)
6. [Controlled vs uncontrolled components](#6-controlled-vs-uncontrolled-components)
7. [`React.memo`, `useMemo`, `useCallback`](#7-reactmemo-usememo-usecallback)
8. [`useRef` and refs](#8-useref-and-refs)
9. [Context, prop drilling, and when you actually need Redux](#9-context-prop-drilling-and-when-you-actually-need-redux)
10. [Custom hooks](#10-custom-hooks)

[Rapid-fire round](#rapid-fire-round)

---

## 1. The Virtual DOM and reconciliation

The Virtual DOM is a plain JavaScript tree describing what the UI *should* look like. On every render React builds a new tree, **diffs** it against the previous one, and applies the minimum set of real DOM mutations. That diff-and-commit step is **reconciliation**.

```jsx
<h1 className="title">Hi</h1>
// is just:
React.createElement('h1', { className: 'title' }, 'Hi')
// → { type: 'h1', props: { className: 'title', children: 'Hi' } }
```

The diff is O(n) instead of O(n³) because React makes two assumptions:

1. **Different element types produce different trees.** `<div>` → `<span>` means unmount the whole subtree and rebuild it. State inside is lost.
2. **Keys tell React which children are the same across renders** (see [#5](#5-keys-in-lists)).

**What they're testing:** whether you think "Virtual DOM = fast". It isn't inherently faster than hand-written DOM code — a targeted `textContent` update always wins. What it buys you is a *declarative* model: you describe the end state, React works out the mutations. Batching many changes into one commit is the real performance story.

**Follow-up — render vs commit.** The **render phase** (calling your components, diffing) is pure and interruptible; with concurrent rendering React can pause and throw it away. The **commit phase** (mutating the DOM, running layout effects) is synchronous and never interrupted. This is exactly why your render function must have no side effects — it may run twice and get discarded.

---

## 2. Props vs state (and why `setState` looks async)

| | Props | State |
|---|---|---|
| **Owner** | The parent | The component itself |
| **Mutable** | No — read-only | Yes, via the setter |
| **Triggers re-render** | When the parent re-renders | When the setter is called with a new value |
| **Analogy** | Function arguments | Local variables that survive renders |

**State updates are not applied immediately.** The setter queues an update; `state` is a constant for the whole render pass.

```jsx
const [count, setCount] = useState(0);

function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  console.log(count);   // 0 — this render's value, not the new one
}                       // result: 1, not 2
```

Fix it with the **updater form**, which receives the latest queued value:

```jsx
setCount(c => c + 1);
setCount(c => c + 1);   // result: 2
```

**Batching:** React groups multiple `setState` calls into a single re-render. Since React 18 this is *automatic batching* — it applies inside promises, `setTimeout` and native event handlers too, not just React events.

**Immutability:** never mutate state. React compares with `Object.is`, so a mutated object is the same reference and nothing re-renders.

```jsx
todos.push(newTodo); setTodos(todos);      // ❌ same reference, no render
setTodos([...todos, newTodo]);             // ✅
setUser({ ...user, name: 'Grace' });       // ✅
```

---

## 3. What are hooks, and the Rules of Hooks

Hooks let function components use state, side effects and context — things that previously needed a class. They arrived in React 16.8 and solved three real problems: reusing stateful logic without HOC/render-prop wrapper hell, related logic being split across `componentDidMount`/`componentDidUpdate`/`componentWillUnmount`, and `this` binding.

**The two rules:**

1. **Only call hooks at the top level.** Not inside conditions, loops, or nested functions.
2. **Only call them from React function components or other hooks.**

**Why rule 1 exists** — this is the answer that scores. React stores hook state in a **linked list per component, indexed by call order**. There are no names, only positions. Skip a hook on one render and every subsequent hook reads the wrong slot:

```jsx
if (isLoggedIn) {
  const [name, setName] = useState('');   // ❌ slot 0 exists only sometimes
}
const [theme, setTheme] = useState('dark'); // reads slot 0 or 1 depending on the branch
```

Put the condition *inside* the hook instead: `useEffect(() => { if (isLoggedIn) … })`.

**Common hooks worth naming:** `useState`, `useEffect`, `useContext`, `useRef`, `useMemo`, `useCallback`, `useReducer`, `useLayoutEffect`, and from React 18+, `useTransition`, `useDeferredValue`, `useId`, `useSyncExternalStore`.

---

## 4. `useEffect`: dependencies, cleanup, and the usual bugs

`useEffect` runs *after* the render is committed to the screen. It's for synchronizing with something outside React: network, subscriptions, timers, the DOM, localStorage.

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);   // cleanup: before the next run and on unmount
}, [delay]);
```

| Dependency array | When the effect runs |
|---|---|
| omitted | After **every** render |
| `[]` | Once after mount (cleanup on unmount) |
| `[a, b]` | On mount, and whenever `a` or `b` change by `Object.is` |

**The four bugs interviewers fish for:**

- **Missing dependency → stale closure.** The effect captured the first render's variables and keeps using them. Trust the exhaustive-deps lint rule; if a dep churns, fix it with the updater form or `useCallback`, don't delete it from the array.
- **Infinite loop.** The effect sets state that is also a dependency, or a dep is an object/array literal recreated every render (`{}` !== `{}`).
- **No cleanup → race condition.** Two fetches resolve out of order and the stale one wins:

  ```jsx
  useEffect(() => {
    let cancelled = false;
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(d => { if (!cancelled) setUser(d); });
    return () => { cancelled = true; };
  }, [id]);
  ```

- **Effects you don't need.** Deriving state from props in an effect causes a double render — just compute it during render, or `useMemo` it if it's expensive.

**`useEffect` vs `useLayoutEffect`:** `useLayoutEffect` fires synchronously after DOM mutation but *before* the browser paints. Use it only to read layout and adjust it (measuring, tooltips) so the user never sees the intermediate frame. It blocks painting, so `useEffect` is the default.

**StrictMode:** in development React 18+ mounts, unmounts and remounts every component once, so effects run twice. That's a feature designed to surface missing cleanup — not a bug to work around.

---

## 5. Keys in lists

A key tells React which item in a list is which across renders, so it can move DOM nodes and component state instead of destroying and recreating them.

```jsx
{todos.map(todo => <Todo key={todo.id} {...todo} />)}   // ✅ stable, unique identity
{todos.map((todo, i) => <Todo key={i} {...todo} />)}    // ⚠️ index
```

**Why `key={index}` is a bug:** insert an item at the front and every index shifts. React thinks item 0 changed its text rather than that a new item appeared, so it reuses the wrong DOM node — internal state (input values, checkboxes, focus, CSS transitions) attaches to the wrong row.

Index keys are fine only when the list is static, never reordered or filtered, and has no per-item state.

**Rules:** keys must be unique *among siblings* (not globally), stable across renders (never `Math.random()`), and belong on the element returned by `map`, not on a child inside it.

**The trick worth mentioning:** changing a key deliberately **resets a component**. `<Profile key={userId} />` throws away all internal state when the user changes — cleaner than an effect that resets it.

---

## 6. Controlled vs uncontrolled components

| | Controlled | Uncontrolled |
|---|---|---|
| **Source of truth** | React state | The DOM node |
| **Read the value** | From state, always current | From a ref, on demand |
| **Setup** | `value` + `onChange` | `defaultValue` + `ref` |
| **Good for** | Validation as you type, formatting, disabling submit, dependent fields | Simple forms, file inputs, integrating non-React widgets |

```jsx
// Controlled
const [email, setEmail] = useState('');
<input value={email} onChange={e => setEmail(e.target.value)} />

// Uncontrolled
const emailRef = useRef(null);
<input defaultValue="" ref={emailRef} />   // read emailRef.current.value on submit
```

**Points that score:** `<input type="file">` is always uncontrolled — its value is read-only for security. Passing `value` without `onChange` makes the field read-only and logs a warning; use `readOnly` or `defaultValue` if that's what you meant. Switching a field from uncontrolled to controlled mid-life (`value={undefined}` then a string) is a classic React warning. Controlled inputs re-render on every keystroke, which is why big forms often reach for `react-hook-form`, which keeps them uncontrolled.

---

## 7. `React.memo`, `useMemo`, `useCallback`

All three are caches. They trade memory and complexity for skipped work — reach for them when you have measured a problem, not by default.

| | Memoizes | Use when |
|---|---|---|
| `React.memo(Component)` | The rendered output | A child re-renders with identical props and its render is expensive |
| `useMemo(fn, deps)` | A **value** | The computation is expensive, or the value is a dep/prop that must keep its identity |
| `useCallback(fn, deps)` | A **function** | The function is a dep of an effect, or a prop to a `React.memo` child |

```jsx
const sorted = useMemo(() => rows.sort(cmp), [rows]);          // value
const onSelect = useCallback(id => setPicked(id), []);         // stable identity
export default React.memo(Row);                                // skips re-render
```

**The trap:** `React.memo` does a **shallow** prop comparison, so one inline object or arrow function in the parent defeats it entirely.

```jsx
<Row onClick={() => pick(id)} style={{ margin: 4 }} />   // new refs every render — memo never hits
```

That's the real reason `useCallback` and `useMemo` exist: not raw speed, but **referential stability**.

**What they're testing:** whether you know the cost. Every memo hook stores a value and runs a deps comparison on each render, so wrapping cheap components makes the app slower. Also mention the alternative that needs no hooks at all — *move state down*, or pass expensive subtrees as `children` so they aren't re-created by the parent's re-render. And note that the React Compiler is designed to make most manual memoization unnecessary.

---

## 8. `useRef` and refs

`useRef` returns a mutable `{ current }` box that **persists across renders and does not trigger a re-render when it changes**.

Two uses:

```jsx
// 1. Reach a DOM node
const inputRef = useRef(null);
useEffect(() => inputRef.current.focus(), []);
<input ref={inputRef} />

// 2. An instance variable — survives renders, invisible to rendering
const timerId = useRef(null);
const renderCount = useRef(0);
renderCount.current++;            // no re-render
```

**state vs ref:** if the value is shown in the UI, it's state. If it's bookkeeping — a timer id, the previous value, a scroll position, a "has this already run" flag — it's a ref. Never read or write `ref.current` during render; that's a side effect, and with concurrent rendering the result is undefined. Refs are for event handlers and effects.

**Follow-ups:** callback refs (`ref={node => …}`) when you need to run code as the node attaches or detaches; `forwardRef` to pass a ref through a component (in React 19 `ref` is just a regular prop for function components, so `forwardRef` is no longer needed); `useImperativeHandle` to expose a narrow API like `{ focus, scrollTo }` instead of the raw node.

---

## 9. Context, prop drilling, and when you actually need Redux

**Prop drilling** is threading a prop through components that don't use it. **Context** lets a provider broadcast a value to any descendant that asks for it.

```jsx
const ThemeContext = createContext('light');

<ThemeContext.Provider value={theme}>   // React 19: <ThemeContext value={theme}>
  <App />
</ThemeContext.Provider>

const theme = useContext(ThemeContext);  // any depth below
```

**The catch:** every consumer re-renders when the context value changes, and an inline object (`value={{ user, setUser }}`) is a new reference on every provider render, so *everything* re-renders. Memoize the value, or split rarely-changing state and frequently-changing state into separate contexts.

Context is a **dependency-injection** mechanism, not a state manager — it has no reducers, middleware, devtools or selective subscriptions.

**Choosing:**

| Need | Reach for |
|---|---|
| One component's state | `useState` |
| Several fields updated together, complex transitions | `useReducer` |
| Low-frequency global values: theme, locale, current user | Context |
| Server data: caching, refetching, loading/error states | React Query / SWR |
| Large, high-frequency, shared client state | Redux Toolkit, Zustand, Jotai |

**`useState` vs `useReducer`:** move to a reducer when the next state depends on several pieces of the old state, when updates are spread over many handlers, or when you want the update logic unit-testable and outside the component.

---

## 10. Custom hooks

A custom hook is just a function whose name starts with `use` and that calls other hooks. It exists to **share stateful logic**, not state — each call site gets its own independent state.

```jsx
function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}

// usage
const query = useDebouncedValue(input, 400);   // fires the search only when typing pauses
```

**What they're testing:** that you know two components calling `useDebouncedValue` do **not** share a value — the hook shares the *logic*. And that the `use` prefix isn't cosmetic: it's how the linter knows to apply the Rules of Hooks.

**Before hooks**, the same job was done with **HOCs** (`withRouter(Component)`) and **render props** (`<Mouse render={pos => …} />`). Both work, both produce deeply nested wrapper trees in the devtools. Custom hooks compose flat — that's the pitch.

---

## Rapid-fire round

**Error boundaries** — a class component with `getDerivedStateFromError` / `componentDidCatch` that renders a fallback instead of unmounting the tree. Still class-only; the usual answer is `react-error-boundary`. Catches render, lifecycle and constructor errors — *not* event handlers, async code or SSR.

**Code splitting** — `const Chart = lazy(() => import('./Chart'))` wrapped in `<Suspense fallback={…}>`. Route-level splitting first, then heavy widgets.

**Fragments** — `<>…</>` groups children without adding a DOM node; use `<React.Fragment key={id}>` when you need a key.

**Portals** — `createPortal(children, document.body)` renders outside the parent DOM node (modals, tooltips) while keeping React context and event bubbling through the React tree.

**Class lifecycle → hooks** — `componentDidMount` → `useEffect(…, [])`; `componentDidUpdate` → `useEffect(…, [deps])`; `componentWillUnmount` → the cleanup return; `shouldComponentUpdate` → `React.memo`.

**Why can't a component return two elements?** It can — an array or a Fragment. A component must return a single *node*, because `createElement` returns one object.

**Reconciler follow-up** — Fiber is the rewrite that made rendering interruptible: work is split into units, prioritized, and can be paused, resumed or aborted.

**React 18/19 vocabulary worth a one-liner:** concurrent rendering, `useTransition` for non-urgent updates, `useDeferredValue`, Suspense for data fetching, Server Components (render on the server, ship no JS), Actions and `useActionState`/`useOptimistic` for form mutations, the `use` hook for reading a promise or context conditionally, and the React Compiler for automatic memoization.

**Controlled question you should ask them back:** Vite or Next.js, React Query or Redux, and whether the app is CSR or SSR — it tells you which half of this list the rest of the interview lives in.
