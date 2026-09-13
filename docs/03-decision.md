# Décision — hébergement géré plutôt que VPS auto-hébergé

**Date :** 13 septembre 2026
**Retenu :** Nitrado, offre Valheim, 13,09 € pour 30 jours
**Écarté :** le projet VPS + panel décrit dans le handoff

---

## Ce qui a fait basculer la décision

### 1. Le coût est identique

| | Nitrado | netcup + projet |
|---|---|---|
| Hébergement | 13,09 € / 30 jours | 12,89 € / mois (engagement 12 mois) |
| Stockage des sauvegardes | inclus | ~1 € / mois |
| **Total** | **~13,09 €** | **~13,90 €** |

Le projet auto-hébergé revient légèrement plus cher, pas moins. L'argument économique, souvent
décisif pour l'auto-hébergement, ne s'applique pas ici.

### 2. Le crossplay interdit les mods de toute façon

Un serveur en crossplay ne peut pas charger de mods BepInEx : les clients console n'ont aucun
moyen de les installer. Or le crossplay console est une exigence du projet.

Cela retire l'argument le plus solide en faveur du contrôle total. « Je pourrai modder plus
tard » ne tient pas tant que des joueurs PS5 ou Switch 2 doivent pouvoir rejoindre.

### 3. Nitrado couvre déjà le cahier des charges fonctionnel

Le handoff demandait un panel offrant : logs en temps réel, redémarrage/arrêt/démarrage,
sauvegardes et restauration, état de santé, mise à jour du jeu, le tout consultable depuis un
téléphone. Le panel Nitrado fournit tout cela, immédiatement, sans développement.

### 4. Le matériel est meilleur pour cet usage

Nitrado déploie de l'**AMD EPYC 9474F** sur ses serveurs grand public, la gamme haute fréquence
(3,6 GHz de base), annoncée à +60 % en mono-cœur par rapport à la génération précédente. Le
netcup envisagé embarque un EPYC 9645 à 2,3 GHz de base — une puce généraliste.

Nuance : Nitrado partage cette machine entre de nombreux clients, là où les 4 cœurs netcup
étaient dédiés. L'avantage n'est donc pas absolu, mais il n'y a pas de sacrifice matériel.

### 5. Le temps

Dix minutes contre plusieurs soirées, puis zéro maintenance contre des mises à jour système,
de la surveillance et du dépannage à assumer dans la durée.

---

## Ce qui est perdu, et comment c'est compensé

| Perdu | Compensation |
|---|---|
| **Sauvegardes chez soi** | Copie FTP périodique du dossier du monde vers sa propre machine — voir le runbook. C'est le seul point qui demande un vrai effort. |
| Accès root, reproductibilité | Accepté. Sans besoin de mods ni de services annexes, ce contrôle ne sert pas. |
| Panel sur mesure | Le panel Nitrado remplit la même fonction. |
| Machine polyvalente | Accepté. Aucun autre service n'était prévu. |

Le point réellement important est le premier. Les sauvegardes Nitrado vivent chez Nitrado : un
compte suspendu, un litige de facturation ou un incident chez eux emporte le monde. Le runbook
décrit la procédure de copie, manuelle ou automatisée.

---

## Ce qui reste valable du travail de Phase 0

Le document [`01-phase-0-recherche.md`](01-phase-0-recherche.md) garde son intérêt
indépendamment de l'hébergeur retenu :

- Contraintes du mot de passe serveur, cause fréquente de refus de démarrage
- Format des sauvegardes en 1.0 — un dossier par monde, à copier intégralement
- Le code de partie change à chaque redémarrage
- Crossplay et mods mutuellement exclusifs
- Verrouillage strict de version entre serveur et clients
- Dimensionnement CPU et RAM, si le sujet du VPS revenait un jour

---

## Dans quels cas rouvrir le dossier

- **Abandon du crossplay** au profit du PC seul : les mods redeviennent possibles, et avec eux
  l'intérêt d'un serveur que l'on contrôle.
- **Insatisfaction des performances Nitrado** en heures de pointe, matériel partagé oblige.
- **Besoin d'héberger autre chose** : un second serveur de jeu, un service annexe.
- **Envie de monter l'infrastructure pour elle-même.** Raison parfaitement valable, mais qui
  doit être assumée comme telle et non déguisée en argument technique.

Dans tous ces cas, la Phase 0 est déjà faite : comparatif chiffré, offre retenue, dimensionnement
validé. Il suffirait de reprendre à la Phase 1.
