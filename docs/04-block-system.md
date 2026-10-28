# Chapter 4 — The Block System: Registry, Handlers, Execution

> Blocks are the vocabulary of Automa. This chapter covers how they're defined, registered,
> executed in each context — and the exact recipe for adding a new one (chapter 5).

---

## 4.1 Where things actually live (and the naming trap)

**`src/execute/` is NOT the block-execution layer.** It's a tiny extension page (`src/execute/index.js`)
that runs a workflow from a URL (`execute.html#/<id>?var=...` by messaging `workflow:execute`).
The real block execution layers:

| Layer | Path | Runs in |
|---|---|---|
| Background handlers | `src/workflowEngine/blocksHandler/handler*.js` (~53 files) | engine context (offscreen on Chrome / background page on Firefox) |
| Content handlers | `src/content/blocksHandler/handler*.js` (21 files) | web page |
| **Registry** | `src/utils/shared.js` → `export const tasks` | editor + engine |
| Node components (canvas) | `src/components/block/*.vue` | vue-flow editor |
| Param panels (edit) | `src/components/newtab/workflow/edit/Edit*.vue` | block edit sidebar |
| Customization seam | `business/dev/blocks/{index,backgroundHandler,contentHandler,editComponents}.js` | all (currently empty stubs) |

Runtime block identity = the vue-flow node `label`, which is always the registry key
(e.g. `webhook`, `loop-data`). Handlers are resolved by `toCamelCase(label)`.

---

## 4.2 The registry — `tasks` in `src/utils/shared.js`

`export const tasks` holds **~62 blocks**. The merged app-wide view comes from `getBlocks()` in
`src/utils/getSharedData.js`: `{ ...tasks, ...customBlocks() }` where `customBlocks` = `@business/blocks`.

### Anatomy of one entry

```js
'element-scroll': {            // key = node label = engine handler lookup (toCamelCase)
  name: 'Scroll element',      // fallback display name
  description: '…',
  icon: 'riMouseLine',         // v-remixicon name; also 'path:<svgPathData>' or https URL
  tag: 'AI',                   // optional corner badge
  component: 'BlockBasic',     // canvas node component => node type `node-BlockBasic`
  editComponent: 'EditScrollElement', // param panel
  category: 'interaction',     // one of the 7 categories
  inputs: 1, outputs: 1,       // handle counts (informational)
  allowedInputs: true, maxConnection: 1,
  disableEdit: true,           // optional: no edit panel
  refDataKeys: ['selector'],   // which data fields get {{ }} templating before execution
  autocomplete: ['variableName'], // which fields feed editor autocomplete
  data: { disableBlock: false, description: '', …defaults },  // node default data
}
```

### Categories (`export const categories`, same file)

`interaction` (Web interaction, green), `browser` (orange), `general` (yellow), `onlineServices`
(red), `data` (lime), `conditions` (Control flow, blue), `package` (cyan). Each has Tailwind
`border`/`color` (bg + `fill-*` for the minimap).

### Category → blocks

- **general**: trigger, ai-workflow, execute-workflow, delay, export-data, webhook, blocks-group,
  clipboard, wait-connections, notification, note, workflow-state, parameter-prompt
- **browser**: active-tab, new-tab, switch-tab, new-window, proxy, go-back, forward-page, close-tab,
  take-screenshot, browser-event, handle-dialog, handle-download, reload-tab, tab-url, cookie
- **interaction**: event-click, get-text, element-scroll, link, attribute-value, forms,
  javascript-code, trigger-event, switch-to, upload-file, hover-element, save-assets, press-key,
  create-element
- **conditions**: repeat-task, conditions, element-exists, while-loop, loop-data, loop-elements,
  loop-breakpoint
- **data**: insert-data, delete-data, log-data, slice-variable, increase-variable, regex-variable,
  data-mapping, sort-data
- **onlineServices**: google-sheets, google-sheets-drive, google-drive · **package**: block-package

### How the editor & engine consume the registry

- **Palette**: `WorkflowDetailsCard.vue` iterates `getBlocks()`, groups by category, i18n-names each.
  **Any new `tasks` entry automatically appears in the palette** — no palette edit needed.
- **Canvas node components**: `WorkflowEditor.vue` auto-registers `require.context('@/components/block')`
  → `node-${ComponentName}` vue-flow node types; each `tasks.component` names one.
- **Edit panels**: `WorkflowEditBlock.vue` auto-registers `require.context('./edit')` merged with
  `customEditComponents()`; resolved from `tasks[label].editComponent`.
- **Engine**: `WorkflowEngine` re-reads `getBlocks()` as `blocksDetail` for every worker (used for
  `refDataKeys`, the interaction-category fallback, and log filtering).
- **Validation map**: `src/newtab/utils/blocksValidation.js` keyed by block id.
- **Handler registration**: `src/workflowEngine/blocksHandler.js` = `require.context('./blocksHandler')`,
  strips `./handler` prefix, camelCases the rest, merges `customHandlers()` from `@business`.
  So `handlerLoopData.js` → `blocksHandler.loopData`. **Dropping a file into the folder IS the
  registration.**

---

## 4.3 Writing a block handler (the contract)

### Dispatch & injected context (`WorkflowWorker.executeBlock`)

1. Handler lookup: `this.engine.blocksHandler[toCamelCase(block.label)]`; if missing **and**
   `blocksDetail[label].category === 'interaction'` → generic `interactionBlock` router.
2. Templating pass first: `templating({block, data: refData, refKeys: blocksDetail[label].refDataKeys})`
   replaces `{{ }}` in the fields listed by `refDataKeys`.
3. Invocation: `handler.bind(this, replacedBlock, { refData, prevBlock, ...execParam })`.

So the **signature is `handler(block, ctx)`** where:
- `this` = the `WorkflowWorker`. Common members: `this.getBlockConnections(blockId, outputIndex=1)`,
  `this.setVariable(name, value)`, `this.addDataToColumn(column, value)`, `this.activeTab`,
  `this.loopList`, `this.loopEls`, `this.repeatedTasks`, `this.engine`, and
  `this._sendMessageToTab(payload, options)` (messages the content script with auto-injection retry).
- Arg 1 `block`: `{ id, label, data: {…template-expanded…}, settings }`.
- Arg 2 `ctx`: `{ refData, prevBlock, prevBlockData, targetHandle, sourceHandle,
  nextBlockBreakpointCount, resume }`.

**Return contract**: resolve `{ data, nextBlockId, status?, logId?, ctxData?, replacedValue?,
destroyWorker? }`. `nextBlockId` is an array from `this.getBlockConnections(id[, outputName])`;
returning without `nextBlockId` (or falsy) ends the worker. **Throw an `Error` to trigger the
block's onError pipeline** (retry / insert data / fallback / continue / error). A per-block timeout
wrapper honors `block.data.settings.blockTimeout`.

### Three representative handler anatomies

**A) Trivial background block — `handlerDelay.js`** (whole file ~13 lines): a plain function using
`block.data.time`, `setTimeout`, resolving `{ data: '', nextBlockId: this.getBlockConnections(block.id) }`.
No imports, no ctx destructuring.

**B) Background + templating + multiple outputs — `handlerWebhook.js`**:
`async function webhook({ data, id }, { refData })`. Renders each header value via
`renderString(value, refData, this.engine.isPopup)`, calls the util `executeWebhook`, uses **two
outputs**: primary `this.getBlockConnections(id)` and fallback `this.getBlockConnections(id,
'fallback')` for HTTP/network failures; assigns variables (`this.setVariable`), writes table columns
(`this.addDataToColumn`, incl. `$assignColumns` array mode), attaches `error.ctxData` for logs, and
sets `error.nextBlockId` so the engine logs the correct path.

**C) Flow-control with loop state — `handlerLoopData.js`**: keyed off `data.loopId` in
`this.loopList`. First pass materializes the iterable per `data.loopThrough`
(`numbers | data-columns | custom-data | table | google-sheets | variable | elements` — `elements`
goes through `this._sendMessageToTab` and registers `this.loopEls`), writes
`refData.loopData[loopId] = { data, $index }` (this is what `{{loopData@loopId}}` reads), snapshots
via `this.engine.addRefDataSnapshot('loopData')`, and returns the current item. Companion
`handlerLoopBreakpoint.js` jumps back to `[{ id: currentLoop.blockId }]` until exhausted.

**D) Interaction (in-page) blocks** — no per-block background handler needed. `handlerInteractionBlock.js`
checks permissions, optionally attaches the Chrome debugger, then `this._sendMessageToTab(block)` —
the **entire block payload** goes to the page. `src/content/index.js` dispatches
`handlers[toCamelCase(data.name || data.label)]` as `handler(data, { handleSelector })`. Content
handlers resolve elements via `handleSelector(block, { onSelected, onError, onSuccess })` (built on
`src/utils/FindElement.js`) and return the value; the background then does generic
`assignVariable`/`saveData` handling. **Any block whose registry category is `'interaction'` gets
this routing for free.**

---

## 4.4 Param UI (edit panels)

- Folder: `src/components/newtab/workflow/edit/Edit*.vue`, **auto-registered by filename** — no
  import list to touch.
- Contract: prop `data` (the block data) + `block-id`; emit `update:data` with the full merged
  object (`emit('update:data', { ...props.data, ...value })`). The panel title uses
  `t('workflow.blocks.${id}.name')` and links docs to
  `https://docs.extension.automa.site/blocks/${blockId}.html`.
- Shared building blocks: `EditInteractionBase.vue` (description + findBy select + element picker +
  selector textarea + timeout/multiple options), `EditAutocomplete.vue` (mustache-triggered
  dropdown), `EditBlockSettings.vue` + `BlockSetting/` (general/lines/on-error tabs),
  `InsertWorkflowData.vue` (assign-variable / save-to-table duo), `Trigger/` (9 trigger
  sub-panels), `Parameter/` (param input widgets).
- Examples: `EditDelay.vue` (25 lines, one input), `EditScrollElement.vue` (extends
  `EditInteractionBase`).

---

## 4.5 Special blocks deep-dive

- **Trigger** (`handlerTrigger.js` is a pass-through): trigger *types* are data
  (`data.type` / `data.triggers[]`), not separate blocks — registration lives in
  `src/utils/workflowTrigger.js` `workflowTriggersMap` (interval, date, cron-job, visit-web,
  on-startup, specific-day, context-menu, keyboard-shortcut). Parameters (`data.parameters`) are
  prompted via the `params.html` page.
- **Conditions**: node component `BlockConditions.vue` renders **one output Handle per condition
  item** (`${id}-output-${item.id}`) + a `fallback` output; `handlerConditions.js` picks
  `outputId = condition.id` or `'fallback'`, evaluating via `testConditions.js` +
  `compareBlockValue.js` + the `conditionBuilder` schema. `while-loop` reuses the same tester with
  output 1 / `'fallback'`.
- **element-exists**: two outputs — `getBlockConnections(id, data ? 1 : 2)`.
- **Loops**: loop-data / loop-elements (with `loadMoreAction`) / loop-breakpoint — see chapter 3.6.
- **blocks-group**: created by the canvas "group" action; `BlockGroup.vue` renders children inline;
  the handler flattens children into `this.engine.blocks` + `connectionsMap` and returns the first
  child as next. Grouping restricted by `excludeGroupBlocks`.
- **block-package**: reusable sub-flow with named input/output handles, stored in the package store
  (`storage.local 'savedBlocks'`); `BlockPackage.vue` renders dynamic `Handle ${id}-input-${input.id}`
  lists; the handler lazily extracts inner nodes into the engine graph with prefix
  `${pkgNodeId}__${itemId}`, routing named inputs/outputs to inner handles; results cached in
  `engine.packagesCache`.
- **execute-workflow**: loads the other workflow, spawns a **nested `WorkflowEngine`** sharing
  `states/logger/blocksHandler`, guards infinite recursion, and on destroy copies child
  `table`+`variables` into `refData.workflow[executeId]`.
- **wait-connections**: barrier synchronizing multiple workers via `engine.waitConnections`;
  supports `destroyWorker` return for merged branches.
- **ai-workflow**: fork-specific block needing `settings.aipowerToken`.

---

## 4.6 Handler inventory (quick reference)

**Background** (`src/workflowEngine/blocksHandler/`): activeTab, aiWorkflow, blockPackage,
blocksGroup, browserEvent, clipboard, closeTab, conditions, cookie, createElement, dataMapping,
delay, deleteData, elementExists, executeWorkflow, exportData, forwardPage, goBack, googleDrive,
googleSheetsDrive, googleSheets, handleDialog, handleDownload, hoverElement, increaseVariable,
insertData, interactionBlock (generic router), javascriptCode, link, logData, loopBreakpoint,
loopData, loopElements, newTab, newWindow, notification, parameterPrompt, proxy, regexVariable,
reloadTab, repeatTask, saveAssets, sliceVariable, sortData, switchTab, switchTo, tabUrl,
takeScreenshot, trigger, waitConnections, webhook, whileLoop, workflowState.

**Content** (`src/content/blocksHandler/`): attributeValue, clipboard, conditions, createElement,
elementExists, elementScroll, eventClick, forms, getText, hoverElement, javascriptCode, link,
loopData, loopElements, pressKey, saveAssets, switchTo, takeScreenshot, triggerEvent, uploadFile,
verifySelector.
