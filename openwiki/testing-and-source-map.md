---
type: Engineering Guide
title: Testing guidance and source map
description: Change-oriented verification guide for Score Shield pipeline logic, processor startup, artifact cleanup, setup, and the built hosted UI surface.
resource: /tests
tags: [testing, source-map, quality, development]
openwiki:
  roles: [testing, repository]
  change_kinds: [validation, source-map]
  source_paths: [package.json, server/pipeline.mjs, server/index.mjs, app/page.tsx, scripts/clean-artifacts.mjs]
  test_paths: [tests/pipeline.test.mjs, tests/server-startup.test.mjs, tests/clean-artifacts.test.mjs, tests/setup.test.mjs, tests/rendered-html.test.mjs]
  validation_commands: [npm run test:unit, npm run lint, npm test, npm run setup:check]
---

# Testing guidance and source map

Use this page to choose the smallest check that proves a change. The highest-risk domain behavior is the deterministic timeline described in [Score timeline workflow](workflows/score-timeline.md); the boundary between its local processor and the hosted UI is described in [Architecture overview](architecture/overview.md).

## Verification commands

| Command | Scope | Use when |
| --- | --- | --- |
| `npm run test:unit` | Node test runner over cleanup, pipeline/config, processor-startup, and setup-plan suites. | Default focused check for processor, pipeline, cache, cleanup, startup, or setup edits. |
| `npm run lint` | ESLint excluding generated `dist` and `.next`. | Pair with any source edit. |
| `npm run setup:check` | Read-only Node/media-tool prerequisite check. | Setup/platform diagnostics; conditional for setup changes. |
| `npm run build` | Vinext/Cloudflare deployment build only. | Conditional build investigation; it does not test processor behavior. |
| `npm test` | Unit tests, production build, then rendered Worker HTML test. | UI, Worker, build, metadata, deployment, or shared consumer-contract changes. |

`npm test` is the shipped-surface check: `tests/rendered-html.test.mjs` imports `dist/server/index.js` after the production build. Passing a `server/` unit test alone does not validate the consumer-facing hosted UI. Conversely, do not run the expensive build route by default for a reconciliation-only change.

## Focused test ownership

| Test file and retrievable behavior | Source contract it protects | Narrow validation |
| --- | --- | --- |
| `tests/pipeline.test.mjs` — interval bounds, final-two-minute cadence, midpoint timestamps, URL-equivalent cache key, null scoreboard observations, regression rejection, label normalization, rapid transitions, closing-window confirmation, VTT output | `server/config.mjs` and exported `server/pipeline.mjs` helpers | `npm run test:unit` |
| `tests/server-startup.test.mjs` — rejects missing/blank key and exits before listening | `validateProcessorEnvironment` and `server/index.mjs` startup guard | `npm run test:unit` |
| `tests/clean-artifacts.test.mjs` — removes UUID jobs while preserving cache/unrelated directories; absent root is safe | `cleanJobArtifacts` | `npm run test:unit` |
| `tests/setup.test.mjs` — Homebrew aggregation, apt plan, separate Winget IDs | `scripts/setup.mjs` planning | `npm run test:unit`, then `npm run setup:check` if prerequisites are relevant |
| `tests/rendered-html.test.mjs` — built Worker returns score-safe landing shell and expected FRA/ENG demo content without template residue | `app/page.tsx`, Worker/build integration | `npm test` |

No suite currently exercises HTTP input validation, SSE reconnect/failure behavior, real downloads/tools, OpenAI responses, iframe interaction, or accessibility. Add a narrow test before expanding those contracts; do not claim production coverage from the existing suites.

## Change paths

- **Score detection, sampling, source cache, reconciliation, or VTT:** begin in `server/config.mjs` and `server/pipeline.mjs`, then consult [the workflow extension recipe](workflows/score-timeline.md#safe-extension-recipe). Add the matching behavior test to `tests/pipeline.test.mjs`; run `npm run test:unit` and `npm run lint`.
- **Processor startup or API/SSE:** start in `server/startup.mjs` and `server/index.mjs`. The existing startup tests are not API tests, so add boundary coverage for URL, body, CORS, or SSE changes. Run `npm run test:unit` and `npm run lint`; use `npm test` only if the client/build contract changes.
- **UI, demo, title/status, or VTT download:** read `app/page.tsx` with [browser interface and player](architecture/overview.md#browser-interface-and-player). Preserve the active-cue and final-reset invariant. Run `npm run lint` and `npm test`.
- **Artifact cleanup:** change `scripts/clean-artifacts.mjs` and the cleanup test together. Preserve the UUID-only deletion boundary and cache preservation. Run `npm run test:unit` and `npm run lint`.
- **Local setup:** change `scripts/setup.mjs` and setup-plan tests together. Preserve the distinction between diagnostic `--check`, non-mutating `--dry-run`, and installation-capable default setup; run `npm run test:unit` and `npm run setup:check`.
- **Worker/build configuration:** start at `worker/index.ts`, `vite.config.ts`, and `build/sites-vite-plugin.ts`; run `npm test` and `npm run lint` because the built Worker is the relevant public surface.
- **Persistence:** establish a schema, migration, and `.openai/hosting.json` binding before using `getDb()` in a product path. Current D1 code is scaffolding, not an integration to validate.

## Source map

| Area | Primary files | Responsibility and first symbols |
| --- | --- | --- |
| Product UI and demo | `app/page.tsx`, `app/preview/page.tsx`, `app/globals.css`, `app/layout.tsx` | `ScoreShieldApp` selects views; `PlayerScreen` derives score/title; `YouTubePlayer` accepts iframe time/end messages. |
| Processor boundary | `server/index.mjs`, `server/startup.mjs` | startup key guard, `validYouTubeUrl`, in-memory job registry, HTTP routes, SSE broadcast. |
| Media/AI/timeline | `server/config.mjs`, `server/pipeline.mjs` | cadence plan, `sourceCacheKey`, subprocesses, `ObservationSchema`, `reconcileObservations`, `cuesToVtt`. |
| Local developer lifecycle | `scripts/setup.mjs`, `scripts/dev.mjs`, `scripts/clean-artifacts.mjs`, `.env.example`, `package.json` | prerequisite install/check, child-process orchestration, constrained cleanup, commands/configuration. |
| Hosted runtime/build | `worker/index.ts`, `vite.config.ts`, `build/sites-vite-plugin.ts`, `.openai/hosting.json` | Worker routing, Cloudflare/Vinext build, packaged deployment metadata. |
| Future persistence | `db/index.ts`, `db/schema.ts`, `drizzle.config.ts`, `examples/d1/` | Unwired D1/Drizzle starting point only. |
| Documentation workflow | `.github/workflows/openwiki-update.yml`, `README.md`, `AGENTS.md` | Wiki refresh configuration and contributor requirements. |

The repository has a shallow history in this checkout: the last wiki metadata `gitHead` is unavailable, while current `HEAD` is a grafted root. This update therefore treats current source and tests as ground truth rather than presenting an unverifiable commit progression.
