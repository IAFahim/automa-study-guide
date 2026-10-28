# Chapter 6 — The Dashboard & Workflow Editor (`src/newtab`)

> Everything the user sees when they open Automa's dashboard lives in `src/newtab`. This chapter maps
> the whole UI: routes, the workflow editor built on **vue-flow**, state management (Pinia), the
> element selector, the recorder, packages, teams, and i18n.

---

## 6.1 App shell & entry points

- **Entry**: `src/newtab/index.js` — creates the Vue app with `@vueuse/head`, `vue-router`,
  `compsUi`, **pinia** (`src/lib/pinia.js`), `vue-i18n` (`src/lib/vueI18n.js`), vue-toastification,
  and `v-remixicon`. The HTML template is `src/newtab/index.html`.
- **Shell**: `src/newtab/App.vue`. It renders the `AppSidebar` icon rail (hidden on `/recording`),
  the `<router-view>`, the global `AppLogs` drawer, the auth dialog, an "extension updated" toast and
  the shared-permissions modal.
- **Startup sequence** (`App.vue`, ~lines 324–397): loads every store (`folderStore.load()`,
  `store.loadSettings()`, `workflowStore/teamWorkflowStore/hostedWorkflowStore/packageStore.loadData()`),
  loads locale, runs `dataMigration()`, `userStore.loadUser()`, the `automa('app')` business hook,
  fetches shared workflows + cloud backup, auto-deletes old logs (`deleteLogAfter`), auto-redirects
  to `/recording` if the `isRecording` storage flag is set. It also keeps a single dashboard tab
  alive (closes duplicates), mirrors running states from `browser.storage.local` key `workflowStates`,
  and warns on unload when there are unsaved changes or running workflows.

### Route map (`src/newtab/router.js`, hash history)

| Route | Page | What it does |
|---|---|---|
| `/` | redirect → `/workflows` | home |
| `/workflows` | `pages/workflows/index.vue` | workflow list with browser-like tabs (`team / local / shared / host`) |
| `/workflows/:id` | `pages/workflows/[id].vue` | **the workflow editor** (~1700 lines, the heart of the UI) |
| `/teams/:teamId/workflows/:id` | same `[id].vue` | team workflow editor (route name `team-workflows`) |
| `/workflows/:id/host` | `pages/workflows/Host.vue` | read-only hosted workflow view + sync |
| `/workflows/:id/shared` | `pages/workflows/Shared.vue` | shared-with-community workflow page |
| `/packages`, `/packages/:id` | `pages/Packages.vue`, `[id].vue` | packages ("blocks groups") list & editor |
| `/schedule` | `pages/ScheduledWorkflow.vue` | all scheduled triggers across workflows |
| `/storage`, `/storage/tables/:id` | `pages/Storage.vue`, `storage/Tables.vue` | tables / variables / credentials |
| `/logs/:id?` | `pages/logs/[id].vue` | emits a `ui:logs` mitt event; `AppLogs` drawer is the real UI |
| `/recording` | `pages/Recording.vue` | live recorder screen |
| `/settings` (+`/profile`, `/about`, `/backup`, `/editor`, `/shortcuts`) | `pages/Settings.vue` + `settings/*` | settings incl. cloud/local backup, editor prefs, shortcuts |
| `/welcome` | `pages/Welcome.vue` | onboarding |
| `/json-render` | `pages/JsonRenderDemo.vue` | demo page for `@automa-platform/dashboard-components` |

The workflows container (`pages/Workflows.vue`) is a draggable tab strip (`vuedraggable`) wrapping a
`<keep-alive>` router-view — the dashboard hosts multiple workflow-editor tabs at once. The sidebar
(`src/components/newtab/app/AppSidebar.vue`) is a 64px icon rail with Mousetrap shortcuts
(`option+1|w|t|l|a|s`), a logs badge showing running count, and a button that injects the element
selector into the active tab.

---

## 6.2 The workflow editor (vue-flow)

**Page**: `src/newtab/pages/workflows/[id].vue` — three modes derived from the route: local
workflow / `isTeamWorkflow` / `isPackage`. Left sidebar swaps between `WorkflowEditBlock` (editing a
block's params) and `WorkflowDetailsCard` + `WorkflowBlockList` (the block palette + pinned blocks).
Modals map: **table** (`WorkflowDataTable`), **workflow-share**, **workflow-share-team**,
**global-data** (`WorkflowGlobalData`), **settings** (`WorkflowSettings`).

**Canvas**: `src/components/newtab/workflow/WorkflowEditor.vue`
- A `<vue-flow>` with `Background`, `MiniMap` (node color = block category color), zoom controls,
  and `EditorSearchBlocks` (Ctrl+B autocomplete over all blocks).
- **Node components are auto-registered** via `require.context('@/components/block')` →
  `node-BlockBasic`, etc. A `fallbackBlocks` map renders `BlockExportData` and `BlockWebhook` with
  `BlockBasic` / `BlockBasicWithFallback` components.
- vue-flow options: `edgeUpdaterRadius: 20`, `deleteKeyCode: 'Delete'`,
  `multiSelectionKeyCode: Meta/Control`, min/max zoom + arrow markers + snap-to-grid from settings.
- Custom edge: `EditorCustomEdge.vue` (BaseEdge + bezier/smoothstep path).

**Node components** (`src/components/block/*.vue`):
- `BlockBase.vue` is the wrapper: hover menu (copy block id, delete, settings, drag-into-group,
  enable/disable, **run-from-here**, edit) and a testing-mode **breakpoint** toggle
  (`data.$breakpoint`).
- `BlockBasic.vue` renders vue-flow `Handle`s using the id convention
  `${id}-input-1` (left), `${id}-output-1` (right), plus `${id}-output-fallback` when the block's
  `onError.toDo === 'fallback'`. Live validation errors come from `useBlockValidation`.
- Special node components: `BlockConditions`, `BlockGroup`/`BlockGroup2`, `BlockLoopBreakpoint`,
  `BlockNote`, `BlockPackage` (dynamic input/output handles `${id}-input-${input.id}`),
  `BlockRepeatTask`, `BlockDelay`, `BlockElementExists`, `BlockBasicWithFallback`.

### Drag & drop (`[id].vue` onDragoverEditor/onDropInEditor; `src/utils/editor/DroppedNode.js`)

The palette sets `dataTransfer('block', JSON.stringify(block))`; saved blocks/packages use
`'savedBlocks'`. On drop, four behaviors:

1. **over a node** → `DroppedNode.replaceNode` (rewires all edges, preserves `onError`);
2. **over a source handle** → `appendNode` (connect);
3. **over an edge** → `insertBetweenNode` (splice into the connection, respecting outputs);
4. **else** → plain `addNodes` at the projected canvas position.

A `trigger` block can only be added once; packages exclude `trigger/block-package/execute-workflow`.
Connection validation (`onEdgesChange`) removes any edge whose sourceHandle **and** targetHandle both
contain `output`; `onEdgeUpdate` blocks output→output reconnection; `onConnectEnd` snaps a drop on
empty canvas to the nearest node's first target handle.

### Editor power features

- **Context menu** (`editor/EditorLocalCtxMenu.vue`): node → copy/duplicate/save-as-package/
  delete (+group or ungroup); **on a handle** → "Record from here" (output handles) or "Set as block
  input/output" (package IO); edge → delete; pane → paste.
- **Group/ungroup** (`[id].vue` ~861–929): group = one `blocks-group` node whose
  `data.blocks = [{id: <blockLabel>, itemId, data}]`; ungroup re-expands to nodes with fresh edges
  `${itemId}-output-1 → ${next}-input-1`.
- **Auto-align** (`autoAlign()` ~1023–1076): a **dagre** graph (`rankdir:'LR'`, `ranksep:100`,
  `ranker:'tight-tree'`) positions nodes, then `fitView()`.
- **Undo/redo**: `src/composable/commandManager.js` (history stack, max 100) +
  `src/utils/editor/EditorCommands.js` (node-added/removed commands; edge commands exist but are
  currently unused). Keys `mod+z` / `mod+shift+z`.
- **Copy/paste**: clipboard JSON `{name:'automa-blocks', data:{nodes, edges}}`; paste requests the
  `clipboardRead` permission; `copyElements()` re-ids nodes with nanoid and offsets positions.
- **Keyboard shortcuts** (Mousetrap): `mod+c/v/z/shift+z`, `Delete`, `mod+[` sidebar,
  `mod+option+d` duplicate, `mod+b` search blocks, `mod+shift+s` save, `option+enter` execute.
  Overridable in `localStorage.shortcuts`.
- **Saving** (`EditorLocalActions.saveWorkflow()`): `editor.toObject()`, strip
  `edge.sourceNode/targetNode`, require a `trigger` node, write `drawflow` + `trigger` + `version`
  into the store, then `registerWorkflowTrigger`. Dirty state = red pulsing dot; route-leave and
  `beforeunload` guards protect unsaved work.

### Debug / testing mode

The workflow `testingMode` toggle (bug icon; requires no unsaved changes). With a running state,
`editor/EditorDebugging.vue` shows pause / resume / **step-next-block** / stop (messages
`workflow:breakpoint|resume|stop` to the background), live ctxData JSON (CodeMirror), per-block log
list, and highlights the currently running block on canvas (`.current-running` ring). Breakpoints
are set per-node via `BlockBase`'s circle icon — this is the single best tool for understanding
block data at runtime.

---

## 6.3 Running, settings, export/import

- **Run from editor**: `EditorLocalActions.executeCurrWorkflow()` →
  `src/service/renderer/RendererWorkflowService.executeWorkflow()` (strips Vue proxies with `toRaw`,
  sends `workflow:execute` message to background; `isTesting: isDataChanged`). Run-from-block adds
  `options.blockId` + the active `tabId`.
- **Workflow settings** (`WorkflowSettings.vue` tabs General/Table/Blocks/Events):
  `onError: keep-running|stop-workflow|restart-workflow` (+`restartTimes`), `execContext
  popup|background`, `notification`, `debugMode`, `inputAutocomplete`, `reuseLastState`, `saveLog`,
  `executedBlockOnWeb`, clear cache, `publicId` (used by the JS-CustomEvent trigger), table
  `insertDefaultColumn`/`defaultColumnName`, per-block default delay, and events. Defaults live in
  `stores/workflow.js defaultWorkflow()`. Per-block **onError** (`edit/BlockSetting/BlockSettingOnError.vue`)
  adds `retry`, `retryTimes`, `retryInterval`, `toDo: error|continue|fallback`, `insertData`.
- **Export/import** (`src/utils/workflowData.js`): export produces `<name>.automa.json`;
  import reads it and registers triggers. Export embeds `includedWorkflows` (follows
  `execute-workflow` refs, maxDepth 3). Protected workflows can't be exported. A permissions check
  (`getWorkflowPermissions`: contextMenus, clipboardRead, notifications, downloads, cookies) shows
  `SharedPermissionsModal`.
- **Host & sync**: `EditorLocalActions.setAsHostWorkflow` uploads `convertWorkflow(workflow,['id'])`
  to `POST /me/workflows/host`; sync via `hostedWorkflowStore.fetchWorkflows([hostId])`
  (`POST /workflows/hosted` with `{hosts:[{hostId, updatedAt}]}` → statuses `deleted|updated`).
  The hosted editor is fully disabled (drag/connect/edit flipped off in `WorkflowEditor` watch).

### Workflow JSON export format

```jsonc
{
  "extVersion": "1.30.00",
  "name": "...", "icon": "riGlobalLine", "description": "...",
  "version": "1.28.1",                 // last extension version that saved it
  "settings": { "publicId": "", "blockDelay": 0, "saveLog": true, "debugMode": false,
                "restartTimes": 3, "notification": false, "execContext": "popup",
                "reuseLastState": false, "inputAutocomplete": true,
                "onError": { "select": "keep-running", ... }, "executedBlockOnWeb": true,
                "insertDefaultColumn": true, "defaultColumnName": "column" },
  "table": [ { "id", "name", "type" } ],   // legacy alias: dataColumns
  "globalData": "{ ... }",             // JSON string
  "drawflow": {                        // legacy field name — now a vue-flow shape
    "x": 0, "y": 0, "zoom": 1.3,
    "nodes": [{
      "id": "a1b2c3d",                 // nanoid(7)
      "label": "get-text",             // block id (key of `tasks` registry)
      "type": "BlockBasic",            // node component name
      "position": { "x": 100, "y": 300 },
      "data": { /* block settings + disableBlock, description, onError{...} */ }
    }],
    "edges": [{
      "id": "edge-xyz",
      "source": "a1b2c3d", "target": "e4f5g6h",
      "sourceHandle": "a1b2c3d-output-1",   // or -output-fallback / -output-<condId> / -output-<pkgOutputId>
      "targetHandle": "e4f5g6h-input-1"
    }]
  },
  "includedWorkflows": { "<execute-workflow id>": { /* same shape, nested */ } }
}
```

**Handle-id convention**: `${nodeId}-input-1`, `${nodeId}-output-1`, `-output-fallback`, conditions
`-output-<cond.id>`, package IO `-input-<inputId>` / `-output-<outputId>`. Legacy DrawFlow JSON
(`{drawflow:{Home:{data:{pos_x, outputs.connections}}}}`) is auto-converted on editor mount by
`src/utils/convertWorkflowData.js` (it traces the graph from the trigger block and rebuilds handles).

> Study trick: export a small workflow as JSON and read it alongside the canvas — this is the
> fastest way to internalize the format.

---

## 6.4 State management — Pinia (not Vuex)

`src/lib/pinia.js` adds a `saveToStorage(key)` plugin that maps stores → `browser.storage.local`
via a `storageMap`. Note: **vuex / @vuex-orm are declared in package.json but unused** in app code.

| Store (`src/stores/`) | storage.local key | Role |
|---|---|---|
| `main.js` (`useStore`) | `tabs`, `settings` | dashboard settings (locale, `deleteLogAfter`, `logsLimit`, editor zoom/arrow/snap/lineType), Google Drive/Sheets state |
| `workflow.js` | `workflows` | **local workflows** map `{id: workflow}`, running `states` + `popupStates` (mirrored from `workflowStates`), seeds 3 sample workflows on first run |
| `teamWorkflow.js` | `teamWorkflows` | `{teamId: {workflowId: wf}}` collaboration |
| `hostedWorkflow.js` | `workflowHosts` | hosted workflows + sync API |
| `sharedWorkflow.js` | (sessionStorage cache) | community workflows from `GET /me/workflows/shared` |
| `package.js` | `savedBlocks` | **packages/blocks-groups**: `{id,name,icon,inputs,outputs,variable,settings.asBlock,data:{nodes,edges}}` |
| `folder.js` | `folders` | workflow folders |
| `user.js` | `user`, `backupIds` | profile, team access validation |

Key insight: **workflows themselves live in `browser.storage.local`, not in IndexedDB** — logs live
in Dexie (see chapter 8).

---

## 6.5 Database layer (Dexie)

- `src/db/logs.js` — DB **`logs`**: tables `items (++id, name, endedAt, workflowId, status,
  collectionId)`, `ctxData (++id, logId)`, `logsData (++id, logId)`, `histories (++id, logId)`.
  Written by `src/workflowEngine/WorkflowLogger.js`; auto-deleted by age in `App.vue`.
- `src/db/storage.js` — DB **`storage`**: `tablesData (++id, tableId)`, `tablesItems (++id, name,
  createdAt, modifiedAt)`, `variables (++id, &name unique)`, `credentials (++id, &name unique)`.
- **Recordings are NOT in Dexie** — they live in `browser.storage.local` keys `isRecording` /
  `recording`.
- Migration: `src/utils/dataMigration.js` moves the legacy `logs` array from storage.local into
  Dexie (max 500 logs).

---

## 6.6 The element selector

**Dashboard side**: `src/newtab/utils/elementSelector.js` — `initElementSelector(tab)` injects
`elementSelector.bundle.js` into all frames (message `automa-element-selector`); `selectElement`
opens a `tabs.connect` port and resolves the picked selector; `verifySelector` sends
`verify-selector` to the content script (auto CSS vs XPath via `isXPath`). Used by the
"select element"/"verify" buttons (`SharedElSelectorActions.vue`) inside edit forms, and by the
sidebar's selector button.

**Content side**: `src/content/elementSelector/` (webpack entry `elementSelector`) — a shadow-DOM
Vue app: draggable card with selector input, CSS/XPath toggle, "list" (multi-element) mode, selector
settings (include id / tag name / class / attributes; `attrNames` default `data-testid`, persisted
in storage.local `selectorSettings`), parent/child path navigation, highlight SVG overlays (yellow
hovered, blue selected), Space-to-select, iframe support (`frameSelector|>innerSelector` protocol).

**Selector generation**: `src/lib/findSelector.js` wraps **`@medv/finder`** (default options force
tagName + id + `data-testid`); list selectors are composed as `listSelector > childSelector` with an
`automa-el-list` marker attr; XPath comes from a hand-rolled `generateXPath()` in
`src/content/utils.js`. Runtime lookup of stored selectors: `src/utils/FindElement.js`
(`cssSelector`/`xpath`, shadow-DOM aware via `query-selector-shadow-dom`).

> Gotcha: `css-selector-generator` is declared in package.json but **not referenced anywhere in
> src** — `@medv/finder` is the actual generator.

---

## 6.7 The recorder

- **Start**: `src/newtab/utils/startRecordWorkflow.js` — sets
  `storage.local {isRecording:true, recording:{flows:[], name, activeTab, workflowId?, connectFrom?}}`,
  badges the action button `rec`, injects `recordWorkflow.bundle.js` into every http tab (allFrames).
  Started from the Workflows page or the editor ctx-menu "Record from here"
  (requires a saved workflow; passes `connectFrom:{id: nodeId, output: handleId}`).
- **Capture** (`src/content/services/recordWorkflow/`): DOM listeners in capture phase (ignoring
  Automa's own UI):
  - click → `event-click` block (`link` block for `<a>`);
  - change → `forms` (select) / `upload-file` / `trigger-event` (change);
  - keydown → `forms` (text-field + Enter submit) or `press-key` (consecutive keys grouped by
    `groupId`);
  - scroll → `element-scroll` (merged);
  - focusin marks text fields with `data-automa-el-selector` and records `forms` text-field on
    input (debounced).
  Iframe events are forwarded to top with a `frameSelector |> selector` prefix. Each event appends
  `{id, description, data:{selector,...}}` to `recording.flows`.
- **Tab/nav events** (`RecordWorkflowUtils.js`, wired in `Recording.vue`): `tabs.onCreated` →
  `new-tab`; `tabs.onActivated` → `switch-tab`; `webNavigation.onCommitted` (link/typed) updates the
  `new-tab` flow URL; `onCompleted` re-injects the recorder.
- **Conversion to workflow** (`Recording.vue generateDrawflow()`): builds vue-flow nodes/edges
  linearly (trigger optional when connecting into an existing workflow), each flow `{id}` maps to
  `tasks[id]` block defaults via `defu`; consecutive `press-key`s collapse into a `blocks-group`
  node; layout 5 nodes per row. Stop → inserts a new workflow or splices nodes+edges into the
  existing one at `connectFrom`.

---

## 6.8 Packages ("blocks groups")

- **Create**: editor ctx-menu "Save blocks to packages" → `EditorAddPackage.vue`
  (name/description/icon/`asBlock`) → stored via `packageStore.insert` → `storage.local
  'savedBlocks'`.
- **Use**: `EditorLocalSavedBlocks.vue` lists packages, draggable; dropped as either a
  `block-package` node (`asBlock:true`) or expanded into nodes+edges. `BlockPackage.vue` renders
  dynamic input/output handles; if the node's package isn't installed it offers install (external
  packages have an `author`) or update (re-diffs handles, removes orphaned edges).
- **Package editor**: route `/packages/:id` reuses the workflow editor with `isPackage` mode; ctx-menu
  "Set as block input/output" fills `workflow.inputs/outputs` (`{handleId, name, id, blockId}`);
  trigger/execute-workflow/block-package blocks are blocked inside a package; external packages are
  read-only ("Preview mode").
- **Sharing**: cloud packages via `POST/PATCH/DELETE /packages`, user list `GET /me/packages`.
- **Execution**: `handlerBlocksGroup.js` expands `data.blocks` into `engine.blocks` and stitches
  internal connections sequentially; `handlerBlockPackage.js` extracts the package's
  `data.data.nodes/edges` into the engine with prefix `${pkgNodeId}__${itemId}`, maps package inputs
  to internal entry blocks and output handles to external connections; results cached in
  `engine.packagesCache`.

---

## 6.9 Teams / collaboration / business hooks

- **Teams**: `WorkflowShareTeam.vue` publishes to `POST /teams/{teamId}/workflows`; the team editor
  route enforces `haveEditAccess = userStore.validateTeamAccess(teamId, ['edit','owner','create'])`;
  read-only users get a sync button (overwrite + `check-update?updatedAt=` poll).
- **Community share**: `WorkflowShare.vue` → `POST /me/workflows/shared`.
- **business/**: `business/dev/index.js` is an empty `export default function(){}` aliased as
  `@business`; `business/dev/blocks/index.js` returns `{}` (custom blocks registry). This is an
  upstream OEM hook — `automa('app')` is called in `App.vue`, and `@business/blocks` is merged into
  the block registry via `src/utils/getSharedData.js`. When forking, you can inject custom blocks
  here without touching core code.

---

## 6.10 i18n

`src/locales/{en,es,fr,it,pt-BR,tr,uk,vi,zh,zh-TW}/`, each with `common.json`, `newtab.json`,
`popup.json`, `blocks.json`. `src/lib/vueI18n.js` lazy-loads per-locale chunks in order
common → popup → page-scoped (`newtab`|`popup`) → blocks, merging into `en` fallback first. Many
block-name lookups guard with `te()` and fall back to `tasks[id].name`.

---

## 6.11 Validation & autocomplete extras

- `src/newtab/utils/blocksValidation.js` (~400 lines): per-block `func(data)` validators (trigger
  cron/context-menu/date/url/shortcut, tabs, proxy, screenshot, sheets, webhook…) surfaced as hover
  tooltips on canvas nodes via `src/composable/blockValidation.js`.
- Editor autocomplete (`src/utils/editor/editorAutocomplete.js`) exposes `loopData`, `variables`,
  `googleSheets`, `table`, `$func` tokens to CodeMirror.
