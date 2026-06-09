# API /optimize -- Technical Documentation

---

## Base URL

```
https://vrp-solver-ppkwlnj4za-ew.a.run.app
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
  "bemap_user": "my.username",
  "bemap_password": "********"
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
| `bemap_user` | string | **yes** | -- | Bemap account username |
| `bemap_password` | string | **yes** | -- | Bemap account password |

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
  "run_id": "a1b2c3d4",
  "solution": {
    "status": "ok",
    "optimize_by": "time",
    "summary": {
      "total_vehicles_used": 3,
      "total_jobs_assigned": 19,
      "total_jobs_dropped": 0,
      "total_time_min": 450.5,
      "total_distance_km": 185.3,
      "dropped_jobs": [],
      "vehicles": [ ... ]
    }
  },
  "solver_log": "[OR-Tools] 3000 solutions | best: 197373 ...",
  "timing": {
    "solver_seconds": 12.5,
    "total_seconds": 18.3
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` if the solver found a solution |
| `run_id` | string | Unique run identifier (8 characters) |
| `solution` | object | Complete solution (see below) |
| `solver_log` | string | C++ solver logs (last 3000 characters) |
| `timing` | object | Execution time (solver and total) |

---

### Solution structure

| Field | Type | Description |
|-------|------|-------------|
| `solution.status` | string | `"ok"` if a solution was found |
| `solution.optimize_by` | string | Criterion used: `"time"` or `"distance"` |
| `solution.summary.total_vehicles_used` | int | Number of vehicles actually used |
| `solution.summary.total_jobs_assigned` | int | Number of customers served |
| `solution.summary.total_jobs_dropped` | int | Number of customers not served |
| `solution.summary.total_time_min` | float | Total cumulative duration in minutes |
| `solution.summary.total_distance_km` | float | Total cumulative distance in kilometers |
| `solution.summary.dropped_jobs` | int[] | IDs of customers not served |
| `solution.summary.vehicles` | array | Details per vehicle |

---

### Vehicle structure

```json
{
  "vehicle_id": 1,
  "steps": [ ... ],
  "metrics": {
    "total_distance_meters": 45000,
    "total_time_seconds": 5400,
    "total_clients": 6
  },
  "route_geometry": ["encoded_polyline_segment_1", "encoded_polyline_segment_2"],
  "total_shift_duration": 5400,
  "formatted_duration": "01h30"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `vehicle_id` | int | Vehicle number (starts at 1) |
| `steps` | array | Ordered list of route steps |
| `metrics.total_distance_meters` | int | Total distance in meters |
| `metrics.total_time_seconds` | int | Total duration in seconds |
| `metrics.total_clients` | int | Number of customers served by this vehicle |
| `route_geometry` | string[] | Encoded polylines (Google Encoded Polyline format) for map rendering |
| `total_shift_duration` | int | Total shift duration in seconds |
| `formatted_duration` | string | Formatted duration (e.g., `"01h30"`) |

---

### Step structure

```json
{
  "node_index": 5,
  "label": "C3 (CAMION)",
  "kind": "CLIENT_STANDARD",
  "coords": [7.1920, 43.7260],
  "lat": 43.7260,
  "lon": 7.1920,
  "solver_cumulative": 1800,
  "details": {
    "service_duration": 900,
    "operation": "ECHANGE Benne ciel ouvert 30m3"
  },
  "arrival_time_str": "0h30",
  "cumul_distance_km": 12.5
}
```

| Field | Type | Description |
|-------|------|-------------|
| `node_index` | int | Internal node index in the solver graph |
| `label` | string | Step name (e.g., `"C3 (CAMION)"`, `"DEPOT"`, `"VIDAGE"`) |
| `kind` | string | Step type (see table below) |
| `coords` | [lon, lat] | Coordinates in [longitude, latitude] format |
| `lat` | float | Latitude |
| `lon` | float | Longitude |
| `solver_cumulative` | int | Cumulative time since departure in seconds (raw solver value) |
| `details.service_duration` | int | Service duration at this step in seconds |
| `details.operation` | string | Operation description (e.g., `"ECHANGE Benne ciel ouvert 30m3"`) |
| `arrival_time_str` | string | Formatted arrival time since departure (e.g., `"1h30"`) |
| `cumul_distance_km` | float | Cumulative distance since departure in kilometers |

---

### Step types (`kind`)

| Kind | Description |
|------|-------------|
| `DEPOT` | Departure from or return to the depot |
| `DEPOT_RESTOCK` | Restocking at the depot |
| `CLIENT_STANDARD` | Standard customer visit (truck only) |
| `CLIENT_DOUBLE` | Customer visit with trailer (LIFO mode) |
| `CLIENT_COMBO` | Combo customer visit (truck while trailer is dropped off elsewhere) |
| `CLIENT_RETURN_TRAILER` | Empty skip return to the customer (trailer) |
| `CLIENT_RETURN_COMBO` | Empty skip return to the customer (combo) |
| `DUMP_CAMION` | Dumping the truck contents at the treatment site |
| `DUMP_REMORQUE` | Dumping the trailer contents at the treatment site |
| `HUB_VIRTUAL` | Stop at a hub/disposal site |
| `HUB_BIN_SWAP` | Skip swap at the hub |
| `TRAILER_DROP` | Trailer drop-off at an intermediate point |
| `TRAILER_PICKUP` | Trailer pickup at an intermediate point |

---

### Error codes

| HTTP Code | Description |
|:---------:|-------------|
| `200` | Success -- solution found |
| `401` | Invalid Bemap credentials |
| `500` | Internal server error or the solver did not produce a solution |
| `502` | Bemap service unavailable |
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
curl -X POST https://vrp-solver-ppkwlnj4za-ew.a.run.app/optimize \
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
    "bemap_user": "my.username",
    "bemap_password": "my_password",
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

- **Bemap credentials**: the `bemap_user` and `bemap_password` credentials are used only for Bemap routing calls during request processing. They are never stored on the server side.

- **Distance matrix**: the server computes the distance/time matrix via the Bemap MODE_MATRIX API (heavy-vehicle profile, height 3m, weight 19t, width 2.3m) before passing it to the solver.

- **Routes**: after optimization, actual routes are calculated via the Bemap MODE_VIAS API to obtain trace polylines and segment-by-segment travel durations, enabling accurate recalculation of arrival times.
