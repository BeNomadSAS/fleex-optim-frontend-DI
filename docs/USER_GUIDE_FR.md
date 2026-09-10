# Fleex Optim Route Planner — Guide utilisateur (FR)

**URL d'accès :** <https://fleex-optim.benomad.net>

Fleex Optim Route Planner construit des tournées optimisées multi-véhicules pour collecter ou livrer des bennes auprès d'un ensemble de clients. L'application s'appuie sur le routing poids-lourd BeNomad et sur le solveur OptimCPP côté backend.

---

## 1. Connexion

À l'ouverture, une fenêtre de connexion BeMap s'affiche.

1. Choisissez votre **environnement** : Beta / Préproduction / Production.
2. Saisissez votre **utilisateur** et **mot de passe** BeMap.
3. Cochez **Se souvenir de moi** si vous travaillez depuis un poste personnel — les identifiants sont stockés en local (obfusqués base64) sur l'appareil.
4. Cliquez **Se connecter**.

Les identifiants sont vérifiés contre le service BeMap avant que l'application n'ouvre.

### Changer d'environnement — le bouton Modifier

En haut du panneau gauche, une carte repliable **Configuration BeMap** affiche l'utilisateur et l'environnement actifs. Cliquez **Modifier** pour ré-ouvrir la fenêtre de connexion. Si vous changez d'environnement (par ex. beta → préproduction), l'application effectue une remise à zéro complète : dépôt, clients et résultats précédents sont effacés afin de repartir proprement sur le nouvel env. Un simple changement d'identifiants sur le même env (rotation de mot de passe) conserve l'état de travail.

---

## 2. Le panneau de gauche — saisie

Les sections du panneau gauche apparaissent progressivement à mesure que vous remplissez les étapes (« progressive disclosure »). Si une section n'est pas encore visible, c'est qu'il manque une étape en amont.

L'en-tête du panneau gauche porte aussi le **logo BeNomad** + nom du produit, et sur une seconde ligne : un badge de statut de connexion, le **sélecteur de langue** (menu déroulant FR / EN / IT / DE), le **bouton 📖 documentation** (qui ouvre la visionneuse intégrée), le **bouton thème clair / sombre** (🌙 / ☀️) et un chevron pour replier le panneau.

### Étape 1 — Configuration

Indiquez le **nombre de véhicules** et, parmi eux, combien sont **équipés d'une remorque**. Validez avec **Continuer →**.

### Étape 2 — Dépôt (auto-armé)

Dès que la section Dépôt apparaît, le bouton **Placer sur la carte** est **déjà armé** — vous verrez sa bordure pulser. **Cliquez directement sur la carte** pour déposer le dépôt ; pas besoin de cliquer le bouton d'abord.

Le point est vérifié sur le réseau routier : s'il tombe hors voirie, un toast s'affiche et le bouton reste armé pour ré-essayer. Pour repositionner le dépôt plus tard, cliquez **Placer sur la carte** afin de ré-armer.

### Étape 3 — Nouveau client (auto-progression)

Le formulaire avance automatiquement de proche en proche. Vous ne cliquez sur les boutons du formulaire que pour reprendre la main sur le flux par défaut.

1. Choisissez l'**opération** : Échange / Aller-retour / Dépose / Retrait.
2. Sélectionnez la **taille de benne** (gérez la liste via **Gérer les tailles** — voir *§ Bennes* plus bas).
3. **Position client** est auto-armée dès l'ouverture de la section → cliquez sur la carte.
4. Sur un géocodage vert :
   - Pour *Échange* / *Aller-retour* / *Retrait* → **Point de vidage** s'auto-arme. Si le vidage se fait sur un hub de recyclage, cochez **Le vidage est un hub** avant de cliquer la carte.
   - Pour *Dépose* → pas de vidage nécessaire ; le focus saute directement à **Ajouter ce client**.
5. Après le géocodage vert du vidage → le focus saute à **Ajouter ce client** avec une pulsation verte. Appuyez sur **Entrée** pour valider.

Après **Ajouter ce client**, le formulaire se réinitialise et **Position client** se ré-arme automatiquement pour le client suivant. Le flux boucle tant que vous ajoutez des clients.

#### Quand l'auto-progression vous rend la main

- Un échec de géocodage (hors voirie, pas de résultat, timeout) maintient le même bouton armé pour réessayer. Override manuel : cliquer un autre bouton (par ex. **Point de vidage** avant que le client soit placé) signale à la FSM que vous prenez la main — elle ne ré-armera plus en cas d'échec.
- Cliquer **Tout supprimer** OU **Lancer l'optimisation** interrompt l'auto-progression : un clic carte égaré n'ajoutera PAS de nouveau client. Cliquez **Position client** pour ré-engager.

Chaque client placé apparaît dans la liste **Clients**. Vous pouvez à tout moment masquer un client (œil — il ne sera pas inclus dans le prochain calcul) ou le supprimer (corbeille).

### Étape 3.5 — Paramètres avancés (facultatif)

Une section repliable **Paramètres avancés** au-dessus du formulaire permet de surcharger les **temps de service** (en minutes) par type d'opération — échange, aller-retour, dépose, retrait, vidage, opérations hub, et variantes remorque. **Réinitialiser** restaure les valeurs par défaut. Ces durées sont transmises au solveur et impactent la durée totale de chaque tournée.

### Étape 4 — Lancer l'optimisation

Une fois au moins un client ajouté, le bouton **Lancer l'optimisation** devient cliquable. Un **loader plein écran à 3 phases** s'affiche pendant l'appel :

1. **Étape 1 / 3** — matrice de routing poids-lourd (BeNomad).
2. **Étape 2 / 3** — solveur OR-Tools.
3. **Étape 3 / 3** — tracés des polylines + rendu des étapes.

---

## 3. Le panneau de droite — résultats

À la fin de l'optimisation, le panneau de droite se déplie automatiquement et présente :

- **Synthèse globale** : nombre de véhicules, clients servis, km parcourus, durée totale, volume collecté (m³).
- **Exporter CSV** en tête de panneau — exporte l'ensemble des tournées en un seul fichier.
- **Une vcard par véhicule**, coloriée d'une teinte distincte. Chaque carte donne :
    - les métriques du tour (distance, durée, volume) ;
    - un bouton **œil** pour afficher / masquer le tracé de ce véhicule sur la carte sans toucher aux autres ;
    - un menu d'**export** (CSV / JSON par véhicule, PDF et BeNav à venir) ;
    - la liste détaillée des étapes (DEPOT → clients → vidages → retour). Pour chaque étape client, la colonne de droite indique le **volume ajouté** à cet arrêt et la **charge cumulée**. Les étapes de vidage affichent une remise à zéro `(X → 0)` indiquant que le camion vient d'être vidé.

Cliquez sur une ligne d'étape pour **recentrer la carte** sur ce point.

L'en-tête contient les boutons **Tout afficher** / **Tout masquer** pour ouvrir ou refermer toutes les vcards d'un coup.

---

## 4. Replier les panneaux

Chaque panneau latéral porte un chevron dans son en-tête. Cliquez pour le replier **verticalement** — le panneau se réduit à une fine pastille de titre en haut, affichant logo + nom du produit (à gauche) ou « Résultats » (à droite) + un chevron ▼. La carte sous la pastille est entièrement révélée. Cliquez ▼ pour ré-ouvrir.

---

## 5. Import / Export

Le panneau gauche contient une section repliable **Import données**.

- **Importer un CSV** — colonnes minimales : `x_client, y_client` (les noms `lng, lat` et `lon, lat` sont également acceptés ; les délimiteurs `,` et `;` sont détectés automatiquement ; BOM retiré).
- **Importer JSON** — collez un tableau JSON de clients.
- **CSV exemple** / **JSON exemple** — téléchargez un modèle à éditer.

Les clients importés entrent dans la liste Clients avec la même forme que ceux saisis manuellement — ils participent au prochain calcul, et vous pouvez les masquer / supprimer.

---

## 6. Thème, langue, guide

- **Thème clair / sombre** — bouton lune/soleil dans l'en-tête du panneau. Respecte la préférence système au premier lancement ; le choix est ensuite mémorisé dans `localStorage` par navigateur.
- **Langue** — menu déroulant dans l'en-tête (FR / EN / IT / DE). Cliquez le drapeau actif pour ouvrir la liste, puis sélectionnez une langue : le changement est instantané sans rechargement. *Note : le choix vaut pour la session courante uniquement ; un rechargement de la page repart sur le français.*
- **Guide pas-à-pas** — si vous débutez, gardez le panneau « Guide » ouvert (collé en haut du panneau gauche). Il met en évidence la prochaine carte d'action. Cliquez la **×** pour le fermer définitivement — `localStorage` retient le choix par navigateur. Pour le restaurer, videz les données du site (DevTools → Application → Local Storage) ou ouvrez l'app dans un autre navigateur.

---

## 7. Visionneuse de documentation

Le bouton **📖** dans l'en-tête du panneau gauche ouvre la **visionneuse de documentation intégrée**. Une fenêtre modale s'affiche avec deux onglets — *Guide utilisateur* et *Référence API* — chacun disponible en **EN** et **FR**. Le bouton **Télécharger (.md)** dans le pied de la modale livre le fichier markdown brut du document affiché.

Le contenu est rendu côté client via `marked` (chargé à la demande pour ne pas alourdir le bundle initial). Les liens externes du document s'ouvrent dans un nouvel onglet.

---

## 8. Support

Pour les identifiants BeMap, l'intégration ou les questions de licence, contactez votre interlocuteur·rice BeNomad.
