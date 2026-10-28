# Automa — The Complete Study Guide

A deep, code-level study guide for [Automa](https://github.com/AutomaApp/automa) — the browser
automation extension where you build workflows by connecting blocks. Everything here was derived by
reading the actual source (v1.30.00, `main`), with file paths throughout so you can follow along in
the code.

**Goal**: take you from "what is a block" to "I can add blocks, triggers, and templating functions
and debug the engine in an afternoon."

---

## 📚 The guide

| # | Chapter | What you'll learn |
|---|---|---|
| 1 | [What Automa Is & How to Read the Codebase](docs/01-what-is-automa.md) | mental model, reading order, fork-specific notes |
| 2 | [Architecture Overview](docs/02-architecture-overview.md) | every context & bundle, manifests, build system, **the full message protocol**, all 12 trigger entry points |
| 3 | [The Workflow Execution Engine](docs/03-workflow-engine.md) | `WorkflowManager/Engine/Worker/State`, life of a run, branching, loops, onError, persistence |
| 4 | [The Block System](docs/04-block-system.md) | the `tasks` registry, handler contract, background vs in-page blocks, all special blocks |
| 5 | **Recipe: Adding a New Block** | the exact end-to-end file checklist (4 files + locales) |
| 6 | [Dashboard & Workflow Editor](docs/06-dashboard-and-editor.md) | vue-flow editor, drag&drop logic, workflow JSON format, recorder, element selector, packages, teams |
| 7 | [Templating & Expressions](docs/07-templating-and-data.md) | the complete `{{ }}` reference: keywords, paths, functions, `!!` JS mode, autocomplete |
| 8 | [Storage, State & Security](docs/08-storage-and-state.md) | chrome.storage vs Dexie, workflow document shape, credentials/crypto, backups |
| 9 | [Triggers & External Control](docs/09-triggers-and-external-control.md) | every trigger type, **all wire formats for controlling Automa from outside** |
| 10 | [Tips, Tricks & Power-User Patterns](docs/10-tips-and-tricks.md) | 37 field-tested tricks and gotchas |
| 11 | [Development Guide](docs/11-development-guide.md) | build & run, stack inventory, conventions, debugging cheat sheet |
| 12 | [File Map & Glossary](docs/12-file-map.md) | annotated directory tree, load-bearing files, glossary |

---

## ⚡ The 10 facts that matter most

1. **On Chrome, the engine runs in an offscreen document** — the service worker is just a router.
   The first thing the SW does is send a fake `'halo'` message to force the offscreen doc to exist.
   Inspect it via `chrome://extensions` → Automa → *Inspect views: offscreen.html*.
2. **A block is three files**: a `tasks` registry entry (`src/utils/shared.js`), a handler
   (`handler*.js`, auto-registered by filename), and an edit panel (`Edit*.vue`, auto-registered).
   Plus i18n keys. That's the whole recipe.
3. **Handlers are plain functions bound to a `WorkflowWorker`**: `handler(block, ctx)` returning
   `{data, nextBlockId}` — throwing triggers the per-block onError pipeline (retry/fallback/…).
4. **There is no scheduler**: each handler picks its own outgoing edge by returning
   `nextBlockId = this.getBlockConnections(id, outputHandle)`. Parallel branches = extra workers
   spawned with cloned state.
5. **Templating only touches fields listed in the block's `refDataKeys`**; unknown paths silently
   resolve to the literal `{{...}}` text; `!!`-prefixed strings evaluate as real JS (Chrome).
6. **The handler decides which context runs the block**: `category: 'interaction'` blocks route to
   the page's content script automatically; everything else runs in the engine context.
7. **Workflows live in `chrome.storage.local`** (key `workflows`), logs in Dexie IndexedDB (4
   tables), running states mirrored to `workflowStates` (1s debounce; toolbar badge = running count).
8. **Browser restart loses interrupted runs** (`onStartup` wipes `workflowStates`) — design
   long jobs to be resumable via `$$`-prefixed persistent variables.
9. **No `externally_connectable`**: external control is via in-page CustomEvents
   (`__automaExecuteWorkflow`), `execute.html#/<id>?var=x` deep links, `.automa.json` install URLs,
   or the automa.site bridge content script.
10. **Only UI pages hot-reload during dev** — background/contentScript/elementSelector/recorder
    bundles need a manual extension reload.

---

## 🗺️ Quick navigation by goal

- *"I want to add my own block"* → [Chapter 5](docs/05-add-a-new-block.md)
- *"I want to scrape a site and export CSV"* → [Chapter 3.10](docs/03-workflow-engine.md) + [Chapter 10.1](docs/10-tips-and-tricks.md)
- *"I want to trigger a workflow from my app/website"* → [Chapter 9.3](docs/09-triggers-and-external-control.md)
- *"Where does X live?"* → [Chapter 12 file map](docs/12-file-map.md)
- *"My workflow doesn't work"* → [Chapter 10.2 debugging](docs/10-tips-and-tricks.md) + [Chapter 11.5](docs/11-development-guide.md)
- *"How do I run this repo?"* → [Chapter 11](docs/11-development-guide.md)

---

## Scope & disclaimer

This guide documents the code as of **October 2026** (`main`, v1.30.00), including
fork-specific modifications noted where they differ from upstream Automa. Automa is actively
developed — file paths and behaviors may drift; the concepts (registry → handler → panel recipe,
engine model, templating) are stable.
