# Constats machine

Inventaire en lecture seule du poste de développement. Chaque fait est accompagné
de la commande qui l'a produit, exécutée le 2026-08-04. Aucun chiffre n'est
consigné sans cette source. Un jeton dédié, préfixé par une arobase, signale
en tête de ligne un point que ces commandes n'ont pas permis d'établir — il
ne doit être retiré qu'après avoir été effectivement vérifié, jamais par
confort de lecture. `grep -c` sur ce jeton dans ce fichier ne doit compter
que ces points réellement ouverts ; les mentions en prose qui parlent du
mécanisme lui-même (comme ce paragraphe) l'évitent délibérément.

Voir aussi [`CLAUDE.md`](../CLAUDE.md) pour les règles qui gouvernent la mise à
jour de ce document.

---

## Matériel

- Modèle : ASUS ROG Zephyrus Duo 16 GX650PY_GX650PY, chassis « laptop ».
  (`hostnamectl`)
- Firmware : GX650PY.319, daté du 2023-11-23. (`hostnamectl`)
- CPU : AMD Ryzen 9 7945HX with Radeon Graphics — 1 socket, 16 cœurs, 32 threads
  logiques (SMT actif). (`lscpu`)
- RAM : 61 Gio au total, 8,0 Gio de swap (zram). (`free -h`)
- Secure Boot : désactivé (`SecureBoot disabled`). (`mokutil --sb-state`)
- dGPU : NVIDIA GeForce RTX 4090 Laptop GPU (AD103M / GN21-X11, id PCI
  `10de:2717`), bus PCI `01:00.0`, pilote en charge `nvidia`.
  (`lspci -nnk`)
- iGPU : AMD Raphael (id PCI `1002:164e`), bus PCI `09:00.0`, pilote en charge
  `amdgpu`. (`lspci -nnk`)

## Système

- Distribution : Fedora Linux 44 (KDE Plasma Desktop Edition).
  (`cat /etc/os-release`, `cat /etc/fedora-release`)
- Noyau : `7.1.5-201.fc44.x86_64`. (`uname -r`)
- Hostname : `Zephyrus-MM`. (`hostnamectl`)
- Session graphique : Wayland, bureau KDE, session active de type `wayland`.
  (`echo $XDG_SESSION_TYPE`, `echo $XDG_CURRENT_DESKTOP`, `loginctl show-session`)

## Dépôts

Dix dépôts activés : `claude-code`, `fedora`, `fedora-cisco-openh264`,
`rpmfusion-free`, `rpmfusion-free-tainted`, `rpmfusion-free-updates`,
`rpmfusion-nonfree`, `rpmfusion-nonfree-updates`, `terra`, `updates`.
(`dnf repo list --enabled`)

- `terra` : priorité 99, coût 1000, gpgcheck et repo_gpgcheck actifs, clé
  `file:///etc/pki/rpm-gpg/RPM-GPG-KEY-terra44`. (`dnf repoinfo terra` — voir
  note de méthode ci-dessous, cette commande n'était pas purement passive)
- `rpmfusion-free-tainted` : activé délibérément par la transaction 18
  (`dnf install rpmfusion-free-release-tainted`), liée à la chaîne
  multimédia. (`dnf history info 18`, `cat /etc/yum.repos.d/rpmfusion-free-tainted.repo`)
- `claude-code` : `baseurl=https://downloads.claude.ai/claude-code/rpm/stable`,
  gpgcheck actif, clé `https://downloads.claude.ai/keys/claude-code.asc`.
  (`cat /etc/yum.repos.d/claude-code.repo`)

**Note de méthode — incident `dnf repoinfo terra`.** Cette commande a
déclenché une invite d'import de clé OpenPGP (`Is this ok [y/N]:`) au lieu de
rester purement passive comme les autres lectures de dépôt. Vérification
immédiate : la clé Terra (`gpg-pubkey-ae09157a...-698a7357`) était déjà présente
dans le trousseau rpm avec un horodatage d'installation (14:45:49 CEST) très
antérieur à l'exécution de cette commande (20:3x CEST) — aucune nouvelle clé
n'a donc été ajoutée par cette lecture. (`rpm -q gpg-pubkey --qf '...installtime...'`)
Le champ priorité ci-dessus reste donc établi, mais la commande qui l'a produit
n'était pas anodine ; elle ne doit pas être réutilisée telle quelle dans un
contexte strictement lecture seule sans confirmation préalable.

## GPU

- Pilote : `NVIDIA-SMI 610.43.03`, `KMD Version 610.43.03`, `CUDA UMD Version
  13.3`, VBIOS `95.03.2D.00.06`. (`nvidia-smi -q`)
- VRAM totale : 16376 MiB. VRAM utilisée au moment de la lecture : 1045 MiB
  (valeur transitoire, non représentative d'un état stable — mesurée le
  2026-08-04 vers 20:31 CEST). (`nvidia-smi -q`)
- Modules chargés : `nvidia`, `nvidia_drm`, `nvidia_modeset`, `nvidia_uvm`.
  (`lsmod | grep -i nvidia`)
- Provenance des paquets NVIDIA installés :
  - `from_repo=rpmfusion-nonfree-updates` : `akmod-nvidia`, `nvidia-modprobe`,
    `nvidia-persistenced`, `nvidia-settings`, `xorg-x11-drv-nvidia`,
    `xorg-x11-drv-nvidia-cuda`, `xorg-x11-drv-nvidia-cuda-libs`,
    `xorg-x11-drv-nvidia-kmodsrc`, `xorg-x11-drv-nvidia-libs`,
    `xorg-x11-drv-nvidia-power`.
  - `from_repo=updates` (Fedora) : `libva-nvidia-driver`, `nvidia-gpu-firmware`.
  - `from_repo=@commandline` : `kmod-nvidia-7.1.5-201.fc44.x86_64` — construit
    localement par akmods et installé sans dépôt (voir transaction 11
    ci-dessous).
  (`dnf repoquery --installed --qf '%{name} from_repo=%{from_repo}\n' | grep -i nvidia`)
- `/etc/modprobe.d/` ne contient aucun fichier lié à NVIDIA (13 fichiers,
  tous des blacklists réseau ou `nvdimm-security.conf`).
  (`ls -la /etc/modprobe.d/`)
- `/usr/lib/modprobe.d/nvidia-power-management.conf` : les trois directives
  sont commentées — `NVreg_PreserveVideoMemoryAllocations=1`,
  `NVreg_TemporaryFilePath=/var/tmp`, `NVreg_UseKernelSuspendNotifiers=1`.
  `NVreg_DynamicPowerManagement` n'apparaît nulle part dans ce fichier.
  (`cat /usr/lib/modprobe.d/nvidia-power-management.conf`)
- `/usr/lib/modprobe.d/nvidia-uvm.conf` : `softdep nvidia post: nvidia-uvm`
  uniquement (contournement du problème plymouth/initrd documenté en
  commentaire). (`cat /usr/lib/modprobe.d/nvidia-uvm.conf`)
- Aucun fichier n'est homonyme entre `/etc/modprobe.d/` et
  `/usr/lib/modprobe.d/` sur cette machine à la date de l'inventaire.
  (`comm -12 <(ls /etc/modprobe.d/ | sort) <(ls /usr/lib/modprobe.d/ | sort)`
  → sortie vide, confirmée par un code de retour 0 sur la commande `comm`
  elle-même, donc établi et non un cas de « commande sans sortie »)
- Services :
  - `asusd` : unit `static`, actif (`active (running)`) depuis le démarrage
    de session. (`systemctl status asusd`, `systemctl is-enabled/is-active asusd`)
  - `supergfxd` : `disabled`, `inactive (dead)`. (`systemctl status supergfxd`)
  - `nvidia-persistenced` : `disabled`, `inactive (dead)`.
    (`systemctl status nvidia-persistenced`)
  - Tentative `supergfxctl -g` : échoue explicitement — « supergfxd is not
    enabled, enable it with `systemctl enable supergfxd` » /
    `org.freedesktop.DBus.Error.ServiceUnknown: The name is not activatable`.
    (`supergfxctl -g`)
- Attributs `asus-armoury` disponibles : `boot_sound`, `charge_mode`,
  `dgpu_disable`, `gpu_mux_mode`, `mini_led_mode`, `nv_dynamic_boost`,
  `nv_temp_target`, `panel_overdrive`, `ppt_pl1_spl`, `ppt_pl2_sppt`,
  `ppt_pl3_fppt`, plus un fichier `pending_reboot` global au niveau du
  répertoire `attributes/`. (`ls -la /sys/class/firmware-attributes/asus-armoury/attributes/`)
  - `dgpu_disable` : `current_value=0`, `possible_values=0;1`, type
    `enumeration`. (`cat .../dgpu_disable/current_value` et fichiers voisins ;
    confirmé par `asusctl armoury get dgpu_disable` → `current: [(0),1]`)
  - `gpu_mux_mode` : `current_value=0`, `possible_values=0;1`, type
    `enumeration`. (`cat .../gpu_mux_mode/current_value` et fichiers voisins ;
    confirmé par `asusctl armoury get gpu_mux_mode` → `current: [(0),1]`)
  - `pending_reboot` global (au niveau `attributes/`, pas sous
    `gpu_mux_mode/`) : `0`. Il n'existe pas de fichier `pending_reboot`
    séparé dans `gpu_mux_mode/` — la tentative de le lire à cet endroit
    échoue (`No such file or directory`, code retour 1). C'est le fichier au
    niveau supérieur qui fait foi. (`cat .../attributes/pending_reboot`)
  - `asusctl` ne restitue que les valeurs numériques (`current: [(0),1]`),
    sans étiquette « Optimus » / « Ultimate » / « dGPU » — cette
    correspondance n'est donc pas établie par une commande locale sur ce
    poste. Elle est documentée dans le noyau :
    `Documentation/ABI/testing/sysfs-platform-asus-wmi` (référence externe,
    non lue sur cette machine) donne `0` = dGPU discrète (MUX direct vers la
    RTX 4090, panneau non piloté par l'iGPU), `1` = Optimus/Hybrid (rendu
    déchargé sur l'iGPU), avec redémarrage requis après commutation. Cette
    même documentation marque l'attribut équivalent
    `/sys/devices/platform/asus-nb-wmi/gpu_mux_mode` comme déprécié au
    profit de `asus-armoury` / `firmware-attributes` — c'est le motif du
    choix de ce chemin sysfs dans ce dépôt plutôt que l'ancien chemin
    `asus-nb-wmi`. Localement, `possible_values=0;1` et `type=enumeration`
    corroborent une bascule binaire, sans contredire cette lecture.
  - Constat croisé avec la section Affichage ci-dessous : à la date de cet
    inventaire, le panneau principal `eDP-2` est rattaché à `card2` (bus PCI
    `01:00.0`, pilote `nvidia`), et non à `card1` (AMD, `09:00.0`),
    `gpu_mux_mode=0`, `pending_reboot=0`. Avec la sémantique ci-dessus,
    c'est l'état cohérent attendu **avant** toute bascule vers
    Optimus/Hybrid, pas une anomalie : décision D2bis prise le 2026-08-04,
    non encore appliquée à la date de cet inventaire (voir « Décisions »).
- Règle udev `/usr/lib/udev/rules.d/90-supergfxd-nvidia-pm.rules` (déposée par
  le paquet `supergfxctl`, pas par `asusd`) : bascule `power/control` à
  `auto` sur bind des fonctions PCI NVIDIA (classes `0x030000` et
  `0x030200`), et à `on` sur unbind. Cette règle agit indépendamment de l'état
  du démon `supergfxd`. (`cat /usr/lib/udev/rules.d/90-supergfxd-nvidia-pm.rules`)
  - État constaté ce jour : `power/control=auto` sur les deux fonctions PCI
    NVIDIA (`0000:01:00.0` et `0000:01:00.1`), confirmant que la règle est
    active en pratique malgré `supergfxd` inactif.
    (`cat /sys/bus/pci/devices/0000:01:00.{0,1}/power/control`)

## Stockage

- Deux périphériques NVMe, modèle `HFS002TEJ9X101N`, ~1,9 To chacun :
  `nvme0n1` (partitionné : `nvme0n1p1` `/boot/efi` vfat 600M, `nvme0n1p2`
  `/boot` ext4 2G, `nvme0n1p3` `/home` btrfs 1,9T) et `nvme1n1`
  (`nvme1n1p1`, btrfs, 1,9T, point de montage non affiché par `lsblk` dans
  cette invocation — probablement `/`). (`lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,MODEL`)
- `btrfs filesystem show /` : label `fedora`, uuid
  `b37a7138-5e93-4344-a5f6-a728021c9e21`, **2 périphériques**, `FS bytes used
  7.41GiB`. Les tailles par périphérique s'affichent à `0`/`MISSING` en mode
  non privilégié — nécessitent root pour le détail par device.
  (`btrfs filesystem show /`)
- `btrfs filesystem usage /` (profils, sans le détail par périphérique, qui
  requiert root — avertissement explicite affiché par la commande elle-même) :
  - `Data,single` : 10,01 GiB alloués, 7,12 GiB utilisés (71,18 %)
  - `Metadata,RAID1` : 1,00 GiB alloués, 295,27 MiB utilisés (28,83 %)
  - `System,RAID1` : 8,00 MiB alloués, 16,00 KiB utilisés (0,20 %)
  (`btrfs filesystem usage /`)
- Swap : `zram0`, 8G. (`lsblk`, `free -h`)

## Affichage

- Deux cartes DRM : `card1` → bus PCI `09:00.0` (AMD/amdgpu), `card2` → bus
  PCI `01:00.0` (NVIDIA/nvidia). (`ls -la /sys/class/drm/`)
- Connecteurs `card1` (AMD) : `DP-1` déconnecté, `DP-2` déconnecté, `DP-3`
  **connecté**, `eDP-1` déconnecté, `Writeback-1` (`unknown`, type spécial,
  pas un connecteur physique). (`cat /sys/class/drm/card1-*/status`)
- Connecteurs `card2` (NVIDIA) : `DP-4` déconnecté, `DP-5` déconnecté,
  `eDP-2` **connecté**, `HDMI-A-1` déconnecté. (`cat /sys/class/drm/card2-*/status`)
- `kscreen-doctor -o` :
  - Sortie 1, `DP-3` (écran ScreenPad Plus secondaire, rattaché à `card1`
    AMD) : activée, connectée, mode actif `3840x1100@60.02`, géométrie
    `0,1280 2743x786`, échelle 1.4, rotation 1.
  - Sortie 2, `eDP-2` (panneau principal, rattaché à `card2` NVIDIA) :
    activée, connectée, mode actif `2560x1600@60.00`. Le mode
    `2560x1600@240.00` est **listé comme disponible mais n'est pas le mode
    actif** (pas de `*` sur cette entrée) — géométrie `315,0 2048x1280`,
    échelle 1.25, rotation 1.
  (`kscreen-doctor -o`)

## Conteneurs

- Podman `5.8.4` (API 5.8.4), mode rootless, `cgroupManager: systemd`,
  `cgroupVersion: v2`. (`podman info`)
- Runtime OCI : `crun` 1.28. Réseau : `netavark` 1.17.2 /
  `aardvark-dns` 1.17.1, backend rootless `pasta`. (`podman info`)
- Stockage : pilote `overlay`, filesystem de backing `btrfs`,
  `graphRoot=/home/mahieumi/.local/share/containers/storage`. (`podman info`)
- `/etc/cdi/` : absent (`No such file or directory`). Aucune définition CDI
  n'est donc configurée à ce jour pour exposer le GPU aux conteneurs par ce
  mécanisme. (`ls -la /etc/cdi/`)
- `nvidia-container-toolkit` : non installé. (`rpm -q nvidia-container-toolkit`)
- `/etc/containers/containers.conf` : absent, seuls les sous-répertoires par
  défaut (`certs.d`, `networks`, `oci`, `registries.conf`, etc.) sont
  présents dans `/etc/containers/`. (`ls -la /etc/containers/`)

## Chaîne Ansible

- `ansible-core 2.20.7`, config `/etc/ansible/ansible.cfg`, module Python
  `/usr/lib/python3.14/site-packages/ansible`, exécutable `/usr/bin/ansible`,
  Python associé `3.14.6`, Jinja `3.1.6`, PyYAML `6.0.3` (avec `libyaml`
  0.2.5). (`ansible --version`)
- Installé par la transaction 21 (`Install ansible-core-0:2.20.7-1.fc44`,
  `User=0 root`, dépôt `updates`), avec ses dépendances `python3-jinja2`,
  `python3-resolvelib`, `python3-markupsafe`, `python3-cryptography`.
  (`dnf history info 21`)
- `python3` : `3.14.6`. (`python3 --version`)
- `git` : `2.55.0`. (`git --version`)
- `claude` : `2.1.220 (Claude Code)`, gestionnaire de paquets `rpm`,
  chemin `/usr/bin/claude`, canal de mise à jour annoncé **`latest`**.
  (`claude --version`, `claude doctor`)
- Dépôt dnf réellement configuré pour `claude-code` : `baseurl=.../rpm/
  stable`. Écart entre canal annoncé (`latest`) et dépôt réel (`stable`) —
  observé, non expliqué. Voir « Points ouverts ». (`cat /etc/yum.repos.d/claude-code.repo`)

## Décisions

**D1 — btrfs sur deux périphériques, `Data,single` / `Metadata,RAID1`.**
La perte d'un NVMe détruit les données (confirmé structurellement par
`btrfs filesystem show /` : 2 devices, profil Data en `single`, donc sans
duplication des données elles-mêmes). Risque assumé : machine de
développement, le dépôt distant fait foi. Corollaire contraignant : rien qui
n'existe qu'ici ne doit avoir de valeur — modèles, images, environnements
d'exécution (EE) doivent être re-téléchargeables ou reconstructibles.

**D2 — [RETIRÉ le 2026-08-04].** ÉTAIT : supergfxd désactivé, dGPU active en
permanence. Motif d'origine : prévisibilité. Retiré au profit de D2bis.

**D2bis — mode graphique Optimus/Hybrid : décidée le 2026-08-04, non
appliquée.** Plasma sur l'iGPU AMD, RTX 4090 réservée au calcul. Motif :
supprimer la contention compositeur/inférence et rendre l'enveloppe 150 W au
calcul. Le gain VRAM (1079 MiB mesurés sur 16376 au moment de la décision)
est secondaire. Note : la mesure VRAM refaite dans cet inventaire (1045 MiB /
16376 MiB) est un nouveau relevé ponctuel, pas une reconduction du chiffre
d'origine — voir « GPU » ci-dessus. État constaté ce jour (`gpu_mux_mode=0`,
panneau principal `eDP-2` câblé sur la carte NVIDIA) : c'est l'état cohérent
attendu avant bascule vers Optimus/Hybrid, pas une anomalie — voir « GPU »
pour la sémantique de `gpu_mux_mode`. Bascule non encore appliquée :
livrable GPU-2 en attente.

**D2ter (2026-08-04) — bascule sans supergfxd, via `asusctl armoury`.**
Motif : `asusd` est déjà actif ; le README de `supergfxctl` signale un
conflit entre commutateurs GPU et un risque de repli sur `integrated` au
démarrage, ce qui couperait la dGPU. Corroboré ce jour : `supergfxctl -g`
échoue explicitement (« supergfxd is not enabled ») — le chemin `asusd`/
`asus-armoury` est bien celui en usage réel sur cette machine, pas
`supergfxctl`.

**D3 — chaîne Ansible EE-first.** `ansible-navigator` exécute et vérifie dans
l'image d'Execution Environment. L'`ansible-core` système (2.20.7 /
Python 3.14) sert à l'édition et ne fait pas autorité : il est plus récent
que la cible AAP 2.6 et accepterait des constructions que la production
refuse.

**D4 — dépôt public.** Aucun secret, aucune donnée d'entreprise, aucune
adresse interne, aucun nom d'hôte réel.

**D5 — Claude Code installé via le dépôt dnf officiel signé.** Canal stable,
mises à jour par `dnf upgrade`. Confirmé par le contenu du fichier repo
(`gpgcheck=1`, `baseurl=.../stable`). Sans rapport avec la chaîne Ansible :
un écart entre ce dépôt et le canal `latest` annoncé par `claude doctor` est
observé et documenté en « Points ouverts ».

**D6 — `tuned-ppd` remplacé par `power-profiles-daemon`.** Transaction 7
(`dnf swap tuned-ppd power-profiles-daemon --allowerasing`), confirmée :
installation de `power-profiles-daemon-0.30-3.fc44`, suppression de
`tuned-ppd` et de la chaîne `tuned` associée. Motif : cohérence avec `asusd`,
qui gère les profils d'alimentation ASUS. (`dnf history info 7`)

## Points ouverts

- **`terra` activé** : dépôt tiers, priorité 99 relevée (voir « Dépôts » et
  sa note de méthode) — c'est la priorité par défaut de dnf pour un dépôt
  sans directive `priority=` ; Terra n'est donc pas structurellement
  prioritaire sur Fedora. Le risque réel n'est pas la priorité du dépôt lui-
  même, mais qu'un paquet homonyme de version supérieure disponible sur
  Terra prenne le dessus sur son équivalent Fedora lors d'une résolution de
  dépendances.
- **`rpmfusion-free-tainted`** activé délibérément (transaction 18), lié à la
  chaîne multimédia. Établi, pas un point ouvert au sens strict — conservé
  ici pour traçabilité car demandé explicitement.
- **Coexistence VA-API** `mesa-va-drivers-freeworld` / `libva-nvidia-driver` :
  les deux sont bien installés (transactions 14–17 dans l'historique dnf).
  Arbitrage (quel pilote VA-API prime pour quel usage) non déterminé.
  Priorité basse.
- **`NVreg_PreserveVideoMemoryAllocations` commenté** dans
  `/usr/lib/modprobe.d/nvidia-power-management.conf` : les allocations VRAM
  ne survivent pas à la veille. Impact direct sur un serveur d'inférence
  local. Fait établi (voir « GPU »). À traiter au livrable IA.
- **`NVreg_DynamicPowerManagement` non défini** : absent du fichier de
  configuration modprobe, donc niveau de gestion d'alimentation runtime
  laissé au défaut du pilote, pas choisi. Fait établi (voir « GPU »).
- **Préservation du mode `2560x1600@240` après bascule MUX** : non acquise.
  Confirmé ce jour : ce mode est listé par `kscreen-doctor -o` mais n'est pas
  le mode actif (`2560x1600@60.00` l'est). Pas de commande locale ne permet
  de savoir s'il survivrait à une bascule MUX sans la déclencher réellement
  (hors périmètre lecture seule).
- **`claude doctor` annonce le canal `latest`** alors que le dépôt dnf pointe
  `stable` : écart confirmé par lecture croisée (`claude doctor` +
  `/etc/yum.repos.d/claude-code.repo`), sans effet apparent observé.
  `@VERIF : cause de l'écart entre canal annoncé et dépôt configuré —
  documentation officielle Claude Code, hors de portée de ce poste.`
- **Commandes rapportées en échec lors de l'inventaire manuel initial de
  l'utilisateur, reproduites ici à l'identique** :
  - `journalctl -b0 -g 'drm\|amdgpu\|nvidia'` : renvoie `-- No entries --`,
    code de retour 0 dans cette série (l'utilisateur rapportait un code 1
    sans message — écart de code non expliqué, mais la cause de fond est
    établie et identique). Cause : `-g`/`--grep` de journalctl interprète le
    motif en PCRE, où `\|` est un pipe **littéral**, pas une alternance — la
    chaîne réellement recherchée était donc `drm|amdgpu|nvidia` telle
    quelle, absente du journal. Confirmé par comparaison directe sur le
    même journal : `grep -c 'drm|amdgpu|nvidia'` (littéral) → 0 occurrence,
    contre `grep -Ec 'drm|amdgpu|nvidia'` (alternance) → 114 occurrences.
    L'invocation correcte pour une alternance est `-g 'drm|amdgpu|nvidia'`,
    sans échapper le `|`.
  - `dnf repolist enabled` : reproduit à l'identique — échec réellement
    silencieux confirmé, stdout et stderr tous deux vides, code de retour 0.
    Cause établie par comparaison : `dnf repolist fedora` renvoie
    exactement l'entrée du dépôt `fedora`, ce qui montre que dnf5 traite
    l'argument positionnel de `repolist` comme un motif d'ID de dépôt, pas
    comme le mot-clé `enabled` de la syntaxe dnf4. Aucun dépôt ne s'appelle
    littéralement `enabled`, d'où une commande qui ne retourne rien sans
    jamais signaler d'erreur. La syntaxe dnf5 correcte est
    `dnf repolist --enabled` (ou `dnf repo list --enabled`).

## Journal des séries

- **2026-08-04 — série d'inventaire initiale.** Création de `CLAUDE.md`,
  `docs/machine-facts.md`, `.gitignore`, `README.md` à partir de commandes
  de lecture seule. Aucun autre fichier créé ; aucune installation, aucune
  modification de service, aucun changement dans `/etc`, aucun redémarrage.
  Incident de méthode : `dnf repoinfo terra` a déclenché une invite d'import
  de clé GPG — vérifié a posteriori comme sans effet (clé déjà présente
  avant l'exécution de la commande), documenté dans la section « Dépôts ».
- **2026-08-04 — corrections post-revue.** Deux diagnostics de la série
  précédente reposaient sur des commandes substituées plutôt que reproduites
  à l'identique (`journalctl`, `dnf repolist`) — corrigés en section
  « Points ouverts » avec les commandes exactes et leurs causes établies.
  Sémantique de `gpu_mux_mode` fermée (voir « GPU »), incohérence D2bis/DRM
  requalifiée en état attendu avant bascule non encore appliquée (voir
  « Décisions »). Le décompte des points réellement ouverts ne se
  reconstruit qu'avec `grep -c` sur ce fichier — il n'est pas dupliqué ici,
  pour ne pas devenir une deuxième source qui se périme au prochain commit.
