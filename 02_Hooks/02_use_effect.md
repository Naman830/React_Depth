# 02. useEffect

> `useEffect` runs a piece of code **after** React has updated the screen. It is the door between your component and the outside world — network requests, timers, browser APIs, subscriptions.

---

## 1. Real-life analogy

Think of moving into a new flat.

**Rendering** is arranging the furniture. That is all a component is allowed to do: describe how the room should look, quickly and quietly, without touching anything outside the room.

But moving in also needs jobs that reach *outside* the flat:

- turn on the electricity connection (**set something up**)
- start a newspaper subscription (**subscribe to something**)
- order a water delivery (**fetch something**)

You do these **after** the furniture is in place, not while you are carrying the sofa. And when you move out, you must **undo** them: cancel the newspaper, close the connection. If you forget, the newspapers keep piling up at a flat where nobody lives. That pile-up is a **memory leak**.

`useEffect` is that after-move-in checklist, and its **cleanup function** is the move-out checklist.

**In simple words:** render describes the room; effects deal with everything outside the room, and clean up after themselves.

---

## 2. The problem — why does this exist?

A component function must be **pure**. Pure means: same props and state in, same JSX out, and *nothing else happens*.

Here is why. Look at this broken component:

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  // ❌ Fetching directly in the render body
  fetch(`/api/users/${userId}`)
    .then((r) => r.json())
    .then((data) => setUser(data));

  return <p>{user ? user.name : "Loading..."}</p>;
}
```

What actually happens:

```
render → fetch starts → data arrives → setUser → re-render
                                                    ↓
                                          fetch starts AGAIN → setUser → re-render → ...
```

An infinite loop of network requests. And there are more problems hiding here:

| Problem | Why it happens |
|---------|----------------|
| Infinite loop | Every render starts a new fetch, which sets state, which re-renders |
| Runs too early | React may render a component and then throw the result away. You would have fetched for nothing. |
| Runs twice for one screen | React can render a component more than once before showing anything |
| Nobody cleans up | A timer started during render is never stopped |

React needs a place to say: *"do this **once the screen is actually updated**, and here is how to undo it."*

That place is `useEffect`.

```jsx
useEffect(() => {
  fetch(`/api/users/${userId}`)
    .then((r) => r.json())
    .then(setUser);
}, [userId]); // ✅ only re-runs when userId changes
```

**In simple words:** rendering must stay pure and fast, so anything that touches the outside world moves into an effect.

---

## 3. What it actually is

`useEffect` takes **two arguments**:

```jsx
useEffect(setupFunction, dependencyArray);
```

| Part | What it is | When it runs |
|------|------------|--------------|
| `setupFunction` | your code | after React has painted the screen |
| return value of setup | an optional **cleanup** function | before the next run, and when the component is removed |
| `dependencyArray` | a list of values | React compares it with last time to decide whether to run again |

Full shape:

```jsx
useEffect(() => {
  // 1. SETUP — runs after render
  const id = setInterval(tick, 1000);

  // 2. CLEANUP — runs before the next setup, and on unmount
  return () => clearInterval(id);
}, [tick]); // 3. DEPENDENCIES
```

### The three dependency-array forms

This is the part everybody gets wrong. There are exactly three cases.

```jsx
useEffect(() => { ... });          // NO array → after EVERY render
useEffect(() => { ... }, []);      // EMPTY array → once, on mount only
useEffect(() => { ... }, [a, b]);  // FILLED array → on mount + whenever a or b changes
```

```
                 mount   a changes   unrelated re-render   unmount
no array           ✅        ✅              ✅                —
[]                 ✅        ❌              ❌                cleanup
[a]                ✅        ✅              ❌                cleanup
```

React compares each dependency with the previous render using `Object.is` (basically `===`). If **every** item is the same, the effect is skipped.

> ⚠️ "Dependency" does not mean "whatever I feel like listing". It means **every reactive value the effect reads** — props, state, and anything derived from them. Leaving one out gives you an effect that reads stale values.

**In simple words:** an effect is setup code, optional cleanup code, and a list of values that decide when to re-run it.

---

## 4. Syntax / setup, step by step

### Step 1 — import

```jsx
import { useEffect, useState } from "react";
```

### Step 2 — call it at the top level

```jsx
function Clock() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    // effect body
  }, []);
}
```

Same rule as `useState`: top level only, never inside `if` or a loop.

### Step 3 — set something up

```jsx
useEffect(() => {
  const id = setInterval(() => setTime(new Date()), 1000);
}, []);
```

### Step 4 — always ask "does this need cleaning up?"

If the setup **started** something — a timer, a listener, a subscription, an open connection — the answer is yes.

```jsx
useEffect(() => {
  const id = setInterval(() => setTime(new Date()), 1000);
  return () => clearInterval(id); // ✅ stop the timer
}, []);
```

Without that return, every re-mount adds another interval and the clock ticks faster and faster.

### Step 5 — fill the dependency array honestly

List every prop or state value the effect body uses.

```jsx
useEffect(() => {
  document.title = `${name} has ${count} messages`;
}, [name, count]); // both are read inside → both are listed
```

> 💡 Do not fight the ESLint rule `react-hooks/exhaustive-deps`. If it asks for a dependency you do not want re-running the effect, that is a signal to restructure the effect, not to silence the warning.

### The document-title effect, complete

```jsx
function Title({ name }) {
  useEffect(() => {
    const previous = document.title;   // remember the old value
    document.title = `Hi ${name}`;     // setup
    return () => {
      document.title = previous;       // cleanup: put it back
    };
  }, [name]);

  return <h1>Hi {name}</h1>;
}
```

**In simple words:** import it, call it at the top, set up, return a cleanup, and list what you read.

---

## 5. Full working example (with comments)

A user profile that fetches data whenever the `userId` prop changes. It handles loading, errors, and — importantly — **stale responses**.

```jsx
import { useEffect, useState } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // AbortController lets us cancel a request that is no longer wanted
    const controller = new AbortController();

    // `ignore` is a backup guard: even if the request finishes,
    // we refuse to use its result once this effect is out of date.
    let ignore = false;

    async function load() {
      setLoading(true);
      setError(null);

      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal, // ties the request to this effect run
        });

        if (!res.ok) throw new Error(`Request failed: ${res.status}`);

        const data = await res.json();
        if (!ignore) setUser(data); // only update if still relevant
      } catch (err) {
        // An aborted request is expected, not a real error
        if (err.name !== "AbortError" && !ignore) setError(err.message);
      } finally {
        if (!ignore) setLoading(false);
      }
    }

    load();

    // CLEANUP: runs before the next fetch and when the component unmounts
    return () => {
      ignore = true;
      controller.abort();
    };
  }, [userId]); // re-fetch only when the id actually changes

  if (loading) return <p>Loading...</p>;
  if (error) return <p style={{ color: "red" }}>Error: {error}</p>;
  if (!user) return <p>No user found.</p>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

export default UserProfile;
```

### Why the `ignore` flag matters — the race condition

Without it, this happens:

```
userId changes 1 → 2 quickly

fetch(user 1) ──────────────────────► responds LAST  (slow)
fetch(user 2) ──────► responds FIRST (fast)

setUser(user 2)   ← correct, briefly
setUser(user 1)   ← ❌ old response overwrites the new one
```

The screen ends up showing user 1 while the prop says 2. The cleanup sets `ignore = true` on the *old* effect run, so its late response is thrown away.

> ⚠️ Every fetch inside an effect needs this guard. It is not optional in real apps.

**In simple words:** fetch in an effect, key it to the id, and always cancel or ignore the previous request.

---

## 6. How it works behind the scenes

### The order of events

```
1. State or props change
2. React calls your component function        (render — pure, no effects yet)
3. React works out the DOM changes            (reconciliation)
4. React applies them to the real DOM         (commit)
5. The browser paints the screen 🖼
6. ───► NOW React runs your effects
```

Effects are **asynchronous** and run *after* paint. That is deliberate: it keeps the screen from being blocked by your network call.

### Re-running: cleanup always comes first

When dependencies change, React does **not** just run setup again. It does this:

```
deps changed?
     │
     ├── no  → do nothing
     │
     └── yes → run OLD cleanup   →   run NEW setup
```

So for a `userId` going `1 → 2 → 3`:

```
mount    : setup(1)
1 → 2    : cleanup(1) → setup(2)
2 → 3    : cleanup(2) → setup(3)
unmount  : cleanup(3)
```

Setup and cleanup always come in matched pairs. That symmetry is the whole design.

### Why your effect runs twice in development

In **StrictMode** (on by default in a new Vite/Next app), React deliberately runs every effect **setup → cleanup → setup** on mount, in development only.

```
mount → setup → cleanup → setup     (dev, StrictMode)
mount → setup                       (production)
```

This is a **test**, not a bug. It proves your effect can survive being run twice. If two API calls appear in the network tab, or your counter jumps by 2, your cleanup is missing or wrong.

> 💡 Do not disable StrictMode to hide the double run. Fix the cleanup — the double run is finding a real bug for you.

### Dependency comparison is by reference

```jsx
useEffect(() => { ... }, [{ id: 1 }]);      // ❌ new object every render → runs every time
useEffect(() => { ... }, [user]);            // ❌ if `user` is rebuilt each render
useEffect(() => { ... }, [user.id]);         // ✅ a primitive — compares by value
```

Objects, arrays, and functions created during render are **new every time**, so `Object.is` says "changed". Depend on primitives (strings, numbers, booleans) where you can.

**In simple words:** render → paint → effect; and on every change, cleanup runs before the next setup.

---

## 7. Comparison with alternatives

### Where should this code go?

| I want to... | Use | Why |
|---|---|---|
| Respond to a user click | an **event handler** | it happens because of an action, not because of a render |
| Transform data for display | plain code in the render body | it is derived, not a side effect |
| Fetch data on screen load | `useEffect` (or a library) | it is an outside-world call tied to rendering |
| Start a timer / listener / socket | `useEffect` + cleanup | it must be undone |
| Read DOM size before paint | `useLayoutEffect` | runs before paint, avoids visible flicker |
| Sync a value without re-rendering | `useRef` | no render involvement at all |

### The most common mistake: an effect that should be an event handler

```jsx
// ❌ Fires whenever `cart` changes for ANY reason, including a page reload
useEffect(() => {
  if (cart.length > 0) sendAnalytics("item_added");
}, [cart]);

// ✅ Fires because the user clicked Add
function handleAdd(item) {
  setCart([...cart, item]);
  sendAnalytics("item_added");
}
```

Ask: *did this happen because the user did something, or because the component appeared on screen?* Only the second one is an effect.

### `useEffect` vs `useLayoutEffect`

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Runs | after the browser paints | before the browser paints |
| Blocks paint | no | yes |
| Use for | fetching, subscriptions, logging, timers | measuring DOM, preventing visual flicker |
| Default choice | ✅ this one | only when you see flicker |

**In simple words:** if it is not caused by rendering, it does not belong in an effect.

---

## 8. Common mistakes beginners make

### 1. Forgetting the dependency array

```jsx
useEffect(() => {
  setCount(count + 1); // ❌ set state → re-render → effect → set state → forever
});
```
No array means "after every render". Combined with `setState`, that is an infinite loop.

### 2. Lying about dependencies

```jsx
useEffect(() => {
  console.log(count); // reads count
}, []);               // ❌ but does not list it → always logs the first value
```
The effect closed over the first render's `count`. It will never see an update.

### 3. Missing cleanup on timers and listeners

```jsx
useEffect(() => {
  window.addEventListener("resize", onResize); // ❌ never removed
}, []);

useEffect(() => {
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize); // ✅
}, []);
```
Listeners stack up on every mount and keep the old component alive in memory.

### 4. An object or array in the dependency list

```jsx
const options = { limit: 10 };            // new object every render
useEffect(() => { ... }, [options]);      // ❌ runs every render

useEffect(() => { ... }, [limit]);        // ✅ depend on the primitive
```

### 5. Making the effect callback `async`

```jsx
useEffect(async () => { ... }, []);   // ❌ returns a Promise, React expects a cleanup fn

useEffect(() => {                      // ✅ async function INSIDE
  async function load() { ... }
  load();
}, []);
```

### 6. Using an effect to copy props into state

```jsx
useEffect(() => { setName(props.name); }, [props.name]); // ❌ extra render, gets out of sync
const name = props.name;                                  // ✅ just use it
```

### 7. Using an effect for derived data

```jsx
useEffect(() => { setTotal(price * qty); }, [price, qty]); // ❌
const total = price * qty;                                  // ✅ calculate during render
```

### 8. Disabling the lint rule

```jsx
}, []); // eslint-disable-line react-hooks/exhaustive-deps   ❌ hiding a real bug
```
The warning means the effect reads something it does not react to. Restructure instead.

### 9. Panicking about the double run in dev

That is StrictMode checking your cleanup. Fix the cleanup, do not remove StrictMode.

### 10. No race-condition guard on fetches

Covered in section 5. Fast-changing props + slow network = the wrong data on screen.

**In simple words:** always give the array, tell the truth in it, and clean up whatever you started.

---

## 9. Cheat sheet

```jsx
import { useEffect } from "react";

// after every render (rare)
useEffect(() => { ... });

// once, on mount
useEffect(() => { ... }, []);

// on mount + when a or b changes
useEffect(() => { ... }, [a, b]);

// with cleanup
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);

// event listener
useEffect(() => {
  const onKey = (e) => console.log(e.key);
  window.addEventListener("keydown", onKey);
  return () => window.removeEventListener("keydown", onKey);
}, []);

// fetch with cancel
useEffect(() => {
  let ignore = false;
  const c = new AbortController();
  fetch(url, { signal: c.signal })
    .then((r) => r.json())
    .then((d) => { if (!ignore) setData(d); })
    .catch((e) => { if (e.name !== "AbortError") setError(e); });
  return () => { ignore = true; c.abort(); };
}, [url]);

// async work
useEffect(() => {
  async function run() { await doThing(); }
  run();
}, []);
```

| Question | Answer |
|---|---|
| When does it run? | After React updates the DOM and the browser paints. |
| No array? | After every single render. |
| Empty array? | Once, on mount. Cleanup on unmount. |
| When does cleanup run? | Before the next setup, and on unmount. |
| Why does it run twice in dev? | StrictMode is testing your cleanup. |
| Can the callback be `async`? | No. Put an async function inside it. |
| What goes in the array? | Every prop/state value the effect reads. |

---

## 10. Revision questions (with answers)

**Q1. Why can't we fetch directly in the component body?**
Rendering must be pure. A fetch there runs on every render, sets state, and re-renders — an infinite loop — and it may run for a render React throws away.

**Q2. What are the three dependency-array forms?**
No array (every render), `[]` (mount only), `[a, b]` (mount plus whenever `a` or `b` changes).

**Q3. When exactly does the cleanup function run?**
Twice over: before every re-run of the effect, and once when the component unmounts.

**Q4. What is wrong with this?**
```jsx
useEffect(() => { setCount(count + 1); });
```
No dependency array, so it runs after every render, and it sets state — an endless render loop.

**Q5. Why does an effect run twice when the app starts in development?**
React StrictMode intentionally mounts, unmounts, and remounts the component to verify your cleanup works. It does not happen in production.

**Q6. Why is `useEffect(async () => {...}, [])` wrong?**
An `async` function returns a Promise, but React expects the return value to be a cleanup function. Define an async function inside and call it.

**Q7. What is a race condition here, and how do you stop it?**
A slow earlier request finishes after a faster later one and overwrites the newer data. Stop it with an `ignore` flag flipped in the cleanup, plus `AbortController`.

**Q8. Should `document.title = ...` after a button click go in an effect?**
No. It was caused by a user action, so it belongs in the click handler. Effects are for things caused by rendering.

**Q9. Why does `useEffect(() => {...}, [{ id: 1 }])` run on every render?**
The object literal is a new reference each render, so `Object.is` reports a change. Depend on `id` instead.

**Q10. Does the effect run before or after the user sees the new screen?**
After. React commits the DOM, the browser paints, then effects run.

---

## 11. What to learn next

- **`useRef`** — for values that must survive renders without causing one; also the standard way to hold a timer id or a DOM node used inside effects.
- **Custom hooks** — the fetch effect above is long. Wrapping it in `useFetch(url)` makes it reusable in one line.
- **TanStack Query** (Chapter 05) — a library that handles fetching, caching, retries, and race conditions so you stop writing this effect by hand.
- **`useLayoutEffect`** — the before-paint version, for measuring the DOM.

➡ Next note: `03_use_ref.md`

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
