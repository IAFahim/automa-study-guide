# Chapter 2 — Architecture Overview: Every Layer, Every Bundle, Every Wire

> Read this once, then use it as a map while diving into the per-layer chapters.

---

## 2.1 The single most important architectural fact

**On Chrome (MV3), the service worker is only a thin router. The real workflow engine lives in an
offscreen document.**

- `src/background/index.js` fires `BackgroundOffscreen.instance.sendMessage('halo')` as the very
  first statement. There is **no `'halo'` handler anywhere** — it exists purely to force
  `BackgroundOffscreen.#ensureDocument()` to create `/offscreen.html` via
  `chrome.offscreen.createDocument` (reasons: BLOBS, CLIPBOARD, IFRAME_SCRIPTING) at every SW start.
- `src/background/BackgroundWorkflowUtils.js` is the singleton bridge:
  `executeWorkflow/stopExecution/resumeExecution/updateExecutionState` forward to the offscreen doc
  on Chrome; on Firefox they dynamically `import()` `WorkflowManager` into the persistent MV2
  background instead.
- Offscreen documents are not subject to the 30s MV3 SW idle kill — that's how long workflows
  survive. **There is no other keep-alive mechanism** (no ping intervals).

State that must outlive contexts is persisted:
- `storage.local` keys: `workflows` (array **or** object map!), `workflowHosts`, `teamWorkflows`,
  `workflowStates`, `visitWebTriggers`, `onStartupTriggers`, `shortcuts`, `workflowQueue`,
  `settings`, `logs` (legacy), `savedBlocks`, `isRecording`, `recording`, `pinnedWorkflows`,
  `selectorSettings`, `automaShortcut`, `state:${id}` (reuseLastState).
- Dexie IndexedDB (`src/db/`): `tablesData`, `tablesItems`, `variables`, `credentials`, and the
  `logs` DB for run logs.

`runtime.onStartup` wipes stale `workflowStates` (interrupted runs are gone; see chapter 3.7).

---

## 2.2 The manifests

| Aspect | `manifest.chrome.json` | `manifest.firefox.json` |
|---|---|---|
| MV | 3 (min Chrome 116) | **2** (gecko ≥91.1, persistent bg scripts) |
| background | `background.bundle.js`, `type: module` | `background.scripts`, `persistent: true` |
| toolbar | `action` | `browser_action` |
| commands | `open-dashboard` Alt+A, `element-picker` Alt+P | same |
| content scripts | `contentScript.bundle.js` on `<all_urls>`, `document_start`, all frames; `webService.bundle.js` on `localhost`, `*.automa.site`, `automa.vercel.app` | same but no localhost |
| permissions | tabs, proxy, alarms, storage, **debugger**, activeTab, **offscreen**, webNavigation, unlimitedStorage, **scripting** | tabs, proxy, **menus**, alarms, storage, webNavigation, unlimitedStorage, `<all_urls>` |
| optional | cookies, downloads, contextMenus, clipboardRead, notifications | clipboardRead/Write, downloads, notifications, cookies |
| sandbox | `{"pages": ["/sandbox.html"]}` | n/a |
| CSP | default MV3 | `script-src 'self' 'unsafe-inline' https:` (needed for inline JS blocks) |

Non-obvious: **no `externally_connectable` and no `onMessageExternal` anywhere** (verified by
grep). The website→extension bridge is the `webService` content script (chapter 9). webpack injects
package.json `name/version/description` into the manifest at build time; prerelease versions get
Chrome `version` + `version_name` and Firefox gets the hyphen stripped (store rules).

---

## 2.3 Build system (`webpack.config.js`, `utils/`)

**Entries** — each becomes `build/<name>.bundle.js`:
`sandbox`, `execute`, `newtab`, `popup`, `params`, `background`, `contentScript`, `offscreen`,
`recordWorkflow`, `webService`, `elementSelector`. HtmlWebpackPlugin emits `newtab.html`,
`sandbox.html`, `execute.html`, `popup.html`, `params.html`, `offscreen.html`.

**Aliases**: `@` → `src/`, `@business` → `business/dev`, `secrets` → `secrets.$NODE_ENV.js`
(fallback `secrets.blank.js`). `BROWSER_TYPE` is a DefinePlugin global; `IS_FIREFOX` derives from it.

**Hot reload** (`utils/webserver.js`): dev server on port 3001, `writeToDisk: true`. Only UI pages
(popup/newtab/params/sandbox/execute/offscreen) hot-reload — `chromeExtensionBoilerplate.notHotReload
= [background, webService, contentScript, recordWorkflow, elementSelector]` means **extension-context
bundles require a manual extension reload** during development.

**Scripts**: `pnpm dev` / `dev:firefox` / `build:prod-chrome` / `build:prod-firefox`;
`build:zip` archives to `build-zip/<version>/automa-<browser>-v<version>.zip`.

**Business seam**: `business/dev/index.js` is an empty function and all `business/dev/blocks/*`
stubs return `{}` — this is where a paid/business build plugs in (`automa('background', message)`,
`customHandlers()`, `customBlocks()`). Also note `@automa-platform/dashboard-components` is a
`link:` dependency to a sibling repo (`../automa-platform`).

---

## 2.4 Background service worker (`src/background/`)

| File | Role |
|---|---|
| `index.js` (~1100 lines) | SW entry: registers all browser-event listeners, owns the single `runtime.onMessage` listener, the message table, JS-block injection machinery, MV3 download rename subsystem |
| `BackgroundEventsListeners.js` | alarms, commands, action click, startup/installed, webNavigation, context menu, notification click |
| `BackgroundWorkflowTriggers.js` | executes visit-web/scheduled/context-menu triggers; `reRegisterTriggers` |
| `BackgroundWorkflowUtils.js` | facade to the engine host; `getWorkflow()` resolves across `workflows`/`workflowHosts`/`teamWorkflows` |
| `BackgroundOffscreen.js` | offscreen doc lifecycle singleton + `'offscreen'`-prefix sender |
| `BackgroundUtils.js` | `openDashboard()` (reuse tab or open ≤715px popup window) and `sendMessageToDashboard()` |

Init sequence: halo warm-up → `alarms.onAlarm` → `commands.onCommand` → `action.onClicked` →
`runtime.onStartup/onInstalled` → `webNavigation` → `contextMenus` → `notifications.onClicked` →
build `MessageListener('background')`.

Notable subsystems:
- **JS-block injection** (`script:execute`): injects a `<script class="automa-custom-js">` into the
  page **MAIN world** via `scripting.executeScript`; the page script talks back over CustomEvents
  (`__automa-next-block__`, `__automa-reset-timeout__`, `__automa-fetch__`). CSP-blocked pages fall
  back to a two-stage `chrome.debugger Runtime.evaluate` flow (`check-csp-and-inject`).
- **Downloads rename subsystem**: `downloads:watch-created/watch-changed` queue requests in memory
  **and** mirror them to `storage.session` (`automa-rename-downloaded-files`) so rename still works
  after SW restarts; `onDeterminingFilename` suggests filenames (Chrome-only).
- **Callback bridge** (`src/utils/callbackBridge.js`): functions serialized cross-context as
  `{__type:'callback_bridge', __callbackId}` markers, resolved via `'callback-bridge'`-prefixed
  messages with a 5-minute timeout.

---

## 2.5 Content scripts (`src/content/`)

### `contentScript.bundle.js` — injected everywhere, all frames, `document_start`
- Singleton guard `window.isAutomaInjected`.
- **Iframe protocol** (window.postMessage, not runtime messaging): `automa:get-frame` (main frame
  answers with iframe rect), `automa:execute-block` → child executes → `automa:block-execute-result`.
  The child validates a **one-time token** stored in `storage.local` keyed `message:<nanoid>` —
  prevents hostile pages from forging execution requests.
- **`|>` frame-selector chaining**: `frameSel|>innerSel` — resolves the iframe via
  `contentDocument` (same-origin) or postMessage (cross-origin).
- `blocksHandler()`: require.context registry of the 21 in-page block handlers; dispatch by
  `toCamelCase(data.name || data.label)`.
- `runtime.onMessage` handles: `isBlock:true` payloads (block execution), `input-workflow-params`,
  `content-script-exists` (injection probe), `automa-element-selector`, `context-element`.
- `__automa-fetch__` window event → background fetch — the CORS-free fetch backing `automaFetch`
  in JS blocks.
- Visiting a `*.automa.json` URL prompts to install the workflow.
- `showExecutedBlock.js`: overlay spinner + block name during execution.
- `commandPalette/`: shadow-DOM `<automa-palette>` opened by `storage.local.automaShortcut`.

### `elementSelector.bundle.js`
Main frame mounts a Vue app into a shadow root; sub-frames get the `[automa-el-list]` outline style.
Accepts a **long-lived `browser.runtime.onConnect` port** from the dashboard; the picked selector is
delivered with `port.postMessage` on save.

### `recordWorkflow.bundle.js`
Turns clicks/changes/file-picks into workflow blocks, posting sub-frame events up as
`automa:record-events`; mounts the recorder toolbar; stops on `{type:'recording:stop'}`.

### `webService.bundle.js` — the website bridge
Runs only on automa.site/vercel/localhost. Listens for `__automa-ext__` window CustomEvents from the
site: `open-dashboard`, `open-workflow`, `add-workflow`, `add-team-workflow`, `check-team-workflow`,
`add-package`/`update-package`, `send-message`. Sets `data-atm-ext-installed` = version.

---

## 2.6 Popup, params, offscreen, sandbox, service

- **Popup** (`src/popup/`): if `isRecording` is set it immediately opens the dashboard at
  `#/recording` and closes itself; otherwise executes workflows, pins workflows, launches the picker.
- **Dashboard** (`src/newtab/`): single `runtime.onMessage` with a `messageEvents` map
  (`refresh-packages`, `open-logs`, `workflow:added`, `recording:stop`); mirrors
  `workflowStates` storage changes into the UI; embeds its own hidden `#sandbox` iframe.
- **params.html** (`src/params/`): pre-run parameter input UI (trigger `parameters` and the
  Parameter Prompt block).
- **offscreen**: `index.html` contains the hidden `<iframe src="/sandbox.html" id="sandbox">`
  required by `messageSandbox` (window.postMessage `{id, type}` → reply `{id, type:'sandbox',
  result}`).
- **sandbox**: manifest-declared sandboxed page (can eval). Handlers: `fetchResponse`,
  `conditionCode` (JS conditions), `blockExpression` (riot-tmpl `!!` expressions), `javascriptBlock`
  (runs user JS with preload scripts, returns `{columns, variables}`).
- **`src/service/`** — browser-API abstraction: `BrowserAPIService` detects context
  (`'tabs' in Browser`); privileged contexts bind real APIs; content scripts get message-proxied
  calls (args serialized via `src/utils/serialization.js`, so even callback-style args work
  cross-context). `BrowserAPIEventHandler` lets the offscreen doc subscribe to browser events it
  cannot access. `browser-api-map.js` is the API whitelist.

---

## 2.7 Messaging core (`src/utils/message.js`)

Wire format: `{name: '[prefix--]name', data}`; Firefox payloads are JSON-stringified/parsed.

**Prefix routing**: a `MessageListener` only *answers* messages whose first `--` segment matches its
own prefix; foreign-prefix messages return `undefined`. This is what lets the same broadcast reach
SW + offscreen + dashboard without the wrong context grabbing the response channel. Handlers are
registered as `message.on(name, fn)` → map key `prefix--name`, invoked with `(data, sender)`.

### Major runtime message types

**Background-handled**: `browser-api` (proxy API call), `browser-api:toggle-browser-event-listener`,
`fetch`/`fetch:text` (CORS-free), `open:dashboard`, `set:active-tab`,
`debugger:send-command`/`debugger:type`, `get:sender`/`get:file`/`get:tab-screenshot`,
`dashboard:refresh-packages`, `workflow:execute` (from popup/dashboard/execute.html/content/
website), `workflow:stop/resume/breakpoint`, `workflow:added`, `workflow:register`, `recording:stop`,
`get:user-id`, `check-csp-and-inject`, `script:execute(-callback)`,
`downloads:register-listeners/watch-created/watch-changed`, `resolve-callback`.

**Offscreen-handled** (`offscreen--` prefix): `workflow:execute/stop/resume/update`,
`browser-api:on-browser-event`.

**Raw `{type}` messages to content**: `isBlock:true` block payloads, `input-workflow-params`,
`content-script-exists`, `automa-element-selector`, `context-element`, `recording:stop`.
**To dashboard**: `refresh-packages`, `open-logs`, `workflow:added`, `recording:stop`.

---

## 2.8 All the ways a workflow can be triggered

1. **Popup** run button → `RendererWorkflowService.executeWorkflow` → background → offscreen.
2. **Dashboard** run/save (dashboard calls `registerWorkflowTrigger` directly — it has full API
   access, no background round-trip needed).
3. **Alarms** (interval / cron-job / specific-day / date) → `scheduleWorkflow`.
4. **Visit-web** (incl. SPA history changes) → `visitWebTriggers` storage list matching.
5. **Context menu** → `context-element` message captures `$ctxLink/$ctxMediaUrl/$ctxElSelector/
   $ctxTextSelection` as variables.
6. **Keyboard shortcuts** — two mechanisms: chrome.commands (`open-dashboard`, `element-picker`
   only) and user-defined shortcuts in `storage.local.shortcuts`, bound per-page by Mousetrap in the
   content script (only fire where the content script runs!).
7. **On-startup** (`reRegisterTriggers(true)`).
8. **execute.html URL launcher**: `execute.html#/<workflowId>?var=value` — query params become
   workflow variables.
9. **From the page**: `__automaExecuteWorkflow` / `automa:execute-workflow` CustomEvents (by
   workflow `id` or `settings.publicId`).
10. **From automa.site** via the `webService` bridge.
11. **Command palette** in-page (`automaShortcut`).
12. Element-observer trigger exists (`src/content/elementObserver.js`, MutationObserver) but is
    **commented out**.

---

## 2.9 Traps & surprises checklist

1. `'halo'` is a fake warm-up message — don't "fix" it.
2. Firefox never uses offscreen; `debugMode` is force-disabled there (no `debugger` permission).
3. User keyboard shortcuts are content-script Mousetrap bindings, not `chrome.commands`.
4. Cron/specific-day alarms are one-shot and **self-re-arming** after each fire.
5. Interval+`fixedDelay` runs that fire while the workflow is running get parked in
   `storage.local.workflowQueue` and drained by `WorkflowEngine.executeQueue` on destroy.
6. `getWorkflow()` mutates hosted workflow objects (`id = hostId`).
7. `src/content/index.js` double-registers its message/contextmenu listeners (harmless but
   confusing when tracing).
8. The dashboard registers its own triggers directly — `workflow:register` in the background
   exists mainly for the website/team path.
9. Badge text on the toolbar icon = number of running workflows.
10. `MessageListener` returning `undefined` on prefix mismatch is deliberate multi-context
    response arbitration.
