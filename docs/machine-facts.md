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

**Récidive le 2026-08-05, commande différente, même invite.** Reproduire
fidèlement `dnf repoquery --file '*/nvidia-ctk'` (commande citée par
l'opérateur, reproduite sans `--disablerepo=terra` ni `--assumeno` pour
rester fidèle) a de nouveau déclenché `Is this ok [y/N]:` pour la même
clé Terra. Session non interactive (pas d'entrée standard) : l'invite
s'est résolue par fin de flux, pas par une réponse — vérifié après coup
par `rpm -q gpg-pubkey` (aucune entrée datée du 2026-08-05) et recherche
de tout fichier récent sous `~/.cache`/`/etc/pki/rpm-gpg` (rien
d'apparenté à un trousseau importé). **Aucune clé importée**, mais la
leçon de méthode se généralise : ce n'est pas une commande précise
(`repoinfo`) qui déclenche l'invite, c'est **toute commande dnf non
privilégiée qui touche les métadonnées de Terra**, quelle qu'elle soit —
`--assumeno` doit être systématique sur ce dépôt en session non
privilégiée, pas seulement sur les commandes déjà identifiées comme
risquées. Détail complet dans `docs/gpu-containers.md` § 7.5.

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

**[ÉCART CONSTATÉ le 2026-08-06, cause non établie]** Relevé fait dans
le cadre du livrable bureau/terminal (`docs/desktop.md`), en lecture
seule, sans changer aucun mode d'affichage — contredit le point ci-dessus
sur deux aspects distincts :
```
$ kscreen-doctor -o
Output: 1 DP-3 ...   Modes: 3840x1100@60.02*!   Scale: 1.5
Output: 2 eDP-1 ...  Modes: 18:2560x1600@240.00*!  19:...@60.00   Scale: 1.5
```
1. **`eDP-1` est actif à `2560x1600@240.00`** (marqueurs `*` et `!` sur
   le même mode), pas `@60.00` comme relevé le 2026-08-05 — la
   « sélection manuelle » documentée ci-dessus comme restant à faire
   semble avoir été faite entre-temps.
2. **L'échelle a changé sur les deux sorties** : `DP-3` `1.4→1.5`,
   `eDP-1` `1.25→1.5` — les deux sorties partagent désormais la même
   échelle, ce qui n'était pas le cas le 2026-08-05.
Cause non établie dans cette série : aucune commande n'a été exécutée
ici qui expliquerait ce changement, et rien dans l'historique `dnf`/les
journaux consultés pour `docs/desktop.md` ne le documente. Non
investigué davantage — hors périmètre d'un livrable en lecture seule
sur le placement d'un terminal ; aucune action entreprise, aucun mode
changé par cette série. Géométries recalculées, cohérentes avec les
nouvelles échelles (`3840/1.5=2560`, `2560/1.5≈1707`) — pas une mesure
aberrante.

## Conteneurs

- Podman `5.8.4` (API 5.8.4), mode rootless, `cgroupManager: systemd`,
  `cgroupVersion: v2`. (`podman info`)
- Runtime OCI : `crun` 1.28. Réseau : `netavark` 1.17.2 /
  `aardvark-dns` 1.17.1, backend rootless `pasta`. (`podman info`)
- Stockage : pilote `overlay`, filesystem de backing `btrfs`,
  `graphRoot=/home/mahieumi/.local/share/containers/storage`. (`podman info`)
- **[ÉTAIT, jusqu'au 2026-08-05] `/etc/cdi/` : absent.** Depuis la Phase 1
  exécutée le 2026-08-05 (`docs/gpu-containers.md` § 7-8, décisions D7/D8/D9) :
  `/etc/cdi/nvidia.yaml` présent, `root:root`, `0644`, 19954 octets
  (relevé le 2026-08-06). (`ls -la /etc/cdi/`)
  **Écart non expliqué, consigné plutôt qu'investigué a posteriori** : ce
  même fichier faisait 19972 octets lors d'une génération antérieure, le
  même jour (`docs/gpu-containers.md` § 9.5, `sha256sum` distincts avant/
  après le premier essai réel de `regen-cdi-spec`) — sans perte
  fonctionnelle constatée (`diff` avec une génération fraîche à ce moment
  : identique, 21 chemins, les quatre nœuds critiques présents, test
  conteneur réussi). Cause non établie : l'ancienne spécification a été
  écrasée par la régénération elle-même et n'est pas versionnée, rien à
  comparer après coup. `@VERIF : cause exacte de l'écart 19972 → 19954
  octets entre deux générations de /etc/cdi/nvidia.yaml par nvidia-ctk
  cdi generate le 2026-08-06 — candidat le plus vraisemblable,
  l'avertissement "Could not locate libglxserver_nvidia.so.610.43.03"
  observé pendant la génération (bibliothèque GLX serveur, sans objet
  pour le calcul), non confirmé faute des deux sorties de génération
  consignées côte à côte. Confirmable seulement à la prochaine
  régénération réelle, en comparant deux relevés (sha256sum +
  avertissements bruts) déjà consignés — pas en relisant un fichier qui
  n'existe plus.` Règle ajoutée à `CLAUDE.md` § Matériel spécifique —
  GPU/MUX : le `sha256sum` et les avertissements de génération se
  consignent à chaque régénération, pas seulement après coup. Non
  implémenté dans `roles/gpu_cdi/` ici — dette technique, hors périmètre
  de ce livrable.
- **[ÉTAIT, jusqu'au 2026-08-05] `nvidia-container-toolkit` : non
  installé.** Depuis la Phase 1 : `nvidia-container-toolkit-1.19.1-1.fc44.x86_64`
  et `nvidia-container-toolkit-selinux-1.19.1-1.fc44.noarch`, source COPR
  `@ai-ml/nvidia-container-toolkit` (D7). (`rpm -q nvidia-container-toolkit
  nvidia-container-toolkit-selinux`, 2026-08-06)
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
- **`ansible-lint 26.6.0` (D3a) — procédure de reconstruction complète,
  déplacée ici depuis `docs/ansible-chain.md` le 2026-08-06** (dette de
  rangement : ce document décrit la chaîne active de ce dépôt, pas une
  chaîne différée — voir motif détaillé dans le renvoi laissé à sa
  place d'origine).
  **Couplage identifié** : `ansible-lint` dépend d'`ansible-core` ; un
  environnement Python isolé (venv classique, `pipx` par défaut)
  installerait sa **propre** copie d'`ansible-core`, potentiellement
  différente du 2.20.7 système qui exécute réellement les rôles de ce
  dépôt — un lint qui valide contre une autre version que celle
  d'exécution reproduit le décalage qu'on cherche à éviter.
  **Commande exacte** :
  ```
  sudo dnf install -y python3-pip   # prérequis, dépôt fedora déjà activé
  python3 -m venv --system-site-packages ~/.venvs/ansible-lint
  ~/.venvs/ansible-lint/bin/pip install ansible-lint
  ```
  Le drapeau `--system-site-packages` est ce qui garantit une seule
  version d'`ansible-core` en jeu : `pip`, à l'installation, détecte
  l'`ansible-core` **système** (métadonnées
  `/usr/lib/python3.14/site-packages/ansible_core-2.20.7.dist-info`,
  déposées par `dnf`) comme satisfaisant la dépendance déclarée
  d'`ansible-lint` (`ansible-core>=2.16.14`) et **ne le duplique pas**
  — confirmé par contraste avant d'installer quoi que ce soit (essai
  `pip install --dry-run`, sans écriture, dans un venv témoin sans
  `--system-site-packages` : celui-là aurait installé un second
  `ansible-core`, `2.21.2`, distinct du système).
  **Preuve du partage, par la sortie de l'outil lui-même, pas déduite** :
  ```
  $ ansible --version | head -1                       # système
  ansible [core 2.20.7]
  $ ~/.venvs/ansible-lint/bin/ansible-lint --version   # venv
  ansible-lint 26.6.0 using ansible-core:2.20.7 ...
  $ ~/.venvs/ansible-lint/bin/pip show ansible-core
  Location: /usr/lib/python3.14/site-packages   # le système, pas le venv
  ```
  Aucune copie dans `~/.venvs/ansible-lint/lib/python3.14/site-packages/`
  (vérifié). `python3-pip` était absent jusqu'au 2026-08-06 — seul
  paquet système ajouté par cette résolution (`rpm -q python3-pip`,
  avant/après, dépôt `fedora`).
  **Choix du lieu de cette entrée, argumenté** : un document dédié
  (à la manière de `docs/gpu-containers.md` pour `gpu_cdi`) serait plus
  propre à terme si D3a accumule d'autres procédures, mais créer un
  nouveau fichier était hors périmètre du livrable qui a fait ce
  déplacement (2026-08-06, résolution bureau/terminal) — reporté si
  D3a grossit encore.
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

**[CORRIGÉE le 2026-08-06] ÉTAIT** — D3 : chaîne EE-first.
`ansible-navigator` exécute et vérifie dans l'image d'Execution
Environment. L'`ansible-core` système (2.20.7 / Python 3.14) sert à
l'édition et ne fait pas autorité : il est plus récent que la cible AAP
2.6 et accepterait des constructions que la production refuse. Note
datée (2026-08-04) : la cible EE-first est maintenue, mais son moyen
d'installation reste à déterminer — `ansible-navigator` n'est pas
empaqueté dans les dépôts Fedora 44 activés sur ce poste, les options
ouvertes sont `pipx`, un venv dédié, ou l'exécution directe des EE en
`podman run`.

**Motif inversé, corrigé par l'opérateur le 2026-08-06** : la cible de
*ce dépôt* est cette machine elle-même (`workstation-config` configure
`localhost`, Fedora 44) — pas AAP 2.6/RHEL 9, géré ailleurs (préambule
de `CLAUDE.md`). L'`ansible-core` système ne « fait pas autorité » que
pour du contenu *destiné* à AAP ; pour les rôles de ce dépôt, c'est
l'inverse — il fait autorité, puisque c'est lui qui exécute réellement
les rôles sur la cible réelle. Le livrable du 2026-08-06
(`docs/ansible-chain.md`) a été mené sous la prémisse inversée : ses
faits sourcés restent valables (registres, versions, mur
`ansible_compat`), seul le cadrage — à quelle chaîne ils s'appliquent —
était faux. Non supprimé, requalifié ci-dessous et dans
`docs/ansible-chain.md` lui-même.

**D3a (2026-08-06) — chaîne de ce dépôt.** `ansible-core` **système**
(2.20.7), qui fait autorité puisqu'il exécute les rôles sur la cible
réelle (`localhost`). Lint local (`ansible-lint`, installé en venv
`--system-site-packages` pour ne jamais dupliquer cet `ansible-core` —
voir « Chaîne Ansible » et `docs/ansible-chain.md` § Résolution avant
installation). Aucun EE, aucun registre. Motif : `workstation-config`
configure cette machine ; y introduire un EE serait de la sur-couverture
pure — le motif qui fermait déjà `ansible-navigator` au livrable
précédent s'applique a fortiori à l'EE lui-même pour cette chaîne-ci.
Ancien marqueur (« voie d'installation... à trancher au livrable dédié »)
**fermé** : la voie est l'`ansible-core` déjà présent, rien à installer
pour l'exécution des rôles ; `ansible-lint` est le seul ajout, résolu
au livrable du 2026-08-06 (§ Journal des séries).

**D3b (2026-08-06, différée) — chaîne de développement de contenu
destiné à AAP 2.6 / RHEL 9**, si ce poste sert un jour à cela. EE
construit par `ansible-builder` depuis une base publique épinglée par
empreinte, exécuté en `podman run` direct, `ansible-navigator` écarté
(aucune capacité manquante établie pour cet usage non plus). **Non
ouverte** — aucun EE construit, aucune image téléchargée, aucune
authentification. La résolution complète est conservée dans
`docs/ansible-chain.md` et reste valable, une fois requalifiée comme
portant sur D3b et non sur la chaîne de ce dépôt. Le jeton de
vérification borné sur la version cible d'`ansible-core` (2.16 ou 2.18,
posé dans `docs/ansible-chain.md` § 4) reste maintenu, sans objet tant
que D3b n'est pas ouverte — à lire par l'opérateur dans son
environnement professionnel avant toute construction réelle, elle-même
un livrable séparé.

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

**D7 (2026-08-05) — source du NVIDIA Container Toolkit : COPR
`@ai-ml/nvidia-container-toolkit`, restreint par `includepkgs`.**
Décision de l'opérateur. Motif : le dépôt officiel NVIDIA présente un
problème de signature documenté par la communauté (vérifications
OpenPGP partiellement ignorées, erreur `repomd.xml` rapportée sur le
forum Fedora — voir `docs/gpu-containers.md` § 2) ; le COPR est
construit et signé par l'infrastructure Fedora, avec des mainteneurs
identifiables (SIG Fedora AI/ML, canal Matrix
`#ai-ml:fedoraproject.org`). `includepkgs` limite l'exposition aux
seuls paquets du toolkit, pas à l'ensemble du COPR.
Phase 1 de résolution (`docs/gpu-containers.md` § 7) a établi le contenu
exact des paquets (aucune variante `-base`, le hook d'exécution est
incompressible depuis cette source) et l'empreinte complète de la clé
(`0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05`, trousseau temporaire
isolé).

**[MOTIF CORRIGÉ le 2026-08-05, série suivante]** **ÉTAIT** :
implémentation bloquée, faute d'une « corroboration indépendante » de
cette empreinte jugée introuvable, sur le modèle du doute non résolu
concernant la clé Terra — arrêt déclaré dans la série qui a établi ce
fait. **Motif de la correction** : cette exigence était mal posée. Une
clé de projet COPR est générée et détenue par l'infrastructure de build
Fedora elle-même, pour ce projet précis, republiée par cette même
infrastructure au même endroit qu'elle sert le dépôt — comparer
l'empreinte « à la page du projet » est circulaire, structurellement,
pour **tout** projet COPR, pas seulement celui-ci. Ce n'est pas
analogue au doute Terra (où une clé tierce identifiable existe, avec un
canal de publication en principe distinct de la machine servant le
dépôt — l'obstacle y est une asymétrie de visibilité locale, pas une
absence de second canal).

**CORRIGÉ** : l'ancrage de confiance réel est explicité plutôt que
recherché où il ne peut exister — **TLS vers
`copr.fedorainfracloud.org`**, plus la confiance accordée à
l'infrastructure Fedora et au groupe `@ai-ml`. La signature GPG garantit
l'intégrité entre le build et cette machine ; elle n'établit rien de
plus sur l'identité du signataire que ce que TLS a déjà fourni.
Positionnement retenu : un cran en dessous d'un paquet Fedora officiel,
un cran au-dessus du dépôt NVIDIA officiel (vérifications OpenPGP
documentées comme défaillantes). **Ce modèle de confiance est accepté
explicitement par cette décision** — l'implémentation (rôle
`roles/gpu_cdi/`) est préparée dans la série qui porte cette correction.
Son exécution réelle reste **suspendue**, pas techniquement bloquée
comme attendu — voir « Points ouverts » (règle `sudo NOPASSWD: ALL`
trouvée active pour ce compte, contredisant l'état documenté ailleurs
dans ce fichier et dans `docs/gpu-mux-recovery.md`).

**D8 (2026-08-05) — SELinux reste en `Enforcing`.** Décision de
l'opérateur. Aucun `setenforce 0`, aucun mode permissif, même
temporaire ; résolution par lecture des refus AVC réels et réponse
ciblée (`ausearch`/`journalctl -t setroubleshoot`), pas par ajustement
préventif copié d'une documentation externe. **Non encore mise à
l'épreuve** : la Phase 3 qui l'exercerait (lancement réel d'un
conteneur avec périphériques NVIDIA montés) dépend de la Phase 2
(paquet installé), elle-même bloquée par D7 ci-dessus.
`getenforce` reconfirmé `Enforcing` en début de cette série, avant toute
tentative — aucun changement à signaler puisqu'aucune action n'a
été entreprise.

**D9 (2026-08-05) — `sudo` sans mot de passe pour le groupe `wheel`**
(`%wheel ALL=(ALL) NOPASSWD: ALL`, `/etc/sudoers` ligne 110). Décision
de l'opérateur, poste de développement personnel sans donnée
professionnelle. `wheel` ne contient qu'un seul compte
(`getent group wheel` → `wheel:x:10:mahieumi`). Vérifié indépendamment,
pas seulement rapporté : `sudo -n visudo -c` →
« /etc/sudoers: parsed OK » ; `/etc/sudoers.d/` vide (`sudo -n ls -la`) ;
ligne 110 relue directement (`sudo -n sed -n '108,112p' /etc/sudoers`)
et conforme au texte ci-dessus.

**Conséquence assumée** : le garde-fou *dur* (le système refuse
l'élévation) est remplacé par un garde-fou *mou* (l'agent respecte le
périmètre écrit). Le livrable qui a précédé cette décision s'était
arrêté proprement sur cette barrière (`docs/gpu-mux-recovery.md`,
passage désormais marqué `[CADUQUE depuis D9]`) ; elle n'existe plus.

**Second effet** : tout processus s'exécutant sous ce compte obtient
`root` sans invite — dépendance compromise, binaire téléchargé,
extension de navigateur. La demande de mot de passe était le dernier
point où un humain voyait passer une élévation. Conséquence consignée
dans `CLAUDE.md` § Avant d'agir : toute action privilégiée doit
désormais être énumérée explicitement dans le rapport de livrable,
faute de quoi cette visibilité perdue par `sudoers` ne l'est nulle
part.

**Point ouvert de basse priorité** : la règle vit dans `/etc/sudoers`,
fichier appartenant au paquet `sudo` — une mise à jour peut produire un
`.rpmnew` et laisser la modification en place sans le signaler. Un
fichier dédié dans `/etc/sudoers.d/` (actuellement vide) y survivrait
plus proprement. Non résolu ici : ce livrable n'a pas mandat pour
modifier `sudoers` dans un sens comme dans l'autre.

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
  ou désactivé.
  **[REQUALIFIÉ le 2026-08-05]** **ÉTAIT** : « Terra n'est requis par
  aucun composant de la chaîne actuelle » — **faux**, établi par
  inventaire réel (`dnf repoquery --installed --qf '%{name}
  from_repo=%{from_repo}\n' | grep -i 'from_repo=terra'`,
  `docs/gpu-containers.md` § 8.4) : **six paquets réellement installés**
  en dépendent — `asusctl`, `asusctl-rog-gui`, `cardwire`, `supergfxctl`,
  `terra-gpg-keys`, `terra-release`. `asusctl` porte la lecture/écriture
  des attributs `asus-armoury` (`gpu_mux_mode`, `dgpu_disable`) sur
  lesquels reposent D2bis et D2ter — Terra n'est donc pas un tiers
  accessoire mais la source de la pile ASUS de ce poste. Le point non
  résolu ci-dessus sur l'empreinte de la clé Terra cesse d'être
  théorique : priorité relevée en conséquence de cette requalification,
  pas de nouvelle information sur la clé elle-même.
  Aucune action entreprise ici (ni désactivation, ni recherche
  d'équivalents, ni import de clé) — traitement réservé à un livrable
  dédié.
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
  - **[RÉSOLU le 2026-08-06]** `roles/recovery/`, `roles/gpu_mux/` et
    `roles/gpu_cdi/` étaient jusqu'ici des **rôles d'amorçage validés
    manuellement** (`--syntax-check`, `--check`, exécution réelle,
    démonstrations d'échec forcé), pas passés au lint. ÉTAIT : motif de
    l'ordre retenu — le basculement MUX conditionne CDI, les conteneurs
    et la construction des EE, outiller Ansible avant de basculer
    reviendrait à bâtir l'outillage sur une couche qui va elle-même
    changer. Les trois rôles sont maintenant passés au lint (§ Journal
    des séries, 2026-08-06) : `ansible-lint` 26.6.0, contre
    `ansible-core` système 2.20.7 (garantie d'une seule version en jeu —
    voir ci-dessous), profil `production` atteint, `0` défaut. Neuf
    constats corrigés (aucun `noqa` posé par ce livrable) —
    `docs/ansible-chain.md` § Lint des rôles d'amorçage pour le détail.
  - **[RÉSOLU le 2026-08-06, cadrage corrigé]** Voir D3a/D3b (ci-dessus,
    correction du motif inversé de D3). La chaîne de *ce dépôt* (D3a) ne
    nécessite ni `pipx`, ni venv isolé, ni EE — seulement `ansible-lint`
    installé de façon à réutiliser l'`ansible-core` système, jamais un
    second. La chaîne EE-first à fidélité AAP 2.6 (D3b) reste différée,
    non ouverte ; son jeton de vérification (version cible 2.16/2.18)
    n'a plus d'objet tant qu'elle n'est pas ouverte.
- **Nouveau point ouvert (2026-08-05) — règle `sudo NOPASSWD: ALL` active
  pour ce compte, contredisant l'état documenté plus tôt le même jour.**
  `sudo -n true` (exit 0) et `sudo -n -l` (« User mahieumi may run the
  following commands on Zephyrus-MM: (ALL) NOPASSWD: ALL ») confirment
  qu'une élévation de privilège sans mot de passe est actuellement
  possible pour ce compte. **Ceci contredit directement**
  `docs/gpu-mux-recovery.md` (§ Résultats observés, révocation partielle
  de D2ter), qui documente, avec la même commande et la même méthode,
  un échec exact plus tôt le même jour (« sudo: a password is required »,
  `groups=mahieumi,wheel`, pas de règle `NOPASSWD`). Cause de l'écart
  **non établie** : changement délibéré des `sudoers` par l'opérateur
  entre les deux séries (le plus probable, vu la récurrence du blocage
  dans les livrables précédents), ou différence de contexte d'exécution
  non identifiée. Non exploité dans la série qui l'a découvert
  (`docs/gpu-containers.md` § 8.6) — la consigne de cette série
  présumait cette règle absente et demandait explicitement de ne
  tenter aucun contournement ; la trouver présente en vérifiant une
  garde ne vaut pas autorisation implicite de s'en servir. Règle
  générale ajoutée à `CLAUDE.md` § Avant d'agir : une capacité
  découverte qui contredit une contrainte déjà établie se signale, elle
  ne s'exploite pas silencieusement. Action attendue : confirmation de
  l'opérateur que cette règle est un choix assumé (auquel cas ce point
  se ferme et l'exécution réelle de `roles/gpu_cdi/` peut reprendre sans
  `--ask-become-pass`) ou qu'elle doit être corrigée avant toute autre
  écriture privilégiée sur ce poste.

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
- **2026-08-05 — CDI natif par COPR, arrêt en Phase 1 (série suivante).**
  D7 et D8 consignées (§ Décisions) sur décision de l'opérateur. Phase 1
  de résolution exécutée intégralement en lecture seule
  (`docs/gpu-containers.md` § 7) : COPR `@ai-ml/nvidia-container-toolkit`
  identifié, contenu réel des paquets vérifié pour `fedora-44-x86_64`
  (aucune variante `-base` dans ce COPR, contrairement au dépôt officiel
  NVIDIA — hypothèse de la série précédente invalidée par lecture des
  fichiers du paquet, pas supposée), empreinte complète de la clé GPG
  obtenue par import dans un trousseau temporaire isolé
  (`0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05`). **Aucune corroboration
  indépendante de cette empreinte trouvée** malgré plusieurs pistes
  (page du projet bloquée par pare-robot, recherche de l'empreinte
  elle-même, API COPR) — arrêt déclaré conformément à la consigne de
  cette même série, avant toute écriture. Conséquence : **Phases 2 à 5
  non exécutées** — aucun rôle `roles/gpu_cdi/` créé, aucun fichier
  écrit dans `/etc/yum.repos.d/` ni `/etc/cdi/`, aucun paquet installé,
  SELinux non touché (`getenforce` reconfirmé `Enforcing`, inchangé
  faute d'action), aucun conteneur lancé. Incident de méthode : la
  reproduction fidèle d'une commande citée par l'opérateur a de nouveau
  déclenché l'invite d'import de clé Terra (§ Dépôts, note de méthode
  mise à jour) — résolue sans import grâce à la session non
  interactive, pas grâce à une garde explicite ; leçon généralisée à
  toute commande dnf non privilégiée touchant Terra, pas seulement aux
  commandes déjà identifiées. Aucune écriture système dans cette série :
  `docs/gpu-containers.md` et ce fichier sont les seuls modifiés.
- **2026-08-05 — motif D7 corrigé, rôle `gpu_cdi` préparé, exécution
  réelle suspendue (série suivante).** L'exigence de « corroboration
  indépendante » de la clé COPR (série précédente) est corrigée : mal
  posée, aucun projet COPR n'a de second canal de publication —
  l'ancrage de confiance est explicité (TLS + empreinte épinglée) et
  accepté par décision de l'opérateur (D7, motif corrigé). Arbitrage
  ajouté à `CLAUDE.md` entre reproduction fidèle et vérification de
  passivité (une commande à effet de bord déjà documenté se reproduit
  avec sa garde). Rôle `roles/gpu_cdi/` construit (huit groupes de
  tâches : gardes, capture `containers.conf` avant, épinglage de clé
  dans un trousseau temporaire isolé, dépôt COPR avec `gpgcheck=1`
  asserté sur le fichier réellement écrit, installation, capture et
  comparaison `containers.conf` après, génération CDI dans `/etc/cdi`
  avec garde anti-`tmpfs`, vérification que les nœuds UVM sont
  référencés) — `--syntax-check` et `--check` passés, deux
  démonstrations d'échec forcé réussies (chemin de spécification vide ;
  conteneur sans périphérique, avec séparation vérifiée entre absence
  d'outil et absence de périphérique). Paquets Terra recensés (six,
  dont `asusctl` — Terra reste nécessaire, désactivation non proposée).
  État de référence SELinux capturé (`Enforcing`, aucun refus AVC
  récent, refus préexistants sans rapport avec ce travail).
  **Exécution réelle du rôle suspendue** — non par la limite de
  privilège attendue, mais par une découverte contredisant cette limite
  elle-même : voir « Points ouverts » (règle `sudo NOPASSWD: ALL`).
  Aucun paquet installé, aucun fichier écrit dans `/etc/`, aucune
  action SELinux, aucun conteneur autre que les deux tests décrits
  ci-dessus (dont une image tierce téléchargée, exception signalée),
  aucun redémarrage.
- **2026-08-05 — D9 confirmée, exécution réelle du rôle `gpu_cdi`,
  fermeture de la série GPU (série suivante).** L'opérateur confirme
  que `sudo NOPASSWD: ALL` (`%wheel`) est un changement délibéré —
  consigné comme **D9** (§ Décisions), vérifié indépendamment
  (`visudo -c`, contenu de `/etc/sudoers.d/`, ligne 110 relue). Passage
  contradictoire de `docs/gpu-mux-recovery.md` marqué
  `[CADUQUE depuis D9]`, pas effacé. Deux règles ajoutées à `CLAUDE.md` :
  énumération explicite de toute action privilégiée dans le rapport de
  livrable ; corollaire généralisant qu'une contradiction entre un fait
  observé et une consigne/un fait documenté appelle à s'arrêter et
  signaler, quel que soit le sens dans lequel elle joue.
  Rôle `roles/gpu_cdi/` exécuté réellement : un défaut trouvé par
  l'exécution (assertion `gpgcheck` cassée par une correspondance de
  sous-chaîne avec `repo_gpgcheck=0`, légitime) corrigé par un ancrage
  de début de ligne — signalé, pas refactorisé. Deuxième exécution
  montre `changed=3` sur les seules tâches délibérément non
  idempotentes (vérification de clé, rejouée à chaque fois par
  conception) ; l'état persistant (dépôt, paquets, spécification CDI)
  est, lui, inchangé — expliqué plutôt que maquillé en `changed=0`.
  Spécification CDI générée dans `/etc/cdi/nvidia.yaml`, nœuds UVM
  confirmés référencés ; `containers.conf` et runtime OCI actif
  confirmés inchangés après installation (hypothèse § 0.3 vérifiée, pas
  supposée). Dépôt COPR confirmé dans `dnf repo list --enabled` avec son
  `includepkgs`. Test nominal rejoué avec succès : l'image de test ne
  contenant pas `nvidia-smi`, sa réussite prouve l'injection CDI depuis
  l'hôte, pas seulement la visibilité du périphérique. Échec forcé sans
  `--device` rejoué, même séparation des deux causes. Aucun refus AVC
  observé (`getenforce=Enforcing` avant et après) — déclaré
  explicitement, aucun ajustement SELinux appliqué. Correction d'une
  hypothèse antérieure infirmée par l'exécution réelle : le sous-paquet
  `-selinux` charge son module automatiquement à l'installation
  (`semodule -i` en `%post`), contrairement à ce qui avait été supposé —
  sans conséquence sur D8, ce chargement est le fait du paquet retenu en
  Phase 1, pas une action SELinux décidée par cette série. Point
  supplémentaire fermé par mesure directe (`docs/gpu-containers.md`
  § 5/§ 8.7) : un conteneur avec périphériques CDI montés mais sans
  charge de travail active ne réveille pas la dGPU. Terra requalifié
  dans les points ouverts (six paquets réels en dépendent, dont
  `asusctl`) — aucune action entreprise.
  Actions privilégiées de cette série, toutes énumérées dans
  `docs/gpu-containers.md` § 8.7 conformément à la nouvelle règle
  `CLAUDE.md` : quatre lectures de `/etc/sudoers`/`sudoers.d`, cinq
  écritures du rôle (dépôt, paquets, répertoire `/etc/cdi`, génération
  de la spécification, permissions), deux lectures `ausearch`, une
  lecture `semodule`. Confirmé en fin de série : `getenforce=Enforcing`
  inchangé, aucun `setenforce`, aucun `--nogpgcheck`, aucun dépôt autre
  que le COPR retenu, `terra` non désactivé, **aucune modification de
  `/etc/sudoers`**, aucun redémarrage.
- **2026-08-06 — détection de péremption de la spécification CDI et
  régénération conditionnelle (`docs/gpu-containers.md` § 9).** Risque
  laissé ouvert par la série précédente : `/etc/cdi/nvidia.yaml`
  référence les bibliothèques du pilote `610.43.03` par des chemins
  versionnés dans leur nom de fichier ; une mise à jour de pilote via
  `rpmfusion-nonfree-updates` les efface sans jamais toucher au contenu
  de la spécification déjà écrite — conteneur qui démarre quand même,
  GPU invisible, repli CPU, sans erreur.
  Établi avant toute conception (aide intégrée `nvidia-ctk`, `rpm -ql`
  du paquet installé, inventaire des plugins `dnf5` réellement
  présents) : aucun mécanisme fourni par le fournisseur ou par
  l'outillage système ne couvre ni la détection de péremption ni la
  régénération conditionnelle — rien à réutiliser.
  Détection implémentée comme tâches réutilisables du rôle
  (`roles/gpu_cdi/tasks/check_spec.yml`, `verify_spec.yml`, tag
  `verify-cdi-spec`) : chemins de bibliothèques référencés vérifiés un à
  un sur l'hôte, version encodée (ancrée sur `libcuda.so.<version>`,
  jamais un motif générique — collision possible avec des bibliothèques
  versionnées indépendamment du pilote, ex. `libnvidia-egl-wayland.so`)
  comparée à la version réellement chargée, lue dans
  `/proc/driver/nvidia/version`. Entièrement en lecture, ne réveille
  jamais le GPU.
  Régénération (`roles/gpu_cdi/tasks/regen_spec.yml`, tag
  `regen-cdi-spec`) délibérément **non automatique** : trois voies de
  déclenchement automatique évaluées et écartées (scriptlet RPM sur le
  paquet pilote, plugin `dnf5` compilé, règle `udev`/service `systemd`
  sur l'apparition de `/dev/nvidia-uvm`) — la dernière, la plus proche
  de résoudre le problème de précondition (nvidia_uvm chargé) par
  construction, écartée pour un mode de silence résiduel identifié par
  analyse (aucun évènement udev si la machine reste allumée, module déjà
  chargé, après une mise à jour) qui aurait reproduit le défaut que ce
  livrable ferme. Voie retenue : vérification et régénération sur
  demande de l'opérateur, taguées, réutilisant les gardes déjà en place
  (nvidia_uvm chargé, nœuds UVM présents). La régénération elle-même
  **constate d'abord** (lecture seule) si la spécification réelle est
  périmée avant d'écrire quoi que ce soit — défaut trouvé et corrigé
  dans cette même série (une première version régénérait
  inconditionnellement, rompant la propriété `sha256sum` inchangé quand
  aucune régénération n'était due) ; génère ensuite dans un fichier de
  travail non privilégié, le revérifie, et n'installe dans `/etc/cdi`
  que si cette revérification passe — jamais d'écriture à partir d'un
  contenu non vérifié.
  Preuve sans mise à jour de pilote (interdite pour ce livrable) : copie
  de travail de la spécification réelle, jamais le fichier réel, avec
  toutes les occurrences de `610.43.03` substituées par une version
  fictive — passée à la vérification par variable
  (`gpu_cdi_verify_spec_path`). Casse avec le message attendu (version
  attendue/trouvée, 35 chemins manquants nommés) ; la vraie spécification
  reste `sha256sum`-identique avant/après. Fidélité de la simulation
  explicitée avec ses deux écarts assumés (§ 9.4).
  Dette du livrable précédent traitée en premier, comme demandé : la
  démonstration de la garde `gpgcheck` (corrigée le 2026-08-05 sans
  rejouer ses deux sens) rejouée dans un playbook isolé — casse à raison
  sur `gpgcheck=0` ancré, passe sur `repo_gpgcheck=0` seul ; règle
  ajoutée à `CLAUDE.md` (une garde corrigée doit rejouer ses deux
  démonstrations). `changed_when: false` ajouté aux trois tâches de
  vérification de clé non idempotentes par conception (§ 8.7 : arbitrage
  revu, la revérification à chaque exécution est conservée, seul le
  comptage change) — deux exécutions réelles complètes du rôle,
  `changed=0` aux deux, contre `changed=3` avant ce livrable.
  Une inexactitude relevée en passant dans `roles/gpu_cdi/gpu_cdi.yml`
  (commentaire affirmant l'absence de règle `NOPASSWD`, périmé depuis D9
  sans avoir été corrigé) a été corrigée par la même occasion — repérée
  en cours de travail sur ce fichier, pas cherchée activement.
  Test nominal rejoué en fin de série, succès inchangé ; `getenforce`
  `Enforcing` avant/après, aucun refus AVC (seuls les refus
  `power-profiles-`/`/etc/passwd` déjà documentés en § 8.6, sans
  rapport). Actions privilégiées de cette série (trois, énumérées dans
  `docs/gpu-containers.md` § 9.5) : une seule écriture réelle dans
  `/etc/cdi/nvidia.yaml` (régénération, après double vérification), deux
  lectures `ausearch`/`sudo -n true`. Aucune mise à jour de pilote, aucun
  nouveau dépôt, aucune modification de `/etc/sudoers`, aucun
  redémarrage.
- **2026-08-06 — chaîne Ansible EE-first, résolution en lecture seule
  (`docs/ansible-chain.md`).** Deux dettes du livrable précédent
  traitées en premier : compteur du jeton de vérification bruité par les
  règles qui le définissent (règle structurelle ajoutée à `CLAUDE.md`,
  CLAUDE.md exclu de tout comptage de validation) ; écart de taille non
  expliqué entre deux générations de la spécification CDI consigné avec
  un jeton borné (§ Conteneurs), règle ajoutée à
  `CLAUDE.md` imposant de consigner `sha256sum` et avertissements à
  chaque régénération future.
  Question centrale posée par la demande : d'où viendraient les EE,
  sachant que D4 exclut toute authentification à un registre. Établi
  sans s'authentifier nulle part (`skopeo inspect --no-creds`, aucune
  image téléchargée au sens couches — métadonnées de manifeste/config
  uniquement) : `registry.redhat.io` (EE officielles AAP) refuse même la
  lecture de manifeste sans compte Customer Portal — démontré, pas
  supposé. L'image communautaire historique (`quay.io/ansible/creator-ee`)
  est archivée depuis 2024-08-26 ; sa remplaçante
  (`ghcr.io/ansible/community-ansible-dev-tools`) est publique et
  actuelle mais ne pin pas `ansible-core` — la consommer ne rapproche pas
  de la fidélité à AAP 2.6, elle déplace le même écart de version dans
  un conteneur. `ansible-builder` (déjà disponible par `dnf`, sans
  `pip`/`pipx`) accepte n'importe quelle base publique et peut, lui,
  épingler la version exacte voulue — seule voie identifiée qui atteigne
  une fidélité réelle sans authentification.
  `ansible-navigator` réévalué sur l'usage réel de ce dépôt (rôles
  d'amorçage exécutés en CLI directe, pas de TUI) : aucune capacité
  manquante par rapport à `podman run` direct pour `run`/`exec`/`lint` —
  seules les sous-commandes interactives (sans objet ici) apportent
  quelque chose de plus. Recommandation : ne pas l'installer.
  Version d'`ansible-core` d'AAP 2.6 sourcée sans authentification (deux
  pages publiques Red Hat, `access.redhat.com` et `redhat.com/blog`) :
  2.16 par défaut, 2.18 en option. Mur trouvé par requête de métadonnées
  (PyPI, aucune installation) et une discussion amont publique
  (`ansible/ansible-lint#4822`) : `ansible-lint` sous Python 3.14 exige
  `ansible-core >= 2.20.0` (contrôle `ansible_compat`) — les deux
  versions réelles d'AAP 2.6 (2.16.14, 2.18.9) sont toutes deux
  inférieures à ce seuil, ce qui élimine structurellement `pipx` et un
  venv dédié comme voies de fidélité sur ce poste (Python 3.14 système,
  sans alternative).
  Recommandation unique posée, argumentée contre les quatre options
  (`docs/ansible-chain.md` § 4) : EE construit localement par
  `ansible-builder`, à partir d'une base publique épinglée par
  empreinte, exécuté exclusivement par `podman run` direct — jamais
  `ansible-navigator`, jamais `pipx`, jamais de venv. **Non appliquée**,
  conformément à la consigne de ce livrable (lecture seule, décision
  différée à l'opérateur) — `D3` marquée `[RÉSOLU le 2026-08-06, décision
  d'application différée]`, un seul point reste ouvert et marqué (version
  cible 2.16 vs 2.18, à confirmer par l'opérateur depuis son
  environnement professionnel). **[CORRIGÉ le 2026-08-06, série
  suivante]** Cadrage de cette entrée lui-même requalifié : D3 était
  scindée sous l'hypothèse fausse que la fidélité AAP concernait la
  chaîne de ce dépôt — voir la note « Motif inversé » ajoutée à D3 et
  les entrées D3a/D3b, plus haut dans cette section.
  Aucune commande exécutée dans cette série n'a modifié l'état de la
  machine : requêtes réseau anonymes (`skopeo inspect --no-creds`,
  `curl` vers l'API PyPI, lectures de pages publiques), lectures locales
  déjà en place (`man`, `registries.conf`), requêtes de métadonnées
  `dnf` (`info`, `repoquery -l`, sans installation). Confirmé en fin de
  série : `rpm -q` négatif sur les quatre outils recherchés, aucune
  nouvelle image dans `podman images`, aucun fichier `auth.json` à
  aucun des trois emplacements documentés, aucune action privilégiée.
- **2026-08-06 (série suivante) — scission de D3, chaîne de lint locale,
  lint des rôles d'amorçage, dette du compteur du jeton de
  vérification.** Correction
  d'un cadrage inversé signalé par l'opérateur : la série précédente
  traitait la fidélité à AAP 2.6 comme la question posée par ce dépôt —
  faux, la cible de `workstation-config` est cette machine elle-même.
  D3 scindée (marquée `[CORRIGÉE le 2026-08-06]`, historique conservé) :
  **D3a**, chaîne active de ce dépôt, `ansible-core` système fait
  autorité ; **D3b**, chaîne EE-first à fidélité AAP 2.6, différée, non
  ouverte. Le livrable précédent (`docs/ansible-chain.md`) entièrement
  relu et requalifié comme portant sur D3b — aucun fait sourcé retiré,
  seuls les titres et conclusions corrigés.
  Pour D3a : établi avant toute installation, par un essai `pip install
  --dry-run` comparant un venv isolé (installerait un second
  `ansible-core`, 2.21.2, distinct du système) à un venv
  `--system-site-packages` (réutilise l'`ansible-core` système 2.20.7,
  ne le duplique pas) — retenue, seule voie garantissant une seule
  version en jeu. `python3-pip` installé (dépôt `fedora`, seule action
  privilégiée de cette série), `ansible-lint` 26.6.0 installé dans
  `~/.venvs/ansible-lint`. Preuve par la sortie de l'outil lui-même :
  `ansible-lint --version` rapporte `ansible-core:2.20.7`, identique à
  `ansible --version` système.
  `recovery`, `gpu_mux`, `gpu_cdi` passés au lint : 9 constats avant
  correction (`name[casing]` ×2, `schema[meta]` ×3 — `author` manquant,
  `name[template]` ×2, `yaml[line-length]` ×2), tous corrigés, aucun
  `noqa` posé. Un `noqa` préexistant dans `roles/recovery/` s'est révélé
  inopérant (règle `command-instead-of-module` ne couvre pas
  `firewall-cmd`) — constaté, non touché, hors périmètre. Garde touchée
  par une correction (`fail_msg` d'un `assert` dans `gpu_mux`) :
  démonstration rejouée dans les deux sens, message identique. Profil
  `production` atteint, `0` défaut, après correction. `--syntax-check`
  et `--check` des trois rôles identiques avant/après (comparaison par
  `git stash`) — aucun changement de comportement.
  Dette du compteur : cause traitée à la racine (le jeton ne s'écrit
  plus nu en prose nulle part dans `docs/machine-facts.md` ni
  `docs/ansible-chain.md` — règle ajoutée à `CLAUDE.md`), pas seulement
  le symptôme (ne plus figer de chiffre, réponse du livrable précédent,
  jugée insuffisante). Compte brut désormais égal au compte de marqueurs
  actionnables aux deux fichiers : 4 dans `docs/machine-facts.md`, 1
  dans `docs/ansible-chain.md`.
  Aucun EE construit, aucune image téléchargée, aucun nouveau dépôt
  système, aucune authentification à un registre, aucune modification
  de `sudoers` ni de `/etc/cdi/`, `terra` non désactivé, aucun
  redémarrage.
- **2026-08-06 (série suivante) — bureau et terminal, résolution en
  lecture seule (`docs/desktop.md`).** Dette de rangement traitée en
  premier : procédure de reconstruction du venv `ansible-lint` déplacée
  depuis `docs/ansible-chain.md` (requalifié D3b) vers ce fichier
  § Chaîne Ansible (D3a) — un renvoi laissé à sa place d'origine, pas
  une copie.
  Contradiction relevée avant d'aller plus loin (`CLAUDE.md` § Avant
  d'agir) : `eDP-1` lu actif à `@240` et les deux sorties à l'échelle
  `1.5`, contre `@60` et des échelles `1.4`/`1.25` documentées le
  2026-08-05 — consigné en « Affichage » ci-dessus, cause non établie,
  aucune action entreprise.
  Question centrale posée par la demande : comment KWin désigne un
  écran dans une règle de fenêtre. Établi par lecture du schéma source
  de KWin à la version exactement installée (`v6.7.3`,
  `rulesettings.kcfg` : `screen` est un `Int` — « Screen number »,
  aucune entrée `output`/`uuid` sur 81 entrées) et du code
  d'application de la règle (`rules.cpp` :
  `workspace()->outputs().indexOf()`/`.value()`) : **l'index est une
  position dans une liste live, pas un identifiant persistant** —
  aucun moyen d'utiliser l'UUID stable de `kscreen-doctor -o` dans une
  règle KWin, cette possibilité n'existe pas dans le schéma. Ordre
  d'énumération DRM identique sur les trois derniers démarrages
  relevés (journal noyau, y compris de part et d'autre de la bascule
  MUX) — mais correspondance exacte avec la liste interne de KWin non
  prouvée, marquée d'un jeton de vérification dans `docs/desktop.md`
  § 1.3. Aucun écran externe branché sur ce poste actuellement — risque
  de branchement à chaud réel en général, sans objet pour la
  configuration actuelle.
  Mécanismes de démarrage établis par lecture du graphe de dépendances
  systemd réel : autostart XDG (`xdg-desktop-autostart.target`,
  générateur systemd natif, clés `X-KDE-autostart-phase`/`-after`
  déjà en usage sur ce poste), restauration de session Plasma
  (`plasma-restoresession.service`, distinct, tourne après tout le
  reste), unité systemd personnalisée. `kwin_wayland` démarre en tout
  premier, avant `ksmserver` et avant la cible d'autostart — favorable
  au risque nommé par la demande, mais l'ordre de démarrage des
  processus n'est pas prouvé équivalent à la disponibilité de l'état
  interne de KWin (topologie entièrement énumérée).
  Vingt candidats terminaux recensés dans les onze dépôts activés
  (Terra : aucun), provenance et coût mesurés par simulation
  `--assumeno` (0 à 82 paquets selon le candidat, `cool-retro-term`
  seul à tirer une pile Qt5 complète en plus du Qt6 déjà présent).
  Support Wayland natif établi par les dépendances déclarées (boîte à
  outils incluse) : quatre candidats (`xterm`, `rxvt-unicode`, `st`,
  `eterm`) sans aucune dépendance GTK/Qt/EFL, XWayland uniquement ; tous
  les autres Wayland-capables nativement, `foot` étant le seul
  exclusivement Wayland (aucune dépendance X11). Konsole présenté comme
  option à part entière (déjà installé, zéro paquet neuf, seule
  intégration KF6 directe du lot). **Aucun terminal recommandé.**
  `DP-3` : échelle `1.5`, surface logique `2560×734` — calcul de
  colonnes/lignes par approximation explicite (≈230-320 colonnes,
  ≈31-43 lignes selon la taille de police), format impraticable en
  panneau unique, intérêt d'un découpage en volets déjà natif dans la
  plupart des candidats.
  `kwinrulesrc` (vide actuellement) et `~/.config/autostart/*.desktop`
  (absent) identifiés comme versionables sous D1/D4. Exemple concret
  trouvé sur ce poste illustrant l'avertissement de la demande :
  `~/.config/kwinrc` mélange une section `[Tiling]` indexée par UUID de
  bureau et de sortie (état d'instance, non portable) avec un réglage
  générique (`[Xwayland] Scale=1.5`, portable) — copier ce fichier tel
  quel romprait D1.
  Incident de méthode signalé : une requête `dnf list` sur les
  candidats terminaux a été lancée sans garde sur Terra, déclenchant
  l'invite d'import de clé déjà documentée — vérifié immédiatement
  après coup, aucune clé importée (celle présente date du 2026-08-04),
  corrigé pour le reste de la série.
  Aucun paquet installé, aucune configuration Plasma modifiée, aucune
  règle KWin créée, aucun autostart créé, aucun mode d'affichage
  changé, aucune action privilégiée, aucun fichier écrit hors de ce
  dépôt.
