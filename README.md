# Groupe-de-4---Docker-m2

Features choisies :

1. Verrouiller l’identification duvotantpar OAuth
2. Votes anonymes
3. Bloquer le vote sur dates/heures passées

## Lancer le serveur

```bash
docker compose up -d        # démarre db + mailpit + rallly
docker compose ps           # vérifie que les 3 services sont "healthy"
docker compose logs -f rallly   # suivre les logs si besoin
```

- App : http://localhost:3000
- Mailpit (boîte mail de lab, pour récupérer les liens de connexion) : http://localhost:8025

Après une modification du code dans `rallly/`, il faut rebuilder l'image avant de relancer :

```bash
docker compose build rallly
docker compose up -d rallly
```

Pour arrêter la stack : `docker compose down` (les données Postgres restent dans le volume `db-data`).

## Feature 2 : Votes anonymes

Les votes des participants restent **toujours visibles** (le total "qui a voté oui/non" n'est jamais masqué), mais **l'identité** de chaque votant est masquée aux autres participants (réglage `hideParticipants` du sondage, déjà existant dans Rallly). C'est le **créateur du sondage qui décide, manuellement**, quand révéler qui a voté quoi.

**Implémentation** (branche `feature/votes-anonymes`, dans `rallly/`) :

- Champ `participantsRevealed` ajouté sur `Poll` — `packages/database/prisma/models/poll.prisma`, migrations `20261026120000_add_scores_revealed_to_poll` + `20261026130000_rename_scores_revealed_to_participants_revealed` (le champ a été renommé en cours de route après clarification du besoin)
- Masquage d'identité conditionné par ce champ, en plus du réglage `hideParticipants` existant — `apps/web/src/features/poll/utils.ts` (`maskParticipantsForViewer`, `filterCommentsForViewer`), branché depuis `apps/web/src/features/poll/loaders.ts`
- Mutation + action serveur réservées au créateur pour basculer le flag — `setParticipantsRevealed` / `setParticipantsRevealedAction` (`mutations.ts`, `schema.ts`, `actions.ts`)
- Bouton "Reveal identities" / "Hide identities again" dans la barre d'actions du sondage, visible uniquement si `hideParticipants` est actif — `apps/web/src/features/poll/components/reveal-participants-toggle.tsx`

Vérifié de bout en bout sur la stack locale : build Docker (type-check Next.js inclus), migrations appliquées automatiquement au démarrage du conteneur.

Remarque : un premier essai avait branché ce mécanisme sur `hideScores` (masquage du total des votes). Après retour du formateur, le bon réglage est `hideParticipants` : le total reste visible, seule l'identité est cachée jusqu'à révélation par le créateur.