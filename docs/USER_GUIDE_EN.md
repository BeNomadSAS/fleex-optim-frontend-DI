# Fleex Optim Route Planner — User Guide (EN)

**Live URL:** <https://fleex-optim.benomad.net>

Fleex Optim Route Planner builds optimised multi-vehicle tours for collecting or delivering skip bins (bennes) across a set of clients. It is powered by BeNomad heavy-vehicle routing and the OptimCPP solver on the backend.

---

## 1. Login

When the app opens, a BeMap login dialog appears.

1. Pick your **environment**: Beta / Preprod / Prod.
2. Enter your BeMap **username** and **password**.
3. Tick **Remember me on this device** if this is a personal workstation — credentials are stored locally (base64) on the device.
4. Click **Sign in**.

Credentials are validated against the BeMap user-details service before the app opens.

---

## 2. Left panel — input

Sections of the left panel reveal progressively as you complete each step ("progressive disclosure"). If a section is not visible yet, the previous step is still pending.

### Step 1 — Configuration

Enter the **number of vehicles** and how many of them are **trailer-equipped**. Confirm with **Continue →**.

### Step 2 — Depot

Click **Place on map**, then click on the map to drop the depot. The point is verified against the road network: if it lands off-road you'll be asked to reposition it.

### Step 3 — New client

For every client:

1. Pick the **operation**: Exchange / Round trip / Drop-off / Pickup.
2. Pick the **bin size** (manage the list via *Manage sizes*).
3. Click **Client position**, then click on the map.
4. If the operation needs it (Exchange / Round trip / Pickup), click **Dump point** and place the dump location.
5. Click **Add this client**.

The client lands in the **Clients** list right below. You can hide a client (eye) or remove it (trash) at any time.

### Step 4 — Run optimisation

Once at least one client is in, the **Start optimisation** button becomes clickable. The run goes through 3 phases: routing matrix → solver → rendering.

---

## 3. Right panel — results

When the optimisation completes, the right panel slides open and shows:

- **Top summary**: vehicles, clients served, km travelled, total duration, total volume (m³).
- **One vcard per vehicle**, colour-coded. Each one gives you:
    - tour metrics (distance, duration, volume);
    - an **eye** button to show / hide the route on the map;
    - an **export menu** (CSV / JSON, PDF + BeNav coming soon);
    - the detailed step list (DEPOT → clients → dumps → return) with, next to every client stop, the volume added and the running total. Dump stops show a reset marker `(X → 0)`.

Click any step to recentre the map on that point.

The **Export CSV** button at the top of the panel exports the whole solution in a single file.

---

## 4. Import / Export

The left panel includes a **Data import** section:

- **Import CSV**: minimum required columns are `x_client,y_client` (`lng,lat` or `lon,lat` are also accepted; both `,` and `;` delimiters auto-detected).
- **Import JSON**: paste a JSON array of clients.
- Click **Example CSV** or **Example JSON** to download a template.

---

## 5. Theme, language, guide

- **Light / dark theme**: button at the top of the left panel. Respects the OS preference on first launch.
- **Language**: selector at the top. The choice is persisted.
- **Step-by-step guide**: if you're new, keep the floating "Guide" panel open — it highlights the next step. Click the **×** to dismiss it permanently (the app remembers per browser).

---

## 6. Support

For BeMap credentials, integration or licensing questions, contact your BeNomad representative.
