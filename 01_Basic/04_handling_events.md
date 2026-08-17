# 04. Handling Events

> An **event** is something the user does — a click, a key press, a form submit. **Handling an event** means giving React a function to run when that thing happens.

---

## 1. Real-life analogy

Think about a doorbell.

You do not stand at the door all day watching for visitors. You install a bell, and you connect it to a **response**: when the bell rings, the dog barks, or a light flashes, or you walk to the door.

- **The button on the wall** = the DOM element (`<button>`)
- **Pressing it** = the event (`click`)
- **What happens afterwards** = the event handler (your function)

Two details from the doorbell matter in React too.

First: you **connect** the bell to the response once. You do not press it yourself while installing it. Beginners often "press the bell" by accident — writing `onClick={handleClick()}` runs the function immediately instead of connecting it.

Second: the bell hands you information. A smart doorbell tells you *who* rang and *when*. In React, the handler receives an **event object** containing exactly that kind of detail — which element, which key, what value.

**In simple words:** you connect a function to an event once, and the browser calls it later when the user acts.

---

## 2. The problem — why does React handle events its own way?

In plain JavaScript, you attach events like this:

```js
// Plain JavaScript
const btn = document.getElementById("saveBtn");   // 1. find the element
btn.addEventListener("click", handleSave);        // 2. attach a listener

// later, when the element is removed:
btn.removeEventListener("click", handleSave);     // 3. clean up, or leak memory
```

Three steps, and the third one is the trap. Look at the problems:

| Problem | Why it hurts |
|---------|--------------|
| You must find the element first | Needs an `id` or a query; if the element does not exist yet, it silently fails |
| Attach and cleanup are separate | Forget the cleanup and you leak memory |
| The code lives far from the markup | The button is in the HTML, the listener is in a JS file — you must jump between them |
| Different browsers behaved differently | Old Internet Explorer used `attachEvent`, event objects had different property names |
| Thousands of listeners get slow | A list of 1000 rows with a button each = 1000 separate listeners |

React solves all five:

```jsx
// React — one step, and the connection sits right on the element
<button onClick={handleSave}>Save</button>
```

- No lookup: you attach it to the element as you write it.
- No cleanup: React removes the handler when the element goes away.
- No jumping: the behaviour sits next to the markup.
- No browser differences: React gives you one normalized event object.
- No performance cliff: React uses one shared listener at the root, not one per element.

**In simple words:** React turns three fragile steps into one attribute that cleans up after itself.

---

## 3. What an event handler actually is

An event handler is just a **function you pass as a prop**. Nothing more.

```jsx
function App() {
  // 1. Define a normal function
  function handleClick() {
    alert("Button was clicked!");
  }

  // 2. Pass it — no parentheses
  return <button onClick={handleClick}>Click me</button>;
}
```

Remember from the JSX note: `{}` means "switch to JavaScript". So `onClick={handleClick}` passes the **function value** itself. React stores it and calls it later.

### The single most common bug

```jsx
<button onClick={handleClick}>    {/* ✅ passes the function */}
<button onClick={handleClick()}>  {/* ❌ CALLS it right now, during render */}
```

Why is the second one so bad? `handleClick()` runs immediately while React is building the JSX, and its **return value** (usually `undefined`) becomes the `onClick` prop. So the alert fires the moment the page loads, and clicking does nothing.

Think of it as the difference between handing someone a recipe and cooking the meal on the spot.

```jsx
onClick={handleClick}       // "here is the recipe, cook it when needed"
onClick={handleClick()}     // "I cooked it now, here are the leftovers"
```

### The naming convention

| Thing | Convention | Example |
|-------|-----------|---------|
| The prop React gives you | `on` + EventName | `onClick`, `onChange`, `onSubmit` |
| Your function | `handle` + What | `handleClick`, `handleSubmit`, `handleNameChange` |
| A callback prop you define | `on` + Something | `onFollow`, `onDelete`, `onSelect` |

```jsx
<button onClick={handleDelete}>   // on... = { handle... }
```

This is not enforced by React, but every React codebase follows it, so follow it too.

### Three places to write the function

```jsx
function App() {
  // Style 1 — named function above the return (BEST for anything real)
  function handleClick() {
    console.log("clicked");
  }

  // Style 2 — arrow function stored in a variable
  const handleHover = () => console.log("hovered");

  return (
    <>
      <button onClick={handleClick}>A</button>
      <button onMouseEnter={handleHover}>B</button>

      {/* Style 3 — inline arrow (fine for one-liners and for passing arguments) */}
      <button onClick={() => console.log("inline")}>C</button>
    </>
  );
}
```

Use Style 1 when the logic is more than one line. Inline arrows make JSX hard to read fast.

**In simple words:** an event handler is a plain function passed by name, never called by you.

---

## 4. Syntax, step by step

### Step 1 — The event object

React passes one argument to every handler: the **event object**, conventionally named `e` (or `event`).

```jsx
function handleClick(e) {
  console.log(e.type);            // "click"
  console.log(e.target);          // the actual DOM element clicked
  console.log(e.target.tagName);  // "BUTTON"
  console.log(e.clientX, e.clientY);  // mouse position on screen
}

<button onClick={handleClick}>Click</button>
```

You do not pass `e` yourself. React passes it automatically.

Useful properties by event type:

| Event | Useful property | What it gives you |
|-------|-----------------|-------------------|
| `onClick` | `e.target` | The element that was clicked |
| `onChange` | `e.target.value` | The current text in an input |
| `onChange` (checkbox) | `e.target.checked` | `true` / `false` |
| `onChange` | `e.target.name` | The input's `name` attribute |
| `onKeyDown` | `e.key` | `"Enter"`, `"a"`, `"Escape"`, `"ArrowUp"` |
| `onSubmit` | `e.preventDefault()` | Stops the page from reloading |
| any | `e.type` | The event name as a string |

### Step 2 — Passing arguments to a handler

This is where beginners get stuck. You want to pass an id, but you cannot call the function.

```jsx
<button onClick={handleDelete(item.id)}>   {/* ❌ calls it during render */}
```

The fix is to wrap it in an arrow function. The arrow function is what gets passed; the real call happens inside it, later.

```jsx
<button onClick={() => handleDelete(item.id)}>Delete</button>   {/* ✅ */}
```

Read it as: *"when clicked, run this little function, which then calls `handleDelete(id)`."*

```
onClick={ () => handleDelete(id) }
          └──┬──┘  └──────┬──────┘
             │            └── runs LATER, on click
             └── this is what React stores now
```

You can pass the event too:

```jsx
<button onClick={(e) => handleDelete(item.id, e)}>Delete</button>
```

### Step 3 — `preventDefault()` — stop the browser's built-in behaviour

Some elements have default behaviour baked into the browser:

| Element | Default behaviour |
|---------|-------------------|
| `<form>` submit | Reloads the whole page |
| `<a href="...">` click | Navigates away |
| `<input type="checkbox">` | Toggles itself |
| Right click | Opens the context menu |

In a React app you almost always want to stop the form reload:

```jsx
function handleSubmit(e) {
  e.preventDefault();          // ⛔ stop the page reload
  console.log("Sending data to the server...");
}

<form onSubmit={handleSubmit}>
  <input type="text" />
  <button type="submit">Send</button>
</form>
```

> ⚠️ Without `e.preventDefault()`, submitting the form reloads the page, your app restarts from scratch, and all your data disappears. This is the number one "why did my app reset?" bug.

### Step 4 — `stopPropagation()` — stop the event from climbing

Events **bubble**: they fire on the element, then on its parent, then its grandparent, all the way to the root.

```jsx
function Card() {
  function handleCardClick() { console.log("card clicked"); }
  function handleButtonClick() { console.log("button clicked"); }

  return (
    <div onClick={handleCardClick}>            {/* parent */}
      <button onClick={handleButtonClick}>Delete</button>   {/* child */}
    </div>
  );
}
```

Clicking the button logs **both**:

```
button clicked      <- the child runs first
card clicked        <- then it bubbles up to the parent
```

```
   ┌─────────────── div (onClick) ───────────────┐
   │                    ▲                        │
   │                    │  2. bubbles up         │
   │            ┌───── button (onClick) ─────┐   │
   │            │   1. click happens here    │   │
   │            └────────────────────────────┘   │
   └─────────────────────────────────────────────┘
```

To stop that:

```jsx
function handleButtonClick(e) {
  e.stopPropagation();       // ⛔ the event stops here, parent never hears it
  console.log("button clicked");
}
```

> 💡 Bubbling is useful, not a bug. It is why you can put one `onClick` on a list container instead of on 500 rows.

### Step 5 — Common event props

```jsx
// Mouse
<div onClick={fn} onDoubleClick={fn} onMouseEnter={fn} onMouseLeave={fn}
     onMouseDown={fn} onMouseUp={fn} onContextMenu={fn} />

// Keyboard  (needs a focusable element)
<input onKeyDown={fn} onKeyUp={fn} />

// Form
<input onChange={fn} onFocus={fn} onBlur={fn} />
<form onSubmit={fn} onReset={fn} />
<select onChange={fn} />

// Clipboard / drag
<div onCopy={fn} onPaste={fn} onDragStart={fn} onDrop={fn} />

// Media / image
<img onLoad={fn} onError={fn} />
```

> ⚠️ `onChange` in React is not the same as HTML's `change` event. HTML fires `change` only when the input loses focus. React fires `onChange` on **every keystroke** — it behaves like the native `input` event. This is what you actually want.

### Step 6 — Handling keyboard keys

```jsx
function handleKeyDown(e) {
  if (e.key === "Enter") {
    console.log("Enter pressed — submit!");
  }
  if (e.key === "Escape") {
    console.log("Escape pressed — close!");
  }
  if (e.key === "s" && e.ctrlKey) {    // Ctrl + S
    e.preventDefault();                 // stop the browser's Save dialog
    console.log("Saving...");
  }
}

<input onKeyDown={handleKeyDown} placeholder="Press Enter" />
```

Modifier flags available on the event: `e.ctrlKey`, `e.shiftKey`, `e.altKey`, `e.metaKey` (Cmd on Mac).

**In simple words:** pass the function, read what you need off `e`, and call `preventDefault` when the browser would otherwise do something you do not want.

---

## 5. Full working example

A small "add task" form. It uses clicks, typing, key presses, form submit, and passing arguments — all without state, so you can read it before learning `useState`.

```jsx
// src/App.jsx
// A task list demo focused purely on events.
// NOTE: the list does not update on screen yet — that needs useState (next chapter).
//       Watch the browser console to see every handler fire.

// Starting data, defined outside the component
const tasks = [
  { id: 1, text: "Learn JSX", done: true },
  { id: 2, text: "Learn components", done: true },
  { id: 3, text: "Learn events", done: false },
];

function App() {
  // ── 1. Simple click, no arguments ─────────────────────────
  function handleClear() {
    console.log("Clear all clicked");
  }

  // ── 2. Click that needs an argument ───────────────────────
  function handleDelete(id, e) {
    e.stopPropagation();                  // don't let the row's onClick also fire
    console.log("Deleting task id:", id);
  }

  // ── 3. Reading a value while typing ───────────────────────
  function handleTextChange(e) {
    console.log("Current text:", e.target.value);   // value of the input
  }

  // ── 4. Reading a checkbox ─────────────────────────────────
  function handleToggle(id, e) {
    e.stopPropagation();
    console.log(`Task ${id} is now`, e.target.checked ? "done" : "not done");
  }

  // ── 5. Keyboard ───────────────────────────────────────────
  function handleKeyDown(e) {
    if (e.key === "Enter") {
      console.log("Enter pressed — would add the task");
    }
    if (e.key === "Escape") {
      e.target.value = "";                // clear the box
      console.log("Escape pressed — cleared");
    }
  }

  // ── 6. Form submit ────────────────────────────────────────
  function handleSubmit(e) {
    e.preventDefault();                   // ⛔ THE important line — no page reload

    // FormData reads every named input in the form at once
    const form = new FormData(e.target);
    const text = form.get("taskText");    // matches name="taskText"

    if (!text.trim()) {                   // trim removes spaces at both ends
      console.log("Empty task — ignored");
      return;                             // stop early
    }
    console.log("Submitting new task:", text);
    e.target.reset();                     // clear the form
  }

  // ── 7. Row click, to show bubbling ────────────────────────
  function handleRowClick(text) {
    console.log("Row clicked:", text);
  }

  // ── 8. Focus and blur ─────────────────────────────────────
  function handleFocus() { console.log("Input focused"); }
  function handleBlur()  { console.log("Input left"); }

  return (
    <div style={{ fontFamily: "sans-serif", padding: "20px", maxWidth: "420px" }}>
      <h1>My Tasks</h1>

      {/* onSubmit fires for both the button click AND the Enter key */}
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          name="taskText"                 // FormData uses this name
          placeholder="What needs doing?"
          onChange={handleTextChange}     // fires on every keystroke
          onKeyDown={handleKeyDown}       // fires for Enter, Escape, etc.
          onFocus={handleFocus}
          onBlur={handleBlur}
          style={{ padding: "8px", width: "260px" }}
        />
        {/* type="submit" makes this button trigger the form's onSubmit */}
        <button type="submit" style={{ padding: "8px 12px" }}>Add</button>
      </form>

      <ul style={{ listStyle: "none", padding: 0 }}>
        {tasks.map((task) => (
          <li
            key={task.id}
            onClick={() => handleRowClick(task.text)}   // arrow → passes an argument
            style={{
              display: "flex",
              alignItems: "center",
              gap: "8px",
              padding: "8px",
              borderBottom: "1px solid #eee",
              cursor: "pointer",
            }}
          >
            <input
              type="checkbox"
              defaultChecked={task.done}                // uncontrolled, for now
              onChange={(e) => handleToggle(task.id, e)}
            />
            <span style={{ textDecoration: task.done ? "line-through" : "none" }}>
              {task.text}
            </span>
            {/* pass BOTH the id and the event */}
            <button onClick={(e) => handleDelete(task.id, e)}>✕</button>
          </li>
        ))}
      </ul>

      {/* type="button" so it does NOT submit the form */}
      <button type="button" onClick={handleClear}>Clear all</button>
    </div>
  );
}

export default App;
```

Open the console and try it: type, press Enter, click a row, click the ✕ (notice the row click does **not** fire, thanks to `stopPropagation`), and submit the form (notice the page does **not** reload).

---

## 6. How it works behind the scenes

### React does not attach a listener to your button

This surprises people. When you write `<button onClick={fn}>`, React does **not** call `button.addEventListener("click", fn)`.

Instead, React attaches **one** listener per event type to the root container of your app. When any click happens anywhere, that single listener catches it, figures out which component it came from, and calls the right handler.

```
        ┌──────────────────────────────────────────────┐
        │  #root  ← React attaches ONE "click"         │
        │            listener here                     │
        │  ┌────────────────────────────────────────┐  │
        │  │  <div>                                 │  │
        │  │    ┌──────────────────────────────┐    │  │
        │  │    │  <button>  ← NO listener     │    │  │
        │  │    └──────────────────────────────┘    │  │
        │  └────────────────────────────────────────┘  │
        └──────────────────────────────────────────────┘

  1. User clicks the button
  2. The native click bubbles up the real DOM to #root
  3. React's single listener catches it
  4. React walks its own component tree to find matching handlers
  5. React calls them in order: child first, then parents (simulated bubbling)
```

This is called **event delegation**. Why bother?

| Benefit | Explanation |
|---------|-------------|
| Memory | A 1000-row table costs 1 listener, not 1000 |
| Speed | Adding a row does not mean attaching new listeners |
| Cleanup | Removing an element cannot leave a dangling listener |
| Consistency | React controls the order and timing of every handler |

### The SyntheticEvent

The `e` you receive is not the browser's raw event. It is a **SyntheticEvent** — React's own wrapper around it.

```
   Native browser event  (differs slightly between Chrome, Firefox, Safari)
              │
              ▼
   React wraps it → SyntheticEvent  (identical everywhere)
              │
              ▼
   Your handler receives `e`
              │
              └─→ e.nativeEvent  gives you the original, if you ever need it
```

The wrapper has the same API you know — `target`, `preventDefault()`, `stopPropagation()`, `key` — but React guarantees the behaviour is the same in every browser. That used to matter enormously; today it mostly saves you from small edge cases.

```jsx
function handleClick(e) {
  console.log(e);               // SyntheticBaseEvent {...}
  console.log(e.nativeEvent);   // PointerEvent {...} — the real browser event
}
```

### Handlers run before the re-render

When you learn `useState` next, this ordering will matter:

```
  user clicks
      │
      ▼
  your handler function runs completely
      │
      ▼
  React processes any state updates it queued
      │
      ▼
  React re-renders the affected components
      │
      ▼
  React updates the real DOM
```

Your handler finishes **first**. Nothing on screen changes while it is still running.

**In simple words:** one listener at the root, a normalized event object, and your handler runs to completion before anything redraws.

---

## 7. Comparison table

### React events vs plain DOM events

| | **Plain JavaScript** | **React** |
|---|---|---|
| Attaching | `el.addEventListener("click", fn)` | `<button onClick={fn}>` |
| Naming | lowercase: `click`, `mouseenter` | camelCase: `onClick`, `onMouseEnter` |
| Value passed | a function | a function in `{}` |
| Where listeners live | on each element | one per type at the root |
| Cleanup | you must call `removeEventListener` | automatic |
| Event object | native, varies by browser | SyntheticEvent, identical everywhere |
| Text input event | `input` fires per keystroke, `change` on blur | `onChange` fires per keystroke |
| Stop default | `e.preventDefault()` or `return false` | `e.preventDefault()` only |

> ⚠️ `return false` from a handler does nothing in React. Always use `e.preventDefault()`.

### Which form of handler to write

| Form | Example | Use when |
|------|---------|----------|
| Direct reference | `onClick={handleSave}` | No arguments needed — the default |
| Inline arrow | `onClick={() => handleDelete(id)}` | You need to pass an argument |
| Inline arrow with event | `onClick={(e) => handleX(id, e)}` | You need both an argument and `e` |
| Inline body | `onClick={() => console.log("hi")}` | One trivial line only |
| Called directly | `onClick={handleSave()}` | ❌ Never — this is the bug |

**In simple words:** React's version is shorter, safer, and behaves the same everywhere.

---

## 8. Common mistakes beginners make

**1. Calling the function instead of passing it**

```jsx
<button onClick={handleSave()}>      {/* ❌ runs at render, click does nothing */}
<button onClick={handleSave}>        {/* ✅ */}
```

**2. Forgetting `e.preventDefault()` on a form**

```jsx
function handleSubmit(e) {
  console.log("saving");             // ❌ page reloads, app resets, log disappears
}
function handleSubmit(e) {
  e.preventDefault();                // ✅
  console.log("saving");
}
```

**3. Lowercase event names**

```jsx
<button onclick={fn}>     {/* ❌ React warns; nothing happens */}
<button onClick={fn}>     {/* ✅ camelCase */}
```

**4. Using `return false` to stop default behaviour**

```jsx
function handleSubmit(e) { return false; }   // ❌ works in jQuery, not in React
function handleSubmit(e) { e.preventDefault(); }  // ✅
```

**5. Wrapping when you do not need to**

```jsx
<button onClick={() => handleSave()}>   {/* works, but pointless wrapper */}
<button onClick={handleSave}>           {/* ✅ cleaner */}
```

**6. Expecting a button inside a form not to submit**

```jsx
<form onSubmit={save}>
  <button onClick={cancel}>Cancel</button>            {/* ❌ also submits the form! */}
  <button type="button" onClick={cancel}>Cancel</button>  {/* ✅ */}
</form>
```

A `<button>` inside a `<form>` defaults to `type="submit"`. Always set `type="button"` on buttons that are not meant to submit.

**7. Reading `e.target.value` on the wrong element**

```jsx
// e.target is whatever was actually clicked — maybe an icon inside the button
function handleClick(e) {
  console.log(e.target);          // could be the <span> inside the button
  console.log(e.currentTarget);   // ✅ always the element the handler is on
}
```

`e.target` = what was clicked. `e.currentTarget` = what the handler is attached to.

**8. Forgetting that a child click bubbles to the parent**

```jsx
<div onClick={openCard}>
  <button onClick={deleteCard}>✕</button>   {/* ❌ deletes AND opens */}
</div>
```

Add `e.stopPropagation()` in `deleteCard`.

**9. Attaching keyboard events to a non-focusable element**

```jsx
<div onKeyDown={fn}>            {/* ❌ a div cannot be focused, so it never fires */}
<div tabIndex={0} onKeyDown={fn}>  {/* ✅ tabIndex makes it focusable */}
<input onKeyDown={fn} />           {/* ✅ inputs are focusable already */}
```

**10. Trying to change props from inside a handler**

```jsx
function Child({ count }) {
  function handleClick() {
    count = count + 1;      // ❌ props are read-only; nothing re-renders
  }
}
```

Ask the parent for a callback prop instead — or use `useState`, which is next.

**11. Using `onChange` on a `<div>`**

Form events only work on form elements: `input`, `textarea`, `select`, `form`. On a `div` they simply never fire.

---

## 9. Cheat sheet

```jsx
// ── PASS THE HANDLER ────────────────────────────────────────
onClick={handleClick}                    // ✅ no arguments
onClick={() => handleClick(id)}          // ✅ with an argument
onClick={(e) => handleClick(id, e)}      // ✅ argument + event
onClick={handleClick()}                  // ❌ NEVER — calls it now

// ── THE EVENT OBJECT ────────────────────────────────────────
e.target                  // the element that triggered it
e.currentTarget           // the element the handler sits on
e.target.value            // text in an input
e.target.checked          // true/false for a checkbox
e.target.name             // the input's name attribute
e.key                     // "Enter" | "Escape" | "a" | "ArrowUp"
e.ctrlKey e.shiftKey      // modifier keys held down
e.preventDefault()        // stop the browser default (form reload, link nav)
e.stopPropagation()       // stop the event bubbling to parents
e.nativeEvent             // the raw browser event

// ── MOST-USED EVENT PROPS ───────────────────────────────────
onClick  onDoubleClick  onMouseEnter  onMouseLeave  onContextMenu
onChange  onInput  onSubmit  onReset  onFocus  onBlur
onKeyDown  onKeyUp
onCopy  onPaste  onDragStart  onDrop
onLoad  onError

// ── FORM PATTERN ────────────────────────────────────────────
function handleSubmit(e) {
  e.preventDefault();                      // always first
  const data = new FormData(e.target);     // read all named inputs
  console.log(data.get("email"));
  e.target.reset();                        // clear the form
}
<form onSubmit={handleSubmit}>
  <input name="email" />
  <button type="submit">Send</button>
  <button type="button" onClick={cancel}>Cancel</button>
</form>

// ── STOP A CHILD CLICK FROM REACHING THE PARENT ─────────────
function handleDelete(e) { e.stopPropagation(); /* ... */ }

// ── KEYBOARD ────────────────────────────────────────────────
function handleKeyDown(e) {
  if (e.key === "Enter") submit();
  if (e.key === "Escape") close();
}
```

| Question | Answer |
|----------|--------|
| Why `onClick={fn}` and not `onClick={fn()}`? | Braces pass the function; parentheses call it immediately |
| How do I pass an id to a handler? | `onClick={() => handleX(id)}` |
| Why does my form reload the page? | Missing `e.preventDefault()` in `onSubmit` |
| Why does clicking a child also fire the parent? | Events bubble — use `e.stopPropagation()` |
| `e.target` vs `e.currentTarget`? | What was clicked vs. what the handler is on |
| Does React attach a listener per element? | No — one per event type at the root |

---

## 10. Revision questions

**Q1. What is the difference between `onClick={handleClick}` and `onClick={handleClick()}`?**
The first passes the function so React can call it on click. The second calls it immediately during render and passes its return value (usually `undefined`), so nothing happens on click.

**Q2. How do you pass an argument to an event handler?**
Wrap it in an arrow function: `onClick={() => handleDelete(id)}`. The arrow function is what gets stored; the real call happens inside it when the click occurs.

**Q3. What does `e.preventDefault()` do, and where do you need it most?**
It stops the browser's built-in behaviour for that event. You need it most in `onSubmit`, where the default is a full page reload that would reset your entire app.

**Q4. What is event bubbling?**
After firing on an element, an event travels up through every ancestor, firing their handlers too. Clicking a button inside a `div` fires the button's handler first, then the `div`'s.

**Q5. How do you stop bubbling?**
Call `e.stopPropagation()` inside the child's handler.

**Q6. What is a SyntheticEvent?**
React's wrapper around the native browser event. It has the same API (`target`, `preventDefault`, `key`) but behaves identically in every browser. The original is available at `e.nativeEvent`.

**Q7. Does React attach a listener to every element with an `onClick`?**
No. React attaches one listener per event type at the app's root container and works out which handlers to call. This is called event delegation.

**Q8. What is the difference between `e.target` and `e.currentTarget`?**
`e.target` is the deepest element the user actually interacted with (possibly a child). `e.currentTarget` is the element the handler is attached to.

**Q9. Why does a `<button>` inside a `<form>` submit the form even without `onSubmit` on it?**
Because a button's default `type` is `"submit"`. Set `type="button"` on any button that should not submit.

**Q10. Why does `onKeyDown` on a `<div>` never fire?**
Keyboard events only fire on focusable elements. Add `tabIndex={0}` to make a `div` focusable, or attach the handler to an `input`.

**Q11. How is React's `onChange` different from HTML's `change` event?**
HTML fires `change` only when the input loses focus. React's `onChange` fires on every keystroke, matching the native `input` event.

**Q12. Why does `return false` not stop the default behaviour in React?**
React ignores the handler's return value entirely. Only `e.preventDefault()` works.

---

## 11. What to learn next

You can now respond to what the user does. But so far, nothing on the screen actually changes — the handlers only log to the console. The missing piece is **memory**.

- **05 — Conditional rendering and lists** — showing and hiding parts of the UI, `map`, and why keys matter
- **02_Hooks / 01 — useState** — giving a component memory, so a click can actually update the screen
- **06_Forms / 01 — Controlled inputs** — combining `useState` with `onChange` to fully own an input's value

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
