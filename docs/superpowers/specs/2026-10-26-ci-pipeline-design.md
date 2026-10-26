# Pipeline CI (GitHub Actions)

## Objectif

Détecter automatiquement les régressions à chaque modification du dépôt
(code Rallly vendored dans `rallly/`, feature « vote réservé OIDC », stack
`docker-compose.yml`). Périmètre : **CI seule**, sans publication d'image ni
déploiement.

## Contexte

- Dépôt GitHub : `GabsBaquie/Groupe-de-4---Docker-m2`.
- `rallly/.github/workflows/` contient les workflows amont de Rallly. GitHub ne
  lit que `.github/workflows/` à la racine : ils ne s'exécutent donc pas.
- Les actions composites `rallly/.github/actions/*` supposent un `package.json`
  et un lockfile à la racine du workspace : **non utilisables** ici (ils sont
  dans `rallly/`). Les étapes setup/install sont inline.
- pnpm est épinglé via `packageManager` (12.14.0) ; `pnpm/action-setup` le lit.

## Design

Fichiers : `.github/workflows/ci.yml` et `docker-compose.ci.yml` (override qui
démarre l'image `rallly-ci:latest` déjà construite avec cache).

**Déclencheurs** : `pull_request` vers `main`, `push` sur `main`. `concurrency`
par PR/ref ; `cancel-in-progress` seulement pour les PR (un push sur `main` va toujours au bout). Permissions : `contents: read`.
**Aucun secret** : seul le `GITHUB_TOKEN` implicite est utilisé.

**Jobs (parallèles)** :

| Job | Contenu |
|---|---|
| `quality` | install `--frozen-lockfile`, `prisma generate`, `type-check`, `turbo check` (Biome), `check:structure`, `check:locales` |
| `unit-tests` | `prisma generate`, `test:unit` (tous les tests Vitest du workspace) |
| `compose-config` | `.env` construit depuis `.env.example` (valeurs vides remplies), `docker compose config -q`, échec si une variable du compose est absente de `.env.example` |
| `docker-smoke` | `.env` factice, image construite par `docker/build-push-action`, puis `docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d --wait rallly` (sans `--build`), `curl /api/status` avec `database == "connected"`, `curl /login`, logs si échec ; cache de couches buildx (`gha`) |

Les commandes `pnpm` s'exécutent dans `rallly/` (`working-directory`).
Node 24. Authentik n'est pas démarré dans `docker-smoke` ; les variables
`OIDC_*` y sont vides (on teste le démarrage, pas la connexion).

**Blocage du merge** : réglage manuel GitHub — protection de `main` avec les 4
jobs en *required status checks*, après un premier run vert.

## Hors périmètre

Playwright e2e (dont un test du vote refusé sans session OIDC), `check:openapi`,
publication ghcr.io, déploiement. Phase 2 éventuelle.

## Phase 2 prévue : Authentik dans la CI

Ajout compatible avec le design ci-dessus (nouveau job `e2e-oidc`, séparé pour
ne pas ralentir les jobs rapides ; démarrage d'Authentik ~1-2 min) :

- Démarrer `authentik-db`, `authentik-server`, `authentik-worker` avec
  `AUTHENTIK_SECRET_KEY` / `AUTHENTIK_DB_PASSWORD` factices (aucun secret GitHub).
- **Prérequis** : provisionner Authentik par script (blueprint YAML ou API) :
  provider OAuth2/OIDC, application de slug `rallly`, utilisateur de test avec
  nom et email. Aujourd'hui fait à la main.
- Test Playwright : vote refusé sans session, connexion via Authentik, vote
  enregistré avec nom/email de l'IdP.
- À vérifier : résolution de `authentik.localhost` par Chromium sur le runner.

## Risques

- Build Docker long au premier run (cache ensuite).
- Le premier run peut révéler des échecs préexistants côté amont : les corriger
  ou les exclure explicitement, sans désactiver silencieusement un check.

## Critères de réussite

- Une PR qui casse le type-check, le lint ou un test unitaire fait échouer la CI.
- Une PR qui rend `docker-compose.yml` invalide ou une image qui ne démarre pas
  fait échouer la CI.
- Le workflow ne demande aucun secret GitHub.
