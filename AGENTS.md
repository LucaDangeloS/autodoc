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

## Branch Policy
- **`master`**: Stable base with merged upstream PRs and critical fixes.
- **`update-dependencies`**: Dependency upgrades and related breaking-change fixes. New features that depend on the upgraded stack also live here until merged.
- **`new-ui`**: Purely visual/UI changes (theming, layout, styling, dark mode polish). No new features (e.g. AI settings, new API endpoints) should be committed to this branch.
- **`ai-features`**: AI infrastructure and feature implementation (not yet active).

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

### Docker dev environment improvements (uncommitted)
- Added named volumes `backend-node-modules` and `frontend-node-modules` in `docker-compose-dev.yml` to persist `node_modules` across container restarts
- Removed wasteful `COPY . .` from `backend/Dockerfile.dev` since source files are bind-mounted at runtime

### AI settings in application settings (uncommitted)
- Added `ai` section to backend settings model (`backend/src/models/settings.js`) with `enabled`, `public`, and `private` fields following the existing pattern
- Supported providers: OpenAI, Anthropic, Ollama, Azure OpenAI, OpenAI-compatible
- Public fields: `provider`, `model`, `temperature`, `maxTokens`
- Private fields (admin-only, never sent to browser): `apiUrl`, `apiKey`, Azure-specific `deploymentName` and `apiVersion`
- Updated `getPublic()` projection to include `ai.enabled ai.public`
- Updated backend settings tests (`backend/tests/settings.test.js`) with AI defaults and edit test coverage
- Added AI settings card to frontend settings page with provider dropdown, model input, API URL/key fields, Azure-specific conditional fields, temperature slider, and max tokens input
- Added computed `aiDefaultUrl` property that shows the default API endpoint as placeholder based on selected provider
- Added i18n translation keys for all AI settings labels in all four locale files (en-US, fr-FR, de-DE, zh-CN)

## TODO
- [x] Merge PR #516 (Some Fixes)
- [x] Merge PR #524 (Hide Delete buttons)
- [x] Merge PR #521 (Update tiptapv3)
- [x] Merge PR #511 (Add Cvss4)
