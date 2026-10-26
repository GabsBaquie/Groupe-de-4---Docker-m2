# Groupe-de-4---Docker-m2

## CI

`.github/workflows/ci.yml` s'exécute sur chaque PR vers `main` et chaque push
sur `main` : `Quality` (type-check, Biome, structure des features, locales),
`Unit tests`, `Compose config` et `Docker smoke test`. Aucun secret GitHub
n'est nécessaire.

Protéger `main` : GitHub → Settings → Branches → Add rule sur `main` →
*Require status checks to pass* → cocher les 4 jobs ci-dessus (ils n'apparaissent
qu'après un premier run).
