# AGENTS.md — Operating Manual for AI Coding Agents

> You are working on **Automa**, a Chrome/Firefox extension for browser automation via connected
> blocks (Vue 3 + webpack 5, MV3 Chrome / MV2 Firefox). This file contains everything an AI agent
> needs to modify this codebase **correctly**: the auto-registration patterns, the block handler
> contract, exact recipes for common changes, the quirks that look like bugs but are intentional,
> and a verification checklist. Every claim here was verified against the source (v1.30.00).
>
> Deeper companion docs (per-layer chapters with full detail) live at `../automa-study-guide/docs/`.

---

## 1. Golden rules — read before editing anything

1. **Check §4 (auto-registration) before adding any import or "wiring".** Handlers, edit panels,
   node components, and block registries are wired by filename via `require.context`. Editing the
   registration files to "register" your new file is almost always wrong.
2. **Do not "fix" anything in §8 (intentional quirks).** Several things look like bugs (`'halo'`,
   prefix-mismatch returning `undefined`, `workflows` being array-or-map) and are load-bearing.
3. **Templating (`{{ }}`) only applies to fields listed in a block's `refDataKeys`.** If you add a
   user-facing input field that should support expressions, you MUST add it to `refDataKeys` in the
   registry entry — otherwise `{{ variables.x }}` will reach the page literally.
4. **Keep all three manifests in sync** when changing permissions/entry points:
   `src/manifest.chrome.json`, `src/manifest.chrome.dev.json`, `src/manifest.firefox.json`.
   (`name`/`version`/`description` are injected by the webpack manifest plugin from package.json.)
5. **`en` locale is the source of truth.** Other locales (`src/locales/{es,fr,it,pt-BR,tr,uk,vi,zh,zh-TW}`)
   are optional; missing keys fall back via `te()` checks. Never let a missing en key break.
6. **Use Pinia for state.** `vuex` and `@vuex-orm` are declared in package.json but unused — do not
   add vuex code or remove those deps casually (risk vs. reward is bad).
7. **Never commit secrets.** `src/utils/getPassKey.js` and `secrets*.js` are gitignored by design.
8. **`business/` is an intentionally empty OEM seam** (webpack alias `@business`). Don't build
   features there; it exists so paid builds can inject blocks/handlers without forking core. New
   code goes in `src/`.
9. **Block ids are kebab-case (`loop-data`); code identifiers are their camelCase (`loopData`).**
   The conversion is `toCamelCase` and it is the universal lookup key — a mismatch silently
   disables your handler.
10. **Respect the storage layer split**: workflows/packages/settings → `chrome.storage.local`;
    logs/tables/variables/credentials → Dexie IndexedDB (`src/db/`). Nothing uses `storage.sync`.

---

## 2. Setup & commands

### Prerequisites

- Node ≥ 14.18.1 (Volta pins 20.11.1), **pnpm**.
- **Pass-key module (required to build/run)**: `src/utils/getPassKey.js` is gitignored. If missing,
  create it:
  ```js
  export default function getPassKey(fallback = '') {
    return 'anything-you-want'; // any non-empty string works for local dev
  }
  ```
- **Sibling repo dependency**: `@automa-platform/dashboard-components` is a `link:` dep pointing at
  `../automa-platform/packages/dashboard-components`. Either clone
  `https://github.com/AutomaApp/automa-platform` next to this repo, or remove that dependency from
  `package.json` (it is used only by `src/newtab/pages/JsonRenderDemo.vue`).

### Commands

| Command | Purpose |
|---|---|
| `pnpm dev` | webpack dev server on **port 3001**, writes bundles to `build/` (Chrome) |
| `pnpm dev:firefox` | same for Firefox (MV2) |
| `pnpm build` / `pnpm build:firefox` | production build → `build/` |
| `pnpm build:prod-chrome` / `build:prod-firefox` | clean cache + build + zip → `build-zip/<version>/` |
| `pnpm lint` | eslint (`.js`, `.vue`) |
| `pnpm prettier` | prettier write |

### Running & debugging

- Load **`build/`** as an unpacked extension (`chrome://extensions` → Developer mode → Load unpacked).
- **Hot reload only applies to UI pages** (popup, newtab, params, sandbox, execute, offscreen).
  `chromeExtensionBoilerplate.notHotReload = [background, webService, contentScript, recordWorkflow,
  elementSelector]` — after touching those, click the extension's **reload** button.
- To debug the workflow engine on Chrome: `chrome://extensions` → Automa → *Inspect views:
  **offscreen.html***. The service worker is mostly a router (see §7).

---

## 3. Architecture in 60 seconds

| Context | File(s) | Role |
|---|---|---|
| Service worker | `src/background/index.js` (+ `Background*.js`) | Thin router: browser event listeners, message table, JS-block injection, downloads. **Not** the engine. |
| Offscreen document (Chrome) | `src/offscreen/` | **Hosts the workflow engine** (`WorkflowManager`). Created on demand; survives SW idle-kill. Has no direct `chrome.tabs` — everything is proxied via `BrowserAPIService` (`src/service/browser-api/`) |
| Background page (Firefox) | dynamic `import()` of `WorkflowManager` | Engine runs here directly (persistent MV2 background) |
| Content scripts | `src/content/index.js` (+ `blocksHandler/`, `elementSelector/`, `services/`) | Executes "interaction" blocks in pages; iframe protocol; recorder; element picker; website bridge |
| Sandbox iframe | `src/sandbox/` | Manifest-sandboxed page that can `eval`: `!!` JS expressions, background-context JS blocks, condition code |
| Dashboard | `src/newtab/` | Vue SPA: workflow editor (vue-flow), logs, storage, settings. Persists via Pinia → `chrome.storage.local` |
| Popup / params / execute | `src/popup/`, `src/params/`, `src/execute/` | Toolbar popup; pre-run parameter forms; `execute.html#/<id>?var=x` deep-link launcher |

**Engine class chain**: `WorkflowManager` (singleton; owns state+logger) → `WorkflowEngine`
(one per run; compiles nodes/edges into `connectionsMap`, builds `referenceData`) → `WorkflowWorker`
(one per execution path; dispatches blocks, owns loops/tabs). Live state mirrors (1s debounce) to
`storage.local['workflowStates']`; toolbar badge = running count.

**Message protocol**: all cross-context messages go through `src/utils/message.js` —
`{ name: '[prefix--]name', data }`, handlers via `message.on(name, fn)` inside a
`MessageListener('background'|'offscreen'|…)`. A listener only answers messages matching its own
prefix. Firefox payloads are JSON-stringified. Raw `{type: …}` messages also exist for
content-script/dashboard traffic.

**Branching**: the first outgoing connection continues on the same worker; every additional
connection spawns a **new worker** with cloned state (`executeNextBlocks` → `engine.addWorker`).

---

## 4. Auto-registration map — what needs NO manual wiring

| You create… | It becomes… | Mechanism |
|---|---|---|
| `src/workflowEngine/blocksHandler/handlerMyBlock.js` | `engine.blocksHandler.myBlock` | `require.context` + strip `handler` + `toCamelCase` |
| `src/content/blocksHandler/handlerMyBlock.js` | in-page handler for interaction blocks | same pattern |
| `src/components/block/BlockMyThing.vue` | vue-flow node type `node-BlockMyThing` | `require.context('@/components/block')` in `WorkflowEditor.vue` |
| `src/components/newtab/workflow/edit/EditMyBlock.vue` | resolvable `editComponent: 'EditMyBlock'` | `require.context('./edit')` in `WorkflowEditBlock.vue` |
| entry in `tasks` (`src/utils/shared.js`) | palette item, search, minimap color, engine `blocksDetail` | `getBlocks()` consumers |
| `src/locales/<lang>/*.json` keys | translations | lazy i18n loader |

**Therefore: never edit `WorkflowEditor.vue`, `WorkflowEditBlock.vue`, `WorkflowBlockList.vue`,
`blocksHandler.js` (either), `WorkflowEngine.js`, or `WorkflowWorker.js` just to register something.**
They are generic over the registry.

---

## 5. The block contract (memorize this)

### Registry entry (`src/utils/shared.js` → `export const tasks`)

```js
'my-block': {
  name: 'My block', description: '…',
  icon: 'riMouseLine',              // v-remixicon name, 'path:<svgPathData>', or https URL
  component: 'BlockBasic',          // canvas node component (BlockBasic fits most blocks)
  editComponent: 'EditMyBlock',     // param panel
  category: 'general',              // interaction | browser | general | onlineServices | data | conditions | package
  inputs: 1, outputs: 1, allowedInputs: true, maxConnection: 1,
  refDataKeys: ['myField'],         // ← fields that get {{ }} templating (see Golden rule 3)
  autocomplete: ['variableName'],   // optional: feeds editor autocomplete
  data: { disableBlock: false, description: '', /* field defaults */ },
}
```

### Handler contract

- Signature: `handler(block, ctx)` where `block = {id, label, data, settings}` (data already
  template-expanded) and `ctx = {refData, prevBlock, prevBlockData, targetHandle, sourceHandle, …}`.
- **`this` is the `WorkflowWorker`**. Available members:
  - `this.getBlockConnections(blockId, outputIndex=1)` → array of `{id}` for the next blocks
  - `this.setVariable(name, value)` (`$push:name` appends; `$$name` persists to Dexie)
  - `this.addDataToColumn(column, value)` (`$assignColumns` name → bulk map)
  - `this.activeTab {id, url, frameId, frames}`, `this.loopList`, `this.loopEls`,
    `this.repeatedTasks`, `this.engine` (`referenceData`, `addRefDataSnapshot`, `isPopup`),
    `this._sendMessageToTab(payload, options)`
- **Return**: `{ data, nextBlockId, status?, logId?, ctxData?, replacedValue?, destroyWorker? }`.
  `nextBlockId` = the array from `getBlockConnections`. Returning without it ends the worker.
- **Errors**: `throw new Error(...)` to trigger the per-block onError pipeline
  (retry → insert data → fallback/continue/error).
- Handle-id conventions: `${id}-input-1`, `${id}-output-1`, `${id}-output-fallback`,
  `${id}-output-${conditionId}`, `${id}-input-${inputId}` (packages).
- **No scheduler exists**: your handler picks its outgoing edge(s) via `nextBlockId`. Use
  `category: 'interaction'` for in-page blocks — the generic `handlerInteractionBlock.js` routes
  them to the content script for free; write the page-side handler with
  `handleSelector(block, { onSelected, onError, onSuccess })` for element targeting.

---

## 6. Recipes (exact file lists)

### 6.1 Add a new block — minimal footprint: **4 files + i18n**

1. `src/utils/shared.js` — registry entry (§5).
2. `src/workflowEngine/blocksHandler/handlerMyBlock.js` — background handler (or skip + add
   `src/content/blocksHandler/handlerMyBlock.js` if `category: 'interaction'`).
3. `src/components/newtab/workflow/edit/EditMyBlock.vue` — param panel; props `data` + `block-id`,
   emit `update:data` with `{ ...props.data, ...changed }`; labels via
   `t('workflow.blocks.my-block.<field>')`.
4. `src/locales/en/blocks.json` — under `workflow.blocks`: name/description/field labels.

Conditional extras:

| Need | File |
|---|---|
| Custom canvas node (dynamic handles) | `src/components/block/BlockMyThing.vue` (wrap `BlockBase.vue`) |
| New icon | `src/lib/vRemixicon.js` `icons` map — or `path:`/https icon |
| Inline node validation | `src/newtab/utils/blocksValidation.js` (keyed by block id) |
| Not groupable / no onError | `excludeGroupBlocks` / `excludeOnError` in `shared.js` |
| Autocomplete for its fields | `src/utils/editor/editorAutocomplete.js` |
| Error-code → message | `src/utils/getBlockMessage.js` + locale maps |

Reference implementation with the minimal footprint: `parameter-prompt` (registry + handler +
edit panel + locales, nothing else).

### 6.2 Add a field to an existing block

Registry `data` default + `refDataKeys` (if templatable) → edit panel input → `en/blocks.json`
label → optional `autocomplete` entry → optional `blocksValidation.js` rule.

### 6.3 Add a templating function `{{ $myFunc(...) }}`

`src/workflowEngine/templating/templatingFunctions.js` — one exported function; it is dispatched
automatically. For `!!` JS-expression mode also expose it in
`src/sandbox/utils/handleBlockExpression.js`, and for editor autocomplete in
`src/utils/editor/editorAutocomplete.js`.

### 6.4 Add a trigger type

`src/utils/workflowTrigger.js` (`workflowTriggersMap` + registration/cleanup) →
`src/background/BackgroundWorkflowTriggers.js` (listener) →
`src/components/newtab/workflow/edit/Trigger/TriggerX.vue` →
`SharedWorkflowTriggers.vue` `triggersData` → locale key `workflow.blocks.trigger.items.<type>`.
Alarm names must follow `trigger:<workflowId>:<triggerId>`.

### 6.5 Add a condition comparison operator

`src/workflowEngine/utils/testConditions.js` (compare map) + `src/utils/compareBlockValue.js` +
`conditionBuilder` schema in `src/utils/shared.js` (drives the UI).

### 6.6 Add a dashboard page

`src/newtab/router.js` (hash routes) → `src/newtab/pages/…` → optionally `AppSidebar.vue`. State
goes in a Pinia store (`src/stores/`) persisted via the `storageMap` in `src/lib/pinia.js`.

### 6.7 Modify permissions / manifest

Edit **all three** manifests (Golden rule 4). Optional permissions are requested at runtime —
follow `src/composable/hasPermissions.js`. Firefox notes: `menus` instead of `contextMenus`,
`browser_action` instead of `action`, MV2 persistent background, no `offscreen`/`debugger`.

### 6.8 Add a cross-context message

Define the wire name in `src/utils/message.js` style (`{name, data}`), register a handler with
`message.on(name, fn)` in the owning context's `MessageListener` (background: `src/background/index.js`;
offscreen: `src/offscreen/message-listener.js`), and call from the other side with the right
`prefix--name`. Remember: listeners only answer their own prefix.

### 6.9 Change the workflow JSON format

Update `defaultWorkflow()` in `src/stores/workflow.js` (defaults), export/import in
`src/utils/workflowData.js`, and **legacy compatibility** in `src/utils/convertWorkflowData.js`
(legacy DrawFlow strings must keep converting). The `drawflow` field name is legacy but is the
storage key — do not rename it without a migration.

---

## 7. Storage map (quick reference)

| Data | Where |
|---|---|
| Workflow documents | `storage.local['workflows']` — **may be an array OR an object map** (legacy) — handle both |
| Packages / blocks groups | `storage.local['savedBlocks']` |
| Team / hosted workflows | `storage.local['teamWorkflows']` / `['workflowHosts']` |
| Dashboard settings | `storage.local['settings']` |
| Running states (live) | `storage.local['workflowStates']` (1s debounced mirror) |
| reuseLastState | `storage.local['state:<id>']` |
| Triggers registries | `visitWebTriggers`, `onStartupTriggers`, `shortcuts`, `workflowQueue` |
| Data tables / global variables / credentials | Dexie DB `storage` (`src/db/storage.js`) |
| Run logs (4 tables) | Dexie DB `logs` (`src/db/logs.js`), written once at engine destroy |
| Recordings | `storage.local['isRecording']` / `['recording']` (not Dexie) |

Credentials are AES+HMAC encrypted via `src/utils/credentialUtil.js` + `getPassKey`; surfaced to
workflows as `{{secrets@name}}`.

---

## 8. Intentional quirks — DO NOT "fix"

1. **`'halo'` message** (`src/background/index.js`, first statement): fake message that forces the
   offscreen document to exist at every SW start. There is no handler — by design.
2. **`MessageListener` returns `undefined` on prefix mismatch**: deliberate multi-context response
   arbitration (same broadcast reaches SW + offscreen + dashboard without wrong-context replies).
3. **`storage.local['workflows']` array-or-map**: legacy installs stored an array. Read defensively.
4. **Firefox differences**: engine in background page (no offscreen), `!!` JS expressions disabled
   (no sandbox eval → plain replacer), `debugMode` disabled (no `debugger` permission).
5. **Cron/specific-day alarms are one-shot and self-re-armed** after each fire
   (`BackgroundWorkflowTriggers.js`). Cleanup goes through `cleanWorkflowTriggers` — deleting a
   workflow must call it or alarms leak.
6. **Interval + `fixedDelay` while already running** → parked in `storage.local['workflowQueue']`,
   drained by `WorkflowEngine.executeQueue` on destroy. Skip-overlap by design.
7. **`runtime.onStartup` wipes `workflowStates`**: no crash-resume. Interrupted runs are lost; the
   Dexie log is the durable record. Don't add "resume after restart" casually — it's a product
   decision.
8. **Keyboard-shortcut triggers are content-script Mousetrap bindings** (fire only where the
   content script runs), not `chrome.commands`. `chrome.commands` is reserved for
   `open-dashboard` / `element-picker`.
9. **No `externally_connectable` / `onMessageExternal`**: external control is via DOM events
   (`__automaExecuteWorkflow`), `execute.html` deep links, `.automa.json` URLs, and the
   `webService` bridge. Don't add external messaging without a security review.
10. **`drawflow` field name** holds vue-flow data `{nodes, edges, x, y, zoom}`.
11. **`css-selector-generator` is an unused dependency** — selector generation is `@medv/finder`
    (`src/lib/findSelector.js`). Don't "switch" it.
12. **`src/content/index.js` registers its message/contextmenu listeners twice** (duplicated block,
    ~lines 203–245). Harmless; deduplicate only if you verify the second registration isn't
    shadowed.
13. **`src/content/elementObserver.js` (element-change trigger) is implemented but disabled**
    (start call commented out). Enabling it is a product decision.
14. **The dashboard registers workflow triggers directly** (it has full API access); the background
    `workflow:register` message exists for the website/team path.
15. **Known bug, handled upstream of this file**: `engine.restartWorkersCount` is never initialized
    (`WorkflowEngine`), so `settings.onError: 'restart-workflow'` can throw a TypeError in
    `WorkflowWorker.js` and fall through to an engine error. If you touch that path, initialize the
    counter.
16. **vuex/@vuex-orm unused**; **`src/execute/` is a URL launcher, not the block-execution layer**.

---

## 9. Code style

- **Prettier**: single quotes, trailing comma `es5`, arrow parens always. **2-space indent**, LF,
  final newline (`.editorconfig`). **ESLint**: `airbnb-base` + `vue/vue3-recommended` +
  `prettier` — imports resolve through the webpack config (`import/resolver`), so `@/` aliases lint
  correctly.
- Extension API: `import browser from 'webextension-polyfill'` (a few service files alias it as
  `Browser`). Don't call `chrome.*` directly in new code.
- Vue SFCs: match the existing mix of options/composition API per file; UI primitives live in
  `src/components/ui/` (`Ui*.vue`); icons via `<v-remixicon>`.
- Comments: English for new code (the fork has Chinese comments in legacy JS-injection areas —
  leave them be unless rewriting that logic).
- i18n: every user-facing string via `t(...)`, keys under the right namespace
  (`workflow.blocks.<id>.*` in `blocks.json` is the big one).
- Tests: there is no test suite. Verification = §10.

---

## 10. Verification checklist (before you declare done)

1. `pnpm lint` — zero errors.
2. `pnpm dev` (and `pnpm dev:firefox` if you touched browser-specific code/manifests) — build
   completes; reload the unpacked extension.
3. If you added/changed a block: create a workflow using it → run once → open the run log and
   confirm `replacedValue` shows your fields templated correctly.
4. If you changed engine behavior: enable workflow **testing mode**, set node breakpoints, step
   block-by-block, and inspect ctxData.
5. If you touched storage shapes: test both a fresh install **and** legacy data (the
   array-or-map `workflows`, DrawFlow-format workflows) — `convertWorkflowData.js` must keep
   converting.
6. If you touched triggers: verify re-registration on workflow save **and** cleanup on workflow
   delete (no leaked alarms in `chrome://extensions` → service worker console).
7. Never ship with real values in `secrets*.js` / `getPassKey.js`.

---

## 11. Landmines by file

| File | Gotcha |
|---|---|
| `src/workflowEngine/WorkflowWorker.js` | `executeBlock` is the dispatch heart — handler lookup, templating, onError. Change with care; branch cloning happens in `executeNextBlocks`. |
| `src/workflowEngine/WorkflowEngine.js` | `referenceData` shape is the templating contract; `restartWorkersCount` bug (§8.15). |
| `src/utils/shared.js` | ~1800-line `tasks` registry + `categories` + `conditionBuilder` + exclusion lists. Single source of truth — syntax errors here break the whole editor. |
| `src/background/index.js` | ~1100 lines; the message table is the extension's real API surface. JS injection has a CSP/Trusted-Types fallback path — don't simplify it away. |
| `src/content/index.js` | Iframe protocol uses one-time storage tokens; `|>` frame-selector chaining; duplicated listener block (§8.12). |
| `src/utils/message.js` | Firefox JSON-stringifies payloads; keep handlers JSON-safe. |
| `src/utils/workflowTrigger.js` | Always run `cleanWorkflowTriggers` before re-registering. |
| `src/db/*` | Dexie schema changes need a version bump + upgrade path. |
| `package.json` | `xlsx` comes from a CDN tarball; `@automa-platform/*` is a sibling `link:` dep. |
