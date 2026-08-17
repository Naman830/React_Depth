# 05. useReducer

> `useReducer` keeps state in one place and changes it through **named actions**, so complicated state logic lives in one function instead of being scattered across many `useState` calls.

---

## 1. Real-life analogy

Think of a bank account.

You cannot walk into the vault and edit your balance. You fill in a **slip**: *deposit ₹500*, *withdraw ₹200*, *transfer ₹1000 to Ravi*. You hand the slip to the counter. A clerk follows the bank's fixed rulebook, applies the slip, and the new balance comes out.

Three parts, and they map exactly onto `useReducer`:

| Bank | React |
|---|---|
| The balance in the ledger | `state` |
| The slip you fill in ("withdraw ₹200") | the **action** |
| The clerk plus the rulebook | the **reducer** function |
| Handing the slip to the counter | `dispatch(action)` |

Notice what the analogy gives you for free. Every change to the balance goes through one counter, so there is exactly one place to look when a number is wrong. Every slip is a record of *what you asked for*, not *what the new balance should be*. And the clerk never invents rules — the same slip on the same balance always produces the same result.

`useState` is the opposite style: you walk up and write the new balance yourself, from anywhere in the building.

**In simple words:** with `useState` you write the new value; with `useReducer` you send a slip and one rulebook decides the new value.

---

## 2. The problem — why does this exist?

### Too many pieces of state that move together

Build a form with a network request. With `useState`:

```jsx
function SignupForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [success, setSuccess] = useState(false);
  // ...five setters, and they must always agree with each other
}
```

Now look at what one submit has to do:

```jsx
async function handleSubmit(e) {
  e.preventDefault();
  setLoading(true);
  setError(null);      // must remember to clear the old error
  setSuccess(false);   // must remember to clear the old success
  try {
    await api.signup({ name, email });
    setSuccess(true);
    setLoading(false);
  } catch (err) {
    setError(err.message);
    setLoading(false);  // easy to forget this one branch
  }
}
```

Every handler must remember to update **all** the related pieces. Forget one line and you get an impossible screen: a spinner *and* an error at the same time, or a success message that never clears.

The state pieces are not really independent. They are one thing — "the status of this form" — chopped into five variables that nothing forces to stay consistent.

### The same update logic repeated in many places

A shopping cart can be changed by a product card, the cart drawer, the checkout page, and a "clear cart" button. With `useState`, the *how* of each update gets copy-pasted into each of those components:

```jsx
// in ProductCard
setItems(items.map((i) => (i.id === id ? { ...i, qty: i.qty + 1 } : i)));

// in CartDrawer — the same logic written a second time
setItems(items.map((i) => (i.id === id ? { ...i, qty: i.qty + 1 } : i)));
```

Two copies means two chances to get it wrong, and a fix has to be applied twice.

### Next state that depends on the current state in a complicated way

"Add this item — but if it is already in the cart, increase its quantity instead, and cap it at the stock limit." That is a paragraph of logic. Written inline inside a click handler, it buries the button under rules that have nothing to do with buttons.

### What we actually want

- All the related state in **one object**, so it can never disagree with itself.
- All the update rules in **one function**, written once.
- Components that say **what happened** (`"increment"`, `"submit_failed"`), not *how to recalculate everything*.

That is exactly `useReducer`.

> 💡 The name comes from `Array.prototype.reduce`. `reduce` folds a list of values into one result. A reducer folds a stream of actions into one state.

**In simple words:** `useReducer` exists because related state and its update rules deserve to live in one place.

---

## 3. What it actually is

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

It returns a pair, just like `useState`:

- `state` — the current state. Usually an object, but it can be anything.
- `dispatch` — a function you call to send an action. It replaces the setter.

And it takes:

- `reducer` — **your** function, written outside the component: `(state, action) => newState`.
- `initialState` — the starting value.

### The reducer function

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + 1 };
    case "decrement":
      return { ...state, count: state.count - 1 };
    case "reset":
      return { ...state, count: 0 };
    default:
      throw new Error("Unknown action: " + action.type);
  }
}
```

Read it as a rulebook: *given the current state and a slip, what is the new state?*

### The action

An action is a plain object. By convention it has a `type` string and any extra data it needs:

```jsx
dispatch({ type: "increment" });
dispatch({ type: "add_item", item: { id: 3, name: "Pen" } });
dispatch({ type: "field_changed", field: "email", value: "a@b.com" });
```

> 💡 Name actions after **what happened**, not what to set. `"item_added"` beats `"set_items"`. It reads like a story of user events and keeps the *how* inside the reducer.

### The three laws of a reducer

A reducer must be **pure**. That word means three concrete things:

1. **Same inputs → same output, always.** No `Math.random()`, no `Date.now()`, no reading from `localStorage`.
2. **No side effects.** No `fetch`, no timers, no DOM changes, no `console.log` in production code.
3. **Never mutate the state argument.** Return a **new** object. `state.count++` is a bug.

```jsx
// ❌ mutation — React sees the same object and skips the re-render
case "increment":
  state.count++;
  return state;

// ✅ new object
case "increment":
  return { ...state, count: state.count + 1 };
```

Why these laws? Because React may call your reducer more than once with the same inputs (in StrictMode it deliberately does, to catch impurity), and because React compares the returned object by reference to decide whether to re-render — the same `Object.is` rule from note 01.

### `dispatch` is stable

`dispatch` never changes identity between renders. React guarantees it. So it is always safe to leave out of a `useEffect` dependency array, and safe to pass down through context without `useCallback`.

**In simple words:** you write a pure `(state, action) => newState` rulebook, and `dispatch` is how components send it slips.

---

## 4. Syntax / setup, step by step

We will build a counter with a step size — small enough to see every part.

### Step 1 — describe the state as one object

```jsx
const initialState = {
  count: 0,
  step: 1,
};
```

Ask "what pieces always change together?" Those belong in one object.

### Step 2 — write the reducer outside the component

```jsx
// Outside the component: it needs no props, no state, nothing from React.
function counterReducer(state, action) {
  switch (action.type) {
    case "incremented":
      return { ...state, count: state.count + state.step };

    case "decremented":
      return { ...state, count: state.count - state.step };

    case "step_changed":
      return { ...state, step: action.value }; // extra data rides on the action

    case "reset":
      return initialState; // returning a whole new state is fine

    default:
      // Catches typos immediately instead of silently doing nothing
      throw new Error("Unknown action type: " + action.type);
  }
}
```

> 💡 Define the reducer at module level. It does not depend on the component, and keeping it outside means it is not recreated on every render — and can be unit-tested as a plain function.

### Step 3 — call the hook

```jsx
function Counter() {
  const [state, dispatch] = useReducer(counterReducer, initialState);
  // ...
}
```

### Step 4 — dispatch from handlers

```jsx
<button onClick={() => dispatch({ type: "decremented" })}>−</button>
<button onClick={() => dispatch({ type: "incremented" })}>+</button>
<input
  type="number"
  value={state.step}
  onChange={(e) => dispatch({ type: "step_changed", value: Number(e.target.value) })}
/>
```

The handlers are now one line each. All the arithmetic lives in the reducer.

### Optional — lazy initialization with the third argument

`useReducer` takes an optional third argument, an `init` function:

```jsx
function init(startingCount) {
  // Runs only on the first render — good for expensive work
  return { count: startingCount, step: 1 };
}

const [state, dispatch] = useReducer(counterReducer, 0, init);
//                                                   ↑   ↑
//                                             passed to init
```

Two reasons to use it: to skip an expensive computation on every render (same idea as lazy `useState`), and to reuse the same `init` function for a `"reset"` action:

```jsx
case "reset":
  return init(action.payload);
```

**In simple words:** one state object, one reducer outside the component, then `dispatch({ type: "..." })` from your handlers.

---

## 5. Full working example (with comments)

A task list: add, toggle, delete, filter, clear finished tasks. Six pieces of behaviour, one reducer.

```jsx
import { useReducer } from "react";

// ============================================================
// 1. The starting state — everything the list needs, in one object
// ============================================================
const initialState = {
  tasks: [],
  draft: "",        // text in the input box
  filter: "all",    // "all" | "active" | "done"
  nextId: 1,        // simple id generator
};

// ============================================================
// 2. The reducer — the ONLY place tasks are ever changed
// ============================================================
function tasksReducer(state, action) {
  switch (action.type) {
    case "draft_changed":
      return { ...state, draft: action.value };

    case "task_added": {
      const text = state.draft.trim();
      if (!text) return state; // returning the SAME object = React skips re-render

      return {
        ...state,
        tasks: [...state.tasks, { id: state.nextId, text, done: false }],
        draft: "",                  // clear the box in the same step
        nextId: state.nextId + 1,   // and move the id forward
      };
    }

    case "task_toggled":
      return {
        ...state,
        // map creates a new array; only the matching task becomes a new object
        tasks: state.tasks.map((t) =>
          t.id === action.id ? { ...t, done: !t.done } : t
        ),
      };

    case "task_deleted":
      return {
        ...state,
        tasks: state.tasks.filter((t) => t.id !== action.id),
      };

    case "filter_changed":
      return { ...state, filter: action.filter };

    case "done_cleared":
      return { ...state, tasks: state.tasks.filter((t) => !t.done) };

    default:
      throw new Error("Unknown action type: " + action.type);
  }
}

// ============================================================
// 3. The component — reads state, sends actions, nothing else
// ============================================================
function TaskList() {
  const [state, dispatch] = useReducer(tasksReducer, initialState);

  // Derived values — calculated during render, never stored in state
  const visible = state.tasks.filter((t) => {
    if (state.filter === "active") return !t.done;
    if (state.filter === "done") return t.done;
    return true;
  });
  const remaining = state.tasks.filter((t) => !t.done).length;

  function handleSubmit(e) {
    e.preventDefault();               // stop the page reloading
    dispatch({ type: "task_added" }); // the reducer reads state.draft itself
  }

  return (
    <section>
      <h2>Tasks ({remaining} left)</h2>

      <form onSubmit={handleSubmit}>
        <input
          value={state.draft}
          placeholder="What needs doing?"
          onChange={(e) => dispatch({ type: "draft_changed", value: e.target.value })}
        />
        <button type="submit">Add</button>
      </form>

      <div>
        {["all", "active", "done"].map((f) => (
          <button
            key={f}
            disabled={state.filter === f}
            onClick={() => dispatch({ type: "filter_changed", filter: f })}
          >
            {f}
          </button>
        ))}
        <button onClick={() => dispatch({ type: "done_cleared" })}>
          Clear finished
        </button>
      </div>

      {visible.length === 0 ? (
        <p>Nothing here.</p>
      ) : (
        <ul>
          {visible.map((task) => (
            <li key={task.id}>
              <label>
                <input
                  type="checkbox"
                  checked={task.done}
                  onChange={() => dispatch({ type: "task_toggled", id: task.id })}
                />
                <span style={{ textDecoration: task.done ? "line-through" : "none" }}>
                  {task.text}
                </span>
              </label>
              <button onClick={() => dispatch({ type: "task_deleted", id: task.id })}>
                ✕
              </button>
            </li>
          ))}
        </ul>
      )}
    </section>
  );
}

export default TaskList;
```

What to notice:

- The component contains **no update logic at all**. Every handler is one `dispatch` call.
- `"task_added"` changes three fields at once — task list, draft, next id — and they can never fall out of sync, because one `return` sets all three.
- The reducer sits outside the component, so you could import it into a test file and check `tasksReducer(state, action)` with no React involved.
- `visible` and `remaining` are **derived** during render, not stored. Same rule as note 01.

**In simple words:** the component describes the screen and reports events; the reducer owns every rule about how the data changes.

---

## 6. How it works behind the scenes

### What `dispatch` actually does

`dispatch` does not run your reducer on the spot. It queues the action and schedules a re-render.

```
you call dispatch({type:"incremented"})
        ↓
React puts the action in this hook's queue
        ↓
React schedules a re-render of the component
        ↓
─────── during the next render ───────
        ↓
React runs: newState = reducer(currentState, action)
        ↓
compares newState with currentState using Object.is
        ↓
different? -> render with the new state
same object? -> bail out, no re-render
```

Two facts fall straight out of this timeline:

**1. `state` does not change immediately.**

```jsx
function handleClick() {
  dispatch({ type: "incremented" });
  console.log(state.count); // still the OLD value — this render's snapshot
}
```
Exactly the same behaviour as `useState`. `state` is a value captured for this render; the new one arrives on the next render.

**2. Multiple dispatches in one handler all apply, in order.**

```jsx
dispatch({ type: "incremented" }); // count 0 -> 1
dispatch({ type: "incremented" }); // 1 -> 2
dispatch({ type: "incremented" }); // 2 -> 3
// one re-render, final count 3
```

This is where reducers beat plain `setState(value)`. Each action is applied to the result of the previous one, so you never need `setX(prev => ...)` gymnastics — the reducer always receives the latest state.

### Returning the same state stops the render

```jsx
case "task_added": {
  const text = state.draft.trim();
  if (!text) return state; // same reference
```

React compares with `Object.is`. Returning the exact same object means "nothing changed", and React skips the re-render entirely. Returning `{ ...state }` — a copy with identical contents — would *not* skip it, because a copy is a different reference.

### Why StrictMode calls your reducer twice

In development, React deliberately calls the reducer twice with the same arguments and checks that the results match. It is a purity test. If your reducer mutates state or calls `Math.random()`, the two results differ and you have found a real bug early.

```
StrictMode (dev only):  reducer(s, a)  and  reducer(s, a)  -> must match
Production:             reducer(s, a)                      -> once
```

### `useState` is built on `useReducer`

Inside React, `useState` is a small wrapper around the same machinery:

```jsx
// roughly what React does
function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}
```

That is why `setCount(prev => prev + 1)` works — the "action" is your function, and the built-in reducer applies it. `useReducer` just lets you supply your own rulebook instead of that trivial one.

**In simple words:** `dispatch` queues an action, React runs your reducer during the next render, and returning the same object means no re-render.

---

## 7. Comparison with alternatives (table)

### `useState` vs `useReducer`

| | `useState` | `useReducer` |
|---|---|---|
| Best for | One independent value | Several values that change together |
| Update logic lives | In every handler that changes it | In one reducer function |
| Handler code | `setItems(items.map(...))` | `dispatch({ type: "toggled", id })` |
| Next state from previous | `setX(prev => ...)` | Automatic — reducer always gets latest |
| Testing | Needs a rendered component | Plain function call, no React |
| Impossible states | Easy to create by accident | Hard, because one return sets everything |
| Setup cost | Almost none | A reducer, action types, an initial state |
| Debugging | Scattered `console.log`s | One `console.log` in the reducer shows every change |

### When to pick which

| Situation | Use |
|---|---|
| A boolean toggle, one input, one counter | `useState` |
| 2–3 unrelated values | `useState` (several times) |
| 4+ values that must stay consistent | `useReducer` |
| The same update logic in several components | `useReducer` |
| Next state depends on current state in a complex way | `useReducer` |
| An explicit list of "things that can happen" | `useReducer` |
| Server data (loading / data / error) | `useReducer`, or a data library later |

### `useReducer` + context vs Redux Toolkit

| | `useReducer` + context | Redux Toolkit |
|---|---|---|
| Extra install | None — built into React | Yes |
| Devtools time-travel | No | Yes |
| Async / middleware | Do it yourself in effects | Built in (thunks, RTK Query) |
| Re-render control | All consumers re-render | Fine-grained selectors |
| Good for | Small and medium apps | Large apps, big teams |

> 💡 Combining `useReducer` with the context from note 04 gives you a "small Redux" with no library at all: put `state` and `dispatch` in a provider, and any component can read state or send actions. Because `dispatch` is stable, passing it through context costs nothing.

**In simple words:** start with `useState`, move to `useReducer` when the state pieces stop being independent, and add a library only when you need devtools or async tooling.

---

## 8. Common mistakes beginners make

**1. Mutating the state argument**

```jsx
case "task_added":
  state.tasks.push(newTask); // ❌ same array, same object
  return state;              // React sees no change — nothing re-renders
```
Return a new object with a new array: `{ ...state, tasks: [...state.tasks, newTask] }`.

**2. Putting side effects inside the reducer**

```jsx
case "saved":
  fetch("/api/save", { method: "POST" }); // ❌ not pure; runs twice in StrictMode
  return { ...state, saved: true };
```
Reducers only compute the next state. Do the `fetch` in the event handler or in a `useEffect` that watches the state.

**3. Reading the new state right after dispatching**

```jsx
dispatch({ type: "incremented" });
console.log(state.count); // ❌ still the old value
```
The new state arrives on the next render. Compute the value yourself if you need it now.

**4. Forgetting `...state` in a case**

```jsx
case "filter_changed":
  return { filter: action.filter }; // ❌ tasks, draft and nextId are now undefined
```
Always spread the old state first, then override.

**5. No `default` case**

Without one, a typo like `"increment"` instead of `"incremented"` silently returns `undefined` (or nothing happens). Throwing makes the bug visible instantly.

**6. Defining the reducer inside the component**

```jsx
function Counter() {
  function reducer(state, action) { /* ... */ } // ❌ recreated every render
```
It works, but it is recreated each render and cannot be tested on its own. Keep it at module level unless it genuinely needs a prop.

**7. Actions named after setters**

`{ type: "set_tasks", tasks: newTasks }` moves the logic back into the component and defeats the point. Name the event: `"task_added"`, `"done_cleared"`.

**8. Cramming unrelated state into one reducer**

The theme and the task list have nothing to do with each other. Two reducers, or one reducer and one `useState`, is clearer than one giant object.

**9. Storing derived data in state**

```jsx
return { ...state, tasks: newTasks, remaining: newTasks.filter(t => !t.done).length };
// ❌ remaining can drift out of sync
```
Calculate `remaining` during render instead.

**10. Reaching for `useReducer` too early**

One checkbox does not need a reducer. The extra structure only pays off when the state has real complexity.

**In simple words:** keep the reducer pure, always spread the old state, and never expect the new state on the same line as the dispatch.

---

## 9. Cheat sheet

```jsx
// 1. initial state — related values in one object
const initialState = { count: 0, step: 1 };

// 2. reducer — module level, pure, (state, action) => newState
function reducer(state, action) {
  switch (action.type) {
    case "incremented":
      return { ...state, count: state.count + state.step };
    case "reset":
      return initialState;
    default:
      throw new Error("Unknown action: " + action.type);
  }
}

// 3. in the component
const [state, dispatch] = useReducer(reducer, initialState);

// 4. send an action
dispatch({ type: "incremented" });
dispatch({ type: "step_changed", value: 5 }); // extra data on the action

// optional: lazy init (third argument)
const [state, dispatch] = useReducer(reducer, 0, init);
```

| Thing | Rule |
|---|---|
| Signature | `useReducer(reducer, initialState, init?)` |
| Returns | `[state, dispatch]` |
| Reducer shape | `(state, action) => newState` — pure, no mutation, no side effects |
| Action shape | Plain object, `type` string + any extra data |
| Naming | Name what **happened**, not what to set |
| `dispatch` identity | Stable forever — safe to skip in dependency arrays |
| Reading new state | Not available immediately; arrives next render |
| Return same object | React bails out — no re-render |
| `default` case | Always throw, to catch typos |
| Where to define | Module level, outside the component |
| StrictMode | Calls the reducer twice in dev to check purity |
| With context | Provide `state` and `dispatch` → a small Redux, no library |

**In simple words:** one object of state, one pure reducer, actions named after events, and `dispatch` from anywhere.

---

## 10. Revision questions (with answers)

**1. What does `useReducer` return?**
`[state, dispatch]` — the current state, and a function to send actions.

**2. What is the signature of a reducer?**
`(state, action) => newState`. It takes the current state and an action, and returns the next state.

**3. What is an action?**
A plain object describing what happened, normally with a `type` string and any extra data the reducer needs.

**4. What are the three purity rules for a reducer?**
Same inputs give the same output; no side effects (no fetch, timers, logging); never mutate the state argument — return a new object.

**5. Why does mutating `state` inside a reducer break things?**
React compares the returned value with the old one using `Object.is`. A mutated object is the same reference, so React sees no change and skips the re-render.

**6. What happens if a reducer returns the exact same state object?**
React bails out and does not re-render. This is a deliberate optimisation — used above for "ignore an empty task".

**7. Can you read the new state on the line after `dispatch`?**
No. `state` is this render's snapshot; the new value arrives on the next render.

**8. Three dispatches fire in one click handler. How many renders, and what is the result?**
One re-render. Each action is applied in order to the result of the previous one, so all three take effect.

**9. Why is it safe to leave `dispatch` out of a `useEffect` dependency array?**
React guarantees `dispatch` keeps the same identity for the life of the component, so it can never be the reason an effect re-runs.

**10. Why does React call the reducer twice in StrictMode?**
To test purity in development. Two identical calls must give identical results; if they differ, the reducer has a side effect or a mutation.

**11. Give two clear signs it is time to move from `useState` to `useReducer`.**
Several state values must always change together to stay consistent, or the same update logic is being repeated in more than one component.

**12. How do `useReducer` and context combine?**
Put `state` and `dispatch` into a context provider. Any component below can read state or dispatch actions with no prop drilling — a small Redux with no library.

**13. Why name an action `"task_added"` instead of `"set_tasks"`?**
`"task_added"` describes the event and keeps the *how* inside the reducer. `"set_tasks"` pushes the logic back into the component, which is the problem `useReducer` solves.

---

## 11. What to learn next

You can now hold complex state and share it across the tree. The next question is **speed**: React re-renders a component whenever its state or props change, and sometimes that means redoing an expensive calculation for no reason.

`useMemo` remembers the *result* of a calculation and reuses it while the inputs are unchanged. It is also the tool you already met in note 04 for keeping a context value stable.

➡ Next note: `06_use_memo.md`

Related notes:
- [01. useState](01_use_state.md) — the simpler sibling; `useState` is `useReducer` underneath
- [04. useContext](04_use_context.md) — pair it with a reducer for app-wide state
- [02. useEffect](02_use_effect.md) — where the side effects a reducer must not do belong

⬅ [Back to chapter index](README.md)
