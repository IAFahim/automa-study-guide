# Chapter 1 — What Automa Is & How to Read the Codebase

## 1.1 The one-paragraph summary

**Automa is a Chrome/Firefox extension for automating the browser by connecting blocks** — a
node-based workflow builder (like n8n / Node-RED, but inside your browser). Workflows are drawn on a
canvas (vue-flow), saved as JSON, and executed by a custom engine that dispatches each block to
either the extension's engine context (offscreen document on Chrome) or into web pages (content
scripts). It supports triggers (cron, keyboard, context-menu, page-visit, in-page API), variables &
templating (`{{ }}`), loops, conditions, data tables, Google Sheets, exports (CSV/JSON/TXT), a
recorder that turns your clicks into workflows, and a marketplace of shareable workflow/package JSONs.

- Upstream: https://github.com/AutomaApp/automa (this guide studies v1.30.00, `main`).
- Docs site (user-facing): https://docs.extension.automa.site
- The repo studied here is a **fork with modifications** (AI-power block, offscreen-document engine
  host, Trusted-Types workaround for JS injection, download rename subsystem, Chinese comments in
  those areas, telemetry) — where behavior differs from upstream, that's called out.

## 1.2 The 60-second mental model

```
              ┌──────────────────── triggers ────────────────────┐
              │ alarms (cron/interval) · context menu · keyboard  │
              │ page-visit · on-startup · popup · dashboard       │
              │ execute.html deep-link · in-page CustomEvent      │
              └────────────────────────┬─────────────────────────┘
                                       ▼  'workflow:execute'
        ┌────────────── service worker ─────────────┐
        │  thin router; forwards to offscreen doc    │
        └──────────────────────┬────────────────────┘
                               ▼
   ┌───────────────── offscreen document ─────────────────┐
   │  WorkflowManager → WorkflowEngine → WorkflowWorker(s) │
   │  - compiles nodes/edges → connectionsMap              │
   │  - templates {{ }} in refDataKeys fields              │
   │  - runs background handlers; sends 'isBlock' payloads │
   │    to content scripts for interaction blocks          │
   └───────────┬───────────────────────────┬──────────────┘
               ▼                           ▼
      content scripts (pages)        sandbox iframe
      (click, get-text, forms…)      (!! JS exprs, JS blocks,
      iframe protocol + tokens        condition code)
               │                           │
               └────────► Dexie logs + storage.local states ◄──────
                          (dashboard reads both for live view/logs)
```

## 1.3 Reading order for mastery

1. **Chapter 2** — architecture: contexts, bundles, manifests, message protocol.
2. **Chapter 3** — the engine: life of a run, workers, branches, flow control.
3. **Chapter 4–5** — blocks: registry, handler contract, **add your own block**.
4. **Chapter 7** — templating: the full `{{ }}` reference.
5. **Chapter 6** — dashboard/editor: vue-flow, packages, recorder, selector.
6. **Chapter 8–9** — storage & triggers/external control.
7. **Chapter 10–11** — tips & tricks, dev guide.
8. Keep **chapter 12** (file map) open while reading the source.

## 1.4 How to get hands-on fast

```bash
pnpm install
# create src/utils/getPassKey.js (any string export) — required
pnpm dev          # → load build/ as unpacked extension
```

Then in the loaded extension: create a workflow, add a **trigger** → **new tab** → **get text** →
**export data** chain, run it, and open the run's log — you're looking at everything chapters 3, 6,
and 8 describe.

## 1.5 Fork-specific notes (vs upstream Automa)

- Engine runs in an **offscreen document** on Chrome (upstream runs it in the service worker with
  different keep-alive strategies).
- `handlerAiWorkflow.js` AI block (`settings.aipowerToken`), `ai-workflow` registry entry.
- `check-csp-and-inject` Trusted-Types workaround + two-stage debugger JS execution.
- Downloads rename subsystem (`storage.session` mirror + `onDeterminingFilename`).
- Telemetry: `POST /workflows/logs/report` on engine destroy + daily `/status` ping.
- `JsonRenderDemo.vue` demo page for the linked `@automa-platform/dashboard-components` package.
