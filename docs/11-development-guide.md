# Chapter 11 — Development Guide: Build, Run, Contribute

## 11.1 Setup

```bash
git clone https://github.com/AutomaApp/automa.git
cd automa
pnpm install

# Required before dev/build: create the pass-key module
# (src/utils/getPassKey.js is gitignored; see secrets.blank.js)
cat > src/utils/getPassKey.js <<'EOF'
export default function () {
  return 'anything-you-want';
}
EOF
```

> Note: this fork additionally links `@automa-platform/dashboard-components` to
> `../automa-platform/packages/dashboard-components` — either clone that sibling repo or remove the
> link dependency if you only need core work.

## 11.2 Commands

| Command | What it does |
|---|---|
| `pnpm dev` | webpack dev server on port 3001, writes to `build/` (Chrome) |
| `pnpm dev:firefox` | same, Firefox MV2 |
| `pnpm build` | production build → `build/` |
| `pnpm build:firefox` | Firefox build |
| `pnpm build:prod-chrome` / `build:prod-firefox` | clean-cache + build + zip (`build-zip/<version>/automa-<browser>-v<version>.zip`) |
| `pnpm lint` / `pnpm prettier` | eslint / prettier |

**Load the extension**: Chrome → `chrome://extensions` → enable Developer mode → "Load unpacked" →
select the `build/` directory. After changes to non-hot-reload bundles (background, contentScript,
webService, recordWorkflow, elementSelector), click the extension's reload button.

## 11.3 Stack inventory

- **Vue 3** (options/composition mix) + **Pinia** (with a `saveToStorage` plugin) — Vuex and
  @vuex-orm are declared but unused.
- **vue-flow** for the editor canvas (the `drawflow` field name is legacy; the shape is
  `{nodes, edges, x, y, zoom}`).
- **Tailwind CSS** + custom CSS (`src/assets/css/`), `v-remixicon` icons, tippy tooltips,
  vue-toastification, Mousetrap shortcuts, vue-i18n (10 locales).
- **CodeMirror 6** for JS/JSON editing; **tiptap** (ProseMirror) for rich text; **dexie** for
  IndexedDB; **crypto-js** for credential/backup crypto; **papaparse** (CSV), **xlsx** (SheetJS,
  CDN tarball), **jsonpath**, **cron-parser/cronstrue**, **dagre** (auto-layout), **@medv/finder**
  (CSS selectors), **riot-tmpl** (`!!` expressions), **mitt** (event bus), **nanoid**,
  **webextension-polyfill**.
- **webpack 5** with `require.context` auto-registration as the dominant pattern (blocks, handlers,
  edit panels, node components, locales).

## 11.4 Codebase conventions worth following

- **Auto-registration by file name**: `handlerMyBlock.js` → `myBlock`; `EditMyBlock.vue` →
  `editComponent: 'EditMyBlock'`; `BlockMyThing.vue` → node type `node-BlockMyThing`. No import
  lists anywhere.
- **`toCamelCase` is the universal key transform** between block labels (`loop-data`) and code
  identifiers (`loopData`).
- **Registry first**: `src/utils/shared.js` `tasks` is the single source of truth for block
  metadata (`refDataKeys`, `autocomplete`, default `data`, category, IO counts).
- **Everything cross-context goes through `message.on/emit`** with `prefix--name` routing — never
  call `chrome.runtime.sendMessage` ad hoc in new code.
- **Templating is opt-in per field** via `refDataKeys` — remember to add new user-facing fields.
- Optional permissions (`cookies, downloads, contextMenus, clipboardRead, notifications`) are
  requested at runtime via `src/composable/hasPermissions.js` — new privileged features should
  follow that pattern.
- `BROWSER_TYPE` / `IS_FIREFOX` guard every Firefox/Chrome difference (`src/common/utils/constant.js`).

## 11.5 Debugging cheat sheet

| To debug… | Do this |
|---|---|
| Workflow engine | Inspect `offscreen.html` from `chrome://extensions` → Automa → "Inspect views" |
| Service worker | "Inspect views: service worker" (it's mostly a router + message table) |
| Content script | DevTools on any page (look for the `window.isAutomaInjected` guard) |
| Dashboard | DevTools on the dashboard tab |
| Sandbox | DevTools on `sandbox.html` view (it evaluates `!!` expressions & background JS) |
| Block behavior | Testing mode + breakpoints (chapter 6.2), `replacedValue` in logs |
| Storage | DevTools → Application → IndexedDB (`logs`, `storage` DBs) & Extension storage |
| Messaging | Log in `src/utils/message.js` `MessageListener` — every cross-context message funnels there |

## 11.6 Release flow (as configured)

`build:prod-chrome`/`build:prod-firefox` → clean build cache → build → zip per browser. Prerelease
versions (`x.y.z-rc1`) get Chrome numeric `version` + `version_name` and Firefox gets the hyphen
stripped (store rules; handled in the webpack manifest plugin). GitHub workflow lives in
`.github/`.
