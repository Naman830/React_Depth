# 03_Routing — Giving Your App Real URLs

A **route** is a rule that connects a URL to a component.
React on its own has no idea what a URL is — it turns state into a screen and nothing more.
**React Router** is the library that fills the gap: it reads the address bar, renders the matching page, and updates the address bar when you navigate — all without ever reloading the page.

Read the files in number order. Every note follows the same 11-part shape.

> ⚠️ These notes use **React Router v7**. The package is now called `react-router`, not `react-router-dom`, and routes are matched by **specificity**, not top-to-bottom order. Older tutorials get both of these wrong.

---

## Notes in this chapter

| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [Setting up React Router](01_react_router_setup.md) | The hotel-lift analogy, why a `useState` page switcher breaks the back button, sharing and refresh, installing `react-router`, the three pieces (`BrowserRouter` / `Routes` / `Route`), why `element` takes JSX and not a function, the `*` catch-all, a four-page chai shop, the History API and `popstate`, why routes are ranked instead of ordered, and the server rewrite rule that stops production 404s |
| 02 | [Links and navigation](02_links_and_navigation.md) | The shopping-mall analogy, exactly what a plain `<a>` destroys, `<Link>` vs `<a>`, `<NavLink>` with `isActive` and why Home needs `end`, `useNavigate` for moves your code decides, `navigate(-1)`, push vs replace drawn as a history stack, the redirect loop `replace` prevents, `location.state` for one-off messages, and why you must never navigate during render |
| 03 | [URL params and search params](03_url_params_and_search_params.md) | The parcel-address analogy, why you cannot write one route per product, `:id` segments and `useParams`, why every param is a string, `useSearchParams` for search / filter / sort / page, why the object setter wipes your other filters, `replace` while typing, splat routes, a product list whose entire view lives in the URL, and a table of what belongs in the path, the query string, `location.state` or plain state |
| 04 | [Nested routes and layouts](04_nested_routes_and_layouts.md) | The newspaper-masthead analogy, why copying a sidebar into every page makes it flicker and lose state, nesting `<Route>` elements, `<Outlet>` as the hole in the frame, index routes, pathless layout routes, relative paths and `..`, `useOutletContext`, the match chain drawn out, and why the layout survives navigation while only the inner part changes |
| 05 | [Protected routes and redirects](05_protected_routes_and_redirects.md) | The security-desk analogy, why per-page auth checks fail open, one guard component returning `<Outlet />` or `<Navigate>`, why "loading" must be a third state, remembering the destination with `state={{ from: location }}`, why `replace` is needed in two places, `RequireGuest` and `RequireRole`, why a 403 is not a login redirect, and the honest reason a client guard is never security |
| 06 | [Route splitting and data mode](06_route_splitting_and_data_mode.md) | The restaurant analogy, why every visitor downloads the admin dashboard, `lazy()` + `<Suspense>`, where to put the boundary, preloading on hover, per-route `<title>` without react-helmet, then data mode side by side — `createBrowserRouter`, `loader`, `action`, `errorElement`, `useNavigation`, render-then-fetch vs fetch-then-render, and the honest cost of switching |

---

## The plan for this chapter

These are the topics this chapter covers, in learning order.
A row moves into the table above once its note is written.

| # | Topic | Short answer to "what is it for?" |
|---|-------|-----------------------------------|
| ~~01~~ | ~~Setting up React Router~~ | ✅ written — see the table above |
| ~~02~~ | ~~Links and navigation~~ | ✅ written — see the table above |
| ~~03~~ | ~~URL params and search params~~ | ✅ written — see the table above |
| ~~04~~ | ~~Nested routes and layouts~~ | ✅ written — see the table above |
| ~~05~~ | ~~Protected routes and redirects~~ | ✅ written — see the table above |
| ~~06~~ | ~~Route splitting and data mode~~ | ✅ written — see the table above |

---

## How a URL becomes a screen (true for every note here)

Four steps. Every feature in this chapter is one of them done well.

```
1. WATCH      <BrowserRouter> listens to the address bar
                  ↓
2. MATCH      <Routes> compares the URL against every <Route>
                  ↓
3. RANK       the most specific match wins — NOT the first one written
                  ↓
4. RENDER     the winning chain renders, outermost first, through <Outlet>
```

Step 3 is the one that surprises people. React Router does not read your routes top to bottom like a `switch`. It scores every match and renders the highest.

```
URL: /products/new

Route path        Matches?   Score
---------------   --------   ---------------------------
/                 no         —
/products/:id     yes        dynamic segment   (medium)
/products/new     yes        literal segment   (highest)  ← wins
*                 yes        catch-all         (lowest)
```

So `path="*"` still works last even if you write it first, and `/products/new` still beats `/products/:id` no matter which you declared first.

> 💡 Write `*` last anyway. Ranking is for the computer; order is for the next human reading the file.

> ⚠️ Two things always break in production and never on `localhost`: forgetting the server rewrite to `index.html` (note 01, step 8), and forgetting `replace` on a redirect (note 05).

**In simple words:** the router watches the URL, ranks every possible match, and renders the winning chain from the outside in.

---

## The three places a value can live once you have routing

This table decides more of your code than any other idea in the chapter.

| Where | Survives refresh | Shareable as a link | Back button restores it | Use it for |
|---|---|---|---|---|
| URL path — `/products/42` | ✅ | ✅ | ✅ | which record is shown |
| Query string — `?sort=price` | ✅ | ✅ | ✅ | filters, sorting, page, tab |
| `location.state` | ❌ | ❌ | ✅ | one-off messages like "saved ✅" |
| `useState` | ❌ | ❌ | ❌ | half-typed text nobody else needs |

**In simple words:** if the user would be annoyed to lose it, it belongs in the URL.

---

## Where routing fits with Chapter 02

| Chapter 02 taught | Chapter 03 adds |
|---|---|
| State decides what one screen shows | The URL decides which screen shows |
| `useState` is lost when a component unmounts | The URL survives refresh, sharing and the back button |
| `useContext` shares a value down the tree | `<Outlet>` shares a **layout** down the tree |
| `useEffect` fetches after render | A `loader` can fetch before render (note 06) |
| Custom hooks package reusable logic | `useParams`, `useNavigate` and `useSearchParams` are exactly that, written by someone else |

**In simple words:** hooks gave a component memory, and routing gives the whole app an address.

---

## Coming next
- ➡ Chapter `04_State_Management` — where shared data lives once no single page owns it

⬅ [Back to master index](../README.md)
