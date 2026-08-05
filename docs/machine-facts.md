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
  - `asus-shutdown.service` (« ASUS Deferred Shutdown Handler ») : unité
    `disabled` mais `active (running)` depuis le démarrage de session
    (démarrée transitivement, pas via `systemctl enable`),
    `CapabilityBoundingSet=CAP_SYS_MODULE CAP_SYS_ADMIN`. Découverte le
    2026-08-04 en cherchant la cause d'un `pending_reboot` resté à `0`
    après écriture de `gpu_mux_mode` — voir « Décisions » (D2bis) et
    `docs/gpu-mux-recovery.md` § Résultats observés pour le détail complet.
    `asusd.service` déclare `Before=asus-shutdown.service` ;
    `asus-shutdown.service` est lui-même ordonné `Before=shutdown.target
    reboot.target halt.target` — cohérent avec un mécanisme d'application
    différée des attributs `asus-armoury` au moment de l'arrêt, pas
    immédiatement à l'écriture. (`systemctl cat asusd.service`,
    `systemctl cat asus-shutdown.service`, `systemctl status
    asus-shutdown.service`)
    **Rôle confirmé le 2026-08-05** par lecture de
    `journalctl -u asus-shutdown -b -1 --no-pager` (journal du boot
    précédent, exécuté dans la session de clôture) : l'unité a bien
    tourné à l'extinction précédant le redémarrage réussi, malgré
    `systemctl is-enabled` → `disabled` (démarrage à la demande, pas
    d'activation statique — mécanisme exact non confirmé, voir « Points
    ouverts »). Séquence observée : arrêt de `nvidia-powerd`,
    `nvidia-persistenced`, `nvidia-fabricmanager` ; cinq tentatives de
    `modprobe -r` sur la pile NVIDIA, **toutes en échec** (`modprobe:
    FATAL: Module nvidia_drm is in use.`) ; application quand même de
    l'attribut différé (`Applying deferred GPU attribute gpu_mux_mode =
    1`) ; relâchement de l'inhibiteur d'extinction logind. **Le
    déchargement des modules NVIDIA n'est donc pas nécessaire au succès
    de la bascule** — voir `docs/gpu-mux-recovery.md` § Résultats
    observés du 2026-08-05 pour le détail complet.
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

### État post-bascule MUX (2026-08-05, après redémarrage)

**Bascule confirmée réussie.** Attributs `asus-armoury`, relus dans cette
session après redémarrage : `gpu_mux_mode/current_value=1`,
`pending_reboot=0`, `dgpu_disable/current_value=0` (inchangé).
(`cat /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value`,
`cat .../pending_reboot`, `cat .../dgpu_disable/current_value`) — le
miroir déprécié confirme la même valeur : `cat
/sys/devices/platform/asus-nb-wmi/gpu_mux_mode` → `1`, `cat
/sys/devices/platform/asus-nb-wmi/dgpu_disable` → `0`.

**Renumérotation DRM — conséquence directe, à traiter comme une rupture
de nommage.** Avant bascule : `card0` = `simple-framebuffer` (pas de bus
PCI), `card1` = AMD (`09:00.0`), `card2` = NVIDIA (`01:00.0`). Après
bascule : plus de carte `simple-framebuffer` du tout, `card0` = NVIDIA
(`01:00.0`), `card1` = AMD (`09:00.0`, stable). Établi par comparaison de
deux relevés : le fichier de trace non versionné
`roles/gpu_mux/trace/pre-bascule-20260805T100255.json` (capturé
automatiquement par `roles/gpu_mux/` juste avant l'écriture réelle) pour
l'état « avant », et une lecture directe dans cette session pour l'état
« après » — `for c in /sys/class/drm/card*; do readlink -f "$c/device"; done`,
`ls -la /dev/dri/by-path/`. **Conséquence retenue** : toute configuration
qui référence `card2` (ou le chemin `card2-eDP-2`) est **périmée** ;
toute règle KWin ou script à venir doit être écrit sur les noms
post-bascule (`card0` = NVIDIA, `card1` = AMD).
- Connecteurs `card0` (NVIDIA) : `DP-4`, `DP-5`, `eDP-2`, `HDMI-A-1` — les
  quatre `disconnected`. (`cat /sys/class/drm/card0-*/status`)
- Connecteurs `card1` (AMD) : `DP-1` déconnecté, `DP-2` déconnecté,
  `DP-3` **connecté** (ScreenPad Plus, survie confirmée — voir
  `docs/gpu-mux-recovery.md`), `eDP-1` **connecté** (panneau principal,
  maintenant piloté par l'iGPU), `Writeback-1` (`unknown`, non
  significatif). (`cat /sys/class/drm/card1-*/status`)

**Renommage des sorties KScreen — même rupture de nommage, côté Wayland
cette fois.** `kscreen-doctor -o` : la sortie du panneau principal, nommée
`eDP-2` avant bascule (trace pré-bascule ci-dessus), est nommée `eDP-1`
après (lecture directe dans cette session). `DP-3` (ScreenPad Plus) est
inchangée. **Conséquence retenue** : toute règle KWin à venir référençant
`eDP-2` est périmée, à écrire sur `eDP-1`.

**Contention compositeur/dGPU — réduite, pas supprimée.** `nvidia-smi`
avant écriture réelle (trace `pre-bascule-20260805T100255.json`,
2026-08-05T10:02:57 CEST) : `P8`, `8W / 150W`, `1070MiB / 16376MiB`, neuf
processus dans la table, dont un seul de type `C+G`
(`kwin_wayland`, 85 MiB) et huit de type `G` (`plasma-keyboard`,
`Xwayland`, `plasmashell`, l'agent d'authentification polkit-kde,
`xwaylandvideobridge`, `xdg-desktop-portal-kde`, `rog-control-center`,
`firefox`). `nvidia-smi` après redémarrage, relu dans cette session :
`P0`, `31W / 155W` (mesure instantanée, `nvidia-smi -q -d POWER` donne
31,15 W en tirage moyen au même instant), `47MiB / 16376MiB`, **un seul**
processus, `kwin_wayland` (`C+G`, 13 MiB). Progrès réel (VRAM et nombre
de clients graphiques divisés respectivement par ~23 et 9), mais
`kwin_wayland` reste un client de la dGPU après bascule — l'objectif
D2bis n'est atteint que partiellement.

`roles/gpu_mux/trace/` n'est pas versionné (`.gitignore` propre à ce
répertoire) — un chiffre qui ne cite que son nom de fichier n'est pas
relisible par quiconque clone ce dépôt public. Extrait vérifié exempt de
toute donnée couverte par D4 (aucune IP, aucun nom d'hôte distant,
aucun secret — uniquement des identifiants de bus PCI et des noms de
processus locaux) et cité ici tel quel plutôt que le fichier versé en
entier, dont le reste (dump ANSI de `kscreen-doctor -o`, connecteurs DRM
déjà consignés ci-dessus) n'apporte rien de plus :

```
$ jq -r '.horodatage, .nvidia_smi[]' roles/gpu_mux/trace/pre-bascule-20260805T100255.json
2026-08-05T08:02:55Z
Wed Aug  5 10:02:57 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 610.43.03              KMD Version: 610.43.03     CUDA UMD Version: 13.3     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090 ...    Off |   00000000:01:00.0  On |                  N/A |
| N/A   46C    P8              8W /  150W |    1070MiB /  16376MiB |     17%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            2492    C+G   /usr/bin/kwin_wayland                    85MiB |
|    0   N/A  N/A            2592      G   /usr/bin/plasma-keyboard                130MiB |
|    0   N/A  N/A            2600      G   /usr/bin/Xwayland                        24MiB |
|    0   N/A  N/A            2662      G   /usr/bin/plasmashell                    145MiB |
|    0   N/A  N/A            2766      G   ...it-kde-authentication-agent-1          3MiB |
|    0   N/A  N/A            3069      G   /usr/bin/xwaylandvideobridge              3MiB |
|    0   N/A  N/A            3159      G   ...ibexec/xdg-desktop-portal-kde          3MiB |
|    0   N/A  N/A            4933      G   /usr/bin/rog-control-center              44MiB |
|    0   N/A  N/A            7689      G   /usr/lib64/firefox/firefox              273MiB |
+-----------------------------------------------------------------------------------------+
```

Voir « Points ouverts » pour l'état d'alimentation (`P0` au repos, dégradation par rapport à `P8` avant
bascule).

**Mode `2560x1600@240.00` — préservé, avec une nuance sur le marqueur
« préféré ».** `kscreen-doctor -o`, comparaison des deux relevés
ci-dessus : avant bascule, la sortie `eDP-2` liste `18:2560x1600@60.00*!`
(mode courant **et** marqué préféré) puis `19:2560x1600@240.00` (listé,
sans marqueur `!`). Après bascule, la sortie `eDP-1` liste
`18:2560x1600@240.00!` (marqué préféré, pas courant) puis
`19:2560x1600@60.00*` (courant, sans marqueur `!`). Le mode `@240.00` est
donc bien préservé et disponible dans les deux cas ; ce qui change, c'est
le mode que le pilote annonce comme préféré — il passe du `@60.00` au
`@240.00` après bascule. Ni l'un ni l'autre n'est le mode actif après
bascule (`@60.00` reste actif). Sélection manuelle requise dans les
réglages d'écran de Plasma pour activer `@240.00` — aucune régression par
rapport à l'état pré-bascule, où `@240.00` n'était déjà pas actif non
plus.

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

**[ÉTAT PRÉ-BASCULE — 2026-08-04, périmé depuis la bascule MUX du
2026-08-05, conservé pour l'historique, ne pas reproduire tel quel]**

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

### État post-bascule — 2026-08-05, après redémarrage (état courant)

**Renumérotation DRM et renommage KScreen — toute référence à `card2` ou
`eDP-2` ci-dessus est périmée.** Deux cartes DRM subsistent, autrement
numérotées : `card0` → bus PCI `01:00.0` (NVIDIA/nvidia, était `card2`),
`card1` → bus PCI `09:00.0` (AMD/amdgpu, numéro inchangé). Plus de carte
`simple-framebuffer` (occupait `card0` avant bascule).
(`for c in /sys/class/drm/card*; do readlink -f "$c/device"; done`,
comparé au fichier de trace non versionné
`roles/gpu_mux/trace/pre-bascule-20260805T100255.json`.)

- Connecteurs `card0` (NVIDIA) : `DP-4`, `DP-5`, `eDP-2`, `HDMI-A-1` —
  les quatre `disconnected`. (`cat /sys/class/drm/card0-*/status`)
- Connecteurs `card1` (AMD) : `DP-1` déconnecté, `DP-2` déconnecté,
  `DP-3` **connecté**, `eDP-1` **connecté** (panneau principal, était
  déconnecté avant bascule), `Writeback-1` (`unknown`, non significatif).
  (`cat /sys/class/drm/card1-*/status`)
- `kscreen-doctor -o` :
  - Sortie 1, `DP-3` (ScreenPad Plus, rattaché à `card1` AMD, nom
    inchangé) : activée, connectée, mode actif `3840x1100@60.02`,
    géométrie `0,1280 2743x786`, échelle 1.4, rotation 1 — identique au
    relevé pré-bascule.
  - Sortie 2, **renommée `eDP-1`** (était `eDP-2` avant bascule ; même
    panneau physique, maintenant rattaché à `card1` AMD au lieu de
    `card2` NVIDIA) : activée, connectée, mode actif `2560x1600@60.00`.
    Le mode `2560x1600@240.00` est **préservé**, listé, et désormais
    **marqué mode préféré** (`!`) — il ne l'était pas avant bascule ; il
    n'est toujours pas le mode actif. Géométrie `315,0 2048x1280`,
    échelle 1.25, rotation 1.
  (`kscreen-doctor -o`, comparé au même relevé pré-bascule que ci-dessus)
- **Conséquence retenue pour tout travail futur** : toute configuration
  Ansible, script ou règle KWin qui référence `card2` ou `eDP-2` est
  **périmée**. Les noms post-bascule (`card0` = NVIDIA, `card1` = AMD,
  `eDP-1` = panneau principal, `DP-3` = ScreenPad Plus) sont ceux à
  utiliser désormais.
- **Sélection manuelle requise** pour activer `2560x1600@240.00` : il est
  disponible et marqué préféré mais n'est pas le mode actif — dans les
  réglages d'écran de Plasma. Ce n'était déjà pas le mode actif avant
  bascule ; aucune régression.

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
**Corollaire précisé le 2026-08-05** : ce dépôt est désormais poussé sur
un distant — fait qui se lit dans `git` (état de la branche par rapport
à `origin`), donc non reconsigné ici par ailleurs. Le corollaire qui
importe n'est pas « existe sur un distant » (un fait ponctuel, déjà
vrai) mais **« poussé après chaque livrable »** : D1 n'est satisfaite que
si l'écart entre l'état local et l'état distant reste petit — un dépôt
distant qui existe mais retarde de plusieurs livrables ne protège pas
mieux qu'un dépôt qui n'existerait pas.

**D2 — [RETIRÉ le 2026-08-04].** ÉTAIT : supergfxd désactivé, dGPU active en
permanence. Motif d'origine : prévisibilité. Retiré au profit de D2bis.

**D2bis — mode graphique Optimus/Hybrid : décidée le 2026-08-04, non
appliquée.** Plasma sur l'iGPU AMD, RTX 4090 réservée au calcul. Motif :
supprimer la contention compositeur/inférence et rendre l'enveloppe 150 W au
calcul. Le gain VRAM (1079 MiB mesurés sur 16376 au moment de la décision)
est secondaire. Note : la mesure VRAM refaite dans cet inventaire (1045 MiB /
16376 MiB) est un nouveau relevé ponctuel, pas une reconduction du chiffre
d'origine — voir « GPU » ci-dessus. État constaté avant toute écriture
(`gpu_mux_mode=0`, panneau principal `eDP-2` câblé sur la carte NVIDIA) :
c'était l'état cohérent attendu avant bascule vers Optimus/Hybrid, pas une
anomalie — voir « GPU » pour la sémantique de `gpu_mux_mode`. Préalable
posé le 2026-08-04 : chemin de retour documenté et testé dans
[`docs/gpu-mux-recovery.md`](gpu-mux-recovery.md) (rôle `roles/recovery/`),
avant toute écriture réelle dans `gpu_mux_mode`.

Écriture réellement tentée le 2026-08-04 (rôle `roles/gpu_mux/`) :
`asusctl armoury set gpu_mux_mode 1` exécuté avec succès (code retour 0),
mais `pending_reboot` n'est pas passé à `1` dans la fenêtre observée — le
journal `asusd` montre la valeur mise en file d'attente (« delayed apply »)
plutôt qu'appliquée de façon synchrone, probablement via
`asus-shutdown.service` (voir « GPU » § Services, et
`docs/gpu-mux-recovery.md` § Résultats observés pour le détail complet et
les deux points non résolus qui en découlent). Le rôle s'est donc arrêté
en échec sur sa propre garde plutôt que de déclarer un succès non prouvé —
comportement voulu, pas un bug du rôle. **Bascule toujours non appliquée à
la date de cet inventaire ; aucun redémarrage n'a eu lieu.** Prochaine
étape : redémarrage, décision et déclenchement manuels, hors de ce
livrable.

**[APPLIQUÉE le 2026-08-05]** Écriture réelle réussie entre les deux
sessions précédentes (`ansible-playbook --ask-become-pass
roles/gpu_mux/gpu_mux.yml`, voir D2ter révocation partielle ci-dessous
pour le mécanisme d'écriture retenu), suivie d'un redémarrage réel.
Vérifié dans la session de clôture, après redémarrage :
`gpu_mux_mode/current_value=1`, `pending_reboot=0`, topologie DRM
conforme (panneau principal rattaché à l'iGPU AMD, RTX 4090 sans sortie
active) — voir « Affichage » et « GPU » § État post-bascule ci-dessus
pour le détail complet et sourcé. Contrepartie identifiée, non anticipée
par la décision d'origine : la dGPU reste en `P0` au repos après bascule
(contre `P8` avant), voir « Points ouverts ».

**D2ter (2026-08-04) — bascule sans supergfxd, via `asusctl armoury`.**
Motif : `asusd` est déjà actif ; le README de `supergfxctl` signale un
conflit entre commutateurs GPU et un risque de repli sur `integrated` au
démarrage, ce qui couperait la dGPU. Corroboré ce jour : `supergfxctl -g`
échoue explicitement (« supergfxd is not enabled ») — le chemin `asusd`/
`asus-armoury` est bien celui en usage réel sur cette machine, pas
`supergfxctl`.

**D2ter — révocation partielle (2026-08-05).** La bascule MUX se fait par
**écriture directe** dans
`/sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value`,
et non par `asusctl armoury set`. Le refus de `supergfxd` (motif
d'origine : conflit entre commutateurs GPU) est **maintenu** — l'écriture
directe l'évite tout autant. Motif de la révocation : `asusd` met la
valeur en file d'attente mémoire pour application différée par
`asus-shutdown.service`, unité qui est `disabled` sur ce système ; la file
n'est persistée nulle part et `pending_reboot` reste à `0`. L'indirection
par `asusd` échange une confirmation vérifiable contre une intention non
persistée — repli silencieux caractérisé. L'écriture directe restaure
`pending_reboot` comme preuve avant redémarrage. [RÉVOQUÉE EN PARTIE le
2026-08-05] ÉTAIT : « bascule sans supergfxd, via `asusctl armoury` »
(texte complet ci-dessus, conservé) — la partie « via `asusctl armoury` »
est révoquée, la partie « sans supergfxd » reste en vigueur.

**[MOTIF CORRIGÉ le 2026-08-05]** Le motif de révocation ci-dessus repose
sur une prémisse fausse, démentie par le journal du boot précédent le
redémarrage réussi (`journalctl -u asus-shutdown -b -1`, voir « GPU » §
Services) : `asus-shutdown.service` a bel et bien tourné à l'extinction
et **a appliqué** la valeur mise en file par `asusd`
(`Applying deferred GPU attribute gpu_mux_mode = 1`), alors qu'il est
rapporté `disabled` par `systemctl is-enabled` — démarré à la demande,
vraisemblablement par `asusd` lui-même lorsqu'un attribut est mis en
file (vocabulaire d'inhibiteur logind et découpage en phases numérotées
dans son propre journal, cohérents avec cette hypothèse sans la
démontrer ; mécanisme exact non confirmé — point compté et marqué en
« Points ouverts », pas ici, pour ne pas dupliquer le marqueur).

**ÉTAIT** (motif de révocation d'origine, ci-dessus, conservé) : « la
file d'attente d'`asusd` n'est jamais consommée car `asus-shutdown` est
désactivé » — **faux**.

**MOTIF CORRIGÉ** : l'écriture directe reste le chemin retenu, mais pour
une autre raison. Elle fournit une **confirmation vérifiable avant
redémarrage** — `pending_reboot` passe à `1`. La voie `asusctl armoury
set` laisse `pending_reboot` à `0` : l'opération réussit probablement (la
file est bien consommée par `asus-shutdown.service` à l'extinction, comme
démontré ci-dessus), mais rien ne permet de l'attester avant de
redémarrer. La préférence tient à la **vérifiabilité**, pas à une
impossibilité de la voie `asusd`.

**Doute résiduel à consigner** : le 2026-08-05, l'écriture directe et la
file d'`asusd` (mise en file depuis la tentative du 2026-08-04, jamais
purgée) portaient la même valeur cible (`gpu_mux_mode = 1`). Laquelle des
deux a effectivement produit le changement matériel observé est
indéterminable a posteriori — sans conséquence pratique ici puisque les
deux menaient à la même cible, mais cela interdit d'affirmer que c'est
l'écriture directe qui a fonctionné du seul fait que la bascule a réussi.

**Leçon de méthode** : `systemctl is-enabled X` → `disabled` signifie
« pas d'activation statique au démarrage », **pas** « ne s'exécutera
jamais ». Une unité peut être démarrée à la demande par un autre service,
par D-Bus ou par activation par socket — c'est le cas ici. Voir le détail
complet, avec le journal cité intégralement, dans
`docs/gpu-mux-recovery.md` § Résultats observés du 2026-08-05.

**D3 — chaîne Ansible EE-first.** `ansible-navigator` exécute et vérifie dans
l'image d'Execution Environment. L'`ansible-core` système (2.20.7 /
Python 3.14) sert à l'édition et ne fait pas autorité : il est plus récent
que la cible AAP 2.6 et accepterait des constructions que la production
refuse. Note datée (2026-08-04) : la cible EE-first est maintenue, mais son
moyen d'installation reste à déterminer — `ansible-navigator` n'est pas
empaqueté dans les dépôts Fedora 44 activés sur ce poste (voir « Points
ouverts »), les options ouvertes sont `pipx`, un venv dédié, ou l'exécution
directe des EE en `podman run`. `@VERIF : voie d'installation de la chaîne
Ansible EE-first, à trancher au livrable dédié.` Motif : une décision dont
le moyen d'exécution n'existe pas encore doit le dire, sinon elle se
découvre inapplicable au moment de l'appliquer.

**D4 (2026-08-04, amendée) — dépôt public.** Interdits : secrets de toute
nature (clés privées, jetons, mots de passe), données d'entreprise,
adressage d'infrastructure interne (IP, sous-réseaux, VLAN), noms de
serveurs ou de domaines professionnels, identifiants de comptes de service.
Acceptés explicitement : hostname et nom d'utilisateur de ce poste
personnel, ainsi que l'identité de l'auteur dans l'historique git. Motif de
l'amendement : ces identifiants sont indissociables des faits consignés
(chemins `graphRoot` de podman, `$HOME`, sorties de commandes) ; les
expurger dégraderait la traçabilité sans rien protéger, puisque l'identité
de l'auteur figure déjà dans chaque commit. [AMENDÉE le 2026-08-04] ÉTAIT :
« Aucun secret, aucune donnée d'entreprise, aucune adresse interne, aucun
nom d'hôte réel. » — formulation inapplicable en pratique, violée dès le
commit `d7f93be` (hostname `Zephyrus-MM`, chemin `graphRoot` contenant le
nom d'utilisateur, tous deux en section « Conteneurs »/« Système ») sans que
la revue le détecte à l'époque.

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

- **[FERMÉ le 2026-08-05] Rôle exact d'`asus-shutdown.service`** (« ASUS
  Deferred Shutdown Handler », capacités `CapabilityBoundingSet=CAP_SYS_MODULE
  CAP_SYS_ADMIN`, `SystemCallFilter=@system-service @module` — voir « GPU »
  § Services). ÉTAIT : « hypothèse non confirmée d'un déchargement des
  modules NVIDIA avant commutation MUX ». **Établi par le journal du boot
  précédent le redémarrage réussi** (`journalctl -u asus-shutdown -b -1
  --no-pager`, lu dans la session de clôture) : l'unité arrête les
  services NVIDIA (`nvidia-powerd`, `nvidia-persistenced`,
  `nvidia-fabricmanager`), tente `modprobe -r` sur la pile NVIDIA —
  **cinq tentatives, toutes en échec** (`modprobe: FATAL: Module
  nvidia_drm is in use.`), émet `Failed to unload NVIDIA modules after 5
  attempts` —, puis **applique quand même** l'attribut différé
  (`Applying deferred GPU attribute gpu_mux_mode = 1`) et relâche son
  inhibiteur d'extinction. **Conclusion : le déchargement des modules
  n'est pas nécessaire au succès de la bascule** — la bascule a réussi
  malgré l'échec des cinq tentatives. Détail complet dans
  `docs/gpu-mux-recovery.md` § Résultats observés du 2026-08-05.
  Nouveau point ouvert issu de cette même investigation : le mécanisme
  précis par lequel l'unité, `disabled` au sens de `systemctl is-enabled`,
  démarre malgré tout à l'extinction — voir plus bas.
- **`terra` activé** : dépôt tiers, priorité 99 relevée (voir « Dépôts » et
  sa note de méthode) — c'est la priorité par défaut de dnf pour un dépôt
  sans directive `priority=` ; Terra n'est donc pas structurellement
  prioritaire sur Fedora. Le risque réel n'est pas la priorité du dépôt lui-
  même, mais qu'un paquet homonyme de version supérieure disponible sur
  Terra prenne le dessus sur son équivalent Fedora lors d'une résolution de
  dépendances.
- **Clé OpenPGP Terra — import déclenché de façon asymétrique selon le
  contexte d'exécution.** Un `dnf` non privilégié (cette session, utilisateur
  courant) déclenche une demande d'import de la clé `0xDE226D6F`
  (« Terra 44 <security@fyralabs.com> ») dès qu'une opération touche les
  métadonnées du dépôt — voir l'incident `dnf repoinfo terra` documenté en
  section « Dépôts ». Une exécution privilégiée ne la déclenche pas : le
  trousseau système la connaît déjà, très probablement depuis la
  transaction 5 (`dnf install --nogpgcheck --repofrompath terra…
  terra-release`, voir « Chaîne Ansible » et l'historique dnf), qui a
  installé `terra-release` sans vérification de signature — c'est
  vraisemblablement l'origine de cette clé dans le trousseau système, mais
  la transaction elle-même ne mentionne pas d'import de clé, seulement
  l'installation du paquet porteur. Trousseaux et caches de confiance dnf5
  sont donc distincts entre exécution privilégiée et non privilégiée sur ce
  poste. Empreinte complète capturée lors de l'incident :
  `AE09157A4DE88B497EA1D5D300CDAB43DE226D6F`. `@VERIF : empreinte complète
  de la clé Terra à comparer à celle publiée par Fyra Labs avant tout
  (ré)import ; l'ID court 0xDE226D6F ne prouve rien à lui seul.` Aucune
  action dans ce livrable : rien n'est importé, aucun dépôt n'est modifié
  ou désactivé — Terra n'est requis par aucun composant de la chaîne
  actuelle.
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
- **[RÉSOLU le 2026-08-05] Préservation du mode `2560x1600@240` après
  bascule MUX.** ÉTAIT : « non acquise ... pas de commande locale ne
  permet de savoir s'il survivrait à une bascule MUX sans la déclencher
  réellement ». **Résolu par la bascule réelle** : le mode est préservé,
  listé, et même marqué mode préféré (`!`) après bascule — il ne l'était
  pas avant. Il n'est pas le mode actif ni avant ni après ; aucune
  régression. Détail sourcé complet en « Affichage » § État post-bascule.
- **Nouveau point ouvert (2026-08-05) — mécanisme exact de démarrage à la
  demande d'`asus-shutdown.service`.** Établi (voir ci-dessus, point
  fermé) : l'unité tourne à l'extinction malgré `systemctl is-enabled` →
  `disabled`. Non établi : *comment* elle démarre. Hypothèse retenue mais
  non démontrée : démarrage déclenché par `asusd` lui-même au moment où un
  attribut est mis en file d'attente — corroborée par le vocabulaire
  d'inhibiteur logind employé dans le propre journal de l'unité et par son
  découpage en phases numérotées (orienté exécution ponctuelle, pas
  service de fond), mais aucune des deux observations ne prouve le
  mécanisme de déclenchement lui-même (`Wants=`/`StartTransientUnit=` côté
  `asusd`, règle D-Bus activable, ou autre). `@VERIF : mécanisme exact de
  démarrage à la demande d'asus-shutdown.service — à confirmer par lecture
  du code source d'asusd/asus-shutdown, ou par un test d'extinction avec
  journal détaillé au niveau DEBUG côté systemd (property
  Wants=/BindsTo=/PropagatesStopTo= et unités de type D-Bus activable).`
- **[CORRIGÉ le 2026-08-05, même jour] dGPU en `P0` au repos après
  bascule — mesure erronée par manque de relevé espacé, pas une
  régression.** **ÉTAIT** : « dGPU maintenue en `P0` au repos après
  bascule, contre `P8` avant... en l'état, la consommation au repos est
  dégradée par rapport à l'état pré-bascule », avec un point non résolu
  sur la cause et la voie de correction. **CORRIGÉ** : résolution dédiée dans
  [`docs/dgpu-power.md`](dgpu-power.md), méthode par relevés `sysfs`
  espacés (compteurs passifs `power/runtime_*`, jamais réveillés par leur
  propre lecture) sans invoquer `nvidia-smi` entre deux mesures.
  Résultat : sur 8 830 s d'uptime au dernier relevé, `0000:01:00.0` passe
  **99,97 % du temps en veille runtime** (`Runtime D3 status: Enabled
  (fine-grained)`, `Video Memory: Off`, lus dans
  `/proc/driver/nvidia/gpus/0000:01:00.0/power` — interface propre au
  pilote NVIDIA, documentée dans
  `/usr/share/doc/xorg-x11-drv-nvidia/html/dynamicpowermanagement.html`,
  local, amont). Le `P0`/`32 W` du 2026-08-05 était réel **au moment
  mesuré**, mais provoqué par la mesure elle-même : deux sondes
  `nvidia-smi` délibérées, isolées et espacées, ont chacune réveillé le
  périphérique de façon instantanée et reproductible (`runtime_status`
  passant de `suspended` à `active` dans la même seconde que l'appel),
  suivi d'un retour spontané en veille ~20-23 s plus tard sans nouvelle
  sonde. **Motif de l'erreur** : la mesure du 2026-08-05 (voir
  `docs/gpu-mux-recovery.md`, § « Résultats observés — bascule réussie »)
  enchaînait plusieurs appels `nvidia-smi` rapprochés sans relevé de
  contrôle espacé entre eux — troisième mode d'échec consigné dans
  `CLAUDE.md` § Sourcing (aux côtés de la commande substituée et de la
  transposition entre interfaces) : une mesure exacte mais dont l'effet
  de bord (le réveil qu'elle provoque elle-même) n'a pas été isolé de ce
  qu'elle prétend observer. `kwin_wayland` apparaît bien comme client
  `nvidia-smi` (`C+G`, 13 MiB) à chaque sonde, mécanisme caractérisé dans
  `docs/dgpu-power.md` § Ce qui retient le périphérique
  (`KWIN_DRM_DEVICES` non défini → énumération automatique de tout GPU du
  siège par KWin) — sans que cela empêche les 99,97 % de veille mesurés.
  **Aucune action recommandée dans ce livrable** ; une option
  (`KWIN_DRM_DEVICES`) reste documentée mais non retenue, son coût
  n'étant pas compensé par un gain mesurable — voir
  `docs/dgpu-power.md` § Recommandation pour l'argumentaire complet et le
  point non résolu résiduel sur son risque (fonctionnalités Plasma
  dépendant de l'énumération KWin).
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
- **Dette technique — outillage Ansible EE-first non installable tel quel.**
  Relevé ce jour, dépôt `terra` désactivé pour la requête (aucune
  modification, aucun import) : `ansible-builder-3.1.0-8.fc44` est
  disponible dans le dépôt `fedora` ; `ansible-navigator`, `ansible-runner`
  et `ansible-lint` sont **absents** des dépôts Fedora 44 activés sur ce
  poste. `pipx-1.15.0-2.fc44` (dépôt `updates`) et
  `python3-pip-26.0.1-2.fc44` (dépôt `fedora`) sont disponibles mais non
  installés ; `python3 -m pip` échoue (`No module named pip`).
  (`dnf --disablerepo=terra list --available ansible-builder
  ansible-navigator ansible-runner ansible-lint pipx python3-pip`,
  `rpm -q pipx python3-pip`, `python3 -m pip --version`)
  Deux conséquences :
  - `roles/recovery/` — et le rôle `gpu_mux` qui exécutera la bascule
    elle-même, pas encore écrit à la date de cet inventaire — sont/seront
    des **rôles d'amorçage validés manuellement** (`--syntax-check`,
    `--check`, exécution réelle, démonstrations d'échec forcé), pas passés
    au lint : `ansible-lint` n'existe sur aucune voie d'installation testée
    sur ce poste à ce jour. À repasser au lint une fois la chaîne Ansible
    établie.
    Motif de l'ordre retenu : le basculement MUX conditionne CDI, les
    conteneurs et la construction des EE — outiller Ansible avant de
    basculer reviendrait à bâtir l'outillage sur une couche (le mode
    graphique) qui va elle-même changer.
  - Voir la note datée ajoutée à D3, qui porte son propre marqueur non
    résolu : la cible EE-first est maintenue, mais son moyen d'installation
    (`pipx`, venv dédié, ou `podman run` direct des EE) reste à trancher au
    livrable dédié à la chaîne Ansible.

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
- **2026-08-04 — chemin de retour avant bascule MUX.** Création de
  `docs/gpu-mux-recovery.md` et du rôle `roles/recovery/` (cible
  `localhost`) : préparent et vérifient un chemin de retour SSH — service
  installé, activé, démarré, `authorized_keys` non vide, état firewalld
  relevé — avant toute bascule réelle de `gpu_mux_mode`. Le rôle a été
  exécuté réellement dans cette série. Changement réel produit : `sshd`
  passe de `active`/`disabled` à `active`/`enabled`. Avant ce rôle, `sshd`
  tournait déjà (`active`) mais n'était pas activé au démarrage
  (`disabled`) — la connectivité SSH fonctionnait sur le boot courant mais
  n'aurait pas survécu à un redémarrage. Motif de la distinction :
  `is-active` et `is-enabled` sont deux faits systemd indépendants ; un test
  de connexion SSH réussi avant bascule aurait prouvé l'état courant, pas
  l'état après le redémarrage qu'une bascule MUX ratée impose justement.
  Rôle exécuté aussi en `--check` et en échec forcé à deux reprises
  (paquet SSH absent simulé, `authorized_keys` introuvable simulé) — les
  deux gardes cassent avec le message attendu. Confirmé après coup et à
  nouveau en fin de série par lecture directe : `gpu_mux_mode=0`,
  `pending_reboot=0`, `dgpu_disable=0` inchangés, `supergfxd` toujours
  `inactive`/`disabled` — ce rôle n'a jamais touché à ces attributs. Deux
  points restent non résolus dans `docs/gpu-mux-recovery.md` (direction de
  la connexion SSH entrante, survie du ScreenPad Plus après bascule) : ils
  y sont marqués et comptés, pas ici, pour éviter une deuxième source. Trois
  amendements apportés à `CLAUDE.md` § Sourcing à cette occasion (portée
  exacte d'une reproduction de commande, recevabilité d'une source externe
  faisant autorité, retrait d'un marqueur uniquement après vérification
  effective — jamais par nettoyage).
- **2026-08-04 — amendement D4 et dette Ansible (rattrapé ici : cette
  entrée manquait à sa propre série, ajoutée après coup sans en changer la
  date réelle).** D4 amendée (hostname et nom d'utilisateur de ce poste
  personnel explicitement acceptés, adressage réseau — y compris RFC 1918
  — toujours interdit en dur) suite à un écart signalé par cette série
  elle-même entre la formulation d'origine et la pratique constatée depuis
  le premier commit. Point ouvert ajouté sur la clé OpenPGP Terra
  (déclenchement asymétrique de l'import selon le contexte d'exécution,
  empreinte complète consignée). Dette technique sur l'outillage Ansible
  EE-first consignée (`ansible-navigator`/`ansible-runner`/`ansible-lint`
  absents des dépôts activés) avec note datée sur D3. Titre de la section
  ScreenPad Plus corrigé dans `docs/gpu-mux-recovery.md` (le corps n'a pas
  changé). Aucune installation, aucun import de clé, aucune modification
  de dépôt.
- **2026-08-04 — tentative de bascule vers Optimus/Hybrid (rôle
  `roles/gpu_mux/`).** Résolution en lecture seule d'abord :
  `asusctl armoury set --help` lu avant d'écrire la moindre tâche —
  syntaxe `asusctl armoury set <property> <value>` confirmée, pas
  présumée ; `asusctl armoury list` déclare exactement les 11 attributs
  présents sous `attributes/` (`pending_reboot` mis à part, simple
  indicateur, pas une propriété pilotable). Pré-vol : rôle `recovery`
  ré-exécuté réellement, entièrement idempotent (`changed=0`) ; direction
  SSH entrante fermée ce jour par preuve locale (`journalctl -u sshd -g
  'Accepted'`, connexion par clé publique acceptée depuis la machine
  distante) — voir `docs/gpu-mux-recovery.md`, qui porte ce marqueur
  fermé, pas ce fichier.
  Rôle `roles/gpu_mux/` validé avant écriture réelle : `--syntax-check`
  passé, `--check` sans écriture ni vérification post-écriture (gardées
  par `when: not ansible_check_mode`), deux échecs forcés casqués avant
  toute action (valeur cible hors `possible_values` ; `dgpu_disable`
  simulé à `1`, jamais écrit réellement).
  Écriture réelle exécutée : `asusctl armoury set gpu_mux_mode 1` a
  réussi (code retour 0), mais `pending_reboot` n'est pas passé à `1`
  dans la fenêtre observée — le rôle s'est arrêté en échec sur sa propre
  garde plutôt que de déclarer un succès non prouvé. Cause identifiée en
  lecture seule : le journal `asusd` montre la valeur mise en file
  d'attente (« delayed apply ») plutôt qu'appliquée de façon synchrone.
  Service `asus-shutdown.service` découvert à cette occasion (voir « GPU »
  § Services) et retenu comme piste d'application réelle, sans être
  confirmé par lecture de code source — deux points non résolus en
  découlent, consignés et comptés dans `docs/gpu-mux-recovery.md`, pas
  ici. Instantané pré-bascule capturé dans un fichier de trace non
  versionné (`roles/gpu_mux/trace/`, ignoré via un `.gitignore` propre à
  ce répertoire plutôt que le `.gitignore` racine, pour rester strictement
  dans le périmètre `roles/gpu_mux/`).
  Confirmé en fin de série : `dgpu_disable=0` inchangé, `supergfxd`
  toujours `inactive`/`disabled` (jamais démarré ni activé), aucun paquet
  installé, aucun dépôt modifié, aucune clé GPG importée, **aucun
  redémarrage**. Bascule toujours non appliquée à la date de cet
  inventaire — décision de redémarrer laissée à l'opérateur, liste de
  commandes de vérification post-redémarrage préparée dans
  `docs/gpu-mux-recovery.md` sans être exécutée.
- **2026-08-05 — révocation partielle de D2ter, écriture directe.**
  Diagnostic de la série précédente confirmé par investigation en lecture
  seule, sans redémarrer `asusd` ni écrire : `asus-shutdown.service`
  (seule consommatrice possible de la file d'attente `gpu_mux_mode`
  d'`asusd`) est `disabled` ; `/var/lib/asusd/` n'existe pas ; `asusd.ron`
  date d'avant la mise en file ; `asusctl armoury get gpu_mux_mode` ne
  restitue aucune valeur en attente. La file est donc purement en mémoire
  et n'est jamais consommée — voir D2ter (révocation partielle) et
  `docs/gpu-mux-recovery.md` § Résultats observés du 2026-08-05 pour le
  détail complet, dont l'investigation d'interférence (garde 2 de la
  demande) : établi qu'`asusd` ne recharge `gpu_mux_mode` qu'à son propre
  démarrage et ne surveille par inotify que son fichier de configuration ;
  non établi (faute de privilège pour inspecter les descripteurs de
  fichiers du processus) qu'aucun mécanisme de surveillance sysfs
  n'existe — limite consignée telle quelle, pas comme une certitude.
  Rôle `roles/gpu_mux/` adapté : gardes inchangées, seul le mécanisme
  d'écriture remplace `asusctl armoury set` par une écriture directe dans
  `current_value` (`become: true`, seule tâche privilégiée du rôle).
  `--syntax-check`, `--check` et les deux échecs forcés rejoués à
  l'identique, mêmes résultats que la série précédente. Pré-vol : rôle
  `recovery` ré-exécuté, de nouveau entièrement idempotent ; preuve de
  connexion SSH entrante rejouée avec succès (nouvelle connexion par clé
  publique acceptée, en plus de celles déjà journalisées).
  **L'écriture réelle n'a pas pu être tentée sur le matériel** : la tâche
  privilégiée a échoué immédiatement (« sudo: a password is required »,
  code retour 2) avant tout accès au fichier cible — ni un échec de garde
  ni une tentative partielle, une limite d'environnement (pas de `sudo`
  sans mot de passe configuré pour ce compte). Confirmé en fin de série :
  `gpu_mux_mode=0`, `pending_reboot=0`, `dgpu_disable=0` inchangés,
  `asus-shutdown` et `supergfxd` toujours `disabled`, aucun paquet
  installé (`sudo` déjà présent de base), aucun dépôt modifié, aucune clé
  GPG importée, **aucun redémarrage**.
- **2026-08-05 — bascule réussie et clôture de série documentaire (cette
  série, en lecture seule uniquement).** Entre la série précédente et
  celle-ci, hors du périmètre de tout livrable documenté (pas cette
  session) : le blocage `sudo` a été levé côté opérateur, l'écriture
  réelle a réussi, suivie d'un redémarrage réel. `~/.bash_history`
  suggérait la séquence (`ansible-playbook --ask-become-pass
  roles/gpu_mux/gpu_mux.yml` puis `sudo reboot`), mais un historique
  shell atteste une intention tapée, pas un résultat (`CLAUDE.md` §
  Sourcing des faits) — cette série a donc établi les faits par une
  source plus forte, le journal systemd persistant (`journalctl -g
  'sudo|ansible|gpu_mux_mode' --since ... --until ...`, relu dans cette
  session) : écriture directe `echo 1 > .../gpu_mux_mode/current_value`
  confirmée exécutée en tant que root (session `sudo` root ouverte puis
  fermée à 10:02:58, `asusd` réagissant immédiatement), redémarrage
  confirmé par l'audit système (`cmd="reboot"`, 10:27:26) — détail complet
  dans `docs/gpu-mux-recovery.md` § Résultats observés. Ce que même ce
  journal ne montre pas — la valeur exacte de `pending_reboot` lue
  immédiatement après l'écriture, `journalctl` ne capturant que
  l'invocation, pas le stdout — reste un point non résolu, marqué dans
  `docs/gpu-mux-recovery.md`, pas ici, pour ne pas dupliquer le marqueur.
  **Note ajoutée le 2026-08-05 (série suivante)** : ce fait relève de la
  classe « observation rapportée par l'opérateur » ajoutée depuis dans
  `CLAUDE.md` § Sourcing — sa résolution n'est pas « aller relire » (la
  fenêtre d'observation est close) mais **corriger le dispositif de
  trace** de `roles/gpu_mux/`, qui capture un instantané pré-bascule
  (`trace/pre-bascule-*.json`) sans capturer d'instantané post-écriture
  équivalent. Correction à apporter au rôle dans un livrable ultérieur
  touchant `roles/gpu_mux/` — pas appliquée dans celui-ci (périmètre
  documentaire strict).
  Cette série se limite à vérifier et consigner l'état résultant, par
  lecture seule exclusivement — aucune écriture système, aucune
  installation, aucun rôle exécuté.
  Vérifié dans cette session par commande directe : `gpu_mux_mode/
  current_value=1`, `pending_reboot=0`, `dgpu_disable=0` (inchangé), les
  deux chemins `asus-armoury` et `asus-nb-wmi` (déprécié) concordants.
  Deux points non résolus fermés par la preuve : survie du ScreenPad Plus
  (`docs/gpu-mux-recovery.md`, comparaison du fichier de trace non
  versionné `roles/gpu_mux/trace/pre-bascule-20260805T100255.json` et
  d'une lecture post-redémarrage) ; rôle exact d'`asus-shutdown.service`
  (`journalctl -u asus-shutdown -b -1`, cinq tentatives `modprobe -r` en
  échec puis application quand même de l'attribut différé — le
  déchargement des modules n'est pas nécessaire au succès de la bascule).
  Motif de la révocation partielle de D2ter corrigé : la prémisse
  d'origine (« la file d'`asusd` n'est jamais consommée car
  `asus-shutdown` est désactivé ») est fausse, démentie par ce même
  journal ; le motif retenu devient la vérifiabilité (`pending_reboot`)
  plutôt qu'une impossibilité de la voie `asusd` — avec un doute résiduel
  consigné (écriture directe et file `asusd` portaient la même valeur
  cible, laquelle a agi est indéterminable). Règle fausse corrigée dans
  `CLAUDE.md` § Matériel spécifique : `current_value` reflète l'écriture
  immédiatement (état voulu), pas l'état réalisé par le MUX matériel —
  l'ancienne règle avait été transposée par analogie depuis l'attribut
  déprécié `asus-nb-wmi` sans vérification sur le chemin réellement
  utilisé ; règle générale ajoutée contre ce type de transposition non
  mesurée. Renumérotation DRM (`card2`→`card0` NVIDIA, `card1` stable
  AMD) et renommage KScreen (`eDP-2`→`eDP-1`) consignés en « Affichage »
  et « GPU », avec la conséquence retenue que toute configuration future
  référençant les anciens noms est périmée. Deux nouveaux points non
  résolus ouverts : mécanisme exact de démarrage à la demande
  d'`asus-shutdown.service` ; cause du maintien de la dGPU en `P0` au
  repos après bascule (contre `P8` avant) et voie de correction — nouveau
  point ouvert, D2bis atteint seulement partiellement (`kwin_wayland`
  reste un client de la dGPU, VRAM et nombre de processus graphiques
  nettement réduits mais pas nuls). Point ouvert « Préservation du mode
  2560x1600@240 » résolu (préservé, et même devenu le mode marqué
  préféré). D2bis marquée appliquée. Aucune écriture système dans cette
  série : `docs/gpu-mux-recovery.md`, `CLAUDE.md` et ce fichier sont les
  seuls fichiers modifiés.
- **2026-08-05 — hygiène `roles/` et résolution alimentation dGPU (série
  suivante, en lecture seule uniquement).** Deux volets.
  **Hygiène** : la règle `pending_reboot` recopiée dans sa version fausse
  a été trouvée dans `roles/gpu_mux/README.md` et les commentaires/messages
  de `roles/gpu_mux/tasks/main.yml` (dont une justification technique
  inventée, « piège ACPI », jamais vérifiée) — remplacée par un renvoi à
  `CLAUDE.md` aux deux endroits, `--syntax-check` rejoué (passé), aucune
  tâche/variable/logique modifiée (seuls des commentaires et des chaînes
  de message). Nouvelle classe de source ajoutée à `CLAUDE.md` §
  Sourcing : « observation rapportée par l'opérateur » — plus faible
  qu'une trace relisible, plus forte qu'une inférence d'historique ;
  conséquence consignée ci-dessus (§ Journal, entrée précédente) sur le
  dispositif de trace de `roles/gpu_mux/` à corriger dans un livrable
  ultérieur.
  **Alimentation dGPU** : résolution complète dans
  [`docs/dgpu-power.md`](dgpu-power.md) (nouveau document). Le point
  ouvert « dGPU maintenue en `P0` » ouvert dans la série précédente est
  **corrigé** ci-dessus (§ Points ouverts) : régression non confirmée,
  mesure initiale entachée par l'absence de relevé espacé — troisième
  mode d'échec de sourcing, consigné dans `CLAUDE.md` aux côtés des deux
  précédents. Mesure refaite avec isolement explicite de l'effet de la
  sonde (`nvidia-smi`) : 99,97 % de veille runtime confirmés sur
  `0000:01:00.0`, `Runtime D3 status: Enabled (fine-grained)`, `Video
  Memory: Off` au repos, remontée à `P0` strictement causée par
  l'invocation de `nvidia-smi` elle-même (deux sondes délibérées sur
  deux, effet reproduit et chronométré). Mécanisme de `kwin_wayland`
  comme client `nvidia-smi` caractérisé (`KWIN_DRM_DEVICES` non défini,
  confirmé par trois lectures locales concordantes et par le code source
  amont de KWin) sans qu'il empêche la veille profonde. Aucune correction
  appliquée ni recommandée dans ce livrable — une option documentée
  (`KWIN_DRM_DEVICES`) reste non retenue, un point non résolu sur son
  risque étant compté dans `docs/dgpu-power.md`, pas ici.
  Confirmé en fin de série : aucun paquet installé, aucun service modifié,
  aucun fichier hors dépôt modifié, **aucun redémarrage** — les seules
  commandes exécutées sont des lectures (`cat`, `journalctl`, `nvidia-smi`
  en sondes délibérées et assumées comme réveillant le périphérique,
  `strings`, `modinfo`, `systemctl show/is-active/is-enabled`,
  `ansible-playbook --syntax-check`).
- **2026-08-05 — accès GPU depuis Podman rootless, résolution en lecture
  seule (série suivante).** Nouveau document
  [`docs/gpu-containers.md`](gpu-containers.md). Établi avant toute
  conclusion : Podman 5.8.4 consomme CDI nativement (`--cdi-spec-dir`,
  répertoires par défaut `/etc/cdi` et `/var/run/cdi`, aucun des deux
  peuplé sur ce poste) — aucun composant tiers requis pour la
  consommation, seule la spécification manque. `nvidia-container-toolkit`
  absent des dépôts Fedora/RPM Fusion/`updates` activés et de Terra
  (recherché avec `--disablerepo=terra` et `--repo=terra --assumeno`,
  aucune invite déclenchée — l'avertissement « Signing key not found »
  obtenu en session non privilégiée corrobore, sans le contredire, le
  point ouvert déjà consigné sur l'asymétrie d'import de la clé Terra).
  Dépôt officiel NVIDIA identifié comme nécessaire pour la voie CDI, avec
  un problème de confiance documenté par la communauté Fedora
  (vérifications OpenPGP partiellement ignorées, rapporté sur le forum
  Fedora) — traité avec plus de prudence que Terra, pas la même. Deux
  nouveaux points non résolus dans `docs/gpu-containers.md` : provenance
  exacte du paquet `nvidia-ctk` (COPR Fedora AI/ML non consultable, page
  bloquée par un mur anti-robot lors de cette série) ; effet réel d'un
  conteneur avec périphériques NVIDIA montés sur la veille runtime D3
  (99,97 % établi dans `docs/dgpu-power.md`), non mesurable sans lancer
  un conteneur, hors périmètre de cette série. Périphériques `/dev/nvidia*`
  et `/dev/dri/renderD129` confirmés en mode `666` (aucun obstacle
  rootless pour un usage calcul) ; `/dev/dri/card0` restreint au groupe
  `video` (absent des groupes de l'utilisateur) mais accessible par ACL
  `uaccess` de `systemd-logind`, liée à la session de siège active — sans
  conséquence pour un usage calcul, qui n'en a pas besoin. Recommandation
  argumentée : CDI natif avec le paquet minimal (`nvidia-ctk` seul, pas
  le toolkit complet avec hook redondant) — décision sur la source du
  paquet différée, hors périmètre.
  **Deux consignations ajoutées à cette occasion** : corollaire de D1
  précisé (« poussé après chaque livrable », pas « existe sur un
  distant » — voir § Décisions, D1) ; rappel que la correction du
  dispositif de trace de `roles/gpu_mux/` (état post-écriture non
  capturé, consignée dans la série du 2026-08-05 précédente) **reste
  due**, non appliquée dans celle-ci non plus — périmètre strictement
  documentaire.
  Confirmé en fin de série : aucun paquet installé, aucune image
  téléchargée, aucun conteneur lancé, aucune clé GPG importée, aucun
  dépôt modifié, aucun fichier hors dépôt écrit. Commandes exécutées :
  lectures uniquement (`podman info`, `podman run --help`, `man`/`zcat`
  sur les pages de manuel, `ls`/`getfacl`/`udevadm info` sur les
  périphériques, `grep` sur `/etc/subuid`/`/etc/subgid`, `lsmod`,
  `dnf --disablerepo=terra --assumeno list --available` et
  `dnf --repo=terra --assumeno list --available` sur des motifs ciblés,
  recherche et lecture de documentation externe (wiki KDE, dépôt GitHub
  KWin, documentation NVIDIA, discussion Fedora), marquée comme telle.
