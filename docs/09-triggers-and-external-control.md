# Chapter 9 — Triggers & External Control (the wire formats)

> How workflows start, and every documented way to make Automa do something from the outside.

---

## 9.1 Trigger types & their registration

A trigger block stores `data.triggers: [{id, type, data}]` (multi-trigger; legacy single-object
shape still supported). UI: `SharedWorkflowTriggers.vue` (`triggersData` map) with per-type editors
in `edit/Trigger/*.vue`. Registration lives in **`src/utils/workflowTrigger.js`**:

| Type | Mechanism | Notes |
|---|---|---|
| `manual` | dashboard/popup run button | |
| `interval` | `alarms.create` with `periodInMinutes` (+ optional `delayInMinutes`) | `fixedDelay` option parks overlapping runs in `workflowQueue` |
| `cron-job` | `cron-parser` next occurrence → **one-shot alarm, self re-armed** after each fire | |
| `date` | one-shot `alarms.create({when})` | stale (>60s past) triggers skipped |
| `specific-day` | next matching weekday/time via dayjs (wraps +7 days) | |
| `visit-web` | stored in `visitWebTriggers`; fired from `webNavigation.onCompleted` / `onHistoryStateUpdated` (SPA-aware via `supportSPA`) | URL or regex match |
| `on-startup` | `runtime.onStartup` → `reRegisterTriggers(true)` executes immediately | also wipes `workflowStates` |
| `context-menu` | `menus.create` under parent `automaContextMenu` ("Run Automa workflow") | |
| `keyboard-shortcut` | stored in `shortcuts` map; **content script binds them with Mousetrap** | only fires on pages where the content script runs; honors `activeInInput` |
| `element-change` | `src/content/elementObserver.js` (MutationObserver) | **implemented but disabled** (start call commented out) |

Alarm name format: `trigger:<workflowId>:<triggerId>`. `registerWorkflowTrigger` always runs
`cleanWorkflowTriggers` first (clears matching alarms, prunes shortcut/startup/visit lists, removes
menu items). The **dashboard registers triggers directly** (it has full API access); the background
`workflow:register` message exists mainly for the website/team path.

The context-menu path asks the content script for `context-element` and executes with
`options.data.variables = {$ctxElSelector, $ctxTextSelection, $ctxLink, $ctxMediaUrl}`.

---

## 9.2 Trigger execution chain (scheduled example)

```
browser.alarms.onAlarm
→ BackgroundEventsListeners.onAlarms
→ BackgroundWorkflowTriggers.scheduleWorkflow({name})
→ parse alarm name → load workflow (local / hosted / team)
→ (interval + fixedDelay + already running) → park in storage.local.workflowQueue
→ skip stale date triggers
→ executeWorkflow(workflow)
→ re-arm one-shot cron/specific-day alarm
```

`executeWorkflow` → `'workflow:execute'` message → offscreen `WorkflowManager.execute` (Chrome) or
in-background manager (Firefox).

---

## 9.3 External control — all the wire formats

> There is **no `externally_connectable` / `onMessageExternal` / `onConnectExternal`** in this
> codebase. All external control goes through the DOM/content-script paths below.

### A) From any web page — CustomEvents (the most useful trick)

Dispatched in the page (picked up by `src/content/services/shortcutListener.js`):

```js
window.dispatchEvent(new CustomEvent('__automaExecuteWorkflow', {
  detail: {
    id: '<workflowId>',            // or use publicId instead
    publicId: '<settings.publicId>',
    data: { variables: { foo: 'bar' }, globalData: { /* … */ } },
  },
}));
// alias event name: 'automa:execute-workflow'
```

Lookup by `detail.id` **or** `detail.publicId`; `detail.data` becomes `workflow.options.data`.
Requires the Automa content script on the page (any http(s) page).

### B) Execute page deep link

Open `execute.html#/<workflowId>?var1=val1&var2=val2` — JSON-parsed query params become
`options.data.variables` (parameter prompts skipped) → workflow runs, tab closes.

### C) `.automa.json` install

Browsing to any URL ending in `.automa.json` prompts to install the workflow into
`storage.local.workflows`.

### D) The website bridge (`webService` content script)

Only injected on localhost / `*.automa.site` / `automa.vercel.app`. Page dispatches:

```js
window.dispatchEvent(new CustomEvent('__automa-ext__', {
  detail: { type: 'add-workflow', data: { /* … */ } },
}));
```

Types: `open-dashboard`, `open-workflow`, `add-workflow`, `add-team-workflow`,
`check-team-workflow`, `add-package`, `update-package`, `send-message`. Responses come back as
`__automa-ext__<type>` events.

### E) Internal message protocol (for extension contexts)

`src/utils/message.js`: `{ name: '[prefix--]name', data }` (JSON-stringified on Firefox). Key
background handlers: `workflow:execute` (payload = whole workflow object + `workflow.options`;
`workflow.includeTabId` injects the sender tab id), `workflow:stop`, `workflow:resume {id,
nextBlock}`, `workflow:breakpoint`, `workflow:register`, `fetch {type: 'text'|'json'|'base64',
resource}` (CORS-free fetch from pages via `__automa-fetch__`), `get:sender`, `open:dashboard`,
`debugger:send-command`.

### F) Command palette

Injected when `storage.local.automaShortcut` is set; opens a shadow-DOM palette on any page to run
workflows (with parameter forms) — hotkey communicated via `window._automaShortcuts`.

### G) In-page JS block API

Inside a JS block: `automaExecWorkflow({ id })` launches another workflow;
`automaNextBlock(data, insert?, {nextBlockId, replaceTable}?)` controls flow.

---

## 9.4 The HTTP Request (webhook) block

Outgoing only. `handlerWebhook.js` + `utils/webhookUtil.js`: methods with/without body, content
types `json|text|form|form-data`, response types `json|text|base64`, `dataPath` supports `$response`
meta (`$response.status`, `$response.data`, …), `fallback` output on HTTP error / network failure,
`ctxData.request` attached to logs, AbortController timeout.
