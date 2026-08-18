# 06. Route Splitting and Data Mode

> **Route splitting** means each page's JavaScript is downloaded only when someone actually visits it. **Data mode** is React Router's other way of declaring routes, where the router fetches a page's data *before* rendering it instead of after.

---

## 1. Real-life analogy

A restaurant, and two different decisions it makes.

**The first decision is when to cook.** A bad kitchen cooks all eighty dishes at six in the morning so that anything ordered is instantly ready. Most of it is thrown away, the kitchen is exhausted before opening, and the first customer waits behind all that pointless work. A good kitchen cooks a dish when it is ordered. There is a short wait the first time — and nothing is wasted.

**The second decision is when to take the order.** In one restaurant you are seated at an empty table, handed a menu, and only then does a waiter come, take your order, and disappear. You stare at an empty table for ten minutes. In another, you order at the door, and when you are shown to the table the food is already on it.

Same food, same kitchen. Completely different feeling.

| Restaurant | React app |
|---|---|
| Cooking all 80 dishes at 6am | one big bundle with every page in it |
| Cooking on order | `lazy()` — load a page's code on demand |
| The short wait for the first plate | the `<Suspense>` fallback |
| Being seated at an empty table | render first, then fetch — a spinner inside the page |
| Ordering at the door | data mode — the `loader` runs before the page renders |
| The waiter saying "two minutes" | `useNavigation().state === "loading"` |

The two decisions are independent. You can cook on demand and still seat people at empty tables. This note covers both, because both are about **when work happens** relative to what the user sees.

**In simple words:** split the code so pages download when needed, and consider fetching data before a page renders instead of after.

---

## 2. The problem — why does this exist?

### Problem A — every visitor downloads the whole app

Your imports are static, at the top of `App.jsx`:

```jsx
import Home from "./pages/Home.jsx";
import Orders from "./pages/Orders.jsx";
import Staff from "./pages/Staff.jsx";
import Settings from "./pages/Settings.jsx";
```

A bundler follows those imports and packs everything into one file. Run `npm run build` and you get something like:

```text
dist/assets/index-a1b2c3d4.js    612.40 kB │ gzip: 184.22 kB
```

Now consider who downloads that. Somebody opens your home page from a phone on a slow connection. They read one paragraph and leave. They downloaded:

- the admin dashboard they will never see,
- the settings screens they cannot access,
- the charting library that only the reports page uses,
- the rich text editor imported by one form.

None of it was needed. All of it had to arrive, be parsed and be executed before the first paragraph appeared.

### Problem B — one heavy dependency poisons the whole bundle

```jsx
// src/pages/Reports.jsx
import Chart from "some-charting-library";   // 300 kB on its own
```

That import is inside a page component, but a static import is a static import. The library is now in the main bundle, downloaded by everyone, forever — because one page, visited by 2% of users, needed it.

### Problem C — the spinner cascade

This is the data half of the note. Look at what a nested route does today:

```
navigate to /dashboard/orders/42
        ↓
DashboardLayout renders    → its useEffect starts fetching the user
        ↓  (spinner in the sidebar)
Orders renders             → its useEffect starts fetching the order list
        ↓  (spinner in the middle)
OrderDetail renders        → its useEffect starts fetching order 42
        ↓  (spinner again)
```

Three sequential waits, because each fetch could only start after its component rendered, and each component could only render after its parent did. This is called a **waterfall**: requests queue up behind each other even though nothing forced them to.

The user sees the layout, then a spinner, then part of the page, then another spinner. The screen assembles itself in front of them.

### Problem D — the guard's blank frame

Note 05 ended with an honest wart. The guard renders "Checking session…" before it can decide anything, because the check happens *during* rendering:

```jsx
if (status === "loading") return <p>Checking your session…</p>;
```

There is no way around it in declarative mode. Rendering is the only moment your code runs.

### What we actually want

- Code arrives when it is needed, not before.
- A heavy library stays out of everyone else's download.
- A page's data starts loading as early as possible, ideally in parallel.
- The old screen stays visible while the new one prepares, instead of flashing through spinners.

**In simple words:** the first two problems are about shipping less code, the last two are about starting the work sooner.

---

## 3. What it actually is

### Part 1 — `lazy` and `<Suspense>`

Two pieces from React itself, not from React Router.

```jsx
import { lazy, Suspense } from "react";

// A dynamic import(): the bundler puts this file in a SEPARATE chunk.
const Orders = lazy(() => import("./pages/Orders.jsx"));

<Suspense fallback={<p>Loading…</p>}>
  <Orders />
</Suspense>
```

`lazy` takes a function that returns a `import(...)` promise and gives you back a component. The first time React tries to render that component, the file is not there yet — so React shows the nearest `<Suspense>` fallback until it arrives.

Two rules:

- The imported file must have a **default export**. `lazy` uses it automatically.
- Every lazy component must have a `<Suspense>` somewhere above it, or React throws an error.

> 💡 The deep mechanism — how a component "suspends" and how React finds the nearest boundary — is covered properly in `07_Advanced/04_code_splitting_and_suspense.md`. Here we only need the practical shape.

### Part 2 — data mode

Everything so far has used **declarative mode**: routes written as JSX inside `<Routes>`. Data mode writes the same routes as an array of plain objects, and each object may carry a `loader`.

```jsx
import { createBrowserRouter, RouterProvider } from "react-router";

const router = createBrowserRouter([
  {
    path: "/orders",
    element: <Orders />,
    loader: async () => {
      const res = await fetch("/api/orders");
      return res.json();
    },
  },
]);

<RouterProvider router={router} />
```

Note what changed: there is no `<BrowserRouter>` and no `<Routes>`. `<RouterProvider>` replaces both.

The component reads what the loader returned:

```jsx
import { useLoaderData } from "react-router";

function Orders() {
  const orders = useLoaderData();     // already here — no loading state at all
  return <ul>{orders.map(o => <li key={o.id}>{o.item}</li>)}</ul>;
}
```

### The one idea that makes data mode different

```
declarative mode                     data mode
----------------                     ---------
render the component                 run the loader
        ↓                                    ↓
useEffect starts the fetch           wait for the data
        ↓                                    ↓
render again with a spinner          render the component ONCE, with data
        ↓
render again with data
```

**Render-then-fetch** versus **fetch-then-render**. Everything else — `action`, `errorElement`, `useNavigation` — follows from that one change.

### What data mode adds, in one table

| Feature | What it does |
|---|---|
| `loader` | fetches data before the route renders; read with `useLoaderData()` |
| `action` | handles a form submission; paired with `<Form method="post">` |
| `errorElement` | renders when a loader, action or render throws; read with `useRouteError()` |
| `useNavigation()` | tells you a navigation is in flight, so you can show a progress bar |
| `redirect()` | a loader can redirect **before** anything renders — no spinner frame |
| `lazy` (route property) | code-splits the route's component *and* its loader together |

That fifth row is the fix for Problem D.

**In simple words:** `lazy` + `<Suspense>` split the code, and data mode moves fetching from "after render" to "before render".

---

## 4. Syntax / setup, step by step

### Step 1 — find out what you are actually shipping

Do not optimise blind.

```bash
npm run build
```

Vite prints every chunk with its size. Anything over a few hundred kB gzipped on the initial load is worth attention. Note the number now so you can compare later.

### Step 2 — convert page imports to `lazy`

```jsx
// before
import Orders from "./pages/Orders.jsx";

// after
import { lazy } from "react";
const Orders = lazy(() => import("./pages/Orders.jsx"));
```

Only do this for **route-level** components at first. Splitting a small button gains nothing and costs an extra request.

### Step 3 — wrap the routes in `<Suspense>`

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/orders" element={<Orders />} />
  </Routes>
</Suspense>
```

### Step 4 — put the boundary in the right place

Where you put `<Suspense>` decides how much of the screen disappears while waiting.

```
<Suspense> around <Routes>            <Suspense> around <Outlet>
--------------------------            -------------------------
┌───────────────────────┐             ┌───────────────────────┐
│ ░░░ fallback ░░░░░░░░ │             │ sidebar   │ ░░░░░░░░░ │
│ ░░░ whole screen ░░░░ │             │ (stays!)  │ fallback  │
└───────────────────────┘             └───────────────────────┘
```

Almost always you want the second one — inside the layout, so the frame stays put:

```jsx
function DashboardLayout() {
  return (
    <div>
      <Sidebar />
      <main>
        <Suspense fallback={<p>Loading page…</p>}>
          <Outlet />        {/* only this area shows the fallback */}
        </Suspense>
      </main>
    </div>
  );
}
```

> ⚠️ Keep the fallback roughly the same size as the real content. A tiny "Loading…" replaced by a full page makes everything jump — a layout shift users notice.

### Step 5 — keep the eager pages eager

Do not lazy-load the page people land on. Splitting `Home` means the browser downloads the main bundle, discovers it needs another file, and requests it — two round trips instead of one, for the most important screen.

```jsx
import Home from "./pages/Home.jsx";                    // ✅ eager: the landing page
const Orders = lazy(() => import("./pages/Orders.jsx")); // ✅ lazy: behind a login
```

### Step 6 — preload on hover

The wait is only felt on the first visit to a page. You can usually remove even that by starting the download when the user's pointer touches the link — a few hundred milliseconds before they click.

```jsx
const loadOrders = () => import("./pages/Orders.jsx");
const Orders = lazy(loadOrders);

<Link to="/orders" onMouseEnter={loadOrders}>Orders</Link>
```

Calling `import()` twice is safe. The browser caches the module, so the second call resolves instantly.

### Step 7 — give each route its own document title

React 19 hoists `<title>`, `<meta>` and `<link>` out of wherever you render them and into `<head>`. So a page can set the browser tab title itself, with no extra library.

```jsx
function Orders() {
  return (
    <>
      <title>Orders · Chai Admin</title>     {/* React 19 moves this to <head> */}
      <h1>Orders</h1>
    </>
  );
}
```

> 💡 Older tutorials install `react-helmet` for exactly this. On React 19 you do not need it.

### Step 8 — (data mode) swap the router

```jsx
// declarative
<BrowserRouter>
  <App />          // which renders <Routes>
</BrowserRouter>

// data mode
<RouterProvider router={router} />
```

### Step 9 — (data mode) move a fetch into a loader

```jsx
// before — inside the component
useEffect(() => {
  fetch("/api/orders").then(r => r.json()).then(setOrders);
}, []);

// after — beside the route
{
  path: "orders",
  element: <Orders />,
  loader: () => fetch("/api/orders"),   // a Response is unwrapped for you
}
```

### Step 10 — (data mode) add an `errorElement`

If a loader throws, the nearest `errorElement` up the tree renders instead of the page. This is the router's own error boundary.

```jsx
{
  path: "orders",
  element: <Orders />,
  loader: loadOrders,
  errorElement: <OrdersError />,
}
```

```jsx
import { useRouteError } from "react-router";

function OrdersError() {
  const error = useRouteError();
  return <p>Could not load orders: {error.message}</p>;
}
```

> 💡 This is a routing-specific error boundary. The general React concept is in `07_Advanced/05_error_boundaries.md`.

**In simple words:** measure first, lazy-load only route components, put `<Suspense>` inside the layout, preload on hover — and in data mode move fetching from `useEffect` into a `loader`.

---

## 5. Full working example (with comments)

Two versions of the same small app, so you can compare them directly.

### Version A — declarative mode with route splitting

```jsx
// ============================================================
// src/App.jsx
// The landing page is eager. Everything behind it is lazy.
// ============================================================
import { lazy, Suspense } from "react";
import { Routes, Route, Link, Outlet, NavLink } from "react-router";

import Home from "./pages/Home.jsx";      // eager — this is where people land

// Each of these becomes its OWN .js file at build time.
// Keep the loader functions so we can also call them on hover.
const loadOrders   = () => import("./pages/Orders.jsx");
const loadReports  = () => import("./pages/Reports.jsx");
const loadSettings = () => import("./pages/Settings.jsx");

const Orders   = lazy(loadOrders);
const Reports  = lazy(loadReports);
const Settings = lazy(loadSettings);

function DashboardLayout() {
  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      <aside style={{ width: 180, padding: "1rem", background: "#f6f1ea" }}>
        <NavLink to="/dashboard" end>Home</NavLink>
        <br />
        {/* onMouseEnter starts the download BEFORE the click.
            By the time the user clicks, the chunk is usually already here. */}
        <NavLink to="/dashboard/orders" onMouseEnter={loadOrders}>
          Orders
        </NavLink>
        <br />
        <NavLink to="/dashboard/reports" onMouseEnter={loadReports}>
          Reports
        </NavLink>
        <br />
        <NavLink to="/dashboard/settings" onMouseEnter={loadSettings}>
          Settings
        </NavLink>
      </aside>

      <main style={{ flex: 1, padding: "1rem" }}>
        {/* The boundary is INSIDE the layout, so the sidebar never disappears */}
        <Suspense fallback={<p>Loading page…</p>}>
          <Outlet />
        </Suspense>
      </main>
    </div>
  );
}

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />

      <Route path="/dashboard" element={<DashboardLayout />}>
        <Route index element={<Home />} />
        <Route path="orders" element={<Orders />} />
        <Route path="reports" element={<Reports />} />
        <Route path="settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}

export default App;
```

```jsx
// ============================================================
// src/pages/Reports.jsx
// The heavy page. Its big import now lives in ITS chunk, not everyone's.
// A default export is REQUIRED for lazy() to work.
// ============================================================
function Reports() {
  // Pretend this is a 300 kB charting library.
  const bars = [40, 70, 30, 90, 55];

  return (
    <>
      {/* React 19 hoists this into <head> — no react-helmet needed */}
      <title>Reports · Chai Admin</title>

      <h1>Reports</h1>
      <div style={{ display: "flex", gap: 8, alignItems: "flex-end", height: 120 }}>
        {bars.map((h, i) => (
          <div key={i} style={{ width: 24, height: h, background: "#8B4513" }} />
        ))}
      </div>
    </>
  );
}

export default Reports;   // ✅ default export
```

Build it and the output changes shape:

```text
before splitting
dist/assets/index-a1b2c3d4.js      612.40 kB │ gzip: 184.22 kB

after splitting
dist/assets/index-9f8e7d6c.js      190.15 kB │ gzip:  61.40 kB   ← everyone
dist/assets/Orders-11aa22bb.js      18.72 kB │ gzip:   6.10 kB   ← only visitors of /orders
dist/assets/Reports-33cc44dd.js    301.88 kB │ gzip:  92.55 kB   ← only visitors of /reports
dist/assets/Settings-55ee66ff.js    24.03 kB │ gzip:   8.31 kB   ← only visitors of /settings
```

The first download went from 184 kB to 61 kB. Nobody lost a feature.

### Version B — the same app in data mode

```jsx
// ============================================================
// src/main.jsx
// RouterProvider replaces BrowserRouter AND Routes.
// ============================================================
import { createRoot } from "react-dom/client";
import { createBrowserRouter, RouterProvider, redirect } from "react-router";

import DashboardLayout from "./layouts/DashboardLayout.jsx";
import Orders from "./pages/Orders.jsx";
import OrdersError from "./pages/OrdersError.jsx";
import { getSession, fetchOrders, createOrder } from "./api.js";

const router = createBrowserRouter([
  {
    path: "/dashboard",
    element: <DashboardLayout />,

    // Runs BEFORE anything renders. If it redirects, no dashboard pixel
    // ever appears — this is the fix for note 05's "Checking session…" frame.
    loader: async () => {
      const user = await getSession();
      if (!user) throw redirect("/login");
      return user;
    },

    children: [
      {
        path: "orders",
        element: <Orders />,

        // Starts in PARALLEL with the parent loader, not after it.
        loader: () => fetchOrders(),

        // Handles <Form method="post"> submissions from this route.
        action: async ({ request }) => {
          const form = await request.formData();
          await createOrder(form.get("item"));
          return redirect("/dashboard/orders");   // reload the list
        },

        // The router's own error boundary for this branch.
        errorElement: <OrdersError />,
      },
    ],
  },
]);

createRoot(document.getElementById("root")).render(
  <RouterProvider router={router} />
);
```

```jsx
// ============================================================
// src/pages/Orders.jsx  (data mode)
// No useEffect. No useState. No loading flag. The data is simply here.
// ============================================================
import { useLoaderData, useNavigation, Form } from "react-router";

function Orders() {
  const orders = useLoaderData();          // already loaded before this rendered
  const navigation = useNavigation();      // "idle" | "loading" | "submitting"

  const busy = navigation.state !== "idle";

  return (
    <>
      <title>Orders · Chai Admin</title>
      <h1>Orders {busy && <small>(updating…)</small>}</h1>

      <ul>
        {orders.map((o) => (
          <li key={o.id}>{o.item}</li>
        ))}
      </ul>

      {/* Router <Form>, not <form>. It calls the route's action.
          No onSubmit, no preventDefault, no fetch call here. */}
      <Form method="post">
        <input name="item" placeholder="New order" required />
        <button type="submit" disabled={busy}>
          {busy ? "Saving…" : "Add"}
        </button>
      </Form>
    </>
  );
}

export default Orders;
```

```jsx
// ============================================================
// src/pages/OrdersError.jsx
// Renders instead of <Orders /> if its loader or action throws.
// ============================================================
import { useRouteError, Link } from "react-router";

function OrdersError() {
  const error = useRouteError();

  return (
    <section>
      <h1>Could not load orders</h1>
      <p>{error?.message ?? "Something went wrong."}</p>
      <Link to="/dashboard">Back to the dashboard</Link>
    </section>
  );
}

export default OrdersError;
```

### What just happened

Compare the two versions honestly:

1. **`Orders.jsx` in version B has no `useState`, no `useEffect`, and no loading flag.** The data arrived before the component did. Roughly twenty lines of plumbing disappeared.
2. **The two loaders run in parallel.** The session check and the order list start together, because the router knows the whole match chain before rendering any of it. In version A, `Orders` could not start fetching until `DashboardLayout` had rendered.
3. **The guard's blank frame is gone.** `throw redirect("/login")` happens before render, so the user never sees "Checking session…".
4. **The old page stays on screen during navigation.** `useNavigation().state` is `"loading"` while the next route's loader runs, and React Router keeps showing the current page. You get a subtle "(updating…)" instead of a blank screen.
5. **In version A, open DevTools → Network, and hover over "Reports".** A new `.js` file downloads on hover, before you click. Click, and the page is instant.
6. **Version B costs you something too.** The router now owns your data fetching. If you are already using TanStack Query or Redux (Chapters 04 and 05), you now have two systems that both cache data, and you have to decide which one owns what.

**In simple words:** splitting cut the first download by two thirds, and data mode removed the loading state from the component entirely.

---

## 6. How it works behind the scenes

### What the bundler does with `import()`

A **static** import is a promise to the bundler that this code is always needed:

```jsx
import Orders from "./pages/Orders.jsx";   // becomes part of the main bundle
```

A **dynamic** import is a marker that says "split here":

```jsx
import("./pages/Orders.jsx");              // becomes a separate file
```

At build time Vite walks the import graph and cuts it at every dynamic import:

```
main bundle                       separate chunks
-----------                       ---------------
index.js                          Orders-11aa22bb.js
├── react                         Reports-33cc44dd.js
├── react-router                  Settings-55ee66ff.js
├── App.jsx
└── Home.jsx
```

At runtime, `import()` inserts a `<script>` tag for the right file and returns a promise that resolves when it loads. That is all it is: a network request wrapped in a promise.

The hash in the filename (`11aa22bb`) is derived from the file's contents. Change the file, the hash changes, and browsers fetch the new version — while unchanged chunks stay cached.

### How `lazy` + `<Suspense>` fit together

```
React tries to render <Orders />
        ↓
the module has not arrived
        ↓
the component "suspends"
        ↓
React walks UP the tree to the nearest <Suspense>
        ↓
that boundary renders its fallback instead
        ↓
        … chunk arrives …
        ↓
React retries the render; <Orders /> appears; fallback disappears
```

The important word is **nearest**. That is why the position of `<Suspense>` decides how much of the screen the fallback covers — it replaces everything below the boundary, not everything on the page.

> 💡 What "suspends" really means, and why the same machinery powers data loading in React 19, is the subject of `07_Advanced/04_code_splitting_and_suspense.md`.

### Render-then-fetch versus fetch-then-render

This is the whole of data mode in one diagram.

```
DECLARATIVE (render-then-fetch)          DATA MODE (fetch-then-render)
-------------------------------          -----------------------------
navigate                                 navigate
   ↓                                        ↓
render Layout                            match the full chain FIRST
   ↓  effect → fetch user ──┐                ↓
render Orders (spinner)     │            run ALL loaders in parallel
   ↓  effect → fetch list ──┼─ serial        ├── layout loader ─┐
render Detail (spinner)     │                └── orders loader ─┼─ together
   ↓  effect → fetch order ─┘                                   ↓
finally the full page                    render the whole page once
```

The router can start every loader at once because it knows the entire match chain *before* rendering anything. Components cannot do that, because a child does not exist until its parent has rendered.

That is the real argument for data mode, and it is a good one.

### Why `redirect()` in a loader beats a guard component

```
guard component (declarative)          loader redirect (data mode)
-----------------------------          ---------------------------
render RequireAuth                     run the loader
   ↓                                      ↓
status === "loading"                   await getSession()
   ↓                                      ↓
render "Checking session…"  ← a frame  throw redirect("/login")
   ↓                                      ↓
status resolves                        the URL changes; NOTHING rendered
   ↓
render <Navigate>
```

The loader runs outside React's render cycle, so it can make the decision before a single pixel is committed.

### What `useNavigation()` tells you

```
navigation.state:
  "idle"        nothing happening
  "loading"     a loader is running for the next route
  "submitting"  an action is running
```

While it is `"loading"`, React Router deliberately keeps the **current** page on screen. This is the opposite of the declarative default, where the new route renders immediately and shows its own spinner. Old-page-with-a-progress-bar generally feels faster than blank-page-with-a-spinner.

### The cost of data mode

Nothing here is free:

- **No `<BrowserRouter>`/`<Routes>`.** Converting an existing app means rewriting the route tree as an array.
- **Two caching systems.** If you already use TanStack Query, loaders duplicate its job. Pick one owner per piece of data.
- **Loaders live outside React.** They cannot use hooks, so shared logic has to be plain functions.
- **More concepts.** `loader`, `action`, `errorElement`, `useNavigation`, `useFetcher`, `defer` — a bigger surface to learn.

For a small app, or one that already has a data layer, declarative mode plus TanStack Query is a perfectly good answer.

**In simple words:** dynamic imports become separate files, `<Suspense>` shows a fallback while one arrives, and data mode can start every loader in parallel because it matches the whole route chain before rendering.

---

## 7. Comparison with alternatives (table)

### The three modes of React Router v7

| | Declarative | Data | Framework |
|---|---|---|---|
| Routes written as | JSX `<Route>` | an array of objects | files on disk |
| Set up with | `<BrowserRouter>` + `<Routes>` | `createBrowserRouter` + `<RouterProvider>` | a Vite plugin / CLI |
| Data fetching | your own, in components | `loader` functions | `loader` + server rendering |
| Form handling | your own `onSubmit` | `action` + `<Form>` | same, plus server actions |
| Error handling | your own boundary | `errorElement` | same |
| Server rendering | no | no | yes |
| Good for | learning; apps with an existing data layer | data-heavy client apps | full-stack apps |

All three share `useParams`, `<Link>`, `<Outlet>` and `<NavLink>` — everything from notes 01–05 carries over unchanged.

### Where to fetch data

| Approach | Fetch starts | Parallel? | Cache | Extra library |
|---|---|---|---|---|
| `useEffect` in the component | after render | ❌ waterfalls | none | none |
| Route `loader` | before render | ✅ | per navigation | none |
| TanStack Query | after render | ✅ if you prefetch | full, shared | yes |
| Loader **plus** a query library | before render | ✅ | full, shared | yes |

Chapter 05 covers the last two properly. The short version: loaders solve *when*, query libraries solve *caching*, and the two can be combined.

### What to split, and what not to

| Candidate | Split it? | Why |
|---|---|---|
| The landing page | ❌ | an extra round trip on the most important screen |
| A route behind a login | ✅ | logged-out visitors never need it |
| A charting or editor library | ✅ | often the single biggest win |
| A modal opened by 5% of users | ✅ | load it on open |
| A button, a card, an icon | ❌ | a request costs more than the bytes saved |

**In simple words:** split routes and heavy libraries, keep the landing page eager, and choose data mode only if you want the router to own fetching.

---

## 8. Common mistakes beginners make

**1. No `<Suspense>` above a lazy component**

```jsx
const Orders = lazy(() => import("./pages/Orders.jsx"));
<Routes><Route path="/orders" element={<Orders />} /></Routes>   // ❌ throws
```

React throws: "A component suspended while responding to synchronous input." Wrap the routes, or the `<Outlet>`, in `<Suspense>`.

**2. The lazy file has no default export**

```jsx
export function Orders() {}    // ❌ lazy() cannot find it
export default Orders;         // ✅
```

**3. Calling `lazy()` inside a component**

```jsx
function App() {
  const Orders = lazy(() => import("./Orders.jsx"));   // ❌ new component every render
  ...
}
const Orders = lazy(() => import("./Orders.jsx"));     // ✅ module scope, once
```

Defining it during render creates a brand-new component type each time, so React unmounts and remounts it endlessly.

**4. A dynamic import with a fully variable path**

```jsx
import(pagePath);                       // ❌ the bundler cannot analyse this
import(`./pages/${name}.jsx`);          // ✅ a partial template it can resolve
```

The bundler needs enough of the string at build time to know which files to prepare.

**5. Splitting everything**

Fifty chunks means fifty requests. Split routes and heavy libraries; leave small components alone.

**6. Lazy-loading the landing page**

Main bundle downloads → discovers it needs another file → requests it. Two round trips for the screen that matters most.

**7. A fallback that is nothing like the content**

```jsx
<Suspense fallback={<p>…</p>}>      // ⚠️ 20px tall, replaced by a 900px page
<Suspense fallback={<PageSkeleton />}>  // ✅ roughly the right shape
```

**8. Putting `<Suspense>` too high**

Wrapping `<Routes>` means the nav bar and sidebar vanish on every first visit to a page. Put it around `<Outlet>` instead.

**9. Mixing the two modes**

```jsx
<RouterProvider router={router}>
  <BrowserRouter>…</BrowserRouter>     // ❌ two routers, two truths
</RouterProvider>
```

Pick one. `RouterProvider` replaces both `<BrowserRouter>` and `<Routes>`.

**10. Using hooks inside a loader**

```jsx
loader: () => {
  const { user } = useAuth();     // ❌ not a component, not a hook context
}
```

Loaders run outside React. Read from a plain module, or pass what you need through `request`.

**11. Expecting `lazy` to make the app faster overall**

It makes the **first load** faster. The total bytes are the same or slightly larger. If every user visits every page anyway, splitting gains you little.

**In simple words:** one `<Suspense>` above every lazy component, `lazy()` at module scope, default exports, and do not split what is not worth a request.

---

## 9. Cheat sheet

```jsx
// ---------- code splitting ----------
import { lazy, Suspense } from "react";

const Orders = lazy(() => import("./pages/Orders.jsx"));   // module scope
// the imported file MUST have `export default`

function Layout() {
  return (
    <div>
      <Sidebar />
      <Suspense fallback={<PageSkeleton />}>
        <Outlet />                {/* boundary inside the layout */}
      </Suspense>
    </div>
  );
}

// preload on hover
const load = () => import("./pages/Orders.jsx");
const Orders = lazy(load);
<Link to="/orders" onMouseEnter={load}>Orders</Link>

// per-route document title (React 19 — no helmet needed)
<title>Orders · Chai Admin</title>
```

```jsx
// ---------- data mode ----------
import {
  createBrowserRouter, RouterProvider,
  useLoaderData, useNavigation, useRouteError,
  Form, redirect,
} from "react-router";

const router = createBrowserRouter([
  {
    path: "/orders",
    element: <Orders />,
    loader: async ({ params, request }) => {
      const res = await fetch("/api/orders");
      if (!res.ok) throw new Response("Not found", { status: 404 });
      return res.json();
    },
    action: async ({ request }) => {
      const form = await request.formData();
      await save(form.get("item"));
      return redirect("/orders");
    },
    errorElement: <OrdersError />,
  },
]);

<RouterProvider router={router} />;

const data = useLoaderData();                 // in the element
const { state } = useNavigation();            // idle | loading | submitting
const error = useRouteError();                // in the errorElement
```

Ten things worth memorising:

```text
1. import()               -> a separate chunk
2. lazy()                 -> at module scope, default export required
3. <Suspense>             -> required above every lazy component
4. put it around <Outlet> -> so the layout survives
5. keep the landing page eager
6. onMouseEnter={load}    -> preload before the click
7. RouterProvider         -> replaces BrowserRouter AND Routes
8. loader                 -> fetch-then-render, runs in parallel
9. throw redirect(...)    -> guard with no spinner frame
10. loaders are not components -> no hooks inside them
```

**In simple words:** `lazy` + `<Suspense>` for the code, `loader` + `RouterProvider` for the data, and measure before and after.

---

## 10. Revision questions (with answers)

**1. What does `lazy()` actually do?**
It takes a function returning a dynamic `import()` and gives back a component. The bundler puts that file in a separate chunk, and the file is downloaded the first time React tries to render the component.

**2. Why must a lazily imported file have a default export?**
`lazy` reads the module's `default` property to find the component. A named-only export leaves it with nothing to render.

**3. What happens if there is no `<Suspense>` above a lazy component?**
React throws an error. There is no boundary to show a fallback while the chunk loads.

**4. Where should `<Suspense>` go in a dashboard, and why?**
Around the `<Outlet>` inside the layout, so only the content area shows the fallback and the sidebar stays on screen.

**5. Why should you not lazy-load the landing page?**
The browser must download the main bundle, discover it needs another file, then request it — two round trips for the most important screen.

**6. Why is calling `lazy()` inside a component wrong?**
It creates a new component type on every render, so React unmounts the old one and mounts a new one every time.

**7. What is preloading on hover, and why is it safe to call `import()` twice?**
Starting the chunk download on `onMouseEnter`, a few hundred milliseconds before the click. The module is cached after the first call, so a second call resolves immediately.

**8. What is the core difference between declarative mode and data mode?**
Declarative renders and then fetches; data mode fetches and then renders. Everything else in data mode follows from that.

**9. Why can loaders run in parallel when `useEffect` fetches cannot?**
The router matches the whole route chain before rendering, so it knows every loader up front. A child component does not exist until its parent has rendered, which forces a waterfall.

**10. How does `throw redirect("/login")` in a loader improve on a guard component?**
It runs before React renders anything, so the user never sees a "checking session" frame.

**11. What does `useNavigation().state` tell you, and what does the router do meanwhile?**
Whether a loader (`"loading"`) or action (`"submitting"`) is running. While it runs, the current page stays on screen instead of being replaced by a spinner.

**12. What replaces `<BrowserRouter>` and `<Routes>` in data mode?**
`<RouterProvider router={router} />`, with the routes defined by `createBrowserRouter`.

**13. Why can a loader not call `useAuth()`?**
Loaders run outside React's render cycle, so there is no component and no hook context. Use plain functions or module-level state instead.

**14. Name a real cost of adopting data mode.**
The router becomes a second data-caching system alongside anything you already use, such as TanStack Query, so you must decide which one owns each piece of data.

**15. Does code splitting reduce the total amount of JavaScript?**
No. It reduces what is needed for the **first** load. The total is about the same, occasionally slightly larger.

---

## 11. What to learn next

Chapter 03 is complete. Your app has real URLs, working links, dynamic and query parameters, nested layouts, guarded routes, and pages that download only when needed.

Now a different problem appears, and it comes from the routing itself. The dashboard knows who is logged in. The nav bar wants the cart count. The checkout page needs the same cart. A theme toggle in settings should change the whole app. None of these belong to a single route, so none of them can live in a single page component. Passing them down through layouts and outlets works for a while, and then stops.

That is state management: deciding where a piece of data lives, who owns it, and how everything else reads it. Chapter 04 walks from lifting state up, through Context done properly, to Redux Toolkit and Zustand — and, just as importantly, when you need none of them.

➡ Next chapter: `04_State_Management/01_lifting_state_and_when_you_need_more.md`

Related notes:
- [05. Protected Routes and Redirects](05_protected_routes_and_redirects.md) — the guard whose spinner frame a loader redirect removes
- [04. Nested Routes and Layouts](04_nested_routes_and_layouts.md) — why the `<Suspense>` boundary belongs beside the `<Outlet>`
- [02. useEffect](../02_Hooks/02_use_effect.md) — the render-then-fetch pattern data mode replaces
- [08. Custom Hooks](../02_Hooks/08_custom_hooks.md) — where the `useFetch` logic lives when you stay in declarative mode

⬅ [Back to chapter index](README.md)
