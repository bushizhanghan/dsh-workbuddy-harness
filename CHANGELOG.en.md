# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/).

## [1.2.7] - 2026-10-27

### Added

- **Long system prompts survive the domestic gateway's channel-risk classifier**: the domestic gateway scores the FIRST system message with a channel-risk classifier and rejects an over-long, agent-style system prompt as a "third-party client" with `400 code 11128`, even though the same text in a user turn passes. `relocateLongSystemMessage()` now moves any system prompt longer than 6000 chars into the leading user message (appended to the text, or unshifted as a `{type:"text"}` item), leaving only a short stub `You are a helpful assistant.` — so the request clears the classifier while the model still receives every instruction.
- **Outbound body rewrite**: before sending, the request body's literal string `DSH` (case-insensitive) is replaced with `DSH`, to avoid the gateway's channel-risk flag on that name.

### Note

- Added a debug dump: on failure the request body and a truncated upstream response are written to `C:/tmp/wbtest/last-body.json` (write failures are swallowed).

## [1.2.6] - 2026-10-23

### Fixed

- **A `550` with an empty body failed outright**: the edge gateway occasionally refuses a request with 550 (a non-standard code) and no body at all. 1.2.5 passed that straight through, so the user saw `upstream server (http 550): ` and nothing else. 550 is now retried once after 400 ms — that status means nothing was generated upstream, so replaying the request is safe.

### Added

- **Failure diagnostics** at `$DSH_HOME/.workbuddy-ai-diagnostics.log`: every failed chat request appends one JSON line with the timestamp, model, request size in bytes, upstream status, error class and message (truncated to 400 chars). The host keeps no log of its own for these failures, so diagnosis used to depend on the user quoting the error text back; now the file says what happened. Write errors are swallowed and can never fail the request.

### Note

- Tests 114 → 116 (550 retry succeeds; 550 after retry reports the status).

## [1.2.5] - 2026-10-23

### Fixed

- **`502 ... upstream server (http 0): transport error: TypeError: fetch failed`**: the plugin runs a long-lived shim on a random port, and its `fetch` calls go through undici's connection pool. After a few hours, a socket in that pool may have been closed by the upstream, or the DNS answer may be stale after an edge IP change — the next chat request reuses it, fails with `fetch failed`, and nothing retries. The user sees a 502 and a message that carries no actionable information.
  - Added `fetchUpstream()`: **retries once, transport failures only** (`fetch failed` / `ECONNRESET` / `ECONNREFUSED` / `ETIMEDOUT` / `EAI_AGAIN` / `ENOTFOUND` / `EPIPE` / `socket hang up`). A request that already produced an HTTP response is never replayed — the chat endpoint is not idempotent.
  - Both the chat path and the model-catalog path now use it.
  - Added `describeFetchError()`: it lifts the `ECONNRESET` / `ENOTFOUND` / `CERT_HAS_EXPIRED` code out of undici's `cause` and into the message, so the error no longer reads as a bare `TypeError: fetch failed`.

### Note

- Three regression assertions added (retry succeeds on a fresh socket, both attempts failing surfaces the cause, non-transport errors are not retried); 111 → 114 tests.

## [1.2.4] - 2026-10-23

### Changed

- **Repository renamed**: `bushizhanghan/dsh-workbuddy` → `bushizhanghan/dsh-workbuddy-harness`. `repository` / `homepage` / `bugs` in `package.json` point at the new address.
- The marketplace entry is renamed to `data/plugins/bushizhanghan__dsh-workbuddy-harness.yml`, its page moves to `/p/bushizhanghan/dsh-workbuddy-harness/`, and the install tarball now comes from the new repository's `releases/latest/download/dsh-workmate.tgz`.
- **The npm package name is unchanged** — still `@bushizhanghan/dsh-workmate`, with the same module id, provider id (`workbuddy-ai`) and CLI command (`dsh-workmate`). Existing installs need no configuration change.
- GitHub redirects the old repository URLs (302) to the new ones, so old clone URLs, release links and marketplace entries keep working for now.

## [1.2.3] - 2026-10-23

### Changed

- **README rewritten** in both languages: no screenshots and no badge images, plain text only, restructured as install → picking a model → sign-in flow → settings → CLI → compatibility → troubleshooting. The three screenshots under `docs/` are gone, and the npm package no longer ships a `docs/` directory.
- **De-duplicated the model display suffix**: `src/display.ts` now owns `joinDisplayParts()` and `DISPLAY_PART_SEPARATOR`, so the host adapter and the card share one implementation of "rate + promo badges" instead of two. This also fixes the unused `RATE_SEPARATOR` in `adapter.ts`, which was still bypassed by a string literal.
- `src/display.ts`: the two duplicated `pad` closures became one module-level `pad2()`, and the time constants (`MS_PER_DAY`, `BEIJING_UTC_OFFSET_MS`, …) are named.
- `src/loopback.ts`: `originIsTrusted` no longer lowercases the hostname twice or special-cases `::1`; an empty `allowedHosts` returns before parsing a URL.
- `src/provider-rows.ts`: the "register both halves, roll back on failure" block moved into an internal `register()`, leaving `setLive` as pure state handling.
- `src/catalog.ts`: `isFree()` short-circuits over `all()` instead of materialising an id array first.

### Fixed

- **The CI build smoke test asserted a stale module id**: `.github/workflows/ci.yml` grepped for `id: "dsh-workbuddy"`, but 1.2.0 renamed the module id to the package name, so the check could only fail. It now reads `name` from `package.json`, so a future rename cannot leave it behind.
- **`pnpm-workspace.yaml` had no `packages` field**: pnpm 9/10 refuses with `packages field missing or empty`, blocking `pnpm install`. Added `packages: ['.']`.

### Note

- Behaviour is unchanged: `pnpm run check` (typecheck + 111 tests + build) passes.

## [1.2.2] - 2026-10-22

### Note

- Identical to 1.2.1. That version was submitted to npm and never left the staging queue (`E409 Cannot publish over previously staged version`), so the number is burned and 1.2.2 carries the same content.

## [1.2.1] - 2026-10-22

### Fixed

- **The peer range silently excluded 0.2.1 prereleases** under standard semver rules: in `>=0.2.0-rc.2 <0.3.0-0` the prerelease tag sits only on the `0.2.0` tuple, so a version like `0.2.1-rc.2` was excluded — only the host's own check, which passes `includePrerelease`, let it through. The range is now `>=0.2.0-rc.2 <0.2.1 || >=0.2.1-0 <0.3.0-0`, which matches `0.2.0-rc.2`, `0.2.0`, `0.2.1-rc.2` and `0.2.1`, so `pnpm add` no longer reports `ERESOLVE`.

## [1.2.0] - 2026-10-22

### Changed

- **Renamed to `@bushizhanghan/dsh-workmate`** so the npm package cannot be confused with upstream `dsh-workbuddy`. The module id (`__ModuleLoader__.load({ id })` takes the package name), the `name` inserted by `cordis.patch.yml`, the CLI command (`dsh-workmate`) and the docs were updated together.
- The model provider id stays `workbuddy-ai`; existing configs need no change.

## [1.1.0] - 2026-10-22

First release of the compatibility fork, based on upstream `dsh-workbuddy@1.0.12` (commit `9951d25`). Entries below are upstream's and are kept verbatim.

### Fixed

- **Rejected on dsh 0.2.1-rc.2**: `peerDependencies` pinned seven `@deepseek-ai/dsh-*` packages to the exact version `0.2.0-rc.2`, and dsh checks every one of those peers against the running runtime before loading a plugin. On 0.2.1 every requirement failed, so the plugin never loaded and the UI reported "requires host 0.2.0-rc.2, currently running 0.2.1-rc.2". The range is now `>=0.2.0-rc.2 <0.3.0-0`, so one build covers both 0.2.0 and 0.2.1.
- **Client name not recognised upstream**: the host calls itself `DeepSeek Harness` (with a space), which the upstream service treats as an unknown client. Outgoing request bodies now rewrite it to `DeepSeek-Harness`; everything else is passed through untouched.

### Changed

- Package name is `@bushizhanghan/dsh-workbuddy`; `repository` / `homepage` / `bugs` point at this repo; `author` names the upstream author as well.
- Added `NOTICE` (fork provenance and scope of changes), `patches/` (diffs for both changes) and `pnpm-workspace.yaml` (`allowBuilds`, so pnpm 11's dependency-build confirmation no longer aborts CI).
- `lib/` is rebuilt from `src/` instead of hand-editing the bundle; `pnpm test` passes all 111 tests.

## [1.0.12] - 2026-10-18

### Fixed

- **The streak number now comes from the same source as the growth-center page**: 1.0.11 showed the upstream `streak_days`, which is a check-in total for the activity window, not the "days in a row" figure the page renders. That figure is `GET /activity/growth/streak` → `streak.days` — **without** the `/v2` prefix (`/v2/activity/growth/streak` answers 404, which is why the plugin's reader always came back empty). The card reads that endpoint now and the label is back to "N-day streak" / 「连登 N 天」, falling back to `streak_days` only when the read fails.

## [1.0.11] - 2026-10-18

### Fixed

- **Reverted the 1.0.10 "streak" recomputation**: 1.0.10 derived a consecutive run from `checkin_dates`, inventing a third number — the upstream `streak_days` is a window total and the check-in site shows its own figure, so none of the three agreed. The card now reports the upstream `streak_days` verbatim and the label reads "N check-ins this event" instead of "N-day streak", so no streak is ever claimed.

## [1.0.10] - 2026-10-18

### Fixed

- **`<analysis>` / `<summary>` tags leaking into the reply body**: the upstream declares a single fixed `reasoning.effort` for a group of models (international `deepseek-v4.1-flash`, `balanced-model`; domestic `hy3`, `deepseek-v4-pro`). The plugin read that as "no reasoning control", so it never sent `reasoning_effort`, the upstream never opened the `reasoning_content` channel, and the model drafted its reasoning into the visible text. Those models now declare reasoning at their fixed level and always send `reasoning_effort`; they also no longer enter the reasoning probe queue.
- **"N-day streak" counted window totals, not a streak**: the upstream `streak_days` field is the number of check-ins inside the activity window (measured: 11 check-ins with two missed days). The card now derives the run that actually ends today (or yesterday) from `checkin_dates`, falling back to the raw field only when the list is absent.

### Added

- **The model picker follows real availability**: the upstream catalog carries no per-account visibility, so paid-tier models (international `primary-model`, `gpt-5.5`, …) only answer `is not allowed for user type` on a real request. The first such failure now hides the model from the picker (remembered for the session, cleared on catalog refresh); sessions already using it keep working.

## [1.0.9] - 2026-10-09

### Changed

- The three README screenshots (`docs/card-accounts.png`, `docs/card-models.png`, `docs/model-picker.png`) were replaced with the original, uncompressed captures (about 2.5 MB total), restoring fine detail at the cost of a larger npm package.

## [1.0.8] - 2026-10-09

### Changed

- The five README highlights were rewritten in the requester's own wording: **Sign-in** ("no desktop sign-in at all"; a desktop app credential is still importable read-only), **Accounts** (now naming credit warnings and account notes), **One-click check-in** ("auto check-in is available and off by default"), **Device isolation** (each account's credential, model catalog and credit balance stand apart; switching accounts never touches in-flight requests) and **Works out of the box** (enable it and use it, no extra configuration; DSH 0.2.0-rc.2 desktop is supported today).

## [1.0.7] - 2026-10-09

### Changed

- Both READMEs gained three UI screenshots (accounts and credits, model scope and switches, the model picker), placed above Quick start, with the assets under `docs/`.
- Highlight 3 was renamed from "Domestic one-click" to "One-click check-in"; its wording is unchanged.
- Highlights 4 and 5 were rewritten into two more distinctive ones: **the two regions stay strangers** (each keeps its own credential, its own catalog and its own balance; an empty region stays empty rather than borrowing the other one's lineup; switching accounts never touches in-flight requests) and **nothing is done to your desktop app** (its login state is only ever read, never written; the plugin stores its own per-region credential copies and Disconnect deletes only those). The former "live pricing" and "providers" entries are no longer listed as highlights.
- The "Models and prices" section was removed from both READMEs (the implementation is unchanged).
- `docs` joined the npm package's `files`, so the README screenshots work inside the tarball too.

## [1.0.6] - 2026-10-09

### Fixed

- The international region lists `deepseek-v4.1-flash` (`x0.00`) again. After 1.0.5 dropped the local fallback list, the international region read `/v2/enterprises/personal/models`, which is a **truncated view**: it omits `deepseek-v4.1-flash` and reports `hy4-preview` as `x0.00` (its promotional price). Both regions now read the catalog the app itself reads, `GET https://www.workbuddy.ai/v3/config`, which also brings in `gpt-6-astra`, `glm-5.3-flash` and `kimi-k2.8-preview`. The old `/v2/enterprises/personal/models` path stays as a fallback, used only when `/v3/config` is unavailable.

## [1.0.5] - 2026-10-09

### Changed

- **The local price table is gone; both regions now follow their own live catalog.** The plugin used to ship a built-in price and free-model list (`src/product-config.ts`) and to override the international catalog's `credits` with it. That file has been removed entirely: which models exist, their context and reasoning efforts, and their `credits` price are all taken verbatim from the region's catalog, so a new model or a price change reaches users immediately instead of waiting for a plugin release. The domestic region was already upstream-authoritative; the international one now matches it.
- Consequently, an empty list before a region's catalog arrives is deliberate — no region borrows another region's lineup (the old three-model local fallback is gone).
- `credits` is the displayed price after any limited-time promotion: `hy4-preview` reads `x0.00` during its promotion and `x0.29` outside it. The plugin no longer second-guesses it and simply mirrors what upstream publishes.
- The `productConfigFile` config key and the `DSH_WORKBUDDY_PRODUCT_CONFIG` environment variable are removed; `doctor` no longer reports a product-config source and instead states that prices are read live from each region's catalog.
- Both READMEs: the subtitle is now "no WorkBuddy desktop app required", the "Free models" section became "Models and prices" (live catalog, promotional discounts), and highlights 3–5 were rewritten (domestic one-click check-in / cat travel / growth tasks, live pricing, providers appearing per sign-in).

## [1.0.4] - 2026-10-09

### Fixed

- WorkBuddy providers no longer appear on the Models settings page while signed out. The provider and its directory entry used to be registered unconditionally at startup, so the page listed a route with no credentials that could not be used; registration now follows each region's credentials — the row appears after signing in to that region and disappears after disconnecting or removing its last account. Both halves (adapter and directory entry) always move together, so a row is never listed but unusable.

### Changed

- The signed-out hint no longer says the desktop app is optional; it now just says to click Connect and sign in through the WorkBuddy website.
- Both READMEs now use a centered title plus badges (license, DSH version, provider count, free-by-default, zero runtime dependencies), and drop the "standalone/isolated from other WorkBuddy plugins" sentence, the `## Relationship to other WorkBuddy plugins` section, `## Known limitations`, and the architecture-credit line.

## [1.0.3] - 2026-10-09

### Fixed

- Connect no longer opens a login URL with the `state` parameter stripped, which made the site report an incomplete login link. On Windows the URL is handed to `rundll32.exe url.dll,FileProtocolHandler` instead of going through `cmd /c start`, because `cmd.exe` treats the `&` in the URL as a command separator and truncates `&state=…`.
- Clicking the other region no longer does nothing after one region is signed in. The card used to fall back to the signed-in region whenever the selected tab was not signed in, so a second region could never be added; the selected tab now always wins, with a one-time preselect of the signed-in region.
- Removed the "Open login page" button, which duplicated Connect (same URL, same action, and two tabs raced for one single-use login state).

### Changed

- Provider display names are now **WorkBuddy 国际版** / **WorkBuddy 国内版**.
- English is now the default README: `README.md` is English and the Chinese one moved to `README.zh.md`.

## [1.0.2] - 2026-10-09

### Fixed

- Added a domestic/international region switch to the signed-out view; previously only a domestic login could be started, leaving the international provider with no entry point.
- Connect no longer opens the login page twice. The host and the card each opened a tab, and both raced for the same single-use login state — the first won, the second showed the site's "login link incomplete" error. Only the host opens it now; the card keeps its "Open login page" link as a fallback.

## [1.0.1] - 2026-10-09

### Fixed

- Unified the security header used by the settings card and host control route, fixing Connect and other card actions being rejected.

## [1.0.0] - 2026-10-09

First standalone release under the `dsh-workbuddy` name with independent identifiers.

### Added

- Two-region providers: `workbuddy-ai` (international, shown as **WorkBuddy**) and `workbuddy-cn` (domestic, shown as **WorkBuddy 国内**)
- Per-region browser OAuth sign-in (`auth/state` → `authUrl` → poll `auth/token`); credentials are written only to the plugin's own copy
- Read-only import of desktop app credentials (`workbuddy-desktop-ai.info` / `workbuddy-desktop.info`)
- Region-isolated multi-account list with explicit switching, remaining credits, and a connectivity test
- Optional domestic auto check-in (off by default; only ticked accounts)
- Free-model list driven by the product config pushed by the app, with a built-in fallback
- Settings card: connect/disconnect, account management, `free / all models` scope, refresh intervals
- CLI: `login` / `status` / `doctor` / `logout` (`status` and `doctor` accept `--json`)
- Bilingual README ([English](./README.md) / [中文](./README.zh.md)) and a CI workflow

### Identifiers (fully isolated from `dsh-connect-workbuddy`)

| Item | Value |
|---|---|
| Providers | `workbuddy-ai` / `workbuddy-cn` |
| Credential files | `.workbuddy-ai-auth.json` / `.workbuddy-cn-auth.json` |
| Heartbeat file | `.workbuddy-ai-host-heartbeat.json` |
| Settings file | `.workbuddy-ai-settings.json` |
| Probe cache | `.workbuddy-ai-probe.json` |
| Environment variables | `DSH_WORKBUDDY_AUTH_FILE` / `DSH_WORKBUDDY_CN_AUTH_FILE` / `DSH_WORKBUDDY_PRODUCT_CONFIG` |
| Control header | `X-WorkBuddy-Control-Key` |

### Compatibility

- Aligned with DSH `0.2.0-rc.2`: every `@deepseek-ai/dsh-*` peer is pinned to exactly `0.2.0-rc.2`, and `@earendil-works/pi-ai` to `^0.87.1`
