# Handoff — Serveur Valheim auto-hébergé + panel d'administration privé

> Document destiné à une session Claude travaillant dans un projet dédié, avec accès SSH autonome au VPS.
> Rédigé le 13 septembre 2026. **Toutes les valeurs techniques ci-dessous doivent être revérifiées** (voir Phase 0) : Valheim 1.0 est sorti le 9 septembre 2026, soit 4 jours avant la rédaction, et les paramètres serveur ont pu changer.

---

## 1. Objectif

Monter et opérer un serveur Valheim dédié **privé** sur un VPS, avec une interface d'administration web **accessible au seul propriétaire**, offrant :

- consultation des logs serveur en temps réel
- redémarrage / arrêt / démarrage du serveur de jeu
- backups automatiques + restauration en un clic
- état de santé (CPU, RAM, disque, joueurs connectés, uptime)
- mise à jour du jeu

Ce n'est **pas** un service d'hébergement multi-tenant. Un seul serveur de jeu, un seul administrateur. Toute solution qui ajoute de la complexité multi-utilisateurs (Pterodactyl, WISP, panels commerciaux) est à écarter sauf demande explicite.

---

## 2. Informations à demander avant de commencer

Ne pas provisionner quoi que ce soit avant d'avoir ces réponses :

| Question | Pourquoi |
|---|---|
| Budget mensuel max ? | Détermine la gamme VPS |
| Où sont géographiquement les joueurs ? | Choix de la région du datacenter (latence) |
| Combien de joueurs simultanés attendus ? | Dimensionnement RAM |
| Monde neuf ou import d'un monde existant ? | Un monde existant peut déjà être gros → plus de RAM |
| Mods prévus (BepInEx, ValheimPlus, Epic Loot…) ? | Double quasiment le besoin RAM |
| Crossplay console souhaité ? | Change les flags de lancement et le mode de connexion |
| Fournisseur cloud déjà utilisé / préféré ? | Évite de multiplier les comptes |
| Nom de domaine disponible ? | Utile pour le panel en HTTPS |
| Clé SSH publique existante ? | Sinon en générer une et la lui transmettre |

Utiliser un format à choix multiples pour ces questions plutôt que de les poser en prose.

---

## 3. Phase 0 — Recherche à mener (obligatoire, avant toute commande)

Le propriétaire attend explicitement une **recommandation d'infrastructure argumentée**, pas une exécution aveugle. À vérifier par recherche web, sources primaires en priorité (documentation Iron Gate, wiki officiel, ArchWiki, pages tarifaires des hébergeurs) :

### 3.1 Valheim, état actuel
- [ ] App ID SteamCMD du serveur dédié (valeur connue : **896660**) — confirmer
- [ ] Variable `SteamAppId` à exporter (valeur connue : **892970**) — confirmer
- [ ] Ports UDP requis (valeur connue : **2456, 2457, 2458** — trois ports consécutifs) — confirmer, la 1.0 a pu changer ça
- [ ] Flags de lancement actuels : `-name`, `-port`, `-world`, `-password`, `-crossplay`, `-backups`, `-saveinterval`, `-public`
- [ ] Mécanique de connexion post-1.0 : code de partie numérique vs IP directe
- [ ] Dépendances système Linux (connues : `libatomic1`, `libpulse0`, `libpulse-dev`; GLIBC ≥ 2.29, GLIBCXX ≥ 3.4.26)
- [ ] Architecture supportée — **x86_64 uniquement à ce jour, ARM/aarch64 non supporté nativement**. À reconfirmer : si un binaire ARM64 existe désormais, cela ouvre des VPS Ampere nettement moins chers
- [ ] Chemin des sauvegardes (connu : `~/.config/unity3d/IronGate/Valheim/worlds_local`, fichiers `.db` + `.fwl`)
- [ ] Contraintes sur le mot de passe serveur (minimum de caractères, interdiction de contenir le nom du serveur)
- [ ] Fichier `adminlist.txt` / `permittedlist.txt` / `bannedlist.txt` : emplacement et syntaxe
- [ ] État de BepInEx et des mods principaux vis-à-vis de la 1.0 (souvent cassés les semaines suivant une release majeure)

### 3.2 Dimensionnement
Consensus à la date de rédaction, à revalider :
- Le serveur Valheim est **fortement mono-thread** sur la boucle de simulation. La **fréquence par cœur prime largement sur le nombre de cœurs**. Un 4 cœurs à 4,0 GHz bat un 8 cœurs à 2,4 GHz. En dessous de ~3,0 GHz, apparition de rubber-banding lors de l'exploration (génération de terrain qui ne suit pas).
- RAM : 2–4 Go pour un monde neuf à 2–4 joueurs ; 4–6 Go pour ~10 joueurs ; 8 Go+ si moddé ; 8–10 Go+ pour un monde ancien, très exploré et terraformé.
- **Les mondes Valheim ne rétrécissent jamais.** Chaque chunk exploré et chaque construction reste dans le fichier de sauvegarde. Prévoir de la marge : un serveur correctement dimensionné à l'ouverture sera sous-dimensionné dans six mois.
- Disque : 20–60 Go SSD/NVMe. Écritures en rafale au moment des sauvegardes → NVMe recommandé.
- Upload : ≥ 10 Mbps stable.

### 3.3 Comparatif VPS à établir
Construire un tableau comparatif avec **prix vérifiés au moment de la recherche** (les tarifs Hetzner ont augmenté en 2026, ne pas se fier à des chiffres mémorisés) :

- Hetzner Cloud (CPX / CCX), régions Nuremberg, Falkenstein, Helsinki
- OVHcloud VPS, régions Gravelines, Strasbourg, Roubaix
- Scaleway, région Paris
- Vultr High Frequency / Optimized Cloud Compute
- Netcup, PulseHeberg, ou tout hébergeur à haute fréquence pertinent

Critères de notation, par ordre d'importance :
1. **Fréquence CPU réelle** (chercher le modèle de processeur derrière l'offre, pas seulement le nombre de vCPU)
2. Latence vers les joueurs (région)
3. RAM
4. Disque NVMe
5. Protection anti-DDoS incluse — OVH et Scaleway sont nettement meilleurs que Hetzner sur ce point, et **un serveur de jeu est une cible récurrente**
6. Bande passante (OVH illimitée, Hetzner quotas généreux)
7. Prix

Livrer au propriétaire **une recommandation principale + une alternative**, avec le raisonnement en 5 lignes maximum et le coût mensuel exact. Attendre sa validation avant de provisionner.

---

## 4. Phase 1 — Provisionnement et durcissement

Le propriétaire crée le compte et le VPS lui-même (pas de carte bancaire à manipuler). Il fournit ensuite l'IP et un accès root initial.

- [ ] OS : Debian 12 ou Ubuntu 24.04 LTS. Rien d'exotique.
- [ ] Créer un utilisateur admin `sudo`, y déployer la clé SSH publique
- [ ] `PermitRootLogin no`, `PasswordAuthentication no` dans `/etc/ssh/sshd_config`
- [ ] Port SSH déplacé (optionnel, cosmétique)
- [ ] `fail2ban` sur SSH
- [ ] Pare-feu `ufw` ou `nftables` :
  - SSH : autorisé
  - **UDP 2456–2458 : ouverts** (attention, UDP, pas TCP — erreur la plus fréquente)
  - Panel : **jamais exposé publiquement en clair**, voir Phase 4
  - Tout le reste : `deny incoming`
- [ ] `unattended-upgrades` pour les correctifs de sécurité
- [ ] Fuseau horaire et `chrony`/`systemd-timesyncd`
- [ ] Swap de 2 Go si la RAM est juste (évite l'OOM kill en pleine sauvegarde)
- [ ] Snapshot de l'instance une fois la base propre, avant l'installation du jeu

---

## 5. Phase 2 — Serveur de jeu

- [ ] Utilisateur système dédié `valheim`, sans shell de connexion, **non privilégié**
- [ ] Installation de SteamCMD (dépôt `non-free` sur Debian, ou binaire officiel)
- [ ] `steamcmd +force_install_dir … +login anonymous +app_update <APPID> validate +quit` sous l'utilisateur `valheim`
- [ ] Installation des dépendances système identifiées en Phase 0
- [ ] Unit systemd `/etc/systemd/system/valheim.service`. Points critiques :
  - `User=valheim`
  - `Environment=SteamAppId=<valeur confirmée>`
  - `LD_LIBRARY_PATH` pointant vers `./linux64` du dossier serveur
  - `ExecStart` avec `-nographics -batchmode`
  - **`KillSignal=SIGINT`** — un `SIGTERM` brutal pendant une sauvegarde corrompt le monde
  - **`TimeoutStopSec=90`** — laisser au serveur le temps d'écrire
  - `Restart=on-failure`, `RestartSec=10`
  - Durcissement : `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `ReadWritePaths=` limité au dossier serveur et aux sauvegardes
- [ ] Mot de passe et nom du serveur **jamais en clair dans le unit file** → `EnvironmentFile=/etc/valheim/server.env`, permissions `600`, propriétaire `valheim`
- [ ] Activer les backups internes du jeu (`-backups`, `-saveinterval`) — ils ne remplacent pas la Phase 3, ils la complètent
- [ ] Vérifier le démarrage : `journalctl -u valheim -f` jusqu'à la ligne indiquant que le serveur est joignable
- [ ] Test de connexion réel par le propriétaire avant de passer à la suite

---

## 6. Phase 3 — Sauvegardes

Règle : **une sauvegarde qui vit sur le même disque que la donnée n'est pas une sauvegarde.**

- [ ] `restic` (ou `borg`) vers un stockage objet externe : Backblaze B2, Hetzner Storage Box, Scaleway Object Storage, OVH Object Storage. Coût réel : quelques centimes par mois.
- [ ] Périmètre : le dossier `worlds_local` (`.db` + `.fwl`) **et** les fichiers de configuration (`adminlist.txt`, `server.env`, unit systemd)
- [ ] Timer systemd horaire, pas cron
- [ ] Politique de rétention : `--keep-hourly 24 --keep-daily 7 --keep-weekly 4 --keep-monthly 6`, puis `prune`
- [ ] Le mot de passe du dépôt restic doit être **communiqué au propriétaire et stocké hors du VPS**. Un dépôt chiffré dont la clé n'existe que sur la machine sauvegardée ne sert à rien.
- [ ] **Test de restauration effectif** avant de déclarer la phase terminée : restaurer dans un dossier temporaire, vérifier l'intégrité des fichiers. Une sauvegarde non testée est une hypothèse.
- [ ] Snapshot automatique du VPS côté hébergeur en complément, si l'offre l'inclut

---

## 7. Phase 4 — Panel d'administration

### Décision d'architecture
Trois options à présenter au propriétaire avant de coder. Sa préférence exprimée jusqu'ici penche vers un panel sur mesure, mais confirmer :

| Option | Effort | Pour qui |
|---|---|---|
| **Cockpit** (`apt install cockpit`) | 10 min | Logs journald, contrôle systemd, métriques. Zéro code. Suffit souvent. |
| **Panel sur mesure** | ~200–400 lignes | Contrôle total, UI adaptée à Valheim (joueurs connectés, backups, restauration) |
| Pterodactyl | Élevé | Uniquement si multi-jeux ou accès tiers prévus un jour |

### Si panel sur mesure

Backend :
- FastAPI (Python) ou Fastify (Node), un seul processus, systemd
- Endpoints : `GET /status`, `POST /restart`, `POST /stop`, `POST /start`, `WS /logs` (flux `journalctl -fu valheim`), `GET /backups`, `POST /backups/restore`, `POST /update`
- **Aucune interpolation de chaîne dans un shell.** Appels `systemctl` / `restic` via tableau d'arguments, jamais `shell=True`. C'est la faille évidente d'un panel de ce type.
- Le service panel tourne sous son propre utilisateur, avec des règles `polkit` ou une entrée `sudoers` **restreinte aux commandes exactes nécessaires**. Jamais de panel root.
- Une restauration de backup doit exiger une confirmation explicite et arrêter proprement le serveur avant d'écraser le monde.

Frontend :
- Page unique, sobre, lisible sur mobile. Le propriétaire consultera ça depuis son téléphone.
- Console de logs en flux, indicateurs CPU/RAM/disque, liste des backups datés, boutons d'action avec état de chargement.

Exposition — **point le plus important de tout ce document** :
- Option recommandée : **Tailscale ou WireGuard**. Le panel écoute sur l'interface VPN uniquement, et n'est jamais joignable depuis Internet. Aucun mot de passe à protéger, aucune surface d'attaque.
- Option alternative : Caddy en reverse proxy, HTTPS automatique, et une authentification réelle (clé matérielle, ou à défaut mot de passe fort + limitation de tentatives). Ne jamais s'arrêter à une simple authentification basique HTTP sur un endpoint qui exécute des commandes système.
- Un panel capable de redémarrer des services et exposé publiquement sans VPN est le moyen le plus rapide de perdre la machine.

---

## 8. Phase 5 — Exploitation

- [ ] Supervision : alerte si le service tombe, si le disque dépasse 80 %, si un backup échoue. Healthchecks.io ou Uptime Kuma suffisent.
- [ ] Procédure de mise à jour du jeu documentée : annoncer, arrêter proprement, backup, `app_update`, redémarrer, vérifier
- [ ] Avertissement explicite : **après une mise à jour majeure de Valheim, les mods BepInEx cassent presque systématiquement.** Ne jamais mettre à jour un serveur moddé sans backup frais et sans vérifier la compatibilité des mods.
- [ ] Runbook remis au propriétaire : redémarrer, restaurer un backup, ajouter un admin, bannir un joueur, changer le mot de passe, agrandir le VPS

---

## 9. Règles de travail pour la session Claude

1. **Recherche avant exécution.** Les chiffres de ce document ont une date. Les revérifier.
2. **Valider la recommandation VPS avec le propriétaire avant tout provisionnement.** C'est une dépense récurrente, c'est sa décision.
3. **Demander confirmation avant toute action destructive** : écrasement d'un monde, restauration de backup, suppression de fichiers, modification de la configuration SSH ou du pare-feu depuis une session SSH (risque de s'auto-exclure — toujours garder une session ouverte en parallèle lors d'un changement de règles firewall ou sshd).
4. **Aucun secret en clair** dans un dépôt Git, un unit systemd, ou un historique shell. Fichiers `.env` en `600`.
5. **Tout est reproductible.** Chaque fichier de configuration créé sur le serveur existe aussi dans le projet. Si le VPS brûle, tout doit se rejouer.
6. **Valider chaque phase avant la suivante**, avec un test concret : connexion au jeu réussie, restauration de backup vérifiée, panel joignable.
7. Rester concis dans les réponses : lecture probable sur mobile.

---

## 10. Critères de réussite

- [ ] Le propriétaire et ses amis se connectent au serveur et jouent
- [ ] Le serveur redémarre seul après un crash ou un reboot du VPS
- [ ] Un backup horaire part vers un stockage externe, et une restauration a été testée pour de vrai
- [ ] Le panel est accessible au propriétaire seul, depuis son téléphone, et permet logs / restart / backups
- [ ] Le panel n'est joignable par personne d'autre depuis Internet
- [ ] Le propriétaire dispose d'un runbook et sait agir sans assistance

---

## 11. Pièges connus

- Ports ouverts en TCP au lieu d'UDP → serveur invisible
- Deuxième et troisième port oubliés (2457, 2458)
- `SteamAppId` non exporté → le serveur démarre puis se ferme immédiatement
- Dépendances `libpulse0` / `libatomic1` manquantes → échec au lancement, message explicite dans `journalctl -u valheim`
- Mot de passe trop court, ou contenant le nom du serveur → refus au démarrage
- Arrêt brutal pendant une sauvegarde → monde corrompu. `SIGINT` + délai généreux.
- VPS ARM choisi pour le prix → binaire x86_64 uniquement, incompatible (à reconfirmer en Phase 0)
- Beaucoup de cœurs lents préférés à peu de cœurs rapides → rubber-banding
- Backups internes du jeu considérés comme suffisants → ils sont sur le même disque
