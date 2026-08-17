# 08. Custom Hooks

> A custom hook is just a normal JavaScript function whose name starts with `use` and which calls other hooks — it lets you share **stateful logic** between components without sharing the state itself.

---

## 1. Real-life analogy

Think about a recipe versus a cooked dish.

You write down a recipe for chai once: boil water, add tea, add milk, add sugar, strain. Now ten people in ten different kitchens can follow that same recipe. They each end up with **their own cup**. Nobody shares a cup. If one person adds extra sugar, the other nine cups are unaffected.

The recipe is shared. The chai is not.

That is exactly a custom hook:

| Kitchen | React |
|---|---|
| The written recipe | the custom hook function |
| One person cooking it | one component calling the hook |
| Their own cup of chai | that component's own state |
| Ten people, ten cups | ten components, ten independent states |
| Changing the recipe file | editing the hook — every kitchen gets the fix |

Now contrast that with a **shared pot** of chai sitting in the middle of the room. Everyone drinks from the same pot; when it runs out, it runs out for everybody. That is what React Context does — one shared value. It is a completely different thing, and confusing the two is the single biggest misunderstanding beginners have about custom hooks.

**In simple words:** a custom hook is a shared recipe, not a shared pot — every component that calls it gets its own separate state.

---

## 2. The problem — why does this exist?

### The same block of code, copied into five files

Here are two components from a real app. Look at how much of them is identical.

```jsx
function UserProfile({ userId }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(res.status);
        return res.json();
      })
      .then(setData)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Something went wrong</p>;
  return <h1>{data.name}</h1>;
}
```

```jsx
function ProductPage({ productId }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // ... the exact same 15 lines, with a different URL
  }, [productId]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Something went wrong</p>;
  return <h1>{data.title}</h1>;
}
```

Twenty lines duplicated. Now imagine that repeated across a dozen components, and then someone finds a bug in the abort handling. You fix it in twelve places. You miss two. Those two now behave differently from the rest, and nobody notices for a month.

### Why the usual solutions do not work here

**"Just extract it into a helper function."**

```jsx
function fetchData(url) {
  const [data, setData] = useState(null); // ❌ breaks the rules of hooks
  // ...
}
```

You cannot. `useState` and `useEffect` can only be called from a component or from another hook. A plain helper called `fetchData` is neither, and React will throw:

```
Error: Invalid hook call. Hooks can only be called inside
the body of a function component.
```

**"Put it in a component and share that."**

A component must return JSX. But the thing we want to share is not *markup* — it is *behaviour*. Two components that fetch identically may render completely different screens.

**"Put it in Context."**

Context shares **one value with many components** — the shared pot. Here we want each component to have its **own** data, its own loading flag, its own error. Context is the wrong shape entirely.

### What is left

We need something that:
- can call hooks (so it must be a hook itself),
- returns plain values, not JSX (so it is not a component),
- creates fresh state for each caller (so it is not Context).

React's answer: **just write a function whose name starts with `use`.** That is the entire feature.

**In simple words:** you cannot extract hook logic into a normal function, and a component or Context is the wrong shape — so React lets you write your own hooks.

---

## 3. What it actually is

```jsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);          // a real hook

  const increment = useCallback(() => setCount((c) => c + 1), []);
  const reset = useCallback(() => setCount(initial), [initial]);

  return { count, increment, reset };                    // plain values out
}
```

That is a complete custom hook. There is no `createHook`, no import, no registration. It is a function.

Three properties define it:

1. **Its name starts with `use`.** This is not decoration — it is how React's linter and dev warnings know to enforce the rules of hooks inside it.
2. **It calls at least one hook.** If it does not, it is just a normal function and should not be named `use...`.
3. **It returns whatever is useful** — a value, an array, an object, a function, or nothing at all.

### Using it

```jsx
function Page() {
  const cart = useCounter(0);     // cart has its OWN count
  const wishlist = useCounter(5); // wishlist has its OWN count, starting at 5

  return (
    <>
      <button onClick={cart.increment}>Cart: {cart.count}</button>
      <button onClick={wishlist.increment}>Wishlist: {wishlist.count}</button>
    </>
  );
}
```

Click "Cart". `wishlist.count` does not move. Two calls, two independent pieces of state.

### The state is not shared — this is the key idea

```
             useCounter (the code)
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
  <Cart/>       <Wishlist/>     <Likes/>
  count: 3       count: 5        count: 0     ← three separate state slots
```

Calling a custom hook is not like importing a variable. Every call site allocates its **own** hook slots on that component's fiber. Sharing the function does not share the data any more than two people using the same recipe share a cup.

> ⚠️ If you *want* shared state, a custom hook is not enough. You need Context (note 04) — often wrapped in a custom hook for convenience, which is the `useTheme()` pattern from that note.

### Custom hooks vs components vs helpers

| | Can call hooks? | Returns | Purpose |
|---|---|---|---|
| Component | yes | JSX | Draw something |
| Custom hook | yes | any value | Share **logic** |
| Plain function | no | any value | Pure calculation |

**In simple words:** a custom hook is a function named `use...` that calls hooks, returns plain values, and gives every caller its own private state.

---

## 4. Syntax / setup, step by step

### Step 1 — write the logic inside a component first

Never start by designing a hook. Write the ugly, duplicated version first and make it work. You cannot see the right abstraction until you have two real examples of it.

### Step 2 — spot the repetition

Look for the same `useState` + `useEffect` shape appearing in more than one place. Two occurrences is the signal; one is not.

### Step 3 — create the file

Convention: one hook per file, in a `hooks/` folder, named after the hook.

```
src/
├── hooks/
│   ├── useFetch.js
│   ├── useLocalStorage.js
│   └── useDebounce.js
└── components/
```

The file is `.js`, not `.jsx` — a hook returns values, not JSX, so there is usually no JSX in it.

### Step 4 — move the code and name the inputs

Whatever the components differed on becomes a **parameter**.

```jsx
// hooks/useFetch.js
import { useState, useEffect } from "react";

function useFetch(url) {          // the URL was the only difference -> parameter
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // ... the shared logic, unchanged
  }, [url]);

  return { data, loading, error };
}

export default useFetch;
```

### Step 5 — decide what to return

| Return shape | Use when | Example |
|---|---|---|
| A single value | there is only one thing | `const width = useWindowWidth();` |
| An **array** | the caller will want to rename things | `const [on, toggle] = useToggle();` |
| An **object** | there are three or more values | `const { data, loading, error } = useFetch(url);` |

Arrays are renamed by position, which is why `useState` returns one — `const [name, setName]` and `const [age, setAge]` from the same hook. Objects are safer once you pass three or more values, because the caller cannot get the order wrong.

> 💡 Return two things → array. Return three or more → object. This is the convention across the whole React ecosystem.

### Step 6 — stabilise the functions you return

Any function a hook returns may end up in a caller's dependency array or be passed to a `memo` child. Wrap it in `useCallback` (note 07) so the caller is not punished for using your hook.

```jsx
const increment = useCallback(() => setCount((c) => c + 1), []); // ✅ stable
```

This matters more in a hook than in a component, because you do not control how callers use the value.

### Step 7 — call it at the top level, like any hook

```jsx
function Profile({ id }) {
  const { data, loading } = useFetch(`/api/users/${id}`); // ✅ top level

  if (loading) {
    const x = useFetch("/other");  // ❌ never — conditional hook call
  }
}
```

Every rule of hooks applies **inside** your hook and **at the call site**. Your hook consumes slots on the caller's fiber, in order.

**In simple words:** write it duplicated first, pull the shared part into a `use...` file, turn the differences into parameters, and wrap returned functions in `useCallback`.

---

## 5. Full working example (with comments)

Three hooks you will genuinely reuse, and a component that uses all of them together.

```jsx
// ============================================================
// hooks/useLocalStorage.js
// Works exactly like useState, but survives a page refresh.
// ============================================================
import { useState, useEffect, useCallback } from "react";

function useLocalStorage(key, initialValue) {
  // Lazy initialiser: this function runs ONCE, on the first render.
  // Reading localStorage is slow-ish, so we must not do it every render.
  const [value, setValue] = useState(() => {
    try {
      const saved = window.localStorage.getItem(key);
      return saved !== null ? JSON.parse(saved) : initialValue;
    } catch {
      // Private browsing, quota errors, or corrupt JSON — fall back safely
      return initialValue;
    }
  });

  // Write to storage whenever the value (or the key) changes
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // Storage full or blocked — the app must keep working anyway
    }
  }, [key, value]);

  // Stable setter, so callers can safely put it in a dependency array
  const set = useCallback((next) => setValue(next), []);

  // Array shape: the caller renames it, exactly like useState
  return [value, set];
}

export default useLocalStorage;
```

```jsx
// ============================================================
// hooks/useDebounce.js
// Returns a value that only updates after the input has been
// still for `delay` ms. Perfect for search boxes.
// ============================================================
import { useState, useEffect } from "react";

function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);

    // CLEANUP is the whole trick: every keystroke cancels the
    // previous timer, so only the final pause actually fires.
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced; // single value -> return it bare
}

export default useDebounce;
```

```jsx
// ============================================================
// hooks/useFetch.js
// Data fetching with loading + error states, safe cleanup,
// and no race conditions.
// ============================================================
import { useState, useEffect } from "react";

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Nothing to fetch yet (e.g. an empty search box)
    if (!url) {
      setData(null);
      setLoading(false);
      return;
    }

    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`Request failed: ${res.status}`);
        return res.json();
      })
      .then((json) => setData(json))
      .catch((err) => {
        // An aborted request is not a real error — the user just moved on
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setLoading(false));

    // Cleanup cancels the in-flight request when `url` changes or
    // the component unmounts. Without it, a slow old response can
    // land AFTER a fast new one and overwrite it.
    return () => controller.abort();
  }, [url]);

  // Three values -> object, so the caller cannot mix up the order
  return { data, loading, error };
}

export default useFetch;
```

```jsx
// ============================================================
// SearchPage.jsx — all three hooks, working together
// ============================================================
import useLocalStorage from "./hooks/useLocalStorage";
import useDebounce from "./hooks/useDebounce";
import useFetch from "./hooks/useFetch";

function SearchPage() {
  // Remembers the last search across refreshes
  const [query, setQuery] = useLocalStorage("last-search", "");

  // Waits until the user stops typing for 400ms
  const debouncedQuery = useDebounce(query, 400);

  // Fires only when the DEBOUNCED value changes — not on every keystroke
  const { data, loading, error } = useFetch(
    debouncedQuery ? `/api/search?q=${encodeURIComponent(debouncedQuery)}` : null
  );

  return (
    <div>
      <input
        value={query}
        placeholder="Search products"
        onChange={(e) => setQuery(e.target.value)}
      />

      {loading && <p>Searching...</p>}
      {error && <p>Could not search: {error.message}</p>}
      {data && data.length === 0 && <p>No results for "{debouncedQuery}"</p>}

      <ul>
        {data?.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}

export default SearchPage;
```

### What just happened

The component is **eleven lines of logic** and reads almost like English: remember the query, wait for a pause, fetch the results. All the hard parts — JSON parsing, storage errors, timer cleanup, aborting stale requests, race conditions — are hidden inside three small files.

And notice the composition: `useDebounce` feeds `useFetch`. Hooks calling hooks calling hooks is normal, and it is the reason custom hooks scale.

If the fetch cleanup logic turns out to be buggy, you fix `useFetch.js` once and every screen in the app is fixed.

**In simple words:** three small hooks turn a messy fetch-on-type screen into eleven readable lines, and each hook is fixed in one place.

---

## 6. How it works behind the scenes

### There is no magic — the hooks belong to the component

This is the part worth understanding properly. A custom hook does **not** own state. When React renders `SearchPage`, it runs the function body top to bottom. Calling `useLocalStorage` simply runs *that* function inline, and the `useState` inside it grabs the next slot on **`SearchPage`'s** fiber.

```
Fiber for <SearchPage>
├── hook #1  useState   ← from useLocalStorage  (query)
├── hook #2  useEffect  ← from useLocalStorage  (write to storage)
├── hook #3  useState   ← from useDebounce      (debounced value)
├── hook #4  useEffect  ← from useDebounce      (the timer)
├── hook #5  useState   ← from useFetch         (data)
├── hook #6  useState   ← from useFetch         (loading)
├── hook #7  useState   ← from useFetch         (error)
└── hook #8  useEffect  ← from useFetch         (the request)
```

Eight slots, in one flat list, in call order. React does not know or care that they came from three different files. As far as React is concerned, you wrote all eight calls directly in `SearchPage`.

Three consequences follow, and they explain everything else in this note:

**1. Why the state is never shared.** Two components each have their own fiber, so each gets its own set of eight slots. Same code, different memory.

**2. Why the rules of hooks apply inside a custom hook.** The slots are matched by order. An `if` inside your hook shifts every slot after it — including slots belonging to hooks the caller wrote afterwards.

```jsx
function useThing(flag) {
  if (flag) {
    const [x] = useState(0); // ❌ shifts every later slot when flag changes
  }
}
```

**3. Why calling the same hook twice is fine.**

```jsx
const cart = useCounter(0);      // takes slots #1, #2
const wishlist = useCounter(5);  // takes slots #3, #4
```

Two calls, two separate blocks of slots. Nothing collides.

### The `use` prefix is a real signal, not just style

React and its lint rules cannot see inside a function to know whether it calls hooks. They rely on the name.

- `eslint-plugin-react-hooks` enforces the rules of hooks in any function starting with `use`.
- It also *permits* hook calls inside such a function — rename `useFetch` to `getFetch` and the linter immediately flags every hook inside it as illegal.
- React DevTools groups a component's state by which hook produced it, using the name.

```jsx
function useCounter() { useState(0); }  // ✅ linted as a hook
function getCounter() { useState(0); }  // ❌ "hooks can only be called..."
```

### Re-render flow

```
state inside a custom hook changes  (e.g. setDebounced fires)
      ↓
React marks the COMPONENT that called the hook as needing a re-render
      ↓
that component's function runs again, top to bottom
      ↓
every custom hook it calls runs again too, in the same order
      ↓
each hook reads its slots and returns the current values
      ↓
new JSX -> reconciliation -> DOM
```

The unit of re-rendering is always the **component**, never the hook. A hook cannot re-render "itself"; it re-renders the component that called it, which then re-runs the hook.

### Hooks compose because they are just function calls

```jsx
function useUser(id) {
  const { data, loading, error } = useFetch(`/api/users/${id}`); // hook in a hook
  const isAdmin = data?.role === "admin";
  return { user: data, isAdmin, loading, error };
}
```

There is no depth limit and no special syntax. `useUser` calls `useFetch` calls `useState` — the slots just flatten out onto the calling component's fiber in order.

**In simple words:** a custom hook is inlined into the component that calls it, so the state lives on that component's fiber and every caller gets a separate copy.

---

## 7. Comparison with alternatives (table)

### Ways to share things in React

| Approach | Shares | Each user gets | Use it for |
|---|---|---|---|
| **Custom hook** | logic | its **own** state | fetching, timers, form state, subscriptions |
| **Context** | one value | the **same** state | theme, current user, language |
| **Props** | data downward | what the parent sends | anything a parent already knows |
| **Component** | markup | its own render | anything visual |
| **Plain function** | a calculation | a return value | formatting, maths — no hooks allowed |

The two that get confused are the first two. The test is one question: *should a change in component A be visible in component B?*

```
Yes -> Context (one shared pot)
No  -> custom hook (one recipe, many cups)
```

They combine constantly. The standard pattern from note 04 is a custom hook that **reads** Context:

```jsx
// A custom hook wrapping a shared value — the best of both
function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}
```

Here the state is shared (Context), and the hook only provides a nicer, safer way to read it. That is why `useAuth()` looks like a custom hook but behaves like a shared pot — it is both.

### Custom hooks vs the older patterns

React had two earlier answers to the same problem. You will meet them in old codebases.

| Pattern | Era | Why hooks replaced it |
|---|---|---|
| **HOC** (higher-order component) — `withRouter(Comp)` | classes | Deep wrapper nesting in DevTools, prop-name collisions, unclear where a prop came from |
| **Render props** — `<Fetch>{(data) => ...}</Fetch>` | classes | "Callback pyramid" in JSX, hard to combine three of them |
| **Custom hooks** | now | Flat, composable, no wrappers, values have obvious names |

```jsx
// render props — three sources of data, three levels of nesting
<Fetch url="/a">{(a) =>
  <Fetch url="/b">{(b) =>
    <Mouse>{(pos) => <Chart a={a} b={b} pos={pos} />}</Mouse>}
  </Fetch>}
</Fetch>

// hooks — flat
const a = useFetch("/a");
const b = useFetch("/b");
const pos = useMouse();
```

### When a library beats your own hook

Writing `useFetch` is an excellent way to learn. In a real app, a data-fetching library gives you caching, retries, deduplication of identical requests, background refetching and pagination — hundreds of hours of edge cases.

| Do it yourself | Reach for a library |
|---|---|
| `useToggle`, `useDebounce`, `useLocalStorage`, `useWindowSize` | Server data — **TanStack Query** or SWR |
| App-specific logic (`useCart`, `useCheckout`) | Complex forms — **React Hook Form** |
| Anything under ~30 lines | Global state — Redux Toolkit, Zustand |

> 💡 Also look at `usehooks-ts` and the `react-use` collection before writing a generic utility hook. Someone has already handled the edge cases in `useWindowSize`.

**In simple words:** custom hooks share logic and Context shares state — and for server data, a library will do it better than your own `useFetch`.

---

## 8. Common mistakes beginners make

**1. Expecting two components to share the hook's state**

```jsx
function A() { const [c, inc] = useCounter(); } // c is A's
function B() { const [c, inc] = useCounter(); } // c is B's — a different number
```
This is the biggest one. Same recipe, different cups. If you need one shared number, put it in Context or lift it to a common parent.

**2. Forgetting the `use` prefix**

```jsx
function fetchData(url) { useState(null); }  // ❌ Invalid hook call
function useFetchData(url) { useState(null); } // ✅
```
The prefix is not a style choice. The linter and React's dev warnings depend on it.

**3. Calling a custom hook conditionally**

```jsx
if (isLoggedIn) {
  const { user } = useUser(); // ❌ shifts every hook slot after it
}
const { user } = useUser();   // ✅ then branch on the result
```
Your hook may contain five `useState` calls. Skipping the call skips five slots at once.

**4. Making a hook out of something that uses no hooks**

```jsx
function useFormatPrice(n) { return `₹${n}`; } // ❌ no hooks inside
function formatPrice(n) { return `₹${n}`; }    // ✅ plain function
```
If it calls no hook, it must not be named `use...` — it forces callers to obey rules of hooks for no reason and cannot be called conditionally.

**5. Returning unstable functions**

```jsx
function useCounter() {
  const [c, setC] = useState(0);
  const inc = () => setC(c + 1);  // ❌ new object every render
  return { c, inc };
}
```
Callers putting `inc` in a deps array get an effect loop, and `memo` children re-render. Wrap it: `useCallback(() => setC(c => c + 1), [])`.

**6. Extracting too early**

One usage is not duplication. A hook written from a single example almost always gets the parameters wrong, and you end up with `useThing(a, b, options, mode)` — harder to read than the code it replaced. Wait for the second real case.

**7. A hook that does five unrelated things**

```jsx
function usePageStuff() { /* fetch + theme + scroll + auth + analytics */ }
```
Small and single-purpose composes; large and general does not. Prefer four small hooks the component calls together.

**8. Calling a hook from a plain function or an event handler**

```jsx
function handleClick() {
  const { data } = useFetch("/api"); // ❌ not a component or hook
}
```
Hooks run during render only. Call the hook at the top level and use its value inside the handler.

**9. Missing cleanup**

```jsx
function useWindowWidth() {
  const [w, setW] = useState(window.innerWidth);
  useEffect(() => {
    window.addEventListener("resize", () => setW(window.innerWidth));
    // ❌ no removeEventListener -> a leak per mount, forever
  }, []);
  return w;
}
```
A leak inside a hook multiplies by every component that uses it. Always return the cleanup.

**10. Reading `window` or `localStorage` during render without a guard**

```jsx
const [w] = useState(window.innerWidth); // ❌ crashes in server rendering
```
On the server there is no `window`. Read it in a lazy initialiser with a fallback, or inside `useEffect`.

**11. Naming the returned array badly**

```jsx
return [value, setValue, loading, error]; // ❌ four positional values
return { value, setValue, loading, error }; // ✅ names, in any order
```

**In simple words:** the two big traps are expecting shared state where there is none, and extracting a hook before you have two real uses for it.

---

## 9. Cheat sheet

```jsx
// The minimal shape
function useSomething(input) {
  const [state, setState] = useState(initial);   // real hooks inside

  useEffect(() => {
    /* setup */
    return () => { /* cleanup */ };
  }, [input]);

  const action = useCallback(() => setState((s) => !s), []); // stable

  return { state, action };
}
export default useSomething;
```

```jsx
// Four hooks worth memorising

function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((o) => !o), []);
  return [on, toggle];
}

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current;                       // last render's value
}

function useWindowSize() {
  const [size, setSize] = useState({ w: window.innerWidth, h: window.innerHeight });
  useEffect(() => {
    const onResize = () => setSize({ w: window.innerWidth, h: window.innerHeight });
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize); // always clean up
  }, []);
  return size;
}

function useDebounce(value, delay = 500) {
  const [d, setD] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setD(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return d;
}
```

| Thing | Rule |
|---|---|
| Name | Must start with `use` — `useFetch`, `useCart` |
| Is it special? | No. A plain function that happens to call hooks |
| State sharing | **None.** Every caller gets its own |
| Where to call it | Top level of a component or another hook — never in `if`/loops/handlers |
| Return 1 value | return it bare — `const w = useWidth()` |
| Return 2 values | return an **array** — `const [on, toggle] = useToggle()` |
| Return 3+ values | return an **object** — `const { data, loading, error } = useFetch()` |
| Returned functions | wrap in `useCallback` so callers can depend on them |
| Effects inside | must clean up — the leak multiplies per caller |
| File layout | one hook per file, in `src/hooks/`, `.js` not `.jsx` |
| Write one when | the same hook logic appears in **two** components |
| Do **not** write one when | it calls no hooks, or you only have one use case |

**The decision flow:**

```
Is the same stateful logic in 2+ components?
├─ no  -> leave it where it is
└─ yes
   └─ Do those components need the SAME value, or their own?
      ├─ the same -> Context (optionally read via a custom hook)
      └─ their own -> custom hook
         └─ Is it server data (fetching, caching, retries)?
            ├─ yes -> use TanStack Query instead of hand-rolling
            └─ no  -> write the hook
```

**In simple words:** name it `use`, call hooks inside, return values out, clean up your effects, and only write one after the second copy-paste.

---

## 10. Revision questions (with answers)

**1. What actually makes a function a custom hook?**
Two things: its name starts with `use`, and it calls at least one other hook. There is no special API — it is an ordinary function.

**2. If two components call `useCounter()`, do they share the count?**
No. Each call creates its own state on its own component's fiber. Same recipe, different cups.

**3. Why can't you put `useState` in a helper called `getData()`?**
Hooks may only be called from a component or another hook, and React identifies hooks by the `use` prefix. `getData` is neither, so React throws "Invalid hook call".

**4. Where does a custom hook's state actually live?**
On the fiber of the **component that called it**. The hook is inlined during render and its `useState` calls take the next available slots.

**5. Why do the rules of hooks apply inside a custom hook?**
Because the slots are matched by call order across the whole component. A conditional hook inside your hook shifts every slot after it — including ones the caller wrote.

**6. When should you return an array instead of an object?**
When there are exactly two values and callers will want to rename them, like `const [on, toggle] = useToggle()`. Three or more values → object.

**7. Why wrap functions returned from a hook in `useCallback`?**
Callers may put them in a dependency array or pass them to a `memo` child. An unstable function causes effect loops and wasted re-renders in code you do not control.

**8. What is the difference between a custom hook and Context?**
A custom hook shares **logic** and gives every caller separate state. Context shares **one value** with many components. Different values → hook; same value → Context.

**9. When is it too early to write a custom hook?**
When you have only one use case. The parameters will be wrong, and you will have added indirection without removing duplication. Wait for the second occurrence.

**10. Can a custom hook call another custom hook?**
Yes, to any depth. `useUser` calling `useFetch` calling `useState` is normal — the slots flatten onto the calling component's fiber in order.

**11. Why is `function useFormat(n) { return n.toFixed(2); }` a mistake?**
It calls no hooks, so the `use` prefix is wrong. It should be a plain function, which can then be called anywhere, including conditionally and inside event handlers.

**12. What happens if a hook's `useEffect` forgets its cleanup?**
Every component using the hook leaks a listener, timer or subscription. One bug becomes N bugs — the same multiplication that makes hooks valuable works against you here.

**13. Why does `const [w] = useState(window.innerWidth)` break server rendering?**
There is no `window` object on the server, so the render crashes. Read browser APIs inside `useEffect`, or in a lazy initialiser with a fallback.

**14. Which patterns did custom hooks replace?**
Higher-order components (`withX`) and render props. Both worked but produced deep wrapper nesting; hooks stay flat and compose by simple function calls.

**15. Name one custom hook you should probably not write yourself.**
`useFetch` for real server data — caching, retries, deduplication and background refetching are what TanStack Query or SWR already solve.

---

## 11. What to learn next

Chapter 02 is complete. You now know every core hook and how to build your own:

```
useState    -> remember a value
useEffect   -> talk to the outside world
useRef      -> a box that survives renders without causing them
useContext  -> read a shared value from above
useReducer  -> organise complex state transitions
useMemo     -> cache a value
useCallback -> cache a function
custom      -> package all of the above into reusable logic
```

So far every example has been a single screen. Real apps have many: a home page, a product page, a login screen — each with its own URL, each reachable by the back button and by pasting a link. React itself has no idea what a URL is.

That is what **React Router** adds: routes mapped to components, `<Link>` instead of `<a>`, URL parameters like `/products/42`, nested layouts, and protected routes that redirect a logged-out user to the login page.

You will see `useParams`, `useNavigate` and `useSearchParams` there — and they will feel familiar, because they are just custom hooks someone else wrote.

➡ Next chapter: `03_Routing/01_react_router_setup.md`

Related notes:
- [07. useCallback](07_use_callback.md) — why hooks should return stable functions
- [04. useContext](04_use_context.md) — the shared-state alternative, usually wrapped in a custom hook
- [02. useEffect](02_use_effect.md) — cleanup, which every hook with a subscription needs
- [03. useRef](03_use_ref.md) — the `usePrevious` pattern, a custom hook in three lines

⬅ [Back to chapter index](README.md)
