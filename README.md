# sp-valheim

Serveur Valheim privé pour un groupe de 2 à 5 joueurs, en crossplay console.

## État : hébergement géré retenu, projet VPS écarté

Après la Phase 0, le projet d'auto-hébergement décrit dans le handoff a été **abandonné au
profit d'un hébergement géré chez Nitrado** (13,09 € / 30 jours).

Trois raisons : le coût est identique, le crossplay interdit les mods de toute façon — ce qui
retire le principal intérêt du contrôle total — et le panel Nitrado couvre déjà les besoins
fonctionnels du cahier des charges. Le raisonnement complet est dans
[`docs/03-decision.md`](docs/03-decision.md).

Le seul point qui demande un effort réel est conservé : **garder une copie des sauvegardes chez
soi**, les sauvegardes Nitrado vivant chez Nitrado.

## Cahier des charges

| Paramètre | Valeur |
|---|---|
| Effectif | 2 à 5 joueurs |
| Monde | Neuf |
| Joueurs | France / Benelux |
| Crossplay console | Oui (PS5 / Switch 2 / Xbox) |
| Mods | Non — impossible en crossplay de toute façon |
| Budget | ≤ 20 €/mois |

## Documents

| Fichier | Contenu |
|---|---|
| [`docs/02-runbook-nitrado.md`](docs/02-runbook-nitrado.md) | **À utiliser au quotidien.** Commande, configuration, connexion, sauvegardes, mises à jour |
| [`docs/03-decision.md`](docs/03-decision.md) | Pourquoi Nitrado plutôt que le VPS, et dans quels cas rouvrir le dossier |
| [`docs/01-phase-0-recherche.md`](docs/01-phase-0-recherche.md) | Recherche technique Valheim 1.0 et comparatif VPS chiffré. Reste valable si le sujet du VPS revenait |
| [`docs/00-handoff.md`](docs/00-handoff.md) | Cahier des charges d'origine, tel que reçu |

## Les trois choses à retenir

1. **Le code de partie à 6 chiffres change à chaque redémarrage du serveur.** Faire ajouter le
   serveur par son nom dans la liste, plutôt que par code.
2. **Une sauvegarde Valheim 1.0, c'est un dossier entier**, plus un couple de fichiers
   `.db` / `.fwl`. Une copie partielle ne se charge pas — le serveur génère un monde neuf
   par-dessus.
3. **Les sauvegardes Nitrado ne t'appartiennent pas.** Garder une copie chez soi.

## Règles

Aucun secret en clair dans ce dépôt : identifiants FTP, mots de passe serveur et clés restent en
dehors, dans des fichiers en permissions `600`.
