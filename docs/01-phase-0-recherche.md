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
| **PulseHeberg** | **PERF-8** (Performance Cloud) | 4 | **Ryzen 9 9900X, 4,4 GHz** (boost 5,66) | 8 Go DDR5 ECC | 100 Go NVMe | **Paris** | **18 € TTC** |
| **netcup** | **RS 1000 G12** | **4 dédiés** | EPYC 9645 « Turin », 3,7 GHz crête | 8 Go DDR5 ECC | 256 Go NVMe | Nuremberg / Vienne / **Amsterdam** | 10,74 € HT ≈ **12,89 € TTC** |
| **RedHeberg** | **Game ULTRA** | 4 partagés | Ryzen 9 5950X, 4,9 GHz | 8 Go | 70 Go NVMe | **Paris** | **12,95 € TTC** |
| PulseHeberg | PERF-4 | 2 | Ryzen 9 9900X, 4,4 GHz | 4 Go DDR5 ECC | 60 Go NVMe | Paris | 10 € TTC |
| PulseHeberg | **CLASSIC-8** | 8 | Xeon Platinum 8260 — **mesuré 2 394 MHz** | 8 Go DDR4 ECC | 120 Go NVMe RAID 10 | France / Suisse | 11 € TTC |
| PulseHeberg | **CLASSIC-4** | 4 | Xeon Platinum 8260 — même CPU, même fréquence | 4 Go DDR4 ECC | 80 Go NVMe RAID 10 | France / Suisse | 7 € TTC |
| OVHcloud | VPS-2 2027 | 4 | Intel ≈ 2,4 GHz. VPSBenchmarks : *Raw CPU Power* **E — 6,5/20** | 8 Go | 75 Go NVMe | Gravelines / Roubaix / Strasbourg | 7,21 € HT / **8,65 € TTC** |
| netcup | VPS 1000 G12 | 4 partagés | EPYC 9645 | 8 Go DDR5 ECC | 256 Go NVMe | idem netcup | 8,71 € HT ≈ 10,45 € TTC |
| RedHeberg | Game PRO | 2 partagés | Ryzen 9 5950X, 4,9 GHz | 6 Go | 50 Go NVMe | Paris | 8,95 € TTC |

### Grille PulseHeberg Performance Cloud

Le chiffre du nom correspond à la **RAM en Go**, pas au nombre de vCore.

| Offre | vCore | RAM DDR5 ECC | NVMe | Bande passante | Prix TTC |
|---|---|---|---|---|---|
| PERF-2 | 2 | 2 Go | 40 Go | 500 Mb/s | 6 € |
| PERF-4 | 2 | 4 Go | 60 Go | 500 Mb/s | 10 € |
| PERF-8 | 4 | 8 Go | 100 Go | 1 Gb/s | 18 € |
| PERF-16 | 8 | 16 Go | 150 Go | 1 Gb/s | 27 € |

### Le piège du benchmark affiché sur la page de commande

PulseHeberg affiche un score **Geekbench 4** sur chaque offre Classic :

| Offre | Prix | vCPU | Score affiché | Score **par cœur** |
|---|---|---|---|---|
| CLASSIC-4 | 7 € | 4 | 13 802 | 3 450 |
| CLASSIC-8 | 11 € | 8 | 25 869 | 3 234 |

Le score double exactement quand le nombre de cœurs double : c'est un score **multi-cœur**,
qui ne mesure que la quantité de cœurs. Rapporté au cœur, les deux offres sont identiques —
même Xeon 8260, même 2,40 GHz.

Comme Valheim n'exploite sérieusement qu'un seul cœur, **CLASSIC-4 et CLASSIC-8 offrent
exactement la même fluidité en jeu**. Les 4 € d'écart achètent de la RAM et du disque, pas
des performances. Le chiffre mis en avant sur la page de commande est donc précisément la
mauvaise métrique pour cet usage.

À noter également : Geekbench 4 est un benchmark de 2016, abandonné depuis. La référence
actuelle est Geekbench 6, dont les scores ne sont pas comparables.

### Comparaison à la bonne métrique : Geekbench 6 mono-cœur, mesuré sur VPS réels

Relevés VPSBenchmarks, sur les offres elles-mêmes quand elles y figurent :

| Offre | CPU | **GB6 mono-cœur** |
|---|---|---|
| PulseHeberg CLASSIC-4 / CLASSIC-8 | Xeon Platinum 8260 | **929** (relevé sur un Classic-8) |
| netcup RS 1000 G12 | EPYC 9645 | **1 600 – 1 650** (relevé sur ce plan précis) |
| RedHeberg — équivalents Ryzen 9 5950X | Ryzen 9 5950X | 2 050 – 2 150 sur hôte correct, mais **600 – 650** sur un hôte surchargé |
| PulseHeberg PERF — Ryzen 9 9900X | Ryzen 9 9900X | non mesuré en VPS ; 3 401 en bare metal, et un 9950X relevé entre 2 700 et 3 350 en VPS |

Deux enseignements : l'EPYC dédié de netcup vaut environ **1,75×** le Xeon des offres Classic,
et l'écart de 600 à 2 150 constaté sur des VPS Ryzen 5950X montre le risque propre au vCPU
partagé — la performance dépend du taux de remplissage de l'hôte, ce que le cœur dédié de
netcup élimine par construction.

### Pourquoi les offres CLASSIC sont écartées

Mesures relevées sur un Classic-8 réel (VPSBenchmarks, Paris, 30 août 2026) :

| Métrique | Valeur |
|---|---|
| Fréquence constatée | **2 394 MHz** — le turbo annoncé « ~3,90 GHz » ne se matérialise pas sur un vCore partagé |
| **Geekbench 6 mono-cœur** | **929** |
| Geekbench 6 multi-cœur | 3 821 |
| Disque (fio, blocs 64k) | 244 / 245 Mio/s — modeste pour du NVMe RAID 10 |
| PassMark single thread du Xeon 8260 | 1 962 → 2076ᵉ sur 5 999 CPU |

À comparer au **Ryzen 9 9900X** des offres PERF : **3 401** en Geekbench 6 mono-cœur en bare
metal, 41ᵉ sur 5 999 en PassMark single thread. Même en tenant compte de la virtualisation,
l'écart par cœur est de l'ordre de **3×** sur le seul critère qui compte pour Valheim.

Les 8 vCore du CLASSIC-8 ne compensent rien : la boucle de simulation est mono-thread et
n'en utilisera qu'un. C'est le profil « beaucoup de cœurs lents » que le handoff identifie
comme cause directe de rubber-banding à l'exploration.

Réserve sur **OVH** : anti-DDoS et bande passante illimitée excellents, prix le plus bas, mais
le CPU est noté E par VPSBenchmarks — même travers que le CLASSIC-8. Écarté pour cette raison.

---

## 3. Recommandation

> **Situation au 13/09/2026 : la gamme Performance Cloud est en rupture de stock.**
> Le choix se joue donc entre attendre le réassort et commander ailleurs. Se rabattre sur
> une offre CLASSIC serait un mauvais arbitrage : c'est le CPU qu'on ne peut pas corriger
> après coup sans migrer toute la machine.

**Si l'attente est acceptable — PulseHeberg PERF-8, Paris : 18 €/mois TTC**
**Commandable aujourd'hui — netcup RS 1000 G12, Amsterdam : ≈ 12,89 €/mois TTC**

Raisonnement :

1. Le Ryzen 9 9900X à 4,4 GHz est le meilleur CPU mono-cœur du comparatif — critère n° 1 pour
   une boucle de simulation mono-thread, et environ 3× le CLASSIC-8 du même hébergeur.
2. Paris : latence minimale pour France et Benelux, sur sol français.
3. 8 Go DDR5 ECC et 100 Go NVMe : marge réelle pour un monde qui ne rétrécira jamais.
4. SLA 99,99 % et anti-DDoS Netrix Intense (mitigation en moins de 5 secondes) inclus.
5. 18 € reste dans le budget de 20 €, chez un hébergeur où le compte existe déjà — pas de
   fournisseur supplémentaire à gérer pour 5 € d'écart sur un serveur prévu pour durer.

Le RS 1000 G12 est commandable immédiatement et mesure **1 600 – 1 650** en Geekbench 6
mono-cœur sur ce plan précis, contre 929 pour les offres Classic — soit 1,75× sur le critère
déterminant, pour 4 € de moins que le PERF-8. Ses cœurs sont **dédiés**, ce qui supprime la
loterie du vCPU partagé. Concession : Amsterdam plutôt que Paris, sans conséquence réelle
puisque le crossplay fait de toute façon transiter le trafic par le relais PlayFab.

Alternatives défendables :

- **PulseHeberg PERF-4 à 10 €** (si réassort) — même CPU rapide, mais 2 vCore et 4 Go. Tient
  pour 2 à 5 joueurs sur un monde neuf, sera à l'étroit d'ici un an. À ne retenir que si le
  redimensionnement en place est possible sans réinstallation.
- **RedHeberg Game ULTRA à 12,95 €** — Paris, Ryzen 9 5950X, 8 Go. Le CPU est excellent quand
  l'hôte n'est pas saturé, mais les relevés VPS sur ce processeur vont de 600 à 2 150 selon
  l'hébergeur, et les avis Trustpilot de RedHeberg font état de problèmes de réactivité du
  support, de litiges de remboursement et de pertes de données sur serveurs dédiés. À ne
  retenir que si le sol français prime sur tout le reste.

### À écarter explicitement

**PulseHeberg CLASSIC-4 (7 €) et CLASSIC-8 (11 €)** : même Xeon Platinum 8260 mesuré à
2 394 MHz, Geekbench 6 mono-cœur à 929 dans les deux cas. Les deux offres délivrent une
fluidité en jeu identique — cf. le piège du benchmark ci-dessus. Elles feraient tourner un
serveur pour 2 à 5 joueurs, mais avec des à-coups perceptibles à l'exploration, qui
s'aggraveront à mesure que le monde grossira. C'est le seul paramètre qu'on ne peut pas
corriger sans changer de machine.

### À vérifier au moment de la commande

- Possibilité de redimensionner un VPS PulseHeberg en place, sans réinstallation.
- Frais d'installation éventuels sur un engagement mensuel.

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
- [PulseHeberg — grille Cloud Performance Ryzen 9 9900X](https://newsroom.pulseheberg.com/univers-cloud-lancement-de-nos-nouvelles-offres-avec-processeurs-amd-ryzen-9-9900x/)
- [PulseHeberg Classic-8 — relevé YABS](https://www.vpsbenchmarks.com/yabs/pulseheberg-8c-8gb-20260830-1d05c4)
- [Intel Xeon Platinum 8260 — PassMark](https://www.cpubenchmark.net/cpu.php?cpu=Intel+Xeon+Platinum+8260+%40+2.40GHz&id=3561)
- [Ryzen 9 9900X — Geekbench 6 mono-cœur](https://www.techpowerup.com/324235/amd-ryzen-9-9900x-benchmarked-in-geekbench-6-beats-intels-best-in-single-core-score)
- [CPU de VPS classés par Geekbench 6 mono-cœur — VPSBenchmarks](https://www.vpsbenchmarks.com/labs/cpus_by_geekbench6_perf)
- [Avis clients RedHeberg — Trustpilot](https://www.trustpilot.com/review/redheberg.fr)
