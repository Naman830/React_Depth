# 03. URL Params and Search Params

> A **URL param** is a changing piece of the path that says *which* thing to show, like the `42` in `/products/42`. A **search param** is a `?key=value` pair after the path that says *how* to show it, like `?sort=price&page=2`.

---

## 1. Real-life analogy

Think about a parcel you are sending.

The **address** on the parcel decides *which house* it goes to. "House 42, Green Park." Change one digit and it arrives at a completely different home. The address is not optional — a parcel with no address goes nowhere.

Stuck next to the address is a small label with **delivery instructions**: "Call before arriving", "Leave with the guard", "Fragile". These do not change *which* house. They change *how* the delivery happens. You can have none of them, or five of them, in any order, and the parcel still arrives.

That is exactly the split between the two kinds of URL data.

| Parcel | URL |
|---|---|
| The house address | the path — `/products/42` |
| House number `42` | the URL param, `:id` |
| The instructions label | the query string — `?sort=price&page=2` |
| "Leave with the guard" | one search param, `sort=price` |
| Missing address → undeliverable | missing param → no route match |
| Missing instructions → still delivered | missing search param → use a default |

One more detail from the analogy that matters in code: the instructions are written **on** the parcel, not whispered to the delivery person. So anyone who picks the parcel up can read them. That is why filters belong in the URL — paste the link into a chat and the other person sees the same filtered list.

**In simple words:** the path says which thing, the query string says how to show it, and both travel with the link.

---

## 2. The problem — why does this exist?

### Problem A — you cannot write a route per item

Your chai shop grows to 200 products. Each needs its own page.

```jsx
// ❌ This is not a joke, people really start doing this
<Route path="/products/1" element={<Product1 />} />
<Route path="/products/2" element={<Product2 />} />
<Route path="/products/3" element={<Product3 />} />
// ...197 more, and a new one every time stock arrives
```

Every route is identical except for one number. The page layout is the same. Only the **data** differs. What you want is one route that says "any number goes here, and tell the component which number it was".

### Problem B — passing the id through `location.state` breaks

Note 02 showed `navigate("/checkout", { state: { item } })`. It is tempting to use the same trick for a product page.

```jsx
// ❌ Works when clicked. Breaks everywhere else.
navigate("/product", { state: { id: 42 } });
```

Now:

- Refresh the page → `state` is gone → blank screen.
- Paste the link to a friend → they see a blank screen.
- Bookmark it → useless.
- Search engines → see one page called `/product`.

The information about *which* product is invisible. It is not in the URL, so it does not survive anything.

### Problem C — filters live in state and cannot be shared

A product list with a search box, a sort dropdown and pagination.

```jsx
// ❌ Every one of these is invisible to the outside world
const [query, setQuery] = useState("");
const [sort, setSort] = useState("name");
const [page, setPage] = useState(1);
```

The user carefully filters to "ginger chai, sorted by price, page 2", then tries to send that view to a colleague. They copy the URL. It says `/products`. The colleague sees an unfiltered list.

Worse, the back button now does something surprising. The user opens a product, presses back, and lands on the list with **all filters cleared** — because the list component unmounted and its state went with it.

### Problem D — everything from a URL is a string

Even once you can read `42` out of the URL, it arrives as `"42"`, not `42`.

```jsx
const id = params.id;             // "42" — a string, always
products.find((p) => p.id === id) // ❌ undefined: 42 === "42" is false
```

This one produces a bug that looks like "my data is missing" rather than "my types are wrong", so it is worth naming up front.

### What we actually want

- One route pattern that matches many URLs.
- The component can read which URL it matched.
- Filters and sorting stored somewhere the URL can carry.
- Back and forward move through filter changes sensibly.
- A clear rule for which values go where.

**In simple words:** anything the user should be able to bookmark, share or return to with the back button has to live in the URL, not in `useState`.

---

## 3. What it actually is

### URL params — a placeholder in the path

Put a colon in front of a path segment and it becomes a placeholder:

```jsx
<Route path="/products/:id" element={<ProductDetail />} />
```

`:id` matches any single segment. `/products/42`, `/products/masala-chai` and `/products/abc` all match. Whatever was in that position is handed to the component:

```jsx
import { useParams } from "react-router";

function ProductDetail() {
  const { id } = useParams();   // { id: "42" }
  ...
}
```

The name after the colon is yours to choose. `:id` in the route means `params.id` in the component. Use several if you need them:

```jsx
<Route path="/shop/:city/products/:id" element={<ProductDetail />} />
// /shop/delhi/products/42  ->  { city: "delhi", id: "42" }
```

> ⚠️ Every value in the object returned by `useParams()` is a **string**. Convert before comparing with numbers.

### Search params — everything after the `?`

```
/products?q=ginger&sort=price&page=2
          └──────────────────────────┘
                the query string
```

Three pairs: `q=ginger`, `sort=price`, `page=2`, joined by `&`. They are not part of route matching at all — `/products` matches whether or not a query string is present.

```jsx
import { useSearchParams } from "react-router";

function Products() {
  const [searchParams, setSearchParams] = useSearchParams();

  const q = searchParams.get("q") ?? "";        // "ginger", or "" if absent
  const page = Number(searchParams.get("page") ?? 1);

  ...
}
```

The shape is deliberately familiar: it looks like `useState`. A value, and a setter that updates it. The difference is that the value lives in the URL, so it survives refresh, sharing and the back button.

### `searchParams` is a browser object, not a plain object

This is the detail that surprises people. `searchParams` is a **`URLSearchParams`** instance — a standard browser API, not something React Router invented.

```jsx
searchParams.get("q");          // "ginger"  | null if missing
searchParams.getAll("tag");     // ["hot", "sweet"]  — repeated keys
searchParams.has("q");          // true / false
searchParams.toString();        // "q=ginger&sort=price"
```

There is no `searchParams.q`. It is always `.get("q")`.

### Which one should a value go in?

| Question about the value | Where it goes |
|---|---|
| Does it decide *which* record is shown? | **URL param** — `/products/:id` |
| Is it required for the page to make sense? | **URL param** |
| Is it a filter, sort, page number or tab? | **Search param** — `?sort=price` |
| Is it optional, with a sensible default? | **Search param** |
| Should it be shareable at all? | either one — both are in the URL |
| Is it a one-time message, like "saved ✅"? | `location.state` (note 02) |
| Is it a half-typed input nobody else cares about? | plain `useState` |

**In simple words:** `:id` in the path names the thing, `?key=value` after the path adjusts the view, and `useParams` / `useSearchParams` read them.

---

## 4. Syntax / setup, step by step

### Step 1 — turn a fixed route into a dynamic one

```jsx
// before
<Route path="/products/42" element={<ProductDetail />} />

// after — one route for every product
<Route path="/products/:id" element={<ProductDetail />} />
```

### Step 2 — read the param

```jsx
import { useParams } from "react-router";

function ProductDetail() {
  const { id } = useParams();      // the name matches ":id" exactly
  return <h1>Product {id}</h1>;
}
```

Spelling matters. `:id` in the route and `params.id` in the component. `:productId` gives you `params.productId`.

### Step 3 — convert the type immediately

```jsx
const { id } = useParams();
const productId = Number(id);      // ✅ convert once, at the top

const product = products.find((p) => p.id === productId);
```

Do the conversion in one place, right after reading it. Scattering `Number(id)` through the file is how you miss one.

### Step 4 — handle "no such id"

A URL param can be anything the user typed. `/products/9999` matches the route perfectly, but there is no such product.

```jsx
if (!product) {
  return (
    <section>
      <h1>Product not found</h1>
      <Link to="/products">Back to all products</Link>
    </section>
  );
}
```

> ⚠️ The route matching succeeded, so your `*` 404 route will **not** run. Handling a missing record is the page's own job.

### Step 5 — read search params with defaults

```jsx
import { useSearchParams } from "react-router";

const [searchParams, setSearchParams] = useSearchParams();

// .get() returns null when the key is absent — always supply a fallback
const q    = searchParams.get("q") ?? "";
const sort = searchParams.get("sort") ?? "name";
const page = Number(searchParams.get("page") ?? "1");
```

`??` (nullish coalescing, Chapter 01 note 05) is right here, because `""` is a legitimate value and `||` would throw it away.

### Step 6 — write search params

The setter accepts an object, and **replaces the whole query string**:

```jsx
setSearchParams({ q: "ginger", sort: "price" });   // -> ?q=ginger&sort=price
setSearchParams({ q: "ginger" });                  // -> ?q=ginger   (sort is GONE)
```

That is the number one gotcha. To change one key and keep the rest, use the function form:

```jsx
// ✅ merge into what is already there
setSearchParams((prev) => {
  prev.set("sort", "price");
  return prev;
});
```

This mirrors the updater function from `useState` (Chapter 02, note 01) — same idea, same reason.

### Step 7 — reset the page number when a filter changes

A classic bug: the user is on page 5, types a new search, and sees "no results" because the new search only has two pages.

```jsx
function handleSearch(value) {
  setSearchParams((prev) => {
    if (value) prev.set("q", value);
    else prev.delete("q");     // ✅ remove the key instead of leaving ?q=
    prev.set("page", "1");     // ✅ any filter change goes back to page 1
    return prev;
  });
}
```

### Step 8 — choose push or replace for filter changes

Typing in a search box fires on every keystroke. Pushing a history entry per letter means the user must press back seven times to undo "ginger".

```jsx
setSearchParams(next, { replace: true });   // ✅ for typing
setSearchParams(next);                      // ✅ for a deliberate click, like page 2
```

> 💡 Rule of thumb: continuous input (typing, dragging a slider) → `replace`. Discrete choices (a page number, a sort dropdown, a tab) → push, so back undoes them one at a time.

**In simple words:** `:name` in the route, `useParams()` to read it, convert to a number straight away, and use the function form of `setSearchParams` so you do not wipe the other filters.

---

## 5. Full working example (with comments)

A product list whose entire view state lives in the URL, plus a detail page.

```jsx
// ============================================================
// src/data/products.js
// A fake database. In a real app this comes from an API (Chapter 05).
// ============================================================
export const PRODUCTS = [
  { id: 1, name: "Masala Chai",  price: 20, tag: "classic" },
  { id: 2, name: "Ginger Chai",  price: 25, tag: "strong"  },
  { id: 3, name: "Elaichi Chai", price: 30, tag: "sweet"   },
  { id: 4, name: "Kadak Chai",   price: 22, tag: "strong"  },
  { id: 5, name: "Lemon Tea",    price: 18, tag: "light"   },
  { id: 6, name: "Green Tea",    price: 28, tag: "light"   },
];
```

```jsx
// ============================================================
// src/App.jsx
// One dynamic route replaces one route per product.
// ============================================================
import { Routes, Route } from "react-router";
import ProductList from "./pages/ProductList.jsx";
import ProductDetail from "./pages/ProductDetail.jsx";
import NotFound from "./pages/NotFound.jsx";

function App() {
  return (
    <Routes>
      <Route path="/" element={<ProductList />} />
      <Route path="/products" element={<ProductList />} />

      {/* ":id" matches ANY single segment: /products/1, /products/abc */}
      <Route path="/products/:id" element={<ProductDetail />} />

      <Route path="*" element={<NotFound />} />
    </Routes>
  );
}

export default App;
```

```jsx
// ============================================================
// src/pages/ProductList.jsx
// Search, filter, sort and page — all stored in the URL.
// Notice there is NOT ONE useState in this file.
// ============================================================
import { Link, useSearchParams } from "react-router";
import { PRODUCTS } from "../data/products.js";

const PER_PAGE = 3;

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  // ---- READ: every value has a default, because a key may be absent ----
  const q    = searchParams.get("q") ?? "";
  const tag  = searchParams.get("tag") ?? "all";
  const sort = searchParams.get("sort") ?? "name";
  const page = Number(searchParams.get("page") ?? "1");

  // ---- WRITE: one helper, so page always resets on a filter change ----
  // `replace` is true while typing so the back button is not flooded.
  function updateParam(key, value, { replace = false } = {}) {
    setSearchParams(
      (prev) => {
        if (value === "" || value === "all") prev.delete(key); // keep URLs clean
        else prev.set(key, value);

        if (key !== "page") prev.set("page", "1"); // any filter change -> page 1
        return prev;
      },
      { replace }
    );
  }

  // ---- DERIVE: filtering is plain JavaScript, not state ----
  const filtered = PRODUCTS
    .filter((p) => p.name.toLowerCase().includes(q.toLowerCase()))
    .filter((p) => (tag === "all" ? true : p.tag === tag))
    .sort((a, b) => (sort === "price" ? a.price - b.price : a.name.localeCompare(b.name)));

  const totalPages = Math.max(1, Math.ceil(filtered.length / PER_PAGE));
  const visible = filtered.slice((page - 1) * PER_PAGE, page * PER_PAGE);

  return (
    <section style={{ padding: "1rem" }}>
      <h1>Our teas</h1>

      {/* Typing replaces the history entry — one back press undoes the whole word */}
      <input
        value={q}
        placeholder="Search…"
        onChange={(e) => updateParam("q", e.target.value, { replace: true })}
      />

      {/* A dropdown is a deliberate choice, so it pushes a history entry */}
      <select value={tag} onChange={(e) => updateParam("tag", e.target.value)}>
        <option value="all">All tags</option>
        <option value="classic">Classic</option>
        <option value="strong">Strong</option>
        <option value="sweet">Sweet</option>
        <option value="light">Light</option>
      </select>

      <select value={sort} onChange={(e) => updateParam("sort", e.target.value)}>
        <option value="name">Sort by name</option>
        <option value="price">Sort by price</option>
      </select>

      {visible.length === 0 && <p>No tea matches that. Try clearing the filters.</p>}

      <ul>
        {visible.map((p) => (
          <li key={p.id}>
            {/* The id goes INTO the path, so the link is shareable */}
            <Link to={`/products/${p.id}`}>{p.name}</Link> — ₹{p.price}
          </li>
        ))}
      </ul>

      <div>
        <button
          disabled={page <= 1}
          onClick={() => updateParam("page", String(page - 1))}
        >
          ← Prev
        </button>
        <span> Page {page} of {totalPages} </span>
        <button
          disabled={page >= totalPages}
          onClick={() => updateParam("page", String(page + 1))}
        >
          Next →
        </button>
      </div>

      <p style={{ fontSize: "0.85rem", color: "#666" }}>
        Look at the address bar. Copy it. Open it in a new tab. You get the exact
        same view — because none of this is in <code>useState</code>.
      </p>
    </section>
  );
}

export default ProductList;
```

```jsx
// ============================================================
// src/pages/ProductDetail.jsx
// Reads :id from the path. Handles the "no such product" case itself.
// ============================================================
import { Link, useParams, useNavigate } from "react-router";
import { PRODUCTS } from "../data/products.js";

function ProductDetail() {
  // The key name must match ":id" in the route exactly.
  const { id } = useParams();
  const navigate = useNavigate();

  // ⚠️ id is the STRING "2". Convert once, right here.
  const productId = Number(id);
  const product = PRODUCTS.find((p) => p.id === productId);

  // The route matched, so the * route never runs. This is our job.
  if (!product) {
    return (
      <section style={{ padding: "1rem" }}>
        <h1>No such tea</h1>
        <p>
          There is nothing with id <code>{id}</code>.
        </p>
        <Link to="/products">See everything we have</Link>
      </section>
    );
  }

  return (
    <section style={{ padding: "1rem" }}>
      <h1>{product.name}</h1>
      <p>₹{product.price}</p>
      <p>Tag: {product.tag}</p>

      {/* navigate(-1) returns to the list WITH its filters intact,
          because those filters were in the URL of the previous entry. */}
      <button onClick={() => navigate(-1)}>← Back</button>
    </section>
  );
}

export default ProductDetail;
```

### What just happened

Six things to try, each proving a point:

1. **Type "chai" and pick "Sort by price".** The address bar becomes `/products?q=chai&sort=price&page=1`. Copy it into a new tab — identical view.
2. **Delete the search text.** The `q` key disappears from the URL entirely instead of leaving `?q=`. That is the `prev.delete(key)` line.
3. **Change a filter while on page 3.** You jump to page 1 automatically. Remove the `prev.set("page", "1")` line and watch the "no results" bug appear.
4. **Type a five-letter search, then press back once.** You get the whole word back, not one letter — because typing used `replace: true`.
5. **Click a product, then click Back.** The list returns with your filters still applied. They were in the URL of that history entry, so they came back with it.
6. **Visit `/products/9999`.** You get "No such tea", not the 404 page. The route matched; only the data was missing.

**In simple words:** the whole list view is a function of the URL, so sharing a link shares the exact screen.

---

## 6. How it works behind the scenes

### How a dynamic segment is matched

React Router splits both the pattern and the URL on `/` and compares them piece by piece.

```
pattern:  /products/:id
url:      /products/42

  "products"  vs  "products"   -> literal, must be equal      ✅
  ":id"       vs  "42"         -> starts with ':' -> capture   ✅ id = "42"

result: match, params = { id: "42" }
```

A dynamic segment matches **exactly one** segment. It does not span slashes:

```
/products/:id   vs  /products/42/reviews   -> ❌ no match (three segments vs two)
```

To match the rest of the path, use a splat:

```jsx
<Route path="/files/*" element={<FileBrowser />} />
// /files/docs/2024/report.pdf  ->  params["*"] === "docs/2024/report.pdf"
```

### Literal beats dynamic — always

Note 01 mentioned ranking. Dynamic segments are where it earns its keep.

```
URL: /products/new

Route                  Matches?   Score
--------------------   --------   --------------------------
/products/:id          yes        dynamic segment  (lower)
/products/new          yes        literal segment  (higher)  ← wins
```

So you can safely have both. `/products/new` shows your "add a product" form, and `/products/42` shows product 42, regardless of the order you wrote them in.

### Why every param is a string

A URL is text. `/products/42` contains the characters `4` and `2`; there is no type information anywhere in it. React Router hands you exactly what was in the URL, unchanged.

```jsx
const { id } = useParams();

typeof id            // "string"
id === 42            // false  💥
Number(id) === 42    // true   ✅
```

The same applies to search params. `searchParams.get("page")` is `"2"`, and `"2" + 1` is `"21"`.

### What `useSearchParams` actually returns

```
const [searchParams, setSearchParams] = useSearchParams();
                ↑                              ↑
   a URLSearchParams instance          a function that navigates
   (a standard browser class)
```

The second half is the important one: **`setSearchParams` is a navigation.** It calls the same machinery as `navigate()`. That is why it adds a history entry by default, and why `{ replace: true }` is available.

```
setSearchParams({ sort: "price" })
        ↓
build the new query string "?sort=price"
        ↓
history.pushState(current pathname + "?sort=price")
        ↓
router state updates -> components re-render
        ↓
useSearchParams returns the NEW URLSearchParams
```

### Why the object form wipes your other filters

```jsx
setSearchParams({ sort: "price" });
```

You handed it a complete description of the query string. It writes exactly that and nothing else — the same way `setUser({ name: "x" })` would drop every other key of a state object.

The function form receives the current `URLSearchParams`, lets you mutate that copy, and writes the result:

```jsx
setSearchParams((prev) => {   // prev = URLSearchParams for the CURRENT url
  prev.set("sort", "price");  // change one key
  return prev;                // everything else is untouched
});
```

### Params, search params and state — where each one lives

```
URL:  https://shop.com/products/42?sort=price#reviews
      └──────┬──────┘└─────┬─────┘└────┬────┘└───┬──┘
          origin        pathname     search     hash
                           │            │
                     useParams()  useSearchParams()

                    location.state  ← not in the URL at all
                                      (lives in the history entry)
```

| Storage | In the URL? | Survives refresh | Survives sharing | Survives back |
|---|---|---|---|---|
| URL param | yes | ✅ | ✅ | ✅ |
| Search param | yes | ✅ | ✅ | ✅ |
| `location.state` | no | ❌ | ❌ | ✅ |
| `useState` | no | ❌ | ❌ | ❌ |

That table is the real reason this note exists.

### Re-render flow when a param changes

```
user clicks <Link to="/products/7">
        ↓
URL becomes /products/7
        ↓
<Routes> re-matches -> still <ProductDetail />, but params changed
        ↓
ProductDetail re-renders; useParams() now returns { id: "7" }
```

Note what does **not** happen: the component is not unmounted and remounted. It is the same component with different props-like input. So `useState` inside it keeps its value across the change — which is occasionally what you want, and occasionally a bug (see mistake 8).

**In simple words:** matching splits the path on slashes and captures `:segments` as strings, and `setSearchParams` is really a navigation that rewrites the query string.

---

## 7. Comparison with alternatives (table)

### Where to put a value

| Value | Put it in | Why |
|---|---|---|
| Which product to show | URL param `/products/:id` | identity, required, shareable |
| Search text, sort, page, active tab | search param `?q=&sort=` | optional, shareable, has a default |
| "Order placed ✅" banner | `location.state` | one-off, should not persist or be shared |
| Text being typed into a comment box | `useState` | nobody else needs it, and it is noisy |
| Logged-in user | Context or a store (Chapter 04) | needed everywhere, never in the URL |
| Auth token | never the URL | it would end up in browser history and server logs |

### `useSearchParams` vs `useState` for a filter

| | `useSearchParams` | `useState` |
|---|---|---|
| Survives refresh | ✅ | ❌ |
| Shareable link | ✅ | ❌ |
| Back button undoes it | ✅ | ❌ |
| Value type | always string | anything |
| Writing it | rebuild the query string | direct |
| Cost | a navigation per change | none |

Use `useSearchParams` for anything the user would be annoyed to lose. Use `useState` for the rest.

### Ways to read the URL

| Hook | Gives you | Example |
|---|---|---|
| `useParams()` | dynamic path segments as an object | `{ id: "42" }` |
| `useSearchParams()` | a `URLSearchParams` + setter | `?q=chai` |
| `useLocation()` | the whole location object | `pathname`, `search`, `hash`, `state` |

**In simple words:** identity goes in the path, options go in the query string, and only private or throwaway values stay in React state.

---

## 8. Common mistakes beginners make

**1. Comparing a param to a number**

```jsx
const { id } = useParams();
products.find((p) => p.id === id);          // ❌ 42 === "42" is false
products.find((p) => p.id === Number(id));  // ✅
```

The list silently comes back empty and it looks like a data problem.

**2. Mismatched names**

```jsx
<Route path="/products/:productId" element={<Detail />} />
const { id } = useParams();     // ❌ undefined
const { productId } = useParams(); // ✅
```

**3. Wiping other filters with the object form**

```jsx
setSearchParams({ page: "2" });          // ❌ q and sort are gone
setSearchParams((p) => { p.set("page", "2"); return p; }); // ✅
```

**4. Forgetting that `.get()` returns `null`**

```jsx
const page = Number(searchParams.get("page"));  // ❌ Number(null) is 0
const page = Number(searchParams.get("page") ?? "1"); // ✅
```

**5. Reaching for a property instead of `.get()`**

```jsx
searchParams.q          // ❌ undefined — it is not a plain object
searchParams.get("q")   // ✅
```

**6. Pushing a history entry on every keystroke**

Seven letters typed means seven back presses to undo. Pass `{ replace: true }` for continuous input.

**7. Expecting the `*` route to catch a missing record**

`/products/9999` matched `/products/:id` perfectly. The route is fine; only your data lookup failed. Render your own "not found" from inside the page.

**8. Assuming the component remounts when the param changes**

```jsx
// /products/1 -> /products/2 : SAME component instance, state kept
const [qty, setQty] = useState(1);   // ⚠️ stays at whatever it was
```

If a param change should genuinely reset the screen, force a remount with a key:

```jsx
<Route path="/products/:id" element={<ProductDetail />} />
// inside ProductDetail's parent, or:
function Detail() {
  const { id } = useParams();
  return <ProductBody key={id} />;   // ✅ new key -> fresh state
}
```

**9. Leaving empty keys in the URL**

`?q=&tag=&sort=name` is ugly and shows up in analytics as distinct URLs. `delete` the key when the value is empty or default.

**10. Putting secrets in the URL**

Tokens and passwords in a query string end up in browser history, in shared links and in server access logs. Never.

**11. Not resetting the page number when a filter changes**

The user is on page 4 of 9, filters down to 1 page, and sees nothing. Always set `page=1` alongside a filter change.

**In simple words:** convert params to numbers, always use the function form of `setSearchParams`, always default `.get()`, and handle a missing record yourself.

---

## 9. Cheat sheet

```jsx
import { useParams, useSearchParams, Link } from "react-router";

// ---------- URL params ----------
<Route path="/products/:id" element={<Detail />} />
<Route path="/shop/:city/products/:id" element={<Detail />} />
<Route path="/files/*" element={<Files />} />          // splat

const { id } = useParams();          // always a string
const productId = Number(id);        // convert immediately

// ---------- search params ----------
const [searchParams, setSearchParams] = useSearchParams();

searchParams.get("q") ?? "";         // read with a default
searchParams.getAll("tag");          // repeated keys -> array
searchParams.has("q");

setSearchParams({ q: "chai" });                 // ⚠️ replaces EVERYTHING
setSearchParams((p) => { p.set("q", "chai"); return p; });   // ✅ merge
setSearchParams((p) => { p.delete("q"); return p; });        // remove a key
setSearchParams(next, { replace: true });       // no new history entry

// ---------- building links ----------
<Link to={`/products/${p.id}`}>{p.name}</Link>
<Link to="/products?sort=price&page=2">Cheapest first</Link>
```

Nine things worth memorising:

```text
1. :name in path        -> params.name in the component
2. every param          -> a STRING, convert it
3. one :segment         -> matches exactly one path segment
4. /files/*             -> params["*"] holds the rest
5. literal beats :dynamic beats *
6. .get() returns null  -> always use ?? "default"
7. object setter wipes  -> use the (prev) => {…} form
8. replace: true        -> for typing / sliders
9. route matched ≠ data found -> handle "not found" in the page
```

**In simple words:** `:id` for identity, `?key=value` for options, strings everywhere, and merge instead of replace when writing.

---

## 10. Revision questions (with answers)

**1. What is the difference between a URL param and a search param?**
A URL param is part of the path and decides *which* record is shown; it is required for the route to match. A search param comes after `?`, is optional, and adjusts *how* the page is displayed.

**2. What does `:id` do in a route path?**
It marks that segment as a placeholder. Any single segment matches, and its value is captured under the name `id`.

**3. What type does `useParams()` return values as?**
Always strings. `/products/42` gives you `"42"`, so `p.id === id` fails against a numeric id.

**4. Can a dynamic segment match more than one segment?**
No. `:id` matches exactly one. Use a splat route, `path="/files/*"`, to capture the rest of the path in `params["*"]`.

**5. If both `/products/new` and `/products/:id` exist, which wins for `/products/new`?**
`/products/new`. React Router ranks a literal segment above a dynamic one, regardless of the order you wrote them.

**6. Why does `setSearchParams({ page: "2" })` lose your other filters?**
Because you passed a complete description of the query string, so it writes exactly those keys. Use the function form to modify the existing params instead.

**7. What kind of object is `searchParams`?**
A standard browser `URLSearchParams` instance. You read from it with `.get()`, `.getAll()` and `.has()` — not with property access.

**8. What does `searchParams.get("page")` return when `page` is not in the URL?**
`null`. `Number(null)` is `0`, so always supply a default: `Number(searchParams.get("page") ?? "1")`.

**9. Why use `??` and not `||` for those defaults?**
Because `||` also replaces `""`, and an empty search box is a real, meaningful value. `??` only replaces `null` and `undefined`.

**10. When should a search param update use `replace: true`?**
For continuous input like typing or dragging a slider, so the user does not have to press back once per keystroke to undo one action.

**11. Why do filters belong in the URL rather than in `useState`?**
So the view survives a refresh, can be shared as a link, and comes back intact when the user presses back from a detail page.

**12. If `/products/9999` matches the route but no such product exists, does the `*` route render?**
No. Matching succeeded, so the `*` route is never considered. The page itself must detect the missing record and render a "not found" message.

**13. Does the component remount when only the param changes?**
No. It re-renders with new params, and any `useState` inside it keeps its value. Pass a `key` if you need a genuine reset.

**14. Where in the URL does `location.state` appear?**
Nowhere. It is stored in the browser's history entry, so it survives back and forward but not a refresh or a pasted link.

**15. Why should an auth token never go in a search param?**
Query strings are saved in browser history, copied when links are shared, and written to server access logs.

---

## 11. What to learn next

Every route so far has rendered one component that fills the whole screen. That is fine for a public site with a nav bar on top.

But look at any dashboard. There is a sidebar that stays put while the content area changes. There is a settings screen with its own sub-tabs inside it. If each route rendered a whole screen, you would have to repeat that sidebar in every single page component — and it would unmount and remount, losing its scroll position, on every navigation.

What you want is routes **inside** routes: a parent that draws the shared frame and leaves a hole for whichever child route matched.

That is the next note: nested routes, `<Outlet>`, index routes and layouts.

➡ Next note: `04_nested_routes_and_layouts.md`

Related notes:
- [02. Links and Navigation](02_links_and_navigation.md) — why `location.state` cannot replace a URL param
- [01. useState](../02_Hooks/01_use_state.md) — the updater form that `setSearchParams` copies
- [05. Conditional Rendering and Lists](../01_Basic/05_conditional_rendering_and_lists.md) — `??` vs `||`, and the empty-state pattern
- [06. useMemo](../02_Hooks/06_use_memo.md) — for when filtering a large list on every render gets slow

⬅ [Back to chapter index](README.md)
