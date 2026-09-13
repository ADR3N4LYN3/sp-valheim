# Phase 0 — Recherche préalable

> Recherches menées le **13 septembre 2026**, soit 4 jours après la sortie de Valheim 1.0.
> Objectif : revalider chaque valeur technique du handoff avant d'exécuter la moindre commande,
> puis produire une recommandation d'infrastructure chiffrée.

## Contexte retenu

| Paramètre | Réponse du propriétaire |
|---|---|
| Budget | ≤ 20 €/mois, idéalement moins |
| Joueurs | France / Benelux |
| Effectif | 2 à 5 joueurs |
| Monde | Neuf |
| Crossplay console | **Oui** (PS5 / Switch 2 / Xbox) |
| Mods | Non |
| Nom de domaine | Non |
| Clé SSH existante | Non — à générer |

---

## 1. Valheim — valeurs vérifiées

| Élément | Valeur du handoff | Vérification | Statut |
|---|---|---|---|
| App ID SteamCMD (serveur dédié) | 896660 | 896660 | Confirmé |
| `SteamAppId` à exporter | 892970 | 892970 — c'est l'App ID **du jeu**, pas du serveur | Confirmé |
| Ports UDP | 2456, 2457, **2458** | Doc officielle : « the specified Port AND specified Port+1 », soit **2456–2457/UDP**. Le 2458 n'apparaît nulle part dans la documentation Iron Gate | **Corrigé** |
| Dépendances Linux | `libatomic1`, `libpulse0`, `libpulse-dev` | Confirmées. Ajouter `libc6` pour le backend crossplay. GLIBC ≥ 2.29, GLIBCXX ≥ 3.4.26 | Confirmé |
| Architecture | x86_64 uniquement | **Toujours x86_64 uniquement en 2026.** Aucun binaire ARM64 officiel ; les solutions ARM existantes reposent sur l'émulation box64 ou FEX | Confirmé |
| Chemin des sauvegardes | `~/.config/unity3d/IronGate/Valheim/worlds_local` | Confirmé. Les fichiers de permissions sont dans le dossier parent `~/.config/unity3d/IronGate/Valheim/` | Confirmé |
| Contraintes mot de passe | à vérifier | **≥ 5 caractères**, et ne doit **pas être contenu** dans le nom du serveur, le nom du monde ou la seed (comparaison **insensible à la casse**). Sinon le serveur logue `Error bad password:` et s'arrête immédiatement | Précisé |
| `adminlist.txt` / `bannedlist.txt` / `permittedlist.txt` | emplacement et syntaxe | Dans `~/.config/unity3d/IronGate/Valheim/`. Un identifiant par ligne, format `[Plateforme]_[UserID]`, **sensible à la casse** | Précisé |
| Joueurs simultanés | — | 1 à 10, **inchangé en 1.0** | Confirmé |

### Flags de lancement documentés en 1.0

`-name` · `-port` · `-world` · `-password` · `-savedir` · `-public 0|1` · `-logFile`
`-saveinterval <s>` · `-backups <n>` · `-backupshort <s>` · `-backuplong <s>`
`-crossplay` · `-instanceid` · `-preset <nom>` · `-modifier <clé> <valeur>` · `-setkey <clé>`

Valeurs par défaut utiles : `-saveinterval 1800`, `-backups 4`, `-backupshort 7200`, `-backuplong 43200`.

### État de la 1.0

- Sortie le **9 septembre 2026** à 12h55 UTC, fin de cinq ans d'accès anticipé.
- Hotfixes **1.0.10** (toutes plateformes) et **1.0.12** (Steam) le 11 septembre 2026.
- **Version lock strict** : serveur et clients doivent tourner exactement sur le même build,
  sinon erreur *Incompatible Version*. Un serveur non mis à jour est hors service pour tout le monde.
- Les mondes existants sont conservés ; le nouveau terrain n'apparaît que là où personne n'est passé.

### Crossplay — quatre conséquences concrètes

1. `-crossplay` bascule le serveur du backend **Steam** vers le backend **PlayFab**. C'est
   obligatoire pour que PS5, Switch 2 et Xbox puissent rejoindre.
2. **L'ouverture de ports n'est plus strictement nécessaire** : le trafic transite par le relais
   PlayFab au lieu d'une connexion IP directe. On garde malgré tout 2456–2457/UDP ouverts sur le
   VPS, ce qui permet aux joueurs PC la connexion directe par IP:port, plus rapide.
3. **Le code de partie à 6 chiffres change à chaque redémarrage du serveur.**
   → Exigence pour la Phase 4 : le panel doit extraire et afficher le code courant depuis les logs.
   Sans ça, chaque redémarrage rend le serveur introuvable pour les joueurs console.
4. Trois modes de connexion coexistent en crossplay : IP publique + port, code de partie, ou liste
   de serveurs.

---

## 2. Comparatif VPS

### Exclusions immédiates

| Écarté | Motif |
|---|---|
| Tout VPS ARM (Ampere, Hetzner CAX, Scaleway COPARM) | Binaire Valheim x86_64 uniquement. L'émulation box64/FEX est inadaptée à un serveur de jeu en production |
| **Hetzner** | Hausse tarifaire du **15 juin 2026** : CCX13 passé de 15,99 € à **42,99 €** (+169 %), CCX23 de 31,49 € à **85,99 €** (+173 %). La gamme CPX (AMD partagé) a quasiment doublé. Hors budget |
| **Scaleway** | PLAY2-NANO (2 vCPU / 4 Go) ≈ 20 €/mois HT, PRO2-XXS ≈ 40,95 €/mois HT. Rapport performance/prix sans intérêt sur ce cahier des charges |

### Offres retenues au comparatif

| Hébergeur | Offre | vCPU | CPU | RAM | Disque | Localisation | Prix mensuel |
|---|---|---|---|---|---|---|---|
| **netcup** | **RS 1000 G12** | **4 dédiés** | EPYC 9645 « Turin », 3,7 GHz crête | 8 Go DDR5 ECC | 256 Go NVMe | Nuremberg / Vienne / **Amsterdam** | 10,74 € HT ≈ **12,89 € TTC** |
| **RedHeberg** | **Game ULTRA** | 4 partagés | Ryzen 9 5950X, 4,9 GHz | 8 Go | 70 Go NVMe | **Paris** | **12,95 € TTC** |
| PulseHeberg | Performance Cloud | n.d. | Ryzen 9 9900X, 4,4 GHz (5,6 boost), DDR5 ECC | n.d. | NVMe local | France / Suisse | dès 6 € TTC — **grille non vérifiable** |
| OVHcloud | VPS-2 2027 | 4 | Intel ≈ 2,4 GHz. VPSBenchmarks : *Raw CPU Power* **E — 6,5/20** | 8 Go | 75 Go NVMe | Gravelines / Roubaix / Strasbourg | 7,21 € HT / **8,65 € TTC** |
| netcup | VPS 1000 G12 | 4 partagés | EPYC 9645 | 8 Go DDR5 ECC | 256 Go NVMe | idem netcup | 8,71 € HT ≈ 10,45 € TTC |
| RedHeberg | Game PRO | 2 partagés | Ryzen 9 5950X, 4,9 GHz | 6 Go | 50 Go NVMe | Paris | 8,95 € TTC |

Réserve honnête sur **PulseHeberg** : leur CPU est le meilleur du lot sur le papier
(Ryzen 9 9900X à 4,4 GHz de base, en France), mais leur site bloque tout accès automatisé
par un challenge navigateur. Impossible de récupérer la grille exacte (vCPU / RAM par palier).
S'il existe un palier 4 vCPU / 8 Go à moins de 15 €, cette offre devient le meilleur compromis
du comparatif : à vérifier manuellement.

Réserve sur **OVH** : anti-DDoS et bande passante illimitée excellents, prix le plus bas, mais
le CPU est noté E par VPSBenchmarks. C'est exactement le profil « beaucoup de cœurs lents »
que le handoff identifie comme cause de rubber-banding à l'exploration. Écarté pour cette raison.

---

## 3. Recommandation

**Principale — netcup RS 1000 G12, région Amsterdam : ≈ 12,89 €/mois TTC**
**Alternative — RedHeberg VPS Game ULTRA, Paris : 12,95 €/mois TTC**

Raisonnement :

1. Cœurs **dédiés** : la boucle de simulation Valheim est mono-thread ; un cœur dédié à 3,7 GHz
   tient un tick constant là où un vCPU partagé décroche dès qu'un voisin s'agite.
2. Amsterdam couvre France et Benelux, et **le crossplay fait de toute façon transiter le trafic
   par le relais PlayFab** — les ~10 ms d'écart avec Paris se noient dans ce saut.
3. 8 Go DDR5 ECC et 256 Go NVMe : large marge pour un monde qui ne rétrécira jamais, sans
   avoir à repayer dans six mois.
4. Société établie, SLA 99,9 %, filtrage DDoS 2 Tbit/s inclus, remboursement sous 30 jours.
5. À 12,89 € on reste dans la moitié basse du budget, ce qui couvre le stockage objet des
   sauvegardes (quelques centimes) et laisse la place à un passage en RS 2000 plus tard.

Prendre l'alternative RedHeberg si la priorité est le sol français, la latence minimale et un
hébergeur qui parle nativement serveur de jeu (profils anti-DDoS par jeu, sauvegardes
quotidiennes incluses, sans engagement). Contrepartie : vCPU partagés sur matériel grand public,
structure plus petite, pas de SLA publié.

### À vérifier au moment de la commande

- Frais d'installation éventuels chez netcup sur un engagement mensuel.
- Disponibilité effective du RS 1000 G12 en région Amsterdam.
- Grille exacte de PulseHeberg Performance Cloud (cf. réserve ci-dessus).

---

## Sources

- [A Guide to Dedicated Servers — Iron Gate / valheimgame.com](https://www.valheimgame.com/support/a-guide-to-dedicated-servers/)
- [Valheim 1.0: What Changes for Server Owners](https://www.gameserverkings.com/blog/valheim-1-0-what-changes-for-server-owners/)
- [Valheim 1.0 crossplay — join code et relais PlayFab](https://berrybyte.net/wiki/games/valheim/join-server)
- [Valheim 1.0 Crossplay Dedicated Server Setup: Startup Flags and Admin IDs](https://gamers.wiki/en/games/valheim/guides/valheim-1-0-crossplay-dedicated-server-setup-startup-flags-and-admin-ids)
- [Contraintes de mot de passe serveur](https://berrybyte.net/wiki/games/valheim/cant-connect)
- [ARM64 : émulation box64 requise](https://github.com/Gornius/valheim_box64)
- [Hetzner — Price Adjustment 15 June 2026](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/)
- [Hetzner cloud server price increases in 2026 — Northflank](https://northflank.com/blog/hetzner-cloud-server-price-increases)
- [OVHcloud VPS — gamme 2027](https://www.ovhcloud.com/fr/vps/)
- [VPS-2 2027 — VPSBenchmarks](https://www.vpsbenchmarks.com/hosters/ovhcloud_us/plans/vps-2-2027)
- [Scaleway — tarifs Instances](https://www.scaleway.com/en/pricing/virtual-instances/)
- [netcup — tarifs 2026 VPS et Root Server G12](https://netcupvoucher.com/blog/netcup-pricing-2026)
- [RedHeberg — VPS Game Ryzen 9 5950X](https://redheberg.fr/vps-game)
- [PulseHeberg — Performance Cloud](https://pulseheberg.com/en/cloud/vps-performance)
