# Runbook — serveur Valheim chez Nitrado

> Décision du 13 septembre 2026 : hébergement géré chez Nitrado plutôt que VPS auto-hébergé.
> Voir [`03-decision.md`](03-decision.md) pour le raisonnement.

---

## 1. Commander

- Offre **Valheim**, boutique européenne : **13,09 € pour 30 jours**.
- **10 slots imposés**, Nitrado ne descend pas en dessous pour ce jeu. Le jeu plafonne de toute
  façon à 10 joueurs simultanés, donc rien n'est perdu.
- Durées possibles : 3, 30, 90 ou 365 jours. Remises : −10 % à 3 mois, −15 % à 6 mois,
  **−20 % à 12 mois**.
- Commencer par **30 jours**. Passer à un engagement plus long une fois le serveur éprouvé.

---

## 2. Configurer

Tout se fait dans le panel web, rubrique **Configuration → Server Options**.

### Crossplay

**Activé par défaut.** C'est ce qui permet à PS5, Switch 2 et Xbox de rejoindre. Ne pas le
désactiver.

Conséquence à connaître : **crossplay et mods BepInEx sont incompatibles.** Les clients console
ne peuvent pas installer de mods. Tant que le crossplay est actif, le serveur reste en vanilla.
Nitrado ne propose de toute façon que ValheimPlus en installation automatique.

### Mot de passe — trois règles, sinon le serveur refuse de démarrer

1. **Au moins 5 caractères.**
2. Il ne doit **pas être contenu** dans le nom du serveur, le nom du monde, ni la seed.
3. La comparaison est **insensible à la casse**.

En cas d'erreur, le serveur logue `Error bad password:` et s'arrête aussitôt. Ne pas chercher à
contourner par une variante orthographique : prendre un mot totalement différent du nom du
serveur.

### Administrateurs

Fichier `adminlist.txt`, un identifiant par ligne, au format `[Plateforme]_[UserID]`,
**sensible à la casse**. Exemple : `Steam_76561198012345678`.

Les mêmes règles valent pour `permittedlist.txt` (liste blanche) et `bannedlist.txt`.

---

## 3. Rejoindre la partie

Trois moyens, selon la plateforme :

| Moyen | Qui |
|---|---|
| Liste de serveurs, recherche par nom | toutes plateformes |
| **Code de partie à 6 chiffres** | toutes plateformes |
| IP publique + port | PC surtout |

### Le piège à connaître

**Le code de partie change à chaque redémarrage du serveur.**

Si le serveur redémarre — mise à jour, maintenance Nitrado, redémarrage manuel — l'ancien code
ne fonctionne plus. Récupérer le nouveau dans le panel et le rediffuser au groupe.

Conseil pratique : demander à tout le monde d'ajouter le serveur **par son nom** dans la liste
de serveurs plutôt que par code. Le nom, lui, ne change pas.

---

## 4. Sauvegardes — le seul vrai point de vigilance

Nitrado gère des sauvegardes, mais **elles vivent chez Nitrado**. Compte suspendu, litige de
facturation, incident chez eux : le monde part avec. Il faut une copie chez soi.

### Format des mondes en 1.0 — à lire avant de copier quoi que ce soit

Depuis la 1.0, un monde **n'est plus un couple de fichiers `.db` + `.fwl`**, c'est **un dossier** :

```
Valheim/worlds_local/<NomDuMonde>/
├── _main.N.fwl2      métadonnées
├── _main.N.db2       données du monde
├── _main.N.chunks    index des chunks
├── _main.N.ok        marqueur de fin d'écriture
└── 00_00__0_1.chunk  terrain, sur de nombreux fichiers
```

**Il faut copier le dossier entier.** Une copie partielle ne se chargera pas : le serveur
générera un monde neuf par-dessus le nom. C'est l'erreur classique, et elle est irréversible si
on n'a rien d'autre.

Bonne nouvelle : plusieurs générations (`N`) coexistent dans le dossier. Même si la copie attrape
la génération la plus récente en cours d'écriture, les précédentes sont complètes.

### Méthode simple — manuelle, une fois par semaine

1. Récupérer les identifiants FTP dans le panel Nitrado.
2. S'y connecter avec **FileZilla** ou **WinSCP**.
3. Aller dans `Valheim/worlds_local/` — attention, c'est **dans** le dossier `Valheim`, pas à la
   racine du serveur.
4. Télécharger le dossier du monde **en entier**, dans un répertoire daté sur sa machine.

Cinq minutes, aucune installation. Suffisant pour un groupe de cinq joueurs.

### Méthode automatique — si une machine tourne en permanence

Sur un PC allumé régulièrement, un NAS ou un Raspberry Pi, avec `lftp` :

```sh
#!/bin/sh
# Miroir daté du monde Valheim depuis Nitrado.
# Identifiants à récupérer dans le panel Nitrado, à ne jamais écrire en clair ici :
# les placer dans ~/.netrc avec des permissions 600.

set -eu

DEST="$HOME/valheim-backups/$(date +%Y-%m-%d)"
MONDE="NomDuMonde"

mkdir -p "$DEST"

lftp -e "set ftp:ssl-allow true; \
         mirror --verbose --parallel=2 \
           'Valheim/worlds_local/$MONDE' '$DEST/$MONDE'; \
         bye" \
     "ftp://$NITRADO_USER:$NITRADO_PASS@$NITRADO_HOST"

# Purge des copies de plus de 60 jours
find "$HOME/valheim-backups" -maxdepth 1 -type d -mtime +60 -exec rm -rf {} +
```

À planifier une fois par jour (cron sous Linux/macOS, Planificateur de tâches sous Windows).

**Ne jamais mettre les identifiants dans le script.** Les placer dans `~/.netrc` en permissions
`600`, ou dans des variables d'environnement chargées depuis un fichier `600`.

### Tester la restauration — au moins une fois

Une sauvegarde non testée est une hypothèse. Une fois, faire l'exercice :

1. Télécharger une copie.
2. La placer dans le dossier `worlds_local` de son propre client Valheim.
3. Vérifier que le monde se charge en solo, avec les constructions attendues.

Si ça marche en local, la copie est bonne.

---

## 5. Mises à jour

**Verrouillage de version strict** : le serveur et tous les clients doivent tourner exactement
sur le même build, sinon erreur *Incompatible Version*. Un serveur non mis à jour est hors
service pour tout le monde.

Nitrado met à jour automatiquement. Le jour d'un correctif Iron Gate, prévoir un court
redémarrage — et un nouveau code de partie.

Avant une mise à jour majeure, prendre une copie fraîche du monde selon la méthode ci-dessus.

---

## 6. Aide-mémoire

| Besoin | Où |
|---|---|
| Logs, redémarrage, état | Panel Nitrado |
| Crossplay, mot de passe, nom | Configuration → Server Options |
| Admins, bannis | `adminlist.txt`, `bannedlist.txt` |
| Fichiers du monde | FTP → `Valheim/worlds_local/<Monde>/` |
| Nouveau code de partie | Panel, après chaque redémarrage |

### Les trois choses à ne pas oublier

1. Le **code de partie change à chaque redémarrage** — faire ajouter le serveur par son nom.
2. Une sauvegarde, c'est **le dossier entier**, jamais des fichiers isolés.
3. Les sauvegardes Nitrado **ne sont pas les tiennes** — garder une copie chez soi.
