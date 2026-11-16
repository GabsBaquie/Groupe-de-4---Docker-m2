# Groupe-de-4---Docker-m2

## Reverse proxy

`docker compose up -d` démarre aussi un proxy Caddy (`proxy/Caddyfile`), point
d'entrée par nom d'hôte vers Rallly et Authentik :

- http://rallly.localhost — l'application
- http://authentik.localhost — l'IdP

`*.localhost` résout automatiquement en loopback dans tous les navigateurs
modernes (RFC 6761) : aucune configuration DNS/`/etc/hosts` nécessaire.

Les ports directs (`3000`, `9000`) restent ouverts en parallèle du proxy — la
CI (`.github/workflows/ci.yml`) et les scripts ci-dessous s'appuient encore
dessus ; seule la base de données n'est jamais exposée.

## Vote réservé aux utilisateurs OIDC (Authentik)

Avec `REQUIRE_OIDC_TO_VOTE=true`, il n'est plus possible de voter en invité :
sans session OIDC le formulaire de vote est remplacé par un bouton de
connexion, et le serveur refuse la réponse (`authRequired`). Une fois connecté,
le nom et l'email fournis par l'IdP sont enregistrés sur le vote (non modifiables).

Mise en place :

1. Renseigner `.env` (voir `.env.example`) : `OIDC_CLIENT_SECRET` (`openssl rand -hex 32`),
   `AUTHENTIK_SECRET_KEY` et `AUTHENTIK_DB_PASSWORD`, puis `docker compose up -d`.
   Au démarrage, Authentik crée tout seul le provider OIDC et l'application `rallly`
   (blueprint `authentik/blueprints/rallly.yaml`, redirect URI
   `http://localhost:3000/api/auth/callback/oidc`).
2. Ouvrir http://authentik.localhost:9000/if/flow/initial-setup/ pour créer l'admin,
   puis créer tes utilisateurs (*Nom* et *Email* renseignés : ils s'affichent sur le vote).

Si tu avais créé le provider à la main avec le même `client_id`, supprime-le d'abord :
le blueprint échouerait (erreur loguée par le worker) et ton ancien provider resterait en place.

## Déploiement sur une VM

La CI (`.github/workflows/publish.yml`, à chaque push sur `main`) construit et publie
sur GHCR : les images `rallly`, `rallly-authentik` (blueprints embarqués) et
`rallly-proxy` (Caddyfile embarqué), ainsi que `docker-compose.yml` lui-même
(`ghcr.io/gabsbaquie/rallly-stack`). La VM n'a besoin ni du repo ni du code source :
seulement d'un `.env` et de Docker Compose ≥ 2.34.

Une fois, après le premier run du workflow : GitHub → Packages → `rallly`,
`rallly-authentik`, `rallly-proxy`, `rallly-stack` → Package settings →
Change visibility → Public (sinon `docker login ghcr.io` sur la VM).

Sur la VM, dans un dossier vide :

```bash
curl -fsSLo .env https://raw.githubusercontent.com/GabsBaquie/Groupe-de-4---Docker-m2/main/.env.example
vi .env   # avec <IP> = IP de la VM :
#   RALLLY_URL=http://<IP>:3000
#   OIDC_DISCOVERY_URL=http://<IP>:9000/application/o/rallly/.well-known/openid-configuration
#   + secrets (SECRET_PASSWORD, POSTGRES_PASSWORD, AUTHENTIK_*, OIDC_CLIENT_SECRET)
export COMPOSE_FILE=oci://ghcr.io/gabsbaquie/rallly-stack:latest
docker compose up -d
```

Mise à jour : `docker compose pull && docker compose up -d`. Retour en arrière :
`COMPOSE_FILE=oci://ghcr.io/gabsbaquie/rallly-stack:sha-<commit>`. Les données
(volumes Docker) sont conservées. Mailpit (8025) n'est pas à exposer publiquement.
En local, `docker compose up -d --build` reconstruit toutes les images depuis le repo.

## CI

`.github/workflows/ci.yml` s'exécute sur chaque PR vers `main` et chaque push
sur `main` : `Quality` (type-check, Biome, structure des features, locales),
`Unit tests`, `Compose config`, `Docker smoke test` et `E2E OIDC (Authentik)`
(Playwright contre une vraie stack Rallly + Authentik : un invité ne peut pas voter,
un utilisateur OIDC vote sous son nom et email IdP). Aucun secret GitHub
n'est nécessaire.

Protéger `main` : GitHub → Settings → Branches → Add rule sur `main` →
*Require status checks to pass* → cocher les 5 jobs ci-dessus (ils n'apparaissent
qu'après un premier run).
