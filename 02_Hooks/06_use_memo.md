# 06. useMemo

> `useMemo` remembers the **result** of a calculation and reuses it on the next render, as long as the inputs did not change.

---

## 1. Real-life analogy

Imagine you run a shop and someone asks, "what is the total value of everything on the shelves?"

You count. It takes twenty minutes. You write the answer on a slip of paper and stick it to the till.

Five minutes later someone asks again. You do **not** count again — you read the slip. Nothing arrived, nothing sold, so the answer cannot have changed.

Then a delivery van unloads forty boxes. Now the slip is stale. You tear it up, count once more, and write a fresh slip.

That is exactly `useMemo`:

| Shop | React |
|---|---|
| The twenty-minute count | the expensive calculation |
| The slip stuck to the till | the memoized value |
| "Did any stock move?" | the dependency array |
| Tearing up the slip after a delivery | recalculating when a dependency changes |

Two lessons hide in the analogy, and both matter later:

- Keeping a slip is only worth it if counting is genuinely slow. Nobody writes a slip to remember `2 + 2`.
- The slip is only trustworthy if you correctly listed everything that could change the answer. Miss the delivery van, and you confidently read out a wrong number.

**In simple words:** `useMemo` is a slip of paper holding an answer you do not want to work out again.

---

## 2. The problem — why does this exist?

### Every render re-runs the whole component body

This is the fact that makes `useMemo` necessary. A component function runs **from the top, completely, on every single render** — every state change, every prop change, every parent re-render.

```jsx
function ProductList({ products, query }) {
  // This runs again on EVERY render, even when only `dark` changed
  const filtered = products
    .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
    .sort((a, b) => a.price - b.price);

  const [dark, setDark] = useState(false);

  return (
    <>
      <button onClick={() => setDark(!dark)}>Toggle theme</button>
      <ul>{filtered.map((p) => <li key={p.id}>{p.name}</li>)}</ul>
    </>
  );
}
```

Click the theme button. `dark` changes, the component re-renders, and `products` gets filtered and sorted all over again — even though neither `products` nor `query` moved. With 20 products nobody notices. With 20,000, the button feels stuck.

### Problem A — a slow calculation repeated for no reason

```jsx
// Pretend this takes 200ms on a big list
const stats = calculateExpensiveStats(transactions);
```

If `transactions` did not change, those 200ms are pure waste, repeated on every unrelated state update. The user feels it as lag while typing or clicking.

### Problem B — a new object or array breaks other optimisations

This one is subtler and, in real apps, more common than problem A.

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // A brand-new array object on every render — same contents, different reference
  const options = { sortBy: "price", limit: 10 };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveChild options={options} />
    </>
  );
}

const ExpensiveChild = memo(function ExpensiveChild({ options }) {
  // memo compares props with Object.is:
  // {sortBy:"price"} !== {sortBy:"price"}  -> different reference -> re-render anyway
  return <div>{/* heavy rendering */}</div>;
});
```

`React.memo` is supposed to skip the child when its props are unchanged. But `options` is a **new object literal** every render, so `Object.is` says "changed", and `memo` does nothing at all. The optimisation is silently dead.

The same trap hits:
- `useEffect` dependency arrays — a new object dependency makes the effect run every render.
- Context `value` — the trap from note 04, where every consumer re-renders.
- Any custom hook that compares an object dependency.

### What we actually want

A way to say: *"keep the previous result unless these specific inputs changed."* That is `useMemo`.

> ⚠️ `useMemo` is a **performance** tool, not a correctness tool. Your app must work correctly with every `useMemo` deleted. If removing one breaks the app, the real bug is elsewhere.

**In simple words:** components re-run everything on every render, and `useMemo` lets you skip the parts whose inputs did not move.

---

## 3. What it actually is

```jsx
const memoizedValue = useMemo(() => computeSomething(a, b), [a, b]);
```

Three parts:

1. **A function** that calculates and **returns** a value. React calls it for you.
2. **A dependency array** — the list of values the calculation reads.
3. **The return** — the calculated value, or the one kept from last time.

React's logic is short:

```
On every render:
  compare each dependency with its value from the last render (Object.is)
  ├─ all the same  -> skip the function, return the stored result
  └─ any different -> run the function, store and return the new result
```

### The word "memoize"

To **memoize** means to cache a function's result against its inputs, so the same inputs never do the work twice. It is not "memorize" — the word comes from *memo*, a note to yourself. That is why the hook is `useMemo`.

### It caches exactly one result

`useMemo` remembers only the **most recent** result. It is not a lookup table of every past value.

```jsx
const value = useMemo(() => slow(n), [n]);
// n: 1 -> calculates    (stored: n=1)
// n: 2 -> calculates    (stored: n=2, the n=1 result is gone)
// n: 1 -> calculates AGAIN — nothing was kept for n=1
```

### Return the value, do not call it

A very common beginner slip:

```jsx
useMemo(() => computeStats(data), [data]);   // ✅ passes a function; React calls it
useMemo(computeStats(data), [data]);         // ❌ calls it NOW, every render
```

The second line runs `computeStats` immediately and hands `useMemo` the *result* instead of a function — so the expensive work happens on every render anyway, and `useMemo` does nothing useful.

### React may throw the cache away

The docs are explicit: React can discard a memoized value whenever it likes — for example to free memory, or when a component unmounts and remounts. `useMemo` is a **hint**, not a promise. This is another way of saying: never depend on it for correctness.

**In simple words:** `useMemo(fn, deps)` runs `fn` only when `deps` change, and hands you last time's answer otherwise.

---

## 4. Syntax / setup, step by step

### Step 1 — write the calculation normally first

Always start without `useMemo`. Make it correct, then make it fast.

```jsx
const visible = products
  .filter((p) => p.name.includes(query))
  .sort((a, b) => a.price - b.price);
```

### Step 2 — measure before optimising

Do not guess. Wrap it and look:

```jsx
console.time("filter");
const visible = products.filter(/* ... */).sort(/* ... */);
console.timeEnd("filter"); // filter: 0.04ms  <- do NOT memoize this
```

A calculation under about 1ms is not worth memoizing. React DevTools' Profiler tab is the better tool: record an interaction and look at what actually takes time.

### Step 3 — wrap it

```jsx
import { useMemo } from "react";

const visible = useMemo(() => {
  return products
    .filter((p) => p.name.includes(query))
    .sort((a, b) => a.price - b.price);
}, [products, query]); // every value the function reads from outside
```

### Step 4 — get the dependency array right

The rule: **every reactive value the function reads must be in the array.** Reactive means props, state, context, or anything derived from them.

```jsx
const total = useMemo(() => {
  return items.reduce((sum, i) => sum + i.price * taxRate, 0);
}, [items, taxRate]); // reads items AND taxRate -> both listed
```

Things you do **not** list: values that never change identity — `dispatch` from `useReducer`, setters from `useState`, refs, module-level constants, and imported functions.

> 💡 Install `eslint-plugin-react-hooks`. Its `exhaustive-deps` rule reads your function and tells you exactly which dependencies are missing. Trust it over your own memory.

### Step 5 — the second use: stabilising an object

The same hook, used for identity rather than speed:

```jsx
// Without useMemo this object is new every render, so memo/effects see a "change"
const config = useMemo(() => ({ sortBy, limit: 10 }), [sortBy]);

return <ExpensiveChild config={config} />;
```

Here the calculation is trivial. The point is that the **reference** stays the same while `sortBy` does not change.

**In simple words:** write it plain, measure it, then wrap it and list every outside value the function reads.

---

## 5. Full working example (with comments)

A product search over a large list. It shows both uses of `useMemo` — skipping slow work, and keeping an object stable — plus a control that proves the calculation is being skipped.

```jsx
import { useMemo, useState, memo } from "react";

// ============================================================
// Fake data — 5,000 products, built once at module level
// ============================================================
const PRODUCTS = Array.from({ length: 5000 }, (_, i) => ({
  id: i,
  name: `Product ${i}`,
  price: (i % 100) + 1,
  category: ["books", "tools", "toys"][i % 3],
}));

// ============================================================
// A memoized child. It only re-renders when its props change
// by reference — which is why `config` below must be stable.
// ============================================================
const ResultSummary = memo(function ResultSummary({ config, count }) {
  console.log("ResultSummary rendered"); // watch this in the console
  return (
    <p>
      Showing {count} products, sorted by {config.sortBy}.
    </p>
  );
});

function ProductSearch() {
  const [query, setQuery] = useState("");
  const [category, setCategory] = useState("all");
  const [sortBy, setSortBy] = useState("price");
  const [dark, setDark] = useState(false); // deliberately unrelated to the list

  // ----------------------------------------------------------
  // USE 1 — skip an expensive calculation
  // Runs only when query, category or sortBy change.
  // Clicking "toggle theme" re-renders the component but does
  // NOT re-run this filter+sort over 5,000 items.
  // ----------------------------------------------------------
  const visible = useMemo(() => {
    console.log("filtering + sorting..."); // proves when it actually runs
    const q = query.toLowerCase();

    return PRODUCTS
      .filter((p) => p.name.toLowerCase().includes(q))
      .filter((p) => category === "all" || p.category === category)
      .sort((a, b) =>
        sortBy === "price" ? a.price - b.price : a.name.localeCompare(b.name)
      );
  }, [query, category, sortBy]); // PRODUCTS is module level -> never changes

  // ----------------------------------------------------------
  // A cheap derived value — NO useMemo needed.
  // Summing an already-filtered array is fast; wrapping it would
  // cost more than it saves.
  // ----------------------------------------------------------
  const totalPrice = visible.reduce((sum, p) => sum + p.price, 0);

  // ----------------------------------------------------------
  // USE 2 — keep an object reference stable
  // Without this, `config` would be a new object every render and
  // memo() on ResultSummary would never skip anything.
  // ----------------------------------------------------------
  const config = useMemo(() => ({ sortBy, showPrices: true }), [sortBy]);

  return (
    <div style={{ background: dark ? "#222" : "#fff", color: dark ? "#eee" : "#111" }}>
      <button onClick={() => setDark((d) => !d)}>
        Toggle theme (does not re-filter)
      </button>

      <input
        value={query}
        placeholder="Search products"
        onChange={(e) => setQuery(e.target.value)}
      />

      <select value={category} onChange={(e) => setCategory(e.target.value)}>
        <option value="all">All</option>
        <option value="books">Books</option>
        <option value="tools">Tools</option>
        <option value="toys">Toys</option>
      </select>

      <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
        <option value="price">Price</option>
        <option value="name">Name</option>
      </select>

      <ResultSummary config={config} count={visible.length} />
      <p>Total: ₹{totalPrice}</p>

      <ul>
        {visible.slice(0, 20).map((p) => (
          <li key={p.id}>
            {p.name} — ₹{p.price}
          </li>
        ))}
      </ul>
    </div>
  );
}

export default ProductSearch;
```

**Try it and watch the console.** Click "Toggle theme" repeatedly:

- `"filtering + sorting..."` does **not** appear — `useMemo` returned the stored array.
- `"ResultSummary rendered"` does **not** appear — `config` kept the same reference, so `memo` skipped the child.

Now type in the search box: both messages appear, because `query` changed.

Delete the two `useMemo` wrappers and repeat. Both messages now fire on every theme toggle. That difference is the entire hook.

**In simple words:** one `useMemo` saved the slow work, the other saved the child from re-rendering, and the console proves both.

---

## 6. How it works behind the scenes

### The slot, again

Like every hook, `useMemo` gets a **slot** on the component's fiber — the internal object React keeps for each component instance. That slot stores two things: the last dependency array, and the last returned value.

```
Fiber for <ProductSearch>
├── hook #1  useState  -> "shoes"
├── hook #2  useState  -> "all"
├── hook #3  useMemo   -> { deps: ["shoes","all","price"], value: [...] }
└── hook #4  useMemo   -> { deps: ["price"],               value: {...} }
```

This is why the rules of hooks matter here too: slots are matched by **call order**, not by name.

### The comparison

On each render React walks the arrays element by element with `Object.is` — the same shallow reference check used by `useState` and `React.memo`.

```
old deps: ["shoes", "all", "price"]
new deps: ["shoes", "all", "name"]
           same     same    DIFFERENT
                              ↓
                    run the function again
```

**Shallow** is the critical word. React compares the *items* in the array, not what is inside them:

```jsx
const user = { name: "Naman" };          // new object every render
useMemo(() => work(user), [user]);       // ❌ dependency differs every time
                                         //    the memo never hits
useMemo(() => work(user.name), [user.name]); // ✅ a string — compares by value
```

> 💡 Prefer primitives in dependency arrays. A string, number or boolean compares by value and behaves the way you expect.

### The render timeline

```
state changes
      ↓
component function runs top to bottom
      ↓
reaches useMemo -> compares deps
      ├─ unchanged: returns stored value instantly    (function skipped)
      └─ changed:   runs the function NOW, during render
      ↓
JSX returned
      ↓
React commits to the DOM
```

Notice: the memoized function runs **during render**, not after it. That has one hard consequence — the function must be **pure**, exactly like a reducer. No `fetch`, no timers, no DOM writes, no `setState`. Side effects belong in `useEffect`.

### `useMemo` is not free

The hook itself costs something on every render:

- storing the function you passed,
- storing the dependency array,
- comparing the array element by element,
- extra memory held for the cached value, for as long as the component lives.

For a cheap calculation, that overhead is **larger** than just doing the work. This is the whole reason "memoize everything" is bad advice rather than merely unnecessary.

### The React Compiler changes the calculus

React 19 ships an optional **compiler** that inserts memoization automatically, at build time, by analysing your code. Where it is enabled, most manual `useMemo` and `useCallback` calls become unnecessary.

It is not on by default in every setup, and understanding `useMemo` by hand is still how you understand *why* re-renders happen. Learn the hook; be glad when the compiler does it for you.

**In simple words:** React stores the deps and the value in a hook slot, shallow-compares them each render, and re-runs your pure function only when something moved.

---

## 7. Comparison with alternatives (table)

### The three memoization tools

| Tool | What it remembers | Use it when |
|---|---|---|
| `useMemo(fn, deps)` | the **value** `fn` returns | a calculation is slow, or an object/array must keep its reference |
| `useCallback(fn, deps)` | the **function itself** | you pass a function to a `memo` child, or use it in a dependency array |
| `React.memo(Component)` | the component's last **render** | a child re-renders often with the same props |

They are designed to work together: `React.memo` on the child, and `useMemo`/`useCallback` on the parent to keep the props stable. `memo` alone on a child receiving an object prop achieves nothing.

```jsx
useCallback(fn, deps)  ===  useMemo(() => fn, deps)   // literally the same thing
```

`useCallback` is a shorthand for memoizing a function value. That is the only difference.

### `useMemo` vs alternatives to memoizing at all

| Approach | Cost | When it is the better answer |
|---|---|---|
| Just calculate it | zero | The calculation is fast — this is the default |
| `useMemo` | small overhead each render | Genuinely slow work, or reference stability is needed |
| Move state down | zero | The slow part does not depend on the state that keeps changing |
| Pass JSX as `children` | zero | The parent re-renders but the child's content is unrelated |
| Store the result in state | risk of drift | Almost never — derived data in state goes stale |
| The React Compiler | zero at runtime | Available in your setup — it does this automatically |

> 💡 The cheapest optimisation is not memoizing — it is **restructuring**. If a fast-changing piece of state lives in a smaller component, the expensive parent never re-renders and needs no `useMemo` at all.

**In simple words:** `useMemo` caches values, `useCallback` caches functions, `React.memo` caches renders — and moving state down often beats all three.

---

## 8. Common mistakes beginners make

**1. Memoizing everything**

```jsx
const doubled = useMemo(() => count * 2, [count]); // ❌ multiplication is free
```
The comparison costs more than the multiplication. Wrap slow work only.

**2. Calling the function instead of passing it**

```jsx
useMemo(computeStats(data), [data]);        // ❌ runs every render
useMemo(() => computeStats(data), [data]);  // ✅
```

**3. Forgetting a dependency**

```jsx
const total = useMemo(() => items.reduce((s, i) => s + i.price * taxRate, 0), [items]);
// ❌ taxRate missing -> the total silently freezes at the old rate
```
This is the one bug class where `useMemo` causes **wrong output**, not just slowness. Let the ESLint rule fill the array.

**4. An empty array to "run it once"**

```jsx
const filtered = useMemo(() => products.filter(f), []); // ❌ never updates
```
`useMemo` is not `useEffect`. An empty array means "these inputs never change" — if that is untrue, you get stale data on screen.

**5. Putting a fresh object in the dependency array**

```jsx
useMemo(() => work(opts), [{ id }]);   // ❌ a new object every render — never equal
useMemo(() => work(opts), [id]);       // ✅ a primitive
```

**6. Side effects inside the memo function**

```jsx
useMemo(() => {
  fetch("/api/log");           // ❌ runs during render, twice in StrictMode
  return compute(data);
}, [data]);
```
Memo functions must be pure. Effects go in `useEffect`.

**7. Expecting it to survive forever**

React may discard the cache. Never write logic that assumes the memo held.

**8. `useMemo` on the child instead of the parent**

The new object is created in the **parent**. Memoizing inside the child cannot help — by then the prop already changed. Stabilise it where it is created.

**9. `React.memo` on a child with object props and no `useMemo`**

```jsx
<MemoChild options={{ a: 1 }} />  // ❌ memo can never skip — new object each time
```
The child looks optimised and is not. This pairing is the most common wasted `memo` in real code.

**10. Optimising without measuring**

Most `useMemo` calls in real codebases guard calculations that take microseconds. Profile first; you will usually find the cost is somewhere else entirely.

**In simple words:** the two real dangers are a missing dependency (wrong data) and memoizing cheap work (slower, not faster).

---

## 9. Cheat sheet

```jsx
// Skip an expensive calculation
const visible = useMemo(() => {
  return products.filter((p) => p.name.includes(query));
}, [products, query]);

// Keep an object/array reference stable for memo children or effects
const config = useMemo(() => ({ sortBy, limit }), [sortBy, limit]);

// Keep a context value stable (from note 04)
const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);
```

| Thing | Rule |
|---|---|
| Signature | `useMemo(() => value, [deps])` |
| Returns | The calculated value, or last render's value |
| First argument | A **function**, not a call. `() => f(x)`, never `f(x)` |
| Runs | During render — must be pure |
| Comparison | Shallow, `Object.is`, item by item |
| Cache size | One. Only the latest result is kept |
| Guaranteed? | No. React may discard it any time |
| Deps to include | Every prop, state or context value the function reads |
| Deps to skip | `dispatch`, `setState` setters, refs, module constants, imports |
| Use it for | Slow calculations, or reference stability |
| Do not use it for | Cheap maths, side effects, or "run once" logic |
| Sibling hooks | `useCallback` for functions, `React.memo` for components |

**The decision flow:**

```
Is the calculation slow (measured, not guessed)?
├─ yes -> useMemo
└─ no
   └─ Does the result get passed to a memo child,
      an effect's deps, or a context value?
      ├─ yes -> useMemo (for the stable reference)
      └─ no  -> just calculate it normally
```

**In simple words:** wrap slow work or shared references, list every input, and leave cheap maths alone.

---

## 10. Revision questions (with answers)

**1. What does `useMemo` return?**
The value returned by your function — either freshly calculated, or the one stored from the previous render if no dependency changed.

**2. When does the function inside `useMemo` actually run?**
During render, and only when at least one dependency has changed since the last render.

**3. What is wrong with `useMemo(compute(data), [data])`?**
It calls `compute` immediately, on every render, and passes the result instead of a function. It must be `useMemo(() => compute(data), [data])`.

**4. How does React decide whether the dependencies changed?**
A shallow, item-by-item comparison with `Object.is` — the same reference check used by `useState` and `React.memo`.

**5. Why does `useMemo(() => work(user), [user])` often never hit the cache?**
If `user` is an object created during render, it is a new reference every time, so the dependency always looks different. Depend on a primitive like `user.id` instead.

**6. How many results does `useMemo` remember?**
One — the most recent. Going back to an earlier input recalculates.

**7. Can you rely on a memoized value still being there?**
No. React may discard it, so `useMemo` must never be needed for correctness — only for speed.

**8. Name the two distinct reasons to use `useMemo`.**
To skip a genuinely slow calculation, and to keep an object or array reference stable for a `memo` child, an effect's dependency array, or a context value.

**9. Why is `<MemoChild options={{ a: 1 }} />` a bug in an optimised component?**
The object literal is new on every render, so `memo`'s prop comparison always fails and the child re-renders anyway. The `memo` is doing nothing.

**10. What happens if you leave out a dependency?**
The memoized value goes stale — the component keeps showing a result computed from old inputs. This is a correctness bug, not just a performance one.

**11. Why can't you `fetch` inside a `useMemo` function?**
It runs during render, where side effects are forbidden. It would also run twice in StrictMode. Use `useEffect`.

**12. What is the relationship between `useMemo` and `useCallback`?**
`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`. One memoizes a returned value; the other memoizes the function itself.

**13. Why is "memoize everything" bad advice?**
Every `useMemo` costs storage and a comparison on every render. For cheap work, that overhead is larger than the work being skipped.

**14. Give an optimisation that beats `useMemo` when it applies.**
Move the fast-changing state down into a smaller component, so the expensive parent stops re-rendering at all — zero runtime cost, no hook needed.

---

## 11. What to learn next

You can now cache a **value**. The obvious next question: what about a **function**?

Functions are objects too. `function handleClick() {}` creates a brand-new object on every render, which breaks `React.memo` on children and re-triggers effects — the exact identity trap from section 2, applied to callbacks.

`useCallback` is the tool for that, and you already know how it works: it is `useMemo` for functions.

➡ Next note: `07_use_callback.md`

Related notes:
- [05. useReducer](05_use_reducer.md) — `dispatch` is stable, so it never needs memoizing
- [04. useContext](04_use_context.md) — the classic `useMemo` use: a stable context value
- [02. useEffect](02_use_effect.md) — same dependency-array rules, different job

⬅ [Back to chapter index](README.md)
