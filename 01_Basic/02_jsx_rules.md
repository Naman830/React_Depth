# 02. JSX Rules in Depth

> JSX is a special syntax that lets you write HTML-looking code inside a JavaScript file. It is not HTML and it is not JavaScript — it is a shortcut that a build tool converts into normal JavaScript function calls before the browser ever sees it.

---

## 1. Real-life analogy

Think about sending a message to a friend in another country who only reads English.

You want to write in your own comfortable shorthand:

```
c u @ 5 :)
```

Your friend cannot read that. So a **translator** sits in the middle. You write shorthand, the translator turns it into proper English, and your friend reads clean English.

- **You** = the developer
- **Shorthand** = JSX
- **Translator** = Babel (built into Vite)
- **Friend** = the browser

The translator is strict. It knows shorthand, but only *its own* version of shorthand. If you invent your own rule, the translator stops and complains. That is exactly why JSX has rules — the translator must be able to understand every single character you write.

**In simple words:** JSX is a convenience for humans, and the rules exist so the machine can translate it without guessing.

---

## 2. The problem — why does JSX exist?

Before React, building UI with plain JavaScript looked like this:

```js
// Plain JavaScript way — no JSX
const heading = document.createElement("h1");   // make an element
heading.className = "title";                     // set a class
heading.textContent = "Hello Naman";             // set the text

const button = document.createElement("button");
button.textContent = "Click me";

const box = document.createElement("div");       // make a container
box.appendChild(heading);                        // put heading inside
box.appendChild(button);                         // put button inside

document.getElementById("root").appendChild(box); // finally show it
```

That is 9 lines to draw a heading and a button. Now imagine a whole page.

Problems with this approach:

| Problem | Why it hurts |
|---------|--------------|
| Very long | Every element needs 2–4 lines |
| Hard to read | You cannot *see* the shape of the UI |
| Hard to nest | Deep nesting means many `appendChild` calls |
| Easy to break | Forget one `appendChild` and the element silently disappears |

The same thing in JSX:

```jsx
// JSX way — you can see the shape immediately
<div>
  <h1 className="title">Hello Naman</h1>
  <button>Click me</button>
</div>
```

You instantly *see* that a `div` holds an `h1` and a `button`. The structure is visible.

**In simple words:** JSX exists so your code looks like the UI it produces.

---

## 3. What JSX actually is

JSX stands for **JavaScript XML**. XML is a family of tag-based languages — HTML is one member of that family.

Here is the important truth: **JSX is not sent to the browser.** Before your code runs, a tool called **Babel** rewrites every JSX tag into a normal function call.

```jsx
// What you write
const element = <h1 className="title">Hello</h1>;
```

```js
// What Babel turns it into (React 17 and older style)
const element = React.createElement("h1", { className: "title" }, "Hello");
```

```js
// What modern React (17+) turns it into — the "automatic runtime"
import { jsx as _jsx } from "react/jsx-runtime";
const element = _jsx("h1", { className: "title", children: "Hello" });
```

Both versions do the same job. They call a function with three pieces of information:

1. **What tag?** — `"h1"`
2. **What attributes?** — `{ className: "title" }`
3. **What is inside?** — `"Hello"`

That function returns a plain JavaScript object. Let's look at it:

```js
// Roughly what a React element looks like in memory
{
  type: "h1",
  props: {
    className: "title",
    children: "Hello"
  },
  key: null,
  ref: null
}
```

This object is called a **React element**. It is just a description — a recipe card. It is *not* a real DOM node yet. React reads these descriptions and then creates or updates the real DOM.

> 💡 Because JSX becomes a function call that returns an object, JSX is an **expression**. You can store it in a variable, put it in an array, return it from a function, or pass it as an argument — anything you can do with a normal value.

```jsx
const box = <div>Hi</div>;              // store in a variable ✅
const list = [<li>A</li>, <li>B</li>];  // put in an array ✅
function get() { return <p>Yo</p>; }    // return from a function ✅
```

**In simple words:** JSX is sugar. Underneath, it is a function call that returns a plain object.

---

## 4. The rules, one by one

These are the rules Babel enforces. Break one and you get an error.

### Rule 1 — Return exactly one root element

A function can only return **one** value. So JSX must have **one** outer wrapper.

```jsx
// ❌ WRONG — two elements side by side, no parent
function App() {
  return (
    <h1>Title</h1>
    <p>Text</p>
  );
}
```

```jsx
// ✅ RIGHT — wrapped in one div
function App() {
  return (
    <div>
      <h1>Title</h1>
      <p>Text</p>
    </div>
  );
}
```

But sometimes you do **not** want an extra `<div>` in the page — it can break CSS layouts like flexbox and grid. For that, React gives you a **Fragment**: an invisible wrapper that groups children but renders nothing to the DOM.

```jsx
// ✅ BEST — Fragment groups without adding a real element
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Text</p>
    </>
  );
}
```

`<>...</>` is the short form. The long form is `<React.Fragment>...</React.Fragment>`. You need the long form only when the fragment needs a `key` (common when mapping over a list).

### Rule 2 — Every tag must be closed

HTML forgives you. JSX does not.

```jsx
<br>          {/* ❌ error */}
<br />        {/* ✅ self-closing */}

<img src="a.png">        {/* ❌ error */}
<img src="a.png" />      {/* ✅ */}

<input type="text">      {/* ❌ error */}
<input type="text" />    {/* ✅ */}

<div>Hello               {/* ❌ never closed */}
<div>Hello</div>         {/* ✅ */}
```

Tags with no children — `br`, `img`, `input`, `hr`, `meta`, `link` — must end with `/>`.

### Rule 3 — Attribute names use camelCase

JSX attributes become **JavaScript object keys**, and JavaScript uses camelCase. Also, some HTML attribute names are reserved words in JavaScript, so React renamed them.

| HTML | JSX | Why |
|------|-----|-----|
| `class` | `className` | `class` is a reserved JS keyword |
| `for` | `htmlFor` | `for` is a reserved JS keyword (loops) |
| `onclick` | `onClick` | camelCase convention |
| `tabindex` | `tabIndex` | camelCase convention |
| `maxlength` | `maxLength` | camelCase convention |
| `readonly` | `readOnly` | camelCase convention |
| `colspan` | `colSpan` | camelCase convention |
| `stroke-width` | `strokeWidth` | camelCase convention (SVG) |

```jsx
<label htmlFor="email" className="lbl">Email</label>
<input id="email" type="email" maxLength={40} readOnly />
```

The exceptions that stay lowercase: `data-*` and `aria-*` attributes.

```jsx
<div data-user-id="42" aria-label="Close button" />   {/* ✅ dashes are fine here */}
```

### Rule 4 — Use `{ }` to put JavaScript inside JSX

Curly braces are an escape hatch. They say: *"stop reading this as markup, start reading it as JavaScript."*

```jsx
const name = "Naman";
const age = 21;

<h1>Hello {name}, you are {age} years old</h1>
<h1>Next year you will be {age + 1}</h1>
<h1>{name.toUpperCase()}</h1>
<img src={`/avatars/${name}.png`} alt={name} />
```

Two very important limits:

**(a) Only expressions, not statements.** An expression produces a value. A statement performs an action.

```jsx
{ 2 + 2 }                          {/* ✅ expression → 4 */}
{ user.name }                      {/* ✅ expression */}
{ isLoggedIn ? "Bye" : "Login" }   {/* ✅ ternary is an expression */}
{ items.map(i => <li>{i}</li>) }   {/* ✅ map returns a value */}

{ if (x) { ... } }                 {/* ❌ if is a statement */}
{ for (let i...) { ... } }         {/* ❌ for is a statement */}
{ const y = 5; }                   {/* ❌ declaration is a statement */}
```

If you need an `if`, put it **above** the `return`:

```jsx
function Greeting({ user }) {
  let message;                       // decide first, outside JSX
  if (user) {
    message = `Welcome ${user.name}`;
  } else {
    message = "Please log in";
  }
  return <h1>{message}</h1>;         // then just use the value
}
```

**(b) Braces are also used for object values.** When an attribute takes an object, you get double braces — the outer `{}` is "enter JavaScript", the inner `{}` is the object itself.

```jsx
{/* outer = JS mode, inner = an object */}
<div style={{ color: "red", fontSize: "20px" }}>Warning</div>
```

Note that CSS property names are camelCase too: `font-size` → `fontSize`, `background-color` → `backgroundColor`.

### Rule 5 — Component names must start with a Capital letter

This is how Babel decides between a **DOM tag** and **your component**.

```jsx
<div />       →  createElement("div")    // lowercase → string → real HTML tag
<Profile />   →  createElement(Profile)  // Capital → variable → your component
```

```jsx
function profile() { return <p>Hi</p>; }

<profile />   {/* ❌ React looks for an HTML tag named "profile" — renders nothing */}
<Profile />   {/* ✅ React calls your function */}
```

> ⚠️ This bug is silent. No error appears in the console. The screen is just empty. Always capitalize component names.

### Rule 6 — Comments inside JSX need braces

```jsx
<div>
  {/* This is a JSX comment */}
  <p>Hello</p>

  // ❌ This is NOT a comment — it prints literally on the screen
</div>
```

Outside JSX (in normal JS parts), regular `//` and `/* */` comments work fine.

### Rule 7 — What JSX renders and what it skips

React renders strings and numbers. It **ignores** `null`, `undefined`, `true`, and `false`.

```jsx
<div>{ "text" }</div>       {/* → text */}
<div>{ 42 }</div>           {/* → 42 */}
<div>{ null }</div>         {/* → nothing */}
<div>{ undefined }</div>    {/* → nothing */}
<div>{ true }</div>         {/* → nothing */}
<div>{ [1, 2, 3] }</div>    {/* → 123  (arrays render each item) */}
<div>{ {a: 1} }</div>       {/* ❌ ERROR: Objects are not valid as a React child */}
```

That "ignore false" behaviour is what makes this popular shortcut work:

```jsx
{ isLoggedIn && <Dashboard /> }   // if false → renders nothing
```

> ⚠️ Be careful with numbers. `{count && <p>Items</p>}` prints a literal `0` on screen when `count` is `0`, because `0` is a number, not `false`. Use `{count > 0 && <p>Items</p>}` instead.

### Rule 8 — Lists need a `key`

When you build elements from an array, each one needs a unique `key` prop so React can track it across re-renders.

```jsx
const users = [
  { id: 1, name: "Amit" },
  { id: 2, name: "Sara" },
];

<ul>
  {users.map((user) => (
    <li key={user.id}>{user.name}</li>   // key must be unique among siblings
  ))}
</ul>
```

Use a stable id from your data. Avoid the array index as a key when the list can be reordered, filtered, or have items inserted — React will match the wrong elements and your UI can show stale values.

**In simple words:** close every tag, wrap in one parent, camelCase the attributes, and use `{}` when you need JavaScript.

---

## 5. Full working example

```jsx
// src/App.jsx — a small profile card that uses every JSX rule above

// A plain JavaScript object holding our data
const user = {
  id: 7,
  name: "Naman",
  role: "React learner",
  isOnline: true,
  skills: ["HTML", "CSS", "JavaScript"],
  avatar: "https://placehold.co/80",
};

// A style object. Note camelCase keys — fontSize, not font-size.
const cardStyle = {
  border: "1px solid #ddd",
  borderRadius: "8px",
  padding: "16px",
  width: "260px",
  fontFamily: "sans-serif",
};

// Component name starts with a Capital letter — Rule 5
function App() {
  // Any real logic goes ABOVE the return, in plain JavaScript — Rule 4a
  const skillCount = user.skills.length;
  const greeting = skillCount > 2 ? "Nice list!" : "Keep learning!";

  return (
    // Fragment groups everything without adding a real <div> — Rule 1
    <>
      {/* This is how you comment inside JSX — Rule 6 */}
      <h1>Profile</h1>

      {/* Double braces: outer = JS mode, inner = the style object — Rule 4b */}
      <div style={cardStyle}>
        {/* Self-closing tag, camelCase attributes — Rules 2 and 3 */}
        <img src={user.avatar} alt={user.name} width={80} height={80} />

        {/* className, not class — Rule 3 */}
        <h2 className="user-name">{user.name}</h2>

        {/* Template literal inside braces — Rule 4 */}
        <p>{`Role: ${user.role}`}</p>

        {/* && renders the element only when the left side is true — Rule 7 */}
        {user.isOnline && <span style={{ color: "green" }}>● Online</span>}

        {/* Ternary works where if/else cannot — Rule 4a */}
        <p>{skillCount > 0 ? greeting : "No skills added yet"}</p>

        <h3>Skills ({skillCount})</h3>
        <ul>
          {/* map returns an array of elements; each needs a key — Rule 8 */}
          {user.skills.map((skill) => (
            <li key={skill}>{skill}</li>
          ))}
        </ul>

        {/* htmlFor, not for — Rule 3 */}
        <label htmlFor="note">Note</label>
        <input id="note" type="text" maxLength={50} placeholder="Type here" />
      </div>
    </>
  );
}

export default App;
```

Copy this into `src/App.jsx` of a fresh Vite React app and run `npm run dev`. It renders a working card.

---

## 6. How it works behind the scenes

Here is the full journey of one line of JSX.

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 1 — You write JSX in App.jsx                            │
│                                                              │
│   <h1 className="title">Hello</h1>                           │
└───────────────────────────┬──────────────────────────────────┘
                            │  npm run dev / npm run build
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ STEP 2 — Babel (inside Vite) transpiles it                   │
│                                                              │
│   _jsx("h1", { className: "title", children: "Hello" })      │
│                                                              │
│   Now it is 100% valid JavaScript. No tags left.             │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ STEP 3 — The function runs and returns a React element       │
│           (a plain object — the "virtual DOM" node)          │
│                                                              │
│   { type: "h1",                                              │
│     props: { className: "title", children: "Hello" },        │
│     key: null, ref: null }                                   │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ STEP 4 — React compares this object with the previous one    │
│           (this comparison is called reconciliation)         │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ STEP 5 — ReactDOM applies only the differences to the browser│
│                                                              │
│   document.createElement("h1")                               │
│   el.className = "title"                                     │
│   el.textContent = "Hello"                                   │
└──────────────────────────────────────────────────────────────┘
```

Two consequences worth remembering:

1. **JSX errors are caught before the app runs.** If you forget a closing tag, the build fails at Step 2. The browser never even loads.
2. **The object in Step 3 is cheap.** Creating thousands of these plain objects is fast. Touching the real DOM is slow. That gap is the whole reason React can be fast.

You can prove the transpilation yourself:

```bash
# Paste JSX into https://babeljs.io/repl and watch the output change live
# Or install Babel locally and compile a file:
npx babel --presets @babel/preset-react src/App.jsx
```

**In simple words:** JSX → function call → plain object → React compares → browser updated.

---

## 7. Comparison with alternatives

You are not forced to use JSX. Here is what else exists.

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| **JSX** | `<h1 className="t">Hi</h1>` | Reads like the UI; caught at build time; huge tooling support | Needs a build step |
| **`React.createElement`** | `React.createElement("h1", {className:"t"}, "Hi")` | No build step needed | Unreadable once nested 3 levels deep |
| **`htm` library** | `` html`<h1 class="t">Hi</h1>` `` | Uses tagged template literals, no build step | Errors appear only at runtime; smaller community |
| **Template strings + innerHTML** | `` el.innerHTML = `<h1>Hi</h1>` `` | Dead simple | XSS security risk; no diffing; loses all state |
| **Vue/Angular templates** | `<h1 v-bind:class="t">Hi</h1>` | Nice separation of markup and logic | Custom template language to learn; less plain JavaScript |

The same nested UI written both ways shows why JSX won:

```jsx
// JSX — you can read the shape at a glance
<div className="card">
  <h2>Title</h2>
  <ul>
    <li>One</li>
    <li>Two</li>
  </ul>
</div>
```

```js
// createElement — same output, unreadable
React.createElement("div", { className: "card" },
  React.createElement("h2", null, "Title"),
  React.createElement("ul", null,
    React.createElement("li", null, "One"),
    React.createElement("li", null, "Two")
  )
);
```

**In simple words:** JSX is optional in theory and essential in practice.

---

## 8. Common mistakes beginners make

**1. Using `class` instead of `className`**

```jsx
<div class="box" />        {/* ❌ React warns; styles may not apply */}
<div className="box" />    {/* ✅ */}
```

**2. Lowercase component name**

```jsx
<myButton />   {/* ❌ silent — treated as an unknown HTML tag, renders nothing */}
<MyButton />   {/* ✅ */}
```

**3. Forgetting the parentheses after `return`**

JavaScript automatically inserts a semicolon after a lone `return`. Your JSX is then dead code.

```jsx
return                  // ❌ JS turns this into `return;` → returns undefined
  <div>Hello</div>;

return (                // ✅ open the parenthesis on the SAME line as return
  <div>Hello</div>
);
```

**4. Writing `if` inside braces**

```jsx
<p>{ if (ok) "Yes" }</p>            {/* ❌ if is a statement */}
<p>{ ok ? "Yes" : "No" }</p>        {/* ✅ ternary is an expression */}
```

**5. Rendering an object directly**

```jsx
const user = { name: "Sara" };
<p>{user}</p>            {/* ❌ "Objects are not valid as a React child" */}
<p>{user.name}</p>       {/* ✅ pick a string out of it */}
<p>{JSON.stringify(user)}</p>  {/* ✅ debugging trick */}
```

**6. Passing a string where a number is expected**

```jsx
<input maxLength="10" />    {/* works, but it is the string "10" */}
<input maxLength={10} />    {/* ✅ real number */}
```

**7. Calling the handler instead of passing it**

```jsx
<button onClick={handleClick()} />   {/* ❌ runs immediately on render */}
<button onClick={handleClick} />     {/* ✅ passes the function itself */}
<button onClick={() => handleClick(id)} />  {/* ✅ when you need arguments */}
```

**8. Single braces for an inline style**

```jsx
<div style={{ color: "red" }} />        {/* ✅ object inside braces */}
<div style={{ color: "red" }} />        {/* keys are camelCase: backgroundColor */}
<div style="color: red" />              {/* ❌ React expects an object, not a string */}
```

**9. Missing `key` in a list**

```jsx
{items.map(i => <li>{i}</li>)}            {/* ❌ warning + subtle UI bugs */}
{items.map(i => <li key={i.id}>{i.name}</li>)}  {/* ✅ */}
```

**10. Using `0` with `&&`**

```jsx
{cart.length && <Cart />}       {/* ❌ shows a literal 0 when the cart is empty */}
{cart.length > 0 && <Cart />}   {/* ✅ */}
```

---

## 9. Cheat sheet

```jsx
// ── STRUCTURE ───────────────────────────────────────────────
<>...</>                       // Fragment — one root, no extra DOM node
<div>...</div>                 // normal wrapper
<br /> <img /> <input />       // self-close empty tags

// ── ATTRIBUTES ──────────────────────────────────────────────
className="box"                // NOT class
htmlFor="id"                   // NOT for
onClick={fn}                   // camelCase events
tabIndex={0} maxLength={10}    // camelCase, numbers in braces
data-id="1" aria-label="Close" // dashes are allowed here

// ── JAVASCRIPT INSIDE ───────────────────────────────────────
{value}                        // print a variable
{a + b}                        // any expression
{fn(x)}                        // function call
{`Hi ${name}`}                 // template literal
{cond ? <A /> : <B />}         // if/else
{cond && <A />}                // if with no else
{list.map(i => <li key={i.id}>{i.t}</li>)}   // loops
{/* comment */}                // comment

// ── STYLES ──────────────────────────────────────────────────
style={{ fontSize: "20px", backgroundColor: "red" }}

// ── RENDERS / SKIPS ─────────────────────────────────────────
"text"  42  [array]            // rendered
null  undefined  true  false   // skipped silently
{ key: 1 }                     // ERROR — not a valid child
```

| Question | Answer |
|----------|--------|
| Can a component return two sibling tags? | No — wrap in `<>...</>` |
| Is JSX HTML? | No — it becomes `React.createElement(...)` |
| Why `className`? | `class` is a reserved JavaScript keyword |
| Why capital component names? | Lowercase means "HTML tag", capital means "your component" |
| Can I use `if` inside `{}`? | No — use a ternary, or move the `if` above `return` |
| Why does `{0 && <X/>}` print `0`? | `0` is a number, and React renders numbers |

---

## 10. Revision questions

**Q1. What does `<h1>Hi</h1>` become after Babel runs?**
A function call: `React.createElement("h1", null, "Hi")` (or `_jsx("h1", { children: "Hi" })` in modern React). It returns a plain JavaScript object describing the element.

**Q2. Why must JSX return only one root element?**
Because JSX becomes a function call, and a JavaScript function can only return one value. Two sibling elements would be two values.

**Q3. What is a Fragment and when do you use it?**
`<>...</>` — an invisible wrapper. Use it when you need one root element but do not want an extra `<div>` in the DOM, which would otherwise break flexbox or grid layouts.

**Q4. Why is it `className` and not `class`?**
JSX attributes become keys on a JavaScript object, and `class` is a reserved word in JavaScript. Same reason `for` became `htmlFor`.

**Q5. What is the difference between `{}` and `{{}}` in JSX?**
`{}` means "switch to JavaScript". `{{}}` is that same switch containing an object literal — used for `style={{ color: "red" }}`.

**Q6. Why does `<mybutton />` render nothing?**
Lowercase tags are treated as HTML tag names. React looks for a real HTML element called `mybutton`, does not find one, and renders nothing. No error is shown.

**Q7. Why can't you write `if` inside curly braces?**
Braces accept an **expression** (something that produces a value). `if` is a **statement** (it performs an action and produces nothing). Use a ternary instead.

**Q8. What happens with `{null}`, `{false}`, and `{undefined}`?**
All three render nothing. React skips them silently. This is exactly what makes `{condition && <Element />}` work.

**Q9. What is wrong with `{items.length && <List />}`?**
When `items` is empty, `items.length` is `0`. React renders numbers, so a literal `0` appears on screen. Fix it with `items.length > 0 && <List />`.

**Q10. Why does `onClick={handleClick()}` misbehave?**
The parentheses call the function immediately during render and pass its return value (usually `undefined`) to `onClick`. Pass the function itself: `onClick={handleClick}`.

**Q11. Why do list items need a `key`?**
React uses the key to match each element with its previous version between re-renders. Without a stable key, React can reuse the wrong DOM node and show stale content.

---

## 11. What to learn next

Now that you can read and write JSX correctly, the next step is putting JSX into reusable pieces.

- **03 — Components and props** — how to split a page into small components and pass data into them
- **04 — Handling events** — `onClick`, `onChange`, `onSubmit`, and the event object
- **05 — Conditional rendering and lists** — going deeper on `&&`, ternaries, `map`, and keys

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
