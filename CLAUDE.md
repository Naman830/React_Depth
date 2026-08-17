# CLAUDE.md — React_Depth

This repo is a **personal learning notebook for React.js and its ecosystem**, written in Markdown only.
No app code, no `node_modules`, no build step. Just notes that a complete beginner can read top-to-bottom and understand.

---

## DEFAULT COMMAND (run this whenever I name a topic)

> When I say a topic name — e.g. *"useState"*, *"JSX"*, *"React Router"*, *"Redux Toolkit"*, *"props"* —
> **create the beginner-friendly note for that topic in this repo, following every rule below, and update the indexes.**
> I do not need to repeat these instructions. Just do it.

Steps to perform, every single time:

1. **Decide where it goes.** Find the right numbered chapter folder (`01_Basic/`, `02_...`).
   If no chapter fits, create the next numbered chapter folder.
2. **Create the note file** at `NN_Chapter/NN_topic_name.md` using the note template below.
3. **Update the chapter index** — `NN_Chapter/README.md` — add the new file as a linked row.
4. **Update the master index** — root `README.md` — add/refresh the topic under its chapter.
5. **Report back** in 3 lines: file created, what it covers, suggested next topic.

Do **not** ask for permission to write these files. Do **not** create code projects unless I explicitly ask.

---

## Hard rules

### Language
- **Simple English only.** No Hindi, no slang.
- Short sentences. One idea per sentence.
- Explain a term the first time it appears — never assume prior knowledge.
- Prefer "we", "you" over passive voice.

### Structure on disk
```
React_Depth/
├── README.md                     <- master index of ALL topics
├── CLAUDE.md                     <- this file
├── 01_Basic/
│   ├── README.md                 <- chapter index
│   ├── 01_setUp_react_env.md
│   └── 02_jsx.md
└── 02_Hooks/
    ├── README.md
    └── 01_useState.md
```
- Folder names: `NN_TitleCase` (e.g. `03_Routing`).
- File names: `NN_snake_case.md` (e.g. `02_use_effect.md`).
- Numbers are always **two digits**, in learning order.
- Every chapter folder MUST contain a `README.md` index.

### Note depth
Every note is **deep but beginner friendly** — roughly 200–400 lines.
Never write a thin summary. Always include the *why*, not only the *how*.

### Code style in examples
- **Function components + hooks only.** No class components.
- Plain JavaScript (`.jsx`), not TypeScript.
- Arrow or `function` components, `export default` at the end.
- Every code block gets a language tag (```jsx, ```bash, ```json).
- Add short `//` comments inside code explaining the tricky lines.
- Code must be complete enough to copy-paste and run.

### Formatting
- Use `##` for main sections, `###` for sub-sections.
- Use tables for comparisons.
- Use ASCII box diagrams for flows (no images, no external links to pictures).
- Use `> 💡` callouts for tips and `> ⚠️` for common mistakes.
- End every major section with a one-line **"In simple words:"** summary.

---

## Note template (follow this order)

```markdown
# NN. <Topic Name>

> One-line plain-English answer to "what is this?"

## 1. Real-life analogy
## 2. The problem — why does this exist?
## 3. What it actually is
## 4. Syntax / setup, step by step
## 5. Full working example (with comments)
## 6. How it works behind the scenes
## 7. Comparison with alternatives (table)  <- only if alternatives exist
## 8. Common mistakes beginners make
## 9. Cheat sheet
## 10. Revision questions (with answers)
## 11. What to learn next
```

---

## Master index format (root README.md)

A table per chapter:

```markdown
### 01_Basic
| # | Topic | What you learn |
|---|-------|----------------|
| 01 | [Set up React environment](01_Basic/01_setUp_react_env.md) | Node, npm, Vite, folder structure |
```

Keep it sorted by number. Never delete an existing row — only add or edit.

---

## Planned learning path (extend as I ask for topics)

1. **01_Basic** — setup, JSX, components, props, events, rendering
2. **02_Hooks** — useState, useEffect, useRef, useContext, useReducer, useMemo, useCallback, custom hooks
3. **03_Routing** — React Router (routes, links, params, nested, protected)
4. **04_State_Management** — Context API, Redux Toolkit, Zustand
5. **05_Data_Fetching** — fetch, axios, loading/error states, TanStack Query
6. **06_Forms** — controlled inputs, validation, React Hook Form
7. **07_Advanced** — reconciliation, virtual DOM, keys, performance, lazy loading, error boundaries, portals
8. **08_Ecosystem** — Tailwind, Next.js, testing, deployment

---

## Things NOT to do

- ❌ Do not create `package.json`, `node_modules`, or any runnable app.
- ❌ Do not use class components in examples.
- ❌ Do not write short/shallow notes.
- ❌ Do not use jargon without explaining it in the same paragraph.
- ❌ Do not forget to update both `README.md` files.
