# Fleex Optim Route Planner — User Guide (EN)

**Live URL:** <https://fleex-optim.benomad.net>

Fleex Optim Route Planner builds optimised multi-vehicle tours for collecting or delivering skip bins (bennes) across a set of clients. It is powered by BeNomad heavy-vehicle routing and the OptimCPP solver on the backend.

---

## 1. Login

When the app opens, a BeMap login dialog appears.

1. Pick your **environment**: Beta / Preprod / Prod.
2. Enter your BeMap **username** and **password**.
3. Tick **Remember me on this device** if this is a personal workstation — credentials are stored locally (base64-obfuscated) on the device.
4. Click **Sign in**.

Credentials are validated against the BeMap user-details service before the app opens.

### Switching environment later — the Modifier button

The top of the left panel has a collapsible **Configuration BeMap** card showing the active user and env. Click **Modifier** to re-open the login dialog. If you switch env (e.g. beta → preprod), the app performs a clean reset: depot, clients and previous results are wiped so you start fresh on the new env. Same-env credential refresh (e.g. password rotation) keeps your working state.

---

## 2. Left panel — input

Sections of the left panel reveal progressively as you complete each step ("progressive disclosure"). If a section is not visible yet, the previous step is still pending.

The left panel header also carries a small **BeNomad logo** + product name, and on a second row: a connection-status badge, the **language selector** (drop-down menu FR / EN / IT / DE), the **📖 documentation** button (opens the in-app viewer), the **light/dark theme toggle** (🌙 / ☀️), and a chevron to collapse the panel.

### Step 1 — Configuration

Enter the **number of vehicles** and how many of them are **trailer-equipped**. Confirm with **Continuer →**.

### Step 2 — Depot (auto-armed)

The moment the Depot section appears, the **Place on map** button is **already armed** — you'll see a pulsing border around it. **Just click on the map** to drop the depot; no extra click on the button needed.

The point is verified against the road network: if it lands off-road, you'll see a toast and the button stays armed so you can try again. To re-place the depot later, click **Place on map** to re-arm.

### Step 3 — New client (auto-progressing)

The form auto-progresses through the steps. You only click the form when you want to override the default flow.

1. Pick the **operation**: Exchange / Round trip / Drop-off / Pickup.
2. Pick the **bin size** (manage the list via **Manage sizes** — see *§ Bennes* below).
3. **Client position** is auto-armed when you enter the section → click on the map.
4. On a green geocode:
   - For *Exchange* / *Round trip* / *Pickup* → **Dump point** auto-arms. If the dump is at a recycling hub, tick **The dump is a hub** before clicking the map.
   - For *Drop-off* → no dump needed; focus jumps straight to **Add this client**.
5. After the dump's green geocode → focus jumps to **Add this client** with a green pulse. Press **Enter** to submit.

After **Add this client**, the form clears and **Client position** re-arms automatically for the next client. The flow loops as long as you keep adding.

#### How the auto-flow yields to you

- A geocode failure (off-road, no result, timeout) keeps the same button armed so you can click again. Manual override: clicking another button (e.g. **Dump point** before the client is placed) tells the FSM you've taken control — it won't re-arm on subsequent failures.
- Clicking **Remove all** or **Start optimisation** halts the auto-progression so a stray map click does NOT add another client. Click **Client position** to re-engage.

Each placed client lands in the **Clients** list. You can hide a client (eye, removes it from the next solve) or remove it (trash) at any time.

### Step 3.5 — Paramètres avancés (optional)

A collapsible **Advanced settings** section above the form lets you override the service durations (in minutes) per operation type — exchange, round-trip, drop-off, pickup, dump, hub, plus trailer-specific variants. **Reset defaults** restores the seeded values. These durations are passed through to the solver and affect each tour's total time.

### Step 4 — Run optimisation

Once at least one client is in, **Start optimisation** becomes clickable. A **full-screen 3-phase loader** appears for the duration of the call:

1. **Phase 1 / 3** — heavy-vehicle routing matrix (BeNomad).
2. **Phase 2 / 3** — OR-Tools solver.
3. **Phase 3 / 3** — polylines + step rendering.

---

## 3. Right panel — results

When the optimisation completes the right panel slides open and shows:

- **Top summary**: vehicles, clients served, km travelled, total duration, total volume collected (m³).
- **Export CSV** at the top — exports the whole solution in a single file.
- **One vcard per vehicle**, colour-coded. Each one shows:
    - tour metrics (distance, duration, volume);
    - an **eye** button to show / hide that vehicle's route on the map without losing the others;
    - an **export menu** (CSV / JSON per vehicle, PDF + BeNav coming soon);
    - the detailed step list (DEPOT → clients → dumps → return). For every client stop, the right side shows the **volume added** at that stop and the **running total**. Dump stops show a reset marker `(X → 0)` indicating the truck just emptied.

Click any step row to **re-centre the map** on that stop.

The header has **Show all** / **Hide all** buttons to expand or collapse every vcard at once.

---

## 4. Collapsing the panels

Each side panel has a chevron in its header. Click it to collapse the panel **vertically** — the panel shrinks to a thin title pill at the top showing logo + product name (left) or "Résultats" (right) + a ▼ chevron. The map below the pill is fully revealed. Click ▼ to expand again.

---

## 5. Import / Export

The left panel includes a **Data import** section (collapsible).

- **Import CSV** — minimum columns: `x_client, y_client` (the column names `lng, lat` and `lon, lat` are also accepted; both `,` and `;` delimiters auto-detected; BOM stripped).
- **Import JSON** — paste a JSON array of client records.
- **Example CSV** / **Example JSON** — download a template you can edit.

Imported clients enter the Clients list with the same shape as manually-added ones — they participate in the next solve, and you can hide / remove them.

---

## 6. Theme, language, guide

- **Light / dark theme** — moon/sun button in the panel header. Honours the OS preference on first launch; the choice is then persisted in `localStorage` per browser.
- **Language** — drop-down menu in the panel header (FR / EN / IT / DE). Click the active flag to open the list, then pick a language — the change is instant, no reload. *Note: the choice applies to the current session only; reloading the page returns to French.*
- **Step-by-step guide** — if you're new, keep the "Guide" panel open (sticky at the top of the left panel). It highlights the next action card. Click the **×** to dismiss it permanently — `localStorage` remembers the choice per browser. To restore the guide, clear site data (DevTools → Application → Local Storage) or use a different browser.

---

## 7. Documentation viewer

The **📖** button in the left-panel header opens the **in-app documentation viewer**. A modal appears with two tabs — *User Guide* and *API Reference* — each available in **EN** and **FR**. The **Download (.md)** button in the modal footer delivers the raw markdown file of the document on screen.

Content is rendered client-side via `marked` (lazy-loaded so the initial bundle stays small). External links in the document open in a new tab.

---

## 8. Support

For BeMap credentials, integration or licensing questions, contact your BeNomad representative.
