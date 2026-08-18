# 04. Nested Routes and Layouts

> Nested routes let a parent route draw the parts of the screen that never change — a header, a sidebar — and leave a hole, called `<Outlet>`, where whichever child route matched gets rendered.

---

## 1. Real-life analogy

Pick up a newspaper.

The masthead at the top — *The Daily Chai* — is printed on every single page. The section strip under it (News, Sports, Business) is on every page too. You never turn a page and find the masthead missing.

Below that, the content changes completely. Page 3 has a news story, page 9 has a cricket report.

Now turn to the sports section. It has its **own** header: a small strip listing Cricket, Football, Tennis. That strip appears on every sports page but on none of the news pages. So there are two frames, one inside the other, and only the innermost part actually changes.

```
┌──────────────────────────────────────┐
│  THE DAILY CHAI          (masthead)  │  ← always printed
│  News | Sports | Business            │  ← always printed
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │ Cricket | Football | Tennis    │  │  ← only in the sports section
│  ├────────────────────────────────┤  │
│  │                                │  │
│  │   India win by 6 wickets       │  │  ← the only part that really changes
│  │                                │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

| Newspaper | React Router |
|---|---|
| The masthead and section strip | a **layout** component |
| The blank area the story is printed into | `<Outlet />` |
| The sports sub-strip | a nested layout |
| Today's cricket report | the deepest matched route |
| The front page you get if you just say "sports" | an **index route** |

The printing press does not reprint the masthead for every page — it is part of the plate. In React terms: the layout component does not unmount when the story changes.

**In simple words:** a layout is the printed frame, `<Outlet>` is the blank space inside it, and nested routes decide what gets printed into that space.

---

## 2. The problem — why does this exist?

### Problem A — the sidebar gets copy-pasted into every page

You are building a dashboard. There is a sidebar on the left, and the content on the right changes.

With flat routes, every page must draw the sidebar itself:

```jsx
// ❌ src/pages/DashboardHome.jsx
function DashboardHome() {
  return (
    <div className="dash">
      <Sidebar />                     {/* repeated */}
      <main><h1>Overview</h1></main>
    </div>
  );
}

// ❌ src/pages/Orders.jsx
function Orders() {
  return (
    <div className="dash">
      <Sidebar />                     {/* repeated */}
      <main><h1>Orders</h1></main>
    </div>
  );
}
// ...and in Customers.jsx, and Settings.jsx, and every future page
```

Add a "Reports" link to the sidebar and you edit one file — fine. But add a notification bar above the sidebar and you edit *every page*. Miss one and the app looks broken on exactly that screen.

### Problem B — the sidebar unmounts on every navigation

This one is subtler and worse.

Because `<Sidebar />` is rendered *inside* `DashboardHome`, navigating from `/dashboard` to `/dashboard/orders` unmounts the whole thing and mounts a fresh copy.

```
navigate /dashboard -> /dashboard/orders

DashboardHome unmounts   -> Sidebar unmounts   -> its state is destroyed
Orders mounts            -> Sidebar mounts     -> brand new state
```

Everything the sidebar remembered is gone:

- Which group was expanded collapses again.
- Its scroll position jumps to the top.
- Any open/closed toggle resets.
- A CSS transition on it re-runs, so it visibly flickers.

The user clicked one link and the entire left-hand side of the app blinked.

### Problem C — sections need their own sub-navigation

Settings has three tabs: Profile, Password, Notifications. Those tabs belong to Settings and nowhere else. With flat routes you either repeat the tab strip in all three files, or you build the tabs with `useState` — and then the tabs have no URLs, which is exactly the problem note 01 solved.

### Problem D — the URL prefix is repeated in every route

```jsx
// ❌ "/dashboard" typed six times
<Route path="/dashboard" element={<DashboardHome />} />
<Route path="/dashboard/orders" element={<Orders />} />
<Route path="/dashboard/orders/:id" element={<OrderDetail />} />
<Route path="/dashboard/customers" element={<Customers />} />
<Route path="/dashboard/settings" element={<Settings />} />
<Route path="/dashboard/settings/password" element={<Password />} />
```

Rename the section to `/admin` and you edit six lines, and you will miss one.

### What we actually want

- The shared frame written **once**.
- The frame stays mounted while the inside changes.
- Sections can have their own nested frames.
- The URL prefix written once, and children stated relative to it.

**In simple words:** we want the parts of the screen that do not change to stop being re-created every time the parts that do change, change.

---

## 3. What it actually is

Nesting is done by putting `<Route>` elements **inside** another `<Route>` instead of self-closing it.

```jsx
<Route path="/dashboard" element={<DashboardLayout />}>
  <Route index element={<Overview />} />
  <Route path="orders" element={<Orders />} />
  <Route path="customers" element={<Customers />} />
</Route>
```

Read it as: "For any URL starting with `/dashboard`, render `DashboardLayout`. Then, inside it, render whichever child matched the rest."

### The four new ideas

| Idea | What it looks like | What it means |
|---|---|---|
| A nested `<Route>` | `<Route>…children…</Route>` | this route's element wraps the children |
| `<Outlet />` | `<Outlet />` inside the parent's element | "put the matched child here" |
| An index route | `<Route index element={…} />` | the child for when the URL is exactly the parent's path |
| A layout route | `<Route element={<X />}>` with **no** `path` | shares a layout without adding anything to the URL |

### `<Outlet />` — the hole in the frame

A parent's element is not replaced by the child. It **contains** it. The parent decides where, using `<Outlet>`:

```jsx
import { Outlet } from "react-router";

function DashboardLayout() {
  return (
    <div style={{ display: "flex" }}>
      <Sidebar />              {/* always here */}
      <main>
        <Outlet />             {/* ← the child route renders exactly here */}
      </main>
    </div>
  );
}
```

Forget the `<Outlet>` and the layout renders fine while every child appears to be broken — because there is nowhere for them to go. That is the single most common bug in this note.

### Child paths are relative

Inside a parent with `path="/dashboard"`, a child writes only the piece that comes after:

```jsx
<Route path="/dashboard" element={<Layout />}>
  <Route path="orders" element={<Orders />} />         {/* → /dashboard/orders */}
  <Route path="orders/:id" element={<OrderDetail />} />{/* → /dashboard/orders/42 */}
</Route>
```

No leading slash. A leading slash means "from the root" and will not do what you expect inside a parent.

### Index routes answer "what shows at the parent's own URL?"

```jsx
<Route path="/dashboard" element={<Layout />}>
  <Route index element={<Overview />} />        {/* URL is exactly /dashboard */}
  <Route path="orders" element={<Orders />} />  {/* URL is /dashboard/orders */}
</Route>
```

Without the index route, visiting `/dashboard` renders the layout with an empty hole — sidebar on the left, blank on the right. `index` is the default child.

> 💡 `index` and `path` are mutually exclusive. An index route has no path of its own; its path *is* the parent's.

### Layout routes have no path at all

Sometimes you want a shared frame without a URL segment. Drop the `path`:

```jsx
<Route element={<PublicLayout />}>          {/* adds nothing to the URL */}
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
</Route>
```

`/about` still means `/about`. The only effect is that `About` renders inside `PublicLayout`. This is how you give different parts of a site different frames.

**In simple words:** nest `<Route>` elements to nest components, put `<Outlet />` where the child should appear, and add an `index` route for the parent's own URL.

---

## 4. Syntax / setup, step by step

### Step 1 — pull the shared parts into a layout component

Take everything that repeats and move it into one file.

```jsx
// src/layouts/DashboardLayout.jsx
import { Outlet } from "react-router";
import Sidebar from "../components/Sidebar.jsx";

function DashboardLayout() {
  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      <Sidebar />
      <main style={{ flex: 1, padding: "1rem" }}>
        <Outlet />
      </main>
    </div>
  );
}

export default DashboardLayout;
```

### Step 2 — nest the routes under it

```jsx
<Route path="/dashboard" element={<DashboardLayout />}>
  <Route index element={<Overview />} />
  <Route path="orders" element={<Orders />} />
</Route>
```

Note the shape change: the parent is no longer self-closing. It opens, holds children, and closes.

### Step 3 — drop the leading slash on children

```jsx
<Route path="orders" />     // ✅ → /dashboard/orders
<Route path="/orders" />    // ❌ absolute — this is a different, top-level URL
```

### Step 4 — add the index route

Otherwise `/dashboard` shows the frame with nothing in it.

```jsx
<Route index element={<Overview />} />
```

### Step 5 — nest a second level for a section with sub-tabs

Nesting has no depth limit. Settings gets its own layout and its own index:

```jsx
<Route path="/dashboard" element={<DashboardLayout />}>
  <Route index element={<Overview />} />
  <Route path="orders" element={<Orders />} />

  <Route path="settings" element={<SettingsLayout />}>
    <Route index element={<Profile />} />
    <Route path="password" element={<Password />} />
    <Route path="notifications" element={<Notifications />} />
  </Route>
</Route>
```

`SettingsLayout` needs its own `<Outlet />`. Three components are now on screen at once for `/dashboard/settings/password`: dashboard frame → settings frame → the password form.

### Step 6 — use relative links inside a layout

```jsx
// inside SettingsLayout, where the current route is /dashboard/settings
<NavLink to="password">Password</NavLink>    // ✅ → /dashboard/settings/password
<NavLink to="/password">Password</NavLink>   // ❌ → /password, a top-level URL
<NavLink to="..">Back to dashboard</NavLink> // ✅ up one level
```

Relative links mean you can rename `/dashboard` to `/admin` in exactly one place.

### Step 7 — get `end` right on nested `NavLink`s

The prefix-matching rule from note 02 becomes more important here, because parent paths are prefixes of every child path.

```jsx
<NavLink to="/dashboard" end>Overview</NavLink>  {/* ✅ only on /dashboard */}
<NavLink to="/dashboard">Overview</NavLink>      {/* ❌ also on /dashboard/orders */}
```

### Step 8 — pass data from a layout to its children (optional)

If a layout has already loaded something the children need, hand it through the outlet instead of prop-drilling or Context:

```jsx
// in the layout
<Outlet context={{ user }} />

// in any child
import { useOutletContext } from "react-router";
const { user } = useOutletContext();
```

> 💡 Use this for data that is genuinely tied to this layout. For app-wide values like the logged-in user, Context (Chapter 02, note 04) is usually clearer.

**In simple words:** move the repeated parts into a layout, nest the routes inside it, add `<Outlet />` and an `index` route, and keep child paths and links relative.

---

## 5. Full working example (with comments)

A two-level dashboard. The sidebar never unmounts, and neither does the settings tab strip.

```jsx
// ============================================================
// src/App.jsx
// One route tree. Notice "/dashboard" is written exactly once.
// ============================================================
import { Routes, Route } from "react-router";

import PublicLayout from "./layouts/PublicLayout.jsx";
import DashboardLayout from "./layouts/DashboardLayout.jsx";
import SettingsLayout from "./layouts/SettingsLayout.jsx";

import Home from "./pages/Home.jsx";
import About from "./pages/About.jsx";
import Overview from "./pages/Overview.jsx";
import Orders from "./pages/Orders.jsx";
import OrderDetail from "./pages/OrderDetail.jsx";
import Profile from "./pages/Profile.jsx";
import Password from "./pages/Password.jsx";
import NotFound from "./pages/NotFound.jsx";

function App() {
  return (
    <Routes>
      {/* A LAYOUT ROUTE: no path, so it adds nothing to the URL.
          It exists only to give these pages a shared public frame. */}
      <Route element={<PublicLayout />}>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Route>

      {/* A normal parent route: "/dashboard" IS part of the URL */}
      <Route path="/dashboard" element={<DashboardLayout />}>
        {/* index = what shows at exactly /dashboard */}
        <Route index element={<Overview />} />

        {/* relative: no leading slash -> /dashboard/orders */}
        <Route path="orders" element={<Orders />} />
        <Route path="orders/:id" element={<OrderDetail />} />

        {/* second level of nesting, with its own layout and index */}
        <Route path="settings" element={<SettingsLayout />}>
          <Route index element={<Profile />} />
          <Route path="password" element={<Password />} />
        </Route>
      </Route>

      <Route path="*" element={<NotFound />} />
    </Routes>
  );
}

export default App;
```

```jsx
// ============================================================
// src/layouts/DashboardLayout.jsx
// The frame. It has state, so you can SEE that it never unmounts.
// ============================================================
import { useState } from "react";
import { Outlet, NavLink } from "react-router";

const link = ({ isActive }) => ({
  display: "block",
  padding: "0.4rem",
  color: isActive ? "#8B4513" : "#333",
  fontWeight: isActive ? "bold" : "normal",
  textDecoration: "none",
});

function DashboardLayout() {
  // If the layout unmounted on every navigation, this would reset to false.
  // It does not. That is the whole point of nesting.
  const [expanded, setExpanded] = useState(false);

  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      <aside style={{ width: 200, background: "#f6f1ea", padding: "1rem" }}>
        <h2>Chai Admin</h2>

        {/* `end` — without it, "Overview" stays highlighted on every sub-page */}
        <NavLink to="/dashboard" end style={link}>
          Overview
        </NavLink>
        <NavLink to="/dashboard/orders" style={link}>
          Orders
        </NavLink>
        <NavLink to="/dashboard/settings" style={link}>
          Settings
        </NavLink>

        <button onClick={() => setExpanded(!expanded)}>
          {expanded ? "▾" : "▸"} Advanced
        </button>
        {expanded && (
          <p style={{ fontSize: "0.8rem" }}>
            Now click Orders, then Settings. This stays open — the sidebar was
            never unmounted.
          </p>
        )}
      </aside>

      <main style={{ flex: 1, padding: "1rem" }}>
        {/* THE HOLE. Remove this line and every child page vanishes. */}
        <Outlet />
      </main>
    </div>
  );
}

export default DashboardLayout;
```

```jsx
// ============================================================
// src/layouts/SettingsLayout.jsx
// A layout inside a layout. Links here are RELATIVE.
// ============================================================
import { Outlet, NavLink } from "react-router";

const tab = ({ isActive }) => ({
  padding: "0.3rem 0.7rem",
  borderBottom: isActive ? "2px solid #8B4513" : "2px solid transparent",
  textDecoration: "none",
  color: "#333",
});

function SettingsLayout() {
  return (
    <section>
      <h1>Settings</h1>

      <nav style={{ display: "flex", gap: "1rem", marginBottom: "1rem" }}>
        {/* "." means this route itself -> /dashboard/settings
            `end` so it is not active on the password tab too */}
        <NavLink to="." end style={tab}>
          Profile
        </NavLink>

        {/* relative: resolves against /dashboard/settings */}
        <NavLink to="password" style={tab}>
          Password
        </NavLink>
      </nav>

      {/* This layout needs its OWN outlet for its own children */}
      <Outlet />
    </section>
  );
}

export default SettingsLayout;
```

```jsx
// ============================================================
// src/layouts/PublicLayout.jsx
// A pathless layout route: a different frame, same URLs.
// ============================================================
import { Outlet, Link } from "react-router";

function PublicLayout() {
  return (
    <div>
      <header style={{ padding: "1rem", background: "#8B4513", color: "white" }}>
        <Link to="/" style={{ color: "white", textDecoration: "none" }}>
          Chai Point
        </Link>{" "}
        · <Link to="/about" style={{ color: "white" }}>About</Link>{" "}
        · <Link to="/dashboard" style={{ color: "white" }}>Admin</Link>
      </header>

      <main style={{ padding: "1rem" }}>
        <Outlet />
      </main>

      <footer style={{ padding: "1rem", fontSize: "0.8rem" }}>
        © Chai Point
      </footer>
    </div>
  );
}

export default PublicLayout;
```

```jsx
// ============================================================
// src/pages/Orders.jsx
// A child page. It knows nothing about the sidebar — that is the point.
// ============================================================
import { Link } from "react-router";

const ORDERS = [
  { id: 101, item: "Masala Chai", qty: 2 },
  { id: 102, item: "Ginger Chai", qty: 1 },
];

function Orders() {
  return (
    <>
      <h1>Orders</h1>
      <ul>
        {ORDERS.map((o) => (
          <li key={o.id}>
            {/* relative link: from /dashboard/orders -> /dashboard/orders/101 */}
            <Link to={String(o.id)}>#{o.id}</Link> — {o.item} × {o.qty}
          </li>
        ))}
      </ul>
    </>
  );
}

export default Orders;
```

```jsx
// ============================================================
// src/pages/OrderDetail.jsx
// ============================================================
import { useParams, Link } from "react-router";

function OrderDetail() {
  const { id } = useParams();          // string, as always (note 03)

  return (
    <>
      <h1>Order #{id}</h1>
      <p>Status: on its way.</p>
      {/* ".." goes up one level -> /dashboard/orders */}
      <Link to="..">← All orders</Link>
    </>
  );
}

export default OrderDetail;
```

Small pages for the rest:

```jsx
// src/pages/Overview.jsx
function Overview() {
  return <><h1>Overview</h1><p>34 orders today.</p></>;
}
export default Overview;

// src/pages/Profile.jsx
function Profile() {
  return <><h2>Profile</h2><p>Name: Naman</p></>;
}
export default Profile;

// src/pages/Password.jsx
function Password() {
  return <><h2>Password</h2><p>Change it here.</p></>;
}
export default Password;
```

### What just happened

1. **Open `/dashboard/settings/password`.** Three components are rendered at once, nested: `DashboardLayout` → `SettingsLayout` → `Password`. One URL, three layers.
2. **Expand "Advanced" in the sidebar, then click Orders, then Settings.** It stays open. Under flat routes it would have snapped shut on every click, because the sidebar would have been destroyed and re-created.
3. **Remove `<Outlet />` from `DashboardLayout` and reload.** The sidebar still renders. Every page is gone. Nothing errors — the child simply had nowhere to go.
4. **Delete the `index` route and visit `/dashboard`.** Sidebar on the left, empty space on the right.
5. **Go to `/about`.** Completely different frame — brown header, footer. Yet `/about` is still just `/about`, because `PublicLayout` is a pathless layout route.
6. **Take `end` off the Overview `NavLink`.** It is now highlighted on Orders and Settings too, because `/dashboard` is a prefix of both.

**In simple words:** one route tree produced three layers of UI, and the outer layers kept their state while only the innermost changed.

---

## 6. How it works behind the scenes

### Matching produces a chain, not a single winner

For a flat route table, matching gives you one route. For a nested tree, it gives you a **list of matches from the outermost route to the innermost**.

```
URL: /dashboard/settings/password

match chain:
  1. <Route path="/dashboard"  element={<DashboardLayout />}>   consumed: /dashboard
  2. <Route path="settings"    element={<SettingsLayout />}>    consumed: /settings
  3. <Route path="password"    element={<Password />} />        consumed: /password
                                                                remaining: (nothing)
```

Each level consumes one part of the path and hands the rest to its children. A chain is only a valid match if the **whole** URL gets consumed.

### `<Outlet />` renders the next link in the chain

React Router then builds the element tree by wrapping each match inside the previous one:

```
<DashboardLayout>            match 1
  └── <Outlet /> renders →
      <SettingsLayout>       match 2
        └── <Outlet /> renders →
            <Password />     match 3
```

`<Outlet>` reads "which match am I?" from context and renders match + 1. That is all it does. This is also why forgetting it silently swallows everything below: the chain is fine, but nobody asked for the next link.

### Why the parent does not remount

Compare the two element trees before and after navigating from `/dashboard/orders` to `/dashboard/settings`:

```
before                          after
------                          -----
<DashboardLayout>               <DashboardLayout>     ← same component, same position
  <Outlet>                        <Outlet>
    <Orders />                      <SettingsLayout>  ← different child
                                      <Profile />
```

React's reconciliation (Chapter 07, note 01) compares element type and position. `DashboardLayout` is the same type in the same slot, so React **updates** it instead of destroying it. Its hooks, its state and its DOM nodes survive. Only the subtree that actually changed is replaced.

That is not a routing feature. It is ordinary React behaviour, unlocked by putting the layout in the right place in the tree.

### How relative paths and links are resolved

Every level of the tree knows its own full path. Relative values are resolved against it.

```
current route: /dashboard/settings

to="password"   →  /dashboard/settings/password    (append)
to="."          →  /dashboard/settings             (this route)
to=".."         →  /dashboard                      (up one)
to="../orders"  →  /dashboard/orders               (up one, then down)
to="/password"  →  /password                       (leading slash = from root)
```

The same rules apply to `<Route path>` and to `<Link to>`, which is why the leading slash mistake shows up in both.

### What an index route really is

An index route is the child that matches when the parent's path is fully consumed and nothing is left over.

```
URL /dashboard          →  parent consumes "/dashboard", remaining ""  → index child
URL /dashboard/orders   →  parent consumes "/dashboard", remaining "/orders" → path child
```

So `index` is not "the first child" and not "the default page". It is specifically "the child for an empty remainder".

```jsx
<Route index element={<Overview />} />       // ✅
<Route path="" element={<Overview />} />     // works, but `index` says the intent
```

### Layout routes with no path

A route with no `path` consumes **nothing**. It contributes an element to the chain and passes the entire remaining path to its children.

```
<Route element={<PublicLayout />}>          consumes: nothing
  <Route path="/about" element={<About />} />  consumes: /about
```

So the URL is unchanged, but `About` is wrapped in `PublicLayout`. This is the cleanest way to give one group of pages a different frame.

### `useOutletContext` in one picture

```
<Outlet context={{ user }} />        the parent puts a value in
        ↓
   React context under the hood
        ↓
const { user } = useOutletContext();  any child of THIS outlet reads it
```

It is scoped to that outlet, so two different layouts can each provide their own value without clashing.

**In simple words:** matching builds a chain of routes, each `<Outlet>` renders the next one, and because the outer components stay in the same place in the tree, React keeps them mounted.

---

## 7. Comparison with alternatives (table)

### Ways to share a layout

| Approach | Written once? | Stays mounted? | Verdict |
|---|---|---|---|
| Copy the sidebar into every page | ❌ | ❌ | duplication, and it flickers |
| Wrap each element: `element={<Layout><Orders/></Layout>}` | ❌ | ❌ | still repeated, still remounts |
| Put the layout above `<Routes>` in `App` | ✅ | ✅ | fine for **one** global frame only |
| **Nested route + `<Outlet />`** | ✅ | ✅ | the right answer for per-section frames |

The third row is what note 01 did with the nav bar — perfectly good when every page shares the same frame. Nesting is what you need the moment different sections need different frames.

### Route kinds side by side

| Written as | Adds to URL? | Has an element? | Purpose |
|---|---|---|---|
| `<Route path="x" element={…} />` | yes | yes | a normal page |
| `<Route path="x" element={…}>…</Route>` | yes | yes | a section with a layout |
| `<Route index element={…} />` | no | yes | the parent's default child |
| `<Route element={…}>…</Route>` | no | yes | a shared frame, invisible in the URL |
| `<Route path="x">…</Route>` | yes | no | grouping a URL prefix with no layout |

### Nested routes vs tabs built with `useState`

| | Nested routes | `useState` tabs |
|---|---|---|
| Each tab has a URL | ✅ | ❌ |
| Shareable / bookmarkable | ✅ | ❌ |
| Back button moves between tabs | ✅ | ❌ |
| Can be code-split per tab | ✅ | harder |
| Setup cost | a few lines of routes | almost none |

If the tabs represent real destinations, use routes. If they are a small local toggle inside one screen, `useState` is fine.

**In simple words:** put the frame above `<Routes>` if it is global, and use a nested route with `<Outlet>` if only part of the app needs it.

---

## 8. Common mistakes beginners make

**1. Forgetting `<Outlet />`**

```jsx
function Layout() {
  return <div><Sidebar /></div>;   // ❌ children have nowhere to render
}
function Layout() {
  return <div><Sidebar /><Outlet /></div>;   // ✅
}
```

No error, no warning. The layout appears and every page inside it is blank. Check this first.

**2. A leading slash on a child path**

```jsx
<Route path="/dashboard" element={<Layout />}>
  <Route path="/orders" element={<Orders />} />   // ❌ means /orders
  <Route path="orders" element={<Orders />} />    // ✅ means /dashboard/orders
</Route>
```

**3. No index route**

`/dashboard` renders the frame with an empty middle. Add `<Route index element={<Overview />} />`.

**4. Putting both `index` and `path` on one route**

```jsx
<Route index path="overview" element={<Overview />} />  // ❌ contradictory
```

An index route has no path by definition.

**5. Forgetting `end` on the parent's own `NavLink`**

```jsx
<NavLink to="/dashboard">Overview</NavLink>       // ❌ highlighted on every sub-page
<NavLink to="/dashboard" end>Overview</NavLink>   // ✅
```

Parent paths are prefixes of all their children, so this bites in every nested layout.

**6. Absolute links inside a nested layout**

```jsx
<NavLink to="/password">Password</NavLink>    // ❌ jumps to a top-level /password
<NavLink to="password">Password</NavLink>     // ✅ /dashboard/settings/password
```

**7. Giving each nested layout the same `<Outlet />` by accident**

Every layout that has children needs its **own** `<Outlet />`. Copying a layout file and forgetting to keep the outlet produces a section that renders its tab strip and nothing else.

**8. Expecting an inner layout to reset when you leave the section**

Navigate `/dashboard/settings/profile` → `/dashboard/orders` → back to settings. `SettingsLayout` did unmount that time (it left the match chain), so its state *is* fresh. But moving between two tabs inside settings keeps it. Know which boundary you crossed.

**9. Nesting `<Routes>` inside `<Routes>` instead of nesting `<Route>`**

```jsx
// ❌ works, but you lose ranking across the whole tree and it is hard to follow
<Route path="/dashboard/*" element={<DashboardRoutes />} />

// ✅ one tree
<Route path="/dashboard" element={<Layout />}>…</Route>
```

Nested `<Routes>` blocks are occasionally useful, but they are not the default answer.

**10. Rendering the whole page inside the child, frame included**

If a child page starts with `<div className="dash"><Sidebar />`, you have two sidebars. Children render **only** their own content.

**In simple words:** every layout needs its own `<Outlet />`, children never start with a slash, and the parent's `NavLink` always needs `end`.

---

## 9. Cheat sheet

```jsx
import { Routes, Route, Outlet, NavLink, useOutletContext } from "react-router";

// ---------- the route tree ----------
<Routes>
  {/* pathless layout: a frame with no URL segment */}
  <Route element={<PublicLayout />}>
    <Route path="/" element={<Home />} />
    <Route path="/about" element={<About />} />
  </Route>

  {/* parent with a path */}
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<Overview />} />          {/* exactly /dashboard */}
    <Route path="orders" element={<Orders />} />    {/* /dashboard/orders   */}
    <Route path="orders/:id" element={<Detail />} />{/* /dashboard/orders/1 */}

    <Route path="settings" element={<SettingsLayout />}>
      <Route index element={<Profile />} />
      <Route path="password" element={<Password />} />
    </Route>
  </Route>

  <Route path="*" element={<NotFound />} />
</Routes>

// ---------- the layout ----------
function DashboardLayout() {
  return (
    <div>
      <Sidebar />
      <Outlet />                 {/* required, once per layout */}
    </div>
  );
}

// ---------- passing data down ----------
<Outlet context={{ user }} />          // parent
const { user } = useOutletContext();   // child
```

Relative path rules:

```text
to="orders"      ->  append to the current route
to="."           ->  this route
to=".."          ->  up one level
to="../orders"   ->  up one, then down
to="/orders"     ->  from the root  (usually a mistake inside a layout)
```

Eight things worth memorising:

```text
1. nest <Route> to nest components
2. every layout needs its OWN <Outlet />
3. child paths have NO leading slash
4. <Route index> = the parent's own URL
5. <Route> with no path = a frame, invisible in the URL
6. parent NavLink always needs `end`
7. the layout stays mounted -> its state survives navigation
8. children render only their content, never the frame
```

**In simple words:** nest the routes, add an outlet per layout, keep paths relative, and mark the default child with `index`.

---

## 10. Revision questions (with answers)

**1. What does `<Outlet />` do?**
It marks the spot in a layout where the matched child route should be rendered. Without it the child has nowhere to appear, and the page silently comes out empty.

**2. How do you nest routes?**
Write `<Route>` elements as children of another `<Route>` instead of self-closing the parent.

**3. Why should a child path not start with `/`?**
A leading slash means "from the root", so `path="/orders"` inside `/dashboard` means the top-level `/orders`, not `/dashboard/orders`.

**4. What is an index route?**
The child that renders when the parent's path is fully matched and nothing is left over — in other words, the parent's own URL.

**5. Can a route have both `index` and `path`?**
No. An index route has no path of its own; its path is the parent's.

**6. What is a layout route (a route with no `path`)?**
A route that consumes nothing from the URL and exists only to wrap its children in a shared element. It changes the frame without changing the address.

**7. Why does the sidebar keep its state when you navigate between child pages?**
Because the layout component stays in the same position in the element tree. React reconciles it as the same component and updates it rather than unmounting it.

**8. How many components render for `/dashboard/settings/password` in the example?**
Three, nested: `DashboardLayout` → `SettingsLayout` → `Password`. Matching produces a chain, and each outlet renders the next link.

**9. What does the match chain mean?**
Each route in the chain consumes part of the path and passes the remainder to its children. The chain is a valid match only when the whole URL is consumed.

**10. What does `to=".."` resolve to?**
One level up from the current route — from `/dashboard/orders/42` it goes to `/dashboard/orders`.

**11. Why does the parent's `NavLink` need `end`?**
Because `NavLink` matches by prefix by default, and a parent path is a prefix of every child path, so it would stay highlighted on all of them.

**12. When is putting the layout above `<Routes>` good enough?**
When every page in the app shares that exact frame. Nesting is needed as soon as different sections need different frames.

**13. What is `useOutletContext` for?**
Reading a value the parent layout passed through `<Outlet context={…} />` — useful for data that belongs to that layout and its children only.

**14. Your child pages render nothing but the layout looks fine. What is wrong?**
The layout is missing its `<Outlet />`.

**15. Should tabs be nested routes or `useState`?**
Routes, if each tab is a real destination that should be linkable and reachable with the back button. `useState` only for small local toggles inside a single screen.

---

## 11. What to learn next

The dashboard now has a clean structure — but anyone who types `/dashboard` into the address bar can see it. There is no lock on the door.

Real apps need routes that check something before rendering: is this person logged in, and are they allowed here? And when the answer is no, the app has to send them to the login page, remember where they were trying to go, and take them back there afterwards — without breaking the back button.

That is the next note: guard components, redirects with `<Navigate>`, remembering the intended destination, and a proper 404 page.

➡ Next note: `05_protected_routes_and_redirects.md`

Related notes:
- [03. URL Params and Search Params](03_url_params_and_search_params.md) — dynamic children like `orders/:id` inside a layout
- [02. Links and Navigation](02_links_and_navigation.md) — `NavLink`, `end`, and relative `to` values
- [04. useContext](../02_Hooks/04_use_context.md) — the alternative to `useOutletContext` for app-wide data
- [03. Components and Props](../01_Basic/03_components_and_props.md) — `children` versus an outlet, two ways to leave a hole

⬅ [Back to chapter index](README.md)
