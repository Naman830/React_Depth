# 07. useCallback

> `useCallback` remembers the **function itself** between renders, so the child components and effects that receive it do not see a "new" function every time.

---

## 1. Real-life analogy

You hire a courier and give them your address on a card.

Every morning you write the address on a **fresh card** and hand it over. The address is identical — same street, same house number — but the card is new. The courier is a careful person: their rule is "if I get a new card, I re-plan the whole route from scratch." So every morning they spend an hour re-planning a route that did not change.

The fix is not to change the address. The fix is to hand over **the same card** each day. Then the courier looks at it, says "this is the card I already have", and skips the re-planning entirely.

That is `useCallback`:

| Courier | React |
|---|---|
| The address written on the card | what your function *does* |
| The physical card | the function **object** in memory |
| "New card → re-plan the route" | `React.memo` seeing a changed prop → re-render |
| Handing over the same card daily | `useCallback` returning the same function |
| Moving house | a dependency changing → a genuinely new function |

The whole hook lives in one uncomfortable fact: React compares the **card**, not the **address**. Two functions that do exactly the same thing are still two different objects, and React has no way to tell they are equivalent.

**In simple words:** `useCallback` hands out the same function object again instead of making a new one that behaves identically.

---

## 2. The problem — why does this exist?

### Functions are objects, and objects are compared by reference

This is the fact everything else follows from.

```jsx
const a = () => console.log("hi");
const b = () => console.log("hi");

a === b; // false — identical behaviour, different objects
```

Two functions with the exact same body are **not equal** in JavaScript. Equality for objects means "is this the same object in memory", not "do these look the same".

Now remember from note 06: **the whole component body re-runs on every render**. That includes every function you declare inside it.

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // A brand-new function object on EVERY render
  function handleSearch(term) {
    console.log(term);
  }

  return <SearchBox onSearch={handleSearch} />;
}
```

Render 1 creates `handleSearch` #1. Render 2 creates `handleSearch` #2. Same code, different object.

### Problem A — `React.memo` on the child stops working

`React.memo` wraps a component and says "skip re-rendering me if my props are the same as last time". It compares props with `Object.is` — the same shallow reference check `useMemo` uses.

```jsx
const SearchBox = memo(function SearchBox({ onSearch }) {
  console.log("SearchBox rendered"); // this fires every single time
  return <input onChange={(e) => onSearch(e.target.value)} />;
});

function Parent() {
  const [count, setCount] = useState(0);

  function handleSearch(term) { /* ... */ } // new object each render

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <SearchBox onSearch={handleSearch} />
    </>
  );
}
```

Click the counter button. `count` changes, `Parent` re-renders, `handleSearch` is recreated, `SearchBox` receives a prop that "changed", and `memo` re-renders it anyway.

The `memo` looks like an optimisation and does **nothing**. This is the single most common wasted `memo` in real React codebases — the exact same trap as passing `options={{ a: 1 }}`, but with a function instead of an object.

### Problem B — an effect runs on every render

```jsx
function Chat({ roomId }) {
  const [message, setMessage] = useState("");

  function connect() {
    return openConnection(roomId);
  }

  useEffect(() => {
    const conn = connect();
    return () => conn.close();
  }, [connect]); // ❌ `connect` is new every render
  // -> disconnect + reconnect on every keystroke in the message box
}
```

Type one character. `message` changes → re-render → new `connect` → the dependency array looks different → cleanup runs, effect runs again. You just closed and reopened a socket because someone typed the letter "h".

This class of bug is worse than problem A. Problem A is slowness you might not notice. Problem B is a socket reconnecting, a fetch firing in a loop, or an infinite render cycle.

### Problem C — a function inside a custom hook's dependencies

The same trap, one level deeper:

```jsx
function useSearch(onResult) {
  useEffect(() => {
    fetchResults().then(onResult);
  }, [onResult]); // caller passes an inline arrow -> refetches forever
}
```

### What we actually want

A way to say: *"give me back the same function I made last time, unless these specific values changed."* That is `useCallback`.

> ⚠️ Like `useMemo`, `useCallback` is a **performance** tool with one exception: when it feeds a dependency array, removing it can change behaviour (extra effect runs). Even then, the correct app must not *depend* on the memoization to be right.

**In simple words:** functions are new objects on every render, and that breaks `memo` children and dependency arrays — `useCallback` keeps the old object.

---

## 3. What it actually is

```jsx
const memoizedFn = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

Three parts, exactly like `useMemo`:

1. **The function you want to keep.** React does **not** call it. It just stores it.
2. **A dependency array** — the values the function reads from the render it was created in.
3. **The return** — your function, or the one kept from the last render.

React's logic:

```
On every render:
  compare each dependency with last render's value (Object.is)
  ├─ all the same  -> return the STORED function (your new one is thrown away)
  └─ any different -> store the NEW function and return it
```

### It is literally `useMemo` returning a function

```jsx
useCallback(fn, deps)  ===  useMemo(() => fn, deps)
```

That is not an analogy — it is the actual relationship. `useCallback` exists only because `useMemo(() => fn, deps)` is awkward to read and easy to get wrong (people write `useMemo(() => fn(), deps)` by mistake, which memoizes the *result*).

| | You pass | React does | You get back |
|---|---|---|---|
| `useMemo` | a function | **calls** it | its return value |
| `useCallback` | a function | **stores** it | the function |

### The new function is still created every render

This surprises people. `useCallback` does **not** stop the arrow function from being created:

```jsx
const handle = useCallback(() => setCount(c => c + 1), []);
//                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ this arrow IS created every render
```

JavaScript evaluates the argument before `useCallback` runs. React then looks at the deps, decides nothing changed, **throws your new function away**, and returns the old one.

So the saving is never "we avoided allocating a function". Allocating a function is cheap. The saving is entirely downstream: the *reference* stays stable, so `memo` skips and effects do not re-run.

### The stored function captures old values

Because React keeps the function from an earlier render, that function still "remembers" the variables from **that** render. This is a JavaScript closure, and it is the source of every `useCallback` bug in section 8.

```jsx
const [count, setCount] = useState(0);

const log = useCallback(() => {
  console.log(count); // captured from the render where this was created
}, []); // empty deps -> frozen at count = 0 forever
```

**In simple words:** `useCallback(fn, deps)` stores your function and hands the stored one back until a dependency changes.

---

## 4. Syntax / setup, step by step

### Step 1 — write the function normally first

Never start with `useCallback`. Make it work, then find out if it is a problem.

```jsx
function handleAddTodo(text) {
  setTodos([...todos, { id: Date.now(), text }]);
}
```

Most functions in most components should stay exactly like this. Passing a fresh function to a plain `<button onClick={...}>` costs nothing — DOM elements do not re-render because of a changed handler reference.

### Step 2 — check whether it actually matters

Wrap it only if **one** of these is true:

```
Is the function passed to a component wrapped in React.memo?
Is the function listed in a useEffect / useMemo / useCallback dependency array?
Is the function returned from a custom hook (so callers may depend on it)?
   └─ any yes -> useCallback is worth it
   └─ all no  -> leave it alone
```

If the answer to all three is no, `useCallback` is pure overhead: an extra array to store and compare on every render, for a benefit nobody collects.

### Step 3 — wrap it

```jsx
import { useCallback } from "react";

const handleAddTodo = useCallback((text) => {
  setTodos((prev) => [...prev, { id: crypto.randomUUID(), text }]);
}, []); // setTodos is stable -> nothing to list
```

Note the shape: the whole function goes **inside** the arrow you pass. A frequent slip is wrapping the call rather than the definition.

```jsx
useCallback(handleAdd, [])      // ✅ stores handleAdd
useCallback(() => handleAdd, []) // ❌ returns a function that returns handleAdd
useCallback(handleAdd(), [])     // ❌ calls it now, stores the result
```

### Step 4 — get the dependency array right

Same rule as `useMemo`: **every reactive value the function body reads must be listed.** Reactive means props, state, context, or anything derived from them.

```jsx
const submit = useCallback(() => {
  api.post(`/rooms/${roomId}`, { text, author: user.name });
}, [roomId, text, user.name]); // reads three outside values -> lists three
```

Values you do **not** list, because their identity never changes:

| Always stable — never list it | Why |
|---|---|
| `setState` setters from `useState` | React guarantees the same function |
| `dispatch` from `useReducer` | same guarantee |
| `ref` objects from `useRef` | the object identity never changes |
| Module-level constants and imports | created once, outside the component |

### Step 5 — use the updater form to remove dependencies

This is the most useful trick in the whole note. If your function only needs the *current* state to compute the *next* state, the updater form removes the dependency completely:

```jsx
// ❌ depends on todos -> a new function on every todo change
const addTodo = useCallback((text) => {
  setTodos([...todos, { text }]);
}, [todos]);

// ✅ no dependency at all -> the same function forever
const addTodo = useCallback((text) => {
  setTodos((prev) => [...prev, { text }]);
}, []);
```

The first version is memoized in name only — `todos` changes constantly, so the function changes constantly, so the `memo` child re-renders anyway. The second is genuinely stable.

> 💡 Whenever a `useCallback` dependency array contains a piece of state that the function only reads in order to update it, reach for the updater form.

### Step 6 — let the linter check you

```bash
npm install --save-dev eslint-plugin-react-hooks
```

The `exhaustive-deps` rule reads the function body and tells you which dependencies are missing. Trust it over your own reading; missing deps in a `useCallback` produce silent stale-value bugs, not crashes.

**In simple words:** write it plain, wrap it only when a `memo` child or a dependency array is watching, list every value it reads, and use the updater form to cut the list down.

---

## 5. Full working example (with comments)

A todo app with a memoized list row. The console proves which rows re-render and when.

```jsx
import { useCallback, useState, memo } from "react";

// ============================================================
// A memoized row. It re-renders ONLY when one of its props
// changes by reference. Watch the console to see this happen.
// ============================================================
const TodoRow = memo(function TodoRow({ todo, onToggle, onDelete }) {
  console.log("render row:", todo.text); // 👈 the whole point of the demo

  return (
    <li>
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => onToggle(todo.id)}
      />
      <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
        {todo.text}
      </span>
      <button onClick={() => onDelete(todo.id)}>delete</button>
    </li>
  );
});

// ============================================================
// A memoized footer. `theme` is a plain string, so memo works
// on it without any help.
// ============================================================
const Footer = memo(function Footer({ count, onClearDone }) {
  console.log("render footer");
  return (
    <div>
      <span>{count} left</span>
      <button onClick={onClearDone}>Clear completed</button>
    </div>
  );
});

function TodoApp() {
  const [todos, setTodos] = useState([
    { id: 1, text: "Learn useState", done: true },
    { id: 2, text: "Learn useEffect", done: false },
    { id: 3, text: "Learn useCallback", done: false },
  ]);
  const [draft, setDraft] = useState("");   // changes on EVERY keystroke
  const [dark, setDark] = useState(false);  // unrelated to the list

  // ----------------------------------------------------------
  // All three handlers use the UPDATER form of setTodos.
  // That means they never read `todos` directly, so the
  // dependency array is empty and the function is created ONCE
  // for the whole life of the component.
  // ----------------------------------------------------------
  const handleToggle = useCallback((id) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t))
    );
  }, []); // ✅ empty — setTodos is stable, nothing else is read

  const handleDelete = useCallback((id) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  }, []);

  const handleClearDone = useCallback(() => {
    setTodos((prev) => prev.filter((t) => !t.done));
  }, []);

  // ----------------------------------------------------------
  // This one DOES need a dependency: it reads `draft`, which is
  // a value from render, not something setTodos can give us.
  // So it is recreated whenever `draft` changes — that is
  // correct, not a failure.
  // ----------------------------------------------------------
  const handleAdd = useCallback(() => {
    if (!draft.trim()) return;
    setTodos((prev) => [...prev, { id: Date.now(), text: draft, done: false }]);
    setDraft("");
  }, [draft]);

  const remaining = todos.filter((t) => !t.done).length;

  return (
    <div style={{ background: dark ? "#222" : "#fff", color: dark ? "#eee" : "#111" }}>
      {/* Typing here re-renders TodoApp on every keystroke.
          Because the row handlers are stable, NO row re-renders. */}
      <input value={draft} onChange={(e) => setDraft(e.target.value)} />
      <button onClick={handleAdd}>Add</button>

      <button onClick={() => setDark((d) => !d)}>Toggle theme</button>

      <ul>
        {todos.map((todo) => (
          <TodoRow
            key={todo.id}
            todo={todo}          // only THIS todo's object changes on toggle
            onToggle={handleToggle}
            onDelete={handleDelete}
          />
        ))}
      </ul>

      <Footer count={remaining} onClearDone={handleClearDone} />
    </div>
  );
}

export default TodoApp;
```

### What the console proves

**Type a letter in the input.** `TodoApp` re-renders. The console shows… nothing. No row logs, no footer log. Every row's props (`todo`, `onToggle`, `onDelete`) are reference-identical to last render, so `memo` skipped all three rows.

**Click "Toggle theme".** Same result — no rows re-render.

**Tick one checkbox.** Exactly **one** row logs. `handleToggle` produced a new object only for that todo; the other two `todo` objects are the same references as before, so those rows skip. The footer logs too, because `count` changed.

**Now delete the three `useCallback` wrappers** and repeat. Every keystroke logs all three rows plus the footer. Three rows is nothing; three hundred rows is a visibly laggy input box.

> ⚠️ `useCallback` alone would achieve nothing here. It works *because* `TodoRow` is wrapped in `memo`. The two are a pair — one without the other is wasted code.

**In simple words:** stable handlers plus `memo` rows means typing in one box does not re-render the entire list.

---

## 6. How it works behind the scenes

### The hook slot

Like every hook, `useCallback` gets a **slot** on the component's fiber — the internal object React keeps per component instance. The slot holds two things: the last dependency array, and the stored function.

```
Fiber for <TodoApp>
├── hook #1  useState     -> [todos]
├── hook #2  useState     -> "lear"
├── hook #3  useState     -> false
├── hook #4  useCallback  -> { deps: [],        fn: handleToggle_v1 }
├── hook #5  useCallback  -> { deps: [],        fn: handleDelete_v1 }
├── hook #6  useCallback  -> { deps: [],        fn: handleClearDone_v1 }
└── hook #7  useCallback  -> { deps: ["lear"],  fn: handleAdd_v4 }
```

Slots are matched by **call order**, never by name — which is exactly why the rules of hooks forbid calling `useCallback` inside an `if`.

### The comparison, render by render

```
render 3          render 4 (user typed "r")
--------          -------------------------
deps: []          deps: []           -> same     -> return handleToggle_v1
deps: ["lea"]     deps: ["lear"]     -> DIFFERENT -> store & return handleAdd_v4
```

Comparison is shallow `Object.is`, item by item, exactly as in `useMemo` and `React.memo`. Three shallow comparisons in a row is the whole mechanism.

### What `React.memo` does with the result

```
Parent re-renders
      ↓
builds new JSX: <TodoRow todo={t} onToggle={fn} onDelete={fn} />
      ↓
React reaches a memo component
      ↓
compares each prop with the previous render's prop (Object.is)
      ├─ all equal      -> reuse the previous output, do NOT call the component
      └─ any different  -> call the component, reconcile as normal
```

The important detail: `memo` compares **every** prop. One unstable prop out of ten is enough to defeat it. That is why `useCallback` and `useMemo` usually appear together — you have to stabilise all the object-ish props, not just the easy one.

### Closures: why the deps array is not optional

Each render creates a new scope. A function created in render 3 permanently sees render 3's variables:

```
render 1: count = 0   fn_v1 closes over count = 0
render 2: count = 1   fn_v2 closes over count = 1
render 3: count = 2   fn_v3 closes over count = 2

useCallback(fn, [])  -> React keeps fn_v1 forever
                     -> calling it in render 3 still logs 0
```

This is not a React quirk; it is how JavaScript closures work. React just makes it visible by keeping an old function alive. The dependency array is how you tell React "this function is out of date now, take the new one".

### `useCallback` is not free

Every call costs, on every render:

- creating the inline function anyway (JavaScript evaluates the argument first),
- storing the dependency array,
- comparing the array element by element,
- memory held for the stored function as long as the component lives.

For a handler passed to a plain `<button>`, that is all cost and no benefit. This is why "wrap every function in `useCallback`" is actively bad advice rather than merely unnecessary.

### The React Compiler

React 19 ships an optional **compiler** that adds this memoization automatically at build time by analysing your code. Where it is enabled, most hand-written `useCallback` calls become unnecessary — the compiler is better at exhaustive dependency tracking than humans are.

It is not on by default everywhere, and understanding the hook by hand is still how you understand why a component re-renders. Learn it; be happy when a tool does it for you.

**In simple words:** React stores your function and its deps in a slot, shallow-compares the deps each render, and returns the old function until something moves.

---

## 7. Comparison with alternatives (table)

### The three memoization tools

| Tool | Wraps | Remembers | Use when |
|---|---|---|---|
| `useMemo(fn, deps)` | a calculation | the **value** returned | work is slow, or an object/array needs a stable reference |
| `useCallback(fn, deps)` | a function | the **function object** | the function goes to a `memo` child or into a deps array |
| `React.memo(Component)` | a component | its last **render output** | a child re-renders often with identical props |

They are a set. `React.memo` on the child is the thing that actually saves work; `useCallback` and `useMemo` in the parent exist to make the child's props stable enough for `memo` to succeed.

### Ways to avoid needing `useCallback` at all

| Approach | Cost | When it is the better answer |
|---|---|---|
| Do nothing | zero | The function goes to a DOM element (`<button onClick>`) — the usual case |
| Updater form: `setX(prev => ...)` | zero | The function only reads state to update it — removes the dependency |
| Move state down | zero | The fast-changing state can live in a smaller child, so the parent stops re-rendering |
| Pass JSX as `children` | zero | The expensive subtree does not depend on the changing state |
| Define the function outside the component | zero | It reads no props or state at all |
| `useCallback` | small per-render overhead | A `memo` child or a dependency array is genuinely watching |
| The React Compiler | zero at runtime | Available in your setup |

```jsx
// Reads nothing from the component -> hoist it, no hook needed
function formatPrice(n) {
  return `₹${n.toFixed(2)}`;
}

function Product({ price }) {
  return <p>{formatPrice(price)}</p>; // stable by construction
}
```

> 💡 The cheapest fix is almost never a hook. Hoisting a pure helper out of the component, or moving fast-changing state down, beats `useCallback` because it costs nothing per render.

### `useCallback` vs `useRef` for "the latest function"

There is a pattern where you want a function that is **always stable** *and* **always current** — for example a callback passed to a long-lived subscription. `useCallback` cannot do both: stable means frozen deps, current means changing identity.

```jsx
function useEventCallback(fn) {
  const ref = useRef(fn);
  useEffect(() => { ref.current = fn; });        // keep the latest version
  return useCallback((...args) => ref.current(...args), []); // stable wrapper
}
```

React has a built-in version of this planned (`useEffectEvent`), still experimental at the time of writing. Recognise the pattern; do not reach for it until a real case demands it.

**In simple words:** `useCallback` caches functions, `useMemo` caches values, `React.memo` caches renders — and the updater form or moving state down often removes the need for any of them.

---

## 8. Common mistakes beginners make

**1. `useCallback` without `React.memo` on the child**

```jsx
const handleClick = useCallback(() => {}, []);
return <PlainChild onClick={handleClick} />; // ❌ PlainChild is not memoized
```
The child re-renders regardless of prop identity, so the stable reference buys nothing. You paid the overhead and got zero. This is the most common wasted `useCallback` in real code.

**2. A dependency that changes every render**

```jsx
const add = useCallback((t) => setTodos([...todos, t]), [todos]);
```
`todos` changes on every add, so the function changes on every add, so the `memo` child re-renders anyway. Memoized in name only. Use `setTodos(prev => ...)` and drop the dependency.

**3. An inline arrow in the JSX, defeating the whole thing**

```jsx
<MemoChild onClick={() => handleClick(id)} />  // ❌ new arrow every render
<MemoChild onClick={handleClick} id={id} />    // ✅ pass the id as a prop instead
```
Wrapping `handleClick` in `useCallback` and then wrapping it in a fresh arrow at the call site cancels it out exactly.

**4. Empty deps when the function reads state**

```jsx
const submit = useCallback(() => {
  api.send(text); // reads `text`
}, []); // ❌ frozen at the first render's text — sends "" forever
```
This is the stale-closure bug, and it is silent: no error, just wrong data. The ESLint rule catches it.

**5. Wrapping the call instead of the definition**

```jsx
useCallback(handleAdd(), [])      // ❌ calls it during render
useCallback(handleAdd, [])        // ✅ stores it
```

**6. Listing stable values in the deps array**

```jsx
useCallback(() => setCount(c => c + 1), [setCount]); // harmless but noise
```
`setState` setters, `dispatch` and refs never change identity. Listing them is not a bug, just clutter that hides the deps that matter.

**7. Wrapping every function in the component**

```jsx
const onFocus = useCallback(() => setActive(true), []);   // goes to <input>
const onBlur  = useCallback(() => setActive(false), []);  // goes to <input>
```
DOM elements do not care about handler identity. React attaches one real listener at the root and looks the handler up when the event fires. Handlers going to `<div>`, `<button>`, `<input>` need no memoization ever.

**8. Expecting it to stop the function from being created**

The inline arrow is still allocated every render. `useCallback` throws it away. The benefit is downstream stability, never allocation.

**9. Forgetting that one unstable prop kills `memo`**

```jsx
<MemoChild onSave={stableFn} config={{ mode: "edit" }} /> // ❌ config is new
```
`memo` needs **all** props stable. Pair `useCallback` with `useMemo` for the object props.

**10. Using it for correctness**

```jsx
useEffect(() => { connect(); }, [connect]); // relies on connect being stable
```
If removing `useCallback` causes an infinite loop, the memoization is holding the app together. That is fragile. Prefer restructuring — move the function inside the effect, or use the updater form — so the effect does not depend on a function at all.

```jsx
useEffect(() => {
  function connect() { return open(roomId); } // defined INSIDE -> no dep needed
  const c = connect();
  return () => c.close();
}, [roomId]); // ✅ depends on a primitive
```

**In simple words:** the two real dangers are memoizing a function nobody is watching (pure waste) and empty deps on a function that reads state (stale values).

---

## 9. Cheat sheet

```jsx
// Stable handler for a memo child — updater form means empty deps
const handleDelete = useCallback((id) => {
  setItems((prev) => prev.filter((i) => i.id !== id));
}, []);

// A dependency that genuinely belongs
const handleSubmit = useCallback(() => {
  api.post(`/rooms/${roomId}`, { text });
}, [roomId, text]);

// Returned from a custom hook, so callers can safely put it in deps
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((o) => !o), []);
  return [on, toggle];
}

// Identical by definition
useCallback(fn, deps) === useMemo(() => fn, deps);
```

| Thing | Rule |
|---|---|
| Signature | `useCallback(fn, [deps])` |
| Returns | Your function, or the one stored last render |
| React calls it? | **No.** It only stores it (`useMemo` calls it) |
| Comparison | Shallow, `Object.is`, item by item |
| Cache size | One — only the latest function |
| Deps to include | Every prop, state or context value the body reads |
| Deps to skip | `setState` setters, `dispatch`, refs, module constants, imports |
| Empty deps `[]` | The function is frozen at the first render — safe only if it reads nothing reactive |
| Use it for | `memo` children, dependency arrays, functions returned from custom hooks |
| Do not use it for | Handlers on plain DOM elements — that is most handlers |
| Needs a partner | Yes — `React.memo` on the child, or a deps array |
| Stops allocation? | No. The arrow is still created; React discards it |

**The decision flow:**

```
Is this function passed to a component wrapped in React.memo?
├─ yes -> useCallback (and useMemo any object props too)
└─ no
   └─ Is it listed in a useEffect / useMemo / useCallback deps array?
      ├─ yes -> useCallback, or better: move it inside the effect
      └─ no
         └─ Is it returned from a custom hook for others to depend on?
            ├─ yes -> useCallback
            └─ no  -> plain function, no hook
```

**In simple words:** wrap it only when a `memo` child or a dependency array is watching, and use the updater form to keep the deps array empty.

---

## 10. Revision questions (with answers)

**1. What does `useCallback` return?**
The function you passed — either the new one, if a dependency changed, or the one stored from a previous render.

**2. Does React call the function you give `useCallback`?**
No. It only stores it. `useMemo` calls its function; `useCallback` does not.

**3. Write `useCallback` in terms of `useMemo`.**
`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`.

**4. Why does `handleClick` change on every render without it?**
Because the component body re-runs completely on every render, and a function declaration creates a **new object** each time. Two identical functions are never `===` in JavaScript.

**5. Does `useCallback` prevent the function from being created?**
No. JavaScript evaluates the argument first, so the arrow is allocated every render. React then throws it away and returns the stored one. The benefit is reference stability, not allocation.

**6. Why is `useCallback` useless without `React.memo` on the child?**
An unmemoized child re-renders whenever its parent does, regardless of prop identity. There is nothing for the stable reference to save.

**7. What is wrong with `useCallback((t) => setTodos([...todos, t]), [todos])`?**
`todos` changes on every add, so the function identity changes too, so any `memo` child re-renders anyway. Use `setTodos(prev => [...prev, t])` with `[]` deps.

**8. What is a stale closure, in one sentence?**
A stored function still sees the variables from the render it was created in, so with missing dependencies it reads old props and state.

**9. Which values never need to appear in a dependency array?**
`setState` setters, `dispatch` from `useReducer`, `useRef` objects, module-level constants, and imports — their identity never changes.

**10. Why does `<MemoChild onClick={() => handle(id)} />` defeat a `useCallback`?**
The inline arrow is a new object every render, so the prop always differs and `memo` never skips — regardless of how stable `handle` is.

**11. Do handlers on `<button onClick={...}>` need `useCallback`?**
No. DOM elements do not re-render based on handler identity; React looks the handler up when the event fires. Memoizing them is pure overhead.

**12. Give a fix for an effect that re-runs because a function dependency changes.**
Move the function **inside** the effect so it is not a dependency at all, and depend on the primitives it reads instead.

**13. Why do `useCallback` and `useMemo` usually appear together?**
`React.memo` compares **all** props. One unstable object prop defeats it, so functions need `useCallback` and objects/arrays need `useMemo`.

**14. How many functions does `useCallback` remember?**
One — the most recent. There is no history.

**15. What does the React Compiler change about all this?**
It inserts the memoization automatically at build time, making most manual `useCallback` calls unnecessary where it is enabled.

---

## 11. What to learn next

You now know all seven core hooks: `useState`, `useEffect`, `useRef`, `useContext`, `useReducer`, `useMemo`, `useCallback`.

Look at the `useToggle` example in the cheat sheet again. It is a plain function that calls `useState` and `useCallback`, returns a pair, and can be dropped into any component. That is a **custom hook** — and it is not a new feature. It is just a function whose name starts with `use`, made of the hooks you already know.

Custom hooks are how you stop copy-pasting the same `useEffect` + `useState` block into five components: fetching data, reading window size, syncing to `localStorage`, debouncing an input. You write `useFetch`, `useLocalStorage`, `useDebounce` once, and every component gets its own separate state from them.

➡ Next note: `08_custom_hooks.md`

Related notes:
- [06. useMemo](06_use_memo.md) — the same hook, caching a value instead of a function
- [02. useEffect](02_use_effect.md) — where an unstable function dependency does real damage
- [05. useReducer](05_use_reducer.md) — `dispatch` is stable, so it removes the need for many callbacks
- [01. useState](01_use_state.md) — the updater form that keeps deps arrays empty

⬅ [Back to chapter index](README.md)
