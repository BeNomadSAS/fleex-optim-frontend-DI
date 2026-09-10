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
  "config_api": {
    "use_api": true,
    "api_geoserver": "osm",
    "api_url": "https://bemap-beta.benomad.com/bgis/service/routing/1.0",
    "api_key": "Basic <base64(identifiant:motdepasse)>",
    "api_timeout_seconds": 60
  }
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
| `config_api` | object | **oui** | -- | Cible Bemap : environnement vise + identifiants (voir ci-dessous) |

#### Objet `config_api`

Le serveur ne choisit plus l'environnement Bemap : c'est l'appelant qui le
fournit, avec les identifiants correspondants. Un compte valide sur un
environnement ne l'est pas forcement sur un autre.

| Champ | Type | Requis | Defaut | Description |
|-------|------|:------:|--------|-------------|
| `api_url` | string | **oui** | -- | Endpoint routing de l'environnement vise. Tout host hors du domaine `benomad.com` est refuse (400) |
| `api_key` | string | **oui** | -- | En-tete d'authentification pret a l'emploi : `"Basic " + base64(identifiant:motdepasse)` |
| `use_api` | bool | non | `true` | Utilisation de l'API Bemap pour la matrice |
| `api_geoserver` | string | non | (defaut serveur) | Reseau routier, ex. `"osm"`. Applique a la matrice ET aux itineraires |
| `api_timeout_seconds` | number | non | `60` | Timeout des appels Bemap, en secondes |

Environnements disponibles :

| Env | `api_url` |
|---------|-----------|
| dev | `http://bemap-dev.int.benomad.com/bgis/service/routing/1.0` |
| beta | `https://bemap-beta.benomad.com/bgis/service/routing/1.0` |
| preprod | `https://bemap-preprod.benomad.com/bgis/service/routing/1.0` |
| prod | `https://bemap.benomad.com/bgis/service/routing/1.0` |

Le schema `http://` n'est accepte que pour les hosts internes
(`*.int.benomad.com`). Ailleurs il est refuse, pour ne pas transmettre les
identifiants Basic en clair.

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

**Hub par defaut :** lorsque `hubs` est omis ou vide, l'endpoint `/optimize` **injecte automatiquement un hub virtuel aux coordonnees du depot** avec un `allowed_types` couvrant l'ensemble des `size_m3` distincts presents dans les jobs. Pour desactiver ce comportement, passez explicitement un tableau `hubs` (meme avec une seule entree realiste).

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

| Champ | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` si le solveur a trouve une solution |
| `run_id` | string | Identifiant unique de l'execution (8 caracteres hex) |
| `solution` | object | Solution complete (voir ci-dessous) |
| `solver_log` | string | Logs du solveur C++ (3000 derniers caracteres) |
| `timing.solver_seconds` | float | Temps passe dans le solveur C++ |
| `timing.total_seconds` | float | Temps total (matrice Bemap + solveur + routing + post-traitement) |

---

### Structure de la solution

| Champ | Type | Description |
|-------|------|-------------|
| `solution.status` | string | `"ok"` si une solution a ete trouvee |
| `solution.optimize_by` | string | Critere utilise : `"time"` ou `"distance"` |
| `solution.summary.vehicles` | array | Detail par vehicule (voir ci-dessous) |
| `solution.summary.total_time_min` | int / float | Temps cumule (valeur brute solveur) en minutes |
| `solution.summary.total_distance_km` | float | Distance cumulee brute du solveur. **Note :** le solveur C++ emet actuellement la valeur non-ponderee ici — pour le total recalibre Bemap, sommez `summary.vehicles[].metrics.total_distance_meters` |
| `solution.summary.dropped_jobs` | int[] | IDs des clients que le solveur n'a pas pu assigner (vide quand toutes les jobs sont desservies) |

---

### Structure d'un vehicule

```json
{
  "vehicle_id": 1,
  "steps": [ ... ],
  "metrics": {
    "total_distance_meters": 12180,
    "total_time_seconds": 5325,
    "total_clients": 2
  },
  "route_geometry": ["polyline_leg_1", "polyline_leg_2", "..."],
  "total_shift_duration": 5325,
  "formatted_duration": "1h28"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `vehicle_id` | int | Numero du vehicule (commence a 1) |
| `steps` | array | Liste ordonnee des etapes de la tournee |
| `metrics.total_distance_meters` | int | Distance totale en metres, recalibree depuis Bemap |
| `metrics.total_time_seconds` | int | Duree totale (deplacement + service) en secondes, recalibree depuis Bemap |
| `metrics.total_clients` | int | Nombre de clients desservis par ce vehicule |
| `route_geometry` | string[] | **Tableau** de polylignes encodees (format Google Encoded Polyline, precision 5) — une entree par leg entre deux etapes consecutives. Concatenez pour tracer toute la tournee |
| `total_shift_duration` | int | Duree totale du shift en secondes |
| `formatted_duration` | string | Duree formatee — heure sans padding zero, ex : `"1h28"`, `"0h59"` |

---

### Structure d'une etape (step)

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

| Champ | Type | Description |
|-------|------|-------------|
| `node_index` | int | Index interne du noeud dans le graphe du solveur. **`-1`** pour les etapes synthetiques injectees par l'API entre les sorties du solveur (ex. `VIDAGE (C2)`, `RETOUR BENNE (C2)`) — elles ne correspondent pas a un vrai noeud du graphe |
| `label` | string | Nom de l'etape (ex : `"C3"`, `"DEPOT"`, `"VIDAGE (C2)"`, `"RETOUR"`) |
| `kind` | string | Type de l'etape (voir tableau ci-dessous) |
| `coords` | [lon, lat] | Coordonnees au format `[longitude, latitude]` |
| `lat` | float | Latitude |
| `lon` | float | Longitude |
| `solver_cumulative` | int | Temps cumule depuis le depart en secondes (valeur brute du solveur, avant recalibrage Bemap applique par `arrival_time_str` / `cumul_distance_km`) |
| `details.service_duration` | int | Duree de service a cette etape en secondes |
| `details.operation` | string | Description de l'operation — pour les clients inclut le label de benne (ex : `"ECHANGE Benne ciel ouvert 10m3"`), pour les vidages `"VIDAGE"`, pour le depart/retour `"DEPART"` / `"FIN"` |
| `arrival_time_str` | string | Heure d'arrivee formatee depuis le depart — heure sans padding zero, ex : `"0h05"`, `"1h28"` |
| `distance_to_next` | int | **Valeur brute du solveur, non recalibree Bemap.** Souvent `0` ; utilisez plutot `cumul_distance_km` |
| `distance_km` | float | **Valeur brute du solveur, non recalibree Bemap.** Souvent `0` ; utilisez plutot `cumul_distance_km` |
| `cumul_distance_km` | float | Distance cumulee depuis le depart en kilometres, **recalibree par le post-traitement** depuis les vrais legs Bemap |

---

### Types d'etapes (`kind`)

Liste canonique emise par le solveur C++ actuel (`solver/main.cpp`) :

| Kind | Description |
|------|-------------|
| `DEPOT` | Depart ou retour au depot (premiere et derniere etape de chaque tournee) |
| `CLIENT_STANDARD` | Visite client standard, camion sans remorque |
| `CLIENT_PRIME` | Client primaire dans un pairing remorque (le plus gros volume — conserve sur la remorque) |
| `CLIENT_DOUBLE` | Client secondaire dans un pairing remorque (sur le camion — LIFO avec le primaire) |
| `CLIENT_COMBO` | Visite client combo — le camion collecte pendant que la remorque est deposee ailleurs |
| `CLIENT_RETURN` | Retour de benne vide chez un client mono-camion (apres le vidage) |
| `CLIENT_RETURN_TRAILER` | Retour de benne vide chez le client de la remorque |
| `CLIENT_RETURN_COMBO` | Retour de benne vide chez un client combo |
| `DUMP` | Vidage au site de traitement pour un flux mono-camion (sans remorque) |
| `DUMP_CAMION` | Vidage du contenu du camion au site de traitement (flux remorque) |
| `DUMP_REMORQUE` | Vidage du contenu de la remorque au site de traitement (flux remorque) |

---

### Codes d'erreur

| Code HTTP | Description |
|:---------:|-------------|
| `200` | Succes -- solution trouvee |
| `400` | `config_api` absent ou invalide : host hors domaine autorise, `api_key` manquant, ou `http://` vers un host non interne |
| `401` | Identifiants Bemap invalides |
| `500` | Erreur interne du serveur ou le solveur n'a pas produit de solution |
| `502` | Service Bemap indisponible, ou compte inconnu / sans droits sur l'environnement vise |
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
      "api_key": "Basic bW9uLnV0aWxpc2F0ZXVyOm1vbl9tb3RfZGVfcGFzc2U="
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

## Notes techniques

- **Timeout** : la requete expire apres 15 minutes maximum. Au-dela, une erreur 504 est renvoyee.

- **Coordonnees** : toutes les coordonnees sont au format **[longitude, latitude]** (standard GeoJSON). Ne pas confondre avec le format [latitude, longitude] utilise par certains outils.

- **Credentials Bemap** : `config_api.api_key` est utilise uniquement pour les appels de routage Bemap pendant le traitement de la requete. Il n'est jamais stocke cote serveur.

- **Environnement** : le serveur ne connait aucune URL Bemap en dur, il utilise `config_api.api_url` apres validation contre une liste blanche de hosts. C'est ce qui garantit que la matrice est calculee sur le meme environnement que celui ou l'utilisateur s'est connecte. Un compte valide sur beta mais absent de prod produit desormais une erreur explicite : Bemap repond a une requete non authentifiee par une redirection vers sa page de login (3xx), et non par un 401.

- **Matrice de distances** : le serveur calcule la matrice de distances/temps via l'API Bemap MODE_MATRIX (profil poids-lourd, hauteur 3m, poids 19t, largeur 2.3m) avant de la transmettre au solveur.

- **Itineraires** : apres l'optimisation, les itineraires reels sont calcules via l'API Bemap MODE_VIAS pour obtenir les polylignes de trace et les durees de trajet segment par segment, permettant un recalcul precis des heures d'arrivee.
