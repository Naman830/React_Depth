# 05. Protected Routes and Redirects

> A protected route is a route that checks something — usually "is this person logged in?" — **before** rendering, and sends the user somewhere else when the answer is no.

---

## 1. Real-life analogy

An office building with a security desk.

You walk in and head for the lift. A guard stops you and asks for your ID card. No card, no lift. Instead the guard points you to reception: "sign in first".

Three details of that interaction matter, and all three show up in code.

**The guard writes down where you were going.** You said "fourth floor, accounts". Reception notes it. When your pass is printed, they do not dump you back in the car park — they send you to the fourth floor, the place you originally wanted.

**The guard checks a card, not the safe.** Anyone determined enough could sneak past a guard. That is why the actual money is in a vault with its own lock. The guard exists to send honest people to the right place, not to be the last line of defence.

**Some doors need a different card.** A visitor pass gets you into the lobby and the canteen. It does not open the server room. That is a second question — not "who are you?" but "what are you allowed to do?".

| Office | React app |
|---|---|
| The security guard | a guard component wrapping your routes |
| Your ID card | the logged-in user in state or Context |
| "Sign in first" | `<Navigate to="/login" replace />` |
| Reception noting your destination | `state: { from: location }` |
| Being sent to the fourth floor after signing in | redirecting back after login |
| The vault with its own lock | the **server** checking every request |
| A visitor pass vs a staff pass | role-based routes |

**In simple words:** a guard component checks the pass, remembers where you were headed, and sends you to reception — but the real lock is always on the server.

---

## 2. The problem — why does this exist?

### Problem A — every URL is public by default

Note 04 built a dashboard at `/dashboard`. Nothing stops anyone typing that into the address bar. React Router does not know or care whether someone is logged in; it matches paths and renders components.

```jsx
// Anyone who knows the URL sees the admin screen
<Route path="/dashboard" element={<DashboardLayout />}>
```

### Problem B — checking inside every page repeats itself

The obvious first fix is to check at the top of each page.

```jsx
// ❌ Copy this into Overview, Orders, OrderDetail, Profile, Password…
function Orders() {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;

  return <h1>Orders</h1>;
}
```

Problems pile up fast:

- The check is repeated in every file, and every new page is one you can forget.
- Forgetting it fails **open** — the page renders for everyone. The dangerous mistake is the silent one.
- The layout still rendered. The user sees the admin sidebar flash before the redirect.
- Any data fetching at the top of those pages already started before the check ran.

### Problem C — logging in throws away the destination

Someone is sent a link to `/dashboard/orders/42`. They are logged out, so they are bounced to `/login`. They sign in successfully… and land on `/dashboard`. Or `/`. Not on order 42.

They now have to navigate back to the thing they were sent, and the original link — the whole reason they opened the app — was wasted.

### Problem D — the redirect loop

Say you send the user to `/dashboard` after login, without `replace`:

```
history: [ /dashboard, /login, /dashboard ]
                                    ↑ user is here, logged in

user presses BACK      -> /login
/login sees a logged-in user, redirects forward -> /dashboard
user presses BACK      -> /login
                       -> /dashboard
```

The back button is now a trampoline. This is Problem D from note 02, and protected routes are where it actually happens to people.

### Problem E — "logged in?" is not known instantly

On a page refresh, your app usually has to *check* whether there is a valid session — read a token from storage, ask the server whether it is still valid. That takes time.

```jsx
const { user } = useAuth();      // null for the first 200ms, then the real user
if (!user) return <Navigate to="/login" replace />;   // ❌ fires during that gap
```

The result: a logged-in user refreshes `/dashboard` and gets thrown to the login screen for no reason. This bug appears only after you add real authentication, which is why it surprises people so late.

### What we actually want

- One place that protects a whole section, so new pages are protected by default.
- Nothing of the protected UI renders before the check.
- The intended destination is remembered and used after login.
- The back button stays sane.
- "Still checking" is a distinct state from "not logged in".

**In simple words:** we want the check to live in one place, run before anything renders, and remember where the user was going.

---

## 3. What it actually is

A protected route is not a special React Router feature. It is an ordinary component that either renders its children or renders a `<Navigate>` instead.

```jsx
function RequireAuth() {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    // Not allowed. Go to login, and remember where we were going.
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;     // Allowed. Render whatever route matched below.
}
```

That is the entire idea. Everything else in this note is detail around it.

### Why it returns `<Outlet />`

Because we use it as a **layout route** (note 04) — a route with no path that wraps children:

```jsx
<Route element={<RequireAuth />}>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<Overview />} />
    <Route path="orders" element={<Orders />} />
    <Route path="settings" element={<SettingsLayout />}>
      <Route index element={<Profile />} />
    </Route>
  </Route>
</Route>
```

Everything inside is protected, at any depth, forever. Add a page tomorrow and it is protected without you doing anything. That is the property we wanted: new routes fail **closed**.

### The three states, not two

The single most important correction to the naive version:

```
authState = "loading"   → we do not know yet   → render a spinner, decide nothing
authState = "signedOut" → we know: no user     → redirect to /login
authState = "signedIn"  → we know: user exists → render <Outlet />
```

Treating `loading` as `signedOut` is what causes Problem E.

### Remembering the destination

```jsx
// in the guard
<Navigate to="/login" state={{ from: location }} replace />

// in the login page, after a successful sign-in
const from = location.state?.from?.pathname || "/dashboard";
navigate(from, { replace: true });
```

`location` here is the whole location object, so `from.pathname` includes the path, and `from.search` even preserves the query string.

### Authorisation is a second, different check

"Are you logged in?" and "are you allowed to do this?" are separate questions.

```jsx
function RequireRole({ role }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  if (user.role !== role) return <Navigate to="/403" replace />;
  return <Outlet />;
}
```

Note the different destination. A logged-out user should go to login. A logged-in user without permission should **not** — sending them to a login page they have already passed is confusing. They get a "not allowed" page.

### The honest warning

A client-side guard is a **user experience** feature, not a security feature.

Every line of your React app is downloaded to the user's machine. They can open DevTools, set `user` to whatever they like, and render the admin screen. What they cannot do is make the server hand over data they are not allowed to have — **if** the server checks. The route guard stops honest users from wandering into broken screens. The server stops everyone else.

> ⚠️ If your API returns admin data to any request that asks, no amount of routing will protect it.

**In simple words:** a guard is a component that returns either `<Outlet />` or `<Navigate>`, used as a pathless layout route so everything nested inside it is protected.

---

## 4. Syntax / setup, step by step

### Step 1 — put the user somewhere every component can read

Context (Chapter 02, note 04) is the natural fit. The whole app needs to know who is signed in.

```jsx
// src/auth/AuthContext.jsx
import { createContext, useContext, useState, useEffect } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [status, setStatus] = useState("loading"); // loading | signedIn | signedOut

  useEffect(() => {
    // Real apps: validate a token with the server here.
    const saved = localStorage.getItem("user");
    setUser(saved ? JSON.parse(saved) : null);
    setStatus(saved ? "signedIn" : "signedOut");
  }, []);

  function login(name, role = "user") {
    const u = { name, role };
    localStorage.setItem("user", JSON.stringify(u));
    setUser(u);
    setStatus("signedIn");
  }

  function logout() {
    localStorage.removeItem("user");
    setUser(null);
    setStatus("signedOut");
  }

  return (
    <AuthContext.Provider value={{ user, status, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// A reader hook that throws if used outside the provider —
// the pattern from Chapter 02, note 04.
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}
```

### Step 2 — wrap the app, inside the router

```jsx
// src/main.jsx
<BrowserRouter>
  <AuthProvider>      {/* inside, so it may use router hooks if it needs to */}
    <App />
  </AuthProvider>
</BrowserRouter>
```

### Step 3 — write the guard

```jsx
// src/auth/RequireAuth.jsx
import { Navigate, Outlet, useLocation } from "react-router";
import { useAuth } from "./AuthContext.jsx";

function RequireAuth() {
  const { status } = useAuth();
  const location = useLocation();

  // Three states, not two.
  if (status === "loading") return <p>Checking your session…</p>;

  if (status === "signedOut") {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

export default RequireAuth;
```

### Step 4 — wrap the routes you want protected

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/login" element={<Login />} />

  {/* Everything below this line requires a signed-in user */}
  <Route element={<RequireAuth />}>
    <Route path="/dashboard" element={<DashboardLayout />}>
      <Route index element={<Overview />} />
      <Route path="orders" element={<Orders />} />
    </Route>
  </Route>

  <Route path="*" element={<NotFound />} />
</Routes>
```

### Step 5 — send the user back where they wanted to go

```jsx
function Login() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();

  // Where the guard wanted to send them. Fallback for a direct visit.
  const from = location.state?.from?.pathname || "/dashboard";

  function handleSubmit(e) {
    e.preventDefault();
    login(e.target.name.value);
    navigate(from, { replace: true });   // replace: back must not return to /login
  }
  ...
}
```

### Step 6 — bounce signed-in users away from the login page

The mirror image of the guard. A logged-in user has no business on `/login`.

```jsx
function RequireGuest() {
  const { status } = useAuth();
  if (status === "loading") return <p>…</p>;
  if (status === "signedIn") return <Navigate to="/dashboard" replace />;
  return <Outlet />;
}
```

```jsx
<Route element={<RequireGuest />}>
  <Route path="/login" element={<Login />} />
  <Route path="/signup" element={<Signup />} />
</Route>
```

### Step 7 — add a role guard where you need one

```jsx
<Route element={<RequireAuth />}>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<Overview />} />

    {/* nested guard: signed in AND an admin */}
    <Route element={<RequireRole role="admin" />}>
      <Route path="staff" element={<Staff />} />
    </Route>
  </Route>
</Route>
```

Guards compose because they are just routes in the chain.

### Step 8 — hide the links too

A guard stops the page rendering. It does not hide the link that leads there. Showing a link that immediately bounces the user is bad manners.

```jsx
{user?.role === "admin" && <NavLink to="/dashboard/staff">Staff</NavLink>}
```

> ⚠️ This is presentation only. Hiding a link is not protection — the guard and the server still do the real work.

**In simple words:** put the user in Context, write one guard component that returns `<Outlet />` or `<Navigate>`, wrap your protected routes in it, and pass the attempted location along so login can send them back.

---

## 5. Full working example (with comments)

A complete, runnable login flow.

```jsx
// ============================================================
// src/main.jsx
// Router outside, auth inside.
// ============================================================
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router";
import { AuthProvider } from "./auth/AuthContext.jsx";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </StrictMode>
);
```

```jsx
// ============================================================
// src/auth/AuthContext.jsx
// A fake auth store. Swap the two functions for real API calls later.
// ============================================================
import { createContext, useContext, useState, useEffect } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  // "loading" is a real state, not a synonym for "signed out".
  const [status, setStatus] = useState("loading");

  useEffect(() => {
    // Pretend we are validating a saved session with the server.
    const timer = setTimeout(() => {
      const saved = localStorage.getItem("chai-user");
      setUser(saved ? JSON.parse(saved) : null);
      setStatus(saved ? "signedIn" : "signedOut");
    }, 400);

    return () => clearTimeout(timer);   // cleanup, as always
  }, []);

  function login(name, role) {
    const u = { name, role };
    localStorage.setItem("chai-user", JSON.stringify(u));
    setUser(u);
    setStatus("signedIn");
  }

  function logout() {
    localStorage.removeItem("chai-user");
    setUser(null);
    setStatus("signedOut");
  }

  return (
    <AuthContext.Provider value={{ user, status, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}
```

```jsx
// ============================================================
// src/auth/guards.jsx
// Three tiny components. Each one returns <Outlet /> or <Navigate />.
// ============================================================
import { Navigate, Outlet, useLocation } from "react-router";
import { useAuth } from "./AuthContext.jsx";

// 1. Signed in? If not, go to login and REMEMBER where we were going.
export function RequireAuth() {
  const { status } = useAuth();
  const location = useLocation();

  if (status === "loading") return <p style={{ padding: "1rem" }}>Checking session…</p>;

  if (status === "signedOut") {
    // state carries the full location, so ?query and #hash survive too.
    // replace, so /dashboard does not stay in history behind /login.
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

// 2. The mirror image: keep signed-in users OFF the login page.
export function RequireGuest() {
  const { status } = useAuth();

  if (status === "loading") return <p style={{ padding: "1rem" }}>Checking session…</p>;
  if (status === "signedIn") return <Navigate to="/dashboard" replace />;

  return <Outlet />;
}

// 3. Authorisation, not authentication. Different question, different redirect.
export function RequireRole({ role }) {
  const { user, status } = useAuth();

  if (status === "loading") return <p style={{ padding: "1rem" }}>Checking session…</p>;
  if (status === "signedOut") return <Navigate to="/login" replace />;

  // Signed in but not allowed -> a "no permission" page, NOT the login page.
  if (user.role !== role) return <Navigate to="/403" replace />;

  return <Outlet />;
}
```

```jsx
// ============================================================
// src/App.jsx
// Guards are pathless layout routes, so they wrap without touching the URL.
// ============================================================
import { Routes, Route } from "react-router";
import { RequireAuth, RequireGuest, RequireRole } from "./auth/guards.jsx";

import DashboardLayout from "./layouts/DashboardLayout.jsx";
import Home from "./pages/Home.jsx";
import Login from "./pages/Login.jsx";
import Overview from "./pages/Overview.jsx";
import Orders from "./pages/Orders.jsx";
import Staff from "./pages/Staff.jsx";
import Forbidden from "./pages/Forbidden.jsx";
import NotFound from "./pages/NotFound.jsx";

function App() {
  return (
    <Routes>
      {/* --- public --- */}
      <Route path="/" element={<Home />} />
      <Route path="/403" element={<Forbidden />} />

      {/* --- for signed-OUT users only --- */}
      <Route element={<RequireGuest />}>
        <Route path="/login" element={<Login />} />
      </Route>

      {/* --- everything below needs a session --- */}
      <Route element={<RequireAuth />}>
        <Route path="/dashboard" element={<DashboardLayout />}>
          <Route index element={<Overview />} />
          <Route path="orders" element={<Orders />} />

          {/* --- and this one also needs the admin role --- */}
          <Route element={<RequireRole role="admin" />}>
            <Route path="staff" element={<Staff />} />
          </Route>
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
// src/pages/Login.jsx
// Reads the remembered destination and returns the user to it.
// ============================================================
import { useLocation, useNavigate } from "react-router";
import { useAuth } from "../auth/AuthContext.jsx";

function Login() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();

  // Where the guard was trying to send us. Fallback for a direct visit to /login.
  const from = location.state?.from?.pathname || "/dashboard";

  function handleSubmit(e) {
    e.preventDefault();                       // without this the page reloads
    const form = new FormData(e.target);

    login(form.get("name"), form.get("role"));

    // replace: pressing back must NOT return to the login screen.
    navigate(from, { replace: true });
  }

  return (
    <section style={{ padding: "1rem", maxWidth: 320 }}>
      <h1>Sign in</h1>
      <p style={{ fontSize: "0.85rem", color: "#666" }}>
        Going to: <code>{from}</code>
      </p>

      <form onSubmit={handleSubmit}>
        <input name="name" placeholder="Your name" defaultValue="Naman" required />

        <select name="role" defaultValue="user">
          <option value="user">Staff member</option>
          <option value="admin">Admin</option>
        </select>

        <button type="submit">Sign in</button>
      </form>
    </section>
  );
}

export default Login;
```

```jsx
// ============================================================
// src/layouts/DashboardLayout.jsx
// Only reachable through RequireAuth, so `user` is guaranteed here.
// ============================================================
import { Outlet, NavLink, useNavigate } from "react-router";
import { useAuth } from "../auth/AuthContext.jsx";

function DashboardLayout() {
  const { user, logout } = useAuth();
  const navigate = useNavigate();

  function handleLogout() {
    logout();
    navigate("/", { replace: true });   // do not leave the dashboard in history
  }

  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      <aside style={{ width: 200, background: "#f6f1ea", padding: "1rem" }}>
        <p>Hi, {user.name}</p>

        <NavLink to="/dashboard" end>Overview</NavLink>
        <br />
        <NavLink to="/dashboard/orders">Orders</NavLink>
        <br />

        {/* Hide the link the user cannot use. Politeness, not security. */}
        {user.role === "admin" && <NavLink to="/dashboard/staff">Staff</NavLink>}

        <hr />
        <button onClick={handleLogout}>Sign out</button>
      </aside>

      <main style={{ flex: 1, padding: "1rem" }}>
        <Outlet />
      </main>
    </div>
  );
}

export default DashboardLayout;
```

```jsx
// ============================================================
// src/pages/Forbidden.jsx  and friends
// ============================================================
import { Link } from "react-router";

export function Forbidden() {
  return (
    <section style={{ padding: "1rem" }}>
      <h1>403 — not allowed</h1>
      <p>You are signed in, but this page needs a different role.</p>
      <Link to="/dashboard">Back to the dashboard</Link>
    </section>
  );
}

export default Forbidden;
```

### What just happened

Try these in order. Each one is a bug you have just avoided.

1. **Visit `/dashboard/orders` while signed out.** You land on `/login`, and the page says `Going to: /dashboard/orders`. The destination was captured.
2. **Sign in as "Staff member".** You go straight to `/dashboard/orders` — the page you originally asked for, not a generic home page.
3. **Press back.** You go to `/` (or leave the app), **not** to `/login`. That is `replace: true` in both the guard and the login submit.
4. **Now visit `/login` while signed in.** `RequireGuest` bounces you to `/dashboard`.
5. **Click "Staff" as a Staff member.** You cannot — the link is hidden. Type `/dashboard/staff` manually and you get the 403 page, not the login page.
6. **Refresh on `/dashboard`.** For about 400ms you see "Checking session…", then the dashboard. Change `status === "loading"` to fall through to the redirect and refresh again: you get thrown to `/login` even though you are signed in. That is Problem E, live.

**In simple words:** three small guard components gave the whole app a login flow that keeps the back button, the destination and the roles all correct.

---

## 6. How it works behind the scenes

### The guard is just another link in the match chain

Note 04 showed matching as a chain. A guard adds one link that contributes an element but consumes no path:

```
URL: /dashboard/orders

chain:
  1. <RequireAuth />        (pathless)   consumes nothing
  2. <DashboardLayout />    /dashboard   consumes /dashboard
  3. <Orders />             orders       consumes /orders
```

Rendering walks that chain outside-in:

```
<RequireAuth>
  ├── status === "signedOut" → returns <Navigate>  → chain stops here ✋
  └── status === "signedIn"  → returns <Outlet />  → renders link 2
                                                      └── renders link 3
```

This is why nothing protected ever appears on screen: `RequireAuth` is rendered **first**, and if it returns `<Navigate>`, `DashboardLayout` and `Orders` are never rendered at all. Their effects never run, so their data fetches never start.

Compare with the per-page check from Problem B, where the layout had already rendered before the page could object.

### What `<Navigate>` actually does

```
React renders <Navigate to="/login" replace />
        ↓
it renders nothing visible
        ↓
in an effect (after commit) it calls navigate("/login", { replace: true })
        ↓
URL changes → <Routes> re-matches → <Login /> renders
```

Because it navigates in an effect rather than during render, it does not break React's purity rule. That is the reason `<Navigate>` exists instead of you calling `navigate()` in the component body.

### Why `replace` is not optional here

Watch the history stack both ways.

```
without replace                          with replace
---------------                          ------------
[ /dashboard/orders ]                    [ /dashboard/orders ]
[ /dashboard/orders, /login ]            [ /login ]              ← overwritten
[ …, /login, /dashboard/orders ]         [ /dashboard/orders ]   ← overwritten again
              ↑                                     ↑
back → /login → guard bounces forward     back → wherever they were before 🎉
     → infinite trampoline 💥
```

You need `replace` in **two** places: in the guard's `<Navigate>`, and in the `navigate(from, …)` call after login. Miss either one and the loop comes back.

### How the destination survives

```
guard renders:  <Navigate to="/login" state={{ from: location }} replace />
                                             └─────────┬─────────┘
                                     the whole location object:
                                     { pathname: "/dashboard/orders",
                                       search: "?page=2", hash: "", … }
        ↓
stored in the browser history entry for /login
        ↓
Login reads:    location.state?.from?.pathname   → "/dashboard/orders"
```

Passing the whole `location` rather than just a string means the query string is available too. If you want to restore filters exactly, use `from.pathname + from.search`.

Remember from note 02: `state` dies on a refresh. If the user reloads the login page, `from` is gone — which is exactly why the `|| "/dashboard"` fallback is there.

### The loading state, drawn out

```
t=0ms    app boots
         status = "loading"          → guard renders a spinner
t=400ms  session check finishes
         status = "signedIn"         → guard renders <Outlet />
         OR
         status = "signedOut"        → guard renders <Navigate>
```

If you collapse `loading` into `signedOut`, the first frame at `t=0` redirects, and by the time the session check finishes the user is already sitting on the login page. Two booleans (`user` and `isLoading`) work just as well as a string — what matters is that "unknown" is representable.

### Why none of this is security

```
Browser (fully under the user's control)      Server (you control this)
----------------------------------------      -------------------------
  RequireAuth checks a value in memory   →      checks the token on
  user could edit it in DevTools                EVERY request
        ↓                                             ↓
  renders the admin screen                     returns 401 / 403
        ↓                                             ↓
  screen is empty or broken                    no data leaks ✅
```

The guard controls what **renders**. The server controls what **exists**. Only one of those is a security boundary.

**In simple words:** the guard renders before everything it protects, `<Navigate>` redirects in an effect, and `replace` keeps the history stack from becoming a loop.

---

## 7. Comparison with alternatives (table)

### Where to put the check

| Approach | Written once? | Blocks child render? | New pages safe by default? |
|---|---|---|---|
| Check at the top of every page | ❌ | ❌ (layout already rendered) | ❌ fails open |
| A `<Protected>` wrapper around each `element` | ❌ | ✅ | ❌ easy to forget |
| **A pathless guard route with `<Outlet />`** | ✅ | ✅ | ✅ |
| A `loader` in data mode (note 06) | ✅ | ✅ (runs before render) | ✅ |

The last row is genuinely better in one way: a loader can redirect *before* React renders anything at all, so there is no spinner frame. That is one of data mode's real selling points.

### `<Navigate>` vs `useNavigate` in an effect

| | `<Navigate />` | `useEffect(() => navigate(…))` |
|---|---|---|
| Where it goes | in the return value | in the component body |
| Renders the page first | no | yes, briefly — a visible flash |
| Amount of code | one line | five, plus a dependency array |
| Use for | guards and redirects | rare cases with extra async logic |

Prefer `<Navigate>` for guards. The effect version renders the protected content for one frame before redirecting, which is exactly what you were trying to avoid.

### Authentication vs authorisation

| | Authentication | Authorisation |
|---|---|---|
| Question | who are you? | what may you do? |
| Failed check means | not signed in | signed in, not permitted |
| Redirect to | `/login` | `/403` |
| Guard | `RequireAuth` | `RequireRole` |

Sending an unauthorised user to `/login` is a common and confusing bug: they log in again, succeed, and get bounced again.

**In simple words:** guard once with a pathless route, redirect with `<Navigate>`, and keep "not signed in" and "not allowed" as separate outcomes.

---

## 8. Common mistakes beginners make

**1. Treating "loading" as "signed out"**

```jsx
if (!user) return <Navigate to="/login" replace />;   // ❌ fires before the check finishes
if (status === "loading") return <Spinner />;         // ✅ handle it first
if (status === "signedOut") return <Navigate … />;
```

Every refresh throws a logged-in user out. The single most common bug in this note.

**2. Forgetting `replace` on the guard's redirect**

The protected URL stays in history behind the login page, so back bounces the user forward again.

**3. Forgetting `replace` after a successful login**

Same loop, from the other side. You need it in both places.

**4. Not passing `from`**

```jsx
<Navigate to="/login" replace />                          // ❌ destination lost
<Navigate to="/login" state={{ from: location }} replace />  // ✅
```

**5. Reading `from` without optional chaining**

```jsx
const from = location.state.from.pathname;                    // ❌ crashes on a direct visit
const from = location.state?.from?.pathname || "/dashboard";  // ✅
```

`state` is null whenever the user typed `/login` themselves or refreshed.

**6. Checking auth inside each page instead of once**

Repetitive, and it fails open when you forget. The failure mode of the guard approach is a page that is *too* protected — visible and harmless. The failure mode of per-page checks is a page that is not protected at all.

**7. Sending an unauthorised user to `/login`**

They are already signed in. Give them a 403 page.

**8. Thinking the guard is security**

```jsx
// The user can set this to anything in DevTools.
if (user.role === "admin") return <Outlet />;
```

The server must check every request. The guard only decides what renders.

**9. Hiding the link and calling it done**

Hiding a `<NavLink>` is presentation. The URL is still typeable. Hide the link *and* guard the route.

**10. Putting `<AuthProvider>` outside `<BrowserRouter>`**

```jsx
<AuthProvider><BrowserRouter>…   // ❌ AuthProvider cannot use router hooks
<BrowserRouter><AuthProvider>…   // ✅
```

If your provider ever wants `useNavigate` or `useLocation`, it must be inside the router.

**11. Redirecting from inside the component body**

```jsx
if (!user) navigate("/login");        // ❌ side effect during render → loop
if (!user) return <Navigate … />;     // ✅
```

**In simple words:** handle "loading" first, always pass `from`, always `replace`, and never mistake a guard for a lock.

---

## 9. Cheat sheet

```jsx
// ---------- the guard ----------
import { Navigate, Outlet, useLocation } from "react-router";

function RequireAuth() {
  const { status } = useAuth();
  const location = useLocation();

  if (status === "loading")  return <Spinner />;                       // 1. unknown
  if (status === "signedOut")                                          // 2. no
    return <Navigate to="/login" state={{ from: location }} replace />;
  return <Outlet />;                                                   // 3. yes
}

// ---------- using it ----------
<Routes>
  <Route path="/" element={<Home />} />

  <Route element={<RequireGuest />}>
    <Route path="/login" element={<Login />} />
  </Route>

  <Route element={<RequireAuth />}>
    <Route path="/dashboard" element={<Layout />}>
      <Route index element={<Overview />} />
      <Route element={<RequireRole role="admin" />}>
        <Route path="staff" element={<Staff />} />
      </Route>
    </Route>
  </Route>

  <Route path="*" element={<NotFound />} />
</Routes>

// ---------- coming back after login ----------
const from = location.state?.from?.pathname || "/dashboard";
navigate(from, { replace: true });
```

Nine things worth memorising:

```text
1. a guard is just a component returning <Outlet /> or <Navigate />
2. use it as a PATHLESS route so it wraps many routes at once
3. three states: loading / signedOut / signedIn
4. state={{ from: location }}   -> remember the destination
5. replace on the guard redirect AND after login
6. location.state?.from?.pathname || "/fallback"
7. not signed in -> /login   |   not allowed -> /403
8. hide the link AND guard the route
9. the server is the only real security
```

**In simple words:** one pathless guard route, three states, `from` in the state, and `replace` in both directions.

---

## 10. Revision questions (with answers)

**1. What is a protected route, in plain terms?**
A route wrapped in a component that checks a condition and renders either the child routes or a redirect.

**2. Why wrap protected routes in a pathless layout route instead of checking inside each page?**
It is written once, it runs before any protected component renders, and every route you add inside it is protected automatically — so new pages fail closed instead of open.

**3. Why does the guard return `<Outlet />`?**
Because it is used as a parent route. `<Outlet />` renders whichever child route matched below it.

**4. Why must "loading" be a separate state from "signed out"?**
Because on a refresh the app does not yet know whether there is a valid session. Treating unknown as "signed out" throws logged-in users to the login page on every reload.

**5. Why is `replace` needed on the guard's `<Navigate>`?**
Otherwise the protected URL stays in history behind the login page, so pressing back returns there, the guard fires again, and the user bounces forward — a loop.

**6. Where else is `replace` needed?**
On the `navigate(from, …)` call after a successful login, so back does not return to the login screen.

**7. How does the app remember where the user was going?**
The guard passes the current location as `state={{ from: location }}` on the redirect, and the login page reads `location.state?.from?.pathname` after signing in.

**8. Why pass the whole `location` object rather than just the pathname?**
So the query string and hash survive too, and the user returns to the exact view — including filters stored in search params.

**9. Why must you read `from` with optional chaining?**
`location.state` is null when the user typed `/login` directly or refreshed the page, so a plain property access would crash.

**10. What should happen when a signed-in user lacks the required role?**
Send them to a 403 page, not to `/login` — they are already authenticated, and logging in again would change nothing.

**11. Is a route guard a security feature?**
No. All of it runs in the user's browser and can be edited. It is a user-experience feature. The server must check permission on every request.

**12. Why is `<Navigate>` preferred over calling `navigate()` in an effect?**
The effect version renders the protected content for one frame before redirecting. `<Navigate>` renders nothing and redirects after commit, without breaking render purity.

**13. Why must `<AuthProvider>` sit inside `<BrowserRouter>`?**
So it can use router hooks like `useNavigate` if it needs to. Outside the router there is no router context.

**14. Is hiding a nav link enough to protect a page?**
No. The URL can still be typed. Hide the link for politeness, and guard the route for correctness.

**15. What is the difference between authentication and authorisation?**
Authentication answers "who are you?" and fails to `/login`. Authorisation answers "what are you allowed to do?" and fails to a 403 page.

---

## 11. What to learn next

The routing itself is now complete: URLs, links, params, layouts and guards. What is left is what happens around it.

Right now every page in the app is bundled into one JavaScript file. A visitor who only opens the home page still downloads the entire admin dashboard, the settings screens and everything they import. On a slow connection that is the difference between a site that feels instant and one that does not.

And there is the other half of React Router you have only heard mentioned: **data mode**, where routes are an array instead of JSX and each route can load its data *before* it renders — which removes the spinner frame you saw in the guard.

The final note of this chapter covers both: splitting your routes with `lazy` and `<Suspense>`, and a fair look at data mode next to the declarative mode you have been using.

➡ Next note: `06_route_splitting_and_data_mode.md`

Related notes:
- [02. Links and Navigation](02_links_and_navigation.md) — `<Navigate>`, `replace`, and `location.state`
- [04. Nested Routes and Layouts](04_nested_routes_and_layouts.md) — the pathless layout route a guard is built on
- [04. useContext](../02_Hooks/04_use_context.md) — the provider and reader-hook pattern `useAuth` uses
- [02. useEffect](../02_Hooks/02_use_effect.md) — why the session check is asynchronous, and why cleanup matters

⬅ [Back to chapter index](README.md)
