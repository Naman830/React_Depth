# 01. Setting Up the React Environment

> A "React environment" is just a folder on your computer with the right tools installed, so that the code **you** write (modern, fancy React code) gets converted into code the **browser** can actually understand.

---

## 1. Real-life analogy

Imagine you want to cook a meal.

- **React** is the *recipe*.
- **Your browser** is the *guest* who eats the food.
- But there is a problem: the guest can only eat cooked food. Your recipe has raw ingredients.
- So you need a **kitchen** — gas, pans, knives, a fridge.

That kitchen is the **development environment**. Tools like **Vite** and **Parcel** are the kitchen.
They take your raw React code and cook it into plain HTML, CSS, and JavaScript that any browser can eat.

**In simple words:** you cannot just write React code and open it in Chrome. Something must translate it first.

---

## 2. The problem — why do we even need a setup?

Let's understand the real problem. A browser only understands three things:

1. HTML
2. CSS
3. Plain JavaScript

Now look at normal React code:

```jsx
import React from "react";
import "./App.css";

function App() {
  return <h1>Hello Naman</h1>;   // <-- HTML inside JavaScript?!
}

export default App;
```

There are **three** things here that a browser cannot handle on its own:

| Line | Problem |
|------|---------|
| `<h1>Hello Naman</h1>` inside JS | This is **JSX**. It is not valid JavaScript. The browser throws a syntax error. |
| `import "./App.css"` | JavaScript cannot import a CSS file. That is not a thing in the language. |
| `import React from "react"` | Where is `"react"`? It is not a URL or a file path. The browser has no idea. |

So we need a program that:

- **Transpiles** JSX → plain JavaScript function calls
- **Bundles** hundreds of small files into a few optimized files
- **Resolves** package names like `"react"` into real file paths
- **Serves** everything on a local address like `http://localhost:5173`
- **Reloads** the page instantly when you save a file

That program is called a **build tool** (or bundler). Vite, Parcel, Webpack, and Parcel are all build tools.

**In simple words:** the browser is picky. Build tools are the translators.

---

## 3. What JSX actually becomes

This is the single most useful thing to understand early.

You write this:

```jsx
const heading = <h1 className="title">Hello</h1>;
```

The build tool converts it into something like this:

```js
const heading = React.createElement("h1", { className: "title" }, "Hello");
```

And `React.createElement` just returns a **plain JavaScript object** that describes the UI:

```js
{
  type: "h1",
  props: { className: "title", children: "Hello" }
}
```

React later reads that object and creates the real DOM element on the page.

> 💡 JSX is **not** HTML. It is sugar (a shortcut) for a JavaScript function call.

**In simple words:** JSX is a shortcut. The build tool expands the shortcut before the browser sees it.

---

## 4. Prerequisites — install these first

### 4.1 Node.js

**Node.js** is JavaScript that runs *outside* the browser, on your computer.
Build tools are themselves written in JavaScript, so they need Node to run.

Download the **LTS** version from <https://nodejs.org>.
(LTS = Long Term Support = the stable, boring, safe version. Always pick this.)

Check it worked:

```bash
node -v
# v22.11.0   <- any v18+ is fine, v20+ recommended
```

### 4.2 npm

**npm** = Node Package Manager. It downloads libraries (like React) from the internet into your project.
It is installed **automatically with Node**. You do not install it separately.

```bash
npm -v
# 10.9.0
```

> 💡 Alternatives to npm: **yarn**, **pnpm**, **bun**. They all do the same job, just faster or with different disk usage. Beginners should stick with npm.

### 4.3 A code editor

**VS Code** (<https://code.visualstudio.com>) is the standard choice.

Useful extensions:

| Extension | Why |
|-----------|-----|
| ES7+ React/Redux snippets | Type `rafce` + Enter → full component boilerplate |
| Prettier | Auto-formats your code on save |
| ESLint | Warns about mistakes while you type |
| Auto Rename Tag | Renaming `<div>` renames `</div>` too |

### 4.4 A browser

Chrome or Edge, plus the **React Developer Tools** extension. It adds a "Components" tab to DevTools so you can inspect your React tree and state.

---

## 5. The different ways to create a React app

There are five common ways. Let's go from worst to best.

### Way 1 — CDN `<script>` tags (learning only)

```html
<!DOCTYPE html>
<html>
  <body>
    <div id="root"></div>

    <script src="https://unpkg.com/react@19/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@19/umd/react-dom.development.js"></script>

    <script>
      // No JSX allowed here! We must write createElement by hand.
      const root = ReactDOM.createRoot(document.getElementById("root"));
      root.render(React.createElement("h1", null, "Hello Naman"));
    </script>
  </body>
</html>
```

- ✅ Zero setup, no Node needed
- ❌ No JSX, no components in separate files, no npm packages, terrible for real apps

**Use it for:** understanding what React *is*. Never for a real project.

### Way 2 — Create React App (CRA) — ❌ dead, do not use

For years, `npx create-react-app my-app` was the default. It is now **officially deprecated** — the React team announced in February 2025 that it is sunset, and the React docs no longer recommend it.

Why it died:

- Built on **Webpack + Babel**, which are slow. A medium app took 30–60 seconds just to start the dev server.
- Every code change took several seconds to appear in the browser.
- It was rarely updated, so its dependencies became outdated and full of security warnings.
- Configuration was hidden. To change anything you had to run `npm run eject`, which dumped ~200 lines of config on you permanently.

> ⚠️ If a tutorial tells you to run `create-react-app`, that tutorial is old. Use Vite instead.

### Way 3 — **Vite** ⭐ (recommended for learning React)

```bash
npm create vite@latest
```

Fast, modern, tiny config, and now the default recommendation in most of the React community.
Full walkthrough in section 7.

### Way 4 — **Parcel**

```bash
npm install --save-dev parcel
```

Zero configuration. Great when you want *literally* no config file. Details in section 6.

### Way 5 — A framework (Next.js, React Router / Remix)

```bash
npx create-next-app@latest
```

These are not just build tools — they are **full frameworks**. They add routing, server-side rendering, API routes, image optimization, and data fetching on the server.

The official React docs now recommend starting with a framework for **production apps**.

> 💡 But for *learning React itself*, a framework adds too much extra to learn at once. Learn React with **Vite** first, then move to Next.js. That is the path this repo follows.

---

## 6. Deep dive — why Vite? Why Parcel? Why not Webpack?

This is the question everyone asks. Let's answer it properly.

### 6.1 What every build tool must do

| Job | Meaning |
|-----|---------|
| **Transpile** | Convert JSX and new JS syntax into older, plain JS |
| **Bundle** | Merge many small files into few big files |
| **Resolve** | Turn `import "react"` into `node_modules/react/index.js` |
| **Dev server** | Host your app at `localhost:PORT` |
| **HMR** | Hot Module Replacement — update the page on save *without losing state* |
| **Optimize** | Minify, tree-shake, split code, hash filenames for caching |

### 6.2 The old way (Webpack / CRA)

```
You press "npm start"
        │
        ▼
Webpack reads EVERY file in your project
        │
        ▼
Bundles ALL of them into one giant bundle.js
        │
        ▼
Only now does the server start   ⏳ 30-60 seconds
```

The killer problem: startup time grows with the **size of your whole project**, even if you only wanted to look at one page.

### 6.3 The Vite way

Vite's trick is that **modern browsers now support ES modules natively** (`import` / `export` directly in the browser). So in development, Vite does not need to bundle at all.

```
You press "npm run dev"
        │
        ▼
Server starts IMMEDIATELY          ⚡ ~200 ms
        │
        ▼
Browser asks for /src/main.jsx
        │
        ▼
Vite transforms ONLY that one file (using esbuild, written in Go — very fast)
        │
        ▼
Browser sees "import App from './App.jsx'" and asks for that file too
        │
        ▼
Vite transforms only that file... and so on, on demand
```

Two more details that matter:

- **Dependency pre-bundling.** Libraries in `node_modules` (like React) are pre-bundled once with **esbuild**, because a library can be split into hundreds of tiny files and requesting each one over HTTP would be slow. This happens once and is cached.
- **Production build uses Rollup.** For `npm run build`, Vite *does* bundle properly with **Rollup**, because for real users on real networks, a few optimized files beat hundreds of requests.

So Vite is really **two tools in one**: an unbundled dev server, and a bundled production build.

> 💡 The name "Vite" is French for "fast" (pronounced *veet*). That is the entire pitch.

### 6.4 The Parcel way

Parcel's pitch is **zero configuration**.

- No config file needed at all. Point it at an HTML file and it figures out the rest.
- It reads your `index.html`, follows every `<script>`, `<link>`, and `<img>`, and handles them.
- Uses **SWC** (written in Rust) for JavaScript, and **Lightning CSS** for CSS — both very fast.
- Has a **filesystem cache**, so the second start is much faster than the first.
- Automatically handles images, fonts, TypeScript, Sass, etc. without you installing plugins.

Its weakness: a smaller community and fewer plugins than Vite, and the "magic" can be hard to debug when it goes wrong.

### 6.5 Side-by-side comparison

| | **Vite** | **Parcel** | **Webpack (CRA)** |
|---|---|---|---|
| Dev server start | ⚡ Instant (~0.2s) | 🙂 Fast after first run | 🐢 Slow (30–60s) |
| Update on save (HMR) | ⚡ Instant | ⚡ Fast | 🐢 Seconds |
| Config file | Tiny `vite.config.js` | None needed | Huge and complex |
| Dev engine | Native ESM + esbuild | SWC + cache | Full bundle every time |
| Prod bundler | Rollup | Its own bundler | Webpack |
| Community / plugins | 🥇 Very large | 🥉 Smaller | 🥈 Large but declining |
| Status in 2026 | Industry default | Alive, niche | CRA deprecated |
| Best for | Almost every React project | Quick demos, zero-config lovers | Legacy projects only |

### 6.6 So which one should you pick?

**Use Vite.** Reasons:

1. It is what the ecosystem has standardized on — most tutorials, plugins, and job codebases use it.
2. It is fast enough that you never think about it.
3. Its config file is 6 lines, so it is easy to learn *and* easy to customize later.
4. Next.js, Remix, and Astro all borrow ideas from it, so the concepts transfer.

Pick **Parcel** only if you specifically want zero config files.
Pick **Next.js** when you start building a real production product that needs SEO or a backend.
Pick **Webpack** only when you join a company that already uses it.

**In simple words:** Vite is fast because it does not bundle during development. Parcel is easy because it has no config. Webpack is slow because it bundles everything before it can even start.

---

## 7. Step by step — create your first React app with Vite

### Step 1 — open a terminal in your projects folder

```bash
cd ~/Projects
```

### Step 2 — run the create command

```bash
npm create vite@latest
```

`npm create` downloads and runs the project generator. `@latest` makes sure you get the newest version.

It will now ask you questions:

```
? Project name: › my-first-react-app
? Select a framework: › React
? Select a variant: › JavaScript
```

- **Project name** — becomes the folder name. Use lowercase with dashes, no spaces.
- **Framework** — choose **React** (arrow keys + Enter).
- **Variant** — choose **JavaScript** for now. (`TypeScript` adds types; `SWC` is an alternative, slightly faster compiler. Both are fine later.)

> 💡 One-line shortcut that skips all questions:
> ```bash
> npm create vite@latest my-first-react-app -- --template react
> ```

### Step 3 — go inside and install

```bash
cd my-first-react-app
npm install
```

`npm install` reads `package.json` and downloads React and Vite into a folder called `node_modules`.
This takes 10–30 seconds and needs internet. You only do it once per project.

### Step 4 — start the dev server

```bash
npm run dev
```

You will see:

```
  VITE v7.0.0  ready in 187 ms

  ➜  Local:   http://localhost:5173/
  ➜  press h + enter to show help
```

Open that URL. Your React app is running. 🎉

### Step 5 — try a live edit

Open `src/App.jsx`, change the text, hit save. The browser updates **instantly** without a full refresh. That is HMR working.

### Step 6 — stop the server

Press `Ctrl + C` in the terminal.

---

## 8. Understanding every file Vite created

```
my-first-react-app/
├── node_modules/          <- downloaded libraries (never touch, never commit)
├── public/                <- static files copied as-is
│   └── vite.svg
├── src/                   <- 👈 YOUR CODE LIVES HERE
│   ├── assets/
│   │   └── react.svg
│   ├── App.css
│   ├── App.jsx            <- your main component
│   ├── index.css          <- global styles
│   └── main.jsx           <- the entry point
├── .gitignore
├── eslint.config.js
├── index.html             <- 👈 the real HTML page
├── package.json           <- project info + dependencies + scripts
├── package-lock.json      <- exact versions lock file
├── vite.config.js         <- Vite settings
└── README.md
```

### 8.1 `index.html` — the starting point

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React</title>
  </head>
  <body>
    <div id="root"></div>                          <!-- React fills this box -->
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

Two important things:

- `<div id="root"></div>` is an **empty box**. Your whole app gets injected inside it.
- `type="module"` tells the browser this is an ES module, which is exactly what lets Vite skip bundling.

> ⚠️ In CRA, `index.html` lived inside `public/`. In Vite it lives at the **project root**. This confuses people migrating from old tutorials.

### 8.2 `src/main.jsx` — the entry point

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

Reading it line by line:

| Line | Meaning |
|------|---------|
| `createRoot(...)` | Find the empty `<div id="root">` and tell React "this is yours now" |
| `.render(<App />)` | Draw the `App` component inside it |
| `<StrictMode>` | A **development-only** helper. It intentionally runs some code twice to expose bugs. It does nothing in production. |

> ⚠️ StrictMode is why your `console.log` sometimes prints twice in development. That is not a bug in your code.

### 8.3 `src/App.jsx` — your first component

A clean version to start from:

```jsx
import { useState } from "react";
import "./App.css";

function App() {
  // useState gives you a value + a function to change it
  const [count, setCount] = useState(0);

  return (
    <div className="app">
      <h1>Hello Naman 👋</h1>
      <p>You clicked {count} times</p>

      {/* onClick takes a function, so we wrap it in an arrow function */}
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}

export default App;
```

Three beginner rules visible here:

1. A component is just a **function that returns JSX**.
2. Its name must start with a **capital letter** (`App`, not `app`) — that is how JSX knows it is a component and not an HTML tag.
3. It must return **one** parent element. Wrap siblings in a `<div>` or an empty fragment `<>...</>`.

### 8.4 `package.json` — the ID card of your project

```json
{
  "name": "my-first-react-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint ."
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "vite": "^7.0.0",
    "@vitejs/plugin-react": "^4.3.0"
  }
}
```

| Key | Meaning |
|-----|---------|
| `scripts` | Shortcuts you run with `npm run <name>` |
| `dependencies` | Packages your app needs **in the browser at runtime** (React) |
| `devDependencies` | Packages needed only **while developing** (Vite, ESLint). Not shipped to users. |
| `^19.0.0` | "19.anything" — allows minor and patch updates, but not version 20 |

**Why two `react` packages?**
`react` is the core library (components, hooks, state). `react-dom` is the part that talks to the browser DOM. They are separate because React also runs on other targets — React Native uses `react` + `react-native` instead.

### 8.5 `package-lock.json`

Records the **exact** version of every package, including packages-of-packages.
It guarantees your teammate installs exactly the same versions as you.

> ⚠️ Never edit it by hand. **Do** commit it to git.

### 8.6 `node_modules/`

Every downloaded library. It can contain 200 MB and 30,000 files.

> ⚠️ Never commit it to git. It is already in `.gitignore`. If you delete it, just run `npm install` again to rebuild it.

### 8.7 `public/` vs `src/assets/`

| | `public/` | `src/assets/` |
|---|---|---|
| How you use a file | `<img src="/logo.png" />` | `import logo from "./assets/logo.png"` |
| Processed by Vite? | ❌ Copied as-is | ✅ Optimized, gets a hashed filename |
| Best for | `robots.txt`, `favicon`, files that need a fixed URL | Images and fonts used inside components |

Rule of thumb: put things in `src/assets/` unless you specifically need a fixed public URL.

### 8.8 `vite.config.js`

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],   // teaches Vite how to handle JSX and Fast Refresh
});
```

That is the entire config. Compare that to CRA's ejected Webpack config, which is hundreds of lines.

Common things you might add later:

```js
export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,     // change the dev port
    open: true,     // auto-open the browser
  },
});
```

### 8.9 `.gitignore`

Tells git what to skip: `node_modules`, `dist`, `.env` files, editor folders. Vite generates a sensible one.

---

## 9. How the app actually boots (the full flow)

```
Browser opens http://localhost:5173
        │
        ▼
index.html loads
        │
        ▼
<script type="module" src="/src/main.jsx">
        │
        ▼
main.jsx runs
        │
        ├── imports React + ReactDOM
        ├── imports App.jsx
        ├── imports index.css
        │
        ▼
createRoot(document.getElementById("root"))
        │
        ▼
.render(<App />)
        │
        ▼
App() function runs and returns JSX
        │
        ▼
React converts JSX into a Virtual DOM object tree
        │
        ▼
React creates real DOM nodes and puts them inside <div id="root">
        │
        ▼
🎉 You see the page
```

**In simple words:** HTML → main.jsx → React takes over the `#root` div → your components render inside it.

---

## 10. The npm scripts you will use daily

| Command | What it does | When to use |
|---------|--------------|-------------|
| `npm install` | Download dependencies into `node_modules` | Once after cloning or creating a project |
| `npm run dev` | Start the dev server with HMR | Every day, while coding |
| `npm run build` | Create an optimized production build in `dist/` | Before deploying |
| `npm run preview` | Serve the `dist/` folder locally to test the real build | After `build`, before deploying |
| `npm run lint` | Check code for mistakes with ESLint | Before committing |
| `npm install axios` | Add a new library | When you need a package |
| `npm uninstall axios` | Remove a library | When you stop using it |

### What `npm run build` produces

```
dist/
├── index.html
├── assets/
│   ├── index-a1b2c3d4.js     <- all your JS, minified
│   └── index-e5f6g7h8.css    <- all your CSS, minified
└── vite.svg
```

The random letters in the filename are a **content hash**. If the file's content changes, the name changes, so browsers are forced to download the fresh version instead of using a stale cache.

You upload this `dist/` folder to Netlify, Vercel, GitHub Pages, or any static host. That is deployment.

> 💡 `npm run dev` output is **never** what users get. Only `dist/` is.

---

## 11. Environment variables (a common beginner trap)

If you have an API key or a base URL, put it in a `.env` file at the project root:

```bash
VITE_API_URL=https://api.example.com
```

Read it in your code:

```jsx
const url = import.meta.env.VITE_API_URL;
```

Two rules:

1. The name **must** start with `VITE_`. Vite hides everything else, so you cannot accidentally leak a secret.
2. Restart `npm run dev` after editing `.env`. Changes are not hot-reloaded.

> ⚠️ CRA used `REACT_APP_` and `process.env.X`. Vite uses `VITE_` and `import.meta.env.X`. This is the #1 error when copying old code.
> ⚠️ Anything in a frontend `.env` ends up in the browser bundle. It is **not** secret. Never put a real private key there.

---

## 12. Common mistakes beginners make

| Mistake | What you see | Fix |
|---------|--------------|-----|
| Running `npm run dev` in the wrong folder | `Missing script: "dev"` | `cd` into the project folder first |
| Forgot `npm install` | `Cannot find module 'react'` | Run `npm install` |
| Naming a file `App.js` but writing JSX in it | `Failed to parse source... Expected ">"` | Rename to **`App.jsx`**. Vite requires the `.jsx` extension for JSX. |
| Component name in lowercase (`function app()`) | Renders nothing, or `<app>` appears in the HTML | Capitalize it: `function App()` |
| Returning two elements side by side | `JSX expressions must have one parent element` | Wrap in `<>...</>` |
| Using `class=` instead of `className=` | Warning in console, style not applied | Use `className` — `class` is a reserved JS keyword |
| Port already in use | `Port 5173 is in use, trying another one` | Harmless — Vite picks 5174. Or set a port in `vite.config.js`. |
| Committed `node_modules` | Git is huge and slow | Add it to `.gitignore`, then `git rm -r --cached node_modules` |
| `console.log` printing twice | Confusing double logs | That is `<StrictMode>` in development. Normal. |
| Wrote `onclick` instead of `onClick` | Nothing happens on click | JSX event names are camelCase |

---

## 13. Cheat sheet

```bash
# --- one-time machine setup ---
node -v                      # check Node is installed (need v18+)
npm -v                       # check npm

# --- create a project ---
npm create vite@latest                              # interactive
npm create vite@latest my-app -- --template react   # one-liner

# --- daily ---
cd my-app
npm install                  # first time only
npm run dev                  # start coding
Ctrl + C                     # stop the server

# --- ship it ---
npm run build                # creates dist/
npm run preview              # test dist/ locally

# --- packages ---
npm install axios            # add a runtime package
npm install -D sass          # add a dev-only package
npm uninstall axios          # remove

# --- when things break ---
rm -rf node_modules package-lock.json
npm install                  # clean reinstall
```

Minimum React file you can write:

```jsx
// src/App.jsx
function App() {
  return <h1>Hello</h1>;
}
export default App;
```

---

## 14. Revision questions

<details>
<summary><b>1. Why can't a browser run React code directly?</b></summary>

Because React code contains JSX (HTML-like syntax inside JavaScript), CSS imports, and bare package imports like `import React from "react"`. None of those are valid browser JavaScript. A build tool must convert them first.
</details>

<details>
<summary><b>2. What does JSX turn into?</b></summary>

`React.createElement(type, props, children)` calls, which return plain JavaScript objects describing the UI.
</details>

<details>
<summary><b>3. Why is Vite so much faster than Create React App?</b></summary>

CRA (Webpack) bundles your entire project **before** the dev server can start, so startup time grows with project size. Vite serves files over native ES modules and transforms each file **on demand** with esbuild (written in Go), so the server starts in milliseconds regardless of project size.
</details>

<details>
<summary><b>4. Does Vite bundle at all?</b></summary>

Yes — but only for production. `npm run build` uses **Rollup** to create optimized bundles. Development is unbundled.
</details>

<details>
<summary><b>5. What is Parcel's main selling point?</b></summary>

Zero configuration. It reads your `index.html`, follows all links, and handles JS/CSS/images/TypeScript automatically without any config file or plugin installation.
</details>

<details>
<summary><b>6. Should I use create-react-app?</b></summary>

No. It was officially deprecated in February 2025. Use Vite for learning, or a framework like Next.js for production apps.
</details>

<details>
<summary><b>7. Difference between `dependencies` and `devDependencies`?</b></summary>

`dependencies` are needed by the app when it runs in a user's browser (React). `devDependencies` are only needed while building or developing (Vite, ESLint) and are not shipped.
</details>

<details>
<summary><b>8. Why are `react` and `react-dom` two separate packages?</b></summary>

`react` is the platform-independent core (components, state, hooks). `react-dom` is the renderer for web browsers. Other renderers exist — React Native uses `react-native` with the same `react` core.
</details>

<details>
<summary><b>9. What does `<StrictMode>` do?</b></summary>

It is a development-only wrapper that intentionally double-invokes certain functions to help you spot side-effect bugs early. It has zero effect in the production build.
</details>

<details>
<summary><b>10. What is inside the `dist/` folder and what do you do with it?</b></summary>

The minified, hashed, production-ready HTML/CSS/JS. You upload it to a static host (Netlify, Vercel, GitHub Pages) to deploy your site.
</details>

<details>
<summary><b>11. Why must environment variables start with `VITE_`?</b></summary>

Vite only exposes variables with that prefix to the client bundle, so you cannot accidentally leak server secrets into the browser.
</details>

<details>
<summary><b>12. What is HMR?</b></summary>

Hot Module Replacement — when you save a file, only that module is swapped in the running page. The page does not fully reload, so your component state is preserved.
</details>

---

## 15. What to learn next

1. **JSX rules in depth** — expressions, conditionals, lists, fragments
2. **Components and props** — breaking UI into reusable pieces
3. **useState** — making the UI react to changes
4. **Handling events** — clicks, forms, inputs

> Next file: `01_Basic/02_jsx.md`
