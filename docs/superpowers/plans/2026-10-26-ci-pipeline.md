# Pipeline CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ajouter une CI GitHub Actions qui détecte les régressions (type-check, lint, tests unitaires, validité du compose, démarrage de l'image) sur chaque PR et push vers `main`.

**Architecture:** Un seul workflow racine `.github/workflows/ci.yml` avec 4 jobs parallèles, sans secret. Les commandes pnpm s'exécutent dans `rallly/` (le dépôt racine n'a ni `package.json` ni lockfile). Le smoke test construit l'image avec `docker/build-push-action` (cache `gha`) puis la démarre via un override compose `docker-compose.ci.yml`.

**Tech Stack:** GitHub Actions, pnpm 12.14.0 (via `packageManager`), Node 24, Docker Compose, Biome, Turbo, Vitest.

**Spec:** `docs/superpowers/specs/2026-10-26-ci-pipeline-design.md`

## Global Constraints

- Aucun secret GitHub : seul le `GITHUB_TOKEN` implicite est utilisé ; permissions `contents: read`.
- Déclencheurs : `pull_request` vers `main` et `push` sur `main`.
- `concurrency` par PR/ref avec `cancel-in-progress: true`.
- Node 24 ; pnpm lu depuis `rallly/package.json` (`packageManager` = `pnpm@12.14.0`).
- Pas de `.env` réel en CI : valeurs factices générées à la volée, jamais les valeurs du `.env` local.
- Les composite actions `rallly/.github/actions/*` ne sont **pas** utilisables (elles supposent `package.json` et lockfile à la racine du workspace) : les étapes setup/install sont inline avec `package_json_file: rallly/package.json` et `cache-dependency-path: rallly/pnpm-lock.yaml`.
- Aucun check n'est désactivé silencieusement : un échec préexistant se corrige ou s'exclut explicitement, avec un commentaire.

## Review Focus

- Le `.env` est absent sur le runner (gitignoré) : `docker compose config` et `up` doivent marcher avec le `.env` factice seul.
- `SECRET_PASSWORD` doit faire au moins 32 caractères (`env.ts`), sinon Rallly crashe au démarrage et le smoke test échoue pour une mauvaise raison.
- `/api/status` répond 200 même si la base est injoignable : le test doit vérifier `database == "connected"`.
- Le lockfile pnpm 12 est multi-documents : un pnpm < 12 échoue (`BROKEN_LOCKFILE`) ; `pnpm/action-setup` doit lire la bonne version.
- `REQUIRE_OIDC_TO_VOTE=true` par défaut avec `OIDC_*` vides : l'app doit quand même démarrer et `/login` répondre.

## File Structure

- Create: `.github/workflows/ci.yml` — les 4 jobs.
- Create: `docker-compose.ci.yml` — override : utilise l'image préconstruite `rallly-ci:latest`.
- Modify: `README.md` — section CI + protection de branche.
- Modify: `CLAUDE.md` — mention de la CI.
- Modify: `docs/superpowers/specs/2026-10-26-ci-pipeline-design.md` — composite actions remplacées par étapes inline, ajout de l'override compose.

Commande pnpm locale (corepack est cassé sur cette machine) :
`PNPM="node $HOME/.cache/node/corepack/v1/pnpm/12.14.0/bin/pnpm.mjs"`

---

### Task 1: Workflow avec jobs `quality` et `unit-tests`

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: workflow `CI` avec les jobs `quality` et `unit-tests` ; les tâches suivantes ajoutent des jobs sous `jobs:` dans ce fichier.

- [ ] **Step 1: Vérifier que les commandes passent en local (référence verte)**

Run (depuis `rallly/`) :
```bash
PNPM="node $HOME/.cache/node/corepack/v1/pnpm/12.14.0/bin/pnpm.mjs"
$PNPM type-check && $PNPM turbo check && $PNPM check:structure && $PNPM check:locales && $PNPM test:unit
```
Expected: tout passe. Si un check échoue pour une raison préexistante, le noter : il sera corrigé ou exclu explicitement avec commentaire dans le workflow.

- [ ] **Step 2: Créer le workflow**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    name: Quality
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: rallly
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v6
        with:
          package_json_file: rallly/package.json
      - uses: actions/setup-node@v4
        with:
          node-version: "24"
          cache: pnpm
          cache-dependency-path: rallly/pnpm-lock.yaml
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:generate
      - name: Type check
        run: pnpm type-check
      - name: Lint (Biome)
        run: pnpm turbo check
      - name: Feature folder structure
        run: pnpm check:structure
      - name: Locale files
        run: pnpm check:locales

  unit-tests:
    name: Unit tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: rallly
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v6
        with:
          package_json_file: rallly/package.json
      - uses: actions/setup-node@v4
        with:
          node-version: "24"
          cache: pnpm
          cache-dependency-path: rallly/pnpm-lock.yaml
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:generate
      - name: Run unit tests
        run: pnpm test:unit
```

- [ ] **Step 3: Valider la syntaxe YAML**

Run (depuis la racine du dépôt) :
```bash
python3 -c "import yaml,sys; d=yaml.safe_load(open('.github/workflows/ci.yml')); print(sorted(d['jobs']))"
```
Expected: `['quality', 'unit-tests']`

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: jobs quality et unit-tests"
```

---

### Task 2: Job `compose-config`

**Files:**
- Modify: `.github/workflows/ci.yml` (ajouter sous `jobs:`)

**Interfaces:**
- Consumes: le workflow de la Task 1.
- Produces: job `compose-config`.

- [ ] **Step 1: Vérifier que la commande échoue sans `.env` complet puis passe avec**

Run (depuis la racine) :
```bash
cd "$(mktemp -d)" && cp "$OLDPWD/docker-compose.yml" . && docker compose config -q 2>&1 | tail -3
```
Expected: avertissements « variable is not set » mais exit 0 (les variables vides sont tolérées). Le job doit donc fournir un `.env` factice pour avoir une config réaliste et attraper les vraies erreurs de YAML.

- [ ] **Step 2: Ajouter le job**

```yaml
  compose-config:
    name: Compose config
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Write dummy .env
        run: |
          cat > .env <<'EOF'
          SECRET_PASSWORD=0123456789abcdef0123456789abcdef
          POSTGRES_PASSWORD=ci-postgres
          SUPPORT_EMAIL=support@example.com
          AUTHENTIK_SECRET_KEY=ci-authentik-secret-key
          AUTHENTIK_DB_PASSWORD=ci-authentik-db
          OIDC_DISCOVERY_URL=http://authentik.localhost:9000/application/o/rallly/.well-known/openid-configuration
          OIDC_CLIENT_ID=ci
          OIDC_CLIENT_SECRET=ci
          REQUIRE_OIDC_TO_VOTE=true
          EOF
      - name: Validate docker-compose.yml
        run: docker compose config -q
```

- [ ] **Step 3: Valider localement avec le même `.env` factice**

Run (dossier temporaire) :
```bash
d=$(mktemp -d) && cp docker-compose.yml "$d/" && cd "$d" && printf 'SECRET_PASSWORD=0123456789abcdef0123456789abcdef\nPOSTGRES_PASSWORD=x\nSUPPORT_EMAIL=a@b.c\nAUTHENTIK_SECRET_KEY=x\nAUTHENTIK_DB_PASSWORD=x\nOIDC_DISCOVERY_URL=http://authentik.localhost:9000/x\nOIDC_CLIENT_ID=c\nOIDC_CLIENT_SECRET=s\n' > .env && mkdir rallly && docker compose config -q; echo "exit=$?"
```
Expected: `exit=0`. Puis casser volontairement l'indentation d'une copie de `docker-compose.yml` et vérifier `exit` non nul.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: validation de docker-compose.yml"
```

---

### Task 3: Job `docker-smoke` et override compose

**Files:**
- Create: `docker-compose.ci.yml`
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: image locale `rallly-ci:latest` ; job `docker-smoke`.

- [ ] **Step 1: Créer l'override compose**

```yaml
# Utilisé uniquement par la CI : démarre l'image déjà construite (avec cache)
# au lieu de la rebuilder via `build:`.
services:
  rallly:
    image: rallly-ci:latest
    pull_policy: never
```

- [ ] **Step 2: Ajouter le job**

```yaml
  docker-smoke:
    name: Docker smoke test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build Rallly image
        uses: docker/build-push-action@v6
        with:
          context: ./rallly
          file: ./rallly/apps/web/Dockerfile
          build-args: SELF_HOSTED=true
          tags: rallly-ci:latest
          load: true
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Write dummy .env
        run: |
          cat > .env <<'EOF'
          SECRET_PASSWORD=0123456789abcdef0123456789abcdef
          POSTGRES_PASSWORD=ci-postgres
          SUPPORT_EMAIL=support@example.com
          AUTHENTIK_SECRET_KEY=ci-authentik-secret-key
          AUTHENTIK_DB_PASSWORD=ci-authentik-db
          REQUIRE_OIDC_TO_VOTE=true
          EOF
      - name: Start Rallly and its database
        run: docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d --wait --wait-timeout 120 rallly
      # /api/status answers 200 even when the database is unreachable, so
      # assert the database field explicitly.
      - name: Check status endpoint and login page
        run: |
          status=$(curl -fsS --connect-timeout 5 --max-time 15 http://localhost:3000/api/status)
          echo "$status"
          echo "$status" | jq -e '.database == "connected"'
          curl -fsSL --connect-timeout 5 --max-time 15 -o /dev/null http://localhost:3000/login
      - name: Show logs on failure
        if: failure()
        run: docker compose -f docker-compose.yml -f docker-compose.ci.yml logs
```

- [ ] **Step 3: Vérifier la config fusionnée et la syntaxe**

Run (racine, `.env` local présent) :
```bash
docker compose -f docker-compose.yml -f docker-compose.ci.yml config | grep -A2 "image: rallly-ci"
python3 -c "import yaml; d=yaml.safe_load(open('.github/workflows/ci.yml')); print(sorted(d['jobs']))"
```
Expected: `image: rallly-ci:latest` avec `pull_policy: never` ; jobs `['compose-config', 'docker-smoke', 'quality', 'unit-tests']`.

- [ ] **Step 4: Rejouer le smoke test en local, comme la CI**

Run (racine) :
```bash
docker build -t rallly-ci:latest --build-arg SELF_HOSTED=true -f rallly/apps/web/Dockerfile rallly
docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d --wait --wait-timeout 120 rallly
curl -fsS http://localhost:3000/api/status | jq -e '.database == "connected"'
```
Expected: `true`. Puis `docker compose down`. (Utilise le `.env` local ; la logique est identique, seules les valeurs diffèrent.)

- [ ] **Step 5: Commit**

```bash
git add docker-compose.ci.yml .github/workflows/ci.yml
git commit -m "ci: smoke test de l'image Docker"
```

---

### Task 4: Documentation, spec et premier run

**Files:**
- Modify: `README.md`, `CLAUDE.md`, `docs/superpowers/specs/2026-10-26-ci-pipeline-design.md`

- [ ] **Step 1: Mettre à jour la spec**

Dans la spec : remplacer « réutilise les actions composites » par « étapes setup/install inline (les composites supposent un lockfile à la racine) » et ajouter `docker-compose.ci.yml` dans la liste des fichiers.

- [ ] **Step 2: Ajouter la section CI au README**

```markdown
## CI

`.github/workflows/ci.yml` s'exécute sur chaque PR vers `main` et chaque push
sur `main` : `Quality` (type-check, Biome, structure, locales), `Unit tests`,
`Compose config` et `Docker smoke test`. Aucun secret GitHub n'est nécessaire.

Protéger `main` : GitHub → Settings → Branches → Add rule sur `main` →
*Require status checks to pass* → cocher les 4 jobs ci-dessus (ils n'apparaissent
qu'après un premier run).
```

- [ ] **Step 3: Mentionner la CI dans `CLAUDE.md`**

Ajouter sous « Commands » : « CI : `.github/workflows/ci.yml` (jobs Quality, Unit tests, Compose config, Docker smoke test) ; commandes pnpm à lancer depuis `rallly/`. »

- [ ] **Step 4: Commit, push, ouvrir la PR**

```bash
git add README.md CLAUDE.md docs
git commit -m "docs: CI, protection de branche et mise à jour de la spec"
git push -u origin ci/pipeline
```
Puis ouvrir une PR vers `main` et attendre les 4 jobs. Les échecs préexistants se corrigent ou s'excluent explicitement (voir Global Constraints). Une fois les 4 jobs verts, activer la protection de branche (README).

- [ ] **Step 5: Vérifier que la CI détecte bien une régression**

Sur une branche jetable, introduire une erreur de type dans un fichier de `rallly/apps/web/src`, pousser, et constater que `Quality` échoue ; supprimer la branche ensuite.

---

## Self-Review

- **Spec coverage:** déclencheurs/concurrency/permissions (Task 1), `quality` et `unit-tests` (Task 1), `compose-config` (Task 2), `docker-smoke` avec cache buildx (Task 3), blocage du merge et docs (Task 4), phase 2 Authentik laissée hors plan comme dans la spec.
- **Placeholders:** aucun ; chaque étape contient la commande ou le YAML complet.
- **Cohérence:** image `rallly-ci:latest` identique dans le build, l'override et la vérification ; noms de jobs identiques dans le workflow, le README et les required checks.
- **Écart avec la spec:** composites remplacées par étapes inline et ajout de `docker-compose.ci.yml`, tous deux consignés en Task 4 Step 1.
