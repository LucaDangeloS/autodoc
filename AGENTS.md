# AGENTS.md

## Project Context
This repository is a fork of `pwndoc-ng`, a pentest report generation tool, serving as a final master's degree thesis project. The core objective is to modernise the application and integrate AI features to assist in report elaboration.

## Goals
- Fix immediate issues from the base `pwndoc-ng` repository.
- Upgrade dependencies and codebase.
- Implement AI-driven features for reporting.

## Methodology
- **Testing**: All implementations must be correctly verified.
  - For automated tests (docker, logs, HTTP calls), the agent will perform them and report the results.
  - For manual testing (e.g. frontend visual checks), the agent will provide the user with the link and instructions to test it.
- **Comments**: Minimise comments in the codebase to maintain readability.
- **Documentation**: Everything implemented must be documented in this file under the Changes Log section. No exceptions.
- **Restart affected containers**: After each change, the agent **must** restart the affected containers and check logs before reporting success:
  - **Backend changes** (`backend/src/**`): `docker compose -f docker-compose-dev.yml restart backend`
  - **Frontend changes** (`frontend/src/**`, `frontend/quasar.conf.js`): HMR picks up most changes automatically; only restart if build config or a boot file is added → `docker compose -f docker-compose-dev.yml restart frontend-app`
  - **Infra changes** (`docker-compose-dev.yml`, `Dockerfile.dev`): `docker compose -f docker-compose-dev.yml up -d`

## Branch Policy
- **`master`**: Main branch, canonical default (`origin/HEAD → origin/master`). All significant work is merged here. Do **not** push a branch named `main` to remote.
- **`update-dependencies`**: Dependency upgrades and related breaking-change fixes.
- **`new-ui`**: Purely visual/UI changes — no new features or API endpoints.
- **`ai-features`**: AI infrastructure and features (not yet active as a separate branch; work has been merged to master).

---

## Dev Environment Reference

### Default credentials
- **Username**: `admin`
- **Password**: `Admin1admin2`

### Access
- App: `https://localhost:8443`
- Backend API: `https://localhost:8443/api`

### Test data on fresh install
After a full restart with data destruction (`docker compose down -v`), the MongoDB volume is wiped. Upon first boot the admin user is pre-seeded automatically. Everything else must be re-created.

**Order matters: add languages before vulnerabilities.**

```bash
TOKEN=$(curl -sk -X POST https://localhost:8443/api/users/token \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Admin1admin2"}' | jq -r '.datas.token')

# 1 — languages
curl -sk -X POST https://localhost:8443/api/data/languages \
  -H "Content-Type: application/json" -H "Cookie: token=JWT $TOKEN" \
  -d '{"language":"English","locale":"en-GB"}'

curl -sk -X POST https://localhost:8443/api/data/languages \
  -H "Content-Type: application/json" -H "Cookie: token=JWT $TOKEN" \
  -d '{"language":"Spanish","locale":"es-ES"}'

# 2 — vulnerabilities
curl -sk -X POST https://localhost:8443/api/vulnerabilities \
  -H "Content-Type: application/json" -H "Cookie: token=JWT $TOKEN" \
  -d @backend/tests/fixtures/test-vulnerabilities.json
```

The backend authenticates via cookie (`token=JWT <token>`), not a Bearer header.

The fixture `backend/tests/fixtures/test-vulnerabilities.json` contains 10 canonical test vulnerabilities (SQL Injection, XSS, IDOR, SSRF, Broken Authentication, Path Traversal, Command Injection, Insecure Deserialization, Security Misconfiguration, Sensitive Data Exposure) using the `en-GB` locale.

---

## Changes Log

### Upstream PRs merged
- PR #516: Various bug fixes (cancel button at audit creation, alphabetic sort order for cloned audits)
- PR #524: Hide "Delete All" buttons for non-admin users in Data Dump; updated backend base image
- PR #521: Updated TipTap to v3, updated Hocuspocus version
- PR #511: Added CVSS 4.0 support (metrics, document generation, default CVSS setting, ordering by CVSS score)

### Dependency upgrades
- Upgraded all backend dependencies: mongoose, express, jest, nodemon, supertest, swagger-autogen, etc.
- Upgraded all frontend dependencies: vue, quasar, tiptap, and related packages
- Fixed breaking changes: Express 5 catch-all syntax, path-to-regexp v8, Mongoose 8 array patterns, `cross-env` for portable `NODE_ENV`
- Reverted bcrypt to v5 to fix login hash format incompatibility introduced by v6
- Fixed `jwt-decode` v4 import syntax in `frontend/src/services/user.js`

### Dev environment stabilisation
- Enabled HMR: configured nginx to proxy WebSocket upgrades; set `webSocketURL` in `quasar.conf.js`
- Added explicit `autodoc-net` bridge network; all services join by name, replacing fragile `links`
- Added `backend/nodemon.json` excluding `config.json` from watch to prevent infinite restart loop
- Restored stable JWT secrets in `backend/src/config/config.json` dev section
- Fixed ChromaDB volume mount path (`/chroma/chroma` → `/data`)
- Added named `backend-node-modules` and `frontend-node-modules` volumes to persist across restarts
- Fixed Vue Router 4 compat in `frontend/src/boot/auth.js`: `router.currentRoute.path` → `router.currentRoute.value.path`
- Fixed `backend/src/models/settings.js` crash loop: `Settings.create({})` duplicate-key error now caught gracefully; `liveSettings.save()` errors logged instead of thrown as unhandled rejections
- Added global `uncaughtException` / `unhandledRejection` handlers in `backend/src/app.js` for visibility
- Fixed input focus loss in custom data lists by generating stable unique IDs instead of index-based ones

### UI modernisation and rebranding
- Rebranded from `pwndoc-ng` to `autodoc` in `package.json`, logo assets, login page, and home layout
- Replaced grey/blue-grey colour palette with Tailwind-inspired slate scale (`$slate50`–`$slate900`)
- Added global border-radius overrides: cards 14px, inputs 10px, buttons 6px
- Dark mode: replaced material shadows with subtle 1px slate borders; fixed toggle off-state contrast; added teal accent for toggle on-state
- Unified table toolbar and header backgrounds in dark mode
- Restyled native `<input type="color">` pickers with dark-mode awareness
- Added settings page side navigation with IntersectionObserver scroll spy and smooth scroll
- Audit edit page: collapsible left drawer with toggle chevron; responsive breakpoint at 1024px
- Breadcrumb component: tightened layout for narrow screens

### Vulnerability update review modal fixes
- Disabled collaborative editing on all `basic-editor` instances in the vulnerability edit/update modals
- Fixed `modelValue` watcher in `editor.vue` to sync content immediately in non-collaborative mode
- Fixed `vulnType` binding, title editability, CVSS v4 binding, and diff highlighting in the update review modal
- Added `acceptUpdate` function to merge proposed changes into vulnerability and save

### AI features

#### Settings model
- Added `ai` section to `backend/src/models/settings.js` with `enabled`, `public` (provider, model, temperature, maxTokens, embedding config, vision config), and `private` (apiKey, apiUrl, system/user prompts — never sent to browser)
- Supported providers: OpenAI, Anthropic, Ollama, Azure OpenAI, OpenAI-compatible (for generation, embedding, and vision independently)

#### Backend services
- `backend/src/lib/ai-service.js`: unified `generate({ action, text, fieldName, context, aiSettings })` with provider routing via LangChain; built-in prompt templates per action (`generate`, `complete`, `rewrite`, `fill-proofs`)
- `backend/src/lib/embedding-service.js`: ChromaDB vector store — `indexVulnerability`, `deleteVulnerability`, `searchSimilar`, `reindexAll`; forces `float` encoding for non-OpenAI providers; strict locale filtering
- `backend/src/lib/vision-service.js`: multimodal proof analysis pipeline — parses POC HTML for images, fetches base64 data, calls vision LLM, optionally anonymises output with LLM instruction or regex

#### Backend routes (`backend/src/routes/ai.js`)
- `POST /api/ai/generate`: RAG search + generation, returns `{ html }`
- `POST /api/ai/search-similar`: semantic similarity search, returns enriched vulnerability data with L2 distance
- `POST /api/ai/analyze-proofs`: vision analysis + similarity search for proof-based workflow
- `POST /api/ai/reindex-all`: background re-indexing of all vulnerabilities

#### ChromaDB
- Added ChromaDB `1.5.5` service to `docker-compose-dev.yml` with persistent volume `chroma-data-dev` at `/data`
- On backend startup, if embedding is enabled and collection is empty, triggers `reindexAll` in background
- Vulnerability CRUD hooks in `backend/src/routes/vulnerability.js` fire-and-forget index/delete on create/update/delete

#### Frontend AI
- `frontend/src/services/ai.js`: `generate`, `searchSimilar`, `reindexAll`, `analyzeProofs`
- `frontend/src/components/ai-assistant.js`: custom TipTap extension with `aiGenerate`, `aiComplete`, `aiRewrite` commands
- Editor toolbar: `q-btn-dropdown` with sparkle icon guarded by `$settings.ai.enabled`, plus CSS pulse animation during generation
- Finding edit page: all four `basic-editor` instances wired with `fieldName` and `aiContext` props

#### Search Similar Vulnerabilities
- `frontend/src/components/similar-vuln-modal.vue`: full-screen two-panel dialog — left panel shows results with colour-coded L2 distance badges; right panel shows field diff with "changed" badges; Apply button overwrites finding fields

#### Proof-based similarity search
- User fills Proofs tab → clicks "Search from Proofs" → vision model analyses screenshots → embedding search → gen model writes proof narrative with images interleaved
- `SimilarVulnModal` extended with proof mode: shows vision summary, generated POC preview, loading states

#### i18n
- Added AI, embedding, vision, and proof workflow keys to all 5 locale files (en-US, fr-FR, de-DE, zh-CN, es-ES)
- Added es-ES locale file; registered in `frontend/src/i18n/index.js` and `language-selector.vue`

### Retest feature

Audits can be marked as retests. When enabled, every finding gains a "Retest Evidence" tab and a Pass/Fail toggle.

#### Backend
- `backend/src/models/audit.js`: `isRetest: Boolean` on `AuditSchema`; `retestEvidence: String` and `retestPassed: Boolean|null` on the `Finding` subdocument
- `backend/src/routes/audit.js`: general update accepts `isRetest`; finding create/update accept `retestEvidence` and `retestPassed`
- `backend/src/lib/report-generator.js`: exposes `audit.is_retest`, `finding.retest_evidence`, `finding.retest_passed` to docx templates
- `backend/src/models/audit.js` `getGeneral` projection updated to include `isRetest`

#### Report template variables

| Variable | Type | Description |
|---|---|---|
| `audit.is_retest` | Boolean | `true` if the audit is a retest |
| `finding.retest_evidence` | HTML object | Use with `\| convertHTML` |
| `finding.retest_passed` | Boolean / null | `true` = passed, `false` = failed, `null` = not set |

```
{#findings}
  {#is_retest}
  Result: {#retest_passed}PASSED{/retest_passed}{^retest_passed}FAILED{/retest_passed}
  {retest_evidence | convertHTML}
  {/is_retest}
{/findings}
```

#### Frontend
- General settings card: `isRetest` toggle
- Finding edit page: "Retest Evidence" tab (visible only when `localAudit.isRetest`); Pass/Fail `q-btn-toggle` with clear button; `basic-editor` for evidence content
- `TemplateHint` component (`frontend/src/components/template-hint.vue`): a `?` icon that shows the docxtemplater variable name on hover; currently wired to retest fields as a pilot — **intended to be extended to all finding/audit fields** (see TODO)
- i18n: 10 keys per locale across all 5 locale files

### Finding data-loss fix and dirty-tracking overhaul

#### Root causes fixed
1. **Race condition on navigation**: navigating before `getFinding()` resolved passed the unsaved-changes guard with blank data
2. **Incomplete dirty tracking**: title, vulnType, priority, CVSS, etc. never set `needSave`; editor initialisation noise could set it falsely
3. **`editor.vue` beforeUnmount loop**: `while(1)` waiting for WebSocket could block navigation forever

#### Changes
- **Parallel fetches**: `getCustomFields()` and `getFinding()` run via `Promise.all`; `initCustomFieldsForFinding()` runs only after both resolve
- **`loading` flag**: blocks navigation and disables all inputs while data is in flight; spinner overlays on all cards
- **Structural dirty check**: deep `watch` on `finding` using `_.isEqual(finding, findingOrig)`; watcher suppressed during baseline sync operations via `_baselining` flag
- **`findingOrig` snapshotted after editor init**: baseline is taken in `onEditorReady()` — after TipTap has connected and normalised the HTML — so the snapshot always matches what the editor actually contains, eliminating false dirty detection
- **`syncEditors` guarded**: only called in route guards when `loading === false`
- **`editor.vue`**: `beforeUnmount` loop now has a 3-second deadline before proceeding regardless of WebSocket state

### Data section navigation reorganisation

The sidebar left-drawer now has clearly separated sections:

| Section | Path | Tabs |
|---|---|---|
| People | `/data/collaborators`, `/companies`, `/clients`, `/templates` | (unchanged) |
| **Vulnerabilities** | `/data/vulnerabilities-data` | Languages, Vulnerability Types, Vulnerability Categories |
| **Audits** | `/data/audits-data` | Audit Types |
| **Custom Data** | `/data/custom` | Custom Fields, Custom Sections |
| Import/Export | `/data/dump` | (unchanged) |

The custom component gained a `section` prop (`'vulnerabilities'`, `'audits'`, `'custom'`) controlling which tabs are shown. Two thin wrapper pages (`vulnerabilities-data/index.vue`, `audits-data/index.vue`) pass the appropriate prop — no logic duplication.

### Dark mode fixes
- `.body--dark .bg-grey-2` → `$slate700` — custom fields preview drag area
- `.body--dark .q-tabs.bg-white` → `$slate800` — tab bars with hardcoded `bg-white`
- `.body--dark .bg-grey-1` → `$slate800` — CVSS calculator card in finding edit

### Toast position
- `frontend/src/boot/notify-defaults.js`: `Notify.setDefaults({ position: 'top-right', offset: [10, 70] })` — pushes toasts 70px from the top, clearing the navbar and breadcrumb action buttons
- Registered in `quasar.conf.js` boot array

### Settings nav label casing fix
- `frontend/src/pages/settings/settings.html`: changed nav item label class from `text-caption` to `text-body2` to restore correct capitalisation

### TemplateHint extended to all finding and audit fields
- `frontend/src/pages/audits/edit/findings/edit/edit.html`: `<template-hint>` added inline next to every field label across all tabs (title, type, description, observation, references, proofs/poc, affected assets, CVSS score, remediation complexity, priority, remediation, retest result, retest evidence)
- `frontend/src/pages/audits/edit/general/general.html`: `<template-hint>` added next to every audit field (name, language, company, client, reviewers, collaborators, isRetest toggle, start date, end date, reporting date, scope)
- `frontend/src/pages/audits/edit/general/general.js`: imported and registered `TemplateHint` component
- Fields using `q-input`/`q-select` use `label-slot` with the hint inside `<template #label>`; standalone labels use a `row items-center` wrapper with the hint beside the text
- Template variable strings shown match the exact docxtemplater syntax used in report templates (e.g. `finding.description | convertHTML`, `{#audit.collaborators}.firstname .lastname{/audit.collaborators}`)

---

## TODO
- [x] Merge upstream PRs (#516, #524, #521, #511)
- [x] Implement AI-assisted editor features in findings (generate, complete, rewrite)
- [x] Implement vulnerability vector database connector (ChromaDB + LangChain embeddings)
- [x] Implement Search Similar Vulnerabilities feature with diff modal
- [x] Implement proof-based similarity search (vision → embedding → generation)
- [x] Implement Retest feature with report template variables and TemplateHint component
- [x] Fix finding data-loss race condition and dirty-tracking
- [x] Reorganise Data section navigation
- [x] Extend TemplateHint to all finding and audit fields for full inline template documentation
- [ ] Auto-translate vulnerabilities to all configured locales on create/update
- [ ] Checklists feature (design TBD)
