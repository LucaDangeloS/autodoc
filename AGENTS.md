# AGENTS.md

## Project Context
This repository is a fork of `pwndoc-ng`, aimed at serving as a final master's degree thesis project. 
The core objective is to modernize the application and integrate AI features to assist in report elaboration.

## Goals
- Fix immediate issues from the base `pwndoc-ng` repository.
- Upgrade dependencies and codebase.
- Implement AI-driven features for reporting.

## Methodology
- **Testing**: All implementations must be correctly verified. 
  - For automated tests (docker, logs, HTTP calls), the agent will perform them and report the results.
  - For manual testing (e.g., frontend visual checks), the agent will provide the user with the link and instructions to test it.
- Minimize comments in the codebase to maintain readability and focus on functionality.
- **Documentation**: Everything implemented in the codebase must be documented in this file under the Changes Log section. No exceptions.
- **Restart affected containers**: After each change, the agent will restart the affected containers to ensure changes are applied and to check for any runtime issues.

## Branch Policy
- **`master`**: The main branch. Default remote branch (`origin/HEAD -> origin/master`). Stable base with merged upstream PRs and critical fixes. All significant work is developed on feature branches and merged here. Do NOT use or publish a branch named `main` — the canonical default is `master`.
- **`update-dependencies`**: Dependency upgrades and related breaking-change fixes. New features that depend on the upgraded stack also live here until merged.
- **`new-ui`**: Purely visual/UI changes (theming, layout, styling, dark mode polish). No new features (e.g. AI settings, new API endpoints) should be committed to this branch.
- **`ai-features`**: AI infrastructure and feature implementation (not yet active).
- **Local `main` branch**: Used as the active working branch during development (equivalent to a working copy of `master`). Never push it to remote — push to `master` instead.

## Changes Log

### Merged upstream PRs (on `master` branch)
- Merged PR #516: Various bug fixes (cancel button at audit creation, alphabetic sort order for cloned audits)
- Merged PR #524: Hide "Delete All" buttons for non-admin users in Data Dump; updated backend base image
- Merged PR #521: Updated TipTap to v3, updated Hocuspocus version
- Merged PR #511: Added CVSS4 support (metrics, document generation, default CVSS setting, ordering by CVSS score)

### Development environment & bug fixes (on `master` branch)
- Added AGENTS.md and restored `package-lock.json`
- Fixed missing CVSS4 translations and internationalized calculator metrics (`frontend/src/i18n/*`)
- Enabled HMR (Hot Module Replacement) in the dev stack:
  - Configured nginx to proxy WebSocket upgrades to the webpack-dev-server (`frontend/.docker/nginx.dev.conf`)
  - Set `webSocketURL` in `quasar.conf.js` so the HMR client resolves the correct host/port at runtime through nginx
- Fixed input focus loss in custom data lists by generating stable unique IDs instead of index-based ones (`frontend/src/pages/data/custom/`)

### Dependency upgrades (on `update-dependencies` branch)
- Upgraded all backend dependencies: mongoose, express, jest, nodemon, supertest, swagger-autogen, etc.
- Upgraded all frontend dependencies: vue, quasar, tiptap, and related packages
- Fixed breaking changes introduced by upgrades:
  - Express 5 catch-all route syntax (`*` → `/{*path}`)
  - path-to-regexp v8 syntax in data routes
  - Mongoose 8 array initialization and query patterns
  - Added `cross-env` for portable `NODE_ENV` usage in npm scripts
  - Fixed settings and docx/html2ooxml test expectations
- Reverted bcrypt to v5 to fix login hash format incompatibility introduced by v6
- Fixed `jwt-decode` v4 import syntax in `frontend/src/services/user.js`

### Vulnerability update review modal fix (on `update-dependencies` branch)
- Disabled collaborative editing (`collab: false`) on all `basic-editor` instances in the vulnerability edit and updates modals to prevent Hocuspocus stale-state from overwriting accepted changes
- Fixed `modelValue` watcher in `editor.vue` to immediately sync content when the model changes externally in non-collaborative mode
- Added missing `btn.accept` translation key to all four locale files (en-US, fr-FR, de-DE, zh-CN)
- Fixed `vulnType` binding in the left panel of the updates modal: was bound to non-existent `currentVulnerability.vulnType`, corrected to `currentVulnerability.details[currentDetailsIndex].vulnType`
- Made the title field editable in the left panel (removed `readonly`)
- Added missing `cvssv4` binding to the left panel CVSS calculator (`v-model:cvssv4Value`)
- Fixed CVSS diff highlighting in right panel to compare both `cvssv3` and `cvssv4`
- Made all fields in the right panel (proposed update) editable: title, vulnType, description, observation, CVSS, remediation, complexity, priority, references (replaced `q-input` with `textarea-array`), and custom fields
- Fixed `vulnType` diff comparison in right panel to use `details[currentDetailsIndex].vulnType`
- Assigned unique `idUnique` values using `update._id` to prevent editor state collisions between left and right panels
- Added `acceptUpdate` function to merge proposed changes into vulnerability and save

### UI modernization (on `new-ui` branch)
- Replaced grey/blue-grey color palette with Tailwind-inspired slate scale (`$slate50`–`$slate900`)
- Added global border-radius overrides: cards 14px, inputs 10px, buttons 6px, menus/dialogs/tables/expansion items
- Dark mode: replaced material shadows with subtle 1px slate borders on cards, tables, menus, dialogs, notifications
- Unified table toolbar (`.q-table__top`) and header (`thead tr`) backgrounds in dark mode to eliminate color banding
- Restyled native `<input type="color">` pickers with rounded borders, no white edges, dark-mode aware border color
- Fixed toggle switch off-state contrast in dark mode (`$slate600` track, `$slate400` thumb)
- Added teal accent (`$secondary`) for toggle on-state across both themes
- Rounded editor toolbar corners to match parent card container (overflow hidden + explicit radius)
- Changed tab border-radius from top-only to pill shape (all corners rounded)
- Added settings page side navigation index with IntersectionObserver scroll spy, smooth scroll, active section tracking
- Fixed scroll spy flicker by pausing observer during programmatic smooth scroll
- Fixed active nav item text contrast in dark mode (`#5eeadb` teal on dark slate)
- Added mobile horizontal padding (`q-px-md`) to settings content column
- Updated light-mode page background to `$slate100`

### Docker dev environment improvements (uncommitted, to be committed to `main`)
- Added named volumes `backend-node-modules` and `frontend-node-modules` in `docker-compose-dev.yml` to persist `node_modules` across container restarts
- Removed wasteful `COPY . .` from `backend/Dockerfile.dev` since source files are bind-mounted at runtime

### AI settings in application settings (uncommitted, on `main` branch)
- Added `ai` section to backend settings model (`backend/src/models/settings.js`) with `enabled`, `public`, and `private` fields following the existing pattern
- Supported providers: OpenAI, Anthropic, Ollama, Azure OpenAI, OpenAI-compatible
- Public fields: `provider`, `model`, `temperature`, `maxTokens`
- Private fields (admin-only, never sent to browser): `apiUrl`, `apiKey`, `systemPrompt`, `userPrompt`, Azure-specific `deploymentName` and `apiVersion`
- Updated `getPublic()` projection to include `ai.enabled ai.public`
- Updated backend settings tests (`backend/tests/settings.test.js`) with AI defaults and edit test coverage
- Added AI settings card to frontend settings page with provider dropdown, model input, API URL/key fields, Azure-specific conditional fields, temperature slider, and max tokens input
- Added computed `aiDefaultUrl` property that shows the default API endpoint as placeholder based on selected provider
- Added "Advanced Settings" section with system prompt and user prompt text areas
- Added i18n translation keys for all AI settings labels in all four locale files (en-US, fr-FR, de-DE, zh-CN)

### Dev environment stabilization (on `main` branch)
- Added explicit `autodoc-net` bridge network to `docker-compose-dev.yml`; all services join it by name, replacing fragile `links` directives — prevents network loss on container recreate
- Fixed ChromaDB volume mount path (`/chroma/chroma` → `/data`) so persistent vector data actually survives restarts
- Added `backend/nodemon.json` excluding `config.json` from watch, preventing an infinite restart loop when JWT secrets are auto-generated into that file; mounted into the backend container via compose
- Removed wasteful `COPY . .` from `backend/Dockerfile.dev` (source is bind-mounted at runtime)
- Restored stable JWT secrets in `backend/src/config/config.json` dev section so tokens survive backend restarts
- Fixed Vue Router 4 compat in `frontend/src/boot/auth.js`: `router.currentRoute.path` → `router.currentRoute.value.path` (was throwing on the 14-minute refresh-token interval)
- Added named `backend-node-modules` and `frontend-node-modules` volumes to persist `node_modules` across container restarts

### AI features (on `main` branch)

#### AI settings — embedding configuration extension
- Extended `ai.public` schema with `embeddingProvider` (default `openai`), `embeddingModel` (default `text-embedding-3-small`), and `embeddingMaxDistance` (default `0.8`)
- Extended `ai.private` schema with `embeddingApiUrl`, `embeddingApiKey`, and `embeddingAzure: { deploymentName, apiVersion }`
- Updated backend settings tests to include all new embedding defaults
- Added "Embedding Configuration" section to frontend settings page with provider dropdown, model input, API URL/key, and conditional Azure fields
- Added `embeddingDefaultUrl` computed property in `settings.js` showing correct `/v1` base URL placeholder per provider
- Added `embeddingMaxDistance` slider (0.01–2.0) and Re-index All button to the embedding section in settings
- Added embedding i18n keys to all 5 locale files

#### ChromaDB vector store
- Added ChromaDB `1.5.5` service to `docker-compose-dev.yml` with persistent named volume `chroma-data-dev` mounted at `/data` (the actual data path used by ChromaDB)

#### Backend AI service layer
- New `backend/src/lib/ai-service.js`: unified `generate({ action, text, fieldName, context, aiSettings })` function
- Provider routing via LangChain: OpenAI / OpenAI-compatible → `ChatOpenAI`; Azure → `AzureChatOpenAI`; Anthropic → `ChatAnthropic`; Ollama → `ChatOllama`
- `ensureV1(url)` normalizes API base URLs by appending `/v1` if missing
- Built-in default system and user prompt templates per action (`generate`, `complete`, `rewrite`) and field context
- Template variable interpolation for `fieldName`, `findingTitle`, `similarVulnsBlock`
- Strips markdown code fences from LLM responses before returning HTML

#### Backend embedding service
- New `backend/src/lib/embedding-service.js`: ChromaDB client-based vector store operations
- `normalizeBaseUrl(url)`: strips trailing endpoint path segments (`/chat`, `/completions`, `/embeddings`) and ensures the URL ends with `/v1` — handles any format the user pastes from their LLM server's curl examples
- Forces `encodingFormat: 'float'` on all non-OpenAI providers to work around `openai` SDK v6 defaulting to `base64` (local servers like LM Studio ignore the format and return plain floats, which the SDK then fails to decode, producing all-zero vectors)
- `indexVulnerability(vuln, aiSettings)`: upserts one document per locale into ChromaDB using LangChain embeddings
- `deleteVulnerability(vulnId, aiSettings)`: removes all embeddings for a vulnerability by ID
- `searchSimilar(query, locale, aiSettings, topK)`: strict locale-filtered semantic search; filters results by `embeddingMaxDistance` threshold; returns L2 distances (lower = more similar)
- `reindexAll(aiSettings)`: re-indexes all vulnerabilities in MongoDB; logs per-item errors without aborting

#### Backend AI API routes (`backend/src/routes/ai.js`)
- `POST /api/ai/generate`: auth-protected (`audits:read`), validates AI enabled, runs RAG search then calls `ai-service.generate`, returns `{ html }`
- `POST /api/ai/search-similar`: auth-protected (`vulnerabilities:read`), calls `embeddingService.searchSimilar`, then fetches full vulnerability data from MongoDB and returns enriched results (title, description, observation, remediation, references, cvssv3, cvssv4, distance) for use in the diff/review modal
- `POST /api/ai/reindex-all`: auth-protected (`settings:update`), starts background re-indexing of all vulnerabilities; returns `{ started: true }` immediately
- Route registered in `backend/src/app.js`

#### Startup ChromaDB sync
- On backend startup, if `ai.enabled` and `ai.embeddingEnabled`, checks if the ChromaDB `vulnerabilities` collection is empty; if so, triggers `reindexAll` in background
- If the collection already has entries, assumes it is sufficiently in sync (relying on CRUD hooks) and skips the re-index
- Suppresses ChromaDB's noisy `"No embedding function configuration found"` messages at the `process.stderr` level (they are expected since embeddings are computed externally via LangChain)

#### Vulnerability CRUD embedding hooks
- `backend/src/routes/vulnerability.js`: fire-and-forget `indexVulnAsync` called after create and update; `deleteVulnAsync` called after delete
- Embedding calls are guarded by `settings.ai.enabled` and `settings.ai.embeddingEnabled` checks; errors logged but never block the HTTP response

#### Frontend AI service
- New `frontend/src/services/ai.js`: `generate(payload)`, `searchSimilar(query, locale)`, and `reindexAll()` wrapping the backend endpoints

#### TipTap AI assistant extension
- New `frontend/src/components/ai-assistant.js`: custom TipTap `Extension` with commands `aiGenerate`, `aiComplete`, `aiRewrite`
- Each command calls the backend, disables the editor during generation, then inserts/replaces content with the returned HTML
- `rewrite` operates on the current selection (or full content if nothing selected)
- Error notifications via Quasar `Notify`

#### Editor toolbar AI button
- `editor.vue`: added `fieldName` and `aiContext` props; added `aiLoading` data flag; added `AiAssistantExtension` to extensions; added `'ai'` to default toolbar array
- Toolbar section (guarded by `$settings.ai.enabled`): `q-btn-dropdown` with sparkle icon (`auto_awesome`) and three menu items (Generate, Complete, Rewrite)
- `runAi(action)` method dispatches the correct editor command with `fieldName` and `aiContext`
- CSS pulsing animation for `.ai-loading` placeholder paragraph in `app.styl`

#### Finding edit page wiring
- `frontend/src/pages/audits/edit/findings/edit/edit.html`: added `fieldName` and `:aiContext` props to all four `basic-editor` instances (description, observation, poc, remediation) passing the finding title and audit language as context

#### i18n
- Added AI action keys (`aiGenerate`, `aiComplete`, `aiRewrite`, `aiGenerating`, `aiError`, tooltips) and embedding configuration keys to all 5 locale files (en-US, fr-FR, de-DE, zh-CN, es-ES)
- Added es-ES locale file and registered it in `frontend/src/i18n/index.js` and `language-selector.vue`

### Search Similar Vulnerabilities feature (on `main` branch)

#### Backend
- `POST /api/ai/search-similar` fetches and returns full vulnerability data from MongoDB (description, observation, remediation, references, cvssv3, cvssv4) alongside L2 distance metadata, enabling the diff view in the frontend modal
- Strict locale filtering: only returns results matching the exact locale of the audit

#### Settings: `embeddingMaxDistance`
- Slider (range 0.01–2.0, step 0.01) in the Embedding Configuration section controls the L2 distance cutoff; results above this value are filtered out before being returned to the frontend
- Re-index All button fires `POST /api/ai/reindex-all` with loading state and success feedback

#### Frontend: SimilarVulnModal (`frontend/src/components/similar-vuln-modal.vue`)
- New full-screen dialog component with two-panel layout
  - Left panel: scrollable results list with L2 distance badge color-coded by threshold (green < 0.4 high match, orange < 0.8 medium, red ≥ 0.8 low)
  - Right panel: side-by-side field diff (description, observation, remediation, references, cvssv3, cvssv4) comparing current finding vs proposed values; each field shows a "changed" badge when values differ
  - Apply button overwrites finding fields and closes modal; user saves manually as usual
- Auto-selects first result on open; `fieldHasChange()` drives diff badge display

#### Frontend: Finding edit page wiring (`frontend/src/pages/audits/edit/findings/edit/`)
- Added `SimilarVulnModal` and `AiService` imports to `edit.js`
- Added `similarVulnModalOpen`, `similarVulnResults`, `similarVulnLoading`, `similarVulnError` data properties
- `searchSimilarVulns()`: builds query from finding title + stripped description (capped 500 chars), calls `AiService.searchSimilar` with audit locale, opens modal; requires title to be set (shows warning otherwise)
- `applySimilarVuln(result)`: overwrites description, observation, remediation, references, cvssv3, cvssv4 from selected result, syncs editors, marks `needSave = true`, and shows a save-reminder notification
- Search Similar button in breadcrumb toolbar, visible only when `ai.enabled && ai.embeddingEnabled`
- `<similar-vuln-modal>` mounted in `edit.html` template

#### i18n
- Added `similarVuln*`, `aiEmbeddingMaxDistance*`, `aiReindex*`, `empty`, `similarVulnCvss4` keys (25 keys per locale) to all 5 locale files

### UI modernization and rebranding (on `new-ui` branch)
- Rebranded frontend package name and product name from `pwndoc-ng` to `autodoc` in `package.json`
- Replaced pwndoc logo assets with autodoc logos; updated login page and home layout to use new branding; added `logo/` source assets directory
- Audit edit page: made left drawer collapsible — `drawerOpen` state with toggle chevron buttons; responsive breakpoint at 1024px
- Breadcrumb component: removed redundant home icon button; tightened layout (`min-height`, padding, `flex-wrap`, `gap`) for better wrapping on narrow screens with many toolbar buttons
- `quasar.conf.js`: expanded HMR `webSocketURL` to object form with explanatory comment about the nginx proxy path

## TODO
- [x] Merge PR #516 (Some Fixes)
- [x] Merge PR #524 (Hide Delete buttons)
- [x] Merge PR #521 (Update tiptapv3)
- [x] Merge PR #511 (Add Cvss4)
- [x] Implement AI-assisted editor features in findings (generate, complete, rewrite)
- [x] Implement connector from vulnerability database to vector database for AI information retrieval
- [x] Implement "Search Similar" vulnerability feature with diff modal, locale filtering, and Re-index All button
- [ ] Auto-translate vulnerabilities to all configured locales on create/update
