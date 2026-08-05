---
type: Project Guide
title: Score Shield quickstart
description: Engineer entry point for Score Shield, a local spoiler-free sports-video score timeline processor and hosted React demonstration.
resource: /README.md
tags: [score-shield, onboarding, spoiler-free-video, node, react]
openwiki:
  roles: [repository, workflow]
  change_kinds: [onboarding, runtime, public-api]
  source_paths: [package.json, app/page.tsx, server/index.mjs, server/pipeline.mjs]
  test_paths: [tests/pipeline.test.mjs, tests/server-startup.test.mjs, tests/rendered-html.test.mjs]
  validation_commands: [npm run test:unit, npm run lint, npm test]
---

# Score Shield quickstart

Score Shield is a reference implementation for spoiler-free sports playback. The local processor downloads authorized YouTube media, samples frames, asks a vision model for candidate scoreboard readings, and turns confirmed observations into time-bounded score cues. The React player displays only the complete score state at its current playhead; generated metadata, not the original video, is the spoiler-control layer.

The product has two deliberate runtimes: the Cloudflare-compatible Vinext UI/demo and a **local-only** Node processor that requires `yt-dlp`, FFmpeg/FFprobe, writable artifacts, and `OPENAI_API_KEY`. The processor is loopback-bound and is not a supported public service. [Architecture overview](architecture/overview.md) explains that boundary, while [Score timeline workflow](workflows/score-timeline.md) owns the cue contract.

## Start locally

Node.js 22.13+ is required (`package.json`). Do not inspect or commit `.env`; use `.env.example` as the non-secret configuration contract.

```bash
npm run setup        # Installs npm dependencies and missing media tools; creates .env if absent
# Add OPENAI_API_KEY to .env for real analysis
npm run dev          # Starts the processor and UI
```

Open `http://localhost:3000` and use `http://localhost:8787/health` for the processor health check. The built-in demo requires neither a processor nor a key. `npm run setup` may use a package manager and `sudo`; use `npm run setup:check` for a read-only prerequisite check or `node scripts/setup.mjs --dry-run` before allowing installation. The processor exits before listening when the key is blank or missing.

## Product invariants

1. Never reveal a future score, scorer, event, or final result in visible UI, titles, labels, accessibility text, or logs before the relevant cue is active.
2. Do not fetch or render the original YouTube title in product UI.
3. Resolve score from current media time: backward seeks restore the earlier cue and forward seeks resolve the destination cue.
4. Display `final` only after the embedded player reports completion; seeking back before the final cue clears that status.
5. Treat model responses as candidate observations. Deterministic reconciliation, not prompting, authorizes cues.
6. Keep real processing local unless a target runtime supplies secure secrets, native tools, durable artifacts, and long-running-job controls.
7. Process only media the operator is authorized to download and analyze.

YouTube-owned chrome can still reveal source information after the user uncovers the embed. The initial cover reduces that surface but does not make a third-party player fully spoiler-safe.

## Task routing

| Change area or user intent | Relevant wiki page | Exact source entry points | Important symbols or types | Focused tests | Minimal validation command |
| --- | --- | --- | --- | --- | --- |
| Score interpretation, VTT, frame cadence, or cache behavior | [Score timeline workflow](workflows/score-timeline.md) | `server/config.mjs`, `server/pipeline.mjs` | `buildFrameSamplingPlan`, `samplingFrameTimestamp`, `sourceCacheKey`, `reconcileObservations`, `cuesToVtt` | `tests/pipeline.test.mjs` | `npm run test:unit` |
| Job endpoint, SSE, validation, or artifact download | [Architecture overview](architecture/overview.md) | `server/index.mjs`, `server/startup.mjs` | `validYouTubeUrl`, `parseFrameInterval`, `validateProcessorEnvironment`, `processVideo` | `tests/server-startup.test.mjs` | `npm run test:unit` |
| Submission, progress display, player title/status, or demo | [Architecture overview](architecture/overview.md) | `app/page.tsx`, `app/preview/page.tsx` | `ScoreShieldApp`, `PlayerScreen`, `YouTubePlayer`, `cueAt` | `tests/rendered-html.test.mjs` | `npm test` |
| Setup, local processes, environment, or artifact retention | [Operations and integrations](operations-and-integrations.md) | `scripts/setup.mjs`, `scripts/dev.mjs`, `scripts/clean-artifacts.mjs`, `.env.example` | `cleanJobArtifacts`, `main` | `tests/setup.test.mjs`, `tests/clean-artifacts.test.mjs` | `npm run test:unit` |
| Hosted UI, worker build, or deployment metadata | [Architecture overview](architecture/overview.md) | `worker/index.ts`, `vite.config.ts`, `build/sites-vite-plugin.ts`, `.openai/hosting.json` | Worker `fetch`, `sitesPlugin` | `tests/rendered-html.test.mjs` | `npm test` |
| Test selection or source ownership | [Testing and source map](testing-and-source-map.md) | `package.json`, `tests/` | npm scripts | relevant suite named there | command named there |

Run `npm run lint` in addition when changing source. `npm test` is the consumer-facing hosted-UI check because it builds and then exercises the rendered Worker HTML; do not use it by default for a pipeline-only edit.

## Documentation map

- [Architecture overview](architecture/overview.md) — UI/processor request flow, API/SSE contract, artifact availability, and hosted boundary.
- [Score timeline workflow](workflows/score-timeline.md) — sampling, source caching, model validation, reconciliation, WebVTT, and playback semantics.
- [Operations and integrations](operations-and-integrations.md) — safe setup, secrets/configuration, cleanup, external tools, hosted build, and wiki automation.
- [Testing and source map](testing-and-source-map.md) — focused suites, shipped-surface checks, and source ownership.

## Backlog

- **Public processor hardening** — `server/index.mjs`, `server/pipeline.mjs`: authentication, ownership, quotas, cancellation, durable job state, retry policy, artifact retention, and SSE recovery do not exist. The current loopback service must not be treated as a production remote API.
- **Database-backed workflows** — `db/schema.ts`, `.openai/hosting.json`: Drizzle/D1 scaffolding has no product tables or configured binding. Document a data model only after persistence is actually wired.
