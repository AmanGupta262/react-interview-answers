# React Interview Answers (4+ Years / Architect Level)

Each answer has a **short answer** (say this first), an explanation, and an **interview tip** for follow-ups. Targets React 18/19.

**Contents:** [1. Fundamentals](#1-react-fundamentals) · [2. Hooks](#2-react-hooks) · [3. Advanced](#3-advanced-react) · [4. Performance](#4-react-performance-optimization) · [5. State Management](#5-state-management) · [6. Architect-Level](#6-architect-level-react)

---

## 1. React Fundamentals

### 1.1 What is React, and why is it used?

**Short answer:** React is a JavaScript library for building user interfaces out of small, reusable components. It is used because you describe what the UI should look like for a given state, and React updates the DOM efficiently when that state changes.

**Explanation:**
- It is a library, not a full framework. It handles the view layer; routing, data fetching and build tooling come from other libraries or frameworks (React Router, TanStack Query, Next.js, Vite).
- **Declarative:** you write `UI = f(state)` instead of manually calling `document.createElement` and `appendChild`.
- **Component-based:** UI is split into isolated pieces that own their markup, logic and state.
- **Efficient updates:** React compares the new output with the previous output and applies only the needed DOM changes.
- **Ecosystem and portability:** large community, React Native for mobile, server rendering and Server Components for performance.
- Analogy: you give a chef (React) the recipe for the final dish (the UI for this state), and the chef decides which steps to redo when an ingredient changes.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Clicked {count}</button>;
}
```

**Interview tip:** Say "library, not framework" and explain declarative vs imperative. Seniors also mention the trade-off: React gives you flexibility but you choose your own router, data layer and structure.

---

### 1.2 What is the difference between React and React DOM?

**Short answer:** `react` contains the core logic (components, hooks, elements, reconciliation contract), independent of any platform. `react-dom` is the renderer that takes React's output and puts it into the browser DOM.

**Explanation:**
- This split lets the same React core target different platforms: `react-dom` for web, `react-native` for mobile, `react-three-fiber` for 3D, `react-test-renderer`/Ink for others.
- `react` exports `useState`, `useEffect`, `createElement`, `Fragment`, `memo`, `Suspense`, etc.
- `react-dom` exports `createRoot` and `hydrateRoot` (from `react-dom/client`), `createPortal`, `flushSync`, and `renderToString`/streaming APIs on the server side (`react-dom/server`).

| Aspect | `react` | `react-dom` |
|---|---|---|
| Purpose | Core: components, hooks, elements | Renderer for the browser DOM |
| Platform | Platform-agnostic | Web only |
| Typical imports | `useState`, `useEffect`, `memo`, `Suspense` | `createRoot`, `hydrateRoot`, `createPortal`, `flushSync` |
| Needed in React Native? | Yes | No (uses `react-native` renderer) |

```jsx
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')).render(<App />);
```

**Interview tip:** Mention that React 18 replaced `ReactDOM.render` with `createRoot`, which enables concurrent features and automatic batching.

---

### 1.3 What is JSX? How does it work internally?

**Short answer:** JSX is a syntax extension that lets you write HTML-like markup inside JavaScript. A compiler (Babel, SWC, esbuild, TypeScript) transforms it into plain function calls that produce React element objects.

**Explanation:**
- Browsers do not understand JSX; it is compiled at build time.
- With the **new JSX transform** (React 17+, default in React 18/19), JSX compiles to calls from `react/jsx-runtime`, so you no longer need `import React from 'react'` just for JSX.
- The old transform compiled to `React.createElement(type, props, ...children)`.
- The result is a plain object (a React element), roughly `{ type, props, key, ref }`, describing what to render. It is not a DOM node.
- Expressions go inside `{}`; attributes use camelCase (`className`, `onClick`, `htmlFor`).

```jsx
// You write
const el = <h1 className="title">Hello, {name}</h1>;

// New transform output (simplified)
import { jsx as _jsx } from 'react/jsx-runtime';
const el = _jsx('h1', { className: 'title', children: ['Hello, ', name] });
```

**Interview tip:** Know that JSX is optional, that it escapes interpolated values by default (XSS protection), and that `dangerouslySetInnerHTML` is the explicit opt-out.

---

### 1.4 What is the Virtual DOM?

**Short answer:** The Virtual DOM is a lightweight in-memory tree of JavaScript objects (React elements) that represents the UI. React compares the new tree to the previous one and updates the real DOM only where something changed.

**Explanation:**
- Every render produces a new tree of React elements describing the desired UI.
- React diffs it against the previous tree, computes the minimal set of changes, and commits those to the real DOM in one batch.
- Analogy: an architect edits a blueprint (cheap) and the builders only change the rooms that differ, instead of rebuilding the house.
- Internally React 16+ uses a Fiber tree (a linked-list-like structure of work units) which allows work to be split, paused and prioritised; "Virtual DOM" is the mental model, Fiber is the implementation.
- It is not magic-fast: it exists to give a declarative model with good-enough performance. Direct hand-tuned DOM updates can be faster.

**Interview tip:** Mention Fiber and that the Virtual DOM's main benefit is the declarative programming model, not raw speed.

---

### 1.5 How does React reconcile changes in the DOM?

**Short answer:** Reconciliation is the process where React diffs the new element tree against the previous one using a heuristic O(n) algorithm, then commits only the necessary DOM operations.

**Explanation:**
- **Render phase:** React calls your components, builds the new tree, and diffs it with the current Fiber tree. This phase is pure and can be interrupted in concurrent rendering.
- **Commit phase:** React applies the computed changes to the DOM, then runs layout effects and, after paint, passive effects (`useEffect`). This phase is synchronous.
- Diffing heuristics:
  - **Different element type** at the same position: the old subtree is unmounted (state lost) and a new one is mounted.
  - **Same type:** the instance is kept, props are updated, and React recurses into children.
  - **Lists:** `key` is used to match children across renders.
- Same component type at the same position preserves state; changing the `key` resets it.

```jsx
// Switching type unmounts <input> state; switching only props keeps it
{isEditing ? <input /> : <p>text</p>}
```

**Interview tip:** State what is preserved or destroyed. A common senior trick is using `key` to deliberately reset a component's state (`<Form key={userId} />`).

---

### 1.6 What is the difference between Real DOM and Virtual DOM?

**Short answer:** The Real DOM is the browser's actual tree of nodes that is rendered on screen; the Virtual DOM is React's in-memory JavaScript representation of that tree used to compute efficient updates.

| Aspect | Real DOM | Virtual DOM |
|---|---|---|
| What it is | Browser's live document tree | Plain JS objects (React elements / Fibers) |
| Updates | Direct manipulation, can trigger layout and repaint | Diffed in memory first, then minimal DOM changes applied |
| Cost of change | Expensive (reflow/repaint) | Cheap to create and compare |
| Visible on screen | Yes | No |
| Who manages it | Browser | React |
| Programming style | Usually imperative | Declarative |

**Explanation:**
- DOM operations themselves are not "slow"; repeated layout and style recalculation from many uncoordinated writes is the cost.
- React batches updates (automatic batching in React 18) and applies them together.

**Interview tip:** Avoid saying "Virtual DOM is faster than the DOM". Say it avoids unnecessary DOM work and enables a declarative model.

---

### 1.7 What are Functional Components and Class Components?

**Short answer:** A functional component is a JavaScript function that takes props and returns JSX; a class component is an ES6 class extending `React.Component` with a `render()` method. Since hooks (16.8), functional components are the standard.

| Aspect | Functional | Class |
|---|---|---|
| Syntax | Function returning JSX | `class X extends Component` with `render()` |
| State | `useState`, `useReducer` | `this.state`, `this.setState` |
| Side effects | `useEffect` | `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` |
| `this` | Not used | Required, binding pitfalls |
| Error boundaries | Not possible directly (needs a class, or a library such as `react-error-boundary`) | Yes (`componentDidCatch`, `getDerivedStateFromError`) |
| Logic reuse | Custom hooks | HOCs, render props |
| Status | Recommended | Still supported, not recommended for new code |

```jsx
function Greeting({ name }) { return <h1>Hello {name}</h1>; }

class GreetingC extends React.Component {
  render() { return <h1>Hello {this.props.name}</h1>; }
}
```

**Interview tip:** Error boundaries are the one thing that still requires a class. Also note that functional components capture props/state per render (closures), while `this.props` in a class is mutable and always reads the latest.

---

### 1.8 What is the difference between props and state?

**Short answer:** Props are inputs passed from a parent to a child and are read-only for the child. State is data owned and managed by the component itself that can change over time and triggers re-renders.

| Aspect | Props | State |
|---|---|---|
| Owner | Parent | The component itself |
| Mutable by component? | No (read-only) | Yes, via setter / dispatch |
| Purpose | Configure a component | Track data that changes over time |
| Triggers re-render | When parent re-renders with new values | When setter is called with a new value |
| Direction | Top-down | Local (can be passed down as props) |

**Explanation:**
- Analogy: props are function arguments; state is a local variable that React remembers between calls.
- A parent's state is often a child's props.
- Never mutate either directly; use `setState` with a new object/array.
- Do not copy props into state unless you need an initial value; derive values during render instead.

```jsx
function Parent() {
  const [count, setCount] = useState(0);        // state
  return <Child value={count} onInc={() => setCount(c => c + 1)} />; // props
}
```

**Interview tip:** Mention "lifting state up" and avoiding derived state duplication (a common source of stale-data bugs).

---

### 1.9 Why are React components called reusable?

**Short answer:** A component encapsulates markup, logic and styling behind a props-based interface, so the same component can be rendered many times with different inputs.

**Explanation:**
- **Props as API:** behaviour and appearance change by inputs, not by editing the component.
- **Composition:** components combine through `children` and render props; this is preferred over inheritance.
- **Isolated state:** each instance has its own state, so two `<Counter />` components do not interfere.
- **Custom hooks** extend reuse to stateful logic without changing the UI tree.
- Analogy: LEGO bricks with standard connectors.

```jsx
function Button({ variant = 'primary', children, ...rest }) {
  return <button className={`btn btn-${variant}`} {...rest}>{children}</button>;
}
<Button variant="danger" onClick={remove}>Delete</Button>
```

**Interview tip:** Mention composition over inheritance and the cost of over-generalising (a component with 20 boolean props is a design smell).

---

### 1.10 What is the purpose of `key` in React lists?

**Short answer:** `key` is a special prop that gives each list item a stable identity so React can match items between renders, and correctly add, remove, move or preserve them.

**Explanation:**
- During reconciliation, React uses keys to know which child is the same item as before.
- Matching keys keep component state and DOM nodes; new keys mount; missing keys unmount.
- Keys must be unique among siblings (not globally), stable across renders, and not random per render.
- `key` is not passed to the component as a prop; use another prop if the child needs the value.
- Changing a `key` on purpose forces a remount, which resets state.

```jsx
<ul>
  {todos.map(t => <TodoItem key={t.id} todo={t} />)}
</ul>
```

**Interview tip:** Mention that keys also apply outside lists, for example `<Profile key={userId} />` to reset state on identity change.

---

### 1.11 Why should we not use array indexes as keys?

**Short answer:** An index identifies a position, not an item. If the list is reordered, filtered or has items inserted/removed, indexes shift and React attaches the wrong state and DOM to the wrong items.

**Explanation:**
- Symptoms: inputs keeping the previous row's text, wrong checkbox state after deletion, animations and focus misbehaving, and unnecessary re-renders or remounts.
- Reason: after removing item 0, the item that was at index 1 now has key 0, so React reuses item 0's instance and state for it.
- Index is acceptable only if the list is static, never reordered or filtered, and items have no state or identity.
- Prefer a stable id from data (database id, UUID created when the item is created). Do not generate `Math.random()` keys in render, since they remount everything every render.

```jsx
// Bad if list can change
items.map((item, i) => <Row key={i} item={item} />);

// Good
items.map(item => <Row key={item.id} item={item} />);
```

**Interview tip:** Walk through the delete-first-row example with uncontrolled inputs; it demonstrates real understanding rather than memorised advice. Also note that React falls back to index silently (with a warning only when keys are missing).

---

### 1.12 What is conditional rendering?

**Short answer:** Conditional rendering is showing different UI depending on a condition (state, props, data) using ordinary JavaScript constructs inside or around JSX.

**Explanation:** Common patterns:
- `if`/early `return` before the JSX (good for loading/error/empty states).
- Ternary `cond ? <A /> : <B />` for either/or.
- `&&` for show-or-nothing.
- `null` (or `false`) renders nothing.
- Extract a lookup object or a small component for many branches.

```jsx
function Inbox({ loading, error, messages }) {
  if (loading) return <Spinner />;
  if (error) return <ErrorMsg error={error} />;
  return (
    <>
      {messages.length > 0 ? <List items={messages} /> : <Empty />}
      {messages.length > 0 && <Footer />}
    </>
  );
}
```

**Interview tip:** The `&&` pitfall: `{count && <X />}` renders `0` when `count` is 0. Use `count > 0 && ...` or a ternary. Also note that switching between branches of a different type unmounts and loses state; hiding with CSS preserves it.

---

### 1.13 What is prop drilling?

**Short answer:** Prop drilling is passing data through several intermediate components that do not use it, just so a deeply nested component can receive it.

**Explanation:**
- Problems: noisy intermediate components, tight coupling, harder refactors, and re-renders of the whole chain when the value changes.
- Options to avoid it:
  - **Composition:** pass the already-built element as `children` or a slot prop, so the middle layers never see the data.
  - **Context API** (`createContext` and `useContext`) for truly global-ish data like theme, locale, auth user.
  - **State libraries** (Redux Toolkit, Zustand, Jotai) for shared app state.
  - **Colocate state** closer to where it is used.
- Context is not a free fix: every consumer re-renders when the context value changes, so split contexts and memoise the value.
- Drilling through 2-3 levels is fine; do not over-engineer.

```jsx
// Composition avoids drilling `user`
<Layout header={<Header user={user} />} />
```

**Interview tip:** Mention composition first, before Context or Redux; it is the answer that shows experience.

---

### 1.14 What are Fragments in React?

**Short answer:** A Fragment lets a component return multiple elements without adding an extra DOM node.

**Explanation:**
- A component must return a single root; Fragment is that root and renders nothing itself.
- Short syntax: `<>...</>`. Long syntax: `<React.Fragment key={id}>...</React.Fragment>` (or `<Fragment>` when imported).
- Only the long syntax accepts `key` (needed when mapping to a group of sibling elements).
- Useful for valid HTML structure, such as returning `<tr>`s into a `<tbody>`, `<dt>/<dd>` pairs in a `<dl>`, and avoiding extra wrapper `<div>`s that break flex or grid layouts.

```jsx
function Row({ term, desc }) {
  return (
    <>
      <dt>{term}</dt>
      <dd>{desc}</dd>
    </>
  );
}

items.map(i => <Fragment key={i.id}><dt>{i.term}</dt><dd>{i.desc}</dd></Fragment>)
```

**Interview tip:** Fragments accept only `key` (and in recent React versions experimental `ref`); no other attributes such as `className`.

---

### 1.15 What is the difference between controlled and uncontrolled components?

**Short answer:** In a controlled component, React state is the single source of truth for the form value, set through `value` and `onChange`. In an uncontrolled component, the DOM holds the value and you read it when needed via a `ref` (or `FormData`).

| Aspect | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | The DOM node |
| Value prop | `value` / `checked` + `onChange` | `defaultValue` / `defaultChecked` |
| Reading value | From state at any time | `ref.current.value` or `FormData` on submit |
| Re-renders per keystroke | Yes | No |
| Instant validation, masking, dependent fields | Easy | Harder |
| Best for | Dynamic, validated forms | Simple forms, file inputs, large forms, integrating non-React code |

```jsx
// Controlled
const [name, setName] = useState('');
<input value={name} onChange={e => setName(e.target.value)} />

// Uncontrolled
const ref = useRef(null);
<input defaultValue="Aman" ref={ref} />
<button onClick={() => console.log(ref.current.value)}>Save</button>
```

**Explanation:**
- `<input type="file">` is always uncontrolled because its value is read-only.
- Setting `value` without `onChange` makes the field read-only and React warns.
- Switching an input from uncontrolled to controlled (value going from `undefined` to defined) triggers a warning; initialise state with `''`.
- Libraries such as React Hook Form use uncontrolled inputs with refs to minimise re-renders.

**Interview tip:** Mention performance trade-offs and that React 19 form Actions (`<form action={fn}>`) read values from `FormData`, which fits the uncontrolled model well.


---

## 2. React Hooks

### 2.1 What are React Hooks?

**Short answer:** Hooks are functions that let function components use React features such as state, lifecycle behavior, context and refs. They all start with `use` (for example `useState`, `useEffect`).

**Explanation:**
- Before Hooks, only class components could hold state and run lifecycle methods.
- Hooks let you "hook into" React from a plain function.
- React remembers each hook's data by the order in which hooks are called on every render.
- Built-in examples: `useState`, `useReducer`, `useEffect`, `useLayoutEffect`, `useRef`, `useMemo`, `useCallback`, `useContext`. React 18 added `useId`, `useTransition`, `useDeferredValue`, `useSyncExternalStore`; React 19 added `use`, `useActionState`, `useOptimistic`, and `useFormStatus` (from `react-dom`).

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Interview tip:** Say that Hooks are not magic: React keeps a linked list of hook slots per component, which is why call order matters.

### 2.2 Why were Hooks introduced?

**Short answer:** Hooks (React 16.8) let function components have state and side effects, and let us reuse stateful logic without wrapper components or classes.

**Explanation:**
- **Reusing logic was painful:** HOCs and render props caused "wrapper hell" and deep trees. Custom Hooks share logic as plain functions.
- **Lifecycle methods split related code:** In classes, one concern (for example a subscription) was spread across `componentDidMount`, `componentDidUpdate` and `componentWillUnmount`. `useEffect` keeps setup and cleanup together.
- **Classes are confusing:** `this` binding, constructors and handlers needed extra care. Classes also optimize less well in tooling.
- Classes are not removed; both styles work, but new code uses function components.

**Interview tip:** Mention that Hooks were opt-in and backward compatible, with no rewrite of existing classes required.

### 2.3 What are the Rules of Hooks?

**Short answer:** Call Hooks only at the top level of a component or custom Hook, and only from React function components or other custom Hooks.

**Explanation:**
- Do not call Hooks inside loops, conditions, nested functions, or after an early `return`.
- Do not call them from regular JS functions or class components.
- Reason: React matches hook state to the call order. If the order changes between renders, state gets mixed up.
- The `eslint-plugin-react-hooks` rules `rules-of-hooks` and `exhaustive-deps` enforce this.

**Interview tip:** Mention `exhaustive-deps` too; it catches stale closure bugs, not just ordering mistakes.

### 2.4 Explain useState with an example.

**Short answer:** `useState(initial)` returns `[value, setValue]`. Calling the setter schedules a re-render with the new value.

**Explanation:**
- State persists between renders; ordinary local variables do not.
- State updates are batched and applied on the next render, so `value` does not change immediately after calling the setter.
- Use the functional form `setX(prev => ...)` when the new value depends on the previous one. It avoids stale values.
- Pass a function for expensive initial values (lazy init): `useState(() => compute())` runs only on the first render.
- Never mutate state; create a new object or array.
- If you set the same value (`Object.is`), React skips the re-render.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const addThree = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1); // result: +3. With setCount(count + 1) x3 it would be +1
  };
  return <button onClick={addThree}>{count}</button>;
}
```

**Interview tip:** Explain why `setCount(count + 1)` three times gives +1: every call closes over the same `count` from that render.

### 2.5 What is the difference between useState and useReducer?

**Short answer:** Both manage state. `useState` suits simple, independent values; `useReducer` suits complex state with several related fields or many transitions, since all logic lives in one pure reducer.

| Aspect | useState | useReducer |
|---|---|---|
| Signature | `[state, setState]` | `[state, dispatch]` with `reducer(state, action)` |
| Best for | Simple values, toggles, inputs | Complex objects, related fields, multi-step transitions |
| Update logic | Scattered across handlers | Centralized in the reducer |
| Testing | Test through the component | Reducer is a pure function, easy to unit test |
| Passing down | Often several setters | One stable `dispatch` (identity never changes) |

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'add': return { ...state, items: [...state.items, action.item] };
    case 'clear': return { ...state, items: [] };
    default: return state;
  }
}
const [state, dispatch] = useReducer(reducer, { items: [] });
dispatch({ type: 'add', item: 'milk' });
```

**Interview tip:** `useState` is built on the same mechanism as `useReducer`; and a stable `dispatch` pairs well with Context to avoid re-creating callbacks.

### 2.6 Explain useEffect and its dependency array.

**Short answer:** `useEffect` runs side effects (fetching, subscriptions, timers, DOM work) after the component renders and the browser has painted. The dependency array tells React when to re-run the effect: only when a listed value changes.

**Explanation:**
- Effects run after render, not during it, so they usually do not block painting. (Exception: if the update came from a user click or keypress, React 18+ may run the effect before the browser paints.)
- React compares each dependency with `Object.is`. If all are equal to the previous render, the effect is skipped.
- Include every reactive value (props, state, values derived from them) that the effect reads. Leaving some out causes stale closures.
- Objects, arrays and functions created inline are new on every render, so they re-trigger the effect. Memoize or move them inside the effect.
- Effects are for syncing with external systems. Do not use them to derive state from props or to respond to clicks; compute during render or use event handlers.

```jsx
useEffect(() => {
  document.title = `${count} items`;
}, [count]); // re-runs only when count changes
```

**Interview tip:** Say "effects synchronize with external systems; the dependency array is not a lifecycle trigger" to show you know the modern mental model.

### 2.7 What happens when the dependency array is empty?

**Short answer:** The effect runs once after the first render (mount), and its cleanup runs once on unmount.

**Explanation:**
- `[]` means "this effect depends on no reactive values".
- It is fine for one-time setup such as a subscription or a fixed event listener.
- Common bug: reading props or state inside an `[]` effect. It captures the first-render values forever (stale closure). Use functional updates, refs, or the proper deps.
- In dev StrictMode (React 18+) it still runs mount, cleanup, mount once, to expose missing cleanup.

```jsx
useEffect(() => {
  const id = setInterval(() => setTick(t => t + 1), 1000); // functional update avoids stale tick
  return () => clearInterval(id);
}, []);
```

**Interview tip:** Do not say "same as componentDidMount"; it is similar but effects always need cleanup thinking and run twice in StrictMode dev.

### 2.8 What happens when we don't provide a dependency array?

**Short answer:** The effect runs after every render of the component.

**Explanation:**
- No array means no comparison, so React re-runs the effect (and its previous cleanup) on every render.
- Risk: if the effect sets state, it can cause an infinite render loop, for example `useEffect(() => setData(x))` with no array.
- Rarely intended. Use it only for something that truly must sync on every render.

| Deps | When it runs |
|---|---|
| none | After every render |
| `[]` | After mount only |
| `[a, b]` | After mount and whenever `a` or `b` change |

**Interview tip:** Mention the infinite-loop trap: fetch, `setState`, re-render, fetch again.

### 2.9 What is the cleanup function in useEffect?

**Short answer:** It is the function returned from the effect. React runs it before the effect re-runs and when the component unmounts, to undo what the effect set up.

**Explanation:**
- Use it for: removing listeners, clearing timers, unsubscribing, closing sockets, and cancelling in-flight requests.
- Think of it as "putting the toy back before taking out a new one".
- For fetch, use `AbortController` so an old request cannot overwrite newer state (race condition) or update an unmounted component.

```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setUser)
    .catch(err => {
      if (err.name !== 'AbortError') setError(err);
    });
  return () => controller.abort();
}, [id]);
```

**Interview tip:** Say that cleanup runs with the old values from the previous render, and that aborting a fetch throws an `AbortError` you must ignore.

### 2.10 What is the difference between useEffect and useLayoutEffect?

**Short answer:** `useLayoutEffect` runs synchronously after React updates the DOM but before the browser paints; `useEffect` runs after paint. Use `useLayoutEffect` only when you must measure or adjust the DOM before the user sees it.

| Aspect | useEffect | useLayoutEffect |
|---|---|---|
| Timing | After browser paint (asynchronous) | After DOM mutations, before paint (synchronous) |
| Blocks paint | No | Yes |
| Typical use | Data fetching, subscriptions, logging | Measuring layout, positioning tooltips, preventing flicker |
| SSR | Fine | Warns on the server in older versions; avoid on the server |
| Default choice | Yes | Only when flicker appears |

```jsx
useLayoutEffect(() => {
  const { height } = ref.current.getBoundingClientRect();
  setTooltipTop(anchorY - height); // applied before paint, no flicker
}, [anchorY]);
```

**Interview tip:** Heavy work in `useLayoutEffect` freezes the UI because it blocks paint; reach for it only after seeing a visual flash.

### 2.11 What is useRef? How is it different from useState?

**Short answer:** `useRef(initial)` returns a mutable object `{ current }` that persists across renders. Changing `.current` does not trigger a re-render, unlike state.

| Aspect | useState | useRef |
|---|---|---|
| Triggers re-render on change | Yes | No |
| Value read | The render's snapshot | Always the latest `.current` |
| Mutate directly | No (use setter) | Yes (`ref.current = x`) |
| Use for | Data shown in the UI | DOM nodes, timer IDs, previous values, mutable flags |

**Explanation:**
- Main uses: accessing DOM elements (`ref={inputRef}`), storing values between renders that do not affect output (interval ID, previous value, "is mounted" flag).
- Do not read or write `ref.current` during render for values that drive UI; do it in effects or handlers.

```jsx
const inputRef = useRef(null);
<input ref={inputRef} />
<button onClick={() => inputRef.current.focus()}>Focus</button>
```

**Interview tip:** Refs are a common fix for stale closures: store the latest callback or value in a ref and read it inside long-lived callbacks.

### 2.12 What is useMemo?

**Short answer:** `useMemo(fn, deps)` caches the result of a calculation between renders and recomputes only when a dependency changes.

**Explanation:**
- Use it for expensive calculations or to keep a stable object/array reference that is passed to a `React.memo` child or used as another hook's dependency.
- It is a performance hint, not a semantic guarantee. React may discard the cache, so the code must still be correct without it.
- Do not wrap everything. Memoization has a cost, and cheap calculations are faster to just redo.
- React Compiler (React 19 era) can add memoization automatically, which reduces manual use.

```jsx
const visible = useMemo(
  () => items.filter(i => i.name.includes(query)),
  [items, query]
);
```

**Interview tip:** Say "measure first with the Profiler"; premature memoization is a smell.

### 2.13 What is useCallback?

**Short answer:** `useCallback(fn, deps)` returns the same function reference between renders until its dependencies change.

**Explanation:**
- Functions are recreated on every render, so a child wrapped in `React.memo` would still re-render because it gets a "new" prop.
- `useCallback` keeps the reference stable so `React.memo` can skip the child.
- It is also useful when a function is a dependency of `useEffect`.
- It does not make the function itself faster; it only preserves identity.

```jsx
const handleSave = useCallback(() => save(id), [id]);
return <Row onSave={handleSave} />; // Row is React.memo
```

**Interview tip:** `useCallback` without `React.memo` (or a hook dependency) on the receiving side usually does nothing useful.

### 2.14 What is the difference between useMemo and useCallback?

**Short answer:** `useMemo` caches a computed value; `useCallback` caches a function. `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

| Aspect | useMemo | useCallback |
|---|---|---|
| Caches | Result of calling `fn` | The function `fn` itself |
| Returns | A value (number, array, object, JSX) | A function |
| Typical use | Expensive calculation, stable object reference | Stable handler passed to memoized child or used in deps |
| Re-computes when | Deps change | Deps change (new function) |

**Interview tip:** Both exist for referential equality and avoiding repeated work; both are useless if deps change every render.

### 2.15 What is a custom Hook? How do you create one?

**Short answer:** A custom Hook is a function whose name starts with `use` and that calls other Hooks, so stateful logic can be reused across components.

**Explanation:**
- Create it by extracting repeated logic into a function named `useSomething` that uses built-in Hooks.
- Each component that calls it gets its own independent state. Hooks share logic, not state.
- The `use` prefix lets React and the linter apply the Rules of Hooks.
- It can take arguments and return anything: values, setters, objects.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    fetch(url, { signal: controller.signal })
      .then(r => r.json())
      .then(setData)
      .catch(e => { if (e.name !== 'AbortError') console.error(e); })
      .finally(() => { if (!controller.signal.aborted) setLoading(false); });
    return () => controller.abort();
  }, [url]);
  return { data, loading };
}
```

**Interview tip:** For real apps, say you would use TanStack Query instead of hand-rolling `useFetch`, which adds caching, dedupe and retries.

### 2.16 Can we call Hooks inside loops or conditions?

**Short answer:** No. Hooks must run in the same order on every render, and loops or conditions can change that order.

**Explanation:**
- React identifies each hook by its position. If a conditional skips a `useState`, every later hook reads the wrong slot.
- Fix: call the Hook unconditionally and put the condition inside it.
- Need a Hook per list item? Extract a child component and call the Hook there.

```jsx
// Bad
if (isLoggedIn) { const [name, setName] = useState(''); }

// Good
const [name, setName] = useState('');
useEffect(() => {
  if (!isLoggedIn) return;
  loadProfile();
}, [isLoggedIn]);
```

**Interview tip:** The one exception-looking API is `use` (React 19): it can be called inside conditions and loops, but it is not a classic Hook and still only works in components or Hooks.

### 2.17 Why does useEffect sometimes execute twice in development?

**Short answer:** In React 18+ `<StrictMode>` development, React intentionally mounts, unmounts and re-mounts each component once, so effects run, clean up, then run again. It happens only in dev, never in production.

**Explanation:**
- Purpose: to reveal effects that lack proper cleanup or are not resilient to being re-run (leaks, duplicate subscriptions, duplicated requests).
- Render functions, state initializers and reducers are also double-invoked in StrictMode to detect impure code.
- The right response is not to remove StrictMode or use a "ran once" ref hack. Make the effect idempotent with a cleanup.
- Fetch example: an abort in cleanup cancels the first request; the second is the real one.

```jsx
useEffect(() => {
  const conn = createConnection(roomId);
  conn.connect();
  return () => conn.disconnect(); // makes the double run harmless
}, [roomId]);
```

**Interview tip:** If asked about duplicate API calls in dev, say it is expected and that a data library (TanStack Query) dedupes them anyway.

### 2.18 How do you prevent unnecessary API calls in useEffect?

**Short answer:** Keep the dependency array accurate, clean up with `AbortController`, debounce user input, and for real apps use a data-fetching library like TanStack Query (React Query) which caches and dedupes requests.

**Explanation:**
- **Correct deps:** Depend on primitives (`id`) rather than inline objects or functions that change every render. Memoize with `useMemo`/`useCallback` if needed.
- **Cleanup/abort:** Cancel stale requests so only the latest response updates state (avoids race conditions).
- **Debounce:** For search boxes, wait until typing pauses before fetching.
- **Guard conditions:** Skip fetching when inputs are missing (`if (!query) return;`).
- **Do not fetch on every event in an effect:** run user-triggered calls (submit, click) from event handlers instead.
- **TanStack Query:** Same `queryKey` means shared cache, in-flight request deduplication, `staleTime` to avoid refetching, automatic retries, and the `enabled` option to control when a query runs. It also handles the StrictMode double mount.
- **Avoid:** A `useRef` "already fetched" flag to silence StrictMode; it hides the problem rather than fixing it.

```jsx
// Plain effect with debounce and abort
useEffect(() => {
  if (!query) return;
  const controller = new AbortController();
  const t = setTimeout(() => {
    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => { if (e.name !== 'AbortError') setError(e); });
  }, 300);
  return () => { clearTimeout(t); controller.abort(); };
}, [query]);

// TanStack Query alternative
const { data } = useQuery({
  queryKey: ['search', query],
  queryFn: ({ signal }) => fetch(`/api/search?q=${query}`, { signal }).then(r => r.json()),
  enabled: !!query,
  staleTime: 60_000,
});
```

**Interview tip:** Also mention that fetching on the server (Server Components, loaders) avoids client effects for initial data altogether.


---

## 3. Advanced React

### 3.1 What is React Fiber?

**Short answer:** Fiber is React's reconciliation engine (since React 16). It breaks rendering into small units of work so React can pause, resume, prioritize, or abandon rendering.

**Explanation:**
- Each component instance has a "fiber": a plain JS object holding its type, props, state, and links to its parent, child, and sibling. Fibers form a linked tree that React walks.
- Work is split into two phases:
  - **Render phase** (interruptible): React calls components and diffs the result to work out what changed. No DOM is touched, so React can pause or throw the work away.
  - **Commit phase** (synchronous, cannot be interrupted): React applies DOM changes and runs `useLayoutEffect`. `useEffect` callbacks are scheduled to run afterwards, usually after the browser paints.
- Analogy: the old stack reconciler was like reading a book in one sitting without stopping. Fiber lets you bookmark the page and answer the door when something urgent (a keystroke) arrives.
- Fiber is what makes concurrent features such as `useTransition` and Suspense possible.
- React keeps two trees: the *current* tree (on screen) and the *work-in-progress* tree, which is swapped in at commit.

**Interview tip:** Say that render must be pure because the render phase can run multiple times or be discarded (and Strict Mode double-invokes it on purpose).

### 3.2 What is reconciliation in React?

**Short answer:** Reconciliation is the process React uses to compare the new element tree with the previous one and work out the minimum set of DOM updates.

**Explanation:**
- React uses a heuristic O(n) diff based on two assumptions:
  - Elements of different types produce different trees. A type change (`<div>` to `<span>`, or `A` to `B`) unmounts the old subtree and mounts a new one, so state is lost.
  - `key` identifies which child is which across renders.
- Same type: React keeps the instance and DOM node and only updates changed props.
- Lists: without stable keys, React matches by index. Inserting at the top then reuses the wrong instances, causing bugs with state and inputs.

```jsx
// Good: stable id
items.map(item => <Row key={item.id} item={item} />)
// Bad when list reorders/inserts: index
items.map((item, i) => <Row key={i} item={item} />)
```

**Interview tip:** Changing a component's `key` deliberately is the official way to reset its state.

### 3.3 What is batching in React?

**Short answer:** Batching means React groups multiple state updates into a single re-render instead of rendering after each `setState`.

**Explanation:**
- It avoids wasted renders and keeps the UI from showing half-updated state.
- In React 17 and earlier, batching only worked inside React event handlers. Updates in `setTimeout`, promises, or native handlers each caused their own render.
- State updates are queued; the component renders once with all of them applied.

```jsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // one re-render, not two
}
```

**Interview tip:** `setState` is not synchronous. `console.log(count)` right after it still shows the old value; use the functional form or an effect for the latest value.

### 3.4 What is automatic batching in React 18?

**Short answer:** In React 18 (with `createRoot`), updates are batched everywhere: event handlers, `setTimeout`, promises, and native event handlers.

**Explanation:**
- Before, only updates inside React event handlers were batched.
- Now React batches by default, so `fetch().then(() => { setA(); setB(); })` causes one render.
- To opt out for a specific update, wrap it in `flushSync`, which forces a synchronous render and DOM flush.
- It only applies if you use `createRoot`. Apps still on legacy `ReactDOM.render` keep the old behavior.

```jsx
import { flushSync } from 'react-dom';

flushSync(() => setCount(c => c + 1)); // DOM updated here
flushSync(() => setFlag(f => !f));     // separate render
```

**Interview tip:** Real uses of `flushSync` are rare, e.g. reading layout right after an update (scroll to a just-added item). It hurts performance, so mention that.

### 3.5 What is concurrent rendering?

**Short answer:** Concurrent rendering lets React prepare a new UI in the background and interrupt it if something more urgent happens, keeping the app responsive.

**Explanation:**
- It is built on Fiber's interruptible render phase. It is not a mode you toggle; it is enabled by `createRoot` and takes effect when you use concurrent features.
- Updates have priorities: urgent (typing, clicking) versus transitions (rendering a big filtered list).
- Main APIs:
  - `useTransition` / `startTransition`: mark an update as non-urgent.
  - `useDeferredValue`: get a lagging copy of a value.
  - Suspense for data and code loading.
- Concurrent rendering is not multithreading. It is still one thread; React just yields between units of work.

```jsx
const [isPending, startTransition] = useTransition();

function onChange(e) {
  setQuery(e.target.value);                       // urgent
  startTransition(() => setFilter(e.target.value)); // can be interrupted
}
```

**Interview tip:** Because renders may be restarted, components must be pure and side-effect free in render.

### 3.6 What is React Suspense?

**Short answer:** Suspense lets a component "wait" for something (lazy code, or data via a Suspense-enabled source) while React shows a fallback UI.

**Explanation:**
- Wrap the part that may suspend in `<Suspense fallback={...}>`. The nearest boundary above shows the fallback.
- Works with `React.lazy`, and with data fetching through Suspense-aware libraries/frameworks (Relay, TanStack Query, Next.js) and the `use` hook in React 19.
- It supports streaming SSR and selective hydration in React 18+.
- Plain `useEffect` + `fetch` does not trigger Suspense.

```jsx
<Suspense fallback={<Spinner />}>
  <Profile />   {/* may lazy-load or suspend on data */}
</Suspense>
```

**Interview tip:** Nested boundaries give fine-grained loading states. Combine with `startTransition` to keep old UI on screen instead of flashing the fallback.

### 3.7 What are Error Boundaries?

**Short answer:** Error Boundaries are components that catch JavaScript errors in their child tree during rendering, log them, and show a fallback UI instead of crashing the whole app.

**Explanation:**
- They are still class components implementing `static getDerivedStateFromError` (to render fallback) and/or `componentDidCatch` (to log). There is no hook equivalent. The `react-error-boundary` library gives a ready-made function-style wrapper.
- They catch errors in rendering, lifecycle methods, and constructors of the tree below them.
- They do **not** catch errors in:
  - event handlers (use try/catch),
  - async code (`setTimeout`, promises),
  - server-side rendering,
  - the boundary itself.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error, info) { logToService(error, info); }
  render() {
    return this.state.hasError ? <h1>Something went wrong</h1> : this.props.children;
  }
}
```

**Interview tip:** Place boundaries at several levels (route, widget) so one failure doesn't blank the page.

### 3.8 What is the difference between React.memo and useMemo?

**Short answer:** `React.memo` memoizes a whole component to skip re-rendering when props are unchanged; `useMemo` memoizes a computed value inside a component between renders.

| | `React.memo` | `useMemo` |
|---|---|---|
| What it is | Higher-order component | Hook |
| Caches | Rendered output of a component | Result of a calculation |
| Compares | Props (shallow by default) | Dependency array |
| Purpose | Skip re-render of a child | Skip expensive recompute / keep stable reference |
| Where used | Wrapping a component | Inside a component body |

```jsx
const Child = React.memo(function Child({ data }) { /* ... */ });

function Parent({ items }) {
  const sorted = useMemo(() => [...items].sort(), [items]);
  return <Child data={sorted} />;  // stable ref lets memo work
}
```

**Interview tip:** They often work together: `useMemo` keeps an object/array prop reference stable so `React.memo` on the child isn't defeated. The React Compiler can add much of this automatically.

### 3.9 What is lazy loading in React?

**Short answer:** Lazy loading means loading a component's code only when it is needed, using `React.lazy` with `Suspense`, so the initial bundle is smaller.

**Explanation:**
- `React.lazy` takes a function returning a dynamic `import()` of a module with a default export.
- The component must render inside a `<Suspense>` boundary.
- Typical targets: routes, modals, heavy charts/editors.
- Images and other assets can be lazy loaded too (`loading="lazy"`), but that is a browser feature, not `React.lazy`.

```jsx
const Settings = React.lazy(() => import('./Settings'));

<Suspense fallback={<Spinner />}>
  <Settings />
</Suspense>
```

**Interview tip:** Pair it with an Error Boundary, because a failed chunk download (network error) throws.

### 3.10 What is code splitting?

**Short answer:** Code splitting is dividing your bundle into smaller chunks that load on demand rather than shipping everything upfront.

**Explanation:**
- It is done by the bundler (Webpack, Vite, etc.) whenever it sees a dynamic `import()`.
- `React.lazy` is the React way to consume a split chunk. Code splitting is the technique; lazy loading is the runtime behavior.
- Common split points: per route (most effective), per feature/modal, and heavy third-party libraries.
- Benefits: faster first load, better Time to Interactive.

```jsx
// route-based splitting
const Dashboard = lazy(() => import('./pages/Dashboard'));
```

**Interview tip:** Mention checking the result with a bundle analyzer (e.g. `source-map-explorer`, `rollup-plugin-visualizer`), and prefetching likely next chunks on hover.

### 3.11 What is the difference between client-side rendering and server-side rendering?

**Short answer:** CSR sends a near-empty HTML shell and builds the UI in the browser with JavaScript; SSR renders HTML on the server so the browser gets real content immediately.

| | CSR | SSR |
|---|---|---|
| Where HTML is built | Browser | Server (per request) |
| First paint | Slower (wait for JS) | Faster |
| SEO | Weaker (works with modern crawlers but less reliable) | Better |
| Server load | Low (static hosting) | Higher |
| Interactivity | After JS runs | After hydration |
| Navigation after load | Fast, no page reload | Fast after hydration |

- SSR needs hydration to make the HTML interactive.
- Related options: SSG (build-time HTML), streaming SSR with Suspense, and React Server Components (frameworks like Next.js).

**Interview tip:** SSR improves perceived load (FCP/LCP), but Time to Interactive still waits for hydration, so the two are not the same.

### 3.12 What is hydration?

**Short answer:** Hydration is React attaching event handlers and state to server-rendered HTML in the browser, turning static markup into an interactive app.

**Explanation:**
- The server sends HTML, so the user sees content early. React then runs on the client and reuses the existing DOM instead of recreating it.
- Analogy: the server delivers a fully built house with no electricity; hydration connects the wiring.
- In React 18 you call `hydrateRoot`. Suspense enables streaming and selective hydration: parts of the page hydrate independently, and a part the user interacts with first is prioritized.

```jsx
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root'), <App />);
```

**Interview tip:** React Server Components and islands-style approaches reduce how much needs hydrating at all.

### 3.13 What causes hydration errors?

**Short answer:** They occur when the HTML rendered on the server differs from what React renders on the client's first render.

**Explanation:** Common causes:
- Non-deterministic values in render: `Date.now()`, `Math.random()`, `new Date()` with timezone/locale differences.
- Reading browser-only APIs during render (`window`, `localStorage`, `window.innerWidth`).
- Invalid HTML nesting (`<div>` inside `<p>`, `<a>` inside `<a>`), where the browser fixes the markup before React sees it.
- Browser extensions or third-party scripts that modify the DOM.
- Different data on server versus client.

Fixes:
```jsx
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
return mounted ? <ClientOnly /> : null;
```
- Use `useId` for stable ids, and `suppressHydrationWarning` for unavoidable single-text mismatches such as timestamps.

**Interview tip:** React 18+ falls back to client rendering the mismatched boundary, which hides the cost; React 19 gives better diff messages in the error.

### 3.14 What are portals in React?

**Short answer:** A portal renders children into a DOM node outside the parent component's DOM hierarchy, while keeping them in the same React tree.

**Explanation:**
- Created with `ReactDOM.createPortal(children, domNode)`.
- Used for modals, tooltips, dropdowns, and toasts, to escape `overflow: hidden`, `z-index`, or stacking-context problems.
- Because it stays in the React tree, context works and events bubble through React ancestors, even though the DOM is elsewhere.

```jsx
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(<div className="modal">{children}</div>, document.body);
}
```

**Interview tip:** Mention accessibility: focus trapping, `aria-modal`, and returning focus on close.

### 3.15 What is the difference between forwardRef and useRef?

**Short answer:** `useRef` creates a ref object (to hold a DOM node or a mutable value); `forwardRef` lets a component pass a ref it receives down to a child element.

| | `useRef` | `forwardRef` |
|---|---|---|
| Kind | Hook | Function wrapper for a component |
| Purpose | Create a ref / persistent mutable value | Expose a ref through a component |
| Re-render on change | No | N/A |
| Used by | The component that owns the ref | A reusable component that receives a ref |

```jsx
// React 18: forwardRef
const Input = forwardRef((props, ref) => <input ref={ref} {...props} />);

// React 19: ref is a regular prop on function components
function Input({ ref, ...props }) { return <input ref={ref} {...props} />; }

function Form() {
  const inputRef = useRef(null);
  return <Input ref={inputRef} />;
}
```

**Interview tip:** In React 19 `forwardRef` is no longer needed for function components but still works; pair with `useImperativeHandle` to expose a limited API instead of the raw DOM node.

## 4. React Performance Optimization

### 4.1 How do you optimize a React application?

**Short answer:** Measure first with the Profiler, then reduce unnecessary renders, reduce work per render, and reduce what is shipped and loaded.

**Explanation:**
- **Rendering:** `React.memo`, stable props, move state down, lift content up (children props).
- **Computation:** `useMemo` for truly expensive work; `useTransition`/`useDeferredValue` for heavy updates.
- **Lists:** stable keys, virtualization, pagination.
- **Bundle:** code splitting, lazy routes, tree shaking, lighter dependencies.
- **Network:** caching/dedup with TanStack Query or SWR, debouncing, prefetching.
- **Assets:** image optimization, lazy images, fonts.
- **Rendering strategy:** SSR/SSG, streaming, Server Components.
- React Compiler can auto-memoize, reducing manual `memo`/`useMemo`/`useCallback`.

**Interview tip:** Lead with "profile before optimizing"; premature memoization adds complexity and comparison cost.

### 4.2 What causes unnecessary re-renders?

**Short answer:** A component re-renders when its state changes, its parent re-renders, or a context it consumes changes, even if its output would be identical.

**Explanation:** Common causes:
- Parent re-render: all children re-render by default, regardless of props.
- New object, array, or function created inline each render, which breaks `memo` comparisons.
- Context value changes (a new object each render) re-render all consumers.
- State placed too high in the tree.
- Missing or unstable keys causing remounts.

```jsx
<Child style={{ color: 'red' }} onClick={() => go()} /> // new refs every render
```

**Interview tip:** A re-render is not a DOM update. Rendering is often cheap; only fix it when profiling shows a problem.

### 4.3 How does React.memo work?

**Short answer:** `React.memo` wraps a component and skips re-rendering if its props are shallowly equal to the previous props.

**Explanation:**
- It compares each prop with `Object.is`. Objects, arrays, and functions are compared by reference, so new references break it.
- It does not block re-renders from the component's own state or from context it consumes.
- An optional second argument is a custom comparison function (use rarely).

```jsx
const Row = React.memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});
```

**Interview tip:** If the memo is not working, the usual culprit is an unstable prop (inline object or function). Fix with `useMemo`/`useCallback` or restructure.

### 4.4 When should you use useMemo?

**Short answer:** Use it for genuinely expensive calculations, or to keep a stable object/array reference that is passed to a memoized child or used as a hook dependency.

**Explanation:**
- Good: filtering/sorting large data, heavy derived values.
- Good: stabilizing a reference for `React.memo` children or `useEffect` deps.
- Bad: trivial calculations. `useMemo` itself costs memory and comparison time.
- It is a performance hint, not a semantic guarantee; React may discard the cache.

```jsx
const visible = useMemo(
  () => todos.filter(t => t.status === filter),
  [todos, filter]
);
```

**Interview tip:** Measure the computation (e.g. `console.time`, Profiler) to justify it; with React Compiler enabled, many of these become unnecessary.

### 4.5 When should you use useCallback?

**Short answer:** Use it when you pass a function to a memoized child or into another hook's dependency array, so the function identity stays stable between renders.

**Explanation:**
- `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.
- It only helps if the receiver cares about identity (`React.memo` child, `useEffect` dependency).
- Without a memoized consumer it gives no benefit and adds overhead.

```jsx
const handleSelect = useCallback(id => setSelected(id), []);
return <MemoList onSelect={handleSelect} />;
```

**Interview tip:** Use the functional setter form (`setX(prev => ...)`) so the callback does not need state in its dependency array.

### 4.6 How do you optimize large lists?

**Short answer:** Virtualize so only visible rows are rendered, and combine that with stable keys, memoized rows, and pagination or infinite scroll.

**Explanation:**
- Use virtualization libraries: `react-window`, `react-virtuoso`, or TanStack Virtual.
- Use stable unique `key`s (not array index when the list changes).
- Wrap rows in `React.memo` and pass stable props.
- Paginate or load incrementally (infinite scroll) from the server.
- Avoid heavy work per row; move filtering/sorting to the server or `useMemo`.
- Use `useDeferredValue` for search-as-you-type over big lists.

**Interview tip:** Mention that virtualization breaks Ctrl+F browser search and needs care for accessibility and variable row heights.

### 4.7 What is virtualization?

**Short answer:** Virtualization (windowing) renders only the items currently visible in the viewport, plus a small buffer, instead of the whole list.

**Explanation:**
- Analogy: a scrolling window over a long film strip; you only develop the frames you can see.
- The library calculates which items fit from scroll position and row size, and absolutely positions them inside a container sized to the full list height.
- DOM nodes stay roughly constant (tens) even for 100,000 items.

```jsx
import { FixedSizeList } from 'react-window';

<FixedSizeList height={400} width="100%" itemCount={10000} itemSize={35}>
  {({ index, style }) => <div style={style}>Row {index}</div>}
</FixedSizeList>
```

**Interview tip:** Fixed row heights are simplest; variable or dynamic heights need measurement (`react-virtuoso` and TanStack Virtual handle this).

### 4.8 How do you optimize API calls?

**Short answer:** Avoid unnecessary requests by caching, deduplicating, debouncing, and cancelling, ideally through a data-fetching library like TanStack Query or SWR.

**Explanation:**
- **Caching and dedup:** TanStack Query / SWR share one request across components and revalidate in the background.
- **Debounce** search input; **throttle** scroll/resize triggers.
- **Cancel** stale requests with `AbortController` to avoid race conditions.
- **Parallelize** independent requests; avoid request waterfalls.
- **Prefetch** likely next data; use pagination and selective fields.
- **Server side:** fetch in Server Components or loaders to cut client round trips.

```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/search?q=${q}`, { signal: controller.signal })
    .then(r => r.json()).then(setResults)
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => controller.abort();
}, [q]);
```

**Interview tip:** Note that Strict Mode double-invokes effects in dev, which is exactly why cleanup/abort matters.

### 4.9 How do you identify performance issues in React?

**Short answer:** Reproduce the slowness, then use the React DevTools Profiler and browser Performance tools to find what renders, how often, and why.

**Explanation:**
- React DevTools Profiler: record interactions, see commit times and which components rendered.
- "Highlight updates when components render" option in DevTools.
- Chrome DevTools Performance tab for long tasks and main-thread blocking; Lighthouse and Web Vitals (LCP, INP, CLS) for user-facing metrics.
- Bundle analyzers for size.
- Test with a production build, since dev mode is much slower.
- Real-user monitoring catches issues that only appear on slow devices.

**Interview tip:** Throttle CPU (4x to 6x slowdown) in DevTools to mimic a low-end phone.

### 4.10 What is the React Profiler?

**Short answer:** The Profiler is a React DevTools tab (and a `<Profiler>` component) that records render timings and shows which components rendered and why.

**Explanation:**
- In DevTools: start recording, interact, stop. View the flame graph, ranked chart, and per-commit details; the "why did this render?" info shows changed props, state, or hooks.
- Programmatic `<Profiler id onRender>` gives you callbacks with timing data (actual duration, phase, etc.).
- Profiling needs a profiling-enabled build in production; development builds work out of the box.

```jsx
<Profiler id="Feed" onRender={(id, phase, actualDuration) => {
  console.log(id, phase, actualDuration);
}}>
  <Feed />
</Profiler>
```

**Interview tip:** Look for components with high actual duration and many commits; those are the optimization targets.

### 4.11 How does component splitting improve performance?

**Short answer:** Splitting a big component into smaller ones isolates state, so a state change re-renders only the small part that needs it instead of the whole tree.

**Explanation:**
- Move frequently changing state (an input, a timer, hover) into its own small component.
- The expensive siblings stay untouched because their parent no longer re-renders.
- It also helps with code splitting (lazy chunks) and memoization targets.
- It often removes the need for `memo` entirely.

```jsx
// Before: typing re-renders <ExpensiveTree />
// After: state lives in SearchBox only
function Page() {
  return (<><SearchBox /><ExpensiveTree /></>);
}
```

**Interview tip:** Call this "colocate state" or "push state down"; it is usually the first fix before reaching for memoization.

### 4.12 How do you prevent unnecessary child component re-renders?

**Short answer:** Keep state close to where it is used, pass stable props, and memoize children where profiling shows it matters.

**Explanation:**
- Wrap the child in `React.memo` and keep its props stable (`useMemo` / `useCallback`).
- Move state down, or pass expensive parts as `children` so they keep the same element reference when the parent re-renders.
- Split contexts and memoize context values so unrelated consumers don't re-render.
- Use selectors in state libraries (Zustand, Redux `useSelector`) to subscribe to only the needed slice.
- Or enable the React Compiler to memoize automatically.

```jsx
function Layout({ children }) {
  const [open, setOpen] = useState(false);
  return <div>{/* open changes */}{children}</div>; // children element is unchanged
}
```

**Interview tip:** Explain why the `children` trick works: the child element was created by the parent's parent, so its reference is identical and React skips it.


---

## 5. State Management

### 5.1 What is Context API?

**Short answer:** Context is a built-in React feature that lets a parent pass a value to any descendant without passing props through every level. It solves prop drilling; it is a way to inject values, not a full state manager.

**Explanation:**
- You create a context with `createContext`, provide a value with `<Context.Provider>`, and read it anywhere below with `useContext`.
- Analogy: it is like a building's Wi-Fi. Anyone inside can connect, and you did not have to run a cable to each room.
- Typical uses: theme, current user, locale, feature flags. These change rarely and are needed in many places.
- Context does not hold state by itself. You pair it with `useState` or `useReducer` and put that state in the provider.

```jsx
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Page />
    </ThemeContext.Provider>
  );
}

function Button() {
  const { theme } = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}
```

**Interview tip:** Say "Context is dependency injection, not state management". It shows you separate transporting a value from owning and updating it.

---

### 5.2 What are the limitations of Context API?

**Short answer:** Every component that consumes a context re-renders whenever the provider's value changes, and there is no built-in way to subscribe to only a slice of it. It also has no devtools, middleware, or async handling.

**Explanation:**
- **Re-render fan-out:** `useContext(Ctx)` subscribes to the whole value. If one field changes, all consumers re-render, even if they use only another field. `React.memo` does not stop this, because the context change bypasses props.
- **New object each render:** `value={{ a, b }}` creates a new object on every provider render, so consumers re-render even when `a` and `b` did not change.
- **No selectors:** you cannot say "re-render me only when `user.name` changes".
- **No built-in tooling:** no time-travel debugging, no middleware, no standard place for async logic.
- **Not a server-state tool:** no caching, deduplication, or refetching.

**Mitigations:**
- Split into several contexts, for example separate `StateContext` and `DispatchContext`, or one context per domain.
- Memoize the value with `useMemo` so it changes only when its inputs change.
- Keep state close to where it is used, and move the provider down the tree.
- Use `use-context-selector` for selector-based subscriptions, or use a store library such as Zustand or Redux.

```jsx
const UserStateContext = createContext(null);
const UserDispatchContext = createContext(null);

function UserProvider({ children }) {
  const [user, dispatch] = useReducer(userReducer, initialUser);
  // dispatch is stable, so components that only dispatch never re-render on state change
  return (
    <UserDispatchContext.Provider value={dispatch}>
      <UserStateContext.Provider value={user}>{children}</UserStateContext.Provider>
    </UserDispatchContext.Provider>
  );
}
```

**Interview tip:** The interviewer often asks "how would you stop unnecessary re-renders from context?" Answer with split contexts, a memoized value, and selectors (`use-context-selector`), then say when you would switch to a store.

---

### 5.3 What is Redux?

**Short answer:** Redux is a predictable state container. All app state lives in one store, changes happen only by dispatching actions, and pure reducers compute the next state.

**Explanation:**
- **Three principles:** a single source of truth (the store), state is read-only (change it only by dispatching an action), and changes are made by pure functions (reducers).
- **Data flow is one-way:** UI dispatches an action, the reducer returns new state, the store notifies subscribers, and the UI re-renders.
- Analogy: a bank ledger. You never edit the balance directly. You submit a transaction (action), and the bookkeeper (reducer) applies it and records the new balance.
- Benefits: predictable updates, easy debugging (Redux DevTools can replay actions), and a clear place for shared logic.
- `react-redux` connects it to React with `Provider`, `useSelector`, and `useDispatch`. `useSelector` re-renders a component only when the selected value changes.

**Interview tip:** Mention that Redux is not needed for every app. Strong candidates say it pays off when many parts of the UI share state, updates are complex, and you want traceability.

---

### 5.4 What is Redux Toolkit?

**Short answer:** Redux Toolkit (RTK) is the official, recommended way to write Redux. It removes boilerplate and bakes in good defaults, such as Immer for "mutating" syntax, thunks, and DevTools.

**Explanation:**
- `configureStore`: creates the store with sensible defaults, such as thunk middleware and Redux DevTools.
- `createSlice`: defines the initial state, the reducers, and the action creators in one place.
- Immer is built in, so you can write `state.count += 1` inside reducers. It produces a new immutable state behind the scenes.
- `createAsyncThunk`: handles async calls and dispatches `pending`, `fulfilled`, and `rejected` actions for you.
- `createSelector` (Reselect) for memoized selectors, and RTK Query for data fetching and caching.

```jsx
import { configureStore, createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const res = await fetch('/api/todos');
  return res.json();
});

const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [], status: 'idle' },
  reducers: {
    todoAdded(state, action) {
      state.items.push(action.payload); // safe: Immer handles immutability
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => { state.status = 'loading'; })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state) => { state.status = 'failed'; });
  },
});

export const { todoAdded } = todosSlice.actions;
export const store = configureStore({ reducer: { todos: todosSlice.reducer } });
```

**Interview tip:** Be ready to explain that Immer does not mutate the state. It records your changes on a draft and produces a new immutable object. Also note you can either mutate the draft or return a new value, but not both.

---

### 5.5 What is the difference between Redux and Context API?

**Short answer:** Context is a built-in way to pass values down the tree. Redux is a full state management library with a store, reducers, middleware, selectors, and devtools. They solve different problems, and Redux can even use Context internally to provide the store.

**Explanation:**

| Aspect | Context API | Redux (Toolkit) |
|---|---|---|
| Purpose | Pass values to deep children (dependency injection) | Manage and update shared app state |
| State logic | You write it yourself (`useState` / `useReducer`) | Slices, reducers, actions built in |
| Re-renders | All consumers re-render when value changes | `useSelector` re-renders only when the selected slice changes |
| Async / side effects | None built in | Thunks, listener middleware, sagas, RTK Query |
| Devtools | None dedicated | Redux DevTools: action log, time travel |
| Middleware | No | Yes |
| Setup | Minimal | More setup, less with RTK |
| Best for | Theme, auth user, locale, low-frequency values | Large shared state, frequent updates, complex logic |

- Context alone is fine for values that rarely change. It becomes painful when state updates often and many components read it.
- Redux gives structure and tooling that pay off as the app and team grow.

**Interview tip:** Avoid the claim "Redux replaces Context". A senior answer is "different tools: Context transports, Redux manages. Many apps use both."

---

### 5.6 What are reducers and actions?

**Short answer:** An action is a plain object that describes what happened. A reducer is a pure function `(state, action) => newState` that decides how state changes in response.

**Explanation:**
- **Action:** `{ type: 'todos/todoAdded', payload: {...} }`. It describes an event, not a setter.
- **Reducer:** pure and synchronous. No API calls, no `Date.now()`, no random values, no mutating the input. The same input always gives the same output.
- Analogy: the action is a form someone fills in ("add item X"), and the reducer is the clerk who processes it and returns the updated record.
- This is what `useReducer` in React does as well.

Plain version, only to show the idea:

```js
const action = { type: 'counter/incremented', payload: 1 };

function counterReducer(state = { value: 0 }, action) {
  if (action.type === 'counter/incremented') {
    return { ...state, value: state.value + action.payload };
  }
  return state;
}
```

With Redux Toolkit you do not write action types or switch statements by hand:

```js
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    incremented(state, action) {
      state.value += action.payload;
    },
  },
});

counterSlice.actions.incremented(1); // { type: 'counter/incremented', payload: 1 }
```

**Interview tip:** Interviewers ask "why must reducers be pure?" The answer is predictability, easy testing, time-travel debugging, and because change detection relies on reference equality of new state objects.

---

### 5.7 What is middleware in Redux?

**Short answer:** Middleware sits between dispatching an action and the moment it reaches the reducer. It lets you add behavior such as logging, async calls, or analytics without touching reducers.

**Explanation:**
- Each middleware has the shape `store => next => action => ...`. It can inspect the action, call `next(action)` to pass it on, delay it, replace it, or stop it.
- Common uses: async logic (thunk, saga), logging, crash reporting, analytics, API calls.
- Analogy: airport security between check-in and the gate. It can inspect, let through, or stop you.
- In RTK, `configureStore` already adds some middleware (thunk, plus dev-time checks). You extend it through the `middleware` option.

```js
const logger = (store) => (next) => (action) => {
  console.log('dispatching', action.type);
  const result = next(action);
  console.log('next state', store.getState());
  return result;
};

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefault) => getDefault().concat(logger),
});
```

**Interview tip:** Explain why side effects live in middleware rather than reducers: reducers must stay pure. Mention that order matters, since middleware runs in the order you list it.

---

### 5.8 What is Redux Thunk?

**Short answer:** Redux Thunk is middleware that lets you dispatch a function instead of an action object. That function receives `dispatch` and `getState`, so it can run async code and dispatch actions when it finishes.

**Explanation:**
- It is included by default in `configureStore`.
- Use it for API calls, conditional dispatches, and multi-step flows.
- `createAsyncThunk` is the standard way to write thunks in RTK. It generates the `pending`, `fulfilled`, and `rejected` actions for you.
- Thunks are simple and need no extra concepts. For complex flows such as cancellation, debouncing, or long-running tasks, other tools are easier to manage.

```js
// Hand-written thunk
const loadUser = (id) => async (dispatch, getState) => {
  dispatch({ type: 'user/loading' });
  try {
    const res = await fetch(`/api/users/${id}`);
    dispatch({ type: 'user/loaded', payload: await res.json() });
  } catch (e) {
    dispatch({ type: 'user/failed', error: e.message });
  }
};

// RTK style
const loadUser2 = createAsyncThunk('user/load', async (id, { rejectWithValue }) => {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) return rejectWithValue('Failed');
  return res.json();
});
```

**Interview tip:** If asked about data fetching in modern Redux, say that for server data you would use RTK Query first, and keep thunks for custom flows that do not fit it.

---

### 5.9 What is Redux Saga?

**Short answer:** Redux Saga is a middleware that handles side effects using generator functions. You describe async steps with effects like `call`, `put`, and `takeLatest`, and the saga middleware runs them.

**Explanation:**
- A saga is a generator (`function*`). Each `yield` produces an effect, which is a plain object describing what to do. The middleware executes it and resumes the generator.
- **Key effects:**
  - `call(fn, ...args)`: call a function or promise and wait for the result.
  - `put(action)`: dispatch an action.
  - `takeLatest(pattern, saga)`: run the saga on each matching action and cancel the previous run if it is still going (good for search or fetch).
  - `takeEvery`: run on every matching action.
  - `select`: read from state.
- **Why teams use it:** cancellation, debouncing, racing, retries, and complex orchestration. Because effects are plain objects, sagas are easy to test without mocking network calls.
- **Cost:** more concepts and more code than thunks.

```js
import { call, put, takeLatest } from 'redux-saga/effects';

function* fetchUserSaga(action) {
  try {
    const user = yield call(api.fetchUser, action.payload);
    yield put({ type: 'user/loaded', payload: user });
  } catch (e) {
    yield put({ type: 'user/failed', error: e.message });
  }
}

export function* rootSaga() {
  yield takeLatest('user/fetchRequested', fetchUserSaga);
}
```

**Interview tip:** Be ready for "thunk vs saga". Thunk is simple and fine for most cases. Saga fits complex, long-running, or cancellable workflows. Many modern apps use RTK Query and the listener middleware instead.

---

### 5.10 What is RTK Query?

**Short answer:** RTK Query is a data fetching and caching tool built into Redux Toolkit. You define endpoints, and it gives you auto-generated hooks plus caching, request deduplication, and tag-based invalidation.

**Explanation:**
- It solves server state, which is data owned by the backend that you cache on the client. Server state has different needs than UI state: loading and error flags, staleness, refetching, and invalidation.
- **Caching:** results are stored by endpoint and arguments, so a second component asking for the same data reuses it.
- **Deduplication:** if several components request the same data at once, only one network request is sent.
- **Tag-based invalidation:** queries declare `providesTags`, mutations declare `invalidatesTags`. When a mutation runs, matching queries refetch automatically.
- **Auto-generated hooks:** `useGetPostsQuery`, `useAddPostMutation`, and so on, which return `data`, `isLoading`, `error`, and more.
- It stores its cache in the Redux store, but you write no reducers or thunks yourself.

```js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post'],
  endpoints: (build) => ({
    getPosts: build.query({
      query: () => '/posts',
      providesTags: ['Post'],
    }),
    addPost: build.mutation({
      query: (body) => ({ url: '/posts', method: 'POST', body }),
      invalidatesTags: ['Post'], // triggers getPosts to refetch
    }),
  }),
});

export const { useGetPostsQuery, useAddPostMutation } = api;
// store: reducer: { [api.reducerPath]: api.reducer }, middleware: gDM => gDM().concat(api.middleware)

function Posts() {
  const { data, isLoading } = useGetPostsQuery();
  return isLoading ? <Spinner /> : data.map((p) => <Post key={p.id} {...p} />);
}
```

**Interview tip:** Compare it with TanStack Query. Both do server-state caching. RTK Query fits when you already use Redux, and TanStack Query is framework-agnostic with no Redux needed.

---

### 5.11 How do you manage global state in a large React application?

**Short answer:** First split state by type, then pick the lightest tool for each. Server state goes in TanStack Query or RTK Query, local UI state stays in components, and only truly shared client state goes in a store such as Redux Toolkit or Zustand.

**Explanation:**
Server state and client state are different problems. Server state is remote data you only borrow, so it needs caching, refetching, and invalidation. Client state is owned by the UI, such as modals, filters, and a wizard step. Putting server data into a Redux store by hand is a common cause of boilerplate and stale data.

| Kind of state | Examples | Tool |
|---|---|---|
| Server state | Lists, user profile, orders | TanStack Query or RTK Query |
| Local UI state | Input value, open dropdown | `useState` / `useReducer` |
| Shared client state | Cart, multi-step form, UI preferences | Redux Toolkit or Zustand |
| Low-change app-wide values | Theme, locale, auth user | Context |
| URL state | Filters, page, tab | Router / search params |
| Form state | Inputs, validation | React Hook Form or similar |

**Practical rules:**
- Keep state as close as possible to where it is used. Lift it only when needed.
- Avoid duplicating server data into a client store. Let the query cache own it.
- Do not store derived data. Compute it with selectors (`createSelector`).
- Organize by feature (one slice or store per feature), not by type.
- Use selectors so components subscribe only to what they need.

**Interview tip:** Lead with "I first classify the state". That one sentence signals experience, and the interviewer will usually follow up with a specific tool choice.

---

### 5.12 When would you choose Zustand or Redux over Context API?

**Short answer:** Choose Zustand or Redux when state is shared widely, updates often, or needs selector-based subscriptions to avoid re-render storms. Context is fine for low-frequency values like theme or the logged-in user.

**Explanation:**
- Context re-renders every consumer on any value change. A store lets each component subscribe to just the slice it reads.
- **Zustand:** tiny, no provider needed, simple API, selector-based subscriptions. Good for small-to-medium apps and quick setup.
- **Redux Toolkit:** best for large apps and teams: strict structure, DevTools, middleware, RTK Query, and well-known conventions.

| Aspect | Context | Zustand | Redux Toolkit |
|---|---|---|---|
| Provider required | Yes | No | Yes (`Provider`) |
| Selector subscriptions | No (whole value) | Yes | Yes (`useSelector`) |
| Boilerplate | Low | Very low | Moderate |
| Devtools / middleware | No | Optional middleware, DevTools support | Built in, rich ecosystem |
| Async / data fetching | Manual | Manual in actions | Thunks, RTK Query |
| Best for | Theme, auth, locale | Small/medium shared UI state | Large apps, many developers |

```js
import { create } from 'zustand';

const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
}));

function CartCount() {
  // re-renders only when items.length changes
  const count = useCartStore((s) => s.items.length);
  return <span>{count}</span>;
}
```

**Interview tip:** Explain the trade-off rather than picking a favorite: "Context for rarely-changing values, Zustand when I want minimal setup, Redux Toolkit when the team and app are big enough to benefit from structure, DevTools, and RTK Query."


---

## 6. Architect-Level React

> **Before the interview:** the "Example story" blocks are templates. Swap in your real projects, numbers and tools. Interviewers dig into stories, so only tell ones you lived.

### 6.1 Structuring a large React app for multiple teams

**30-second answer:** I start with the simplest thing that gives teams independence: a feature-based folder structure inside a monorepo (Nx or Turborepo), with a shared design-system package and enforced module boundaries. Teams own vertical slices (feature = UI + state + API + tests). I only move to micro-frontends with Module Federation when teams truly need independent deploy cycles, different release cadences, or separate tech stacks. Otherwise a well-governed monorepo gives 90% of the benefit with far less runtime complexity.

**Deep dive:**

#### Layer 1 - Feature-based folders (not type-based)
- Type-based (`components/`, `hooks/`, `services/`) scales badly: a single feature is scattered across 5 folders and everyone touches everything.
- Feature-based keeps code that changes together, together. A team owns a folder.
- Rule: a feature exposes a **public API** (`index.ts`); other features import only from that, never from deep paths.

```
apps/
  web/                    # thin shell: routing, providers, layout
packages/
  ui/                     # design system (Button, Modal, tokens)
  utils/                  # pure helpers
  api-client/             # generated typed client (OpenAPI/GraphQL codegen)
  features/
    checkout/
      index.ts            # public API only
      components/ hooks/ api/ state/ __tests__/
    catalog/
    account/
```

#### Layer 2 - Monorepo (Nx / Turborepo)
| Concern | Nx | Turborepo |
|---|---|---|
| Task caching + affected-only builds | Yes (computation cache, `nx affected`) | Yes (remote cache, `--filter`) |
| Module boundary enforcement | Built-in ESLint rule `@nx/enforce-module-boundaries` with tags | Not built-in; use ESLint `import/no-restricted-paths` or similar |
| Generators/scaffolding | Strong | Minimal |
| Learning curve | Higher, more opinionated | Low, thin layer over package workspaces |

- Benefits: atomic cross-package changes, one dependency version policy, shared tooling, only rebuild/test what a PR affects, remote cache cuts CI time.
- Cost: CI and repo size need care; needs ownership rules (CODEOWNERS) so "everyone can edit everything" does not become chaos.

#### Layer 3 - Shared design system
- Separate versioned package: tokens (colour, spacing, type) -> primitives -> composed components. Documented in Storybook.
- Headless primitives (Radix/React Aria) for accessibility; teams skin via tokens.
- Trade-off: a central DS team can become a bottleneck. Mitigate with inner-source contribution model and a clear "promote to DS only when used by 2+ teams" rule.

#### Layer 4 - Micro-frontends (Module Federation)
- Module Federation (Webpack 5 built in; also Rspack, and a Vite plugin via `@module-federation/vite`) lets a **host** load **remote** bundles at runtime and share dependencies (React as singleton).
- Each team builds and deploys its remote independently; the host picks up the new version without a rebuild.

**When to use**
- Many teams (roughly 5+), separate deploy pipelines needed, different release cadence.
- Gradual migration of a legacy app (strangler pattern), or merging acquired products.

**When NOT to use**
- One or two teams, or a team that can ship from a monorepo with fast CI.
- Heavy shared state between "micro" pieces (you will rebuild a distributed monolith).
- Performance-critical, SEO-heavy pages: more network hops, duplicate dependencies, harder SSR.
- Costs: version skew of shared libs (two React copies break hooks), inconsistent UX, harder debugging and e2e testing, runtime contract failures that the compiler cannot catch, extra infra (remote hosting, fallback when a remote is down).
- Mitigations if you do use it: singleton `react`/`react-dom`, typed contracts for exposed modules, error boundaries + fallback around each remote, contract tests, shared design system to keep UI consistent.

#### Decision ladder
| Situation | Pick |
|---|---|
| 1 team, one product | Single app, feature folders |
| 2-6 teams, same release train | Monorepo + feature libs + DS + boundaries |
| Many teams, independent deploys needed | Monorepo + Module Federation (or route-level composition) |
| Legacy migration | MFE as strangler, retire it later |

**What the interviewer wants to hear:**
- Micro-frontends solve an organisational problem (deploy independence), not a code-structure problem; start with a modular monorepo.
- You know the real costs (shared deps, UX drift, observability) and have mitigations.
- Boundaries are enforced by tooling, not by convention.

**Example story:** "In my last project, we had four squads on one Next.js codebase and merge conflicts and surprise regressions were constant. We moved to an Nx monorepo with feature libs, tagged them by domain, and added the module-boundary lint rule so `checkout` could not import `catalog` internals. We extracted a design-system package with Storybook. CI used `nx affected` plus remote cache, which cut PR checks from about 25 to 8 minutes. We evaluated Module Federation, but all squads shared one release train, so we deliberately did not adopt it; we kept it as an option for the acquired product that had its own pipeline."

---

### 6.2 State management: local, Context, Redux/Zustand, TanStack Query

**30-second answer:** The key insight is that server state and client state are different problems. Server state is remote data you do not own: it is async, can go stale, needs caching, deduping, retries and invalidation. That belongs in TanStack Query. Client state is data the browser owns: UI toggles, form drafts, wizard steps. That goes in local state first, then Context for low-frequency shared values, then Zustand or Redux Toolkit only when many distant components need frequently-changing shared client state. Most apps that "need Redux" actually needed a server-state cache.

**Deep dive:**

#### Server state vs client state
| | Server state | Client state |
|---|---|---|
| Owner | Backend (source of truth) | The browser |
| Nature | Async, shared, can be stale | Synchronous, ephemeral |
| Needs | Caching, dedupe, background refetch, retries, pagination, invalidation, optimistic updates | Simple read/write, persistence sometimes |
| Tool | TanStack Query (or SWR, RTK Query, Apollo) | useState/useReducer, Context, Zustand, Redux |
| Anti-pattern | Copying fetched data into Redux/useState, then hand-syncing | Putting UI flags in the query cache |

- Hand-rolled `useEffect` + `useState` fetching causes race conditions, no cache, duplicate requests, and loading/error boilerplate everywhere.
- TanStack Query gives: `staleTime`/`gcTime`, request dedupe by query key, refetch on focus/reconnect, `invalidateQueries` after mutations, optimistic updates via `onMutate`, pagination/infinite queries, and prefetch/hydration for SSR.

#### Decision table
| Need | Use | Why / when NOT |
|---|---|---|
| Value used by one component (input, toggle) | `useState` / `useReducer` | Default. Do not lift or globalise early. |
| Shared by a small subtree, rarely changes (theme, locale, auth user) | Context | Every consumer re-renders on any value change; do NOT use for fast-changing data (typing, cursor, big objects) unless split into contexts or memoised. |
| Derived from the URL (filters, page, tab) | URL / search params | Shareable, back-button friendly. Do not mirror it in a store. |
| Shared client state across distant components, updates often | Zustand (selectors = fine-grained subscriptions) | Light, little boilerplate. Less structure for huge teams. |
| Large team, complex client state, need time-travel/devtools, strict patterns, middleware | Redux Toolkit | More boilerplate; overkill for small apps. |
| Atomic, derived, fine-grained state graph | Jotai/Recoil-style atoms | Can scatter state; needs discipline. |
| Remote data from APIs | TanStack Query | Do not use for pure client state. |
| Form state | React Hook Form / local state | Avoid global store for form drafts. |

#### Practical architecture
- Server state: TanStack Query with query-key factories per feature, typed API client.
- Client state: small Zustand stores per feature (or Redux slices), local state by default.
- URL for navigational state.
- Avoid duplication: never copy query data into a store; derive with selectors (`select` option).

```tsx
// server state
const { data, isPending } = useQuery({
  queryKey: ['orders', { status }],
  queryFn: () => api.getOrders({ status }),
  staleTime: 60_000,
});

// client state
const useCartUi = create<{ open: boolean; toggle: () => void }>((set) => ({
  open: false,
  toggle: () => set((s) => ({ open: !s.open })),
}));
const open = useCartUi((s) => s.open); // selector: re-render only if `open` changes
```

- With RSC/Next App Router: fetch initial data in Server Components; hydrate/prefetch into TanStack Query only where client-side interactivity (mutations, polling, infinite scroll) needs the cache.

**What the interviewer wants to hear:**
- "Server state and client state are different problems", and you use a cache library for the first.
- Context is a dependency-injection mechanism, not a performant global store.
- You start local and escalate only when pain appears.

**Example story:** "In my last project the dashboard kept all API data in Redux with hand-written loading/error reducers, and stale data bugs were common. We moved all remote data to TanStack Query with key factories and invalidation after mutations, which deleted a large amount of reducer code. What remained in Redux was tiny, so we replaced it with a couple of Zustand stores. Filters moved to the URL. Bug count around stale data dropped and onboarding got easier."

---

### 6.3 Rendering strategies: CSR vs SSR vs SSG vs ISR vs RSC

**30-second answer:** It is a per-route decision driven by how personalised and how fresh the data is, and by SEO and performance needs. CSR for authenticated, highly interactive apps. SSG for content that rarely changes. ISR for content that changes occasionally and should stay cached. SSR for personalised or always-fresh pages needing SEO. React Server Components change the question: instead of choosing per page, you choose per component what runs on the server (data fetching, heavy libraries, zero client JS) and what ships to the client (interactivity). I pick Next.js when I need SEO, fast first paint, mixed strategies per route, and a full-stack framework.

**Deep dive:**

| Strategy | Where HTML is built | When | Pros | Cons | Use for |
|---|---|---|---|---|---|
| CSR | Browser | Runtime | Simple hosting, rich interactivity | Blank until JS loads, poor LCP and SEO unless mitigated | Dashboards, internal tools behind login |
| SSR | Server | Each request | Fresh + personalised, SEO, good LCP | Server cost, TTFB depends on data, hydration cost | Personalised feeds, product pages with live stock |
| SSG | Build | Build time | Fastest (CDN), cheap, secure | Rebuild for changes, not per-user | Docs, marketing, blog |
| ISR | Build + background regenerate | On a revalidate interval or on-demand | SSG speed with controlled freshness | Stale window, cache invalidation complexity | Catalogues, news, CMS content |
| RSC | Server (component-level), streamed | Per request or cached | Zero client JS for server components, direct data access, smaller bundles, streaming with Suspense | Mental model, serialisable props boundary, cannot use state/effects/browser APIs, needs a framework | Data-heavy UI mixed with small interactive islands |

#### Important clarifications
- RSC is not the same as SSR. SSR renders client components to HTML then hydrates them; RSC components never hydrate and never ship their JS. In Next.js App Router they work together: server components render on the server, client components (`'use client'`) are SSR'd and hydrated.
- In the App Router, caching and rendering mode are set per route/segment: static by default when no dynamic APIs are used; dynamic when using `cookies()`, `headers()`, or uncached fetches; `revalidate` / `revalidateTag` for ISR-style behaviour. Caching defaults changed across Next.js versions (14 to 15), so check the version in use.
- Streaming SSR with Suspense lets the shell render early and slow parts arrive later, which helps LCP and perceived speed.

#### When Next.js
- Need SEO or social previews, fast LCP on public pages.
- Mixed rendering per route, image/font optimisation, route-based code splitting, server actions/route handlers in one project.
- Team wants conventions rather than assembling Vite + router + SSR themselves.

#### When NOT Next.js
- Pure internal SPA behind login with no SEO: Vite + React Router/TanStack Router is simpler and cheaper to host.
- Team cannot operate a Node/edge runtime or does not want framework lock-in and upgrade churn.
- Embedded widgets or micro-frontend remotes (SSR with Module Federation is harder).

#### Mini decision flow
1. Public + SEO? No -> CSR is fine. Yes ->
2. Same for every user and rarely changes? -> SSG.
3. Changes occasionally? -> ISR.
4. Per-user or real-time? -> SSR (+ streaming), with RSC to keep client JS small.

**What the interviewer wants to hear:**
- The choice is per route and per component, not per app.
- RSC vs SSR is clearly distinguished.
- You can justify not using Next.js.

**Example story:** "In my last project the marketing and product pages needed SEO and good LCP, while the account area was a heavy dashboard. We used Next.js App Router: marketing pages static, product catalogue with ISR and on-demand revalidation from the CMS webhook, product detail as server components streaming reviews behind Suspense, and the account area as client-heavy components with TanStack Query. LCP on product pages improved noticeably and the client bundle on those routes shrank because data fetching and formatting libraries stayed on the server."

---

### 6.4 Finding and fixing a slow page: measure -> find -> fix -> verify

**30-second answer:** I never optimise by guessing. First I measure with real data: field data (CrUX/RUM) for Core Web Vitals and lab tools (Lighthouse, DevTools Performance) to reproduce. Then I find the bottleneck: is it network/bundle (LCP), long main-thread tasks or re-renders (INP), or layout shifts (CLS)? React Profiler tells me which components re-render and why. Then I apply the targeted fix, and finally verify with the same measurement and add a performance budget in CI so it does not regress.

**Deep dive:**

#### Step 1 - Measure
- Field: RUM (web-vitals library, your analytics), CrUX / Search Console. This is what users actually experience, at the 75th percentile.
- Lab: Lighthouse, WebPageTest, Chrome DevTools Performance panel with CPU/network throttling.
- Targets (good): **LCP < 2.5s**, **INP < 200ms** (INP replaced FID as a Core Web Vital in March 2024), **CLS < 0.1**.
- Record a baseline before any change.

#### Step 2 - Find
| Symptom | Tool | Look for |
|---|---|---|
| Slow load | Network tab, Lighthouse, bundle analyzer (`webpack-bundle-analyzer`, `@next/bundle-analyzer`) | Large JS, blocking requests, unoptimised hero image, slow TTFB |
| Janky interaction | DevTools Performance, React DevTools Profiler | Long tasks (>50ms), many re-renders, expensive commits |
| Layout jumps | Performance panel "Layout Shifts", Lighthouse | Images with no dimensions, late-injected banners, font swap |

React Profiler: record an interaction, check the flamegraph/ranked view, and enable "Record why each component rendered". Look for components that render often or for a long time, and ones that re-render with unchanged props.

#### Step 3 - Fix

**3a. Re-renders and memoisation**
- First fix the structure: move state down, lift expensive children up as `children` props, split contexts. This is free.
- `React.memo` skips re-render when props are shallow-equal. `useMemo` caches a computed value. `useCallback` caches a function identity (only useful when passed to a memoised child or used in a dependency array).
- **Costs:** each memo adds memory, comparison work, and code complexity; it is useless if a prop changes every render (new object/array/function literals); stale-closure bugs from wrong deps. Do not memoise by default; memoise where the Profiler shows a real win.
- React Compiler (stable in React 19 era tooling, opt-in) can auto-memoise, reducing manual `memo`/`useMemo`/`useCallback`; it still needs code that follows the Rules of React.

**3b. Code splitting**
```tsx
const Chart = lazy(() => import('./Chart'));
<Suspense fallback={<ChartSkeleton />}>
  <Chart />
</Suspense>
```
- Split by route first, then by heavy, rarely-used widgets (charts, editors, modals). Prefetch on hover/idle for likely next routes.
- Not worth it for tiny components: extra requests can cost more than they save.

**3c. List virtualization**
- Rendering thousands of rows creates thousands of DOM nodes. Use a windowing library (`react-window`, `@tanstack/react-virtual`) to render only visible rows.
- When NOT: short lists (< a few hundred simple rows), or when you need native find-in-page, or SEO on the list content; consider pagination instead.

**3d. Core Web Vitals - what hurts and how to fix**
| Metric | What hurts it | Fixes |
|---|---|---|
| **LCP** (<2.5s) | Slow TTFB, render-blocking CSS/JS, big hero image, client-only rendering of the LCP element, late-discovered resources | SSR/SSG or RSC for the above-the-fold, CDN, `priority` on the hero image (`next/image`) / `fetchpriority="high"` / preload, responsive modern image formats (AVIF/WebP), inline critical CSS, trim JS |
| **INP** (<200ms) | Long main-thread tasks, heavy event handlers, big re-renders on interaction, large hydration, third-party scripts | Break up work (`scheduler.yield` where available, `setTimeout`), `useTransition`/`useDeferredValue` for non-urgent updates, reduce re-renders, move heavy work to a Web Worker, defer third-party scripts, less client JS via RSC |
| **CLS** (<0.1) | Images/iframes/ads without dimensions, late-loaded content pushing layout, web-font swap, injected banners | Set `width`/`height` or `aspect-ratio`, reserve space for dynamic content/skeletons, `font-display` + size-adjusted fallbacks (`next/font` does this), animate with `transform` not layout properties |

#### Step 4 - Verify
- Re-measure in the same conditions; compare to baseline. Confirm in field data after rollout (it takes time to appear in CrUX; RUM is faster).
- Add guardrails: Lighthouse CI / bundle-size budgets in CI, RUM alerts on p75 regressions.
- Keep a short note of what was tried and what actually moved the metric.

**What the interviewer wants to hear:**
- A disciplined process driven by measurement, not folklore.
- You know memoisation has a cost and structural fixes come first.
- You can map each Web Vital to causes and fixes, and know INP (not FID) is current.

**Example story:** "In my last project the product listing felt slow. Field data showed p75 LCP of 4.1s and INP of 380ms. Lighthouse and the bundle analyzer showed a 600KB chart library loaded on every page, and the hero image was lazy-loaded. The Profiler showed each filter click re-rendering all 2,000 rows. We lazy-loaded the chart, made the hero image high priority, virtualised the list, and wrapped the filter-results update in `useTransition`. Post-release RUM showed LCP at 2.2s and INP at about 150ms, and we added a bundle budget to CI so it would not creep back."

---

### 6.5 Reconciliation, the virtual DOM, keys, and concurrent rendering

**30-second answer:** The virtual DOM is a lightweight tree of JS objects describing the UI. When state changes, React renders a new tree, diffs it against the previous one (reconciliation), and applies the minimal set of real DOM changes in the commit phase. The diff uses two heuristics: elements of different types produce different trees, and keys identify which children are the same across renders. Wrong keys cause bugs and wasted work. Concurrent rendering makes rendering interruptible so urgent updates like typing are not blocked by slow, non-urgent ones; `useTransition` is how you mark an update as non-urgent.

**Deep dive:**

#### Virtual DOM and reconciliation
- Render phase: call components, produce a new element tree (pure, can be paused/discarded in concurrent mode). Commit phase: apply DOM mutations and run layout effects (synchronous, cannot be interrupted); `useEffect` callbacks run afterwards, usually after paint.
- A general tree diff is O(n^3); React uses heuristics for O(n):
  1. Different element type at the same position -> tear down the old subtree (state lost), build a new one.
  2. Same type -> keep the instance, update props, recurse into children.
  3. Lists use **keys** to match children between renders.
- The virtual DOM is not "faster than the DOM" by itself; its value is a declarative model with batched, minimal updates. Fine-grained reactive libraries (Solid, Svelte) can skip the diff, which is a fair trade-off to mention.
- React Fiber is the internal architecture: a linked work-unit structure that makes rendering incremental and prioritisable, which is what enables concurrent features.

#### Why keys matter
- Key = stable identity of an item among siblings. React uses it to decide whether to reuse, move, or remove an instance (and its state).
- Index as key: if items are reordered, inserted, or deleted, indexes shift and React reuses the wrong instance; local state and uncontrolled inputs end up attached to the wrong row. Fine only for static lists that never reorder or filter.
- Random keys (`Math.random()`, new `uuid()` each render): a new key every render means remount of every item (lost state/focus, poor performance).
- Good key: stable, unique among siblings, e.g. database id.
- Changing a key on purpose is a valid way to reset a component's state: `<Form key={userId} />`.

```tsx
// Bug-prone when list can reorder/filter
items.map((item, i) => <Row key={i} item={item} />);
// Correct
items.map((item) => <Row key={item.id} item={item} />);
```

#### Concurrent rendering
- Problem in React 17 and earlier: once rendering starts it cannot be interrupted, so a heavy render (filtering 10k items) blocks the main thread and typing feels laggy.
- React 18 concurrent rendering (enabled via `createRoot`): React can start a render, yield to the browser for urgent input, then resume or discard stale work.
- Tools: `useTransition` / `startTransition` (mark updates as non-urgent), `useDeferredValue` (defer a value), Suspense for data/code with streaming. In React 19, transitions can also wrap async functions (Actions), with `useActionState` and `useOptimistic` building on this.
- It does not make code faster; it improves responsiveness by prioritising. Expensive renders still cost CPU. Fix the slow render too when possible (virtualise, memoise).

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [filter, setFilter] = useState('');
  const [isPending, startTransition] = useTransition();

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value);                         // urgent: keep input responsive
    startTransition(() => setFilter(e.target.value)); // non-urgent: heavy list can be interrupted
  }

  const results = useMemo(
    () => items.filter((i) => i.name.includes(filter)),
    [items, filter]
  );

  return (
    <>
      <input value={query} onChange={onChange} />
      {isPending && <Spinner />}
      <List items={results} />
    </>
  );
}
```
- Notes: state updates inside `startTransition` must be synchronous-called setters (in React 18); transitions cannot control text inputs themselves (the controlled input update must stay urgent); if the transition's Suspense boundary is already showing content, React keeps the old UI instead of falling back to a spinner.
- `useTransition` vs debounce: debounce delays work by a fixed time; transition starts immediately but is interruptible. They can be combined, and debounce still helps to avoid network calls.

**What the interviewer wants to hear:**
- Render phase vs commit phase, and why render must be pure.
- Keys express identity, not just "silence the warning".
- Concurrent rendering is about prioritisation and interruptibility, not raw speed.

**Example story:** "In my last project an admin table had a search box that froze while typing because every keystroke re-filtered and re-rendered thousands of rows. We kept the input state urgent and put the filter state into `startTransition`, and added `isPending` styling. Typing became smooth. While there, we found rows keyed by array index caused checkbox selections to jump to the wrong row after sorting; switching to the record id fixed it."

---

### 6.6 Enforcing consistency across teams

**30-second answer:** Consistency has to be enforced by tooling in the pipeline, not by documents people forget. I layer it: ESLint, Prettier and strict TypeScript catch code-level issues in the editor and CI; design tokens and a Storybook-documented design system keep UI consistent; CODEOWNERS, PR templates and required CI checks enforce review and quality gates; and ADRs record why we decided things so that decisions are discoverable and not re-argued.

**Deep dive:**

#### Code-level
| Tool | Purpose | Notes |
|---|---|---|
| Prettier | Remove formatting debates | Run on pre-commit (lint-staged + husky) and in CI |
| ESLint | Correctness and conventions | `eslint-plugin-react-hooks` (exhaustive-deps), `jsx-a11y`, `import` rules, `@typescript-eslint`; shared config as a package (`@org/eslint-config`) |
| Custom ESLint rules | Encode your architecture | Ban deep imports across features, ban direct use of raw `fetch` (use the API client), ban `<button>` where `<Button>` from DS exists, no hardcoded colours |
| TypeScript `strict` | Catch null/any bugs | Enable `strict`, `noUncheckedIndexedAccess`, and ban `any` via lint; shared `tsconfig.base.json`; roll out gradually with `// @ts-expect-error` budget on legacy code |
| Module boundaries | Prevent spaghetti | Nx tags or `import/no-restricted-paths` |

- Trade-off: too many strict rules slow people down and get disabled with `eslint-disable`. Start with warnings, ratchet to errors, and make auto-fixable rules the default. Track `eslint-disable` count.

#### UI-level
- **Design tokens** (colour, spacing, typography, radius, motion) as a single source of truth (JSON, e.g. Style Dictionary), compiled to CSS variables/Tailwind theme/native. Theming (dark mode, brands) becomes a token swap, not a rewrite.
- **Design-system package** with versioned releases and a changelog (Changesets); components built on accessible primitives.
- **Storybook**: living documentation, visual states, a11y addon, interaction tests, and visual regression (Chromatic/Playwright snapshots). Stories double as the contract between design and engineering.

#### Process-level
- **PR template:** what/why, screenshots, test plan, risk, a11y checked, feature flag/rollback notes.
- **CODEOWNERS:** DS folder owned by the DS team, shared libs by platform, each feature by its squad; required review from owners on those paths.
- **CI required checks:** lint, type-check, unit/integration tests, Storybook build, a11y checks, bundle-size budget, Lighthouse CI on key routes, affected-only via Nx/Turborepo to keep it fast. Branch protection so none can be bypassed.
- **ADRs (Architecture Decision Records):** short markdown in the repo (context, decision, consequences, status) for decisions like "we use TanStack Query for server state" or "we adopt Module Federation for X". They give new joiners the why and make reversals explicit with a superseding ADR.
- **Golden paths:** generators/templates (`nx g feature`) so the easy way is the right way. Consistency from convenience beats consistency from policing.

#### When NOT to over-enforce
- Early-stage product with one small team: heavy governance is overhead; keep Prettier, ESLint, strict TS, and a simple PR template.
- Do not add a rule you cannot automate or explain; unenforced rules rot.

**What the interviewer wants to hear:**
- Automation over documentation; make the right thing the easy thing.
- Governance is balanced with developer velocity (warnings-to-errors ratchet, fast CI).
- ADRs capture decisions and reasoning.

**Example story:** "In my last project, five squads had five button styles and three data-fetching patterns. We created a shared ESLint config with a custom rule banning raw `fetch` in components and a rule forbidding cross-feature deep imports, introduced design tokens and a Storybook-backed design system, and added CODEOWNERS plus a PR template. We rolled lint rules out as warnings for a sprint, then flipped them to errors with codemods for the migration. We wrote ADRs for the big calls, such as TanStack Query for server state. Within a quarter, UI review comments about spacing and style dropped sharply, and new hires could ship in their first week using the generators."


---

