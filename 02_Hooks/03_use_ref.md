# 03. useRef

> `useRef` gives a component a **box** that survives every render. Changing what is inside the box does **not** re-render the component. It is used for two things: pointing at a DOM element, and remembering a value the screen does not display.

---

## 1. Real-life analogy

Imagine a shop with a big display board out front and a small notebook under the counter.

The **display board** is state. Every time you change it, you climb up, rewrite it, and every customer on the street sees the new text. Changing it is a public event.

The **notebook** is a ref. You scribble in it — the delivery van's number plate, the time the last customer left, which drawer the keys are in. Nobody outside sees it. Nothing about the shop's appearance changes. But the notebook is still there tomorrow morning; it is not thrown away at closing time.

There is a second use for the notebook: writing down **where things are**. "Cash box → third drawer." That is exactly what a DOM ref is — a note saying "the `<input>` element is *right here*", so you can walk up and focus it later.

**In simple words:** state is the public board, a ref is the private notebook — both remembered, only one visible.

---

## 2. The problem — why does this exist?

### Problem A — a value that must survive, but must not re-render

Build a stopwatch. `setInterval` gives you a timer id, and you need that id later to stop the timer.

```jsx
function Stopwatch() {
  const [seconds, setSeconds] = useState(0);
  let intervalId = null; // ❌ a normal variable

  function start() {
    intervalId = setInterval(() => setSeconds((s) => s + 1), 1000);
  }

  function stop() {
    clearInterval(intervalId); // ❌ intervalId is null again — the timer never stops
  }
}
```

`setSeconds` re-renders the component, the function runs again from the top, and `let intervalId = null` wipes the id. The timer runs forever with no way to reach it.

So use state instead?

```jsx
const [intervalId, setIntervalId] = useState(null); // ❌ works, but wasteful
```

It works, but it forces an extra re-render every time you store an id nobody ever sees on screen. Re-rendering to store an invisible value is pure waste.

We need: **survives renders** ✅ + **does not trigger a render** ✅. Neither a plain variable nor state gives us both.

### Problem B — reaching a real DOM element

React normally builds the DOM for you. But some jobs have no React equivalent:

```jsx
// ❌ You cannot do these declaratively
document.querySelector("input").focus();
videoElement.play();
divElement.scrollIntoView();
```

Using `document.querySelector` inside a component is fragile — it searches the whole page and may grab the wrong element if the component appears twice. You want a direct handle on *your* element.

`useRef` answers both problems with the same tool.

**In simple words:** refs exist for values that must persist silently, and for talking to real DOM nodes.

---

## 3. What it actually is

```jsx
const myRef = useRef(initialValue);
```

`useRef` returns a **plain object with exactly one property**:

```js
{ current: initialValue }
```

That is the entire hook. Three facts make it useful:

| Fact | Meaning |
|------|---------|
| React keeps the **same object** on every render | `myRef` on render 5 is the identical object from render 1 |
| You can freely write `myRef.current = x` | it is a normal mutable property, no setter needed |
| Writing to it does **not** re-render | React does not watch this object at all |

```jsx
const ref = useRef(0);

ref.current = 5;      // allowed, immediate, silent
console.log(ref.current); // 5 — right away, unlike state
```

Compare that with state, where `setCount(5)` does not change `count` until the next render.

### Why `.current` exists at all

A function cannot return a value that later changes — numbers and strings are copied, not shared. But an **object** is shared by reference. So React returns a box, and you change what is inside the box. The box itself never changes identity.

```
render 1 ─┐
render 2 ─┼──► the SAME object ──► { current: <whatever you last put in> }
render 3 ─┘
```

**In simple words:** `useRef` is one stable object with a `current` slot you can write to at any time.

---

## 4. Syntax / setup, step by step

### Use A — pointing at a DOM element

**Step 1.** Create a ref, starting at `null`.

```jsx
const inputRef = useRef(null);
```

**Step 2.** Attach it with the special `ref` attribute.

```jsx
<input ref={inputRef} />
```

**Step 3.** After React puts the element on screen, it sets `inputRef.current` to the real DOM node.

```jsx
function focusIt() {
  inputRef.current.focus(); // a real <input> element, with all its browser methods
}
```

**Step 4.** Read it only in event handlers or effects — never during render.

```jsx
useEffect(() => {
  inputRef.current.focus(); // ✅ the DOM exists by now
}, []);
```

```jsx
function Bad() {
  const inputRef = useRef(null);
  console.log(inputRef.current); // ❌ null — the element does not exist yet
  return <input ref={inputRef} />;
}
```

> ⚠️ During the first render `ref.current` is still `null`. React attaches the node **after** the DOM is committed.

### Use B — remembering a value

```jsx
const timerRef = useRef(null);

function start() {
  timerRef.current = setInterval(tick, 1000); // write
}

function stop() {
  clearInterval(timerRef.current);            // read
  timerRef.current = null;
}
```

No setter, no dependency array, no re-render.

### Passing a ref to your own component

A ref attaches to a DOM tag by default. To let a parent focus an input that lives inside *your* component, accept `ref` as a normal prop:

```jsx
// Modern React (19+): ref is just a prop
function TextField({ label, ref }) {
  return (
    <label>
      {label}
      <input ref={ref} /> {/* forward it down to the real DOM tag */}
    </label>
  );
}

// Parent
const nameRef = useRef(null);
<TextField label="Name" ref={nameRef} />;
```

> 💡 In older tutorials you will see `forwardRef(function TextField(props, ref) {...})`. That wrapper was required before React 19 and still works. New code does not need it.

**In simple words:** attach with `ref={}` for DOM nodes, or just read and write `.current` for plain values.

---

## 5. Full working example (with comments)

A stopwatch. It uses **both** kinds of ref, plus state for the part the user actually sees.

```jsx
import { useState, useRef, useEffect } from "react";

function Stopwatch() {
  // STATE — the user sees these, so changing them must re-render
  const [ms, setMs] = useState(0);
  const [running, setRunning] = useState(false);
  const [laps, setLaps] = useState([]);

  // REF 1 — the interval id. Invisible plumbing, must survive renders.
  const timerRef = useRef(null);

  // REF 2 — a DOM node we want to control directly.
  const lapListRef = useRef(null);

  function start() {
    if (timerRef.current !== null) return; // already running — do not stack timers

    // Updater form: never read stale state inside an interval
    timerRef.current = setInterval(() => setMs((prev) => prev + 10), 10);
    setRunning(true);
  }

  function stop() {
    clearInterval(timerRef.current);
    timerRef.current = null; // reset the box so start() works again
    setRunning(false);
  }

  function reset() {
    stop();
    setMs(0);
    setLaps([]);
  }

  function addLap() {
    setLaps((prev) => [...prev, ms]); // new array, never push
  }

  // Scroll the lap list to the bottom whenever a lap is added.
  // There is no declarative way to say "scroll here" — we need the DOM node.
  useEffect(() => {
    if (lapListRef.current) {
      lapListRef.current.scrollTop = lapListRef.current.scrollHeight;
    }
  }, [laps]);

  // CLEANUP — if the component is removed while running, kill the timer.
  // Without this the interval keeps firing against a component that is gone.
  useEffect(() => {
    return () => clearInterval(timerRef.current);
  }, []);

  const format = (t) =>
    `${String(Math.floor(t / 1000)).padStart(2, "0")}.${String(t % 1000).padStart(3, "0")}`;

  return (
    <div>
      <h2>{format(ms)}</h2>

      <button onClick={running ? stop : start}>{running ? "Stop" : "Start"}</button>
      <button onClick={addLap} disabled={!running}>Lap</button>
      <button onClick={reset}>Reset</button>

      <ul ref={lapListRef} style={{ height: 120, overflowY: "auto" }}>
        {laps.map((lap, i) => (
          <li key={i}>Lap {i + 1}: {format(lap)}</li>
        ))}
      </ul>
    </div>
  );
}

export default Stopwatch;
```

Read the split carefully:

| Value | Kind | Why |
|---|---|---|
| `ms`, `running`, `laps` | state | shown on screen — the screen must update |
| `timerRef` | ref | an id number nobody sees; changing it should not re-render |
| `lapListRef` | ref | a handle on a real `<ul>` element |

> 💡 Ask one question: **"if this value changes, does the screen need to redraw?"** Yes → state. No → ref.

**In simple words:** state for what the user sees, refs for the machinery behind it.

---

## 6. How it works behind the scenes

### Storage

`useRef` uses the **same slot mechanism as `useState`** — a numbered position on the component's fiber. That is why it obeys the same rules of hooks: top level, same order every render.

The difference is what React does afterwards:

```
setState(x)               ref.current = x
    │                          │
    ├─ store the value         ├─ store the value
    ├─ mark for re-render      │  (that is all)
    └─ schedule work           └─ nothing else happens
```

React literally does not know you wrote to `.current`. There is no watcher, no proxy, no comparison.

In fact `useRef(initial)` is close to:

```jsx
const [ref] = useState(() => ({ current: initial })); // one object, created once, never replaced
```

### When does React set a DOM ref?

During the **commit** phase, between updating the DOM and running your effects:

```
1. render      → your function runs        (ref.current is still the OLD value)
2. commit      → DOM is updated
3. ───────────► React sets ref.current = the DOM node
4. paint       → the user sees it
5. effects     → your useEffect runs       (ref.current is ready ✅)
```

And when the element is removed, React sets `ref.current = null` for you.

This ordering is the whole reason for the rule *"do not read a DOM ref during render"* — at step 1 it has not been assigned yet.

### Why writing during render breaks things

```jsx
function Broken() {
  const renders = useRef(0);
  renders.current++;              // ❌ mutation during render
  return <p>{renders.current}</p>;
}
```

Two problems. First, rendering must be pure, and React may render a component twice (StrictMode) or throw a render away — your count becomes wrong. Second, the number is displayed, so it is really state pretending to be a ref.

Do the increment in an effect instead:

```jsx
useEffect(() => { renders.current++; }); // ✅ after commit, once per real render
```

**In simple words:** refs live in the same slots as state, but React never reacts to them — and DOM refs are filled in after the DOM is built.

---

## 7. Comparison with alternatives

| | `useState` | `useRef` | plain `let` in the component | variable outside the component |
|---|---|---|---|---|
| Survives re-render | ✅ | ✅ | ❌ | ✅ |
| Triggers re-render | ✅ | ❌ | ❌ | ❌ |
| Separate per instance | ✅ | ✅ | ✅ (but resets) | ❌ shared by all |
| Read the new value immediately | ❌ next render | ✅ instantly | ✅ | ✅ |
| Safe to change during render | ❌ | ❌ | ✅ | ❌ |
| Good for | anything visible | timers, DOM nodes, previous values | temporary math | true constants |

### Typical jobs for a ref

| Job | Example |
|---|---|
| Timer / interval id | `timerRef.current = setInterval(...)` |
| DOM element | `inputRef.current.focus()` |
| Previous value of a prop or state | see the pattern below |
| "Has this already run once?" flag | `if (didInit.current) return;` |
| A value read inside a callback that must not re-create the callback | avoids adding a dependency |

### The previous-value pattern

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);

  useEffect(() => {
    ref.current = value; // runs AFTER render, so during render it holds the old value
  }, [value]);

  return ref.current;
}

// usage
const prevCount = usePrevious(count);
// render shows: now = 5, before = 4
```

This works because the effect runs after render: during render, `ref.current` still holds the value from last time.

**In simple words:** reach for a ref only when a value must persist and the screen must not react to it.

---

## 8. Common mistakes beginners make

### 1. Expecting the screen to update

```jsx
const countRef = useRef(0);
<button onClick={() => countRef.current++}>Count: {countRef.current}</button>
// ❌ the number never changes on screen
```
The value really does increase — React just never re-renders. If it is displayed, it is state.

### 2. Reading a DOM ref during render

```jsx
const inputRef = useRef(null);
console.log(inputRef.current.value); // ❌ TypeError: current is null
```
Read it in an event handler or an effect.

### 3. Writing to a ref during render

```jsx
ref.current = props.value; // ❌ impure; StrictMode and re-renders make it unreliable
```
Do it in an effect or a handler.

### 4. Using a ref to dodge the dependency array

```jsx
const dataRef = useRef(data);
useEffect(() => { console.log(dataRef.current); }, []); // ❌ often reads a stale value
```
This is sometimes a real technique, but as a beginner treat it as a smell. Fix the dependencies instead.

### 5. Forgetting to clear a timer stored in a ref

```jsx
useEffect(() => () => clearInterval(timerRef.current), []); // ✅ always add this
```
A ref keeps the id alive, but nothing stops the timer for you.

### 6. Manually changing the DOM React owns

```jsx
divRef.current.innerHTML = "<b>hi</b>";  // ❌ React will overwrite it on the next render
```
Use refs to *read* or to call browser methods (`focus`, `play`, `scrollIntoView`), not to rewrite content React manages.

### 7. Creating a ref outside the component

```jsx
const ref = useRef(null);        // ❌ at module level — hooks only work inside components
function Comp() { ... }
```

### 8. `useRef(0)` when you meant `useState(0)`

If a value drives what the user sees, it is state. Every time.

### 9. Not resetting `.current` after use

```jsx
clearInterval(timerRef.current);
timerRef.current = null; // ✅ otherwise "is it running?" checks lie
```

### 10. Reaching for `document.getElementById` instead of a ref

```jsx
document.getElementById("name").focus(); // ❌ breaks with two copies of the component
nameRef.current.focus();                 // ✅ always your own element
```

**In simple words:** never display a ref, never touch it during render, and always clean up what you stored in it.

---

## 9. Cheat sheet

```jsx
import { useRef, useEffect } from "react";

// create
const ref = useRef(initialValue);   // -> { current: initialValue }

// read / write — instant, silent, no re-render
ref.current = 42;
console.log(ref.current);

// DOM element
const inputRef = useRef(null);
<input ref={inputRef} />
useEffect(() => { inputRef.current.focus(); }, []);   // focus on mount

// common DOM calls
inputRef.current.focus();
inputRef.current.select();
videoRef.current.play();
boxRef.current.scrollIntoView({ behavior: "smooth" });
const { width, height } = boxRef.current.getBoundingClientRect();

// timer id
const timerRef = useRef(null);
timerRef.current = setInterval(tick, 1000);
clearInterval(timerRef.current);
useEffect(() => () => clearInterval(timerRef.current), []);

// run-once flag
const didInit = useRef(false);
useEffect(() => {
  if (didInit.current) return;
  didInit.current = true;
  init();
}, []);

// previous value
function usePrevious(value) {
  const ref = useRef(undefined);
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}

// pass a ref into your own component (React 19+)
function Field({ ref }) { return <input ref={ref} />; }
```

| Question | Answer |
|---|---|
| What does `useRef` return? | An object `{ current: ... }`, the same one every render. |
| Does changing `.current` re-render? | No. Never. |
| When is a DOM ref filled in? | After commit, before your effects run. |
| Can I read `.current` during render? | No — it may be `null` or stale, and it is impure. |
| Do I need a dependency array? | No. Refs have nothing to react to. |
| Ref or state? | Screen must change → state. Otherwise → ref. |

---

## 10. Revision questions (with answers)

**Q1. What does `useRef(5)` actually give you?**
An object `{ current: 5 }`. React returns the exact same object on every render.

**Q2. Why does a plain `let id = null` not work for a timer id?**
It resets to `null` on the next render, so `clearInterval` gets nothing and the timer never stops.

**Q3. Why not store the timer id in state instead?**
It would work but would trigger a pointless re-render to store a value the user never sees.

**Q4. What is `inputRef.current` during the first render?**
`null`. React attaches the DOM node after the commit phase.

**Q5. Why does this show a stale number?**
```jsx
const c = useRef(0);
<button onClick={() => c.current++}>{c.current}</button>
```
The ref increases, but nothing tells React to re-render, so the JSX on screen is never rebuilt. It should be state.

**Q6. When exactly does React set a DOM ref?**
Between updating the real DOM and running your effects — which is why effects can safely use it.

**Q7. Why does the `usePrevious` pattern work?**
The effect that writes `ref.current = value` runs after render, so during the next render the ref still holds the previous value.

**Q8. Is it safe to write `ref.current = x` inside the render body?**
No. Rendering must be pure, and React may render more than once or discard a render.

**Q9. Give three good jobs for a ref.**
Holding a timer id, holding a DOM node to call `focus()` on, and remembering a previous prop value.

**Q10. How do you pass a ref into your own component in React 19?**
Accept `ref` as a normal prop and forward it to a DOM tag. `forwardRef` is no longer required.

---

## 11. What to learn next

- **`useContext`** — share a value with deeply nested components without passing props through every level.
- **Custom hooks** — `usePrevious` above is already one; the pattern generalises to any reusable logic.
- **`useReducer`** — for state with several fields that change together.
- **Chapter 07: performance** — where refs help you avoid re-renders that are genuinely unnecessary.

➡ Next note: `04_use_context.md`

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
