# API /optimize -- Technical Documentation

---

## Base URL

```
https://vrp-solver-bennes-ppkwlnj4za-ew.a.run.app
```

---

## GET /health

Checks the server status and the availability of the C++ solver.

**Response (200)**

```json
{
  "status": "ok",
  "solver_available": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Always `"ok"` if the server responds |
| `solver_available` | boolean | `true` if the C++ binary is present and executable |

---

## POST /optimize

All-in-one endpoint that performs the entire optimization process:

1. **Distance/time matrix computation** via the Bemap API (MODE_MATRIX, heavy-vehicle profile)
2. **Route optimization** using the Google OR-Tools C++ solver
3. **Route calculation** via the Bemap API (MODE_VIAS, heavy-vehicle profile)
4. **Arrival time recalculation** for each step based on actual travel durations

**Required header**

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |

---

### Request parameters

```json
{
  "num_vehicles": 3,
  "num_trailer_vehicles": 0,
  "depot": [7.2620, 43.7102],
  "jobs": [ ... ],
  "hubs": [ ... ],
  "optimize_by": "time",
  "service_times": { ... },
  "config_api": {
    "use_api": true,
    "api_geoserver": "osm",
    "api_url": "https://bemap-beta.benomad.com/bgis/service/routing/1.0",
    "api_key": "Basic <base64(username:password)>",
    "api_timeout_seconds": 60
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|:--------:|---------|-------------|
| `num_vehicles` | int | **yes** | -- | Number of available vehicles |
| `num_trailer_vehicles` | int | no | `0` | Number of vehicles with a trailer (among the `num_vehicles`) |
| `depot` | [lon, lat] | **yes** | -- | Depot coordinates (start and return point) |
| `jobs` | array | **yes** | -- | List of customers to serve (see format below) |
| `hubs` | array | no | `[]` | List of disposal sites/hubs (see format below) |
| `optimize_by` | string | no | `"time"` | Optimization criterion: `"time"` or `"distance"` |
| `service_times` | object | no | (see defaults) | Service time per operation type, in seconds |
| `config_api` | object | **yes** | -- | Bemap target: environment plus credentials (see below) |

#### `config_api` object

The server no longer picks the Bemap environment: the caller supplies it,
together with matching credentials. An account valid on one environment is
not necessarily valid on another.

| Field | Type | Required | Default | Description |
|-------|------|:--------:|---------|-------------|
| `api_url` | string | **yes** | -- | Routing endpoint of the target environment. Any host outside the `benomad.com` domain is rejected (400) |
| `api_key` | string | **yes** | -- | Ready-made auth header: `"Basic " + base64(username:password)` |
| `use_api` | bool | no | `true` | Use the Bemap API for the matrix |
| `api_geoserver` | string | no | (server default) | Road network, e.g. `"osm"`. Applied to BOTH the matrix and the routes |
| `api_timeout_seconds` | number | no | `60` | Bemap call timeout, in seconds |

Available environments:

| Env | `api_url` |
|---------|-----------|
| dev | `http://bemap-dev.int.benomad.com/bgis/service/routing/1.0` |
| beta | `https://bemap-beta.benomad.com/bgis/service/routing/1.0` |
| preprod | `https://bemap-preprod.benomad.com/bgis/service/routing/1.0` |
| prod | `https://bemap.benomad.com/bgis/service/routing/1.0` |

Plain `http://` is accepted only for internal hosts (`*.int.benomad.com`).
Anywhere else it is rejected, so Basic credentials never travel in clear.

---

### Job format

Each element in the `jobs` array represents a customer to serve.

```json
{
  "id": 1,
  "x_client": 7.2690,
  "y_client": 43.7034,
  "x_dump": 7.2041,
  "y_dump": 43.7123,
  "op": "ECHANGE",
  "size_m3": "Benne ciel ouvert 30m3",
  "dump_is_hub": true,
  "service_duration": 900
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | int | **yes** | Unique customer identifier |
| `x_client` | float | **yes** | Customer longitude |
| `y_client` | float | **yes** | Customer latitude |
| `x_dump` | float | **yes** | Dump site longitude |
| `y_dump` | float | **yes** | Dump site latitude |
| `op` | string | **yes** | Operation type: `ECHANGE`, `ALLER_RETOUR`, `DEPOSE`, `RETRAIT` |
| `size_m3` | string | **yes** | Container type and size (e.g., `"Benne ciel ouvert 30m3"`) |
| `dump_is_hub` | bool | no | `true` if the dump site is a hub/disposal site. Default: `false` |
| `service_duration` | int | no | Service duration at the customer site in seconds. Default: value from `service_times` |

---

### Operations

| Operation | Description |
|-----------|-------------|
| `ECHANGE` | Drop off a clean skip, pick up the full skip, go empty it at the treatment site |
| `ALLER_RETOUR` | Pick up the full skip from the customer, go empty it, bring the empty skip back to the customer |
| `DEPOSE` | Drop off a container at the customer site (no trip to the dump) |
| `RETRAIT` | Pick up a container from the customer and haul it away |

---

### Hub format

Hubs are disposal/treatment sites shared by multiple customers.

```json
{
  "id": 100,
  "x": 7.2041,
  "y": 43.7123,
  "allowed_types": ["10", "15", "20", "30"]
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | int | **yes** | Unique hub identifier |
| `x` | float | **yes** | Hub longitude |
| `y` | float | **yes** | Hub latitude |
| `allowed_types` | string[] | no | Skip types/sizes accepted by this hub |

**Default-hub fallback:** when `hubs` is omitted or empty, the `/optimize` endpoint **injects a synthetic hub at the depot coordinates** with `allowed_types` covering every distinct `size_m3` from the jobs. To suppress this, pass an explicit `hubs` array (even an empty one with one realistic entry).

---

### Service times (`service_times`)

Optional object to customize the duration of each operation type. All values are in seconds.

```json
{
  "client_exchange": 900,
  "client_rotation": 900,
  "client_depose": 480,
  "client_retrait": 720,
  "dump": 840,
  "hub": 1800,
  "client_exchange_trailer": 1200,
  "client_retrait_trailer": 840,
  "dump_trailer": 1140
}
```

| Key | Default (s) | Default (min) | Description |
|-----|:-----------:|:-------------:|-------------|
| `client_exchange` | 900 | 15 | Standard exchange at the customer site |
| `client_rotation` | 900 | 15 | Round trip (pickup, dump, return) |
| `client_depose` | 480 | 8 | Container drop-off |
| `client_retrait` | 720 | 12 | Container pickup |
| `dump` | 840 | 14 | Dumping at the treatment site |
| `hub` | 1800 | 30 | Operations at the disposal site/hub |
| `client_exchange_trailer` | 1200 | 20 | Exchange with a trailer vehicle |
| `client_retrait_trailer` | 840 | 14 | Pickup by a trailer vehicle |
| `dump_trailer` | 1140 | 19 | Trailer dumping |

If `service_times` is omitted, the default values above are used.

---

### Response (200 OK)

```json
{
  "status": "ok",
  "run_id": "26ea4eba",
  "solution": {
    "status": "ok",
    "optimize_by": "time",
    "summary": {
      "vehicles": [ ... ],
      "total_time_min": 145,
      "total_distance_km": 0,
      "dropped_jobs": []
    }
  },
  "solver_log": "[OR-Tools] 3000 solutions | best: 197373 ...",
  "timing": {
    "solver_seconds": 1.85,
    "total_seconds": 2.82
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` if the solver found a solution |
| `run_id` | string | Unique run identifier (8 hex characters) |
| `solution` | object | Complete solution (see below) |
| `solver_log` | string | C++ solver logs (last 3000 characters) |
| `timing.solver_seconds` | float | Time spent in the C++ solver |
| `timing.total_seconds` | float | End-to-end including Bemap matrix + routing + post-processing |

---

### Solution structure

| Field | Type | Description |
|-------|------|-------------|
| `solution.status` | string | `"ok"` if a solution was found |
| `solution.optimize_by` | string | Criterion used: `"time"` or `"distance"` |
| `solution.summary.vehicles` | array | Per-vehicle tour details (see below) |
| `solution.summary.total_time_min` | int / float | Aggregate solver time across vehicles, in minutes |
| `solution.summary.total_distance_km` | float | Aggregate solver distance, in kilometres. **Note:** the C++ solver currently emits the unweighted value here — sum `summary.vehicles[].metrics.total_distance_meters` for the Bemap-recalibrated total |
| `solution.summary.dropped_jobs` | int[] | IDs of customers the solver could not assign (empty when every job is served) |

---

### Vehicle structure

```json
{
  "vehicle_id": 1,
  "steps": [ ... ],
  "metrics": {
    "total_distance_meters": 12180,
    "total_time_seconds": 5325,
    "total_clients": 2
  },
  "route_geometry": ["encoded_polyline_leg_1", "encoded_polyline_leg_2", "..."],
  "total_shift_duration": 5325,
  "formatted_duration": "1h28"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `vehicle_id` | int | Vehicle number (starts at 1) |
| `steps` | array | Ordered list of route steps |
| `metrics.total_distance_meters` | int | Total distance in metres, recalibrated from Bemap routing |
| `metrics.total_time_seconds` | int | Total duration in seconds (travel + service), recalibrated from Bemap |
| `metrics.total_clients` | int | Number of customers served by this vehicle |
| `route_geometry` | string[] | **Array** of encoded polylines (Google Encoded Polyline format, precision 5) — one entry per leg between consecutive steps. Concatenate to render the full tour |
| `total_shift_duration` | int | Total shift duration in seconds |
| `formatted_duration` | string | Formatted duration — hour without zero-padding, e.g. `"1h28"`, `"0h59"` |

---

### Step structure

```json
{
  "node_index": 2,
  "label": "C2",
  "kind": "CLIENT_STANDARD",
  "coords": [7.25775, 43.69854],
  "lat": 43.69854,
  "lon": 7.25775,
  "solver_cumulative": 323,
  "details": {
    "service_duration": 900,
    "operation": "ECHANGE Benne ciel ouvert 10m3"
  },
  "arrival_time_str": "0h05",
  "distance_to_next": 0,
  "distance_km": 0,
  "cumul_distance_km": 1.5
}
```

| Field | Type | Description |
|-------|------|-------------|
| `node_index` | int | Internal node index in the solver graph. **`-1`** for synthetic steps the API injects between solver outputs (e.g. `VIDAGE (C2)`, `RETOUR BENNE (C2)`) — they do not correspond to a real node in the solver graph |
| `label` | string | Step name (e.g., `"C3"`, `"DEPOT"`, `"VIDAGE (C2)"`, `"RETOUR"`) |
| `kind` | string | Step type (see table below) |
| `coords` | [lon, lat] | Coordinates in `[longitude, latitude]` format |
| `lat` | float | Latitude |
| `lon` | float | Longitude |
| `solver_cumulative` | int | Cumulative time since departure in seconds (raw solver value, before the Bemap routing re-calibration applied by `arrival_time_str` / `cumul_distance_km`) |
| `details.service_duration` | int | Service duration at this step in seconds |
| `details.operation` | string | Operation description — for clients includes the bin label (e.g., `"ECHANGE Benne ciel ouvert 10m3"`), for dumps is `"VIDAGE"`, for the start/end depot is `"DEPART"` / `"FIN"` |
| `arrival_time_str` | string | Formatted arrival time since departure — hour without zero-padding, e.g. `"0h05"`, `"1h28"` |
| `distance_to_next` | int | **Raw solver value, not Bemap-recalibrated.** Often `0`; use `cumul_distance_km` for the trustworthy per-step accumulator |
| `distance_km` | float | **Raw solver value, not Bemap-recalibrated.** Often `0`; use `cumul_distance_km` instead |
| `cumul_distance_km` | float | Cumulative distance since departure in kilometres, **recalibrated by the post-processor** from the actual Bemap routing legs |

---

### Step types (`kind`)

Canonical list emitted by the current C++ solver (`solver/main.cpp`):

| Kind | Description |
|------|-------------|
| `DEPOT` | Departure from or return to the depot (the first and last step of every vehicle's tour) |
| `CLIENT_STANDARD` | Standard customer visit by a truck without a trailer |
| `CLIENT_PRIME` | Primary customer in a trailer pairing (the larger of the two skip volumes — kept on the trailer) |
| `CLIENT_DOUBLE` | Secondary customer in a trailer pairing (kept on the truck — LIFO with the prime) |
| `CLIENT_COMBO` | Combo customer visit — the truck collects while the trailer is dropped off elsewhere |
| `CLIENT_RETURN` | Empty-skip return to a single-truck customer (after the dump) |
| `CLIENT_RETURN_TRAILER` | Empty-skip return to the trailer's customer |
| `CLIENT_RETURN_COMBO` | Empty-skip return to a combo customer |
| `DUMP` | Dump at the treatment site for a single-truck flow (no trailer) |
| `DUMP_CAMION` | Dump of the truck contents at the treatment site (trailer flow) |
| `DUMP_REMORQUE` | Dump of the trailer contents at the treatment site (trailer flow) |

---

### Error codes

| HTTP Code | Description |
|:---------:|-------------|
| `200` | Success -- solution found |
| `400` | `config_api` missing or invalid: host outside the allowlist, missing `api_key`, or `http://` to a non-internal host |
| `401` | Invalid Bemap credentials |
| `500` | Internal server error or the solver did not produce a solution |
| `502` | Bemap service unavailable, or account unknown / not entitled on the target environment |
| `504` | Timeout -- the solver exceeded the 15-minute limit |

**Error format**

```json
{
  "detail": "Error description"
}
```

For 500 errors with no solution, the `detail` field may contain an object:

```json
{
  "error": "Solver produced no output",
  "log": "... last solver logs ..."
}
```

---

## Full example with curl

```bash
curl -X POST https://vrp-solver-bennes-ppkwlnj4za-ew.a.run.app/optimize \
  -H "Content-Type: application/json" \
  -d '{
    "num_vehicles": 2,
    "num_trailer_vehicles": 0,
    "depot": [7.2620, 43.7102],
    "optimize_by": "time",
    "service_times": {
      "client_exchange": 900,
      "client_rotation": 900,
      "client_depose": 480,
      "client_retrait": 720,
      "dump": 840,
      "hub": 1800,
      "client_exchange_trailer": 1200,
      "client_retrait_trailer": 840,
      "dump_trailer": 1140
    },
    "config_api": {
      "use_api": true,
      "api_geoserver": "osm",
      "api_url": "https://bemap-beta.benomad.com/bgis/service/routing/1.0",
      "api_key": "Basic bXkudXNlcm5hbWU6bXlfcGFzc3dvcmQ="
    },
    "jobs": [
      {
        "id": 1,
        "x_client": 7.2690,
        "y_client": 43.7034,
        "x_dump": 7.2041,
        "y_dump": 43.7123,
        "op": "ECHANGE",
        "size_m3": "Benne ciel ouvert 30m3",
        "dump_is_hub": true,
        "service_duration": 900
      },
      {
        "id": 2,
        "x_client": 7.2150,
        "y_client": 43.6720,
        "x_dump": 7.1085,
        "y_dump": 43.6574,
        "op": "ECHANGE",
        "size_m3": "Benne ciel ouvert 20m3",
        "dump_is_hub": true,
        "service_duration": 900
      },
      {
        "id": 3,
        "x_client": 7.1920,
        "y_client": 43.7260,
        "x_dump": 7.2041,
        "y_dump": 43.7123,
        "op": "ALLER_RETOUR",
        "size_m3": "Benne ciel ouvert 30m3",
        "dump_is_hub": true,
        "service_duration": 1000
      }
    ],
    "hubs": [
      {
        "id": 100,
        "x": 7.1085,
        "y": 43.6574,
        "allowed_types": ["10", "15", "20", "30"]
      },
      {
        "id": 101,
        "x": 7.2041,
        "y": 43.7123,
        "allowed_types": ["10", "15", "20", "30"]
      }
    ]
  }'
```

---

## Technical notes

- **Timeout**: the request expires after a maximum of 15 minutes. Beyond that, a 504 error is returned.

- **Coordinates**: all coordinates are in **[longitude, latitude]** format (GeoJSON standard). Do not confuse with the [latitude, longitude] format used by some tools.

- **Bemap credentials**: `config_api.api_key` is used only for Bemap routing calls during request processing. It is never stored on the server side.

- **Environment**: the server holds no hardcoded Bemap URL. It uses `config_api.api_url` after validating it against a host allowlist. That is what guarantees the matrix is computed on the same environment the user signed in to. An account valid on beta but absent from prod now produces an explicit error: Bemap answers an unauthenticated request with a redirect to its login page (3xx), not a 401.

- **Distance matrix**: the server computes the distance/time matrix via the Bemap MODE_MATRIX API (heavy-vehicle profile, height 3m, weight 19t, width 2.3m) before passing it to the solver.

- **Routes**: after optimization, actual routes are calculated via the Bemap MODE_VIAS API to obtain trace polylines and segment-by-segment travel durations, enabling accurate recalculation of arrival times.
