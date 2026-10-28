# Chapter 7 — Templating & Expressions: The Complete `{{ }}` Reference

> Every block input field listed in that block's `refDataKeys` is rendered through the templating
> engine before execution. This is the syntax surface — master it and you can wire data through
> any workflow.

**Implementation files**: `src/workflowEngine/templating/` (`index.js`, `renderString.js`,
`mustacheReplacer.js`, `templatingFunctions.js`), `src/lib/tmpl.js` (riot-tmpl wrapper), and
`src/sandbox/utils/handleBlockExpression.js` (JS-expression evaluation).

---

## 7.1 The pipeline

1. `templating({block, refKeys, data, isPopup})` clones the block and renders **every string in the
   block's `refDataKeys` list only** (declared per block in `src/utils/shared.js`). Fields not in
   `refDataKeys` are NOT templated. Substitutions are recorded in `block.replacedValue` (shown in
   logs/debug view).
2. `renderString.js` per string:
   - No `{{ }}` → returned untouched.
   - String **starts with `!!`** → JS-expression mode (see 7.4).
   - Otherwise → `mustacheReplacer.js`.

**Invalid expressions never throw**: an unknown dataKey/path (or unknown `$function`) leaves the raw
`{{...}}` text in place, because `objectPath.get` returns `undefined` and the original match is
restored. (Wrong function arity can still throw *inside* a handler and surface as a workflow error.)

---

## 7.2 Data paths — `{{ keyword@path }}`

The tag content goes through `keyParser`, which splits on the first `@` **or** `.` —
`variables@user.name` and `variables.user.name` are equivalent.

### Keywords (the reference-data object)

| Keyword | What it holds |
|---|---|
| `variables` | workflow variables (set by Insert-data, JS block `automaSetVariable`, HTTP request `assignVariable`, etc.) |
| `table` (aliases `dataColumn`, `dataColumns`) | the workflow's data table |
| `loopData` | current loop iterations `{[loopId]: {data, $index}}` |
| `googleSheets` | results keyed by the Google Sheets block's `refKey` |
| `globalData` | the workflow's global data (JSON-parsed) |
| `workflow` | results of sub-workflows: `workflow@<executeId>.table | .variables` |
| `secrets` | Dexie credentials — decrypted at render time via `credentialUtil.decrypt` |
| `prevBlockData` | the previous block's output |
| `activeTabUrl` | current active tab URL |

### Special rules

- **`table`**: `{{table@ColumnA}}` → first row (`0.ColumnA`); `{{table@$last.ColumnA}}` → last row;
  a non-numeric first segment defaults to row 0.
- **`loopData`**: a `data` segment is auto-inserted — `{{loopData@myLoop.id}}` resolves to
  `loopData.myLoop.data.id`. `$index` escapes this: `{{loopData@myLoop.$index}}`.
- **Dynamic bracket indexing**: `[...]` segments in the path are themselves rendered first, so
  `variables@items[{{loopData@l.$index}}]` works.
- **Stringify prefix `!`**: `{{!variables@obj}}` forces `JSON.stringify` of the result even for
  strings. Non-string results are JSON-stringified by default; strings pass through raw.
- **`checkExistence`** mode: the condition builder's "Data exists" value type wraps your dataPath in
  `{{ }}` and returns `objectPath.has(...)` as a boolean.

---

## 7.3 Built-in functions — `{{ $funcName(arg1, arg2) }}`

Syntax regex: `/^\$\s*(\w+)\s*\((.*)\)/`. Params are comma-split (quote-aware), quotes stripped,
each param re-rendered with `[...]` sub-replacement, then `parseJSON`-coerced. From
`templatingFunctions.js`:

| Function | Purpose |
|---|---|
| `$date([date,] format)` | dayjs tokens; special formats `relative` and `timestamp` |
| `$randint(min, max)` | random integer |
| `$getLength(str)` | length |
| `$slice(value, start, end)` | slice |
| `$multiply / $increment / $divide / $subtract / $modulo` | arithmetic |
| `$randData(str)` | password generator: placeholders `?l ?u ?d ?s ?f ?n ?m ?i ?a`, or pick from an array |
| `$filter(data, jsonPathExpr)` | jsonpath query |
| `$replace / $replaceAll` | string replace |
| `$toLowerCase / $toUpperCase` | casing |
| `$stringify` | JSON.stringify |

> Note: there are **no** `{{webhookUrl}}`/`{{userAgent}}` shortcodes in this codebase — `userAgent`
> is just a templated field of the New-tab block, and the "webhook" block is *outgoing* HTTP only.

---

## 7.4 JS-expression mode — `!!`

If the string starts with `!!`, everything inside the braces is evaluated as **JavaScript** (Chrome
only): `keyword@path` is converted to `keyword.path` for
`variables|table|secrets|loopData|workflow|googleSheets|globalData`, and the expression is evaluated
by **riot-tmpl** (brackets set to `{{ }}`) inside the **sandbox iframe** (`messageSandbox('blockExpression')`).
The result has `!!` stripped. Templating functions are exposed in the sandbox as `$date`,
`$randint`, etc. **On Firefox `!!` is stripped and the plain replacer is used** (no JS eval).

Examples:
- `!!variables.items.length > 5`
- `!!table[0].price * 1.2`

---

## 7.5 `automaRefData` — runtime access from JS blocks / pages

`src/workflowEngine/helper.js` generates the `automaRefData(keyword, path)` helper injected into
pages for JS-code/element blocks (with `$last` array support), plus:

| API | Purpose |
|---|---|
| `automaRefData(keyword, path)` | read reference data |
| `automaSetVariable(name, value)` | set a variable |
| `automaNextBlock(data, insert?, {nextBlockId, replaceTable}?)` | control flow / return data |
| `automaFetch('json'|'text'|'base64', resource)` | CORS-free fetch proxied through the background |
| `automaResetTimeout()` | reset the block timeout clock |
| `automaExecWorkflow({ id, … })` | launch another workflow |

---

## 7.6 Autocomplete wiring

- The registry marks which fields get autocomplete: `autocomplete: ['variableName','loopId','refKey']`.
- `src/utils/editor/editorAutocomplete.js` builds the editor-wide map: `variableName` → `variables`,
  `loopId` → `loopData`, `refKey` → `googleSheets`; special extractors add trigger `parameters` (as
  variables) and context-menu variables (`$ctxElSelector`, `$ctxTextSelection`, `$ctxLink`,
  `$ctxMediaUrl`) for the trigger block, variables created by `insert-data` blocks, and nested
  autocomplete from `blocks-group` children. Provided via `provide('autocompleteData', …)`.
- `EditAutocomplete.vue` wraps inputs with `trigger-char: ['{{','}}']` and
  `replace-after: ['@','.']` — the dropdown lists keys of the current path.
- JS-block editor completion: `src/utils/codeEditorAutocomplete.js` (CodeMirror 6) with the
  `automaFuncsSnippets` above.

---

## 7.7 Data writing model

- **Variables**: `worker.setVariable(name, value)` — `$push:name` appends to an array variable;
  `$$name` reads/writes the **Dexie-persisted global variables**.
- **Table**: `worker.addDataToColumn(column, value)` — an array of values inserts each key;
  `$assignColumns` as the column name bulk-inserts; respects `insertDefaultColumn` and column `type`
  conversion (`convertData`: `integer|boolean|array|string|any`).
- **Execution options** (`WorkflowEngine` constructor): `options.data.globalData` overrides
  workflow globalData; `options.data.variables` seeds variables; `options.tabId` pins the active
  tab; `options.blockId` starts at a specific block; `checkParams: false` skips parameter prompts.
- **Parameters**: trigger `parameters` and the Parameter-prompt block render a form in `params.html`;
  answers travel via `chrome.storage.local[promptId]` watched by the handler; `preferParamsInTab`
  injects the palette into the page instead.
