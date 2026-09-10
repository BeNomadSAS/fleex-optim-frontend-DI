# INSTALL — self-hosting Fleex Optim Route Planner

This repository contains the **built distribution** of the Fleex Optim Route Planner frontend. There is no build step required to deploy it — just serve the files as a static site.

> Looking for the source? It lives in BeNomad's private GitLab. This GitHub repository is a read-only release mirror.

---

## Prerequisites

- A static-file HTTP server (any of: Nginx, Caddy, Apache, S3 + CloudFront, Cloudflare Pages, Netlify, GitHub Pages, Vercel, or `python -m http.server` for quick tests).
- A reachable OptimCPP solver instance — the default URL is configured at build time but can be overridden at runtime (see "Configuration" below).
- Valid BeMap credentials (entered by end-users in the in-app login).
- A modern browser (latest 2 versions of Chrome, Firefox, Edge, or Safari).

---

## Quick start (Python — for local testing)

```bash
git clone https://github.com/BeNomadSAS/fleex-optim-frontend-DI.git
cd fleex-optim-frontend-DI
python -m http.server 5500
```

Open <http://localhost:5500/>.

---

## Production hosting

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name your-host.example.com;
    root /var/www/fleex-optim;
    index index.html;

    # --- Security headers (mirror of public _headers — apply to every response) ---
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
    add_header X-Content-Type-Options    "nosniff" always;
    add_header X-Frame-Options           "DENY" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header Content-Security-Policy   "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: blob: https:; connect-src 'self' https://*.benomad.com https://*.benomad.net https://*.a.run.app https://fonts.googleapis.com https://fonts.gstatic.com; font-src 'self' https://fonts.gstatic.com data:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'" always;

    # SPA-style fallback: any unknown path → index.html.
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Long-cache the hashed asset bundles, never the entry HTML.
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }
    location /bemap-js-api/ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }
    location = /index.html {
        add_header Cache-Control "no-cache, must-revalidate" always;
    }
}
```

### Cloudflare Workers Static Assets

A `wrangler.jsonc` config is **not** included here (it lives in the source repo). To deploy to your own Cloudflare Worker, create a Worker pointing at this folder with `assets.directory = "./"` and `assets.not_found_handling = "single-page-application"`. Wrangler ≥ 4.x reads the bundled `_headers` file automatically — there is **no** `assets.headers` field in the wrangler schema, so do not try to inline the headers into `wrangler.jsonc`.

<!--
  HP-A TODO: when the headers-config theme lands, this section should be
  rewritten to describe the source repo's `wrangler.jsonc` headers section
  rather than `_headers`. The published mirror will still ship a built
  artefact, but the authoritative config location will have moved.
-->

### Cache + security headers

The included [`_headers`](./_headers) file is the **single source of truth** for both cache policy AND the full security-header set (Content-Security-Policy, Referrer-Policy, HSTS, X-Content-Type-Options, X-Frame-Options). The source repo's `wrangler.jsonc` does **not** inline these — `_headers` is authoritative.

The cache policy is differentiated: `no-cache, must-revalidate` on the root catch-all (including `index.html`) so a redeploy is picked up on next page load, and `public, max-age=31536000, immutable` on hashed bundles under `/assets/*` and `/bemap-js-api/*` so the browser can long-cache content-addressed files safely.

| Host | How `_headers` is applied |
|---|---|
| Cloudflare Pages | Native — Pages parses `_headers` automatically. |
| Cloudflare Workers Static Assets (wrangler ≥ 4) | Native — wrangler uploads `_headers` with the asset bundle at deploy. |
| Netlify | Native — Netlify implements the same `_headers` syntax. |
| Nginx / Apache / S3+CloudFront | Documentation only — translate manually (see the Nginx `add_header` block above for the equivalent directives). |

The `Referrer-Policy: strict-origin-when-cross-origin` line is **important** if your hosting origin is reachable from the public Internet: without it, the browser can leak the visited URL (including any query-string parameters) on cross-origin requests such as the BeMap WMS tile GETs. Do not drop it.

---

## Configuration

The frontend resolves the solver URL at runtime in this order:

1. URL query `?solver=https://...` — useful for shareable links to a specific environment.
2. `localStorage.or_solver_url` — set once via DevTools, persists per browser.
3. The compiled-in default (whichever URL the build was configured with).

```js
// In DevTools console:
localStorage.setItem('or_solver_url', 'https://your-solver.example.com');
location.reload();
```

The BeMap environment (Beta / Préproduction / Production) and credentials are entered by end-users in the in-app login modal at boot.

---

## CORS

The solver must allow your hosting origin in its FastAPI CORS configuration. Without this, the `/optimize` POST will be blocked by the browser.

Contact your BeNomad account manager to have your origin added to the allowlist.

---

## Updating

This repository is rebuilt and force-pushed by CI on every release. To update your deployment, pull the latest and redeploy:

```bash
git pull
# then re-run your deploy step (rsync to nginx, redeploy to Cloudflare Pages, etc.)
```

The hashed asset filenames mean the new bundle won't collide with the cached old one; users get the new app on next page load thanks to the `no-cache` policy on `index.html`.
