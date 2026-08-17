# 05. Conditional Rendering and Lists

> **Conditional rendering** means showing a piece of UI only when some condition is true. **Rendering a list** means turning an array of data into an array of elements on screen. Both are plain JavaScript — there is no special React syntax to memorise.

---

## 1. Real-life analogy

Think about a restaurant menu board.

**Conditional rendering** is the "Sold Out" sticker. The board does not have a separate design for sold-out days. It has one design, and a rule: *if the dish is finished, cover it with a sticker.* One board, two possible looks, decided by a condition.

**Lists** are how the board gets filled. Nobody hand-paints 30 dish names. The kitchen keeps a list of today's dishes, and someone walks down that list writing one row per dish. Add a dish to the list, a new row appears. Remove one, a row disappears. The board is **generated from the data**, never written by hand.

And there is a subtle detail: each row has a small number or slot on the board. If the chef says "the item in slot 4 is sold out", the staff know exactly which sticker to move. Without those slots, they would have to guess by position — and if a dish were removed from the middle, every sticker after it would end up on the wrong row. That slot number is React's **`key`**.

**In simple words:** you describe *what the UI should look like for this data*, and React figures out the rest.

---

## 2. The problem — why does this exist?

In plain JavaScript you change the UI by giving **instructions**:

```js
// Plain JavaScript — you must tell the browser every single step
if (user.isLoggedIn) {
  loginBtn.style.display = "none";     // hide this
  profileBox.style.display = "block";  // show that
  greeting.textContent = "Hi " + user.name;
} else {
  loginBtn.style.display = "block";    // undo everything above
  profileBox.style.display = "none";
  greeting.textContent = "Please log in";
}
```

This is **imperative** code — a list of commands. The problems:

| Problem | Why it hurts |
|---------|--------------|
| You must undo every change | Forget one `display = "block"` and the UI gets stuck in a half-state |
| The number of states explodes | 3 booleans = 8 combinations you must handle by hand |
| Reading it tells you nothing | You see steps, not the final look |
| Lists are worse | Adding one item means `createElement` + `appendChild` + tracking what already exists |

React flips this. You write **declarative** code — a description of the result:

```jsx
// React — just describe the final look for this data
function Header({ user }) {
  if (user) {
    return <p>Hi {user.name}</p>;
  }
  return <button>Log in</button>;
}
```

No hiding, no showing, no undoing. You say what it should be, and React works out the difference from what is currently on screen.

The same flip applies to lists:

```jsx
// Never write 30 <li> tags. Generate them from the data.
<ul>
  {dishes.map((dish) => <li key={dish.id}>{dish.name}</li>)}
</ul>
```

**In simple words:** stop giving instructions, start describing the result.

---

## 3. What it actually is

Here is the key idea for this whole note: **there is no React syntax for `if` or `for`.** You are using ordinary JavaScript inside the `{}` braces you learned in note 02.

Recall the rule from that note: braces accept an **expression** (something that produces a value), not a **statement** (something that performs an action). That single rule explains every pattern below.

| JavaScript tool | Statement or expression? | Usable inside `{}`? |
|-----------------|--------------------------|---------------------|
| `if / else` | Statement | ❌ No — use it above the `return` |
| Ternary `? :` | Expression | ✅ Yes |
| `&&` | Expression | ✅ Yes |
| `??` | Expression | ✅ Yes |
| `for` loop | Statement | ❌ No — use it above the `return` |
| `.map()` | Expression (returns an array) | ✅ Yes |
| `.filter()` | Expression (returns an array) | ✅ Yes |

And remember what React does with each value type (note 02, rule 7):

```jsx
{"text"}      // → renders the text
{42}          // → renders 42
{null}        // → renders NOTHING
{undefined}   // → renders NOTHING
{false}       // → renders NOTHING
{true}        // → renders NOTHING
{[a, b, c]}   // → renders each item in order
```

That "renders nothing for `false`/`null`" behaviour is not an accident. It is the entire mechanism behind conditional rendering. When you write `{isOpen && <Modal />}` and `isOpen` is `false`, React receives `false` and renders nothing. There is no hidden magic.

**In simple words:** conditional rendering is JavaScript expressions plus React's habit of ignoring `false`, `null`, and `undefined`.

---

## 4. Syntax, step by step

### Pattern 1 — `if / else` above the return

The most readable option when the two branches look very different.

```jsx
function Greeting({ user }) {
  if (!user) {
    return <button>Log in</button>;      // early return — stop here
  }

  return <h1>Welcome back, {user.name}</h1>;
}
```

The `if (!user) return ...` shape is called an **early return** or **guard clause**. It handles the odd case first and lets the main path stay unindented. Use it for loading states, error states, and empty data.

```jsx
function ProfilePage({ isLoading, error, user }) {
  if (isLoading) return <p>Loading…</p>;        // guard 1
  if (error)     return <p>Something broke.</p>; // guard 2
  if (!user)     return null;                    // guard 3 — render nothing

  // Main path: by here, we know we have a user and no error
  return <h1>{user.name}</h1>;
}
```

> 💡 `return null` is the official way to render nothing. The component still exists in the tree, it just produces no DOM.

### Pattern 2 — Ternary `? :` (if/else inside JSX)

Use this when only a small part of the JSX changes.

```jsx
condition ? valueIfTrue : valueIfFalse
```

```jsx
function Status({ isOnline }) {
  return (
    <p>
      Status: {isOnline ? "Online" : "Offline"}
    </p>
  );
}
```

It works on whole elements too:

```jsx
<div>
  {isLoggedIn
    ? <Dashboard user={user} />
    : <LoginForm />}
</div>
```

And on attribute values, which is very common for styling:

```jsx
<span style={{ color: isOnline ? "green" : "gray" }}>●</span>
<div className={isActive ? "tab tab-active" : "tab"}>Home</div>
<button disabled={items.length === 0}>Checkout</button>
```

> ⚠️ Do not nest ternaries more than one level. `a ? b : c ? d : e` is technically valid and genuinely unreadable. Move that logic above the `return`.

### Pattern 3 — `&&` (if with no else)

When there is nothing to show in the false case, `&&` is shorter than a ternary with `null`.

```jsx
{hasNewMessages && <span className="dot">●</span>}
{cart.length > 0 && <button>Checkout</button>}
{user.isAdmin && <AdminPanel />}
```

How does it work? `&&` in JavaScript returns the **left** value if it is falsy, otherwise the **right** value:

```js
false && <Modal />        // → false      → React renders nothing
true  && <Modal />        // → <Modal />  → React renders it
```

**The `0` trap.** This is the single most common bug in this whole note.

```jsx
{cart.length && <p>You have items</p>}
```

When the cart is empty, `cart.length` is `0`. `0` is falsy, so `&&` returns `0` — and React **renders numbers**. A stray `0` appears on your page.

```jsx
{cart.length > 0 && <p>You have items</p>}     // ✅ forces a real boolean
{Boolean(cart.length) && <p>…</p>}             // ✅ also fine
{!!cart.length && <p>…</p>}                    // ✅ double-negation trick
```

Always make the left side a real boolean.

### Pattern 4 — `??` (nullish coalescing) for missing values

`??` gives a fallback only when the left side is `null` or `undefined`.

```jsx
<p>{user.nickname ?? "No nickname set"}</p>
<p>{user.postCount ?? 0} posts</p>
```

Compare it with `||`:

```jsx
{count || "none"}     // "" , 0 , false all fall through to "none"
{count ?? "none"}     // only null/undefined fall through — 0 stays 0
```

Use `??` when `0` or `""` are legitimate values you want to keep.

### Pattern 5 — Optional chaining `?.` for nested data

Reading `user.address.city` crashes if `address` is missing. `?.` stops safely and gives `undefined`.

```jsx
<p>{user?.address?.city ?? "Unknown city"}</p>
```

This matters a lot with data fetched from a server, where objects arrive half-empty at first.

### Pattern 6 — A variable that holds JSX

When the logic gets long, build the element above the `return` and drop it in.

```jsx
function Alert({ type, message }) {
  let icon;                       // JSX is just a value, so a variable can hold it

  if (type === "error")   icon = <span style={{ color: "red" }}>✕</span>;
  else if (type === "ok") icon = <span style={{ color: "green" }}>✓</span>;
  else                    icon = <span>ℹ</span>;

  return <p>{icon} {message}</p>;
}
```

### Pattern 7 — An object lookup instead of a long if-chain

For "one of many" cases, an object map is cleaner than five `else if`s.

```jsx
const STATUS_LABELS = {
  pending:  "⏳ Waiting for approval",
  approved: "✅ Approved",
  rejected: "❌ Rejected",
};

function OrderStatus({ status }) {
  // the ?? handles an unexpected status string
  return <p>{STATUS_LABELS[status] ?? "Unknown status"}</p>;
}
```

### Pattern 8 — Rendering a list with `.map()`

`.map()` takes an array of data and returns an array of elements. React renders arrays by rendering each item.

```jsx
const fruits = ["Apple", "Mango", "Banana"];

<ul>
  {fruits.map((fruit) => (
    <li key={fruit}>{fruit}</li>
  ))}
</ul>
```

Step by step, here is what `map` produces:

```
  ["Apple", "Mango", "Banana"]
        │
        │  .map(fruit => <li key={fruit}>{fruit}</li>)
        ▼
  [<li key="Apple">Apple</li>,
   <li key="Mango">Mango</li>,
   <li key="Banana">Banana</li>]
        │
        │  React sees an array as a child
        ▼
  <ul>
    <li>Apple</li>
    <li>Mango</li>
    <li>Banana</li>
  </ul>
```

With objects — the normal case:

```jsx
const users = [
  { id: 1, name: "Amit", role: "Dev" },
  { id: 2, name: "Sara", role: "Designer" },
];

<div>
  {users.map((user) => (
    <UserCard key={user.id} name={user.name} role={user.role} />
  ))}
</div>
```

> ⚠️ Watch your arrow-function brackets. `=> (` returns the JSX. `=> {` opens a function body, and you must then write `return` yourself — forget it and you render nothing.

```jsx
{users.map(u => <li>{u.name}</li>)}           // ✅ implicit return
{users.map(u => (<li>{u.name}</li>))}          // ✅ same, parentheses for multi-line
{users.map(u => { <li>{u.name}</li> })}        // ❌ returns undefined — blank screen
{users.map(u => { return <li>{u.name}</li> })} // ✅ explicit return
```

### Pattern 9 — `filter` before `map`

Chain them: `filter` picks which items, `map` decides how each looks.

```jsx
{products
  .filter((p) => p.inStock)                  // 1. keep only in-stock items
  .map((p) => <Product key={p.id} {...p} />) // 2. turn each into an element
}
```

Order matters for performance: filter first so `map` runs over fewer items.

### Pattern 10 — Handling the empty list

An empty array renders nothing at all, which looks like a broken page. Always handle it.

```jsx
function TaskList({ tasks }) {
  if (tasks.length === 0) {
    return <p className="empty">No tasks yet. Add one above!</p>;
  }

  return (
    <ul>
      {tasks.map((t) => <li key={t.id}>{t.text}</li>)}
    </ul>
  );
}
```

### Pattern 11 — `key`, properly

Every element produced by `map` needs a `key` prop.

```jsx
{users.map((user) => <UserCard key={user.id} user={user} />)}
```

Rules for keys:

| Rule | Detail |
|------|--------|
| Must be unique **among siblings** | Two different lists can both use `key={1}` |
| Must be **stable** | The same item keeps the same key across renders |
| Goes on the **outermost** element of the map callback | Not on a child inside it |
| Is **not** a prop you can read | `props.key` is `undefined` inside the component |
| Should not be `Math.random()` | A new key every render destroys and rebuilds the DOM |

The index-as-key question:

```jsx
{items.map((item, index) => <li key={index}>{item}</li>)}
```

This is **safe only** when all three are true: the list never reorders, items are never inserted or deleted from the middle, and the list is not filtered or sorted. Otherwise use a real id. Section 6 shows exactly what breaks.

When you need a key on a Fragment, use the long form:

```jsx
{rows.map((row) => (
  <React.Fragment key={row.id}>     {/* <>...</> cannot take a key */}
    <dt>{row.term}</dt>
    <dd>{row.definition}</dd>
  </React.Fragment>
))}
```

**In simple words:** ternary for either/or, `&&` for maybe, `map` for many, and always give each mapped item a stable `key`.

---

## 5. Full working example

A product page that uses every pattern above.

```jsx
// src/App.jsx
// A product list showing conditional rendering + list rendering together.
// No state yet — change the constants at the top and save to see it react.

const isLoading = false;      // flip to true to see the loading guard
const error = null;           // set to "Server down" to see the error guard

const user = {
  name: "Naman",
  isAdmin: true,
  nickname: null,             // deliberately missing, to show ?? in action
};

const products = [
  { id: "p1", name: "Keyboard", price: 2499, stock: 4,  tags: ["input", "usb"] },
  { id: "p2", name: "Mouse",    price: 899,  stock: 0,  tags: ["input"] },
  { id: "p3", name: "Monitor",  price: 9999, stock: 12, tags: ["display", "hdmi"] },
  { id: "p4", name: "Webcam",   price: 1799, stock: 0,  tags: [] },
];

// Object lookup — cleaner than a chain of else-ifs (Pattern 7)
const STOCK_BADGE = {
  out:  { text: "Out of stock", color: "#c00" },
  low:  { text: "Only a few left", color: "#e67e22" },
  good: { text: "In stock", color: "#2e7d32" },
};

// A small helper — plain JavaScript, no React involved
function getStockLevel(stock) {
  if (stock === 0) return "out";
  if (stock < 5)   return "low";
  return "good";
}

function ProductCard({ product, showAdminInfo }) {
  const level = getStockLevel(product.stock);
  const badge = STOCK_BADGE[level];
  const isOut = product.stock === 0;

  return (
    <li
      style={{
        border: "1px solid #ddd",
        borderRadius: "8px",
        padding: "12px",
        marginBottom: "10px",
        // ternary inside a style value (Pattern 2)
        opacity: isOut ? 0.55 : 1,
      }}
    >
      <h3 style={{ margin: "0 0 4px" }}>{product.name}</h3>

      <p style={{ margin: "0 0 6px" }}>₹{product.price}</p>

      {/* the badge comes from the lookup object */}
      <span style={{ color: badge.color, fontSize: "13px" }}>{badge.text}</span>

      {/* A NESTED list — tags inside a product inside the product list.
          Keys only need to be unique among their own siblings. */}
      {product.tags.length > 0 && (
        <ul style={{ display: "flex", gap: "6px", listStyle: "none", padding: 0 }}>
          {product.tags.map((tag) => (
            <li key={tag} style={{ background: "#eee", padding: "1px 6px" }}>
              #{tag}
            </li>
          ))}
        </ul>
      )}

      {/* && for "show only if" (Pattern 3) */}
      {showAdminInfo && (
        <p style={{ fontSize: "12px", color: "#888" }}>
          Admin: {product.stock} units in warehouse
        </p>
      )}

      {/* ternary choosing between two whole elements */}
      {isOut
        ? <button disabled>Notify me</button>
        : <button>Add to cart</button>}
    </li>
  );
}

function App() {
  // ── Guard clauses first (Pattern 1) ───────────────────────
  if (isLoading) return <p>Loading products…</p>;
  if (error)     return <p style={{ color: "red" }}>Error: {error}</p>;

  // ── Derived values: compute above the return ──────────────
  const inStock = products.filter((p) => p.stock > 0);   // Pattern 9
  const total = products.length;

  return (
    <div style={{ fontFamily: "sans-serif", padding: "20px", maxWidth: "460px" }}>
      {/* ?? gives a fallback only for null/undefined (Pattern 4) */}
      <h1>Hello, {user.nickname ?? user.name}</h1>

      <p>
        {/* > 0 keeps a stray 0 off the screen — the classic trap */}
        {inStock.length > 0
          ? `${inStock.length} of ${total} products available`
          : "Everything is sold out right now"}
      </p>

      {/* Empty-list guard, one level down (Pattern 10) */}
      {products.length === 0 ? (
        <p>No products in the catalogue.</p>
      ) : (
        <ul style={{ listStyle: "none", padding: 0 }}>
          {products.map((product) => (
            <ProductCard
              key={product.id}              // stable id, never the index
              product={product}
              showAdminInfo={user.isAdmin}
            />
          ))}
        </ul>
      )}

      {user.isAdmin && (
        <footer style={{ marginTop: "16px", fontSize: "12px", color: "#888" }}>
          Admin view · {total} total products
        </footer>
      )}
    </div>
  );
}

export default App;
```

Things to try: set `isLoading` to `true`, set `error` to a string, set `user.isAdmin` to `false`, and empty the `products` array. Each change shows a different branch — and you never wrote a single "hide this element" instruction.

---

## 6. How it works behind the scenes

### Conditional rendering is just a different element tree

React does not "hide" or "show" anything. On each render you hand React a description, and React compares it to the previous description.

```
  RENDER 1                          RENDER 2
  isLoggedIn = false                isLoggedIn = true

  { type: "div", children:          { type: "div", children:
      { type: LoginForm } }             { type: Dashboard } }
              │                                 │
              └────────────┬────────────────────┘
                           ▼
          React compares the two trees:
          type LoginForm  ≠  type Dashboard
                           ▼
          Different type → destroy LoginForm's DOM entirely,
          build Dashboard's DOM from scratch
```

The rule React follows is simple: **same `type` in the same position → keep the DOM node and just update it. Different `type` → throw it away and build new.**

This has a practical consequence worth knowing early. Because the DOM node is thrown away, everything attached to it — typed input text, scroll position, and later, component state — is lost. Swapping between two different components resets them.

### Why `{false}` renders nothing

Nothing clever happens here. React's rendering code literally checks the value's type:

```
  value is a string or number  →  create a text node
  value is an array            →  render each item
  value is a React element     →  render that component
  value is null / undefined /
        true / false           →  do nothing, skip it
  value is a plain object      →  throw "Objects are not valid as a React child"
```

That is the whole mechanism. `{isOpen && <Modal />}` evaluates to `false` and hits the "skip it" line.

### What `key` actually does

This is the part worth understanding properly, because it explains a whole family of confusing bugs.

When React re-renders a list, it must answer one question for each old element: *"is this the same item as before, or a different one?"*

**Without keys**, React matches by **position**:

```
  BEFORE            AFTER (removed "Mango" from the middle)
  ────────────      ─────────────────────────────────────
  pos 0: Apple      pos 0: Apple      → same position, keep, update text: "Apple"  ✓
  pos 1: Mango      pos 1: Banana     → same position, keep, update text: "Banana"
  pos 2: Banana     (gone)            → delete the third node

  React thinks: "one item was deleted from the END and item 1's text changed."
  Reality:      "the MIDDLE item was deleted."
```

For plain text that happens to produce the right screen. But the DOM node that used to be Mango's is now reused for Banana — carrying over its typed input values, its scroll position, its animation state, and later its component state.

**With keys**, React matches by **identity**:

```
  BEFORE                     AFTER
  ───────────────────        ───────────────────
  key="a": Apple             key="a": Apple    → same key → reuse this exact node ✓
  key="m": Mango             key="b": Banana   → key "m" is gone → DELETE that node
  key="b": Banana                              → key "b" existed → MOVE its node up ✓

  React thinks: "Mango was removed." Correct.
```

The classic bug this causes with `key={index}`:

```jsx
// A list of inputs, keyed by index
{items.map((item, i) => (
  <li key={i}>
    <input defaultValue={item.name} />
  </li>
))}
```

Type something into the second input, then delete the **first** item. Every item shifts up one position, but the keys `0, 1, 2` stay attached to the same positions. React sees "the keys are unchanged, just update the data" and leaves the DOM nodes exactly where they are — so your typed text is now sitting next to the wrong item.

```
  index keys:  the key describes WHERE, not WHAT
  id keys:     the key describes WHAT, regardless of where
```

> 💡 Rule of thumb: if the list can be reordered, filtered, sorted, or have items removed from anywhere but the end, you need real ids. If the list is static and display-only, `key={index}` is harmless.

> ⚠️ Never use `key={Math.random()}`. Every render produces new keys, so React thinks every single item is brand new. It destroys and recreates the entire list on every render — the exact opposite of what keys are for.

### The `key` is not a prop

```jsx
function Item({ key, name }) {   // ❌ key is always undefined here
  console.log(key);
}

<Item key={id} id={id} name={n} />  // ✅ pass it twice if you need the value
```

React consumes `key` for its own bookkeeping and strips it before your component sees it. Same for `ref`.

**In simple words:** React compares descriptions; `key` is how it knows which item is which.

---

## 7. Comparison table

### Choosing a conditional pattern

| Pattern | Syntax | Best for | Watch out for |
|---------|--------|----------|---------------|
| `if / else` | above the `return` | Two very different whole outputs; guard clauses | Cannot go inside JSX |
| Ternary | `{c ? <A/> : <B/>}` | Either/or, inside JSX or an attribute | Never nest more than one level |
| `&&` | `{c && <A/>}` | Show something, or nothing | `0` renders as `0` — use `> 0` |
| `??` | `{v ?? "fallback"}` | Missing values where `0`/`""` are valid | Different from `||` |
| `?.` | `{a?.b?.c}` | Data that may be half-loaded | Returns `undefined`, not an error |
| Variable | `let el = ...` | Long, multi-branch logic | Slightly more code |
| Object lookup | `MAP[status]` | One of many fixed cases | Add `??` for unknown keys |

### `&&` vs ternary vs `if` — same output, three shapes

```jsx
// Show a badge only when there are unread messages.

{unread > 0 && <Badge count={unread} />}                 // ✅ shortest, no else

{unread > 0 ? <Badge count={unread} /> : null}           // same thing, more typing

{(() => {                                                 // ❌ never do this
  if (unread > 0) return <Badge count={unread} />;
  return null;
})()}
```

### `||` vs `??` — the difference that bites

| Expression | `value = 0` | `value = ""` | `value = null` |
|------------|-------------|--------------|----------------|
| `value \|\| "none"` | `"none"` ❌ | `"none"` ❌ | `"none"` ✅ |
| `value ?? "none"` | `0` ✅ | `""` ✅ | `"none"` ✅ |

### List keys at a glance

| Key source | Safe? | When |
|------------|-------|------|
| `item.id` from a database | ✅ Always best | The normal case |
| A unique string like an email or slug | ✅ Good | When there is no numeric id |
| `crypto.randomUUID()` assigned **once** when the item is created | ✅ Good | Client-side items with no server id |
| `index` | ⚠️ Only sometimes | Static, never-reordered, display-only lists |
| `Math.random()` | ❌ Never | Rebuilds the whole list every render |
| Nothing at all | ❌ Never | React warns and matches by position |

**In simple words:** pick the shortest pattern that stays readable, and always key by identity.

---

## 8. Common mistakes beginners make

**1. The `0` leak with `&&`**

```jsx
{items.length && <List />}       {/* ❌ prints a bare 0 when empty */}
{items.length > 0 && <List />}   {/* ✅ */}
```

**2. Writing `if` inside braces**

```jsx
{ if (ok) <A /> }                {/* ❌ if is a statement */}
{ ok ? <A /> : null }            {/* ✅ */}
```

**3. Curly braces in the arrow function without `return`**

```jsx
{items.map(i => { <li>{i}</li> })}          {/* ❌ returns undefined → blank */}
{items.map(i => ( <li>{i}</li> ))}          {/* ✅ */}
{items.map(i => { return <li>{i}</li>; })}  {/* ✅ */}
```

**4. Missing `key`, or putting it on the wrong element**

```jsx
{users.map(u => <div><Card key={u.id} /></div>)}   {/* ❌ key is too deep */}
{users.map(u => <div key={u.id}><Card /></div>)}   {/* ✅ outermost element */}
```

**5. `key={Math.random()}`**

```jsx
<li key={Math.random()}>     {/* ❌ new key every render, list rebuilt every time */}
<li key={item.id}>           {/* ✅ */}
```

**6. Trying to read `props.key`**

```jsx
function Row({ key }) { ... }        {/* ❌ always undefined */}
<Row key={id} id={id} />             {/* ✅ pass it under another name too */}
```

**7. Using `forEach` instead of `map`**

```jsx
{items.forEach(i => <li>{i}</li>)}   {/* ❌ forEach returns undefined */}
{items.map(i => <li>{i}</li>)}       {/* ✅ map returns an array */}
```

**8. Forgetting the empty state**

```jsx
{tasks.map(t => <Task key={t.id} {...t} />)}   {/* ❌ blank page when empty */}

{tasks.length === 0
  ? <p>Nothing here yet</p>
  : tasks.map(t => <Task key={t.id} {...t} />)}  {/* ✅ */}
```

**9. Deeply nested ternaries**

```jsx
{a ? <X/> : b ? <Y/> : c ? <Z/> : <W/>}   {/* ❌ unreadable in a week */}
// ✅ move it above the return, or use an object lookup
```

**10. Calling `.map` on something that might not be an array**

```jsx
{data.map(...)}                 {/* ❌ crashes while data is still undefined */}
{data?.map(...)}                {/* ✅ optional chaining */}
{(data ?? []).map(...)}         {/* ✅ default to an empty array */}
```

**11. Mutating the array while rendering**

```jsx
{items.sort().map(...)}         {/* ❌ sort() mutates the original array */}
{[...items].sort().map(...)}    {/* ✅ copy first, then sort */}
```

**12. Duplicate keys**

```jsx
{users.map(u => <Row key={u.name} />)}   {/* ❌ two people named "Amit" → warning */}
{users.map(u => <Row key={u.id} />)}     {/* ✅ ids are unique */}
```

---

## 9. Cheat sheet

```jsx
// ── CONDITIONS ──────────────────────────────────────────────
if (loading) return <Spinner />;         // guard clause, above the return
return null;                             // render nothing

{cond ? <A /> : <B />}                   // either / or
{cond && <A />}                          // show, or nothing
{cond > 0 && <A />}                      // ✅ guard numbers with a comparison
{value ?? "fallback"}                    // only null/undefined fall through
{value || "fallback"}                    // 0 and "" also fall through
{obj?.deep?.value}                       // safe access, no crash

{STATUS[key] ?? "Unknown"}               // object lookup instead of if-chains

className={active ? "on" : "off"}        // conditional attribute
style={{ color: ok ? "green" : "red" }}  // conditional style
disabled={items.length === 0}            // conditional boolean attribute

// ── LISTS ───────────────────────────────────────────────────
{items.map(i => <li key={i.id}>{i.text}</li>)}          // basic
{items.filter(i => i.done).map(i => <li key={i.id}/>)}  // filter then map
{[...items].sort((a,b) => a.n - b.n).map(...)}          // copy before sorting
{items?.map(...)}                                        // safe when maybe undefined

{items.length === 0                                      // empty state
  ? <p>Nothing yet</p>
  : items.map(i => <Row key={i.id} {...i} />)}

<React.Fragment key={id}>...</React.Fragment>            // keyed fragment

// ── KEY RULES ───────────────────────────────────────────────
key={item.id}          // ✅ stable identity — the default choice
key={index}            // ⚠️ only for static, never-reordered lists
key={Math.random()}    // ❌ never
// unique among siblings · stable across renders · on the outermost element
// not readable as props.key
```

| Question | Answer |
|----------|--------|
| `&&` or ternary? | `&&` when there is no else branch |
| Why did a `0` appear on my page? | `{count && ...}` — use `{count > 0 && ...}` |
| Can I use `if` inside JSX? | No — ternary, or move it above the `return` |
| Why does my `map` render nothing? | `=> { }` without `return` |
| What does `key` do? | Tells React which item is which between renders |
| Is `key={index}` allowed? | Only if the list never reorders, filters, or deletes |
| Can I read `key` inside the component? | No — pass the value again under another prop name |

---

## 10. Revision questions

**Q1. Why can't you write `if` inside JSX braces?**
Braces accept an expression — something that produces a value. `if` is a statement; it performs an action and produces nothing. Use a ternary, or put the `if` above the `return`.

**Q2. What does React render for `null`, `undefined`, `true`, and `false`?**
Nothing. React skips all four silently. This is exactly what makes `{cond && <X />}` work.

**Q3. Why does `{cart.length && <Cart />}` show a `0` on an empty cart?**
`cart.length` is `0`, which is falsy, so `&&` returns `0` — and React renders numbers. Fix it with `cart.length > 0 && <Cart />`.

**Q4. What is the difference between `||` and `??`?**
`||` falls back for any falsy value, including `0` and `""`. `??` falls back only for `null` and `undefined`. Use `??` when `0` or an empty string are legitimate values.

**Q5. What is a guard clause?**
An early `return` at the top of a component that handles a special case — loading, error, missing data — so the main code path stays simple and unindented.

**Q6. Why `.map()` and not `.forEach()`?**
`map` returns a new array of elements, which React can render. `forEach` returns `undefined`, so React gets nothing.

**Q7. Why does `{items.map(i => { <li>{i}</li> })}` render a blank list?**
The `{` after `=>` opens a function body, not an object. Without an explicit `return`, the function returns `undefined`. Use `=> (` or add `return`.

**Q8. What is a `key` for?**
It tells React which element corresponds to which data item between renders, so React can reuse, move, or delete the correct DOM node instead of guessing by position.

**Q9. When is `key={index}` acceptable, and when does it break?**
Acceptable for a static, display-only list that never reorders, sorts, filters, or deletes from the middle. It breaks when items move, because the key describes a position rather than an item — so React keeps the wrong DOM node (and its typed text or state) attached to the wrong data.

**Q10. Why is `key={Math.random()}` always wrong?**
Every render generates new keys, so React believes every item is brand new. It destroys and rebuilds the entire list on each render, losing all DOM state and wasting work.

**Q11. Can you read `key` as a prop inside the component?**
No. React strips `key` for its own bookkeeping. If you need the value, pass it again under a different name: `<Row key={id} id={id} />`.

**Q12. Does a `key` have to be unique across the whole app?**
No — only among its immediate siblings. Two separate lists can both use `key={1}`.

**Q13. Why should you copy an array before sorting it in JSX?**
`.sort()` mutates the original array in place. If that array came in as a prop, you would be modifying the parent's data — which breaks the read-only rule for props.

**Q14. How do you put a `key` on a Fragment?**
Use the long form `<React.Fragment key={id}>`. The shorthand `<>...</>` cannot accept any props, including `key`.

---

## 11. What to learn next

You can now build a UI that describes itself from data, and shows different things for different conditions. But every note so far has one hard limit: **the data never changes.** Every example uses constants defined at the top of the file. Clicking a button logs to the console and nothing on screen moves.

The missing piece is memory — a way for a component to hold a value that can change and trigger a re-render.

- **02_Hooks / 01 — useState** — give a component its own memory, so clicks and typing actually update the screen
- **02_Hooks / 02 — useEffect** — run code after render, for things like fetching data
- **07_Advanced — Reconciliation and keys** — the full story of how React diffs trees, once you have more React under your belt

⬅ [Back to chapter index](README.md) · [Master index](../README.md)
