# Chapter 10 — Tips, Tricks & Power-User Patterns

> A field guide to the non-obvious. Each item cites the code that implements it.

---

## 10.1 Workflow authoring tricks

1. **`{{table@Column}}` reads row 0; `{{table@$last.Column}}` reads the last row.** With
   `loop-data` set to loop through the table, each iteration writes/reads row cursors per column —
   this is how batch scraping → export works.
2. **`$assignColumns` trick**: blocks that save data (Get text, HTTP Request) accept the special
   column name `$assignColumns` with an array of column names to bulk-map response fields → columns
   (`WorkflowWorker.addDataToColumn`).
3. **Dynamic paths**: `variables@items[{{loopData@l.$index}}]` — bracket segments are templated
   first, giving you index-into-array loops without JS.
4. **`!` stringify prefix**: `{{!variables@obj}}` forces JSON — essential when passing objects into
   JS blocks or the Google Sheets custom data.
5. **Loop escape hatch**: `loopData@<id>.$index` is the only `loopData` path that skips the
   auto-inserted `data` segment.
6. **Variables that persist across runs**: prefix with `$$` (Dexie-backed global variables).
   `$push:name` appends to an array variable.
7. **Reuse last state**: workflow setting `reuseLastState` carries the table + variables into the
   next run — great for incremental scraping with interval triggers.
8. **Multiple triggers**: one trigger block holds `data.triggers[]` — combine cron + context-menu +
   keyboard shortcut on a single workflow.
9. **Error routing**: per-block onError `toDo: 'fallback'` adds a red `output-fallback` handle on
   the canvas node (`BlockBasic.vue` renders it when `data.onError.toDo === 'fallback'`). Webhook
   errors route there automatically.
10. **wait-connections** is the join node for parallel branches — branch workers literally poll it
    every second until all arrive (or timeout).
11. **The JS block's `everyNewTab`** registers your script as a preload for every subsequent tab —
    the clean way to run JS early on every page a workflow visits.
12. **Custom user-agent per tab**: the New-tab block's `customUserAgent` applies it via the Chrome
    debugger; the Proxy block sets `chrome.proxy.settings` (cleared automatically on engine destroy).
13. **CORS-free fetch**: inside JS blocks use `automaFetch('json'|'text'|'base64', resource)` — it's
    proxied through the background's host-permission fetch.

## 10.2 Debugging tricks

14. **Testing mode is the debugger**: enable it, set per-node breakpoints (circle icon on the node),
    and `EditorDebugging.vue` gives you step-next-block, pause/resume/stop plus a live
    variable/table inspector (`ctxData` + `dataSnapshot`).
15. **`replacedValue` in logs** shows exactly how each templated field was rendered per block — the
    fastest way to debug `{{ }}` mistakes.
16. **`debugMode` workflow setting** attaches the Chrome DevTools Protocol debugger and pauses on
    breakpoints (traffic-light flow view in the editor). Chrome only — force-disabled on Firefox
    (no `debugger` permission in the Firefox manifest).
17. **Log limits**: `logsLimit` (default 1001 history entries, errors always kept) and
    `deleteLogAfter` days; both in dashboard settings.
18. **On-page execution overlay**: `showExecutedBlock.js` renders a spinner + block name on the page
    while a block runs — disable noise via `executedBlockOnWeb` setting.

## 10.3 Sharing & packaging tricks

19. **Export embeds dependencies**: `execute-workflow` targets are bundled up to depth 3 as
    `includedWorkflows` (`src/utils/workflowData.js`).
20. **Packages as reusable IO**: ctx-menu "Set as block input/output" marks package inputs/outputs;
    `BlockPackage.vue` renders dynamic handles; updates re-diff handles and remove orphaned edges.
21. **Hosted workflows** are read-only sync targets (`workflowHosts` keyed by hostId; the Host page
    polls for updates and re-registers triggers).
22. **Shareable execution**: `settings.publicId` is what the in-page CustomEvent API uses — give a
    page a stable handle to your workflow without exposing its id.

## 10.4 Development tricks (for working on Automa itself)

23. **The engine runs in an offscreen document on Chrome.** Debug it by inspecting
    `chrome://extensions` → Automa → "Inspect views: offscreen.html". The service worker is only a
    router.
24. **Only UI pages hot-reload** (`notHotReload = [background, webService, contentScript,
    recordWorkflow, elementSelector]`) — after editing engine/content code, reload the extension
    manually.
25. **`'halo'` is intentional** — the fake message that spins up the offscreen doc at SW start.
26. **`secrets`/`getPassKey.js` must exist** for dev: create `src/utils/getPassKey.js` exporting any
    string (README documents this; the repo ships `secrets.blank.js`).
27. **Firefox build**: `pnpm dev:firefox` — MV2 persistent background, engine runs in the background
    page, no offscreen, `!!` JS expressions disabled, `debugMode` disabled.
28. **Workflows are just JSON**: export a workflow (`<name>.automa.json`), hand-edit nodes/edges, and
    re-import — the format is documented in chapter 6.3. Legacy DrawFlow JSON auto-converts on open.
29. **The `business/` seam** (`@business` alias) is the no-op injection point for custom
    blocks/handlers/params without touching core files.
30. **storage.local `workflows` may be an array OR an object map** — handle both shapes when reading
    it (legacy installs).

## 10.5 Gotchas

31. **Only `refDataKeys` fields are templated.** If your `{{ }}` in a field isn't rendering, check
    the block's registry entry.
32. **Invalid `{{ }}` silently stays as literal text** — no error is thrown for unknown paths.
33. **Keyboard-shortcut triggers only fire where the content script runs** (they're Mousetrap
    bindings in the page, not `chrome.commands`).
34. **Cron alarms are one-shot** and re-registered after each fire — deleting a workflow without
    `cleanWorkflowTriggers` leaks alarms.
35. **Interrupted runs are lost on browser restart** (`onStartup` wipes `workflowStates`); the Dexie
    log is the only durable record. Design long-running jobs to be resumable (e.g. `$$` variables).
36. **Same-origin iframes**: `switch-to` uses `contentDocument`; cross-origin frames go through
    postMessage with one-time storage tokens. Selectors chain with `frameSelector|>innerSelector`.
37. **The minimap/palette/validation are generic** — a new block missing from the palette means your
    registry entry has a syntax problem, not that you need to register it somewhere.
