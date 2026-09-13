# sp-valheim

Serveur Valheim dédié privé sur VPS, avec panel d'administration accessible au seul propriétaire.

Ce dépôt est la **source de vérité** de l'infrastructure : chaque fichier de configuration présent
sur le VPS existe aussi ici. Si la machine disparaît, tout doit se rejouer depuis ce dépôt.

## Cahier des charges

| Paramètre | Valeur |
|---|---|
| Effectif | 2 à 5 joueurs |
| Monde | Neuf |
| Joueurs | France / Benelux |
| Crossplay console | Oui (PS5 / Switch 2 / Xbox) |
| Mods | Non |
| Budget VPS | ≤ 20 €/mois |

## Avancement

| Phase | État |
|---|---|
| 0 — Recherche et recommandation VPS | Terminée — [`docs/01-phase-0-recherche.md`](docs/01-phase-0-recherche.md) |
| 1 — Provisionnement et durcissement | En attente de validation du VPS |
| 2 — Serveur de jeu | À faire |
| 3 — Sauvegardes externes | À faire |
| 4 — Panel d'administration | À faire |
| 5 — Exploitation et runbook | À faire |

## Documents

- [`docs/00-handoff.md`](docs/00-handoff.md) — cahier des charges d'origine, tel que reçu
- [`docs/01-phase-0-recherche.md`](docs/01-phase-0-recherche.md) — valeurs techniques revérifiées,
  comparatif VPS chiffré, recommandation

## Règles

- Aucun secret en clair dans ce dépôt. Les mots de passe et clés vivent dans des fichiers `.env`
  en `600` sur le VPS, jamais versionnés.
- Chaque phase est validée par un test concret avant de passer à la suivante.
