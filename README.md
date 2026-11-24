# Transport SaaS

SaaS multi-compagnies de gestion pour compagnie de transport en commun (monolithe modulaire, ADR-001 à 004).

## Démarrer
```bash
npm install
npm test            # tests ; l'API exige PostgreSQL (voir ci-dessous)
npm run lint        # ESLint
npm run arch        # contrôle des frontières entre modules
npm run build
copy .env.example .env      # PowerShell ; sous Linux/macOS : cp .env.example .env
docker compose up --build   # http://localhost:8080 , API : /api/v1/health
```

## Structure
- `packages/shared` : types et validation partagés (zod), rôles.
- `apps/api` : NestJS. `src/modules/<module>` : un dossier par module métier, interface publique = `index.ts`.
- `apps/web` : React + Vite, PWA (service worker généré au build), espaces chargés à la demande. `npm run dev -w apps/web` (proxy vers l'API sur :3000). Thème dans `src/theme/tokens.css`, nom du produit dans `src/config.ts`.
- `infra` : configuration nginx (proxy inverse).

## Tests de l'API (PostgreSQL requis)
Sans PostgreSQL local, le plus simple est un conteneur jetable (port 5433 pour ne pas gêner une installation existante) :
```powershell
docker run --name pgtest -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d postgres:16
$env:TEST_ADMIN_URL = "postgres://postgres:postgres@localhost:5433/postgres"
npm test
```
Avec un PostgreSQL déjà installé, donne simplement son mot de passe dans `TEST_ADMIN_URL`.

Les tests d'intégration utilisent une vraie base (la RLS ne se simule pas). Par défaut :
`postgres://postgres:postgres@localhost:5432/postgres` (variable `TEST_ADMIN_URL`). La base `transport_test`
est recréée à chaque lancement.

## Démarrage de l'API
Au lancement du conteneur : migrations SQL (`apps/api/migrations`) → création du compte plateforme
(si `BOOTSTRAP_ADMIN_*` sont définis) → serveur. L'API se connecte avec l'utilisateur `transport_app`,
soumis à la RLS ; les migrations utilisent l'utilisateur propriétaire.

## Routes (étape 2)
`POST /api/v1/auth/login|refresh|logout`, `GET /auth/me`, `GET|POST /users`, `PUT /users/:id/roles`,
`PATCH /users/:id`, `GET|POST /tenants`, `GET|PATCH /tenants/:id`, `GET /health`.

## Routes (étape 3)
Clients : `GET /clients?q=&limit=&offset=`, `GET|PATCH /clients/:id`, `POST /clients`.
Lignes : `GET|POST /lignes`, `GET|PATCH /lignes/:id`, `POST /lignes/:id/arrets`, `PATCH|DELETE /lignes/:id/arrets/:stopId`,
`PUT /lignes/:id/tarifs` (crée ou met à jour un tarif), `PATCH /lignes/:id/tarifs/:fareId` (activer/désactiver).
Départs : `GET /departs?lineId=&from=&to=&limit=&offset=`, `GET|PATCH /departs/:id`, `POST /departs`.
Droits : lecture `clients:lire` / `lignes:lire`, écriture `clients:ecrire` / `lignes:ecrire` (matrice du PRD §3.7).

## Routes (étape 4)
Billetterie : `POST /billetterie/ventes` (201 créé, 200 vente rejouée), `GET /billetterie/departs` (places restantes),
`GET /billetterie/tickets`, `GET /billetterie/tickets/:id`, `POST /billetterie/tickets/:id/annulation` (justification obligatoire).
Contrôle : `POST /controle/validations` (`{qr}` ou `{number}`, `tripId` facultatif), `GET /controle/departs/:id`, `GET /controle/cles` (clés publiques).
Droits : vendre `billetterie:vendre`, lire `billetterie:lire`, annuler `billetterie:annuler` (directeur), valider `controle:valider`, `controle:lire`.

**Nouvelle variable obligatoire : `SIGNING_KEY_SECRET`** (32 caractères minimum). Elle chiffre les clés privées de signature des QR
stockées en base. Ne la changez jamais après les premières ventes (les clés deviendraient illisibles) et sauvegardez-la à part.

## Routes (étape 5 : postes et hors ligne)
Postes : `POST /postes/demandes` (depuis l'appareil), `GET /postes`, `POST /postes/:id/activation`, `POST /postes/:id/revocation`
(administration), `GET /postes/:id/etat`, `POST /postes/:id/plages` (réserve une plage de numéros).
Données de référence : `GET /billetterie/referentiel`, `GET /controle/referentiel` (`?from=&to=`).
Synchronisation : `POST /billetterie/synchronisation` (ventes), `POST /controle/synchronisation` (validations). Rejouables sans doublon.
Anomalies : `GET /anomalies`, `GET /anomalies/resume`, `PATCH /anomalies/:id` (directeur).
Paiement mobile saisi hors ligne : `POST /billetterie/tickets/:id/paiement-verifie` (directeur).

## Moteur hors ligne (`packages/offline`, `@transport/offline`)
Bibliothèque sans interface qui tourne sur l'appareil : clés Ed25519 (générées sur place, jamais envoyées), IndexedDB, plages de numéros,
vente et contrôle sans réseau, file de synchronisation. L'interface (étape 7) n'aura qu'à l'appeler :
`init` → `register` → (activation par un administrateur) → `refreshDevice` → `refreshReference` → `ensureRange`, puis `sellOffline` /
`validateOffline`, `status()` pour le bandeau d'état, `sync()` dès que le réseau revient.
Les tests (`apps/api/test/hors-ligne.e2e.spec.ts`) font tourner ce moteur sur un IndexedDB simulé contre la vraie API.

## Routes (étape 6 : tableau de bord, droit `reporting:lire` : directeur, exploitation, comptable)
`GET /tableau-de-bord?from=AAAA-MM-JJ&to=AAAA-MM-JJ&lineId=&granularity=jour|semaine|mois` : recettes, billets, places, annulations,
paiements à vérifier, part hors ligne, évolution sur la période précédente, série pour graphique, classement des lignes avec taux de
remplissage, répartition par moyen de paiement et par catégorie. Sans paramètre : les 30 derniers jours.
`GET /tableau-de-bord/departs` : fréquentation départ par départ (paginée).
`GET /tableau-de-bord/export` : journal des ventes en CSV (s'ouvre dans Excel).
