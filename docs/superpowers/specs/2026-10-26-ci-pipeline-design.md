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
- Les actions composites `rallly/.github/actions/{setup-node,pnpm-install}` sont
  réutilisables depuis la racine (`./rallly/.github/actions/...`).
- pnpm est épinglé via `packageManager` (12.14.0) ; `pnpm/action-setup` le lit.

## Design

Un seul fichier : `.github/workflows/ci.yml`.

**Déclencheurs** : `pull_request` vers `main`, `push` sur `main`. `concurrency`
par PR/ref avec `cancel-in-progress`. Permissions : `contents: read`.
**Aucun secret** : seul le `GITHUB_TOKEN` implicite est utilisé.

**Jobs (parallèles)** :

| Job | Contenu |
|---|---|
| `quality` | install `--frozen-lockfile`, `prisma generate`, `type-check`, `turbo check` (Biome), `check:structure`, `check:locales` |
| `unit-tests` | `prisma generate`, `test:unit` (inclut `voter-identity.test.ts`) |
| `compose-config` | `.env` factice depuis `.env.example`, `docker compose config -q` |
| `docker-smoke` | `.env` factice, `docker compose up -d --build --wait rallly`, `curl /api/status` avec `database == "connected"`, `curl /login`, logs si échec ; cache de couches buildx (`gha`) |

Les commandes `pnpm` s'exécutent dans `rallly/` (`working-directory`).
Node 24. Authentik n'est pas démarré dans `docker-smoke` ; les variables
`OIDC_*` y sont vides (on teste le démarrage, pas la connexion).

**Blocage du merge** : réglage manuel GitHub — protection de `main` avec les 4
jobs en *required status checks*, après un premier run vert.

## Hors périmètre

Playwright e2e (dont un test du vote refusé sans session OIDC), `check:openapi`,
publication ghcr.io, déploiement. Phase 2 éventuelle.

## Risques

- Build Docker long au premier run (cache ensuite).
- Le premier run peut révéler des échecs préexistants côté amont : les corriger
  ou les exclure explicitement, sans désactiver silencieusement un check.

## Critères de réussite

- Une PR qui casse le type-check, le lint ou un test unitaire fait échouer la CI.
- Une PR qui rend `docker-compose.yml` invalide ou une image qui ne démarre pas
  fait échouer la CI.
- Le workflow ne demande aucun secret GitHub.
