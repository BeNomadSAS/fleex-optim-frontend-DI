# Fleex Optim Route Planner — Guide utilisateur (FR)

**URL d'accès :** <https://fleex-optim.benomad.net>

Fleex Optim Route Planner construit des tournées optimisées multi-véhicules pour collecter ou livrer des bennes auprès d'un ensemble de clients. L'application s'appuie sur le routing poids-lourd BeNomad et sur le solveur OptimCPP côté backend.

---

## 1. Connexion

À l'ouverture, une fenêtre de connexion BeMap s'affiche.

1. Choisissez votre **environnement** : Beta / Préproduction / Production.
2. Saisissez votre **utilisateur** et **mot de passe** BeMap.
3. Cochez **Se souvenir de moi** si vous travaillez depuis un poste personnel — les identifiants sont stockés en local (base64) sur l'appareil.
4. Cliquez **Se connecter**.

Les identifiants sont vérifiés contre le service BeMap avant que l'application n'ouvre.

---

## 2. Le panneau de gauche — saisie

Les sections du panneau gauche apparaissent progressivement à mesure que vous remplissez les étapes (« progressive disclosure »). Si une section n'est pas encore visible, c'est qu'il manque une étape en amont.

### Étape 1 — Configuration

Indiquez le **nombre de véhicules** et, parmi eux, combien sont **équipés d'une remorque**. Validez avec **Continuer →**.

### Étape 2 — Dépôt

Cliquez sur **Placer sur la carte**, puis cliquez sur la carte pour déposer le dépôt. Le point est automatiquement vérifié sur le réseau routier : s'il tombe hors voirie, vous serez invité·e à le repositionner.

### Étape 3 — Nouveau client

Pour chaque client :

1. Choisissez l'**opération** : Échange / Aller-retour / Dépose / Retrait.
2. Sélectionnez la **taille de benne** (gérez la liste via *Gérer les tailles*).
3. Cliquez **Position client**, puis cliquez sur la carte pour déposer le marqueur du client.
4. Si l'opération l'exige (Échange / Aller-retour / Retrait), cliquez **Point de vidage** et indiquez le lieu de déchargement.
5. Cliquez **Ajouter ce client**.

Le client apparaît dans la liste **Clients** juste en dessous. Vous pouvez à tout moment masquer un client (œil) ou le supprimer (corbeille).

### Étape 4 — Lancer l'optimisation

Une fois au moins un client ajouté, le bouton **Lancer l'optimisation** devient cliquable. L'optimisation se déroule en 3 phases : matrice de routing → solveur → rendu des tracés.

---

## 3. Le panneau de droite — résultats

À la fin de l'optimisation, le panneau de droite se déplie automatiquement et présente :

- **Synthèse globale** : nombre de véhicules, clients servis, km parcourus, durée totale, volume collecté (m³).
- **Carte par véhicule** (« vcard ») coloriée d'une teinte distincte. Chaque carte donne :
    - les métriques du tour (distance, durée, volume) ;
    - un bouton **œil** pour afficher / masquer le tracé sur la carte ;
    - un menu d'**export** (CSV / JSON, PDF et BeNav à venir) ;
    - la liste détaillée des étapes (DEPOT → clients → vidages → retour) avec, à droite de chaque étape client, la charge ajoutée et la charge cumulée. Les étapes de vidage indiquent une remise à zéro `(X → 0)`.

Cliquez sur une étape pour recentrer la carte sur ce point.

Le bouton **Exporter CSV** en tête de panneau exporte l'ensemble des tournées en un seul fichier.

---

## 4. Import / Export

Le panneau gauche contient également une section **Import données** :

- **Importer un CSV** : colonnes minimales `x_client,y_client` (les noms `lng,lat` ou `lon,lat` sont également acceptés ; les délimiteurs `,` et `;` sont détectés automatiquement).
- **Importer JSON** : collez une liste de clients au format JSON.
- Cliquez sur **CSV exemple** ou **JSON exemple** pour télécharger un fichier modèle.

---

## 5. Thème, langue, guide

- **Thème clair / sombre** : bouton en haut du panneau gauche. Respecte la préférence système au premier lancement.
- **Langue** : sélecteur en haut du panneau. Le choix est persistant.
- **Guide pas-à-pas** : si vous êtes débutant·e, gardez le panneau « Guide » ouvert — il met en évidence la prochaine étape. Cliquez sur la croix **×** pour le fermer définitivement (l'application retient le choix sur ce navigateur).

---

## 6. Support

Pour les identifiants BeMap, l'intégration ou les questions de licence, contactez votre interlocuteur·rice BeNomad.
