# 01. useState

> `useState` gives a component **memory**. It stores a value between renders, and when you change that value React redraws the component so the screen matches it.

---

## 1. Real-life analogy

Think of a scoreboard at a cricket ground.

The scoreboard has a **number** on it, and it has an **operator** sitting behind it. Nobody walks up and scratches the number off the board by hand. Instead you tell the operator: *"the score is now 47"*. The operator writes the new number, and everyone watching sees it change.

Two important details:

- The board **remembers** the score. It does not reset to zero every time somebody looks at it.
- The only way to change it is **through the operator**. That is what keeps every viewer in sync.

`useState` is exactly this pair:

```
const [score, setScore] = useState(0);
              ↑              ↑
        the operator     the number written on the board
```

`score` is the number currently on the board. `setScore` is the operator. You never write on the board yourself — you always go through the operator, and the operator makes sure the screen updates too.

**In simple words:** state is a value the component remembers, plus the only official way to change it.

---

## 2. The problem — why does this exist?

A component is just a function. Functions forget everything when they finish.

Try counting clicks with a normal variable:

```jsx
function Counter() {
  let count = 0; // a normal variable

  function handleClick() {
    count = count + 1;      // the number really does change...
    console.log(count);     // ...the console prints 1, 2, 3
  }

  return <button onClick={handleClick}>Clicked {count} times</button>;
}
```

Click the button. The console prints `1`, `2`, `3`. **But the screen keeps saying "Clicked 0 times".**

Two separate things went wrong:

| Problem | What happened |
|---------|---------------|
| React does not know anything changed | Changing a plain variable is invisible to React. Nothing tells it to redraw. |
| The value is thrown away | If React *did* re-render, the function would run again from the top and `let count = 0` would reset it. |

So we need a value that:

1. **survives** across renders, and
2. **tells React** to re-render when it changes.

A plain variable gives us neither. A variable outside the component fixes (1) but not (2) — and it would be shared by every copy of the component, so two counters on the page would fight over one number.

`useState` solves both at once.

```jsx
function Counter() {
  const [count, setCount] = useState(0); // survives + notifies React

  return <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>;
}
```

**In simple words:** normal variables reset and are silent; state survives and speaks up.

---

## 3. What it actually is

`useState` is a **hook** — a function React gives you, whose name starts with `use`.

You call it inside a component. It returns an **array of exactly two items**:

```jsx
const [value, setValue] = useState(initialValue);
```

| Part | What it is | What it does |
|------|------------|--------------|
| `value` | any JavaScript value | the current state for **this render** |
| `setValue` | a function | asks React to store a new value and re-render |
| `initialValue` | any JavaScript value | used **only on the very first render** |

Three things beginners are surprised by:

**a) `value` is a constant inside one render.**
It is declared with `const`. It will not change while your function is running. It is a snapshot.

**b) The names are yours.**
`useState` returns a plain array, and `[a, b]` is just array destructuring. `const [count, setCount]` and `const [x, y]` both work. Everyone writes `[thing, setThing]` because it reads well — follow that.

**c) `initialValue` is ignored after the first render.**
React only uses it to set up the slot. On render 2, 3, 4… React hands back whatever is stored, and the argument you passed is thrown away.

```jsx
const [count, setCount] = useState(0);
// render 1 -> count is 0   (used the argument)
// render 2 -> count is 1   (argument ignored, stored value wins)
```

**In simple words:** `useState` hands you a snapshot of a remembered value and a function to replace it.

---

## 4. Syntax / setup, step by step

### Step 1 — import it

```jsx
import { useState } from "react"; // named import, curly braces required
```

### Step 2 — call it at the top of the component

```jsx
function Form() {
  const [name, setName] = useState(""); // top level — not inside if/loop/function
  // ...
}
```

### Step 3 — read the value in JSX

```jsx
return <p>Hello {name}</p>;
```

### Step 4 — update it from an event

```jsx
return <input value={name} onChange={(e) => setName(e.target.value)} />;
```

### Step 5 — repeat for each independent piece of state

```jsx
const [name, setName] = useState("");
const [age, setAge] = useState(0);
const [isMember, setIsMember] = useState(false);
```

Any number of `useState` calls is fine. They are separate slots.

### Updating from the previous value

If the new value depends on the old one, pass a **function** instead of a value:

```jsx
setCount(count + 1);           // "set it to this number"
setCount((prev) => prev + 1);  // "take whatever is there and add 1"   ← safer
```

The second form is called an **updater function**. React calls it with the latest stored value. Section 8 shows exactly when the first form breaks.

### Lazy initial value

If computing the initial value is expensive, pass a function. React runs it **once**, on the first render only:

```jsx
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem("items")) || []);
//                                 ↑ runs once, not on every render
```

> ⚠️ `useState(expensiveCall())` runs `expensiveCall()` on **every** render and throws the result away. `useState(() => expensiveCall())` runs it once.

**In simple words:** import it, call it at the top, read the value, and change it only through the setter.

---

## 5. Full working example (with comments)

A small shopping list. It shows text state, number state, boolean state, and array state together.

```jsx
import { useState } from "react";

function ShoppingList() {
  // 1. text state — what the user is typing right now
  const [text, setText] = useState("");

  // 2. array state — the saved items. Each item: { id, name, done }
  const [items, setItems] = useState([]);

  // 3. boolean state — a filter toggle
  const [hideDone, setHideDone] = useState(false);

  function handleAdd(e) {
    e.preventDefault();              // stop the form reloading the page
    if (text.trim() === "") return;  // ignore empty input

    const newItem = {
      id: Date.now(),                // quick unique id (fine for a demo)
      name: text.trim(),
      done: false,
    };

    // NEVER items.push(newItem) — that changes the old array in place.
    // Build a BRAND NEW array so React sees a different value.
    setItems([...items, newItem]);

    setText(""); // clear the input by clearing its state
  }

  function toggleDone(id) {
    // map returns a new array; we replace only the matching item
    setItems(items.map((item) =>
      item.id === id
        ? { ...item, done: !item.done } // new object, flipped flag
        : item                          // untouched items are reused
    ));
  }

  function removeItem(id) {
    // filter also returns a new array
    setItems(items.filter((item) => item.id !== id));
  }

  // Derived value — calculated during render, NOT stored in state.
  const visible = hideDone ? items.filter((i) => !i.done) : items;
  const remaining = items.filter((i) => !i.done).length;

  return (
    <div>
      <h2>Shopping list ({remaining} left)</h2>

      <form onSubmit={handleAdd}>
        <input
          value={text}                                 // state drives the input
          onChange={(e) => setText(e.target.value)}    // input updates the state
          placeholder="Add an item"
        />
        <button type="submit">Add</button>
      </form>

      <label>
        <input
          type="checkbox"
          checked={hideDone}
          onChange={() => setHideDone((prev) => !prev)} // flip the boolean
        />
        Hide done items
      </label>

      {visible.length === 0 ? (
        <p>Nothing to show.</p>
      ) : (
        <ul>
          {visible.map((item) => (
            <li key={item.id}>
              <input
                type="checkbox"
                checked={item.done}
                onChange={() => toggleDone(item.id)}
              />
              <span style={{ textDecoration: item.done ? "line-through" : "none" }}>
                {item.name}
              </span>
              <button onClick={() => removeItem(item.id)}>x</button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

export default ShoppingList;
```

Notice `remaining` and `visible`. They are **not** state. They can be calculated from `items`, so we calculate them on every render. Storing them in state would mean keeping two sources of truth in sync by hand.

> 💡 Rule: if you can compute it from existing state or props, do not put it in state.

**In simple words:** state holds the raw facts; everything else is calculated from them during render.

---

## 6. How it works behind the scenes

### Where is the value actually kept?

Not in your function. React keeps a list of state slots attached to the component instance, in a structure called the **fiber**.

```
Your component            React's memory for it
--------------            ---------------------
useState("")      slot 0 -> ""
useState([])      slot 1 -> []
useState(false)   slot 2 -> false
```

On the first render React creates the slots and fills them with your initial values. On every later render React just walks the same list in order and hands each value back.

This is why hooks must be called in the **same order every time**. React matches them by position, not by name. An `if` around a `useState` shifts every slot after it.

### What happens when you call `setCount`?

```
setCount(1)
   │
   ├─ 1. React stores 1 in the slot (your `count` variable does NOT change)
   ├─ 2. React marks this component as "needs re-render"
   ├─ 3. React schedules the work — it does not run it immediately
   │
   ▼  (later, in the same tick)
   4. React calls your function again from the top
   5. useState now returns 1
   6. React compares the new JSX with the old JSX
   7. Only the changed DOM parts are updated
```

Step 1 is the one that trips everybody up:

```jsx
function handleClick() {
  console.log(count); // 0
  setCount(count + 1);
  console.log(count); // still 0 — NOT 1
}
```

`count` is a `const` captured by this render. Setting state does not reach back in time and edit it. You will see `1` in the **next** render.

### Batching

React groups multiple `setState` calls in the same event into **one** re-render:

```jsx
function handleClick() {
  setA(1);
  setB(2);
  setC(3);
} // ONE re-render, not three
```

### Bail-out

If you set the same value, React may skip the re-render entirely:

```jsx
setCount(5);
setCount(5); // same value -> React can skip the work
```

The comparison is `Object.is`, which is basically `===`. This is exactly why mutating an array or object fails: `items.push(x)` keeps the **same array reference**, so `Object.is(old, new)` is `true` and React concludes nothing changed.

```
setItems(items.push(x))       ❌ push returns a number, and mutates the old array
setItems([...items, x])       ✅ brand new array, new reference, React re-renders
```

**In simple words:** React stores state by slot order, replaces the value on set, then re-runs your function — your old variables never change.

---

## 7. Comparison with alternatives

| Way to store a value | Survives re-render? | Triggers re-render? | Per component instance? | Use it for |
|---|---|---|---|---|
| `let x = 0` inside the component | ❌ resets every render | ❌ | — | temporary calculations during one render |
| `let x = 0` outside the component | ✅ | ❌ | ❌ shared by all instances | constants, module config |
| `useState` | ✅ | ✅ | ✅ | anything the user sees change |
| `useRef` | ✅ | ❌ | ✅ | timer ids, DOM nodes, values the screen does not show |
| `useReducer` | ✅ | ✅ | ✅ | many related values updated together by clear actions |
| props | ✅ (owned by parent) | ✅ (when parent re-renders) | ✅ | data the parent owns |

Quick decision guide:

```
Does the screen need to change when this value changes?
├── No  -> plain variable, or useRef if it must survive renders
└── Yes -> Can you calculate it from existing state/props?
          ├── Yes -> calculate it during render, do not store it
          └── No  -> Is it several fields that always change together?
                    ├── Yes -> useReducer
                    └── No  -> useState  ✅
```

**In simple words:** use state only for values that change *and* that the user can see the effect of.

---

## 8. Common mistakes beginners make

### 1. Expecting the value to change immediately

```jsx
setCount(count + 1);
console.log(count); // ❌ old value
```
The variable belongs to this render. Read the new value in the next render, not on the line after.

### 2. Calling the setter twice with the stale value

```jsx
setCount(count + 1);
setCount(count + 1); // ❌ count is 0 both times -> final value 1, not 2
```
```jsx
setCount((prev) => prev + 1);
setCount((prev) => prev + 1); // ✅ 0 -> 1 -> 2
```
Use the updater form whenever the new value depends on the old one.

### 3. Mutating arrays or objects

```jsx
items.push(newItem);  setItems(items);        // ❌ same reference, no re-render
user.name = "Naman";  setUser(user);          // ❌ same object

setItems([...items, newItem]);                // ✅
setUser({ ...user, name: "Naman" });          // ✅
```
Always create a new array/object. Copy with `...`, then change the copy.

### 4. Calling the setter during render

```jsx
function Bad() {
  const [n, setN] = useState(0);
  setN(n + 1); // ❌ set -> render -> set -> render -> infinite loop
  return <p>{n}</p>;
}
```
Setters belong in event handlers or effects, never in the render body.

### 5. Calling the handler instead of passing it

```jsx
<button onClick={setCount(count + 1)}>   {/* ❌ runs during render */}
<button onClick={() => setCount(count + 1)}>  {/* ✅ runs on click */}
```

### 6. Putting derived data in state

```jsx
const [items, setItems] = useState([]);
const [count, setCount] = useState(0); // ❌ now you must update it everywhere
const count = items.length;            // ✅ always correct, free
```

### 7. Hooks inside conditions

```jsx
if (loggedIn) {
  const [name, setName] = useState(""); // ❌ breaks slot order
}
```
Always top level.

### 8. Forgetting nested copies

```jsx
setUser({ ...user, address: { ...user.address, city: "Delhi" } });
// ✅ spread every level you change — a shallow copy shares the inner objects
```

### 9. Wrong initial type

```jsx
const [items, setItems] = useState();      // ❌ undefined.map crashes
const [items, setItems] = useState([]);    // ✅ start with the right shape
```

### 10. One state per keystroke field, then trying to sync them

Two `useState` calls that must always agree usually mean one of them should be derived, or both belong in a single object.

**In simple words:** never mutate, never read the value right after setting it, and never store what you can calculate.

---

## 9. Cheat sheet

```jsx
import { useState } from "react";

// declare
const [value, setValue] = useState(initial);

// lazy initial (runs once)
const [v, setV] = useState(() => expensive());

// replace
setValue(newValue);

// update from previous  ← use when new depends on old
setValue((prev) => prev + 1);

// booleans
setOpen((prev) => !prev);

// strings from an input
onChange={(e) => setText(e.target.value)}

// arrays — always a NEW array
setItems([...items, item]);                                   // add end
setItems([item, ...items]);                                   // add start
setItems(items.filter((i) => i.id !== id));                   // remove
setItems(items.map((i) => (i.id === id ? { ...i, done: true } : i))); // update one

// objects — always a NEW object
setUser({ ...user, name: "Naman" });                          // change one field
setUser({ ...user, address: { ...user.address, city: "Delhi" } }); // nested
```

| Question | Answer |
|---|---|
| Does `setX` change `x` right away? | No. Next render. |
| Do 3 setters cause 3 renders? | No. React batches them into 1. |
| When is the initial value used? | First render only. |
| Can I have many `useState` calls? | Yes, any number. |
| Why must the array/object be new? | React compares by reference (`Object.is`). |
| Where do I call `useState`? | Top level of a component or custom hook. |

---

## 10. Revision questions (with answers)

**Q1. Why does a plain `let count = 0` not work for a counter?**
It resets to `0` every render, and changing it does not tell React to re-render. State fixes both.

**Q2. What does `useState` return?**
An array of two things: the current value for this render, and a setter function.

**Q3. What does this print?**
```jsx
const [n, setN] = useState(0);
function click() {
  setN(n + 1);
  console.log(n);
}
```
`0`. `n` is a constant in this render. The new value appears in the next render.

**Q4. What is the final value after these two lines, starting from 0?**
```jsx
setN(n + 1);
setN(n + 1);
```
`1`. Both calls read the same stale `n` (0). With `setN(prev => prev + 1)` twice you get `2`.

**Q5. Why does `items.push(x); setItems(items);` not update the screen?**
`push` mutates the existing array, so the reference is unchanged. React compares with `Object.is`, sees the same array, and skips the re-render.

**Q6. When is the argument to `useState` used?**
Only on the first render. After that React returns the stored value and ignores the argument.

**Q7. Difference between `useState(expensive())` and `useState(() => expensive())`?**
The first runs `expensive()` on every render and discards the result. The second runs it once, on the first render.

**Q8. Should `const total = price * qty` be state?**
No. It is derived from existing state, so calculate it during render. Storing it creates a second source of truth.

**Q9. Why can hooks not go inside an `if`?**
React matches hooks to stored values by call order. Skipping a call shifts every later hook to the wrong slot.

**Q10. Two `<Counter />` components on one page — do they share the count?**
No. Each instance gets its own state slots.

---

## 11. What to learn next

- **`useEffect`** — run code after render: fetching data, timers, subscriptions, and cleaning them up. This is the natural partner to `useState`.
- **Lifting state up** — when two components need the same value, move the `useState` to their nearest common parent and pass it down as props.
- **`useRef`** — for values that must survive renders but should *not* cause a re-render.
- **`useReducer`** — when one component has five or six related `useState` calls that always change together.

➡ Next note: `02_use_effect.md`

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
