# Changelog

All public, user-visible changes to Fleex Optim Route Planner. The full internal changelog lives in the source repository.

Format: [Keep a Changelog](https://keepachangelog.com/). Versioning: [SemVer](https://semver.org/).

---

## [0.0.6] — 2026-09-10 — Route calculation now uses the environment you signed in to (OP-5)

> **Coordinated release.** This version requires the matching VRP solver deployment. The `/optimize` request contract changed and is **not** backwards compatible — an older solver rejects this build with HTTP 422, and an older frontend is rejected by the new solver with HTTP 400. Deploy both together.

### Fixed
- **"Bemap MODE_MATRIX erreur HTTP 302" when launching an optimisation.** The solver had the BeMap production host hardcoded, while the app authenticates you against the environment picked in the UI (beta by default). An account that exists on beta but not on production therefore passed sign-in and then failed at matrix time. BeMap answers an unauthenticated request with a redirect to its login page rather than a `401`, so the solver reported an opaque `502`. Route calculation now runs on the same environment you signed in to.

### Changed
- **`/optimize` payload** — `bemap_user` / `bemap_password` are replaced by a nested `config_api` block:

  ```json
  "config_api": {
    "use_api": true,
    "api_geoserver": "osm",
    "api_url": "https://bemap-beta.benomad.com/bgis/service/routing/1.0",
    "api_key": "Basic <base64(user:password)>",
    "api_timeout_seconds": 60
  }
  ```

  `api_url` is derived from the selected environment, `api_key` from the login. Same contract already used by PAV, so the two products now speak one language. The solver validates the host against an allowlist before calling it, and accepts plain `http://` only for internal hosts.
- **Distance matrix and drawn route now share one road network.** The matrix call sent no `geoserver` (server default) while the route call forced `here`, so the durations being optimised and the polyline being displayed came from different networks. Both now use `api_geoserver`.
- **The password no longer reaches the browser console.** The debug payload log redacts `api_key`; it previously printed the credentials in clear whenever debug logging was on.

### Notes
- Test suite grew from 417 → **424** (+7): per-environment URL mapping for `buildConfigApi()`, Basic-header encoding, absent-credential handling, and unknown-environment rejection.
- No user-visible UI change. Same screens, same workflow.
- The environment selector still offers beta / preprod / prod. The solver additionally accepts a dev target, which is not exposed in the UI.

---

## [0.0.5] — 2026-07-30 — Stale-cache prevention (build-token + runtime guard)

### Added
- **Per-build token** — `<pkg.version>-<git-short-hash>` (plus `-dev<epoch>` when the tree is dirty). Computed once in `vite.config.js` and injected into the bundle as `__BUILD_TOKEN__` so the runtime and the emitted URLs agree by construction.
- **`?v=<token>` stamping** on every unhashed asset reference in `index.html` — a Vite `transformIndexHtml` plugin appends the token to every local `/bemap-js-api/…` `src` / `href`. `/src/main.js` is left alone (Vite rewrites it to a content-hashed asset). External URLs (Google Fonts) untouched. Idempotent, whitespace-boundary regex so a future hyphenated attribute like `data-src` is not mis-stamped.
- **`dist/version.json`** — `{ version, token, commit, builtAt }`, emitted from the SAME build so the manifest and the shipped bundle carry the same token.
- **Runtime version guard** (`src/app/versionGuard.js`) — invoked as the FIRST thing in `main.js`. Fetches `version.json` with `no-store`; on a token mismatch scoped-clears our own Cache API entries (`bemap*` / `mptiles*` / `or*` prefixes only) and reloads once via `location.replace('?_v=<token>')`. Recovers a browser stuck on stale `index.html` + old bundle.
- Guard carries the five hard-won details from evmove5:
  1. Bounded retry — max 3 attempts, 60 s apart, per-token in localStorage.
  2. Cooldown re-check — schedules its own single re-check when a cooldown is active, so a page that reloaded into a still-stale shell escapes without a manual reload.
  3. Storage-unavailable cap — falls back to inspecting the URL as storage-independent proof, capped at ONE reload. Whole-parameter buster comparison so `?other=_v=token` and `?_v=token-extra` do NOT match.
  4. Scoped cache deletion — other apps or a host portal sharing the origin are left alone; skipped names are logged.
  5. Fail-soft with one log line per outcome (`CLEARED` / `NOT CLEARED` + reason) + `getLastClear()` receipt so a post-reload boot can prove the eviction happened.
- **`npm run test:build`** — dedicated post-build validation. Runs `scripts/require-dist.mjs` first (hard-fails if `dist/index.html` or `dist/version.json` is missing) then the dist-based `build-hygiene` + `security-headers` assertions (37 specs — stamping present on every `/bemap-js-api/…` ref, token in `version.json` matches the one compiled into the bundle, `_headers` block for `/version.json` carries `no-store` + the full security header set). Wired into the GitLab CI **build** stage, `npm run deploy`, and `npm run github` so a build that fails these assertions cannot ship.

### Changed
- **`/index.html` + `/version.json` cache policy → `no-store`** in `public/_headers`. The old `no-cache, must-revalidate` on `index.html` still allowed intermediaries to hand back a stale shell (which pins the OLD bundle hash and the OLD version-guard), undermining the whole detection mechanism. `_headers` blocks do not merge across path patterns, so each specific block re-states the security-header set.
- **Runtime guard values now live in `cfg('versionGuard.*')`** — `manifestPath`, `maxAttempts`, `cooldownMs`, `busterParam`, `cachePrefixes`, `storageKeyPrefix`. No thresholds / URLs / storage prefixes outside `src/app/config.js`. A parity spec asserts `_internals` mirrors the config values verbatim.

### Notes
- Test suite grew from 372 → **417** (+45). New coverage:
  - `build-hygiene` extended for the Vite plugin + dist stamping (10 specs — run in CI via `test:build`).
  - `security-headers` extended for the `/version.json` block + `/index.html` `no-store` (2 specs).
  - `version-guard.test.js` (34 specs) — URL helpers (12), config-sourcing parity (1), every `check()` outcome (14), `getLastClear()` receipt (2), storage-unavailable cap including the false-positive traps from the handoff, test-trap protection (assertions delivered through the `done` callback so the guard's terminal `.catch` cannot swallow them).
- Bundle grew ~4 kB (guard module + URL helpers). `marked` still lazy-loaded.
- The `?v=` stamping only covers the FUTURE bugs; the CDN `no-store` header on `/index.html` + `/version.json` is what repairs users already affected by the incident this release is designed to prevent.

---

## [0.0.4] — 2026-07-11 — bemap-js-api v2.0.1 (EVMOVE-457 stale-cache fix)

### Changed
- **bemap-js-api → v2.0.1** — fixes the "blank/grey map after a same-name archive rebuild" bug (EVMOVE-457). The SDK now auto-appends `?v=<token>` to tile URLs (read from `/api/maps`) and restores the pmtiles ETag self-heal (weak-`W/`-stripped compare → `pmtiles.EtagMismatch` → `cache:'reload'`). 100 % backwards-compatible drop-in — no application code changes needed.
- Re-vendored `bemap-sw-tiles.js` alongside the main SDK bundle. Not wired into the app (we still run `tilesSliceMode:'200'` — default HTTP-200 slice reads, browser HTTP cache handles reuse) but present so the opt-in range mode remains available if needed later. The SDK auto-unregisters any stale registration on load.

### Notes
- **Server side matters** — the fix only *activates* against `mptiles-api` Workers that publish `/api/maps.versions` + `aliasVersions` and expose `ETag`. As of this release: **dev** and **beta** have it; **preprod** and **prod** still run the pre-fix Worker. On those envs the SDK degrades gracefully (no `?v`, self-heal no-ops → behaviour identical to today). Safe to ship everywhere; the fix turns on automatically once each env's Worker is upgraded.
- Bundle sizes: `bemap-js-api.min.js` 419 kB → 436 kB (SDK carries the version-resolution + ETag-compare code). App bundle unchanged.
- No test changes; the version banner now reports `bemap-js-api v2.0.1` alongside the app version.

---

## [0.0.3] — 2026-07-10 — German locale, in-app docs viewer, SDK v2 tiles

### Added
- **German (DE) locale** — 4th UI language (FR / EN / IT / DE). Full parity with the other locales (191 keys each).
- **In-app documentation viewer** — a 📖 button in the left-panel header opens a modal that renders the User Guide + API Reference in EN / FR, with a download button for the raw markdown. Content rendered client-side by `marked` (lazy-loaded so the initial bundle stays small).
- **PAV-style language dropdown** — replaces the 3-flag inline row with a compact click-to-open menu that scales as more locales land. Announces selection via `aria-pressed` (native `<button>` semantics, no dual-role surface).

### Changed
- **bemap-js-api v2.0.0 tile stack** — tiles are now fetched as cacheable **HTTP 200** `?r=<start>-<end>` requests (previously HTTP 206 ranges). The browser HTTP cache handles reuse; no companion Service Worker is needed. `bemap-sw-tiles.js` was removed from `public/`.
- User Guide (FR + EN) refreshed to match the current runtime — new dropdown + 📖 button in the header diagram, session-only language persistence, in-app docs viewer section.
- API docs (`/optimize`) aligned with the current solver: canonical step-kind list, missing step fields documented, default-hub-at-depot fallback described.

### Fixed
- Language menu ARIA — dropped `role="option"` from the `<button>` items (dual-role ambiguity for screen readers); selection now announced via `aria-pressed` only.
- `wireLanguageDropdown` no longer accumulates `i18n.locale-changed` listeners on repeat init (defensive parity with the namespaced jQuery handlers).
- `docs.js` + `wireLanguageDropdown` handlers are namespaced (`.docs`, `.lang-dropdown`) so a second init replaces rather than stacks them — matters for HMR and multi-run tests.

### Notes
- Test suite grew from 280 → **372** across 31 files (+92 tests). i18n parity now covers 4 locales; new coverage for the docs viewer (error path, external-link decoration, DE-UI fallback to EN docs) and the lang dropdown (locale-change re-render, aria-expanded sync, DE option, double-init guard).
- Build is leak-free — `marked` ships as a **36 kB** lazy chunk loaded only on first 📖 click.
- `/optimize` request payload remains byte-identical to v0.0.2 — no backend dependency.

---

## [0.0.2] — 2026-06-09 — Hardening + MapLibre integration

### Added
- Full **MapLibre + PMTiles** integration via `bemap.MapLibreMap` — per-env tiles host (beta / preprod / prod), single `POST /api/login` per session.
- Global 3-phase loader overlay during `/optimize` calculation.
- Full security headers (CSP, HSTS, Referrer-Policy, X-Frame, X-CTO) + differentiated cache policy.
- `env.changed` reset chain — every screen wipes itself when the user switches env.
- Map auto-fits after solve and after CSV import.
- Internal dev README + GitHub publish script (`npm run github`) with paranoid leak scanner.
- `tests/fixtures/solver-request.sample.json` — byte-identical `/optimize` contract test (deep-equal + key-set).
- Build-hygiene regression test (`sourcemap:'hidden'` + `*.map` filter on the github publish).
- Tri-lingual log-in modal + locale-aware aria labels across the whole UI.

### Changed
- Map basemap switched from BeMap WMS (`bemap.BemapLayer`) to MapLibre + PMTiles.
- Geoserver flipped to `osm` on every env.
- Map lifecycle inverted — the map is constructed AFTER the user logs in, never before (avoids the MapLibre TilesAuth one-shot empty-creds trap).
- Public mirror now force-pushed via CI through the `--scan-only` gate before `git push`.
- Source repo origin migrated from GitHub to GitLab; GitHub is now a read-only release mirror.

### Fixed
- Env switch via "Modify" no longer needs a hard-reload — `TilesAuth.logout()` clears `localStorage['bemap_tiles_token']` on rebuild.
- MapLibre container element fully replaced between rebuilds (was leaving `maplibregl-map` class + listeners).
- Map teardown now uses `map.remove()` (SDK chain) instead of bare `map.native.remove()`.
- Auth check URL no longer carries the jQuery `?_=<timestamp>` cache-buster (was failing strict CORS).
- `?solver=` URL override now gated behind a host allowlist (closes credential-phishing vector).
- `moveToBoundingBox` no longer silently no-ops — fixed by constructing `new bemap.BoundingBox(...)` instead of passing raw arrays.
- Per-step volume tracking now reads `step.details.operation` (the real solver shape), not the non-existent `step.actions[].size_m3`.
- Source maps no longer ship to the public GitHub mirror.
- 11 dead `PLAN.md` references across source + CI cleared.
- All hardcoded French strings on the clients / import / bennes screens now go through `t()`.
- Accessibility: modal labels, focus trap, language flags, live regions, focus-visible ring, semantic interactive elements.

### Removed
- Legacy jQuery site (`public/index.html`, `public/css/`, `public/js/`, `vercel.json`) — Phase G cutover.
- Stale planning docs (`PLAN.md`, `PLAN-bemap*.md`, `PRODUCTION-READINESS.md`, `INTEGRATION.md`, `research/`).
- Old root `README.md`, `INSTALL.md`, `CHANGELOG.md` (replaced by dev-focused versions + public versions under `dist-meta/`).
- "Rejouer la tournée" button (dead feature).

### Notes
- Test suite grew from 141 → **280** across 24 files (+139 tests).
- 322-agent multi-dimension production-readiness audit identified 5 blockers + 19 high-priority items + 41 nice-to-haves — all blockers + all high-priority items closed in this release.
- Build is leak-free and **275 kB** total (down from 374 kB after dropping legacy passthrough).
- `/optimize` request payload remains byte-identical to v0.0.1 — no backend dependency.

---

## [0.0.1] — 2026-06-08 — PoC / first build

First proof-of-concept build on the BeNomad Front-End Playbook stack: Vite + ES modules + jQuery + SCSS + bemap-js-api v2. Not yet a production release — the version line is intentionally low while the new shell is validated end-to-end.

### Added

- **BeMap login gate** at boot — credentials are validated against `service/acl/1.0/user/details` before the app opens.
- **Reverse-geocode + snap-to-road** for every point the user places on the map. A point that cannot be snapped is rejected, not silently accepted.
- **Light / dark theme** toggle. Honours `prefers-color-scheme` on first visit; the choice is persisted per browser.
- **Progressive disclosure** of the left-panel sections — Configuration → Depot → New client → Solve. The next step only reveals once the previous one is complete.
- **Per-step volume tracking** in the results panel. The running load (m³) is shown next to each stop and resets to 0 at every dump. A "Volume collecté" summary tile is added when at least one vehicle picks up volume.
- **Per-vehicle eye toggle** to show / hide a single route on the map without losing the others.
- **Dismissable beginner guide** — the floating "Guide" panel can be closed with the **×** button and stays hidden on subsequent visits.
- **Tri-lingual interface** — French (default), English, Italian. All user-facing text is translated.
- **Vehicle CSV / JSON export** per vehicle and a global "Export CSV" covering the whole solution.

### Notes

- Map layer goes through the BeNomad JS API (Leaflet bindings + WMS basemap) — no raw Leaflet tile layers.
- All URLs, thresholds, palette colours and storage keys live in a single frozen config tree (`src/app/config.js`).
- The `/optimize` request sent to the solver is byte-identical to the previous frontend — no new backend dependency.
