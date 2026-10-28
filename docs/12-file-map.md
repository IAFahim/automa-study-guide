# Chapter 12 — File Map & Glossary

## 12.1 Directory map (annotated)

```
automa/
├── webpack.config.js            # all entries, aliases, manifest plugin, hot-reload lists
├── utils/                       # build.js, webserver.js (dev server :3001), build-zip.js
├── business/                    # @business alias — empty OEM/business seam (blocks, params)
├── secrets.blank.js             # template for the gitignored secrets file
└── src/
    ├── manifest.chrome.json     # MV3 (min Chrome 116)
    ├── manifest.chrome.dev.json # dev name/icon variant
    ├── manifest.firefox.json    # MV2, persistent background, menus permission
    │
    ├── background/              # service worker (thin router on Chrome)
    │   ├── index.js             #   listeners, message table, JS-block injection, downloads
    │   ├── BackgroundEventsListeners.js     # alarms/commands/webNavigation/startup
    │   ├── BackgroundWorkflowTriggers.js    # schedule/visit-web/context-menu trigger execution
    │   ├── BackgroundWorkflowUtils.js       # bridge to engine host; getWorkflow resolver
    │   ├── BackgroundOffscreen.js           # offscreen doc lifecycle ('halo' warm-up)
    │   └── BackgroundUtils.js               # openDashboard, sendMessageToDashboard
    │
    ├── offscreen/               # MV3 engine host + hidden sandbox iframe
    │   ├── index.js / message-listener.js   # 'offscreen' prefix messages → WorkflowManager
    │   └── index.html            # includes <iframe src="/sandbox.html" id="sandbox">
    │
    ├── sandbox/                 # manifest-sandboxed page (can eval)
    │   └── index.js              # fetchResponse, conditionCode, blockExpression, javascriptBlock
    │
    ├── workflowEngine/          # ★ the engine (runs in offscreen doc / Firefox background)
    │   ├── WorkflowManager.js   # singleton: execute/stop/resume, post-run events, telemetry
    │   ├── WorkflowEngine.js    # per-run: init, connectionsMap, referenceData, destroy
    │   ├── WorkflowWorker.js    # per-path: executeBlock, branching, loops, tab management
    │   ├── WorkflowState.js     # running states → storage.local 'workflowStates' + badge
    │   ├── WorkflowLogger.js    # 4-table Dexie log write at destroy
    │   ├── workflowEvent.js     # post-run webhook/JS automations
    │   ├── blocksHandler.js     # require.context registry (handlerX.js → camelCase key)
    │   ├── blocksHandler/       # ~53 background handlers
    │   ├── templating/          # index.js, renderString.js, mustacheReplacer.js, templatingFunctions.js
    │   └── utils/               # testConditions, conditionCode, javascriptBlockUtil, webhookUtil
    │
    ├── content/                 # content scripts
    │   ├── index.js             # main script: iframe protocol, block dispatch, messages
    │   ├── blocksHandler/       # 21 in-page handlers
    │   ├── elementSelector/     # picker UI (shadow-DOM Vue app)
    │   ├── services/
    │   │   ├── recordWorkflow/  # recorder capture + toolbar
    │   │   ├── webService.js    # automa.site bridge (__automa-ext__)
    │   │   └── shortcutListener.js # Mousetrap shortcuts + __automaExecuteWorkflow
    │   ├── commandPalette/      # in-page palette (<automa-palette>)
    │   └── elementObserver.js   # disabled element-change trigger
    │
    ├── newtab/                  # dashboard SPA
    │   ├── index.js / index.html / router.js / App.vue
    │   ├── pages/               # workflows/[id].vue (the editor), Workflows, Host, Shared,
    │   │                        # Packages, ScheduledWorkflow, Storage, Logs, Recording,
    │   │                        # Settings, Welcome
    │   └── utils/               # elementSelector.js, startRecordWorkflow.js, blocksValidation.js
    │
    ├── popup/                   # toolbar popup (run/pin/record/pick)
    ├── params/                  # params.html — workflow parameter forms
    ├── execute/                 # execute.html — URL deep-link workflow launcher
    │
    ├── components/
    │   ├── block/               # canvas node components (BlockBase, BlockBasic, BlockConditions…)
    │   ├── newtab/
    │   │   ├── workflow/edit/   # ~60 Edit*.vue param panels (+ Trigger/, Parameter/)
    │   │   ├── workflow/editor/ # EditorDebugging, ctx menu, search, saved blocks
    │   │   ├── shared/          # SharedWorkflowTriggers, SharedElSelectorActions…
    │   │   ├── storage/         # tables/variables/credentials UI
    │   │   └── app/             # AppSidebar, AppLogs, settings pages
    │   └── ui/                  # 26 Ui*.vue primitives
    │
    ├── stores/                  # Pinia stores (workflow, package, folder, main, user,
    │                            # teamWorkflow, hostedWorkflow, sharedWorkflow)
    ├── db/                      # Dexie: storage.js (tables/variables/credentials), logs.js
    ├── service/                 # BrowserAPIService (offscreen↔background API proxy), renderer
    ├── lib/                     # pinia.js, vueI18n.js, tmpl.js, findSelector.js, dayjs, mitt…
    ├── utils/                   # shared.js (★ tasks registry), message.js, workflowTrigger.js,
    │                            # FindElement.js, workflowData.js, dataExporter.js,
    │                            # convertWorkflowData.js, credentialUtil.js, editor/*
    ├── composable/              # blockValidation, commandManager, shortcut, theme, editorBlock…
    ├── locales/<lang>/          # blocks.json, common.json, newtab.json, popup.json (10 langs)
    └── assets/                  # css (tailwind, flow), images
```

## 12.2 The load-bearing files (start here)

| File | Why |
|---|---|
| `src/utils/shared.js` | the `tasks` block registry + `categories` (~1800 lines) — the vocabulary of the whole app |
| `src/workflowEngine/WorkflowWorker.js` | `executeBlock` — dispatch, templating, handler contract, onError |
| `src/workflowEngine/WorkflowEngine.js` | `init` — graph compile, referenceData, lifecycle |
| `src/workflowEngine/templating/mustacheReplacer.js` | the entire `{{ }}` syntax implementation |
| `src/background/index.js` | the message table = the real API surface of the extension |
| `src/newtab/pages/workflows/[id].vue` | the editor page — drag&drop, save, debug wiring |
| `src/utils/message.js` | cross-context routing (`prefix--name`) |

## 12.3 Glossary

| Term | Meaning |
|---|---|
| **Block** | a node type; registry key = node `label` (e.g. `loop-data`) |
| **Handler** | the function that executes a block (`handlerX.js`), bound to a `WorkflowWorker` |
| **refDataKeys** | per-block list of data fields that get `{{ }}` templating |
| **referenceData** | the keyword namespace (`variables`, `table`, `loopData`, …) |
| **Worker** | one execution path through the graph; branches spawn additional workers |
| **Engine** | one workflow run (`WorkflowEngine`), owned by the `WorkflowManager` singleton |
| **State** | live run state mirrored to `storage.local 'workflowStates'` |
| **connectionsMap** | `sourceHandle → targets` compile of the drawn edges |
| **Handle ids** | `${id}-input-1`, `${id}-output-1`, `${id}-output-fallback`, `${id}-output-<conditionId>` |
| **drawflow** | legacy field name holding vue-flow data `{nodes, edges, x, y, zoom}` |
| **Package / blocks group** | reusable sub-flow stored in `savedBlocks` |
| **Connected table** | Dexie-backed table linked to a workflow (vs workflow-embedded columns) |
| **`$$var`** | Dexie-persisted global variable |
| **`refKey`** | Google Sheets block reference key for `{{googleSheets@refKey}}` |
| **`<name>.automa.json`** | workflow export format |
| **offscreen document** | MV3 hidden page hosting the engine (survives SW idle-kill) |
| **`'halo'`** | fake message that warms up the offscreen doc — intentional |
