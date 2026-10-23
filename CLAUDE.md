# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

God's Eye View — a browser-based 3D globe (CesiumJS) with live public-data layers (aircraft, ships, satellites, CCTV, weather, …) and OpenAI Realtime voice control. Vanilla JS ES modules + Vite, **no UI framework**. Node 24.14.x or 26.x.

Fork layout: `origin` = `TerminatedProcess/gods-eye-view`, `upstream` = `bilawalsidhu/gods-eye-view` (push disabled). `main` mirrors upstream — never commit to it; work lives on `mryan`. Sync with `gitupdate` (in `.salias_f`): fast-forwards `main` from upstream, rebases `mryan` on top, force-with-lease pushes.

**`docs/CURRENT-STATE.md` is the authoritative runtime reference** (~6k lines — grep it for the feature, don't read it whole). It also names the QA gate for many features.

## Commands

```bash
npm install && npm run doctor       # setup + environment policy checks
./scripts/dev-fresh.sh              # dev server on :4173 with CCTV packs/env wired (or: npm run dev)
npm run build && npm run preview    # check a production build (preview also serves local /api providers)

npm test                            # all unit tests (src/**/*.test.mjs + server/**/*.test.mjs, node:test)
node --test src/geoDistance.test.mjs   # single test file
npm run test:track                  # tracking regression — needs dev server running on :4173

npm run format                      # prettier on adopted scope; then:
npm run format:check
npm run check:boundaries            # import-direction + package-export boundary gates
npm run i18n                        # must pass when user-visible text changes
npm run layer-token:check -- --base-ref origin/main
```

PR baseline: `npm run build`, `npm test`, `npm run test:track` all green, plus the feature gate for what you touched.

- **Feature gates:** `scripts/qa-*.mjs` drive the real app in headless Chromium (puppeteer). Each entrypoint's header documents args/requirements — most need the dev server up, some need a provider key or `--fixtures`. Don't assume uniform arguments. CI runs none of them except `qa:street-level:fixtures`.
- Allocation microbenchmarks (`src/data/focusAllocations.test.mjs`, `src/overlays/worldOverlayAllocation.test.mjs`) are calibrated for **Node 24** and run separately with `--expose-gc`; `scripts/run-unit-tests.mjs` handles that.
- New test directories must be added to `UNIT_TEST_ROOTS` in `scripts/run-unit-tests.mjs` (a test asserts coverage).
- Headless runs use SwiftShader — pixels are relative evidence only, not real-GPU visual sign-off.

## Architecture

- **Entry/assembly:** `src/main.js` → `src/standalone/application.js`, which supplies four constructors (`createScene` → `createControls` → `createData` → `createTools`) to `createApplication` in `src/app/`. Each gets `signal` + `defer(cleanup)`; teardown runs tools, controls, data, scene. Register `defer` right after acquiring a resource, before any `await`. Only one standalone app per page. See `docs/APPLICATION.md`.
- **Ownership** (full table in `docs/CODE-BOUNDARIES.md`):
  - `src/layers/<family>/` — layer factories split into source / record / controller / renderer. Layer interface: `init/enable/disable/update/destroy/getStats`, optional `getDetectableObjects`. Copy an existing layer as a template.
  - `src/sources/` — portable protocols; must not reach Cesium, Node, browser globals (`window`, `document`) or app/rendering code.
  - `src/ui/` controllers, `src/services/` app operations, `src/search/` geocoding/routing providers, `src/styles/` GLSL post-process shaders.
  - `src/data/<layer>.js` — legacy compatibility facades; new code goes to the focused owner.
  - `src/voice/` — `actionSchemas.js` (tool args), `gevActions.js` (client execution); server-side tool descriptions in `server/providers/openai/tools.js`.
- **Server:** `vite.config.js` re-exports `server/standalone/vite.config.js`. There is no separate backend — `server/providers/*` are Vite middleware plugins exposing `/api/*` proxies. **Secrets stay server-side**; the browser only sees the restricted Google Maps key and ephemeral tokens. Proxies fetch only server-registered URLs, never client-supplied ones. `server/mcp/` is an MCP server (`npm run mcp` for stdio).
- **Package exports:** `package.json` `exports` is the public inventory; each export is assigned in `scripts/package-boundaries.json`. Adding/expanding an export means updating that file and consumer tests. Reusable exports must not import standalone bootstrap or Node services.
- `check:boundaries` statically parses every runtime file in `src/` and `server/`: browser graphs can't reach Node/server/test modules, providers can't import rendering, computed dynamic imports and `require` are rejected.

## Rules that bite

- **World text labels** must use the world-overlay host; Cesium text labels are forbidden (a test enforces it).
- **i18n:** all user-visible text via `t()`, `tn()`, `msg()` or `data-i18n*`. English text is the key. See `docs/I18N.md`.
- **Share-link layer tokens** (`LAYER_STATE_REGISTRY` + `src/data/layerStateTokenReservations.json`) are immutable — never change, reuse or repurpose one. Get the next with `npm run layer-token:next -- <layer-id>` after rebasing on `main`, add to both registry and ledger, and update the pinned registered-layer count test. Details in `CONTRIBUTING.md`.
- **Changelog:** add `changelog.d/<pr-number>-<slug>.md` (a single `-` bullet, written as it should ship); never edit `CHANGELOG.md` directly.
- Runtime behavior changes update `docs/CURRENT-STATE.md` in the same PR. New/changed data sources update `DATA_SOURCES.md` (license + attribution); don't bundle data you can't redistribute — fetch at runtime.
- **Style:** 2-space indent, single quotes, semicolons, LF; JSDoc on exported functions. Keep mechanical formatting in separate commits. Formatting scope = auto-discovered runtime roots (`scripts/format-runtime.json`) + explicit list (`scripts/format-scope.json`).
- Some tests assert on source text shapes; when a move/rewrap breaks one, preserve the behavioral coverage rather than deleting it.
- Keys: all optional (keyless Esri imagery + terrain by default). Put them in `.env` (see `.env.example`); browser keys are baked in at build time, so rebuild after changing them.
- Reviewing community PRs: follow `.agents/skills/community-pr/SKILL.md` and `docs/MAINTAINER_WORKFLOW.md`.
