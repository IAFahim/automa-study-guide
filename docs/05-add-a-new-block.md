# Chapter 5 — Recipe: Adding a Brand-New Block End-to-End

> Verified against the newest real examples in the repo: **`parameter-prompt`** and
> **`workflow-state`**, which touch exactly the minimal file set. A plain background block needs
> **4 files + locales**; nothing else requires manual registration because all consumers use
> `require.context` / `getBlocks()`.

---

## 5.1 The minimal recipe (plain background block)

### Step 1 — Registry entry (`src/utils/shared.js`)

Add to `export const tasks` (e.g. after `parameter-prompt`, ~line 1503):

```js
'my-block': {
  name: 'My block', description: '…',
  icon: 'riSomeIcon',                 // must exist in src/lib/vRemixicon.js `icons`
  component: 'BlockBasic',            // existing canvas node component
  editComponent: 'EditMyBlock',       // param panel (step 3)
  category: 'general',                // key of `categories`
  inputs: 1, outputs: 1, allowedInputs: true, maxConnection: 1,
  refDataKeys: ['myField'],           // fields that get {{ }} templating
  autocomplete: ['variableName'],     // optional
  data: { disableBlock: false, description: '', /* all defaults */ },
}
```

This **alone** puts the block in the palette, block search, minimap coloring, validation-map keys,
and the engine's `blocksDetail`.

### Step 2 — Engine handler (`src/workflowEngine/blocksHandler/handlerMyBlock.js`)

```js
export default async function ({ data, id }, { refData }) {
  // … your logic; `this` is the WorkflowWorker …
  return { data, nextBlockId: this.getBlockConnections(id) };
}
```

The filename **auto-registers** the handler: `blocksHandler.js` require.contexts the folder,
strips the `handler` prefix, and camelCases — `toCamelCase('MyBlock') === 'myBlock'` must equal the
camelCased registry key (`google-sheets-drive` → `googleSheetsDrive` shows the pattern).

**Alternative — in-page interaction block**: set `category: 'interaction'` in the registry (no
background handler; the generic `handlerInteractionBlock.js` routes it) and create
`src/content/blocksHandler/handlerMyBlock.js`:

```js
export default function (block, { handleSelector }) {
  // resolve elements via handleSelector(block, { onSelected, onError, onSuccess })
}
```

**Alternative — business-layer only**: `business/dev/blocks/backgroundHandler/index.js` returning
`{ myBlock: fn }` / `contentHandler/index.js` (custom content handlers merge first).

### Step 3 — Param panel (`src/components/newtab/workflow/edit/EditMyBlock.vue`)

Props: `data` (+ `block-id`); emit `update:data` merging `{...props.data, ...changed}`. Wrap
selector fields in `EditInteractionBase`/`EditAutocomplete` as appropriate; all labels via
`t('workflow.blocks.my-block.xxx')`. **Auto-registered by filename.** (Alternative without a file:
`editComponent` as an inline object, or register via `business/dev/blocks/editComponents/index.js`.)

### Step 4 — i18n (`src/locales/en/blocks.json`)

Under `workflow.blocks`:

```json
"my-block": { "name": "My block", "description": "…", "myField": "Label for myField" }
```

Optionally replicate in the other 9 locale dirs (`src/locales/{es,fr,it,pt-BR,tr,uk,vi,zh,zh-TW}/blocks.json`).
Missing keys fall back to `tasks.name` (`te()` checks everywhere).

### Done — verify

The full file-touch list: `src/utils/shared.js`, `src/workflowEngine/blocksHandler/handlerMyBlock.js`,
`src/components/newtab/workflow/edit/EditMyBlock.vue`, `src/locales/en/blocks.json` (+ optionally
other locales, `vRemixicon.js`, `blocksValidation.js`).

**What you never touch**: `WorkflowEditor.vue`, `WorkflowBlockList.vue`, `WorkflowDetailsCard.vue`,
`WorkflowEditBlock.vue`, `DroppedNode.js`, both `blocksHandler.js` registry files, `WorkflowEngine.js`,
`WorkflowWorker.js`, `convertWorkflowData.js` — all are generic over the registry.

---

## 5.2 Conditional additions

| Need | Where |
|---|---|
| Custom canvas node (custom handles, like `BlockPackage`'s dynamic IO or `BlockConditions`' per-condition outputs) | `src/components/block/BlockMyThing.vue` — wrap `BlockBase.vue`, add `Handle`s with ids `${id}-input-1` / `${id}-output-<n|name>`; set `component: 'BlockMyThing'`; auto-registers as a node type via `WorkflowEditor.vue` require.context |
| New remix icon | `src/lib/vRemixicon.js` `icons` map — or use `path:<svgPathData>` / an `https://` icon URL in the registry |
| Inline validation ("Issues" tooltip on the node) | `src/newtab/utils/blocksValidation.js` — add keyed by block id |
| Block can't be grouped | `excludeGroupBlocks` (`shared.js`) |
| onError settings make no sense | `excludeOnError` (`shared.js`) |
| Block defines variables/loops (autocomplete) | `src/utils/editor/editorAutocomplete.js` |
| Error-code → human message | `src/utils/getBlockMessage.js` + locale message maps (`getTranslateLog.js`) |
| New **trigger type** | `src/utils/workflowTrigger.js` map + `src/background/BackgroundWorkflowTriggers.js` listener + `edit/Trigger/TriggerX.vue` + entry in `SharedWorkflowTriggers.vue` `triggersData` + locale `workflow.blocks.trigger.items.<type>` |
| Docs page | External site `docs.extension.automa.site/blocks/my-block.html` (out of repo); the in-editor link works automatically once the block id exists |

---

## 5.3 A concrete walkthrough: what `parameter-prompt` touches

`parameter-prompt` (a real, minimal, recent block) touches exactly:
1. `src/utils/shared.js` — registry entry (`data.parameters`, `editComponent: 'EditParameterPrompt'`).
2. `src/workflowEngine/blocksHandler/handlerParameterPrompt.js` — background handler that opens the
   params page and watches `storage.local[promptId]` for the answer.
3. `src/components/newtab/workflow/edit/EditParameterPrompt.vue` — param panel.
4. `src/locales/en/blocks.json` — name/description/labels.

That's the whole footprint. Trace it with:
`grep -ri "parameter-prompt\|parameterPrompt" src/ --include="*.js" --include="*.vue" --include="*.json" -l`

---

## 5.4 Extending other things (non-block additions)

**Add a trigger type**: registry (`workflowTriggersMap`) → background listener → UI sub-panel →
locale. See chapter 9 for the trigger chain.

**Add a condition comparison operator**: `src/workflowEngine/utils/testConditions.js` (compare map)
+ `src/utils/compareBlockValue.js` + `conditionBuilder` schema in `src/utils/shared.js`
(operator options in the condition UI come from there).

**Add a templating function** (`{{ $myFunc(...) }}`): `src/workflowEngine/templating/templatingFunctions.js`
— one exported function; it's automatically available (dispatch via `data.functions[name]`). For the
sandbox JS-expression mode, also expose it in `src/sandbox/utils/handleBlockExpression.js` and
`src/utils/editor/editorAutocomplete.js` for autocomplete.

**Add a package ("blocks group")**: no code needed — created in the editor UI, stored in
`storage.local 'savedBlocks'`.

**Inject custom blocks without forking core**: the `business/dev/blocks/*` seam (`@business` alias)
returns `{}` by default — override `customBlocks()`, `customHandlers()`, `customEditComponents()`,
and `customParameters` from there.

---

## 5.5 Testing your block

1. `pnpm dev` → load `build/` as an unpacked extension.
2. Create a test workflow in the dashboard, add your block.
3. Use **testing mode** (`debugMode` + `$breakpoint` per block) to step block-by-block and inspect
   the resolved block data / ctxData (see chapter 6.2).
4. Check the run's log entry — `replacedValue` shows exactly how your `refDataKeys` fields were
   templated.
5. Validation errors appear as tooltips on the node if you added a `blocksValidation` entry.
