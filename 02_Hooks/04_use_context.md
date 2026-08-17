# 04. useContext

> `useContext` lets a component read a value from far above it in the tree **without** that value being handed down through every component in between.

---

## 1. Real-life analogy

Think of a school building.

The principal wants every classroom to know today's timetable. One way: the principal tells the vice-principal, the vice-principal tells the head of department, the head tells the class teacher, and the class teacher finally tells the students. Four people repeated a message that was never about them. If one of them is absent, the message stops.

The other way: the principal switches on the **school announcement speaker**. The sound fills the whole building. Any classroom that wants the timetable simply listens. Nobody in the middle has to carry anything.

`useContext` is that speaker system. One component at the top broadcasts a value. Any component below, at any depth, tunes in directly.

Two important details in the analogy, and they are both true in React:

- The speaker only reaches rooms **inside** the building. A component outside the provider hears nothing.
- If two speakers are on, a room hears the **nearest** one. A provider inside another provider wins for the components inside it.

**In simple words:** context is a loudspeaker for data, so middle components stop being messengers.

---

## 2. The problem — why does this exist?

### Prop drilling

In Chapter 01 we learned props: a parent gives data to a child. That is one-way and predictable, and for two or three levels it is perfect.

Now imagine an app where the logged-in user's name must appear in a small avatar deep inside the page.

```jsx
function App() {
  const [user, setUser] = useState({ name: "Naman", role: "admin" });

  // App has the user, but the component that needs it is 4 levels down
  return <Layout user={user} />;
}

function Layout({ user }) {
  // Layout does not use `user` at all — it only passes it on
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  // Sidebar does not use `user` either
  return <Menu user={user} />;
}

function Menu({ user }) {
  // Neither does Menu
  return <Avatar user={user} />;
}

function Avatar({ user }) {
  return <span>{user.name[0]}</span>; // ✅ finally used
}
```

Three components — `Layout`, `Sidebar`, `Menu` — accept a prop they do not care about. This is called **prop drilling**: drilling a value down through layers that only act as pipes.

Why it hurts:

| Problem | What actually goes wrong |
|---|---|
| Noise | Every middle component's props list grows with things it never reads |
| Fragile | Forget one `user={user}` in the chain and the value silently becomes `undefined` |
| Hard to change | Adding a `theme` means editing four files, not one |
| Bad reuse | `Sidebar` cannot be used anywhere else without also supplying `user` |
| Painful refactor | Move `Avatar` one level deeper and the whole chain changes again |

### What we actually want

Some data is genuinely **app-wide**: the current user, the theme (dark/light), the chosen language, the auth token, the items in a shopping cart. It does not belong to any single branch of the tree. Passing it hand to hand is the wrong shape for it.

We want a way to say: *"this value is available to everything inside here"* — and let the components that care read it directly.

> ⚠️ Prop drilling is not a sin. Two levels of props is simpler and clearer than context. Reach for context when the chain gets long **and** the data is shared by many components.

**In simple words:** context exists to stop components from carrying data they never use.

---

## 3. What it actually is

Context has three separate pieces. Beginners mix them up, so keep them apart in your head.

```
createContext()   ->  creates the channel        (do this once, in its own file)
<Ctx.Provider>    ->  broadcasts a value on it   (a component you render)
useContext(Ctx)   ->  listens to the channel     (a hook you call)
```

**1. `createContext(defaultValue)`** — creates the context object. Think of it as *reserving a radio frequency*. It holds no data yet.

```jsx
import { createContext } from "react";
const ThemeContext = createContext("light"); // "light" is the fallback
```

**2. `<ThemeContext.Provider value={...}>`** — a component you wrap around part of your tree. Everything inside can read `value`.

```jsx
<ThemeContext.Provider value="dark">
  <Page />
</ThemeContext.Provider>
```

**3. `useContext(ThemeContext)`** — the hook. It returns the `value` from the **nearest provider above** this component.

```jsx
const theme = useContext(ThemeContext); // "dark"
```

### The default value

The value passed to `createContext(...)` is used **only** when a component reads the context and there is **no provider above it at all**.

```jsx
const ThemeContext = createContext("light");

// Rendered outside any provider:
const theme = useContext(ThemeContext); // "light"  <- the default
```

> 💡 A good default makes a component usable on its own — handy in tests and in Storybook. If there is no sensible default, use `null` and throw a clear error when it is missing (shown in section 5).

### Context is not state

This is the single biggest misunderstanding.

Context does **not** store anything and does **not** re-render anything by itself. It only **transports** a value from a provider to the readers below.

The value usually *comes from* `useState` or `useReducer` in the provider component. State is the engine; context is the pipe.

```
useState  =  where the value lives and changes
context   =  how the value travels down the tree
```

**In simple words:** `createContext` makes the channel, a provider broadcasts on it, `useContext` listens — and real state still lives in `useState`.

---

## 4. Syntax / setup, step by step

We will build a theme switcher: a `dark` / `light` value any component can read, and any component can change.

### Step 1 — create the context in its own file

```jsx
// src/context/ThemeContext.jsx
import { createContext } from "react";

// null default = "there must be a provider"; we will enforce that in step 4
const ThemeContext = createContext(null);

export default ThemeContext;
```

> 💡 Keep the context in its own file. If you define it inside a component file, you get import cycles the moment two components need it.

### Step 2 — build a provider component

The provider component owns the state and renders the real `Provider`.

```jsx
// src/context/ThemeProvider.jsx
import { useState } from "react";
import ThemeContext from "./ThemeContext";

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light"); // the real state lives here

  function toggleTheme() {
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  }

  // The object below is what every reader will receive
  const value = { theme, toggleTheme };

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

export default ThemeProvider;
```

Note `children`. The provider does not know or care what is inside it — it just wraps whatever you give it. That is what makes it reusable.

### Step 3 — wrap the app

```jsx
// src/main.jsx
import { createRoot } from "react-dom/client";
import ThemeProvider from "./context/ThemeProvider";
import App from "./App";

createRoot(document.getElementById("root")).render(
  <ThemeProvider>
    <App />
  </ThemeProvider>
);
```

Everything inside `<App />`, at any depth, can now read the theme.

### Step 4 — write a custom hook to read it

You *can* call `useContext(ThemeContext)` directly. A tiny wrapper hook is better:

```jsx
// src/context/useTheme.js
import { useContext } from "react";
import ThemeContext from "./ThemeContext";

export default function useTheme() {
  const value = useContext(ThemeContext);

  // Clear error instead of a confusing "cannot read property of null"
  if (value === null) {
    throw new Error("useTheme must be used inside a <ThemeProvider>");
  }

  return value;
}
```

Three wins: a shorter call, no import of the context object everywhere, and a readable error when someone forgets the provider.

### Step 5 — use it anywhere

```jsx
import useTheme from "./context/useTheme";

function ThemeButton() {
  const { theme, toggleTheme } = useTheme(); // no props at all
  return <button onClick={toggleTheme}>Switch to {theme === "light" ? "dark" : "light"}</button>;
}
```

> ⚠️ In React 19 you may also write `<ThemeContext value={value}>` without `.Provider`. Both work in React 19; `.Provider` works in every version, so this note uses it.

**In simple words:** create the context, wrap the app in a provider that owns the state, then read it with a small custom hook.

---

## 5. Full working example (with comments)

A small notes app. It uses **two** contexts — theme and user — to show that contexts stack, and it lets a deeply nested component both read and change shared data.

```jsx
// ============================================================
// src/context/AppContexts.jsx  — the channels
// ============================================================
import { createContext, useContext, useState } from "react";

const ThemeContext = createContext(null);
const UserContext = createContext(null);

// ---------- Theme ----------
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  function toggleTheme() {
    // updater function: safe even if called twice quickly
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used inside <ThemeProvider>");
  return ctx;
}

// ---------- User ----------
export function UserProvider({ children }) {
  const [user, setUser] = useState(null); // null = logged out

  function login(name) {
    setUser({ name, id: 1 });
  }

  function logout() {
    setUser(null);
  }

  return (
    <UserContext.Provider value={{ user, login, logout }}>
      {children}
    </UserContext.Provider>
  );
}

export function useUser() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error("useUser must be used inside <UserProvider>");
  return ctx;
}
```

```jsx
// ============================================================
// src/App.jsx  — the tree
// ============================================================
import { ThemeProvider, UserProvider, useTheme, useUser } from "./context/AppContexts";

// --- Level 4: deepest component. Reads BOTH contexts, no props. ---
function Greeting() {
  const { theme } = useTheme();
  const { user, login, logout } = useUser();

  const styles = {
    padding: "12px",
    background: theme === "dark" ? "#222" : "#f4f4f4",
    color: theme === "dark" ? "#eee" : "#111",
  };

  if (!user) {
    return (
      <div style={styles}>
        <p>You are not logged in.</p>
        <button onClick={() => login("Naman")}>Log in</button>
      </div>
    );
  }

  return (
    <div style={styles}>
      <p>Hello, {user.name} 👋</p>
      <button onClick={logout}>Log out</button>
    </div>
  );
}

// --- Level 3: pure pipe. Notice: ZERO props. ---
function Sidebar() {
  return (
    <aside>
      <h3>Sidebar</h3>
      <Greeting />
    </aside>
  );
}

// --- Level 2: also zero props. ---
function Layout() {
  const { theme, toggleTheme } = useTheme();

  return (
    <div>
      <header>
        <button onClick={toggleTheme}>Theme: {theme}</button>
      </header>
      <Sidebar />
    </div>
  );
}

// --- Level 1: the root, wrapping everything ---
function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <Layout />
      </ThemeProvider>
    </UserProvider>
  );
}

export default App;
```

What to notice:

- `Layout` and `Sidebar` pass **no props at all**. Compare that with the drilling example in section 2.
- `Greeting` sits four levels deep and still reads two shared values plus two functions to change them.
- Providers nest freely. Order between unrelated providers does not matter; it matters only when one provider needs another's value.
- The functions (`login`, `toggleTheme`) travel through context just like data. That is how a deep component changes state that lives at the top.

**In simple words:** the providers hold the state, the middle components stay clean, and the deep component reads and updates directly.

---

## 6. How it works behind the scenes

### The lookup — walking up the tree

When a component calls `useContext(ThemeContext)`, React does **not** search the whole app. It walks **upward** from that component through its parents until it finds a provider for that exact context object.

```
        <ThemeContext.Provider value={{theme:"dark"}}>
                     |
                  <Layout>
                     |
                  <Sidebar>
                     |
                  <Greeting>  useContext(ThemeContext)
                     |
                     └──── walks UP ────► finds the nearest provider
                                          returns {theme:"dark"}
```

If it reaches the top of the tree and finds no provider, it returns the default from `createContext(default)`.

Two consequences fall straight out of this:

1. Context flows **down only**. A component can never read a provider that sits below it or beside it.
2. The **nearest** provider wins. Nesting the same context twice is legal and useful:

```jsx
<ThemeContext.Provider value={{ theme: "light" }}>
  <Page />                              {/* sees light */}
  <ThemeContext.Provider value={{ theme: "dark" }}>
    <Modal />                           {/* sees dark — nearest wins */}
  </ThemeContext.Provider>
</ThemeContext.Provider>
```

### The re-render — who updates and when

This is the part that surprises people.

When a provider's `value` changes, React re-renders **every consumer below it** — every component that called `useContext` for that context. It does this even if the consumer is wrapped in `React.memo`. Context deliberately bypasses memoization, because otherwise the value could never get through.

```
value changes in provider
        ↓
React finds every consumer of this context below
        ↓
each consumer re-renders (memo does not stop it)
        ↓
their children re-render as usual
```

### The identity trap

React decides "did the value change?" with `Object.is` — the same reference check we saw with `useState` in note 01. And a fresh object literal is a new reference every single render.

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  // ⚠️ brand-new object on EVERY render of ThemeProvider
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

If `ThemeProvider` re-renders for any unrelated reason, `{ theme, setTheme }` is a different object, so every consumer re-renders even though the theme did not change.

The fix is `useMemo` (chapter topic 06) — it keeps the *same* object while the inputs are unchanged:

```jsx
import { useMemo, useCallback } from "react";

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  // same function object across renders
  const toggleTheme = useCallback(() => {
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  }, []);

  // same object unless `theme` actually changes
  const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

> 💡 If the value is a single primitive — `value={theme}` — you do not need `useMemo`. Strings and numbers compare by value, not by reference.

### Why `{children}` saves you anyway

A provider that renders `{children}` has a hidden advantage. The `children` element is created by the **parent**, not by the provider. So when the provider re-renders because of its own state, `children` is the same element object as before, and React skips re-rendering that subtree. Only the actual context **consumers** re-render.

That is why the `<Provider>{children}</Provider>` shape is the standard pattern and not just a style choice.

**In simple words:** `useContext` walks up to the nearest provider, and every consumer re-renders when the provider's value changes identity — so keep that value stable.

---

## 7. Comparison with alternatives (table)

| Approach | Good for | Not good for | Boilerplate |
|---|---|---|---|
| **Props** | 1–3 levels, data used by the direct child | Long chains, app-wide data | None |
| **Component composition** (pass JSX as `children`) | Avoiding drilling without any new tool | Data needed by many scattered components | None |
| **`useContext`** | Low-frequency app-wide data: theme, user, language, auth | Values that change many times per second | Small |
| **`useReducer` + context** | Shared state with many action types | Very large apps with async flows | Medium |
| **Redux Toolkit** | Big apps, devtools, time-travel debugging, middleware | Small apps — it is overkill | High |
| **Zustand / Jotai** | Shared state without provider wrapping, fine-grained updates | Teams that want one blessed standard | Low |

### The composition alternative — worth knowing first

Before reaching for context, check if plain composition solves it. Instead of drilling `user` through `Layout`, pass the finished element:

```jsx
function App() {
  const [user] = useState({ name: "Naman" });

  // Layout receives ready-made JSX, so it needs no `user` prop
  return <Layout content={<Avatar user={user} />} />;
}

function Layout({ content }) {
  return <main>{content}</main>; // just renders what it was handed
}
```

No context, no drilling. Use this when the data is needed in **one** place. Use context when it is needed in **many**.

> ⚠️ Context is a delivery mechanism, not a state manager. Putting fast-changing data (mouse position, every keystroke of a form, animation frames) in context re-renders large parts of your app on each change.

**In simple words:** try props, then composition, then context — and only reach for a library when context genuinely stops being enough.

---

## 8. Common mistakes beginners make

**1. Forgetting the provider**

```jsx
const { theme } = useTheme(); // 💥 crash if no <ThemeProvider> above
```
The value is the default (often `null`), and destructuring it throws. Fix: throw a clear error inside your custom hook, as in step 4.

**2. Reading from a sibling or a parent of the provider**

```jsx
function App() {
  const { theme } = useTheme();        // ❌ App is OUTSIDE the provider
  return <ThemeProvider><Page /></ThemeProvider>;
}
```
Context only flows **down**. Move the provider up, usually into `main.jsx`.

**3. A new object as `value` on every render**

```jsx
<Ctx.Provider value={{ user, setUser }}>   // ❌ re-renders all consumers
```
Wrap it in `useMemo`.

**4. Putting everything into one giant context**

One `AppContext` holding theme + user + cart + settings means a theme toggle re-renders cart components too. Split by how often each piece changes.

**5. Using context for data that changes constantly**

Form input on every keystroke, scroll position, mouse coordinates — keep these in local state. Context broadcasts to everyone.

**6. Thinking context replaces state**

```jsx
const ThemeContext = createContext("light");
// ❌ There is no way to "change" this. It is not state.
```
Context carries a value; `useState`/`useReducer` create and change it.

**7. Calling `useContext` conditionally**

```jsx
if (loggedIn) {
  const { user } = useContext(UserContext); // ❌ breaks the rules of hooks
}
```
Always call it at the top level. Use the result conditionally instead.

**8. Defining the context inside a component**

```jsx
function App() {
  const Ctx = createContext(null); // ❌ a brand-new context every render
}
```
Every render creates a new channel, so consumers lose their value. Define it at module level.

**9. Mutating the context value**

```jsx
const { user } = useUser();
user.name = "New"; // ❌ nothing re-renders
```
Same rule as state: call the setter with a new object.

**10. Reaching for Redux too early**

Theme + user + language is a job context handles perfectly. Add a library when you feel real pain, not before.

**In simple words:** most context bugs are a missing provider, an unstable value object, or context being mistaken for state.

---

## 9. Cheat sheet

```jsx
// 1. create (module level, own file)
const MyContext = createContext(defaultValue);

// 2. provide (owns the state)
<MyContext.Provider value={value}>{children}</MyContext.Provider>
// React 19 shorthand: <MyContext value={value}>{children}</MyContext>

// 3. consume (top level of a component)
const value = useContext(MyContext);
```

| Thing | Rule |
|---|---|
| `createContext(x)` | `x` is used **only** when no provider exists above |
| Where to define | Module level, own file — never inside a component |
| Direction | Flows **down** only; parents and siblings cannot read it |
| Multiple providers | Nearest one above wins |
| Who re-renders | Every consumer below, when `value` changes identity |
| `React.memo` | Does **not** block a context update |
| Object as `value` | Wrap in `useMemo`, functions in `useCallback` |
| Primitive as `value` | No `useMemo` needed |
| Good for | Theme, user, language, auth token, cart |
| Bad for | Keystrokes, mouse position, anything changing every frame |
| Best practice | Export a custom `useThing()` hook that throws when the provider is missing |

**The standard file layout:**

```
src/
└── context/
    ├── ThemeContext.jsx     <- createContext + provider component
    └── useTheme.js          <- the custom reader hook
```

**In simple words:** create at module level, provide state at the top, read with a custom hook, and memoize object values.

---

## 10. Revision questions (with answers)

**1. What problem does context solve?**
Prop drilling — passing a value through components that never use it, just to reach a deep one.

**2. What are the three pieces of context?**
`createContext()` makes the channel, `<Ctx.Provider value={...}>` broadcasts a value, `useContext(Ctx)` reads it.

**3. When is the default value from `createContext` used?**
Only when a component reads the context and there is **no** provider above it anywhere in the tree.

**4. Is context a state manager?**
No. It only transports a value. The value itself normally comes from `useState` or `useReducer` inside the provider.

**5. A component sits beside the provider, not inside it. What does it read?**
The default value. Context flows downward only.

**6. Two providers for the same context are nested. Which one does an inner component see?**
The nearest one above it.

**7. Why does `value={{ theme, setTheme }}` cause extra re-renders?**
It creates a new object on every render, so `Object.is` sees a change and every consumer re-renders. `useMemo` fixes it.

**8. Does `React.memo` stop a context update from re-rendering a component?**
No. Context updates deliberately bypass `memo`, otherwise the value could never reach the consumer.

**9. Why should the context object be created at module level, not inside a component?**
A component body runs on every render, so it would create a brand-new context each time and consumers would lose their value.

**10. Name two kinds of data that should *not* go in context.**
Anything changing very frequently — form keystrokes, mouse position, scroll offset, animation values — because every consumer re-renders on each change.

**11. Why is exporting a custom `useTheme()` hook better than calling `useContext` directly?**
Shorter calls, one less import in every file, and a clear error message when the provider is missing.

**12. Why does the `{children}` pattern reduce re-renders?**
`children` is created by the parent, so it is the same element object when the provider re-renders — React skips that subtree and only re-renders the actual consumers.

---

## 11. What to learn next

You can now share data across the tree. The next problem is **complicated state**: when one piece of state has many possible updates (add, remove, toggle, clear, reset), a pile of `useState` calls becomes hard to follow.

`useReducer` gathers all those updates into one function with a clear list of actions. It also pairs beautifully with what you just learned — `useReducer` + context is the classic "small Redux without Redux" setup.

➡ Next note: `05_use_reducer.md`

Related notes:
- [01. useState](01_use_state.md) — where the value in a provider actually lives
- [02. useEffect](02_use_effect.md) — loading the logged-in user before putting it in context
- [03. useRef](03_use_ref.md) — the other hook that survives renders

⬅ [Back to chapter index](README.md)
