# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ezBookkeeping is a self-hosted personal finance app shipped as a **single Go binary** that serves both a REST API and an embedded SPA. The frontend is actually **two separate Vue 3 SPAs in one repo** — a desktop app (Vuetify) and a mobile app (Framework7) — built from the same `src/` tree, sharing models/stores/lib but with their own entry HTML, router, and view trees.

## Commands

### Building (full pipeline)
The canonical build is `./build.sh <type>` (or `build.bat` / `build.ps1` on Windows). It runs lint + tests before building unless skipped.

```bash
./build.sh backend       # Go binary only → ./ezbookkeeping
./build.sh frontend      # Vite build → ./dist
./build.sh package -o ezbookkeeping.tar.gz   # full tarball (backend + frontend + conf + templates)
./build.sh docker        # docker image
./build.sh <type> --no-lint --no-test        # skip checks
SKIP_TESTS=TestPattern ./build.sh backend    # skip specific Go tests by pattern
RELEASE_BUILD=1 ./build.sh ...               # release build (omits build timestamp)
```

### Frontend dev loop
```bash
npm install
npm run serve          # Vite dev server on :8081, proxies /api, /mcp, /oauth2, /pictures, etc. to backend on :8080
npm run build          # production build → ./dist
npm run lint           # vue-tsc --noEmit + eslint --fix (TS + Vue)
npm run test           # vitest run (one-shot)
npx vitest run path/to/file.test.ts          # single test file
npx vitest run -t "test name substring"      # filter by test name
```

The Vite dev server (`:8081`) **requires the Go backend running on `:8080`** — it proxies all API/auth/asset routes there. To run the backend in dev: `go run ezbookkeeping.go server run` from the repo root (needs `conf/ezbookkeeping.ini` and `public/` — for full UI, build frontend once so `dist/` exists, or symlink `dist` → `public`).

### Backend dev/test
```bash
go vet ./...
go test ./...                                # all backend tests
go test ./pkg/services/... -run TestName     # single test or package
go test ./pkg/models -v
go run ezbookkeeping.go server run --conf-path conf/ezbookkeeping.ini
go run ezbookkeeping.go --help               # see all subcommands: server, database, user-data, cron, security, utility
```

The binary is a multi-command CLI (urfave/cli) — `server run`, `database upgrade`, `user-data export`, `cron <job>`, etc. See `cmd/*.go` for command definitions.

## Architecture

### Backend (`pkg/`, `cmd/`, `ezbookkeeping.go`)

Layered Go service, Gin HTTP framework, xorm ORM:

- `ezbookkeeping.go` — main; registers CLI subcommands from `cmd/`.
- `cmd/` — CLI command definitions (one file per top-level subcommand). `webserver.go` is where the entire HTTP route tree is built — **this is the map of the API surface**; every API handler is registered here.
- `pkg/api/` — HTTP handlers. One file per resource (`transactions.go`, `accounts.go`, …). Handlers are thin: parse → call service → return.
- `pkg/services/` — business logic and DB access (via xorm sessions). Tests live next to the code (`*_test.go`).
- `pkg/models/` — DTOs and DB row structs. Many models have `_test.go` files exercising validation/conversion.
- `pkg/datastore/` — xorm engine wrapper supporting SQLite / MySQL / Postgres (chosen at runtime from `conf/ezbookkeeping.ini`).
- `pkg/middlewares/` — Gin middlewares (JWT auth variants, request id, request log, IP limits for MCP/API tokens).
- `pkg/core/` — cross-cutting primitives: request `Context` wrappers (`context_web.go`, `context_cli.go`, `context_cron.go`), JWT claims, JSON-RPC helpers, calendar/currency/datetime/fiscal-year logic. Handlers receive a `*core.WebContext`, not `*gin.Context`.
- `pkg/converters/` — import/export adapters, one subpackage per format (csv, ofx, qif, beancount, gnucash, firefly III, camt, mt940, alipay, wechat, …). All implement the `transaction_data_converters.go` interface.
- `pkg/mcp/` — Model Context Protocol server (JSON-RPC over HTTP at `/mcp`). One file per tool handler. Gated by `[mcp] enable_mcp` in the ini.
- `pkg/llm/` — pluggable LLM provider container for receipt image recognition.
- `pkg/auth/oauth2/`, `pkg/exchangerates/`, `pkg/storage/`, `pkg/mail/`, `pkg/cron/`, `pkg/locales/` — self-explanatory subsystems.
- `conf/ezbookkeeping.ini` — single config file; loaded via `--conf-path` flag or default lookup. Holds DB type, server protocol, MCP toggle, OAuth2 providers, exchange rate sources, etc.
- `templates/` — server-side templates (emails, etc.).

### Frontend (`src/`)

Two SPAs sharing a single `src/` tree. Vite builds three HTML entries from `src/`:

| Entry | Output | Stack | Router |
| --- | --- | --- | --- |
| `index.html` + `index-main.ts` | landing/redirect | minimal | — |
| `desktop.html` + `desktop-main.ts` (`DesktopApp.vue`) | desktop SPA | Vue 3 + Vuetify + vue-router | `src/router/desktop.ts` |
| `mobile.html` + `mobile-main.ts` (`MobileApp.vue`) | mobile SPA / PWA | Vue 3 + Framework7-Vue | `src/router/mobile.ts` |

Shared code (do not duplicate across desktop/mobile):
- `src/models/`, `src/stores/` (Pinia), `src/core/`, `src/consts/` — domain layer.
- `src/lib/` — utilities (currency, datetime, statistics, math, evaluator, qrcode, webauthn, …). Tests in `src/lib/__tests__/`.
- `src/locales/*.json` — i18n bundles (one per language). Backend has its own translations under `pkg/locales/`.
- `src/components/{base,common}` — cross-platform components.
- `src/components/desktop`, `src/components/mobile`, `src/views/desktop`, `src/views/mobile` — platform-specific UI. Do not import desktop-only code from mobile views or vice versa; the Vite `manualChunks` config in `vite.config.ts` splits them into separate vendor bundles.
- `@` is aliased to `src/` in both `vite.config.ts` and `vitest.config.ts`.

PWA support (mobile) is wired via `vite-plugin-pwa` with a custom service worker at `src/sw.ts` and a Web Share Target endpoint at `/__share__image__` for sharing receipt images into the app.

### Skills (`skills/ezbookkeeping/`)
Standalone Bash/PowerShell scripts (`ebktools.sh` / `ebktools.ps1`) that wrap the REST API for shell/agent use. Self-contained — not part of the build.

## Conventions worth knowing

- **i18n is mandatory.** Any user-facing string goes through `vue-i18n` (frontend) or `pkg/locales` (backend). Don't hardcode English in views.
- **Backend tests are co-located** (`pkg/foo/bar_test.go`); frontend tests live under `src/lib/__tests__/` or alongside as `*.test.ts`. There is no top-level `tests/` dir.
- **Lint is strict in CI** (`build.sh` runs `go vet` and `npm run lint` which includes `vue-tsc --noEmit`). A type error in any `.vue`/`.ts` file fails the frontend build, not just the lint step (see `Checker({ vueTsc: true })` in `vite.config.ts`).
- **Database migrations** are applied automatically when `[database] auto_update_database_table_structure = true` in the ini. Schema is defined via xorm struct tags on models in `pkg/models/`, not separate migration files.
- **Adding an HTTP endpoint** requires three edits: handler in `pkg/api/<resource>.go`, service method in `pkg/services/<resource>.go`, and route registration in `cmd/webserver.go`. Do not skip the route registration — there is no auto-discovery.
- **Adding an MCP tool** requires a handler file in `pkg/mcp/` and registration in the `tools/call` map in `cmd/webserver.go`.
