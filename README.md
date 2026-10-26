# Groupe-de-4---Docker-m2

## Vote réservé aux utilisateurs OIDC (Authentik)

Avec `REQUIRE_OIDC_TO_VOTE=true`, il n'est plus possible de voter en invité :
sans session OIDC le formulaire de vote est remplacé par un bouton de
connexion, et le serveur refuse la réponse (`authRequired`). Une fois connecté,
le nom et l'email fournis par l'IdP sont enregistrés sur le vote (non modifiables).

Mise en place :

1. Renseigner `.env` (voir `.env.example`), puis `docker compose up -d`.
2. Ouvrir http://authentik.localhost:9000/if/flow/initial-setup/ pour créer l'admin.
3. Dans Authentik : créer un *Provider OAuth2/OpenID* + une *Application* de slug
   `rallly`, type client *Confidential*, redirect URI (strict)
   `http://localhost:3000/api/auth/callback/oidc`, scopes `openid email profile`.
4. Copier Client ID / Client Secret dans `.env` (`OIDC_CLIENT_ID`,
   `OIDC_CLIENT_SECRET`) et relancer : `docker compose up -d --force-recreate rallly`.

## CI

`.github/workflows/ci.yml` s'exécute sur chaque PR vers `main` et chaque push
sur `main` : `Quality` (type-check, Biome, structure des features, locales),
`Unit tests`, `Compose config` et `Docker smoke test`. Aucun secret GitHub
n'est nécessaire.

Protéger `main` : GitHub → Settings → Branches → Add rule sur `main` →
*Require status checks to pass* → cocher les 4 jobs ci-dessus (ils n'apparaissent
qu'après un premier run).
