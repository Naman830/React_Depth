# 02. Links and Navigation

> `<Link>` is how a user moves between pages by clicking, `<NavLink>` also tells them which page they are on, and `useNavigate` moves them when **code** decides — like after a form is submitted.

---

## 1. Real-life analogy

Walk into a big shopping mall and find the floor directory.

Three different things on that board help you get around:

- **Signboards.** "Food Court → Floor 3". You read it, you decide, you walk. Nothing happens until *you* choose.
- **A "You are here" dot.** A red marker on the map showing your current position. It does not move you anywhere. It answers a different question: *where am I right now?*
- **The escalator at the exit of the parking ticket counter.** You did not choose it. You finished paying, and the layout carried you to the lobby. The move happened *because a task completed*, not because you clicked something.

React Router has exactly these three tools, and beginners tend to reach for the wrong one.

| Mall | React Router |
|---|---|
| A signboard you click on | `<Link to="/menu">` |
| The "You are here" red dot | `<NavLink>` and its `isActive` flag |
| Being carried along after paying | `useNavigate()` inside an event handler |
| A permanently closed corridor that pushes you elsewhere | `<Navigate to="/login" replace />` |
| Walking back out of the mall | `<a href>` — you left the building |

The important distinction is the last one in the middle: **a link is something the user chooses; `useNavigate` is something your code decides.** Almost every navigation bug comes from mixing those two up.

**In simple words:** links are for the user to click, `useNavigate` is for your code to move the user after something happens.

---

## 2. The problem — why does this exist?

### Problem A — a plain `<a>` tag destroys your app

You already met this in note 01, but it is worth seeing what actually gets thrown away.

```jsx
// ❌ Looks harmless. Is not.
<a href="/menu">Menu</a>
```

Click it and the browser does a full navigation:

```
Before the click                 After the click
----------------                 ---------------
your JS bundle is loaded         downloaded again
React tree is mounted            destroyed, rebuilt from scratch
all useState values              gone
items in the cart (in state)     gone
scroll position                  reset
a half-typed form                gone
~50ms to switch screens          ~800ms and a white flash
```

Everything that made a single-page app feel fast is undone by one wrong tag.

### Problem B — the user cannot tell which page they are on

A nav bar where all three links look identical is genuinely disorienting. You want the current page's link to be bold, or coloured, or underlined.

The obvious attempt:

```jsx
// ❌ Works, but you now maintain "which page am I on?" by hand
const [current, setCurrent] = useState("/");

<Link to="/menu" onClick={() => setCurrent("/menu")}
      style={{ fontWeight: current === "/menu" ? "bold" : "normal" }}>
  Menu
</Link>
```

Two sources of truth for one fact. Press the back button and `current` is now wrong — the URL says `/` while your state still says `/menu`. The highlight lies.

### Problem C — some navigation is not a click at all

Think about the real moments an app moves you:

- You submit the login form → go to `/dashboard`.
- You save a new order → go to `/orders/42`.
- You delete something → go back to the list.
- Your session expires → go to `/login`.
- A payment succeeds → go to `/thank-you`.

None of those are a user clicking a link. There is no link to click. The move is a **consequence** of an operation finishing.

```jsx
// ❌ You cannot render a <Link> and click it for the user
async function handleSubmit(e) {
  e.preventDefault();
  await saveOrder();
  // ...and now what? There is nothing to click.
}
```

### Problem D — redirects poison the back button

Say you redirect from `/login` to `/dashboard` after a successful login, and the redirect adds a normal history entry.

```
History stack after login:
  1. /login
  2. /dashboard   ← user is here
```

The user presses back. They land on `/login` — but they are already logged in, so your app immediately redirects them forward to `/dashboard` again. Press back, bounce forward. Press back, bounce forward. The back button is now a trap.

### What we actually want

- Clicking never reloads the app.
- The "current page" highlight is derived from the URL, never stored separately.
- Code can navigate without a click.
- We can choose whether a move should be undoable with the back button.

**In simple words:** we need a link that does not reload, a highlight that cannot go stale, and a way to navigate from code without a click.

---

## 3. What it actually is

Four tools, each with one job.

| Tool | Type | Job |
|---|---|---|
| `<Link>` | component | a clickable link that does not reload |
| `<NavLink>` | component | a `<Link>` that also knows if it is the current page |
| `useNavigate()` | hook | returns a function that navigates from code |
| `<Navigate />` | component | navigates immediately when it renders |

### `<Link>`

```jsx
import { Link } from "react-router";

<Link to="/menu">Menu</Link>
```

It renders a real `<a href="/menu">` in the HTML. That matters: right-click → "open in new tab" works, middle-click works, screen readers announce it as a link, and search engines can follow it. It only hijacks the **plain left click**.

### `<NavLink>`

The same thing, plus it tells you whether its `to` matches the current URL. Instead of a plain string, `className` and `style` can be **functions**:

```jsx
import { NavLink } from "react-router";

<NavLink
  to="/menu"
  className={({ isActive }) => (isActive ? "nav-item active" : "nav-item")}
>
  Menu
</NavLink>
```

React Router calls that function on every render and hands you an object. In declarative mode the flag you care about is `isActive`. (There is also `isPending`, but it only ever becomes `true` in data mode — see note 06.)

### `useNavigate()`

```jsx
import { useNavigate } from "react-router";

function OrderButton() {
  const navigate = useNavigate();      // ✅ called at the top level, like any hook

  async function handleClick() {
    await placeOrder();
    navigate("/orders");               // ✅ called inside an event handler
  }

  return <button onClick={handleClick}>Place order</button>;
}
```

The hook itself runs at the top level of the component. The **function it returns** is called later, inside a handler or an effect.

`navigate` accepts three shapes:

```jsx
navigate("/orders");                        // go to a path
navigate("/orders", { replace: true });     // go there, but overwrite the current entry
navigate(-1);                               // go back one entry, like the back button
navigate(1);                                // go forward one entry
```

### `<Navigate />`

A component that navigates the moment it renders. Useful when the answer to "what should this screen show?" is "not this screen".

```jsx
if (!user) return <Navigate to="/login" replace />;
```

Note 05 builds protected routes out of exactly this.

### Push versus replace — the one setting that fixes Problem D

Every navigation either **adds** an entry to the history stack or **overwrites** the current one.

```
Start:              [ /login ]
navigate("/dash")               push     ->  [ /login, /dash ]   back returns to /login 💥
navigate("/dash", {replace:true}) replace ->  [ /dash ]           back leaves the app ✅
```

Rule of thumb: if the user would be confused to land back on the page they just left, use `replace`. Logins, redirects and "the thing you were viewing no longer exists" all want `replace`.

**In simple words:** `<Link>` for clicks, `<NavLink>` for clicks that need a highlight, `useNavigate` for code, and `replace` when going back would make no sense.

---

## 4. Syntax / setup, step by step

### Step 1 — replace every internal `<a>` with `<Link>`

```jsx
import { Link } from "react-router";

<Link to="/menu">Menu</Link>                 // ✅ internal page
<a href="https://react.dev">React docs</a>   // ✅ external site — keep the <a>
```

The test is simple: **does the destination live inside this app?** Yes → `<Link>`. No → `<a>`.

### Step 2 — understand `to` — absolute vs relative

```jsx
<Link to="/menu">   // absolute: always /menu, wherever you are
<Link to="menu">    // relative: /menu from "/", but /about/menu from "/about"
<Link to="..">      // up one level
```

A leading `/` means "from the root". No leading `/` means "from wherever this route currently is". Relative links become genuinely useful with nested routes (note 04). Until then, prefer absolute — it is easier to reason about.

### Step 3 — upgrade the nav bar to `<NavLink>`

```jsx
import { NavLink } from "react-router";

<NavLink to="/menu" className={({ isActive }) => (isActive ? "active" : "")}>
  Menu
</NavLink>
```

### Step 4 — fix the Home link with `end`

Here is a trap that catches everyone exactly once.

By default `NavLink` is active when the current URL **starts with** its `to`. Since every URL starts with `/`, the Home link is *always* highlighted.

```jsx
<NavLink to="/">Home</NavLink>        // ❌ active on /menu, /about, everywhere
<NavLink to="/" end>Home</NavLink>    // ✅ active only on exactly "/"
```

`end` means "the match must end here". Use it on any link whose path is a prefix of other paths.

### Step 5 — navigate from code with `useNavigate`

```jsx
function LoginForm() {
  const navigate = useNavigate();

  async function handleSubmit(e) {
    e.preventDefault();                    // stop the browser reloading the page
    await logIn();
    navigate("/dashboard", { replace: true }); // replace, so back does not return to /login
  }

  return <form onSubmit={handleSubmit}>{/* fields */}</form>;
}
```

> ⚠️ Never call `navigate()` in the component body. That runs during render, and navigating during render is a side effect — React will warn, and you can end up in an infinite loop. Call it from an event handler or an effect.

### Step 6 — carry data along with the move

Sometimes the next screen needs to know *why* it was opened.

```jsx
navigate("/orders", { state: { justPlaced: true } });
```

On the other side:

```jsx
import { useLocation } from "react-router";

function Orders() {
  const location = useLocation();
  const justPlaced = location.state?.justPlaced;   // ?. because state may be null

  return <>{justPlaced && <p>Your order is confirmed ✅</p>}</>;
}
```

> 💡 `state` is invisible in the URL. That makes it good for one-off messages, and bad for anything the user should be able to bookmark or share. If it should survive a refresh, put it in the path or the query string (note 03) instead.

### Step 7 — go back without hardcoding a destination

```jsx
<button onClick={() => navigate(-1)}>Back</button>
```

This is exactly the browser's back button. Use it for "Cancel" buttons where you do not know where the user came from.

**In simple words:** `<Link>` for internal moves, `end` on the Home `NavLink`, `useNavigate` inside handlers only, and `replace` when the current page should not be returnable.

---

## 5. Full working example (with comments)

The chai shop again, now with a highlighted nav bar, a fake login, and a checkout that navigates on its own.

```jsx
// ============================================================
// src/App.jsx
// Routes plus a nav bar that is aware of the current page.
// ============================================================
import { Routes, Route } from "react-router";
import NavBar from "./components/NavBar.jsx";
import Home from "./pages/Home.jsx";
import Menu from "./pages/Menu.jsx";
import Checkout from "./pages/Checkout.jsx";
import Orders from "./pages/Orders.jsx";
import Login from "./pages/Login.jsx";
import NotFound from "./pages/NotFound.jsx";

function App() {
  return (
    <div>
      <NavBar />
      <main style={{ padding: "1rem" }}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/menu" element={<Menu />} />
          <Route path="/checkout" element={<Checkout />} />
          <Route path="/orders" element={<Orders />} />
          <Route path="/login" element={<Login />} />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </main>
    </div>
  );
}

export default App;
```

```jsx
// ============================================================
// src/components/NavBar.jsx
// NavLink gives us isActive for free — no state, nothing to keep in sync.
// ============================================================
import { NavLink } from "react-router";

// A plain function, not a component. It receives { isActive } from NavLink.
const linkStyle = ({ isActive }) => ({
  padding: "0.4rem 0.8rem",
  borderRadius: "6px",
  textDecoration: "none",
  color: isActive ? "white" : "#333",
  background: isActive ? "#8B4513" : "transparent",
  fontWeight: isActive ? "bold" : "normal",
});

function NavBar() {
  return (
    <nav style={{ display: "flex", gap: "0.5rem", padding: "1rem" }}>
      {/* `end` is essential here: without it, "/" matches every URL */}
      <NavLink to="/" end style={linkStyle}>
        Home
      </NavLink>

      <NavLink to="/menu" style={linkStyle}>
        Menu
      </NavLink>

      <NavLink to="/orders" style={linkStyle}>
        Orders
      </NavLink>

      {/* An external link keeps the plain <a> — we WANT to leave the app */}
      <a
        href="https://react.dev"
        target="_blank"
        rel="noreferrer"
        style={{ marginLeft: "auto" }}
      >
        React docs ↗
      </a>
    </nav>
  );
}

export default NavBar;
```

```jsx
// ============================================================
// src/pages/Menu.jsx
// A Link per item, plus a button that navigates from code.
// ============================================================
import { Link, useNavigate } from "react-router";

const ITEMS = [
  { id: 1, name: "Masala Chai", price: 20 },
  { id: 2, name: "Ginger Chai", price: 25 },
  { id: 3, name: "Elaichi Chai", price: 30 },
];

function Menu() {
  const navigate = useNavigate(); // top level, always

  function orderNow(item) {
    // Imagine an API call here.
    // The user clicked "Order now" — the MOVE is our decision, not a link.
    navigate("/checkout", { state: { item } });
  }

  return (
    <section>
      <h1>Menu</h1>
      <ul style={{ listStyle: "none", padding: 0 }}>
        {ITEMS.map((item) => (
          <li key={item.id} style={{ marginBottom: "0.5rem" }}>
            {item.name} — ₹{item.price}{" "}
            <button onClick={() => orderNow(item)}>Order now</button>
          </li>
        ))}
      </ul>

      <Link to="/">← Back to home</Link>
    </section>
  );
}

export default Menu;
```

```jsx
// ============================================================
// src/pages/Checkout.jsx
// Reads the data that came with the navigation, then redirects on success.
// ============================================================
import { useState } from "react";
import { useNavigate, useLocation, Navigate } from "react-router";

function Checkout() {
  const navigate = useNavigate();
  const location = useLocation();
  const [placing, setPlacing] = useState(false);

  // location.state is null if the user typed /checkout directly.
  const item = location.state?.item;

  // Nothing to check out -> do not render an empty screen, send them to the menu.
  // `replace` so that pressing back does not bounce them here again.
  if (!item) return <Navigate to="/menu" replace />;

  async function confirm() {
    setPlacing(true);
    await new Promise((r) => setTimeout(r, 800)); // pretend API call

    // replace: true — the user must not be able to press back into a
    // checkout screen for an order that is already placed.
    navigate("/orders", { replace: true, state: { justPlaced: item.name } });
  }

  return (
    <section>
      <h1>Checkout</h1>
      <p>
        {item.name} — ₹{item.price}
      </p>

      <button onClick={confirm} disabled={placing}>
        {placing ? "Placing…" : "Confirm order"}
      </button>{" "}
      {/* navigate(-1) = the browser back button, without knowing where we came from */}
      <button onClick={() => navigate(-1)}>Cancel</button>
    </section>
  );
}

export default Checkout;
```

```jsx
// ============================================================
// src/pages/Orders.jsx
// Shows a one-time message passed through navigation state.
// ============================================================
import { useLocation, Link } from "react-router";

function Orders() {
  const location = useLocation();
  const justPlaced = location.state?.justPlaced;

  return (
    <section>
      <h1>Your orders</h1>

      {justPlaced && (
        <p style={{ background: "#e6ffe6", padding: "0.5rem" }}>
          ✅ {justPlaced} is on its way.
        </p>
      )}

      <p>Nothing else here yet.</p>
      <Link to="/menu">Order something</Link>
    </section>
  );
}

export default Orders;
```

### What just happened

Run it and watch four specific things:

1. **The highlight follows the URL.** Click Menu — it turns brown. Press back — the highlight moves back too. You wrote no state for this. `NavLink` derives it from the URL, so it can never disagree with the screen.
2. **Home is not always highlighted.** Delete the `end` prop and reload. Now Home stays brown on every page, because every path starts with `/`. Put it back.
3. **Checkout is unreachable directly.** Type `/checkout` in the address bar. You land on `/menu` instead, because `location.state` was null and `<Navigate>` sent you away.
4. **The back button behaves after ordering.** Confirm an order. You are on `/orders`. Press back — you go to `/menu`, *not* back into checkout, because we passed `replace: true`. Remove `replace` and try again to feel the difference.

**In simple words:** links handle the clicks, `NavLink` handles the highlight, and `useNavigate` handles every move the user did not click.

---

## 6. How it works behind the scenes

### What `<Link>` actually renders

```jsx
<Link to="/menu">Menu</Link>
```

produces, in the real DOM:

```html
<a href="/menu">Menu</a>
```

A genuine anchor tag with a genuine `href`. React Router attaches an `onClick` that runs before the browser's default behaviour:

```
user clicks
      ↓
Link's onClick runs
      ↓
is this a plain left click?          -- no modifier keys, no target="_blank", left button
      ↓ yes                                         ↓ no
event.preventDefault()                    do nothing — let the browser handle it
      ↓                                             ↓
history.pushState("/menu")               opens a new tab / new window as normal
      ↓
router state updates -> <Routes> re-renders
```

That branch on the right is why Ctrl+click and middle-click still open a new tab. React Router deliberately steps aside for those. It is also why keeping the real `href` matters — without it, a new tab would open a blank page.

### How `NavLink` computes `isActive`

There is no subscription and no stored flag. On every render, `NavLink` simply compares:

```
current URL:  /menu/spicy
NavLink to:   /menu

default (no `end`):   does "/menu/spicy" start with "/menu" at a segment boundary?  -> yes -> isActive
with `end`:           is "/menu/spicy" exactly "/menu"?                             -> no  -> not active
```

Because it is recomputed from the URL every time, it is impossible for the highlight to be stale. That is the whole reason `NavLink` exists instead of you tracking it with `useState`.

```
/           to="/"        no end  -> active on EVERY page  (prefix of everything)
/           to="/"        end     -> active only on "/"
/menu       to="/menu"    no end  -> active on /menu AND /menu/anything
/menu/1     to="/menu"    end     -> not active
```

### The history stack, drawn out

This is the mental model that makes `replace` obvious.

```
push (the default)                      replace: true
------------------                      -------------
[ / ]                                   [ / ]
[ /, /menu ]        <- navigate         [ /, /menu ]        <- navigate
[ /, /menu, /checkout ]                 [ /, /menu, /checkout ]
[ /, /menu, /checkout, /orders ]        [ /, /menu, /orders ]   <- checkout overwritten
                          ↑                              ↑
back goes to /checkout 💥                back goes to /menu ✅
```

`navigate(-1)` moves the pointer left by one. `navigate(1)` moves it right. They do not delete anything.

### Why you must not call `navigate()` during render

React's render phase must be pure — no side effects (Chapter 02, note 02). Navigating changes the URL and the router's state, which is very much a side effect.

```jsx
function Bad() {
  const navigate = useNavigate();
  navigate("/login");          // ❌ runs during render
  return <p>hi</p>;
}
```

What happens: render → navigate → router state changes → re-render → navigate again → forever.

Two correct alternatives:

```jsx
// ✅ render-time redirect: use the component
if (!user) return <Navigate to="/login" replace />;

// ✅ event-time redirect: use the hook
<button onClick={() => navigate("/login")}>Sign in</button>
```

`<Navigate>` is safe because it navigates in an effect, after the render is committed.

### Is `navigate` stable across renders?

Yes. `useNavigate()` returns the same function identity between renders, so it is safe to list in a dependency array without causing loops:

```jsx
useEffect(() => {
  const id = setTimeout(() => navigate("/"), 3000);
  return () => clearTimeout(id);
}, [navigate]);       // ✅ stable — this effect will not re-run every render
```

(That is the same stability idea as `useCallback` in Chapter 02, note 07.)

### What `useLocation()` gives you

```
location = {
  pathname: "/orders",          the path
  search:   "?sort=price",      the query string  (note 03)
  hash:     "#top",             the fragment
  state:    { justPlaced: … },  invisible data passed with navigate()
  key:      "x7f3q1"            a unique id for this history entry
}
```

`state` lives in the browser's history entry, not in the URL. So it survives the back button, but **not** a refresh or a pasted link — the entry is recreated empty. Always read it with `?.` and always have a fallback.

**In simple words:** `<Link>` is a real anchor with a smart click handler, `NavLink` recomputes `isActive` from the URL every render, and `replace` overwrites the top of the history stack instead of pushing onto it.

---

## 7. Comparison with alternatives (table)

### Which navigation tool to reach for

| Tool | Triggered by | Renders anything? | Typical use |
|---|---|---|---|
| `<Link>` | the user clicking | yes, an `<a>` | menus, cards, "back to list" |
| `<NavLink>` | the user clicking | yes, an `<a>` + active state | nav bars, sidebars, tabs |
| `useNavigate()` | your code | no | after save / login / delete |
| `<Navigate />` | rendering | no | "this screen should not show — go elsewhere" |
| `<a href>` | the user clicking | yes | **only** for external sites |

### `Link` vs `NavLink`

| | `<Link>` | `<NavLink>` |
|---|---|---|
| Navigates | yes | yes |
| Knows if it is the current page | no | yes, via `isActive` |
| `className` / `style` | plain values | plain values **or** functions |
| Extra cost | none | slightly more work per render |
| Use for | links inside content | nav bars and tab strips |

Use `<Link>` unless you actually need the highlight.

### Push vs replace

| | `push` (default) | `replace: true` |
|---|---|---|
| History stack | grows by one | current entry overwritten |
| Back button returns to | the page you just left | the page **before** that |
| Right for | normal browsing | logins, redirects, post-submit screens, guards |

**In simple words:** clicks get `Link` or `NavLink`, code gets `useNavigate`, render-time redirects get `<Navigate>`, and external sites keep the plain `<a>`.

---

## 8. Common mistakes beginners make

**1. Forgetting `end` on the Home link**

```jsx
<NavLink to="/">Home</NavLink>       // ❌ highlighted on every single page
<NavLink to="/" end>Home</NavLink>   // ✅
```

Every path starts with `/`, so without `end` the prefix match always succeeds.

**2. Calling `navigate()` during render**

```jsx
function Page() {
  const navigate = useNavigate();
  if (!user) navigate("/login");   // ❌ infinite loop, React warning
  ...
}

if (!user) return <Navigate to="/login" replace />;  // ✅
```

**3. Calling the hook inside the handler**

```jsx
function handleClick() {
  const navigate = useNavigate();   // ❌ breaks the rules of hooks
  navigate("/x");
}

const navigate = useNavigate();     // ✅ top level
function handleClick() { navigate("/x"); }
```

**4. Using `<Link>` for an external site**

```jsx
<Link to="https://react.dev">Docs</Link>   // ❌ treated as an in-app path
<a href="https://react.dev">Docs</a>       // ✅
```

React Router will try to route to a path literally called `https://react.dev` and land on your 404 page.

**5. Forgetting `replace` after a login or a submit**

The back button takes the user to a login screen they have already passed, your guard bounces them forward, and back becomes unusable. Add `{ replace: true }`.

**6. Relying on `location.state` surviving a refresh**

```jsx
const item = location.state.item;    // ❌ crashes on refresh or a pasted link
const item = location.state?.item;   // ✅ and handle the missing case
```

State lives in the history entry. Reload the page and it is gone. Anything that must survive belongs in the URL.

**7. Forgetting `e.preventDefault()` in a form that then navigates**

```jsx
async function handleSubmit(e) {
  e.preventDefault();   // ❌ if omitted, the browser reloads and your navigate never runs
  await save();
  navigate("/done");
}
```

**8. Putting `onClick` navigation on a `<div>`**

```jsx
<div onClick={() => navigate("/menu")}>Menu</div>   // ❌ not focusable, not a link
<Link to="/menu">Menu</Link>                        // ✅
```

A `div` cannot be reached by keyboard, is not announced as a link, and cannot be opened in a new tab. If the user is choosing where to go, it must be a link.

**9. Building the highlight by hand with `useState`**

Two sources of truth. The back button will break it. `NavLink` derives it from the URL — use it.

**10. Passing an object to `to` incorrectly**

```jsx
<Link to={{ pathname: "/menu" }}>Menu</Link>    // ✅ valid, but rarely needed
<Link to={"/menu?sort=price"}>Menu</Link>       // ✅ simpler — a string is fine
```

**In simple words:** `end` on Home, never navigate during render, always `preventDefault` before navigating from a form, and never fake a link with a `div`.

---

## 9. Cheat sheet

```jsx
import { Link, NavLink, useNavigate, useLocation, Navigate } from "react-router";

// 1. plain link
<Link to="/menu">Menu</Link>

// 2. link that knows it is current
<NavLink to="/" end className={({ isActive }) => (isActive ? "on" : "")}>
  Home
</NavLink>

// 3. navigate from code
const navigate = useNavigate();          // top level
navigate("/orders");                     // push
navigate("/orders", { replace: true });  // replace
navigate(-1);                            // back
navigate("/orders", { state: { ok: 1 } });

// 4. read where we are / what came with us
const location = useLocation();
location.pathname;        // "/orders"
location.state?.ok;       // 1  (may be undefined)

// 5. redirect during render
if (!user) return <Navigate to="/login" replace />;
```

Eight things worth memorising:

```text
1. <Link>            internal navigation, no reload
2. <a href>          external sites ONLY
3. <NavLink> + end   highlight, and `end` stops "/" matching everything
4. useNavigate()     called at top level, used inside handlers
5. navigate(-1)      the back button
6. replace: true     after login / submit / guard redirect
7. <Navigate />      the only safe way to redirect during render
8. location.state    invisible, survives back, dies on refresh
```

Decision table:

| The move happens because… | Use |
|---|---|
| the user clicked something | `<Link>` |
| …and you want it highlighted when current | `<NavLink>` |
| an async task finished | `useNavigate()` |
| this screen must not be shown at all | `<Navigate replace />` |
| the destination is another website | `<a href>` |

**In simple words:** user chooses → `Link`; code decides → `navigate`; screen refuses to render → `<Navigate>`.

---

## 10. Revision questions (with answers)

**1. What does `<Link>` render in the actual HTML?**
A real `<a>` tag with a real `href`. It only intercepts the plain left click, which is why Ctrl+click and middle-click still open a new tab normally.

**2. Why is `<a href="/menu">` wrong inside a React app?**
It triggers a full browser navigation: the bundle is downloaded again, the React tree is destroyed, and every piece of state is lost.

**3. What is the difference between `<Link>` and `<NavLink>`?**
`<NavLink>` does everything `<Link>` does and additionally tells you whether its `to` matches the current URL, through the `isActive` flag passed to `className` and `style`.

**4. Why does the Home `NavLink` need `end`?**
By default `NavLink` matches by prefix, and every URL starts with `/`. `end` requires an exact match, so Home is highlighted only on `/`.

**5. When do you use `useNavigate` instead of a `<Link>`?**
When the user did not click a destination — the move is a consequence of code finishing, like a successful login, a saved form, or a deleted record.

**6. Why can you not call `navigate()` in the component body?**
Render must be pure. Navigating is a side effect, so it triggers a re-render, which navigates again — an infinite loop. Use `<Navigate>` for render-time redirects.

**7. What does `{ replace: true }` do?**
It overwrites the current history entry instead of adding a new one, so the back button skips over the page you are leaving.

**8. Give a concrete case where forgetting `replace` breaks the app.**
After login you push `/dashboard`. Back returns to `/login`, your guard sees a logged-in user and forwards to `/dashboard` again. Back is now stuck in a loop.

**9. What does `navigate(-1)` do?**
Exactly what the browser back button does: moves one entry back in history. Useful for Cancel buttons, where you do not know where the user came from.

**10. Where does `location.state` live, and what kills it?**
In the browser's history entry, not in the URL. It survives back and forward navigation but is lost on refresh or when the link is pasted somewhere else.

**11. Is the function returned by `useNavigate()` stable?**
Yes, so you can safely list it in a `useEffect` dependency array without causing the effect to re-run every render.

**12. How does `NavLink` avoid a stale highlight?**
It does not store anything. It recomputes the match from the current URL on every render, so there is only ever one source of truth.

**13. Why is `<div onClick={() => navigate("/x")}>` a bad link?**
It cannot be focused with a keyboard, is not announced as a link by screen readers, and cannot be opened in a new tab. Use `<Link>`.

**14. When is `<a href>` the right answer inside a React app?**
When the destination is another website — you genuinely want the browser to leave your app.

**15. What is `<Navigate />` and when does it fire?**
A component that performs a navigation when it renders. It is the safe way to redirect from render, because it navigates after the render is committed rather than during it.

---

## 11. What to learn next

Every URL so far has been a fixed string typed into a `<Route>`. That works while you have five pages. It stops working the moment you have five hundred products.

You cannot write `<Route path="/products/1">`, `/products/2`, `/products/3` forever. You need one route that says "any product id goes here", and a way for the page to read which id it got. You also need somewhere to keep things like the current filter, the sort order and the page number — values that should be shareable in a link, unlike `location.state`.

That is the next note: dynamic URL segments with `useParams`, and query strings with `useSearchParams`.

➡ Next note: `03_url_params_and_search_params.md`

Related notes:
- [01. Setting Up React Router](01_react_router_setup.md) — why `<Link>` exists and what a reload destroys
- [04. Handling Events](../01_Basic/04_handling_events.md) — `preventDefault`, which every navigating form needs
- [07. useCallback](../02_Hooks/07_use_callback.md) — why a stable `navigate` is safe in a dependency array
- [02. useEffect](../02_Hooks/02_use_effect.md) — why navigating during render breaks React's purity rule

⬅ [Back to chapter index](README.md)
