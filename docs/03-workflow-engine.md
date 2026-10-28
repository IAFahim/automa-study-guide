# Chapter 3 — The Workflow Execution Engine (`src/workflowEngine`)

> This is the heart of Automa: how a drawn graph of blocks becomes an actual execution.
> Master this chapter and everything else in the codebase falls into place.

---

## 3.1 Where the engine runs (topology)

The engine does **not** run in the background service worker on Chrome — it runs in an
**offscreen document**, created on demand:

| Context | Role |
|---|---|
| `src/background/index.js` | Service worker: browser event listeners, message router, privileged browser APIs, script injections, download listeners |
| Offscreen document (`src/offscreen/index.js` → `src/offscreen/message-listener.js`) | **Hosts `WorkflowManager` on Chrome MV3.** Created on demand (`chrome.offscreen.createDocument`, reasons BLOBS/CLIPBOARD/IFRAME_SCRIPTING) by `src/background/BackgroundOffscreen.js` |
| Background page (Firefox) | `BackgroundWorkflowUtils.#ensureWorkflowManager` dynamically imports `WorkflowManager` — the engine runs directly in the background |
| Content script (`src/content/index.js`) | Executes "interaction" blocks inside web pages |
| Sandbox iframe (`src/sandbox/index.js`) | Evaluates `!!` JS expressions, background-context JS blocks, condition code |

Because the offscreen document has **no direct `chrome.tabs` etc.**, `src/service/browser-api/BrowserAPIService.js`
transparently proxies every browser API: if the tabs API isn't present, calls are serialized via the
`'browser-api'` message to the background (`BrowserAPIService.runtimeMessageHandler` in
`src/background/index.js`); browser events come back through `BrowserAPIEventHandler`
(`src/offscreen/message-listener.js`).

> MV3 survival trick: the offscreen document is a long-lived DOM context, so the engine (loops,
> timers, state) survives service-worker termination. This is the single most important
> architectural decision in the fork/codebase.

---

## 3.2 The class hierarchy

### `WorkflowManager` — `src/workflowEngine/WorkflowManager.js` (singleton)
- Owns the shared `WorkflowState` and `WorkflowLogger`.
- `execute(workflowData, options)`: blocks duplicate runs while `testingMode`; normalizes legacy
  drawflow via `convertWorkflowData`; constructs a `WorkflowEngine` with
  `{states, logger, blocksHandler}`; `engine.init()`; subscribes to `engine.on('destroyed')` →
  desktop notification (if `settings.notification`) + fires post-run `settings.events` through
  `WorkflowEvent` (HTTP request or JS code) with `{workflow: {status, logs…}, variables, globalData}`.
- Also does a daily `/status` telemetry ping.

### `WorkflowEngine` — `src/workflowEngine/WorkflowEngine.js` (one per run)
Key fields:
```js
this.referenceData = { variables, table: [], secrets: {}, loopData: {},
  workflow: {}, googleSheets: {}, globalData: parseJSON(globalData, globalData) };
```
plus `workers: Map`, `connectionsMap`, `waitConnections`, `blocks` (nodeId→node), `history`,
`historyCtxData`, `refDataSnapshots`, `columns`/`rowData` (table schema + per-column cursors).

**`init()` sequence:**
1. Reject disabled / no-blocks / no-trigger workflows; trigger = node with `label === 'trigger'` or
   `options.blockId`.
2. **Workflow parameters**: if the trigger has `parameters`, open the `params.html` popup
   (or inject an `input-workflow-params` form into the active page when `preferParamsInTab`) and
   **return early** — the run restarts later from the params page with
   `options: {checkParams: false, data: {variables}}`.
3. Build `this.blocks` map and `connectionsMap` from edges:
   `connectionsMap[sourceHandle] = Map(target → {id, targetHandle, sourceHandle})`.
4. Load data columns — `workflow.table`/`dataColumns`, or a **connected table** from Dexie
   (`dbStorage.tablesItems/tablesData`).
5. `debugMode` → attach `BrowserAPIService.debugger.onEvent` listener.
6. `reuseLastState` → restore `columns` + `referenceData` from `storage.local['state:${workflow.id}']`.
7. Load Dexie credentials → `referenceData.secrets`; Dexie variables →
   `referenceData.variables['$$name']`.
8. Register on `states` `stop`/`resume` events, `states.add(this.id, {status:'running'})`, then
   **`addWorker({blockId: triggerBlock.id})`**.

Other methods: `addWorker` (sequential `worker-N` ids, optional state snapshot for branch workers);
`addLogHistory` (caps at `logsLimit` (default 1001), never drops `error` entries, stores
`replacedValue` + ref-data snapshot keys `##variablesN`/`##loopDataN`); `stop()`; `destroyWorker`;
`destroy(status)` (clears proxy, detaches debuggers, `states.delete`, telemetry POST, emits
`'destroyed'`, persists `reuseLastState`, merges child-workflow referenceData, writes the log via
`WorkflowLogger`, cleans up); `executeQueue` (relaunches queued interval runs); `updateState`
(per-worker `currentBlock` + `tabIds` + logs → `states.update`); `on/dispatchEvent`.

### `WorkflowWorker` — `src/workflowEngine/WorkflowWorker.js` (one per execution path)
Per-worker mutable state: `activeTab {id,url,frameId,frames,groupId}`, `windowId`, `loopList`
(loopId → `{index, data, maxLoop, type, blockId}`), `loopEls`, `repeatedTasks`, `preloadScripts`,
`frameSelector` (iframes), `breakpointState`, `childWorkflowId`, `debugAttached`.

**All block handlers are bound to the worker** (`this` = worker). Common members handlers use:
- `this.getBlockConnections(blockId, outputIndex=1)`
- `this.setVariable(name, value)` — supports `$push:name` and `$$`-prefixed Dexie-persisted variables
- `this.addDataToColumn(column, value)` — column-major write with per-column cursors
- `this.activeTab`, `this.loopList`, `this.loopEls`, `this.repeatedTasks`, `this.engine`
- `this._sendMessageToTab(payload, options)` — messages the content script with auto-injection retry

### `WorkflowState` — `src/workflowEngine/WorkflowState.js`
A `Map` of running states `{id, status, state, workflowId, parentState, teamId}`; debounced (1s)
persistence to `storage.local['workflowStates']`; badge text = running count; emits
`stop`/`resume`/`update`/`delete`. `stop()` marks `isDestroyed: true` — and `executeBlock`
re-checks `states.get(engine.id)` at the start of **every block**, so a stop (or a dead state after
service-worker loss) halts the graph at the next block boundary.

### `WorkflowLogger` & `WorkflowEvent`
`WorkflowLogger.add` writes 4 Dexie records at destroy time into the `logs` DB: `items` (metadata),
`histories` (block history), `ctxData` (template-replaced values + ref-data snapshots), `logsData`
(final table+variables). `WorkflowEvent` = post-run webhook/JS automations.

---

## 3.3 Life of a workflow run (numbered walkthrough)

1. **Trigger**: dashboard run button → `RendererWorkflowService.executeWorkflow` →
   `'workflow:execute'` message → background → `BackgroundWorkflowUtils.executeWorkflow` →
   offscreen `'workflow:execute'` → `WorkflowManager.execute`. (Popup, context menu, alarms,
   visit-web all funnel into the same path.)
2. **`WorkflowEngine.init()`** — ends with `addWorker({blockId: triggerBlock.id})`.
3. `worker` → `executeBlock(triggerBlock)`; the trigger handler is a pass-through returning
   `nextBlockId: getBlockConnections(id)`.
4. **Per block** (`WorkflowWorker.executeBlock`):
   1. Verify state still alive — dead state → `engine.destroy('stopped')`.
   2. Set `currentBlock`; testing-mode **breakpoint check** (`$breakpoint` flag) → stash
      `breakpointState`, mark state `breakpoint`, return (dashboard can resume/step).
   3. Resolve handler: `engine.blocksHandler[toCamelCase(block.label)]`; if missing and the block
      category is `'interaction'` → generic `interactionBlock` router.
   4. Build `refData = {prevBlockData, ...engine.referenceData, activeTabUrl}`.
   5. **Template the block**: `templating({block, data: refData, refKeys: blocks[label].refDataKeys})`
      → clone with all `{{ }}` placeholders replaced (only fields declared in `refDataKeys`!).
   6. Run handler as `handler(replacedBlock, {refData, prevBlock, ...execParam})` wrapped in
      `blockExecutionWrapper` (enforces `block.data.settings.blockTimeout`).
   7. Log entry via `engine.addLogHistory` (with `replacedValue`, duration, status); optional
      `settings.blockDelay` before continuing.
   8. `result.nextBlockId && !result.destroyWorker` → `executeNextBlocks(...)`; else
      `engine.destroyWorker(this.id)`.
5. **Continue** (`executeNextBlocks`): skip `disableBlock` nodes; **first connection continues on the
   same worker; every additional connection spawns a new worker** via `engine.addWorker` with a
   `cloneDeep` of `{windowId, loopList, activeTab, currentBlock, repeatedTasks, preloadScripts,
   debugAttached}` — that's how parallel branches work.
6. **Finish**: when the last worker has no next block → `destroyWorker` → `finish` log →
   `'finish'` event → `destroy('success')`: telemetry POST, `'destroyed'` event (notifications +
   workflow events), Dexie log write, connected-table flush, `reuseLastState` save.

### Edge / handle selection — there is no scheduler

Each handler decides its outgoing handle(s) by returning
`nextBlockId = this.getBlockConnections(id, handle)`. Handle ids:
`${blockId}-output-${outputIndex}` where `outputIndex` may be a number, `'fallback'`, a condition
id, or `2` (repeat/loop body). Examples:

| Block | Output logic |
|---|---|
| conditions | matched condition id, else `'fallback'` |
| element-exists | `data ? 1 : 2` |
| repeat-task | output `2` to loop, `1` to exit |
| while-loop | `isMatch ? 1 : 'fallback'` |
| webhook | `'fallback'` on network errors |
| loop-breakpoint | jumps back to `[{id: currentLoop.blockId}]` |

Legacy DrawFlow data is converted by `src/utils/convertWorkflowData.js` (DFS from trigger; the
**last output** of `conditions`/`BlockBasicWithFallback` is renamed `'fallback'`).

---

## 3.4 Data flow & the reference-data model

**Keywords** in `engine.referenceData`:
`variables` (incl. `$$`-persistent ones), `table` (aliases `dataColumn`/`dataColumns`),
`loopData` (`{[loopId]: {data, $index}}`), `googleSheets` (per `refKey`), `globalData`,
`workflow` (child-workflow results), `secrets` (Dexie credentials, decrypted on read); plus
per-call `prevBlockData`, `activeTabUrl`, and `functions`.

Writes:
- `worker.setVariable(name, value)` — `$push:name` appends to array; `$$name` persists to Dexie.
- `worker.addDataToColumn(column, value)` — column-major write with type coercion (`convertData`).

(The full templating syntax — `{{ }}`, `@` paths, `$func()`, `!!` JS mode — is Chapter 7.)

---

## 3.5 Background vs content-script contexts

- **Background-only handlers** (`src/workflowEngine/blocksHandler/`): trigger, delay, conditions,
  while-loop, loop-data, loop-breakpoint, repeat-task, blocks-group, block-package,
  wait-connections, webhook, export-data, google-sheets(+drive), google-drive, cookie, proxy,
  new-tab, new-window, switch-tab, active-tab, close-tab, reload-tab, go-back/forward-page,
  tab-url, element-exists, javascript-code, create-element, handle-dialog, handle-download,
  browser-event, clipboard, notification, parameter-prompt, insert-data, delete-data, data-mapping,
  sort-data, increase-variable, regex-variable, slice-variable, log-data, workflow-state,
  execute-workflow, ai-workflow, take-screenshot, save-assets, switch-to, hover-element, link.
- **Content-script handlers** (`src/content/blocksHandler/`): forms, get-text, event-click,
  press-key, trigger-event, upload-file, element-scroll, attribute-value, loop-data/loop-elements
  (selector collection), verify-selector, conditions (element checks).

Routing: any block whose registry category is `'interaction'` without a background handler goes
through `handlerInteractionBlock.js` → permission checks → `this._sendMessageToTab(block)` →
content script `blocksHandler[toCamelCase(label)]` → result → generic `saveData` handling
(`addDataToColumn`/`setVariable`).

**Tab state machine**: `worker.activeTab.id` is the cursor. `new-tab` creates/updates tabs
(customUserAgent via debugger override, tab groups, zoom, preload re-injection, `waitTabLoaded`);
`switch-tab` finds by match-pattern/title/index/next/prev; `active-tab` picks the focused
non-dashboard tab; `switch-to` handles iframes (same-origin → `frameSelector`, cross-origin →
`frameId` from `webNavigation.getAllFrames`); `proxy` sets `chrome.proxy.settings` and the engine
clears it on destroy.

**JS code block contexts** (`handlerJavascriptCode.js`): `data.context === 'background'` → sandbox
iframe; otherwise injected into the page's **MAIN world** (`script:execute` message), with CSP-
blocked pages falling back to `chrome.debugger Runtime.evaluate`. `everyNewTab` registers a
per-tab preload script re-injected on every tab switch. Results return via `__automa-next-block__`
DOM events; returned variables/tables/next-block-id are all honored.

---

## 3.6 Flow control

- **Conditions**: condition groups evaluated in order; first match wins → that condition's output;
  none → `'fallback'`. Item types: `data` (existence check), `value` (typed prefixes
  `json::|string::|number::|boolean::`), `code` (JS on page or sandbox), `element` (content script).
  Comparisons: `eq, eqi, nq, gt, gte, lt, lte, cnt, cni, nct, nci, stw, enw, rgx, itr, ifl`.
  Optional retry-polling (`retryConditions/retryCount/retryTimeout`).
- **Loops**: `loop-data` iterates `numbers | table | custom-data | data-columns | google-sheets |
  variable | elements`; iteration state in `worker.loopList[loopId]`; current item in
  `refData.loopData[loopId] = {data, $index}`; `maxLoop` caps. `loop-breakpoint` returns to
  `currentLoop.blockId` while items remain; element loops can `loadMoreAction` (scroll/click-link,
  waiting page load). `loop-elements` stores selectors in `worker.loopEls`. `repeat-task` loops N
  times via `worker.repeatedTasks`. `while-loop` re-evaluates conditions per iteration.
- **Try/catch semantics = per-block `onError`**: `{enable, retry, retryTimes, retryInterval,
  insertData, toDo: 'continue'|'fallback'|'error', errorMessage}`. Retry re-invokes the block after
  `retryInterval` seconds. Then workflow-level `settings.onError`: `keep-running` (continue with
  `error.data` as prevBlockData), `restart-workflow` (max `restartTimes` default 3), or
  `destroy('error')`.
- **Timeouts**: block `blockTimeout`, tab `tabLoadTimeout` (default 30s), webhook timeout,
  browser-event timeout, condition `retryTimeout`, wait-connections timeout.
- **Breakpoints** (testing mode): park worker in `breakpointState`, mark state `breakpoint`;
  dashboard `EditorDebugging.vue` sends `workflow:breakpoint` (pause), `workflow:resume`
  (`nextBlock: true` = single-step), `workflow:stop`.
- **Synchronization**: `wait-connections` barrier — parallel workers poll every second until all
  arrive or timeout; supports `specificFlow`; merged branches return `destroyWorker: true`.

---

## 3.7 State persistence & crash behavior

- **Live state**: `storage.local['workflowStates']` (debounced 1s) — used for badge count, dashboard
  live view, stop/resume/breakpoint commands, and the `log-data` block reading another run's state.
- **No true crash-resume**: on browser startup `onRuntimeStartup` deletes `workflowStates` and
  clears the badge — interrupted runs are simply gone; the durable record is the Dexie log written
  at destroy. Partial mitigations: `reuseLastState` (table/variables carried into the next run),
  `resumeLastWorkflow` in loop-data, `workflowQueue` in storage.
- **Logs**: written once in `destroy()` → Dexie `logs` DB. Skipped when `workflow.isTesting`.

---

## 3.8 Engine events & consumers

- `'update'` (per-block state snapshots), `'finish'`, `'destroyed'` (carries
  `status/message/history/blockDetail/timestamps`).
- Consumers: `WorkflowManager`'s `'destroyed'` listener (notifications + workflow events) and
  `handlerExecuteWorkflow` (parent waits on child completion). Live block progress reaches the UI
  via `states.update` → `storage.local['workflowStates']` → `storage.onChanged` in the dashboard →
  `workflowStore.states` → `EditorDebugging.vue` (canvas highlighting + variable/table inspector).

---

## 3.9 Schedules (cron / specific time)

- **Registration** (`src/utils/workflowTrigger.js`): `interval` → `alarms.create` with
  `periodInMinutes`; `date` → one-shot `when`; `cron-job` → next occurrence via `cron-parser`;
  `specific-day` → next matching weekday/time. Alarm name: `trigger:${workflowId}:${triggerId}`;
  `registerWorkflowTrigger` first runs `cleanWorkflowTriggers`.
- **Alarm → engine chain**: `alarms.onAlarm` → `BackgroundEventsListeners.onAlarms` →
  `BackgroundWorkflowTriggers.scheduleWorkflow` → load workflow → resolve trigger data →
  (interval + `fixedDelay` + already running → push to `workflowQueue`, drained by
  `executeQueue`) → skip stale `date` triggers → `executeWorkflow` → re-register next alarm.
- Other triggers: `visit-web` (webNavigation URL matching), context-menu (with clicked-element
  variables `$ctxLink`, `$ctxMediaUrl`, `$ctxElSelector`), `on-startup`, `keyboard-shortcut`.

---

## 3.10 The table, Google Sheets & export

- **Table model**: `engine.referenceData.table` = array of row objects; schema in `engine.columns`
  with per-column cursors; writes via `worker.addDataToColumn` with `convertData` coercion
  (`integer` strips non-digits, `boolean`, `array`, `string`). Connected tables live in Dexie and
  are re-synced at destroy.
- **Extraction sources**: interaction blocks (get-text/forms/attribute-value/screenshot),
  webhook, google-sheets, insert-data (parses CSV via Papa, XLSX via `xlsx`, JSON from files/URLs),
  data-mapping/sort-data, delete-data (clears columns).
- **Google Sheets**: actions `get | getRange | update | append | clear | create | add-sheet`;
  `get` stores rows into `refData.googleSheets[refKey]` (consumable by loop-data and export-data);
  API wrapper `src/utils/googleSheetsApi.js`; `google-sheets-drive` delegates with Drive-backed
  fetch.
- **Export** (`src/utils/dataExporter.js`): source `data-columns | google-sheets[refKey] |
  variable`; formats `plain-text (.txt)`, `json`, `csv` (Papa.unparse + optional BOM, configurable
  delimiter); downloaded via `chrome.downloads` with `onConflict` handling.

---

## 3.11 Known quirks (load-bearing)

- `restartWorkersCount` is never initialized — the `onError: restart-workflow` path can TypeError
  and fall through to engine error.
- `destroyWorker` fires `'finish'`/`destroy('success')` when the *last* worker ends, then destroys
  again if empty — guarded by `isDestroyed`.
- `isPopup` defaults to `true` and nothing sets it false — popup-context behaviors are always on.
- Blocks executed inside `blocks-group` are skipped in per-block history logging.
- The engine telemetry (`POST /workflows/logs/report`, daily `/status` ping) exists in this fork.
