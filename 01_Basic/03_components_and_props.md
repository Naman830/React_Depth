# 03. Components and Props

> A **component** is a JavaScript function that returns JSX — one reusable piece of your UI. **Props** are the inputs you hand to that function, the way you hand arguments to any normal function.

---

## 1. Real-life analogy

Think about a rubber stamp.

You carve a stamp once — say, a stamp that prints an ID card outline. Then you press it on paper 500 times. Every card has the same shape: a box, a photo slot, a name line, a role line.

But every card shows a **different** name and a **different** photo. The stamp gives the shape. The ink you fill in gives the details.

- **The stamp** = your component (the reusable shape)
- **The details you fill in** = props (name, photo, role)
- **Each printed card** = one instance of the component on screen

One more thing about a stamp: you cannot change the carving *while* you are stamping. The shape is fixed from the outside. In React this is exactly the rule — a component **cannot change its own props**. Whoever uses the stamp decides what goes on it.

**In simple words:** a component is the shape, props are the details poured into that shape from outside.

---

## 2. The problem — why do components exist?

Imagine building a page without components. Everything lives in one function.

```jsx
// ❌ One giant App — this is what beginners write first
function App() {
  return (
    <div>
      <div className="card">
        <img src="/amit.png" alt="Amit" />
        <h2>Amit</h2>
        <p>Frontend Developer</p>
        <button>Follow</button>
      </div>

      <div className="card">
        <img src="/sara.png" alt="Sara" />
        <h2>Sara</h2>
        <p>Backend Developer</p>
        <button>Follow</button>
      </div>

      <div className="card">
        <img src="/ravi.png" alt="Ravi" />
        <h2>Ravi</h2>
        <p>Designer</p>
        <button>Follow</button>
      </div>
      {/* ...and 40 more cards */}
    </div>
  );
}
```

Look at what is wrong here:

| Problem | What it costs you |
|---------|-------------------|
| The same 6 lines repeat 43 times | The file becomes 300+ lines of copy-paste |
| Change the design once → change it 43 times | One missed copy = one broken card |
| You cannot test one card alone | Everything is tangled in one function |
| You cannot reuse the card on another page | You copy-paste it again, into another file |
| Reading the file tells you nothing | You see markup, not meaning |

Now the same page with a component:

```jsx
// ✅ Define the shape ONCE
function UserCard({ name, role, avatar }) {
  return (
    <div className="card">
      <img src={avatar} alt={name} />
      <h2>{name}</h2>
      <p>{role}</p>
      <button>Follow</button>
    </div>
  );
}

// ✅ Use it as many times as you like
function App() {
  return (
    <div>
      <UserCard name="Amit" role="Frontend Developer" avatar="/amit.png" />
      <UserCard name="Sara" role="Backend Developer" avatar="/sara.png" />
      <UserCard name="Ravi" role="Designer" avatar="/ravi.png" />
    </div>
  );
}
```

Change the card design in **one** place and all 43 cards update. And now `App` reads like a sentence: *"this page has three user cards."*

**In simple words:** components exist so you write the UI once and use it everywhere.

---

## 3. What a component actually is

A React component is nothing magical. It is a plain JavaScript function with two rules:

1. Its name starts with a **Capital letter**.
2. It returns **JSX** (or `null`).

```jsx
function Welcome() {
  return <h1>Hello</h1>;
}
```

That is a complete, valid React component. There is no special keyword, no `extends`, no registration step.

### You never call it yourself

This is the part that confuses beginners. You *could* call it like a normal function, but you do not:

```jsx
{Welcome()}     // ⛔ works, but wrong — React sees plain JSX, not a component
<Welcome />     // ✅ correct — React sees a component and manages it
```

When you write `<Welcome />`, Babel turns it into `createElement(Welcome)`. React stores the **function itself** in the element object and calls it later, at the right moment:

```js
{ type: Welcome, props: {}, key: null, ref: null }
//      ↑ the actual function, not its result
```

Because React is the one calling it, React can track that component, remember its state, skip re-rendering it when nothing changed, and show it by name in DevTools. Call it yourself and you throw all of that away.

### A component is a function, so all normal JavaScript works

```jsx
function Clock() {
  const now = new Date();                 // normal JS
  const hours = now.getHours();           // normal JS
  const isMorning = hours < 12;           // normal JS

  return <p>Good {isMorning ? "morning" : "evening"}</p>;
}
```

Everything above the `return` is ordinary JavaScript. Only the `return` is special.

### One component per file (the normal convention)

```
src/
├── App.jsx
└── components/
    ├── UserCard.jsx      <- one component
    ├── Navbar.jsx        <- one component
    └── Button.jsx        <- one component
```

The file name matches the component name, in `PascalCase`. This is a convention, not a rule enforced by React — but every React codebase follows it.

**In simple words:** a component is a capitalized function that returns JSX, and React — not you — calls it.

---

## 4. Syntax, step by step

### Step 1 — Write the component in its own file

```jsx
// src/components/Greeting.jsx

function Greeting() {
  return <h1>Hello from a component!</h1>;
}

export default Greeting;   // makes it available to other files
```

### Step 2 — Import and use it

```jsx
// src/App.jsx

import Greeting from "./components/Greeting";   // no .jsx extension needed

function App() {
  return (
    <div>
      <Greeting />
      <Greeting />   {/* use it as many times as you want */}
    </div>
  );
}

export default App;
```

> 💡 `export default` means "this file's main thing". You can import it under any name: `import Hi from "./components/Greeting"` still works. Named exports (`export function Greeting`) must be imported with the exact name in braces: `import { Greeting } from ...`.

### Step 3 — Pass props (data going in)

Props look like HTML attributes, but you can pass **any** JavaScript value.

```jsx
<UserCard
  name="Amit"                          // string → quotes
  age={25}                             // number → braces
  isAdmin={true}                       // boolean → braces
  isActive                             // shorthand: same as isActive={true}
  skills={["React", "CSS"]}            // array → braces
  address={{ city: "Delhi" }}          // object → braces (double, like style)
  onFollow={handleFollow}              // function → braces
  icon={<StarIcon />}                  // even JSX → braces
/>
```

> ⚠️ Only strings use plain quotes. Everything else needs braces. `age="25"` gives you the **string** `"25"`, and `"25" + 1` is `"251"`, not `26`.

### Step 4 — Receive props inside the component

React collects every attribute into **one object** and passes it as the first argument.

```jsx
// Way 1 — take the whole props object
function UserCard(props) {
  return (
    <div>
      <h2>{props.name}</h2>
      <p>{props.role}</p>
    </div>
  );
}
```

```jsx
// Way 2 — destructure in the parameter list (most common)
function UserCard({ name, role }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{role}</p>
    </div>
  );
}
```

Both are identical. Way 2 is preferred because the parameter list immediately documents what the component needs.

### Step 5 — Default values

If a prop is not passed, it arrives as `undefined`. Give it a fallback right in the destructuring:

```jsx
function Button({ label = "Click me", color = "blue", size = "medium" }) {
  return <button className={`btn-${color} btn-${size}`}>{label}</button>;
}

<Button />                      // → "Click me", blue, medium
<Button label="Save" />         // → "Save", blue, medium
<Button label="Delete" color="red" />   // → "Delete", red, medium
```

> ⚠️ A default only fires for `undefined`, not for `null`. `<Button label={null} />` renders an empty button.

### Step 6 — `children`: the content between the tags

Anything you put *between* an opening and closing tag arrives as a special prop named `children`.

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>   {/* whatever was inside */}
    </div>
  );
}
```

```jsx
<Card title="Profile">
  <p>Name: Amit</p>          {/* ─┐            */}
  <p>Role: Developer</p>     {/*  ├─ children  */}
  <button>Edit</button>      {/* ─┘            */}
</Card>
```

This is how you build **wrapper** components — layouts, modals, panels — that do not know or care what goes inside them.

### Step 7 — Passing a function down (data coming back up)

Props flow **down**. To send something back **up**, the parent passes a function and the child calls it.

```jsx
// Parent owns the logic
function App() {
  function handleFollow(name) {
    alert(`You followed ${name}`);
  }

  return <UserCard name="Amit" onFollow={handleFollow} />;
}

// Child just reports "something happened"
function UserCard({ name, onFollow }) {
  return (
    <div>
      <h2>{name}</h2>
      {/* arrow function so it runs on click, not during render */}
      <button onClick={() => onFollow(name)}>Follow</button>
    </div>
  );
}
```

The child does not know what following means. It only announces the event. The parent decides what to do. This pattern is everywhere in React.

**In simple words:** props go down as values, and events come back up as functions.

---

## 5. Full working example

A small team page built out of four components.

```jsx
// src/components/Avatar.jsx
// The smallest component — it does one thing only.

function Avatar({ src, name, size = 64 }) {
  return (
    <img
      src={src}
      alt={name}                 // alt text matters for screen readers
      width={size}               // number in braces, not a string
      height={size}
      style={{ borderRadius: "50%", objectFit: "cover" }}
    />
  );
}

export default Avatar;
```

```jsx
// src/components/Badge.jsx
// A tiny component that returns null when it has nothing to show.

function Badge({ text, color = "gray" }) {
  if (!text) return null;        // returning null renders nothing at all

  const styles = {
    display: "inline-block",
    padding: "2px 8px",
    borderRadius: "10px",
    fontSize: "12px",
    color: "white",
    backgroundColor: color,
  };

  return <span style={styles}>{text}</span>;
}

export default Badge;
```

```jsx
// src/components/UserCard.jsx
// Composes Avatar + Badge. A component can use other components freely.

import Avatar from "./Avatar";
import Badge from "./Badge";

function UserCard({ user, onFollow }) {
  // Plain JavaScript above the return — Rule 4a from the JSX note
  const { name, role, avatar, isOnline, skills } = user;
  const statusText = isOnline ? "Online" : "Offline";
  const statusColor = isOnline ? "green" : "gray";

  return (
    <div
      style={{
        border: "1px solid #ddd",
        borderRadius: "8px",
        padding: "16px",
        width: "220px",
        fontFamily: "sans-serif",
      }}
    >
      {/* passing props down to a child component */}
      <Avatar src={avatar} name={name} size={72} />

      <h3 style={{ margin: "8px 0 4px" }}>{name}</h3>
      <p style={{ margin: 0, color: "#666" }}>{role}</p>

      <Badge text={statusText} color={statusColor} />

      <ul style={{ paddingLeft: "18px" }}>
        {skills.map((skill) => (
          <li key={skill}>{skill}</li>   // key is required in lists
        ))}
      </ul>

      {/* calling the parent's function — data flows back up */}
      <button onClick={() => onFollow(name)}>Follow</button>
    </div>
  );
}

export default UserCard;
```

```jsx
// src/App.jsx
// The parent: owns the data and the behaviour.

import UserCard from "./components/UserCard";

// Data lives in one place, as a plain array
const team = [
  { id: 1, name: "Amit", role: "Frontend Dev", avatar: "https://placehold.co/72",
    isOnline: true,  skills: ["React", "CSS"] },
  { id: 2, name: "Sara", role: "Backend Dev",  avatar: "https://placehold.co/72",
    isOnline: false, skills: ["Node", "SQL"] },
  { id: 3, name: "Ravi", role: "Designer",     avatar: "https://placehold.co/72",
    isOnline: true,  skills: ["Figma"] },
];

function App() {
  // The parent decides what "follow" means
  function handleFollow(name) {
    alert(`You followed ${name}`);
  }

  return (
    <div>
      <h1>Our Team ({team.length} people)</h1>

      <div style={{ display: "flex", gap: "16px", flexWrap: "wrap" }}>
        {team.map((member) => (
          <UserCard
            key={member.id}        // key goes on the OUTERMOST element of the map
            user={member}          // pass the whole object as one prop
            onFollow={handleFollow}
          />
        ))}
      </div>
    </div>
  );
}

export default App;
```

Four files, and now: change `Avatar` once and every avatar in the app changes. Add a fourth team member by adding one line to the array.

---

## 6. How it works behind the scenes

### The component tree

React builds a tree of components, and each one hands props to its children.

```
                     App
                      │  data: team[], handleFollow
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   UserCard      UserCard      UserCard
   user={amit}   user={sara}   user={ravi}
   onFollow=fn   onFollow=fn   onFollow=fn
        │
   ┌────┴────┐
   ▼         ▼
 Avatar    Badge
 src, name  text, color
```

Data flows **down** the arrows. This is called **one-way data flow**. A child can never reach up and change its parent's data — it can only call a function the parent gave it.

### What happens on render

```
┌───────────────────────────────────────────────────────────┐
│ 1. You write  <UserCard user={amit} onFollow={fn} />      │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌───────────────────────────────────────────────────────────┐
│ 2. Babel converts it to a function call                   │
│    createElement(UserCard, { user: amit, onFollow: fn })  │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌───────────────────────────────────────────────────────────┐
│ 3. A React element object is created                      │
│    { type: UserCard, props: { user, onFollow } }          │
│    NOTE: UserCard has NOT run yet. This is just a plan.   │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌───────────────────────────────────────────────────────────┐
│ 4. React CALLS the function, passing props as argument 1  │
│    UserCard({ user: amit, onFollow: fn })                 │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌───────────────────────────────────────────────────────────┐
│ 5. It returns more elements — including <Avatar /> and    │
│    <Badge />. React repeats step 4 for each of them,      │
│    going deeper until only real HTML tags are left.       │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌───────────────────────────────────────────────────────────┐
│ 6. React diffs the finished tree against the previous one │
│    and updates only the changed DOM nodes.                │
└───────────────────────────────────────────────────────────┘
```

Two consequences that matter a lot:

**(a) Props are read-only.** React passes you the props object and expects you not to touch it.

```jsx
function UserCard(props) {
  props.name = "Changed";      // ❌ never do this
  return <h2>{props.name}</h2>;
}
```

Why is this forbidden? Because the parent still holds the original object. If the child edits it, the parent's data silently changes, and now nobody can tell where a value came from. React calls a component that does not modify its inputs a **pure** component, and purity is what lets React safely skip re-renders and re-run components during development checks.

If you need a changed value, make a new one:

```jsx
function UserCard({ name }) {
  const displayName = name.toUpperCase();   // ✅ new variable, original untouched
  return <h2>{displayName}</h2>;
}
```

**(b) New props mean a re-render.** When a parent passes a different prop value, React calls the child function again with the new props and compares the result. You never write update code — you just describe what the UI should look like for a given set of props.

**In simple words:** React calls your function with props, you return a description, React updates the screen.

---

## 7. Comparison table

### Props vs State (the most common confusion)

You have not learned state yet — that is the next chapter — but the difference is worth seeing now.

| | **Props** | **State** |
|---|---|---|
| Where does it come from? | The parent component | Inside the component itself |
| Can the component change it? | ❌ No — read-only | ✅ Yes, with a setter function |
| Who owns it? | The parent | The component itself |
| Analogy | Arguments you pass to a function | A variable declared inside the function |
| Changes cause a re-render? | Yes, when the parent sends new ones | Yes, when you call the setter |
| Used for | Configuring a component from outside | Data that changes over time (input text, counters, toggles) |

### Ways to receive props

| Style | Code | When to use |
|-------|------|-------------|
| Whole object | `function C(props) { props.name }` | Many props, or you forward them all |
| Destructured | `function C({ name, role })` | Default choice — clearest |
| With defaults | `function C({ size = 64 })` | Optional props |
| Rest spread | `function C({ id, ...rest })` | Pull out a few, pass the rest onward |

```jsx
// Rest spread in action — a wrapper around a real <button>
function Button({ label, ...rest }) {
  // rest holds onClick, disabled, type, and anything else passed in
  return <button {...rest}>{label}</button>;
}

<Button label="Save" onClick={save} disabled type="submit" />
```

**In simple words:** props are the component's settings, given from outside and never edited inside.

---

## 8. Common mistakes beginners make

**1. Lowercase component name**

```jsx
function userCard() { ... }
<userCard />     {/* ❌ silent — React looks for an HTML tag "usercard" */}
<UserCard />     {/* ✅ */}
```

**2. Forgetting `export default` or the import path**

```jsx
export default UserCard;                       // ✅ in UserCard.jsx
import UserCard from "./components/UserCard";  // ✅ relative path starts with ./
import UserCard from "components/UserCard";    // ❌ not a relative path
```

**3. Modifying props**

```jsx
function C({ items }) {
  items.push("new");        // ❌ mutates the parent's array
  const next = [...items, "new"];   // ✅ make a copy
}
```

**4. Passing a number as a string**

```jsx
<Avatar size="64" />     {/* string "64" */}
<Avatar size={64} />     {/* ✅ number 64 */}
```

**5. Calling the handler instead of passing it**

```jsx
<button onClick={onFollow(name)} />          {/* ❌ runs during render */}
<button onClick={() => onFollow(name)} />    {/* ✅ runs on click */}
<button onClick={onFollow} />                {/* ✅ when no argument needed */}
```

**6. Destructuring the wrong shape**

```jsx
<UserCard user={member} />

function UserCard({ name }) { ... }         {/* ❌ name is undefined */}
function UserCard({ user }) { user.name }   {/* ✅ matches what was passed */}
```

**7. Defining a component inside another component**

```jsx
function App() {
  function Card() { return <div>Hi</div>; }   // ❌ redefined on every render
  return <Card />;                            //    React sees a brand-new type
}                                             //    each time and destroys the DOM
```

Move `Card` outside `App`, at the top level of the file.

**8. Putting `key` on the wrong element**

```jsx
{team.map(m => <div><UserCard key={m.id} /></div>)}   {/* ❌ key is too deep */}
{team.map(m => <UserCard key={m.id} user={m} />)}     {/* ✅ outermost element */}
```

**9. Expecting `children` to be a string**

```jsx
function Card({ children }) {
  return <div>{children.toUpperCase()}</div>;   // ❌ crashes when children is JSX
  return <div>{children}</div>;                 // ✅ just render it
}
```

**10. Prop drilling without noticing**

Passing a prop down through five components that do not use it is a smell. It works, but it means the data probably belongs in Context or a store — chapter 04 covers that. For now, just recognize the pattern when it appears.

---

## 9. Cheat sheet

```jsx
// ── DEFINE ──────────────────────────────────────────────────
function Hello() { return <h1>Hi</h1>; }        // Capital name, returns JSX
const Hello = () => <h1>Hi</h1>;                // arrow form works too
export default Hello;                            // one main export per file

// ── IMPORT ──────────────────────────────────────────────────
import Hello from "./components/Hello";          // default export
import { Hello } from "./components/Hello";      // named export

// ── USE ─────────────────────────────────────────────────────
<Hello />                                        // self-closing
<Hello>content</Hello>                           // content becomes children

// ── PASS PROPS ──────────────────────────────────────────────
<C text="hi" />              // string  → quotes
<C count={5} />              // number  → braces
<C active={true} />          // boolean → braces
<C active />                 // shorthand for active={true}
<C list={[1, 2]} />          // array
<C obj={{ a: 1 }} />         // object  → double braces
<C onSave={fn} />            // function
<C icon={<Star />} />        // JSX as a prop
<C {...someObject} />        // spread every key as a prop

// ── RECEIVE PROPS ───────────────────────────────────────────
function C(props) { props.text }                 // whole object
function C({ text }) { text }                    // destructured
function C({ text = "hi" }) { text }             // with a default
function C({ id, ...rest }) { <div {...rest} /> } // rest spread
function C({ children }) { children }            // content between tags

// ── SEND DATA BACK UP ───────────────────────────────────────
// Parent: <Child onDone={handleDone} />
// Child:  <button onClick={() => onDone(value)}>Go</button>

// ── RENDER NOTHING ──────────────────────────────────────────
if (!data) return null;
```

| Question | Answer |
|----------|--------|
| Can I change a prop inside a component? | No. Props are read-only. Make a new variable. |
| How do I give a prop a default? | `function C({ size = 64 })` |
| What is `children`? | Whatever sits between the opening and closing tags |
| How does a child talk to its parent? | The parent passes a function; the child calls it |
| Why must names be capitalized? | Lowercase means HTML tag, Capital means your component |
| Should I write `<C />` or `C()`? | `<C />` — let React call it |

---

## 10. Revision questions

**Q1. What is a component in one sentence?**
A JavaScript function whose name starts with a capital letter and which returns JSX describing a piece of the UI.

**Q2. What are props?**
The single object of inputs React passes to a component as its first argument, built from the attributes written on the JSX tag.

**Q3. Why can't a component change its own props?**
Because the parent owns that data. If the child edited it, the parent's value would change behind its back and nothing would be predictable. Read-only props are what make components pure and let React safely skip or repeat renders.

**Q4. What is the difference between `<Welcome />` and `Welcome()`?**
`<Welcome />` creates an element object holding the function, and React calls it — so React can track state and re-renders. `Welcome()` calls it immediately and React only sees the returned JSX, losing all component identity.

**Q5. How do you pass a number `25` as a prop?**
`age={25}` with braces. `age="25"` passes the string `"25"`.

**Q6. What is `children`?**
A built-in prop containing everything written between a component's opening and closing tags. It is what makes wrapper components possible.

**Q7. How does a child send data back to its parent?**
It cannot push data upward. The parent passes a function as a prop, and the child calls that function with the data. The parent then decides what to do.

**Q8. What does `function Card({ title = "Untitled" })` do?**
It destructures the `title` prop and uses `"Untitled"` when `title` is `undefined` — that is, when the parent did not pass it.

**Q9. Why is defining a component inside another component a bug?**
The inner function is recreated on every render, so React sees a different component type each time. It unmounts the old DOM and mounts new DOM, destroying state and hurting performance.

**Q10. What does `<Button {...props} />` do?**
It spreads every key of the `props` object as an individual prop. `{ a: 1, b: 2 }` becomes `a={1} b={2}`.

**Q11. Where does `key` go when mapping a list of components?**
On the outermost element returned by the `map` callback — the component itself, not a wrapper inside it.

**Q12. What does returning `null` from a component do?**
Renders nothing. The component still exists in the tree, it just produces no DOM. This is the standard way to conditionally hide a component.

---

## 11. What to learn next

You can now split a UI into pieces and feed them data. The next question is: what happens when that data needs to **change**?

- **04 — Handling events** — `onClick`, `onChange`, `onSubmit`, the event object, and passing arguments to handlers
- **05 — Conditional rendering and lists** — going deeper on `&&`, ternaries, `map`, and why keys matter
- **02_Hooks / 01 — useState** — giving a component its own memory, so the screen can change

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
