# Fleex Optim Route Planner

> **Public distribution mirror** — by **BeNomad**.
> Built artefact + end-user documentation for the route planner shipped at
> <https://fleex-optim.benomad.net>.

[![License](https://img.shields.io/badge/license-proprietary-blue)](LICENSE)
![Stack](https://img.shields.io/badge/stack-Vite%20%7C%20jQuery%20%7C%20bemap--js--api-6366f1)

---

## What this is

Fleex Optim Route Planner is a multi-vehicle route planner for skip / bin collection and delivery. Operators drop clients on a map, configure their bin sizes (**bennes**), place a depot and dump points, define a fleet, and request optimised tours from the OptimCPP solver and BeNomad heavy-vehicle routing.

This repository is **read-only**. It contains the built static-site artefact — no source code, no build step. Source lives in a private BeNomad GitLab.

The repository is **rebuilt and force-pushed by CI on every release**. Manual changes will be overwritten on the next release.

---

## What's in this repo

```
index.html              # The single-page app entry
assets/                 # Bundled JS + CSS (build output — hashed filenames)
bemap-js-api/           # Vendored BeNomad SDK (Leaflet bindings, gep, geocoder)
bemap-sw-tiles.js       # Service worker — caches map tiles
_headers                # Cloudflare Pages cache-control / security headers
docs/                   # User Guide + API Doc (FR + EN) downloadable from the app
  USER_GUIDE_FR.md      # End-user guide (French)
  USER_GUIDE_EN.md      # End-user guide (English)
  API_DOC_FR.md         # /optimize REST contract — payload + response (French)
  API_DOC_EN.md         # /optimize REST contract — payload + response (English)

README.md               # ← you are here
INSTALL.md              # Self-hosting walk-through (Nginx, Cloudflare, S3, GitHub Pages…)
CHANGELOG.md            # Release notes
LICENSE                 # Usage terms
```

The user guides and API docs are part of the build output (the `docs/` folder
above). The four top-level Markdown files (README, INSTALL, CHANGELOG, LICENSE)
are the only "meta" files dropped on top of the build at release time.

---

## Quick start — try locally

```bash
git clone https://github.com/BeNomadSAS/fleex-optim-frontend-DI.git
cd fleex-optim-frontend-DI
python -m http.server 5500
```

Open <http://localhost:5500/>. Sign in with your BeMap credentials.

(Note: the `/optimize` solver call requires the backend to allow your origin via CORS. Contact BeNomad to whitelist `http://localhost:5500` for testing.)

---

## Self-hosting

See [`INSTALL.md`](INSTALL.md) for production hosting recipes on:

- Nginx
- Cloudflare Workers (Static Assets) / Cloudflare Pages
- Netlify / GitHub Pages
- Any plain static host (S3 + CloudFront, Apache, etc.)

The included [`_headers`](./_headers) file encodes the right cache policy for Cloudflare Pages and Netlify.

---

## Documentation

All user-facing and integrator documentation lives in the [`docs/`](docs/) folder. The deployed app downloads from the same files via the *Documentation* card.

### End-user guides
- **French (default):** [docs/USER_GUIDE_FR.md](docs/USER_GUIDE_FR.md)
- **English:** [docs/USER_GUIDE_EN.md](docs/USER_GUIDE_EN.md)
- **Italian:** the app UI is fully translated; a full Italian guide is forthcoming.

### Integrator / API reference

For developers calling the BeNomad OptimCPP solver directly — the `/optimize` REST contract that this frontend uses:
- **French:** [docs/API_DOC_FR.md](docs/API_DOC_FR.md)
- **English:** [docs/API_DOC_EN.md](docs/API_DOC_EN.md)

Covers `GET /health`, `POST /optimize`, the full request payload (clients, dumps, hubs, fleet, service times) and the response shape (vehicles, steps, route geometry, metrics).

---

## Release history

See [`CHANGELOG.md`](CHANGELOG.md). Versioning follows [SemVer](https://semver.org/); the `CHANGELOG` follows [Keep a Changelog](https://keepachangelog.com/).

---

## Reporting a problem

This mirror does not accept issues directly (changes are force-pushed and overwrite anything you'd add). For bug reports or integration questions, contact your BeNomad account manager — they will relay to the private source repo.

---

## Contact

For BeMap credentials, integration support, or licensing questions, contact your BeNomad account manager.

© BeNomad. All rights reserved. See [LICENSE](LICENSE).
