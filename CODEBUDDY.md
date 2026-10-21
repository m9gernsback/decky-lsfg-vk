# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Project Overview

Decky Loader plugin for Steam Deck that installs and configures **lsfg-vk** (Lossless Scaling frame generation Vulkan layer). Standard Decky plugin structure: a **Python backend** (`py_modules/lsfg_vk/`, entry point `main.py` at repo root) and a **React/TypeScript frontend** (`src/`) that communicate via Decky's RPC bridge.

Targets **lsfg-vk v2** (currently bundling `lsfg-vk-2.0.0`). The v1 config format and the old mailbox present mode are gone; see "Config Schema" below.

## Commands

- `pnpm install` — install frontend dependencies
- `pnpm build` / `just build` — frontend bundle only (rollup, config in `rollup.config.js`)
- `pnpm test` / `just test` — **real test suite**: `node --experimental-strip-types --test` over `tests/*.test.ts` plus `python3.12 -m unittest` over `tests/test_*.py`
- `just deploy` — `scripts/deploy-to-deck.sh`; scp's the plugin to a Steam Deck (hardcoded `deck@192.168.0.241` — adjust for your own device; reads `.env`, uses `expect` when `DECK_PASSWORD` is set)
- `just watch` — tail `journalctl` on the Deck over SSH (backend logs); `just cef` — tail local `~/.local/share/Steam/logs/cef_log.txt` (frontend logs)
- `just clean` — remove `node_modules`, `dist`, `/tmp/decky`
- `scripts/build_i18n_json.sh` — merge `defaults/i18n/*.json` into `src/i18n/languages.json` (requires `jq`)

There is no linter configured.

## Config Schema (lsfg-vk v2)

The backend schema is the single source of truth: `py_modules/lsfg_vk/config_schema.py` (`PROFILE_DEFAULTS` / `GLOBAL_DEFAULTS` / `ConfigurationManager`). The frontend mirror `src/config/generatedConfigSchema.ts` (re-exported via `src/config/configSchema.ts`) is **synchronized by hand** — when adding or changing a field, update both sides plus the i18n strings.

The on-disk config is `~/.config/lsfg-vk/conf.toml` in the v2 format (`version = 2`, `[global]` with `dll` / `allow_fp16`, then `[[profile]]` sections). Profile fields:

- `pacing_mode` — locked to `"vsync"`; validation rejects anything else (mailbox mode was removed upstream)
- `multiplier`, `flow_scale` (0.25–1.0), `performance_mode`
- `override_present_mode` (default `true`), `preserve_swapchain_image_count` (default `false`)
- `active_in` — list of AppID/executable identifiers scoping a profile

v1 configs are treated as discardable legacy (`is_discardable_legacy_version`) and reset rather than migrated.

## Backend Architecture (`py_modules/lsfg_vk/`)

Root `main.py` re-exports `Plugin` from the package. `plugin.py` defines the `Plugin` class, a thin async facade over services (all extending `base_service.BaseService`):

- `runtime_service.py` — runtime/environment detection
- `steam_service.py` — Steam install/shortcut integration
- `installation.py` — extracts the lsfg-vk tarball to `~/.local`, installs the Vulkan layer JSON
- `configuration.py` — reads/writes the v2 TOML config; supports multiple named profiles
- `flatpak_service.py` — installs per-runtime Flatpak Vulkan layer extensions
- `flatpak_profile_service.py` — per-app Flatpak overrides (env vars, `DXVK_CONFIG`)
- `wrapper_service.py` — per-app workarounds and the generated launch wrapper (see below)

Every `async` method on `Plugin` is callable from the frontend via Decky RPC. `plugin.py` also implements the Decky lifecycle hooks: `_main`, `_unload`, `_uninstall`, `_migration`. Filesystem paths and env-var names live in `constants.py`.

## Launch Wrapper & Per-App Workarounds

`wrapper_service.py` generates the `~/.lsfg` wrapper script (format marker `# lsfg-vk-wrapper-format: 2`) that users put in Steam launch options (`~/.lsfg %command%`). Per-app state lives in `~/.config/lsfg-vk/workarounds.json` (versioned document, currently version 2). Workaround fields:

- `dxvkFrameRate` — Base FPS Cap, integer 0–60 (0 = off). Rendered as `DXVK_CONFIG="dxvk.maxFrameRate = N"`, **not** the legacy `DXVK_FRAME_RATE` env var (the wrapper strips that legacy key)
- Boolean toggles: `disableGamescopeWsi`, `disableHdr`, `disableSteamdeckMode`, `disableVkbasalt`, `enableZink`

The frontend has matching launch-option cleanup logic in `src/utils/steamLaunchOptions.ts`.

## Frontend Architecture (`src/`)

- `index.tsx` — `definePlugin` entry, renders `components/Content.tsx`
- `api/lsfgApi.ts` — typed wrappers around backend RPC calls (`@decky/api` `callable`)
- `hooks/` — `useLsfgHooks`, `useGameConfiguration`, `usePerAppWorkarounds`, `useFlatpakConfiguration`
- `components/` — tab-based UI: `ConfigurationTab`, `SettingsTab`, `NowPlayingTab`, `FlatpakTab` (+ `FlatpakNowPlayingTab`), with shared sections like `ConfigurationSection` (global toggles) and `WorkaroundsSection` / `FlatpakWorkaroundsSection` (per-app workarounds incl. the Base FPS Cap slider)
- `utils/` — `steamLaunchOptions.ts` (parse/clean Steam launch options), `gameTargets.ts`, `nowPlaying.ts`, `toastUtils.ts`

Note: `ConfigurationSection.tsx` labels are currently hardcoded English; other sections use `t()` i18n keys.

## Build-Time Binaries

`package.json`'s `remote_binary` array declares binaries the Decky build downloads and bundles — currently just `lsfg-vk-2.0.0.tar.xz` from `builds.lsfg-vk.dev`, pinned with a `sha256hash`. Updating lsfg-vk versions means updating the URL and hash.

## i18n

Translation sources live in `defaults/i18n/*.json` (`template.json` = English, plus `ja`/`ko`, `language_metadata.json`, `steam_language_map.json`) and are merged into `src/i18n/languages.json` by `scripts/build_i18n_json.sh`; `src/i18n/i18n.ts` consumes the merged file. **Regenerate the merged file after editing sources.** Watch for stale keys — e.g. `CONFIG_PRESENT_MODE_*` (mailbox) strings remain in the sources but are no longer referenced by code.

## Tests

Tests live in `tests/`: TypeScript tests (`steamLaunchOptions.test.ts`, `nowPlaying.test.ts`, `gameTargets.test.ts`) run under `node --experimental-strip-types`; Python tests (`test_wrapper_service.py`, `test_configuration_profiles.py`, `test_installation_*.py`, `test_plugin_migration.py`, `test_steam_service.py`, `test_flatpak_*.py`) run under `python3.12 -m unittest`. Both are wired into `pnpm test`.
