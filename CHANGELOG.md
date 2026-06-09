# Changelog

All public, user-visible changes to Fleex Optim Route Planner. The full internal changelog lives in the source repository.

Format: [Keep a Changelog](https://keepachangelog.com/). Versioning: [SemVer](https://semver.org/).

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
