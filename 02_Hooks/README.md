# 02_Hooks — Memory and Side Effects in Components

A **hook** is a special React function whose name starts with `use`.
It lets a plain function component do things a plain function cannot do on its own:
remember values between renders, talk to the outside world, and share logic with other components.

Read the files in number order. Every note follows the same 11-part shape.

---

## Notes in this chapter

| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [useState](01_use_state.md) | Why plain variables fail, the `[value, setValue]` pair, initial value used once, lazy init, updater functions, batching, why you must never mutate arrays/objects, derived vs stored data, state slots inside React |
| 02 | [useEffect](02_use_effect.md) | Why render must stay pure, setup + cleanup + dependency array, the three array forms, render→paint→effect order, why effects run twice in dev (StrictMode), fetching with `AbortController`, race conditions, and when an event handler is the right answer instead |
| 03 | [useRef](03_use_ref.md) | The `{ current }` box, why plain variables and state both fail for a timer id, attaching `ref` to a DOM node, when React fills it in, stopwatch example, the `usePrevious` pattern, ref-vs-state decision table, why you must not read or write refs during render |
| 04 | [useContext](04_use_context.md) | Prop drilling and why it hurts, the three pieces (`createContext` / Provider / `useContext`), the default value, building a theme + user provider, a custom reader hook that throws, how React walks up to the nearest provider, why every consumer re-renders and `memo` cannot stop it, the unstable-value trap, context vs composition vs Redux |
| 05 | [useReducer](05_use_reducer.md) | The bank-slip analogy, when many `useState` calls start disagreeing, `(state, action) => newState`, the three purity rules, actions named after events, a full task-list example, what `dispatch` really does, why returning the same object skips the render, why StrictMode runs the reducer twice, `useState` vs `useReducer` decision table, reducer + context as a small Redux |
| 06 | [useMemo](06_use_memo.md) | The shop-slip analogy, why every render re-runs the whole component body, the two reasons to memoize (slow work + stable references), why a new object kills `React.memo`, passing a function not a call, getting the dependency array right, a searchable 5,000-product example you can prove in the console, hook slots and shallow `Object.is` comparison, why `useMemo` is not free, and the React Compiler |
| 07 | [useCallback](07_use_callback.md) | The courier-card analogy, why two identical functions are never equal, how a fresh function silently kills `React.memo` and re-runs effects, `useCallback` vs `useMemo`, why the arrow is still created every render, the updater form that empties the deps array, a todo app whose console proves which rows skip, stale closures, when *not* to wrap (most DOM handlers), and the `useEventCallback` ref pattern |
| 08 | [Custom hooks](08_custom_hooks.md) | The recipe-vs-shared-pot analogy, why you cannot extract hook logic into a plain function, what the `use` prefix really does, choosing array vs object returns, three reusable hooks (`useLocalStorage`, `useDebounce`, `useFetch`) composed into one search page, how hook slots flatten onto the caller's fiber, custom hooks vs Context vs HOCs vs render props, cleanup leaks, and when to reach for a library instead |

---

## The plan for this chapter

These are the topics this chapter will cover, in learning order.
A row moves into the table above once its note is written.

| # | Topic | Short answer to "what is it for?" |
|---|-------|-----------------------------------|
| ~~01~~ | ~~`useState`~~ | ✅ written — see the table above |
| ~~02~~ | ~~`useEffect`~~ | ✅ written — see the table above |
| ~~03~~ | ~~`useRef`~~ | ✅ written — see the table above |
| ~~04~~ | ~~`useContext`~~ | ✅ written — see the table above |
| ~~05~~ | ~~`useReducer`~~ | ✅ written — see the table above |
| ~~06~~ | ~~`useMemo`~~ | ✅ written — see the table above |
| ~~07~~ | ~~`useCallback`~~ | ✅ written — see the table above |
| ~~08~~ | ~~Custom hooks~~ | ✅ written — see the table above |

---

## The rules of hooks (true for every hook)

Two rules. Break them and React breaks.

1. **Only call hooks at the top level of a component.**
   Never inside an `if`, a loop, or a nested function.
2. **Only call hooks from React function components or from other custom hooks.**
   Never from a normal helper function or a class.

```jsx
function Profile() {
  const [name, setName] = useState("Naman"); // ✅ top level

  if (name) {
    const [age, setAge] = useState(0);       // ❌ inside an if — not allowed
  }

  return <p>{name}</p>;
}
```

**Why these rules exist:** React does not know hook *names*. It only remembers hooks by **order**.
Render 1 sees hook #1, hook #2, hook #3. Render 2 must see the exact same three, in the exact same order.
An `if` can change that order, so React would hand back the wrong value.

```
Render 1        Render 2 (with the if)     Result
--------        ----------------------     ------
#1 name         #1 name                    ok
#2 age          #2 (skipped!)              ok-ish
#3 email        #3 -> gets age's slot      💥 wrong value
```

> 💡 Name every custom hook `useSomething`. The `use` prefix is how the linter knows to check these rules for you.

> ⚠️ The rules apply to **all** hooks — built-in and custom.

**In simple words:** hooks must be called the same number of times, in the same order, on every render.

---

## Where hooks fit with Chapter 01

| Chapter 01 taught | Chapter 02 adds |
|---|---|
| Props — data a parent **gives** a component | State — data a component **owns** |
| A component renders once from its props | A component can re-render itself when its state changes |
| Events call a function | Events change state, and state changes the screen |

**In simple words:** props come from outside, state lives inside, and hooks are how state gets there.

---

⬅ [Back to master index](../README.md)
