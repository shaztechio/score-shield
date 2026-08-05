---
type: Operations Runbook
title: Local operations and integrations
description: Runbook for Score Shield setup, local process orchestration, secret boundaries, media artifacts, external services, hosted UI packaging, and documentation automation.
resource: /scripts/setup.mjs
tags: [operations, setup, integrations, openai, cloudflare, github-actions]
openwiki:
  roles: [operations, integration]
  change_kinds: [configuration, cleanup, deployment]
  source_paths: [scripts/setup.mjs, scripts/dev.mjs, scripts/clean-artifacts.mjs, .env.example, .github/workflows/openwiki-update.yml]
  symbols: [cleanJobArtifacts, validateProcessorEnvironment]
  test_paths: [tests/setup.test.mjs, tests/clean-artifacts.test.mjs, tests/server-startup.test.mjs]
  invariants: [Secrets stay in .env and outside browser configuration, Cleanup deletes UUID job directories only and preserves cache, Local processor requires an OpenAI key before listening.]
  validation_commands: [npm run setup:check, npm run test:unit, npm test]
---

# Local operations and integrations

Real Score Shield processing is intentionally local: it requires native media tools, writable media artifacts, and a server-side OpenAI credential. [Architecture overview](architecture/overview.md) explains the two-runtime boundary, while [Score timeline workflow](workflows/score-timeline.md) explains how those tools produce reusable metadata.

## Setup and local lifecycle

| Command | Meaning | Side effects |
| --- | --- | --- |
| `npm run setup` | Runs `scripts/setup.mjs`; verifies Node 22.13+, installs npm dependencies, finds or installs media tools, creates `.env` when absent, and verifies tools. | Can install packages and invoke `sudo` on non-Windows Linux. |
| `npm run setup:check` | Runs setup with `--check`. | Read-only prerequisite verification. |
| `node scripts/setup.mjs --dry-run` | Shows the setup plan. | Does not install or create `.env`. |
| `npm run dev` | Runs `scripts/dev.mjs`. | Starts processor and UI; signals stop both child processes. |
| `npm run processor` | Starts `server/index.mjs` with `.env` if present. | Refuses to listen if `OPENAI_API_KEY` is missing/blank. |
| `npm run dev:web` | Starts Vinext development server. | UI only; demo can still run. |
| `npm run artifacts:clean` | Runs `cleanJobArtifacts`. | Deletes UUID-named per-job artifacts; preserves cache. |

Setup needs `ffmpeg`, `ffprobe`, and `yt-dlp`. It uses Homebrew on macOS; Homebrew, apt, dnf, or pacman on Linux; and Winget or Chocolatey on Windows. Installer calls use argument arrays and `shell: false`. Existing `.env` is left untouched. `tests/setup.test.mjs` covers install-plan construction, not real installation—use `npm run setup:check` when changing machine detection or prerequisite behavior.

## Configuration and secret boundary

Read `.env.example`, not local `.env`. Keep `OPENAI_API_KEY` out of `NEXT_PUBLIC_*`, React state, fixtures, logs, and commits. `validateProcessorEnvironment` enforces this server-side prerequisite before the processor accepts work; the demo bypasses the processor rather than weakening this check.

| Variable | Default | Consumer |
| --- | --- | --- |
| `OPENAI_API_KEY` | none | Required at processor startup and by OpenAI frame analysis. |
| `OPENAI_MODEL` | `gpt-5.6` | Responses model in `server/pipeline.mjs`. |
| `FRAME_INTERVAL_SECONDS` | `10` | Processor fallback interval; request value overrides it. |
| `PROCESSOR_PORT` | `8787` | Loopback listener port. |
| `NEXT_PUBLIC_PROCESSOR_URL` | `http://localhost:8787` | Browser processor base URL. It is public configuration, never a secret. |
| `YTDLP_PATH`, `FFMPEG_PATH`, `FFPROBE_PATH` | executable on `PATH` | Tool overrides for the processor. |
| `ARTIFACTS_DIR` | `artifacts` | Root for job output and the source cache. |
| `WEB_ORIGIN` | `http://localhost:3000` | Single CORS origin emitted by processor routes. |

The processor only accepts HTTPS `youtube.com`, `www.youtube.com`, and `youtu.be` URLs. Its local bind, CORS, URL allowlist, bounded request body, and server-side key are one combined boundary—not a design for safely pointing `NEXT_PUBLIC_PROCESSOR_URL` at a public server.

## Artifact retention and cleanup

Job-specific frames, observations, manifest, and VTT live under `artifacts/<uuid>/`. Cached authorized source media lives under `artifacts/cache/youtube/<cache-key>/source.*` and is reused by equivalent YouTube URLs; see [Sampling and source cache](workflows/score-timeline.md#sampling-and-source-cache).

`cleanJobArtifacts` reads the immediate artifact-root entries and removes only directories that match a UUID pattern. It preserves `cache/` and unrelated directories and succeeds if the root is absent. The **removes isolated job directories while preserving the YouTube cache** test in `tests/clean-artifacts.test.mjs` is the focused regression test. Stop processing before cleanup; it is not a retention scheduler and does not evict cached media. Do not use a wildcard or recursive deletion as a substitute.

## External integrations and hosted build

- **YouTube:** `yt-dlp` downloads an authorized no-playlist source copy; the UI embeds YouTube with `enablejsapi=1` only to follow playback time. The UI does not request the video title, though provider chrome can still disclose it after the cover is removed.
- **FFmpeg/FFprobe:** probe duration and extract frames locally. Their absolute timestamps drive the score timeline.
- **OpenAI:** the JavaScript SDK sends sampled JPEGs for candidate observations. Zod validation and reconciliation remain authoritative; raw model responses are not shown in UI or logs.
- **Cloudflare/Vinext:** `worker/index.ts` serves the hosted UI/demo and image transform route. `vite.config.ts` and `build/sites-vite-plugin.ts` create the build/package path. Run `npm test` after changing this surface because its rendered HTML test executes the built Worker output.
- **Drizzle/D1:** `db/index.ts` can open a `DB` binding, but `db/schema.ts` contains no product tables and `.openai/hosting.json` binds neither D1 nor R2. `examples/d1/` is illustrative only.

## Documentation automation

`.github/workflows/openwiki-update.yml` runs daily and manually. It installs OpenWiki, runs `openwiki code --update --print`, and creates or updates an `openwiki/update` pull request. The workflow uses `OPENAI_API_KEY` with the OpenAI provider and `gpt-5.6-terra`; its PR `add-paths` covers `openwiki` and `AGENTS.md` only. It does not add the workflow file itself to the generated PR.

Use `npm run wiki:init` or `npm run wiki:update` for local documentation work. Generated wiki content belongs in `openwiki/`; source, tests, README, and `AGENTS.md` remain the evidence to change first.
