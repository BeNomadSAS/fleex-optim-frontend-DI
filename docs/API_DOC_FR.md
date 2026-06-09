# API /optimize -- Documentation technique

---

## Base URL

```
https://vrp-solver-ppkwlnj4za-ew.a.run.app
```

---

## GET /health

Verification de l'etat du serveur et de la disponibilite du solveur C++.

**Reponse (200)**

```json
{
  "status": "ok",
  "solver_available": true
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `status` | string | Toujours `"ok"` si le serveur repond |
| `solver_available` | boolean | `true` si le binaire C++ est present et executable |

---

## POST /optimize

Endpoint tout-en-un qui realise l'ensemble du processus d'optimisation :

1. **Calcul de la matrice de distances/temps** via l'API Bemap (MODE_MATRIX, profil poids-lourd)
2. **Optimisation des tournees** par le solveur C++ Google OR-Tools
3. **Calcul des itineraires** via l'API Bemap (MODE_VIAS, profil poids-lourd)
4. **Recalcul des temps d'arrivee** pour chaque etape a partir des durees de trajet reelles

**Header requis**

| Header | Valeur |
|--------|--------|
| `Content-Type` | `application/json` |

---

### Parametres de la requete

```json
{
  "num_vehicles": 3,
  "num_trailer_vehicles": 0,
  "depot": [7.2620, 43.7102],
  "jobs": [ ... ],
  "hubs": [ ... ],
  "optimize_by": "time",
  "service_times": { ... },
  "bemap_user": "mon.utilisateur",
  "bemap_password": "********"
}
```

| Champ | Type | Requis | Defaut | Description |
|-------|------|:------:|--------|-------------|
| `num_vehicles` | int | **oui** | -- | Nombre de vehicules disponibles |
| `num_trailer_vehicles` | int | non | `0` | Nombre de vehicules avec remorque (parmi les `num_vehicles`) |
| `depot` | [lon, lat] | **oui** | -- | Coordonnees du depot (point de depart et retour) |
| `jobs` | array | **oui** | -- | Liste des clients a desservir (voir format ci-dessous) |
| `hubs` | array | non | `[]` | Liste des exutoires/hubs (voir format ci-dessous) |
| `optimize_by` | string | non | `"time"` | Critere d'optimisation : `"time"` ou `"distance"` |
| `service_times` | object | non | (voir defauts) | Temps de service par type d'operation, en secondes |
| `bemap_user` | string | **oui** | -- | Nom d'utilisateur du compte Bemap |
| `bemap_password` | string | **oui** | -- | Mot de passe du compte Bemap |

---

### Format d'un job

Chaque element du tableau `jobs` represente un client a desservir.

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

| Champ | Type | Requis | Description |
|-------|------|:------:|-------------|
| `id` | int | **oui** | Identifiant unique du client |
| `x_client` | float | **oui** | Longitude du client |
| `y_client` | float | **oui** | Latitude du client |
| `x_dump` | float | **oui** | Longitude du point de vidage |
| `y_dump` | float | **oui** | Latitude du point de vidage |
| `op` | string | **oui** | Type d'operation : `ECHANGE`, `ALLER_RETOUR`, `DEPOSE`, `RETRAIT` |
| `size_m3` | string | **oui** | Type et taille du contenant (ex : `"Benne ciel ouvert 30m3"`) |
| `dump_is_hub` | bool | non | `true` si le point de vidage est un hub/exutoire. Defaut : `false` |
| `service_duration` | int | non | Duree d'intervention chez le client en secondes. Defaut : valeur de `service_times` |

---

### Operations

| Operation | Description |
|-----------|-------------|
| `ECHANGE` | Deposer une benne propre, recuperer la benne pleine, aller vider au site de traitement |
| `ALLER_RETOUR` | Prendre la benne pleine chez le client, aller vider, rapporter la benne vide chez le client |
| `DEPOSE` | Deposer un contenant chez le client (pas de passage au vidage) |
| `RETRAIT` | Recuperer un contenant chez le client et l'evacuer |

---

### Format d'un hub

Les hubs sont des exutoires/sites de traitement partages par plusieurs clients.

```json
{
  "id": 100,
  "x": 7.2041,
  "y": 43.7123,
  "allowed_types": ["10", "15", "20", "30"]
}
```

| Champ | Type | Requis | Description |
|-------|------|:------:|-------------|
| `id` | int | **oui** | Identifiant unique du hub |
| `x` | float | **oui** | Longitude du hub |
| `y` | float | **oui** | Latitude du hub |
| `allowed_types` | string[] | non | Types/tailles de bennes acceptes par ce hub |

---

### Temps de service (`service_times`)

Objet optionnel permettant de personnaliser la duree de chaque type d'operation. Toutes les valeurs sont en secondes.

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

| Cle | Defaut (s) | Defaut (min) | Description |
|-----|:----------:|:------------:|-------------|
| `client_exchange` | 900 | 15 | Echange standard chez le client |
| `client_rotation` | 900 | 15 | Aller-retour (prise, vidage, retour) |
| `client_depose` | 480 | 8 | Depose d'un contenant |
| `client_retrait` | 720 | 12 | Retrait d'un contenant |
| `dump` | 840 | 14 | Vidage au site de traitement |
| `hub` | 1800 | 30 | Operations a l'exutoire/hub |
| `client_exchange_trailer` | 1200 | 20 | Echange avec vehicule a remorque |
| `client_retrait_trailer` | 840 | 14 | Retrait par vehicule a remorque |
| `dump_trailer` | 1140 | 19 | Vidage de la remorque |

Si `service_times` est omis, les valeurs par defaut ci-dessus sont utilisees.

---

### Reponse (200 OK)

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

| Champ | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` si le solveur a trouve une solution |
| `run_id` | string | Identifiant unique de l'execution (8 caracteres) |
| `solution` | object | Solution complete (voir ci-dessous) |
| `solver_log` | string | Logs du solveur C++ (3000 derniers caracteres) |
| `timing` | object | Temps d'execution (solveur et total) |

---

### Structure de la solution

| Champ | Type | Description |
|-------|------|-------------|
| `solution.status` | string | `"ok"` si une solution a ete trouvee |
| `solution.optimize_by` | string | Critere utilise : `"time"` ou `"distance"` |
| `solution.summary.total_vehicles_used` | int | Nombre de vehicules effectivement utilises |
| `solution.summary.total_jobs_assigned` | int | Nombre de clients desservis |
| `solution.summary.total_jobs_dropped` | int | Nombre de clients non desservis |
| `solution.summary.total_time_min` | float | Duree totale cumulee en minutes |
| `solution.summary.total_distance_km` | float | Distance totale cumulee en kilometres |
| `solution.summary.dropped_jobs` | int[] | IDs des clients non desservis |
| `solution.summary.vehicles` | array | Detail par vehicule |

---

### Structure d'un vehicule

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

| Champ | Type | Description |
|-------|------|-------------|
| `vehicle_id` | int | Numero du vehicule (commence a 1) |
| `steps` | array | Liste ordonnee des etapes de la tournee |
| `metrics.total_distance_meters` | int | Distance totale en metres |
| `metrics.total_time_seconds` | int | Duree totale en secondes |
| `metrics.total_clients` | int | Nombre de clients desservis par ce vehicule |
| `route_geometry` | string[] | Polylignes encodees (format Google Encoded Polyline) pour le trace sur la carte |
| `total_shift_duration` | int | Duree totale du shift en secondes |
| `formatted_duration` | string | Duree formatee (ex : `"01h30"`) |

---

### Structure d'une etape (step)

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

| Champ | Type | Description |
|-------|------|-------------|
| `node_index` | int | Index interne du noeud dans le graphe du solveur |
| `label` | string | Nom de l'etape (ex : `"C3 (CAMION)"`, `"DEPOT"`, `"VIDAGE"`) |
| `kind` | string | Type de l'etape (voir tableau ci-dessous) |
| `coords` | [lon, lat] | Coordonnees au format [longitude, latitude] |
| `lat` | float | Latitude |
| `lon` | float | Longitude |
| `solver_cumulative` | int | Temps cumule depuis le depart en secondes (valeur brute du solveur) |
| `details.service_duration` | int | Duree de service a cette etape en secondes |
| `details.operation` | string | Description de l'operation (ex : `"ECHANGE Benne ciel ouvert 30m3"`) |
| `arrival_time_str` | string | Heure d'arrivee formatee depuis le depart (ex : `"1h30"`) |
| `cumul_distance_km` | float | Distance cumulee depuis le depart en kilometres |

---

### Types d'etapes (`kind`)

| Kind | Description |
|------|-------------|
| `DEPOT` | Depart ou retour au depot |
| `DEPOT_RESTOCK` | Reapprovisionnement au depot |
| `CLIENT_STANDARD` | Visite client standard (camion seul) |
| `CLIENT_DOUBLE` | Visite client avec remorque (mode LIFO) |
| `CLIENT_COMBO` | Visite client combo (camion pendant que la remorque est deposee ailleurs) |
| `CLIENT_RETURN_TRAILER` | Retour de benne vide chez le client (remorque) |
| `CLIENT_RETURN_COMBO` | Retour de benne vide chez le client (combo) |
| `DUMP_CAMION` | Vidage du contenu du camion au site de traitement |
| `DUMP_REMORQUE` | Vidage du contenu de la remorque au site de traitement |
| `HUB_VIRTUAL` | Passage a un hub/exutoire |
| `HUB_BIN_SWAP` | Echange de benne au hub |
| `TRAILER_DROP` | Depot de la remorque a un point intermediaire |
| `TRAILER_PICKUP` | Reprise de la remorque a un point intermediaire |

---

### Codes d'erreur

| Code HTTP | Description |
|:---------:|-------------|
| `200` | Succes -- solution trouvee |
| `401` | Identifiants Bemap invalides |
| `500` | Erreur interne du serveur ou le solveur n'a pas produit de solution |
| `502` | Service Bemap indisponible |
| `504` | Timeout -- le solveur a depasse la limite de 15 minutes |

**Format d'erreur**

```json
{
  "detail": "Description de l'erreur"
}
```

Pour les erreurs 500 sans solution, le champ `detail` peut contenir un objet :

```json
{
  "error": "Solver produced no output",
  "log": "... derniers logs du solveur ..."
}
```

---

## Exemple complet avec curl

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
    "bemap_user": "mon.utilisateur",
    "bemap_password": "mon_mot_de_passe",
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

## Notes techniques

- **Timeout** : la requete expire apres 15 minutes maximum. Au-dela, une erreur 504 est renvoyee.

- **Coordonnees** : toutes les coordonnees sont au format **[longitude, latitude]** (standard GeoJSON). Ne pas confondre avec le format [latitude, longitude] utilise par certains outils.

- **Credentials Bemap** : les identifiants `bemap_user` et `bemap_password` sont utilises uniquement pour les appels de routage Bemap pendant le traitement de la requete. Ils ne sont jamais stockes cote serveur.

- **Matrice de distances** : le serveur calcule la matrice de distances/temps via l'API Bemap MODE_MATRIX (profil poids-lourd, hauteur 3m, poids 19t, largeur 2.3m) avant de la transmettre au solveur.

- **Itineraires** : apres l'optimisation, les itineraires reels sont calcules via l'API Bemap MODE_VIAS pour obtenir les polylignes de trace et les durees de trajet segment par segment, permettant un recalcul precis des heures d'arrivee.
