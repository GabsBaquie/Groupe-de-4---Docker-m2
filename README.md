# Groupe-de-4---Docker-m2

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

## CI

`.github/workflows/ci.yml` s'exécute sur chaque PR vers `main` et chaque push
sur `main` : `Quality` (type-check, Biome, structure des features, locales),
`Unit tests`, `Compose config` et `Docker smoke test`. Aucun secret GitHub
n'est nécessaire.

Protéger `main` : GitHub → Settings → Branches → Add rule sur `main` →
*Require status checks to pass* → cocher les 4 jobs ci-dessus (ils n'apparaissent
qu'après un premier run).
