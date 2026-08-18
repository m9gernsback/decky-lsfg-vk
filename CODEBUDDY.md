# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Project Overview

Decky Loader plugin for Steam Deck that installs and configures **lsfg-vk** (Lossless Scaling frame generation Vulkan layer). Standard Decky plugin structure: a **Python backend** (`py_modules/lsfg_vk/`, entry point `main.py`) and a **React/TypeScript frontend** (`src/`) that communicate via Decky's RPC bridge. Build output is `out/Decky LSFG-VK.zip`.

## Commands

- `pnpm install` — install frontend dependencies
- `just build` — full plugin build: regenerates the schema, wipes `node_modules`, then runs `.vscode/build.sh` which invokes the Decky CLI (`./cli/decky plugin build`). Requires the decky CLI checked out at `./cli` and sudo.
- `pnpm build` / `pnpm watch` — frontend bundle only (rollup, config in `rollup.config.js`)
- `just generate-schema` (or `python3 scripts/generate_ts_schema.py`) — regenerate derived schema files after editing `shared_config.py` (see below)
- `scripts/build_i18n_json.sh` — merge `defaults/i18n/*.json` into `src/i18n/languages.json` (requires `jq`)
- `just test` — **not a test suite**; scp's the built zip to a Steam Deck at a hardcoded IP (`deck@192.168.0.6`) — adjust for your own device
- `just watch` — tail `journalctl` on the Deck over SSH (backend logs); `just cef` — tail `~/.local/share/Steam/logs/cef_log.txt` (frontend logs)
- `just clean` — remove `node_modules`, `dist`, `/tmp/decky`

There are no unit tests or linters configured; `npm test` is a stub that exits 1.

## Config Schema: Single Source of Truth

`shared_config.py` (repo root) defines `CONFIG_SCHEMA_DEF` — the canonical schema for every user-facing setting. Each field has a `location`:

- `"toml"` — written to `~/.config/lsfg-vk/conf.toml` (consumed by the lsfg-vk Vulkan layer)
- `"script"` — rendered as environment variables into the generated `~/lsfg` launch script (e.g. `dxvk_frame_rate`, `enable_wow64`, `mangohud_workaround`)

`scripts/generate_ts_schema.py` derives **both** `src/config/generatedConfigSchema.ts` and (via `scripts/generate_python_boilerplate.py`) `py_modules/lsfg_vk/config_schema_generated.py`. **When adding or changing a config field, edit `shared_config.py` and run `just generate-schema` — never edit the generated files directly.**

## Backend Architecture (`py_modules/lsfg_vk/`)

`main.py` re-exports `Plugin` from the package. `plugin.py` defines the `Plugin` class, which is a thin async facade over four services (all extending `base_service.BaseService`):

- `installation.py` — extracts the lsfg-vk zip to `~/.local`, installs the Vulkan layer JSON, generates the `~/lsfg` launch script
- `configuration.py` — reads/writes the TOML config **and** parses/regenerates the launch script; a full config read merges both sources (`merge_config_with_script`). Supports multiple named profiles.
- `dll_detection.py` — locates `Lossless.dll` from the Steam install of Lossless Scaling
- `flatpak_service.py` — installs per-runtime Flatpak Vulkan layer extensions and manages per-app overrides

Every `async` method on `Plugin` is callable from the frontend via Decky RPC. `plugin.py` also implements the Decky lifecycle hooks: `_main`, `_unload`, `_uninstall` (cleans up lsfg-vk files and flatpak extensions), `_migration`. Filesystem paths and env-var names live in `constants.py`.

## Frontend Architecture (`src/`)

- `index.tsx` — `definePlugin` entry, renders `components/Content.tsx`
- `api/lsfgApi.ts` — typed wrappers around backend RPC calls (`@decky/api` `callable`)
- `hooks/` — React hooks that own state and call the API (`useLsfgHooks`, `useInstallationActions`, `useProfileManagement`)
- `components/` — UI sections (config controls, profile management, flatpak modal, clipboard buttons for the `~/lsfg %command%` launch option)
- `config/configSchema.ts` + `generatedConfigSchema.ts` — frontend-side field metadata (generated file mirrors the Python schema)

## Build-Time Binaries

`package.json`'s `remote_binary` array declares binaries the Decky build downloads and bundles (lsfg-vk zip, three Flatpak runtime extensions, an arm64 `.so`), each pinned with a `sha256hash`. Updating lsfg-vk versions means updating these URLs and hashes.

## i18n

Translation sources live in `defaults/i18n/*.json` and are merged into `src/i18n/languages.json` by `scripts/build_i18n_json.sh`; `src/i18n/i18n.ts` consumes the merged file.
