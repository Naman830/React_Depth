# React_Depth 📘

Beginner-friendly notes on **React.js** and its ecosystem — written in simple English, step by step, with the *why* behind everything.

- 📝 Notes only. No app code, no `node_modules`, nothing to install.
- ⚛️ All examples use modern React: **function components + hooks**.
- 📂 Topics are numbered in learning order. Just start at `01` and go down.

---

## How to use this repo

1. Open the chapter folder you want.
2. Read the files in number order.
3. Each note ends with revision questions — answer them before moving on.

---

## Index

### 01_Basic — React Fundamentals

| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [Set up React environment](01_Basic/01_setUp_react_env.md) | Node & npm, why a build tool is needed, Vite vs Parcel vs CRA vs Next.js, creating an app step by step, every file explained, npm scripts, common errors |
| 02 | [JSX rules in depth](01_Basic/02_jsx_rules.md) | What JSX compiles to, one root element & Fragments, closing tags, camelCase attributes, `{}` expressions, capital component names, what renders and what is skipped, keys, 10 common mistakes |
| 03 | [Components and props](01_Basic/03_components_and_props.md) | Why components exist, capitalized functions returning JSX, `export`/`import`, passing every value type as a prop, destructuring & defaults, `children`, sending data back up with callbacks, one-way data flow, props vs state |
| 04 | [Handling events](01_Basic/04_handling_events.md) | `onClick` and friends, pass vs call the handler, the event object, passing arguments with arrow functions, `preventDefault`, bubbling & `stopPropagation`, keyboard events, event delegation, SyntheticEvent |
| 05 | [Conditional rendering and lists](01_Basic/05_conditional_rendering_and_lists.md) | Guard clauses, ternary, `&&` and the `0` trap, `??` vs `\|\|`, `?.`, object lookups, `.map()` and `.filter()`, empty states, and what `key` really does |

### 02_Hooks — Memory and Side Effects

> Chapter index: [02_Hooks/README.md](02_Hooks/README.md) — includes the two rules of hooks.

| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [useState](02_Hooks/01_use_state.md) | Why plain variables fail, the `[value, setValue]` pair, initial value used once, lazy init, updater functions, batching, why you must never mutate arrays/objects, derived vs stored data, state slots inside React |
| 02 | [useEffect](02_Hooks/02_use_effect.md) | Why render must stay pure, setup + cleanup + dependency array, the three array forms, render→paint→effect order, why effects run twice in dev (StrictMode), fetching with `AbortController`, race conditions, and when an event handler is the right answer instead |
| 03 | [useRef](02_Hooks/03_use_ref.md) | The `{ current }` box, why plain variables and state both fail for a timer id, attaching `ref` to a DOM node, when React fills it in, stopwatch example, the `usePrevious` pattern, ref-vs-state decision table, why you must not read or write refs during render |
| 04 | [useContext](02_Hooks/04_use_context.md) | Prop drilling and why it hurts, the three pieces (`createContext` / Provider / `useContext`), the default value, building a theme + user provider, a custom reader hook that throws, how React walks up to the nearest provider, why every consumer re-renders and `memo` cannot stop it, the unstable-value trap, context vs composition vs Redux |
| 05 | [useReducer](02_Hooks/05_use_reducer.md) | The bank-slip analogy, when many `useState` calls start disagreeing, `(state, action) => newState`, the three purity rules, actions named after events, a full task-list example, what `dispatch` really does, why returning the same object skips the render, why StrictMode runs the reducer twice, `useState` vs `useReducer` decision table, reducer + context as a small Redux |
| 06 | [useMemo](02_Hooks/06_use_memo.md) | The shop-slip analogy, why every render re-runs the whole component body, the two reasons to memoize (slow work + stable references), why a new object kills `React.memo`, passing a function not a call, getting the dependency array right, a searchable 5,000-product example you can prove in the console, hook slots and shallow `Object.is` comparison, why `useMemo` is not free, and the React Compiler |
| 07 | [useCallback](02_Hooks/07_use_callback.md) | The courier-card analogy, why two identical functions are never equal, how a fresh function silently kills `React.memo` and re-runs effects, `useCallback` vs `useMemo`, why the arrow is still created every render, the updater form that empties the deps array, a todo app whose console proves which rows skip, stale closures, when *not* to wrap (most DOM handlers), and the `useEventCallback` ref pattern |
| 08 | [Custom hooks](02_Hooks/08_custom_hooks.md) | The recipe-vs-shared-pot analogy, why you cannot extract hook logic into a plain function, what the `use` prefix really does, choosing array vs object returns, three reusable hooks (`useLocalStorage`, `useDebounce`, `useFetch`) composed into one search page, how hook slots flatten onto the caller's fiber, custom hooks vs Context vs HOCs vs render props, cleanup leaks, and when to reach for a library instead |


### 03_Routing — Giving Your App Real URLs

> Chapter index: [03_Routing/README.md](03_Routing/README.md) — includes how a URL becomes a screen, and where a value should live.

| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [Setting up React Router](03_Routing/01_react_router_setup.md) | The hotel-lift analogy, why a `useState` page switcher breaks the back button, sharing and refresh, installing `react-router`, the three pieces (`BrowserRouter` / `Routes` / `Route`), why `element` takes JSX and not a function, the `*` catch-all, a four-page chai shop, the History API and `popstate`, why routes are ranked instead of ordered, and the server rewrite rule that stops production 404s |
| 02 | [Links and navigation](03_Routing/02_links_and_navigation.md) | The shopping-mall analogy, exactly what a plain `<a>` destroys, `<Link>` vs `<a>`, `<NavLink>` with `isActive` and why Home needs `end`, `useNavigate` for moves your code decides, `navigate(-1)`, push vs replace drawn as a history stack, the redirect loop `replace` prevents, `location.state` for one-off messages, and why you must never navigate during render |
| 03 | [URL params and search params](03_Routing/03_url_params_and_search_params.md) | The parcel-address analogy, why you cannot write one route per product, `:id` segments and `useParams`, why every param is a string, `useSearchParams` for search / filter / sort / page, why the object setter wipes your other filters, `replace` while typing, splat routes, a product list whose entire view lives in the URL, and a table of what belongs in the path, the query string, `location.state` or plain state |
| 04 | [Nested routes and layouts](03_Routing/04_nested_routes_and_layouts.md) | The newspaper-masthead analogy, why copying a sidebar into every page makes it flicker and lose state, nesting `<Route>` elements, `<Outlet>` as the hole in the frame, index routes, pathless layout routes, relative paths and `..`, `useOutletContext`, the match chain drawn out, and why the layout survives navigation while only the inner part changes |
| 05 | [Protected routes and redirects](03_Routing/05_protected_routes_and_redirects.md) | The security-desk analogy, why per-page auth checks fail open, one guard component returning `<Outlet />` or `<Navigate>`, why "loading" must be a third state, remembering the destination with `state={{ from: location }}`, why `replace` is needed in two places, `RequireGuest` and `RequireRole`, why a 403 is not a login redirect, and the honest reason a client guard is never security |
| 06 | [Route splitting and data mode](03_Routing/06_route_splitting_and_data_mode.md) | The restaurant analogy, why every visitor downloads the admin dashboard, `lazy()` + `<Suspense>`, where to put the boundary, preloading on hover, per-route `<title>` without react-helmet, then data mode side by side — `createBrowserRouter`, `loader`, `action`, `errorElement`, `useNavigation`, render-then-fetch vs fetch-then-render, and the honest cost of switching |

---

## Learning path (topics get added as I study them)

| Chapter | Covers |
|---------|--------|
| **[01_Basic](01_Basic/README.md)** | Setup, JSX, components, props, events, rendering |
| **[02_Hooks](02_Hooks/README.md)** | useState, useEffect, useRef, useContext, useReducer, useMemo, useCallback, custom hooks |
| **[03_Routing](03_Routing/README.md)** | React Router — routes, links, params, nested & protected routes, guards, code splitting |
| **04_State_Management** | Context API, Redux Toolkit, Zustand |
| **05_Data_Fetching** | fetch, axios, loading & error states, TanStack Query |
| **06_Forms** | Controlled inputs, validation, React Hook Form |
| **07_Advanced** | Virtual DOM, reconciliation, keys, performance, lazy loading, error boundaries, portals |
| **08_Ecosystem** | Tailwind CSS, Next.js, testing, deployment |

---

## Note format

Every note follows the same shape, so you always know where to look:

```
1. Real-life analogy
2. The problem — why does this exist?
3. What it actually is
4. Syntax / setup, step by step
5. Full working example
6. How it works behind the scenes
7. Comparison with alternatives
8. Common mistakes
9. Cheat sheet
10. Revision questions
11. What to learn next
```

---

*Notes are generated with the rules in [CLAUDE.md](CLAUDE.md).*
