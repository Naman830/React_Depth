# 01. Setting Up React Router

> React Router is the library that gives your single-page React app real **URLs** — so `/about` shows the About page, the browser back button works, and a link you paste in WhatsApp opens the right screen.

---

## 1. Real-life analogy

Think about a hotel with a lift.

The hotel is one building. It has one entrance. But inside there are floors: reception on floor 0, the restaurant on floor 2, the gym on floor 5. When you step into the lift and press **5**, you do not leave the building and walk into a different building. You stay inside. The lift just takes you to a different floor.

Two small details make the lift useful:

- A little **indicator** above the door always shows which floor you are on.
- Because of that indicator, you can text a friend "meet me on floor 5", and they can go straight there.

Now imagine a hotel with no indicator. You can still move between floors, but nobody can tell where you are, and you cannot tell a friend where to come. That hotel is a React app **without** a router.

| Hotel | React app |
|---|---|
| The building | your single-page app |
| A floor | a page (a component) |
| The floor number on the panel | the URL path, like `/menu` |
| The lift | React Router |
| The indicator above the door | the browser address bar |
| Texting "meet me on floor 5" | sharing or bookmarking a link |
| Walking out and into another building | a full page reload |

The whole point of React Router is the **indicator**. Moving between screens was always easy — you could do it with `useState` from day one. Keeping the address bar honest about which screen you are on is the hard part, and that is what the router does.

**In simple words:** React Router keeps the URL and the screen in agreement, without ever reloading the page.

---

## 2. The problem — why does this exist?

### What React gives you on its own

React knows how to turn state into a screen. It does **not** know what a URL is. React has never read `window.location` for you, and it never will. That is not its job.

So the first time a beginner needs two screens, they reach for the tool they already have: state.

### Problem A — the whole app becomes one giant switch

```jsx
// ❌ The "poor man's router". It works. It is also a dead end.
import { useState } from "react";

function App() {
  const [page, setPage] = useState("home");

  return (
    <div>
      <button onClick={() => setPage("home")}>Home</button>
      <button onClick={() => setPage("menu")}>Menu</button>
      <button onClick={() => setPage("about")}>About</button>

      {page === "home" && <Home />}
      {page === "menu" && <Menu />}
      {page === "about" && <About />}
    </div>
  );
}
```

This genuinely works. Click a button, the screen changes. For about ten minutes it feels like you have solved routing.

Then the problems arrive, and they arrive together.

### Problem B — the address bar never changes

Open the app and click through to About. Look at the address bar. It still says `http://localhost:5173/`.

That single fact breaks four things at once:

1. **You cannot share a screen.** Copy the URL, send it to a friend, and they land on Home. There is no way to say "look at this page".
2. **You cannot bookmark a screen.** Every bookmark is the home page.
3. **Refresh throws you out.** Press F5 on the About page and you are back on Home, because `page` was a `useState` value that lived in memory and memory is gone.
4. **Analytics see one page.** Every visitor looks like they viewed exactly one screen.

### Problem C — the back button lies

This is the one users actually complain about.

The browser back button walks through **browser history**. Your `setPage` calls never touched browser history. So the browser thinks the user has been on exactly one page this whole time.

```
What the user did              What the browser recorded
-----------------              -------------------------
opened the site                entry 1: localhost:5173/
clicked Menu                   (nothing)
clicked About                  (nothing)
pressed BACK                   leaves your site entirely 💥
```

The user expected to go from About back to Menu. Instead they were thrown off your website. On a phone, where "back" is a system gesture people use constantly, this makes an app feel broken.

### Problem D — every screen must be reachable from one file

`App.jsx` now holds the state for which page is showing. So every component that wants to navigate needs `setPage` passed down to it — through every layer in between. That is prop drilling, and it grows with every new screen.

### What we actually want

Line the wishes up:

- The URL changes when the screen changes.
- The screen changes when the URL changes — including when the user types a URL, presses back, or opens a bookmark.
- No full page reload, because a reload throws away all React state and refetches everything.
- Any component anywhere can trigger navigation without props being threaded through the tree.

That list is exactly the job description of React Router.

**In simple words:** swapping components with state is easy, but only a router keeps the URL, the back button and the screen in sync.

---

## 3. What it actually is

React Router is a **library** — not part of React. You install it separately.

At its core it does two things:

1. **Reads** the current URL from the browser.
2. **Renders** the component you said belongs to that URL.

Plus the reverse: when you navigate inside the app, it **writes** a new URL into browser history without asking the server for a new page.

### The three pieces you start with

```jsx
import { BrowserRouter, Routes, Route } from "react-router";
```

| Piece | What it is | How many |
|---|---|---|
| `<BrowserRouter>` | Watches the URL and shares it with everything inside | Exactly **one**, wrapping your whole app |
| `<Routes>` | A group of possible matches. Picks the single best one | One per place you want a screen to appear |
| `<Route>` | One rule: "this path shows this element" | One per screen |

Put together, they read almost like a sentence:

```jsx
<BrowserRouter>            {/* watch the URL */}
  <Routes>                 {/* pick exactly one of these */}
    <Route path="/" element={<Home />} />
    <Route path="/menu" element={<Menu />} />
  </Routes>
</BrowserRouter>
```

### `element` takes JSX, not a component

This trips up almost everyone once.

```jsx
<Route path="/menu" element={<Menu />} />   // ✅ JSX — angle brackets
<Route path="/menu" element={Menu} />       // ❌ the function itself
<Route path="/menu" element={<Menu() />} /> // ❌ not a thing
```

You pass `<Menu />`, which is a React **element** — a plain object describing what to render. React Router stores it and renders it when the path matches. Because it is an element, you can pass props right there: `element={<Menu spicy={true} />}`.

### Which package do I install?

React Router version 7 ships as a single package called **`react-router`**. Older tutorials say `react-router-dom` — that package still exists and still works, but in v7 it is just a thin re-export. New code should import from `react-router`.

### Three "modes", and why we use the simple one

Version 7 can be used in three ways. You will see all three names online, so it helps to know what they mean.

| Mode | What it adds | Who it is for |
|---|---|---|
| **Declarative** | Routes written as JSX, exactly as above | Learning, and most normal apps. **This chapter.** |
| **Data** | Routes written as an array, plus `loader`/`action` functions that fetch before rendering | Apps that want the router to own data fetching |
| **Framework** | Data mode plus a build setup, file conventions and server rendering | Full-stack apps, close to Next.js |

Everything you learn in declarative mode carries over. `useParams`, `<Link>`, `<Outlet>` are identical in all three. Note 06 of this chapter shows data mode side by side so you can decide later.

**In simple words:** React Router is a library that maps URL paths to components, and `BrowserRouter` / `Routes` / `Route` are the three pieces that do it.

---

## 4. Syntax / setup, step by step

### Step 1 — start from a normal React app

Nothing special. The same Vite app from Chapter 01.

```bash
npm create vite@latest chai-point -- --template react
cd chai-point
npm install
```

### Step 2 — install React Router

```bash
npm install react-router
```

One package. No configuration file, no plugin, nothing to add to `vite.config.js`.

### Step 3 — wrap the whole app in `<BrowserRouter>`

Open `src/main.jsx`. This is the file that mounts React onto the page.

```jsx
// src/main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router";
import App from "./App.jsx";
import "./index.css";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    {/* BrowserRouter must be OUTSIDE App, so everything inside can use routing */}
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

Why here and not inside `App`? Because `BrowserRouter` works like a Context provider (Chapter 02, note 04). Anything **inside** it can ask "what is the current URL?". Anything outside it cannot. Putting it at the very top means every component in your app is inside it.

> ⚠️ Exactly one `<BrowserRouter>` per app. Two routers means two independent ideas of "the current URL", and they will fight.

### Step 4 — create the page components

A "page" is not a special kind of component. It is an ordinary component that happens to fill the screen.

```jsx
// src/pages/Home.jsx
function Home() {
  return <h1>Welcome to Chai Point</h1>;
}

export default Home;
```

Keeping them in a `src/pages/` folder is only a convention, but it is a good one — later you can tell at a glance which components are screens.

### Step 5 — declare the routes

```jsx
// src/App.jsx
import { Routes, Route } from "react-router";
import Home from "./pages/Home.jsx";
import Menu from "./pages/Menu.jsx";
import About from "./pages/About.jsx";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/menu" element={<Menu />} />
      <Route path="/about" element={<About />} />
    </Routes>
  );
}

export default App;
```

Now type `localhost:5173/menu` into the address bar. The Menu page appears.

### Step 6 — add a catch-all for unknown URLs

Right now `/pizza` renders nothing at all — a blank white screen, which looks like a crash.

```jsx
<Route path="*" element={<NotFound />} />
```

The `*` path means "anything that nothing else matched". Put it last for readability (React Router does not actually care about order — see section 6 — but humans do).

### Step 7 — move between pages with `<Link>`

Navigation gets a full note next, but you need one piece of it now, because clicking is more fun than typing URLs.

```jsx
import { Link } from "react-router";

<Link to="/menu">Menu</Link>   // ✅ swaps the screen, no reload
<a href="/menu">Menu</a>       // ❌ reloads the entire app
```

`<Link>` renders a real `<a>` tag in the HTML — so it is still a proper link you can middle-click or right-click — but it intercepts the normal click and updates the URL without a network request.

### Step 8 — tell the host about client routes (before you deploy)

This step costs people hours, so learn it now even though it only bites later.

In development, Vite already handles it. In production it depends on your host. The problem: your built app is a single `index.html`. When a user opens `yoursite.com/menu` **directly**, the browser asks the server for a file at `/menu`. There is no such file, so a plain static server returns 404.

The fix is a rewrite rule: "for any path, serve `index.html`". Then React Router boots up, reads `/menu` and renders the right page.

```text
Netlify  -> create public/_redirects containing:   /*  /index.html  200
Vercel   -> works automatically for SPAs
Apache   -> a .htaccess rewrite to index.html
nginx    -> try_files $uri /index.html;
```

We come back to this in `08_Ecosystem/06_build_and_deployment.md`.

**In simple words:** install one package, wrap the app in `<BrowserRouter>` once, list your screens as `<Route>`s, and add a `*` route so unknown URLs are not blank.

---

## 5. Full working example (with comments)

A small chai shop site with four screens. Copy these files into a fresh Vite app and it runs.

```jsx
// ============================================================
// src/main.jsx
// The one and only place BrowserRouter appears.
// ============================================================
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

```jsx
// ============================================================
// src/App.jsx
// The route table. Everything outside <Routes> shows on every page.
// ============================================================
import { Routes, Route } from "react-router";
import NavBar from "./components/NavBar.jsx";
import Home from "./pages/Home.jsx";
import Menu from "./pages/Menu.jsx";
import About from "./pages/About.jsx";
import NotFound from "./pages/NotFound.jsx";

function App() {
  return (
    <div className="app">
      {/* Outside <Routes>, so the nav bar survives every navigation.
          It never unmounts, so its state is never lost. */}
      <NavBar />

      <main style={{ padding: "1rem" }}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/menu" element={<Menu />} />
          <Route path="/about" element={<About />} />

          {/* Runs only when no other route matched */}
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
// Link, not <a>. The difference is a full page reload.
// ============================================================
import { Link } from "react-router";

function NavBar() {
  return (
    <nav style={{ display: "flex", gap: "1rem", padding: "1rem" }}>
      <Link to="/">Home</Link>
      <Link to="/menu">Menu</Link>
      <Link to="/about">About</Link>

      {/* Left in on purpose so you can feel the difference.
          Click it and watch the browser tab spinner: the whole app restarts. */}
      <a href="/about" style={{ marginLeft: "auto", opacity: 0.5 }}>
        About (the slow way)
      </a>
    </nav>
  );
}

export default NavBar;
```

```jsx
// ============================================================
// src/pages/Home.jsx
// A counter, to prove something important.
// ============================================================
import { useState } from "react";

function Home() {
  const [cups, setCups] = useState(0);

  return (
    <section>
      <h1>Chai Point</h1>
      <p>Hot chai, since 2011.</p>

      <button onClick={() => setCups(cups + 1)}>
        Cups ordered: {cups}
      </button>

      <p style={{ fontSize: "0.9rem", color: "#666" }}>
        Click a few times, go to Menu, come back. The count is 0 again — this
        component unmounted when you left. That is normal, and section 6
        explains why.
      </p>
    </section>
  );
}

export default Home;
```

```jsx
// ============================================================
// src/pages/Menu.jsx
// ============================================================
const ITEMS = [
  { id: 1, name: "Masala Chai", price: 20 },
  { id: 2, name: "Ginger Chai", price: 25 },
  { id: 3, name: "Elaichi Chai", price: 30 },
];

function Menu() {
  return (
    <section>
      <h1>Menu</h1>
      <ul>
        {/* key = the stable id, never the array index (Chapter 01, note 05) */}
        {ITEMS.map((item) => (
          <li key={item.id}>
            {item.name} — ₹{item.price}
          </li>
        ))}
      </ul>
    </section>
  );
}

export default Menu;
```

```jsx
// ============================================================
// src/pages/About.jsx
// ============================================================
function About() {
  return (
    <section>
      <h1>About us</h1>
      <p>One shop, three kinds of chai, no shortcuts.</p>
    </section>
  );
}

export default About;
```

```jsx
// ============================================================
// src/pages/NotFound.jsx
// Every app needs this. A blank screen looks like a crash.
// ============================================================
import { Link } from "react-router";

function NotFound() {
  return (
    <section>
      <h1>404 — no such page</h1>
      <p>We looked everywhere. There is no page at this address.</p>
      <Link to="/">Go back home</Link>
    </section>
  );
}

export default NotFound;
```

### What just happened

Run `npm run dev` and try these five things in order. Each one proves a different part of the problem list from section 2.

1. **Click "Menu".** The address bar changes to `/menu`. The tab spinner never appears — no network request was made.
2. **Press the browser back button.** You return to Home. Back and forward now work, because `<Link>` pushed a real entry into browser history.
3. **Copy the URL while on `/about` and paste it into a new tab.** The About page opens directly. The screen is shareable.
4. **Press F5 on `/menu`.** The Menu page comes back. (In dev, Vite handles this. In production you need Step 8.)
5. **Click the greyed-out "About (the slow way)" `<a>` tag.** Watch the tab. It flickers and the spinner runs — the browser threw away your entire app and downloaded it again. That is what `<Link>` saves you from.

**In simple words:** four small files turn one screen into a real website with working URLs, a working back button and shareable links.

---

## 6. How it works behind the scenes

### There is no magic — it is one browser API

Browsers expose the **History API**. It has one method that matters here:

```js
window.history.pushState({}, "", "/menu");
```

That line changes the address bar to `/menu` and adds an entry to the back-button stack — **without contacting the server at all**. No request, no reload, nothing thrown away.

That is the entire trick. React Router is a well-built wrapper around this and one event.

### The full click-to-screen path

```
You click <Link to="/menu">
        ↓
Link calls event.preventDefault()        (stops the browser's normal navigation)
        ↓
Router calls history.pushState(..., "/menu")
        ↓
address bar now reads /menu   —  no network request
        ↓
Router updates its own state: "current path is /menu"
        ↓
That state lives in Context, so every <Routes> re-renders
        ↓
<Routes> compares "/menu" against each <Route path>
        ↓
best match wins -> renders <Menu />
        ↓
React unmounts <Home />, mounts <Menu />
```

The last line explains the counter on the Home page. Leaving a route **unmounts** the component, which destroys its state, exactly as if you had deleted it from the tree — because you did.

### What happens when the user presses BACK

`pushState` does not fire any event, but the back button does. The browser fires `popstate`. `BrowserRouter` listens for it:

```
User presses BACK
        ↓
browser pops its history stack, URL becomes /
        ↓
browser fires the "popstate" event
        ↓
BrowserRouter's listener reads window.location.pathname
        ↓
sets its state -> <Routes> re-renders -> <Home /> appears
```

This is why the back button "just works" once the router is in place, and why it could never work with `useState`.

### `<Routes>` picks exactly ONE child — and it is not "first match wins"

A common belief is that routes are checked top to bottom, like a `switch`. They are not. React Router **ranks** every route by how specific it is, then renders the single highest-scoring match.

```
URL: /menu

Route path       Does it match?   Specificity score
--------------   --------------   -----------------
/                no               —
/menu            yes              high  (a literal segment)
/:slug           yes              medium (a dynamic segment)
*                yes              lowest (catch-all)
                                  ↓
                        winner: /menu
```

Two consequences worth remembering:

- Reordering your `<Route>` lines changes nothing. `*` works last even if you write it first.
- Literal beats dynamic beats catch-all. So `/menu` wins over `/:slug`, which is almost always what you want.

### Why a refresh 404s on a real host

Compare the two ways of arriving at `/menu`:

```
Clicking a <Link>                     Typing the URL / refreshing
-----------------                     ---------------------------
app is already running                nothing is running yet
pushState changes the URL             browser sends GET /menu to the server
zero requests                         server looks for a file at /menu
router renders <Menu />               there is no such file -> 404 💥
```

The server has never heard of your routes. They exist only inside JavaScript that has not loaded yet. The rewrite rule from Step 8 says "whatever the path, hand back `index.html`", which starts the app, which then reads the URL and renders the right screen.

### What `BrowserRouter` actually provides

It is a Context provider (Chapter 02, note 04) holding roughly:

```
BrowserRouter
├── location   { pathname: "/menu", search: "", hash: "" }
├── navigate   a function to change the URL
└── history    the underlying browser history object
```

`useParams`, `useNavigate`, `useLocation`, `<Link>` and `<Routes>` are all just consumers of that context. That is why forgetting `<BrowserRouter>` produces the error **"useNavigate() may be used only in the context of a Router component"** — a context read with no provider above it.

**In simple words:** the router calls `history.pushState` to change the URL silently, listens for `popstate` to catch the back button, and re-renders the best-matching route through Context.

---

## 7. Comparison with alternatives (table)

### The three router components

| Component | URL looks like | Use it when |
|---|---|---|
| `<BrowserRouter>` | `site.com/menu` | Almost always. Needs the server rewrite rule. |
| `<HashRouter>` | `site.com/#/menu` | The host cannot be configured at all (some old shared hosting, some GitHub Pages setups). Ugly URLs, worse for search engines, but zero server config. |
| `<MemoryRouter>` | *(none — kept in memory)* | Tests, React Native, Storybook. Nothing touches the address bar. |

### React Router vs the alternatives

| Option | What it is | Trade-off |
|---|---|---|
| `useState` switch | What we wrote in section 2 | Free and instant to write. No URLs, no back button, no sharing. Fine only for a tab strip *inside* one page. |
| **React Router** | A routing library you add to any React app | The default choice. You keep full control of your build and hosting. |
| Next.js App Router | A whole framework; routes come from the folder structure | Routing plus server rendering, image optimisation and more — but you adopt the entire framework. See `08_Ecosystem/02`. |
| TanStack Router | A newer alternative router | Excellent type safety and search-param handling. Much smaller community, most tutorials assume React Router. |

### Declarative vs data mode (same library)

| | Declarative (this chapter) | Data mode |
|---|---|---|
| Routes written as | JSX `<Route>` elements | a `createBrowserRouter([...])` array |
| Data fetching | in components, with `useEffect` or a query library | in `loader` functions, before the screen renders |
| Form submissions | your own `onSubmit` | `action` functions plus `<Form>` |
| Loading spinners | your own state | `useNavigation()` gives it to you |
| Best for | learning, and apps that already use TanStack Query or Redux | apps that want the router to own data too |

Start declarative. Note 06 shows the same app both ways.

**In simple words:** `BrowserRouter` for real sites, `HashRouter` only when you cannot configure the server, and `MemoryRouter` for tests.

---

## 8. Common mistakes beginners make

**1. Forgetting `<BrowserRouter>` entirely**

```jsx
// ❌ Error: useNavigate() may be used only in the context of a <Router>
createRoot(el).render(<App />);

// ✅
createRoot(el).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

Every router hook and component reads Context. No provider, no context, instant crash.

**2. Putting `<BrowserRouter>` inside a component that re-mounts**

```jsx
// ❌ Router inside a component that unmounts on some state change
function Layout() {
  return <BrowserRouter>{/* ... */}</BrowserRouter>;
}
```

If that component ever unmounts, history state resets. Keep the router at the very top of the tree, in `main.jsx`.

**3. Passing the component instead of an element**

```jsx
<Route path="/menu" element={Menu} />     // ❌ renders nothing, no error
<Route path="/menu" element={<Menu />} /> // ✅
```

Silent failure — the page is simply blank. Check this first when a route "does nothing".

**4. Using `<a href>` inside the app**

```jsx
<a href="/menu">Menu</a>       // ❌ full reload: state gone, app re-downloaded
<Link to="/menu">Menu</Link>   // ✅
```

Use a plain `<a>` **only** for links leaving your site.

**5. No `*` route**

An unknown URL renders nothing. Users see a white screen and assume the site is broken. Always add `<Route path="*" element={<NotFound />} />`.

**6. Expecting state to survive a route change**

```jsx
// The counter on Home resets when you come back. That is not a bug.
```

Leaving a route unmounts the component. If a value must outlive navigation, lift it above `<Routes>`, put it in Context, or put it in the URL.

**7. Writing routes in a "clever" order and expecting it to matter**

```jsx
// Both of these behave identically — ranking, not order, decides.
<Route path="*" element={<NotFound />} />
<Route path="/menu" element={<Menu />} />
```

Write `*` last anyway, for the next human reading the file.

**8. Deploying without the rewrite rule**

Works perfectly on `localhost`, 404s in production the moment someone refreshes a sub-page. Step 8 fixes it.

**9. Installing the wrong package**

```bash
npm install react-router-dom   # ⚠️ works, but it is the old name
npm install react-router       # ✅ v7
```

Mixing imports from both packages in one app can produce two separate contexts and very confusing bugs.

**10. Nesting a second `<BrowserRouter>` for a sub-section**

Use nested `<Routes>` or `<Outlet>` (note 04) instead. Two routers means two conflicting sources of truth for one address bar.

**In simple words:** wrap once at the top, always pass `element={<Component />}`, always use `<Link>`, and never ship without a `*` route.

---

## 9. Cheat sheet

The minimum shape of a routed app:

```jsx
// main.jsx
import { BrowserRouter } from "react-router";

createRoot(document.getElementById("root")).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);

// App.jsx
import { Routes, Route, Link } from "react-router";

function App() {
  return (
    <>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/menu">Menu</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/menu" element={<Menu />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </>
  );
}
```

Seven things worth memorising:

```text
1. npm install react-router          -> the v7 package name
2. <BrowserRouter>                   -> exactly one, at the very top
3. <Routes>                          -> renders exactly ONE child route
4. <Route path element={<X />} />    -> element takes JSX, not a function
5. path="*"                          -> the 404 catch-all
6. <Link to="/x">                    -> never <a href> inside the app
7. rewrite all paths to index.html   -> or refresh 404s in production
```

Paths at a glance:

| `path` | Matches |
|---|---|
| `/` | the home page only |
| `/menu` | exactly `/menu` |
| `/menu/*` | `/menu` and anything under it |
| `*` | anything not matched by another route |

**In simple words:** one provider, one `<Routes>` block, one `<Route>` per screen, and a `*` at the end.

---

## 10. Revision questions (with answers)

**1. Does React know what a URL is?**
No. React turns state into a screen and nothing more. Reading and writing the URL is entirely React Router's job.

**2. Why is a `useState` page switcher not good enough?**
It never touches the address bar, so the back button, refresh, bookmarks and shared links all break. The screen changes but nothing else knows about it.

**3. How many `<BrowserRouter>` components should an app have?**
Exactly one, wrapping everything. It is a Context provider, and two providers would mean two conflicting versions of "the current URL".

**4. What does `<Routes>` do?**
It looks at the current URL, compares it against all the `<Route>` children, and renders the single best match — never two at once.

**5. Are routes matched top to bottom?**
No. React Router ranks them by specificity and renders the highest-scoring match. A literal path beats a dynamic one, which beats `*`.

**6. Why must `element` be `<Menu />` and not `Menu`?**
`element` expects a React element — a description object. `Menu` is a function. Passing the function renders nothing, and there is no error message, so the page just goes blank.

**7. What is the actual difference between `<Link to="/x">` and `<a href="/x">`?**
`<Link>` cancels the browser's default navigation and calls `history.pushState`, so no request is made and React state survives. `<a>` makes the browser download and restart the whole app.

**8. Which browser API makes client-side routing possible?**
The History API — `history.pushState` changes the URL and adds a back-button entry without any network request.

**9. How does the router know the user pressed back?**
The browser fires a `popstate` event. `BrowserRouter` listens for it, reads the new `window.location`, and re-renders the matching route.

**10. Why does a component's state disappear when you navigate away and come back?**
Because leaving the route unmounts it. A remounted component starts fresh. Move the value above `<Routes>`, into Context, or into the URL if it must survive.

**11. Why does `/menu` work when clicked but 404 after a refresh on a live site?**
A click never touches the server. A refresh sends a real `GET /menu`, and there is no file at that path. The host must be told to serve `index.html` for every path.

**12. When would you use `<HashRouter>`?**
Only when you cannot configure the server. It puts the path after a `#`, which browsers never send to the server, so no rewrite rule is needed — at the cost of uglier URLs and weaker SEO.

**13. What does `path="*"` mean, and why does every app need it?**
It matches anything no other route matched. Without it an unknown URL renders a blank page that looks exactly like a crash.

**14. Which package should you install for React Router v7?**
`react-router`. The older `react-router-dom` still works but is now just a re-export, and mixing both can create two separate contexts.

**15. Where should `<BrowserRouter>` go and why there?**
In `main.jsx`, outside `<App />`, because every component that uses routing must be inside the provider — and because it must never unmount.

---

## 11. What to learn next

You now have real URLs. But so far the only way to reach them is by typing into the address bar or clicking a bare `<Link>` we barely explained.

Real navigation needs more: highlighting the link for the page you are currently on, so the user knows where they are; navigating **after** something happens, like redirecting to `/orders` once a form is submitted; and choosing whether a move should be undoable with the back button or should replace the current entry entirely.

That is the next note: `<Link>` in full, `<NavLink>` and active styling, and the `useNavigate` hook.

➡ Next note: `02_links_and_navigation.md`

Related notes:
- [04. useContext](../02_Hooks/04_use_context.md) — `<BrowserRouter>` is a provider, and every router hook is a consumer
- [05. Conditional Rendering and Lists](../01_Basic/05_conditional_rendering_and_lists.md) — the `useState` page switch that routing replaces
- [01. Setting Up the React Environment](../01_Basic/01_setUp_react_env.md) — the Vite app these routes are added to
- [02. useEffect](../02_Hooks/02_use_effect.md) — why unmounting a route destroys its state and runs cleanup

⬅ [Back to chapter index](README.md)
