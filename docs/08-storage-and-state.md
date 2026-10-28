# Chapter 8 — Storage, State & Security

> Where every kind of state lives: `chrome.storage.local`, two Dexie IndexedDB databases,
> `storage.session`, and the debounced running-state mirror.

---

## 8.1 The storage map

| What | Where | Notes |
|---|---|---|
| Workflows (docs) | `chrome.storage.local` → `workflows` | **array OR object map** (both shapes appear!) — via Pinia `workflowStore` |
| Team / hosted workflows | `teamWorkflows` / `workflowHosts` | keyed by teamId / hostId |
| Packages ("blocks groups") | `savedBlocks` | `{id,name,icon,inputs,outputs,variable,settings.asBlock,data}` |
| Dashboard settings | `settings` (+`tabs`) | locale, logsLimit, editor prefs, theme… |
| Folders | `folders` | |
| User/session | `user`, `session`, `sessionToken` | Google OAuth tokens |
| **Running states** | `workflowStates` | debounced (1s) mirror of `WorkflowState`; badge = running count |
| Last state (reuse) | `state:<id>` | `reuseLastState` restores table+variables |
| Editor drafts | `draft:<id>` / `draft-team:<id>` | removed on workflow delete |
| Triggers | `visitWebTriggers`, `onStartupTriggers`, `shortcuts` | registration lists |
| Schedule queue | `workflowQueue` | interval+fixedDelay runs parked while workflow is running |
| Misc | `pinnedWorkflows`, `pinnedBlocks`, `localBackupSettings`, `backupIds`, `lastBackup`, `theme`, `automaShortcut`, `isFirstTime`, `isRecording`, `recording`, `selectorSettings`, `checkStatus`, `automa-rename-downloaded-files` (session) | |
| Data tables | **Dexie DB `storage`** → `tablesData`, `tablesItems` | connected tables |
| Global variables | Dexie `variables` (`&name unique`) | exposed as `$$name` |
| Credentials | Dexie `credentials` (`&name unique`) | encrypted |
| Run logs | **Dexie DB `logs`** → `items`, `histories`, `ctxData`, `logsData` | written once at engine destroy |

**Nothing uses `storage.sync`.** Logs auto-delete after `deleteLogAfter` days (default 30); per-run
history capped by `logsLimit` (default 1001 entries, errors always kept).

---

## 8.2 The workflow document shape

`defaultWorkflow()` in `src/stores/workflow.js`:

```
id (nanoid), name, icon, folderId, content (description),
connectedTable, drawflow {edges, zoom, nodes[]} (vue-flow format),
table/dataColumns [{id,name,type}], description, trigger (denormalized),
createdAt/updatedAt, isDisabled, settings, version, globalData (JSON string)
```

Settings defaults: `publicId`, `blockDelay`, `saveLog`, `debugMode`, `restartTimes: 3`,
`notification`, `execContext: 'popup'`, `reuseLastState`, `inputAutocomplete`,
`onError: 'stop-workflow' | 'keep-running' | 'restart-workflow'`, `executedBlockOnWeb`,
`insertDefaultColumn`, `defaultColumnName: 'column'`, plus `events` (post-run webhook/JS actions
fired by `WorkflowEvent`).

---

## 8.3 Dexie databases (`src/db/`)

- **`storage`** (v2): `tablesData (++id, tableId)`, `tablesItems (++id, name, createdAt,
  modifiedAt)`, `variables (++id, &name unique)`, `credentials (++id, &name unique)`. Backs the
  dashboard Storage page and the engine's connected tables.
- **`logs`** (v1): `items (++id, name, endedAt, workflowId, status, collectionId)`, `ctxData`,
  `logsData`, `histories`. `WorkflowLogger.add` writes all four in parallel bulkAdds at engine
  destroy.
- Migration: `src/utils/dataMigration.js` one-time move of legacy storage.local logs into Dexie.

Engine integration: connected tables load in `WorkflowEngine.init()` and are re-synced at destroy;
`$$` variables load into `referenceData.variables` and `setVariable('$$name', …)` writes back;
credentials decrypt into `referenceData.secrets`.

---

## 8.4 Secrets & crypto (`crypto-js`)

- **`src/utils/credentialUtil.js`** — AES + HMAC-SHA256 (HMAC prefix, 64 hex chars) with the pass
  from `src/utils/getPassKey.js`. This checkout contains a **stub** returning `'local-dev-pass-key'`;
  the real module is gitignored — see `secrets.blank.js`. **You must create it for dev** (README:
  create `src/utils/getPassKey.js` returning any string).
- Credentials surface to workflows as `{{secrets@name}}` (decrypted at render time).
- **Workflow protection**: `WorkflowProtect.vue` + `decryptFlow.js` — HMAC check then AES decrypt;
  the pass itself is AES-wrapped with the app pass key.
- **Backup encryption**: `SettingsBackup.vue` AES-encrypts the `workflows` JSON with a user
  password + HMAC.

---

## 8.5 State lifecycle & crash behavior

- Live running states mirror into `storage.local['workflowStates']` (1s debounce); used for badge,
  dashboard live view, stop/resume/breakpoint commands, and cross-run reads (`log-data` block).
- `runtime.onStartup` **wipes** `workflowStates` — there is no crash-resume; the durable record is
  the Dexie log. Mitigations: `reuseLastState`, `workflowQueue` (fixed-delay schedule parking), and
  the offscreen document keeping the engine alive through service-worker deaths.
- The toolbar badge text = number of running workflows.

---

## 8.6 Backups

- **Local backups**: manual + cron-scheduled (`schedule-local-backup` alarm in
  `BackgroundEventsListeners.js`) — JSON-encodes non-protected workflows + optional Dexie
  tables/variables into a `data:` URL download; re-arms via cron-parser; optional encryption.
- **Cloud backup**: automa.site (`SettingsCloudBackup.vue`, `backupIds`, `lastBackup`).
- **Workflow hosting**: `workflowHosts` keyed by hostId; the Host page polls `/workflows/hosted` and
  re-registers triggers on updates.
