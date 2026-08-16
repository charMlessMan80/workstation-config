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

**[ÉTAIT, jusqu'au 2026-08-05] Dix dépôts activés.** Depuis l'ajout du
COPR `@ai-ml/nvidia-container-toolkit` (D7, 2026-08-05) : **onze**.
Inventaire complet avec ancrage de confiance et statut de vérification
de chacun : [`docs/repositories.md`](repositories.md), qui fait
désormais foi pour cette vue d'ensemble — pas dupliqué ici. Liste des
noms, pour référence rapide seulement : `claude-code`,
`copr:copr.fedorainfracloud.org:group_ai-ml:nvidia-container-toolkit`,
`fedora`, `fedora-cisco-openh264`, `rpmfusion-free`,
`rpmfusion-free-tainted`, `rpmfusion-free-updates`, `rpmfusion-nonfree`,
`rpmfusion-nonfree-updates`, `terra`, `updates`. (`dnf repo list
--enabled`)

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

- **Fait daté (2026-08-04), pilote depuis mis à jour deux fois — voir
  ci-dessous pour l'état courant.** Pilote alors : `NVIDIA-SMI
  610.43.03`, `KMD Version 610.43.03`, `CUDA UMD Version 13.3`, VBIOS
  `95.03.2D.00.06`. (`nvidia-smi -q`)
  **État courant (2026-08-09, GPU-4)** : `NVIDIA-SMI 610.57.04`, `KMD
  Version 610.57.04`, `CUDA Version 13.3`, VBIOS inchangé (mise à jour
  logicielle, pas de flash). (`nvidia-smi -q`) Mise à jour intermédiaire
  à `610.43.03 → 610.57.04` non documentée ici par un relevé distinct —
  seule la valeur au moment de l'installation initiale et la valeur
  courante sont consignées ; tout ce qui suppose `610.43.03` comme
  version **du pilote réellement chargé aujourd'hui**, plus loin dans ce
  document, est à cette date faux — vérifié occurrence par occurrence,
  `docs/packages.md`/journal, entrée GPU-4.
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
    ci-dessous). **Fait daté (2026-08-04) : un seul à cette date.**
    Deux mises à jour de noyau depuis en ont ajouté chacune un
    (`kmod-nvidia-7.1.6-201`, `kmod-nvidia-7.1.7-200` — transactions 31
    et 40, `docs/packages.md` § 4.2) : trois au total à la date de
    GPU-4 (2026-08-09), les anciennes non purgées par ce mécanisme.
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
- **[CORRIGÉ le 2026-08-09, GPU-4] ÉTAIT** : règle udev
  `/usr/lib/udev/rules.d/90-supergfxd-nvidia-pm.rules`, attribuée au
  paquet `supergfxctl` — **faux**, une corrélation lue comme une cause
  (la règle existait, la valeur mesurée correspondait, le lien n'a
  jamais été démontré ; détail complet et preuve expérimentale non
  provoquée — `supergfxctl` retiré depuis le 2026-08-07, redémarrage
  depuis, `power/control` toujours `auto` — dans `docs/dgpu-power.md`
  § Mécanismes en présence, point 1). Règle réelle : udev
  `/usr/lib/udev/rules.d/80-nvidia-pm.rules`, déposée par
  `xorg-x11-drv-nvidia` (le pilote lui-même, pas `asusd` ni
  `supergfxctl`) : bascule `power/control` à `auto` sur bind des
  fonctions PCI NVIDIA (classes `0x030000` et `0x030200`), et à `on`
  sur unbind. (`rpm -qf /usr/lib/udev/rules.d/80-nvidia-pm.rules`,
  `cat /usr/lib/udev/rules.d/80-nvidia-pm.rules`)
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
  **Donnée ajoutée le 2026-08-09 (GPU-4), marqueur non fermé — elle
  n'explique pas l'écart historique, elle en éloigne l'explication la
  plus simple.** Première régénération réelle depuis cet écart
  (nécessitée par la mise à jour du pilote vers `610.57.04`,
  `docs/gpu-containers.md` § Péremption) : `sha256sum` et taille relevés
  avant (`303b2bcc…`, 19954 octets) et après (`b1e555d8…`, 19954 octets
  — **taille inchangée**, seul le contenu diffère). L'avertissement
  `Could not locate libglxserver_nvidia.so.610.57.04` (candidat cité
  ci-dessus pour l'écart 19972 → 19954) **est réapparu**, deux fois,
  dans cette génération — mais sans corrélation avec un écart de taille
  cette fois : avant et après portent le même nombre d'octets. Ce n'est
  pas la preuve que l'hypothèse est fausse (le contexte diffère : ici
  une bascule de version de pilote, alors le 2026-08-06 deux
  régénérations le même jour, même version) — c'est la preuve qu'elle
  ne se vérifie pas trivialement par simple présence de
  l'avertissement. Les deux sorties de génération complètes de cette
  série sont consignées dans `docs/gpu-containers.md` § Péremption pour
  qu'une future régénération sur un pilote inchangé puisse enfin
  reproduire les conditions exactes de l'écart original. **Marqueur
  maintenu ouvert** : ces avertissements n'expliquent toujours pas
  l'écart de 2026-08-06, faute des deux sorties de cette date
  précise — ils écartent seulement l'explication la plus simple
  (« l'avertissement suffit à lui seul à faire varier la taille »).
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
`supergfxctl`. **Fait daté, exact pour le 2026-08-04** : à cette date,
`supergfxctl` était installé, son démon `supergfxd` simplement
désactivé — décrire cet état comme « installé mais désactivé » était
vrai alors. **[CORRIGÉ le 2026-08-09, GPU-4] Périmé depuis, à ne pas
lire comme l'état courant** : `supergfxctl` n'est plus installé du tout
depuis le 2026-08-07, retiré silencieusement par obsolescence Terra
(`terra-obsolete`), pas par un choix délibéré de l'opérateur ni par ce
rôle — détail complet `docs/repositories.md` § 9, `docs/packages.md`
§ 2.3. Le motif de D2ter (écarter `supergfxd` par prudence, quel qu'en
soit l'état) reste valide indépendamment de cette disparition — la
décision elle-même n'est pas remise en cause, seule la description de
l'état du paquet l'était.

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

**[DÉPLACÉE le 2026-08-08, ORC-3] D9 (2026-08-05) — `sudo` sans mot de
passe pour le groupe `wheel`.** **ÉTAIT** : règle écrite en dur dans
`/etc/sudoers` ligne 110 (`%wheel ALL=(ALL) NOPASSWD: ALL`). Motif
conservé — décision de l'opérateur, poste de développement personnel
sans donnée professionnelle ; `wheel` ne contient qu'un seul compte
(`getent group wheel` → `wheel:x:10:mahieumi`). **Déplacée** vers
`/etc/sudoers.d/99-wheel-nopasswd` (`roles/bootstrap/`, tag
`bootstrap-sudoers`, exécuté réellement le 2026-08-08, ORC-3) — motif
du déplacement : `/etc/sudoers` appartient au paquet `sudo`, une mise
à jour peut produire un `.rpmnew` qui écarte la règle en silence,
motif déjà nommé dès l'origine de D9 (point ouvert ci-dessous, fermé
par ce même livrable). État vérifié indépendamment à l'origine, pas
seulement rapporté : `sudo -n visudo -c` → « /etc/sudoers: parsed OK » ;
`/etc/sudoers.d/` vide (`sudo -n ls -la`) ; ligne 110 relue directement
(`sudo -n sed -n '108,112p' /etc/sudoers`) et conforme au texte
ci-dessus — **cet état de départ est celui d'avant le déplacement**,
voir le journal ORC-3 pour l'état final.

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

**[FERMÉ le 2026-08-08, ORC-3] Point ouvert de basse priorité** (ÉTAIT,
depuis le 2026-08-05) : la règle vivait dans `/etc/sudoers`, fichier
appartenant au paquet `sudo` — une mise à jour pouvait produire un
`.rpmnew` et laisser la modification en place sans le signaler. Fermé
par déplacement effectif vers `/etc/sudoers.d/99-wheel-nopasswd`
(ci-dessus), fichier dédié, appartenant à aucun paquet — pas par
requalification, par le déplacement lui-même, exécuté et vérifié dans
les deux sens (journal ORC-3, plus bas).

**Procédure de retour — écrite le 2026-08-08, avant toute écriture de
D9-MIG (déplacement vers `/etc/sudoers.d/`), pas après.** Si un
contrôle échoue à n'importe quelle étape de D9-MIG, depuis un shell
root déjà ouvert (`sudo -i` dans un terminal séparé, ou la session SSH
active — ne fermer aucun des deux avant confirmation finale) :

1. **Restaurer la ligne dans `/etc/sudoers`** — si elle a été retirée
   (étape 4 de D9-MIG) et que quelque chose ne va pas :
   ```
   sed -n '108,112p' /etc/sudoers   # vérifier l'état actuel d'abord
   visudo    # ré-ouvrir, ré-insérer à la main la ligne exacte :
   #   %wheel	ALL=(ALL)	NOPASSWD: ALL
   # entre "## Same thing without a password" et la ligne
   # "## Allows members of the users group..." (position d'origine,
   # ligne 110 avant tout retrait) — visudo refuse d'enregistrer si la
   # syntaxe est mauvaise, c'est la garantie qu'il ne peut pas aggraver
   # l'état.
   ```
   Alternative non interactive, depuis le shell root (pas depuis un
   compte qui dépendrait de la règle cassée) :
   ```
   printf '%%wheel\tALL=(ALL)\tNOPASSWD: ALL\n' >> /etc/sudoers.tmp
   # puis fusionner à la position d'origine — visudo reste la voie
   # sûre, cette alternative n'est qu'un filet si visudo lui-même est
   # indisponible.
   ```
2. **Si le nouveau fichier `/etc/sudoers.d/99-wheel-nopasswd` pose
   problème** (mauvaise syntaxe, mauvaise permission, ignoré) — le
   supprimer purement et simplement restaure l'état antérieur tant que
   la ligne dans `/etc/sudoers` est encore en place (étapes 1-3 de
   D9-MIG) :
   ```
   rm -f /etc/sudoers.d/99-wheel-nopasswd
   ```
3. **Valider après toute restauration** :
   ```
   visudo -c
   ```
   Doit rapporter `/etc/sudoers: parsed OK` (et l'absence d'erreur sur
   tout fichier de `/etc/sudoers.d/`).
4. **Vérifier que l'accès est réellement revenu**, depuis le compte
   normal (`mahieumi`), pas depuis le shell root qui n'a jamais perdu
   l'accès :
   ```
   sudo -n -l -l
   ```
   Doit montrer une entrée `Sudoers entry:` avec `Options:
   !authenticate` (équivalent affiché de `NOPASSWD`) — le nom du
   fichier après `Sudoers entry:` confirme la source exacte de la
   règle effective (mécanisme établi avant d'écrire quoi que ce soit,
   § D9-MIG ci-dessous).

**D10 (2026-08-06) — dépôt `terra` (Fyra Labs) : porteur, désactivation
exclue ; ancrage de confiance formulé explicitement.**

`terra` fournit six paquets réellement installés sur ce poste —
`asusctl`, `asusctl-rog-gui`, `cardwire`, `supergfxctl`,
`terra-gpg-keys`, `terra-release` — dont `asusctl`, sur lequel reposent
D2bis et D2ter. Désactivation exclue, aucun paquet remplacé.

**Ancrage de confiance réel, formulé plutôt que sous-entendu, comme pour
D7** : TLS vers les serveurs de Fyra Labs au moment de l'ajout du dépôt
(transaction 5, `dnf install --nogpgcheck --repofrompath`, conforme à
la procédure d'installation publiée par l'éditeur — vérifié par lecture
du `README.md` du dépôt source, § `docs/repositories.md` § 1), plus
l'acceptation de la clé dans la base rpm le 2026-08-04 (transaction 6,
`dnf install asusctl`, interactive). Empreinte faisant foi :
`AE09157A4DE88B497EA1D5D300CDAB43DE226D6F`.

**Corroboration indépendante : recherchée activement sur neuf canaux
distincts du serveur de dépôt, introuvable.** Détail complet, canal par
canal, y compris les échecs, dans `docs/repositories.md` § 1 — pas
dupliqué ici.

**Différence avec D7 à noter** : pour une clé de projet COPR, la
corroboration n'existe structurellement pas (générée et republiée par
la même infrastructure). Pour Terra, elle **pourrait** exister — Fyra
Labs est une organisation tierce identifiable, avec un site, un
`security.txt`, une organisation GitHub — mais l'examen actif de ces
canaux ne montre pas que cette empreinte y soit publiée. L'écart est
donc d'une autre nature : une lacune de publication chez l'éditeur, pas
une propriété du modèle.

**Fait à consigner pour éviter une fausse alerte future** :
`fyralabs.com/pgp.asc` publie une clé d'empreinte
`00AEBB9B28D1D0E97D0713B390C30D0FE8C19CC3` — **différente**, c'est la
clé de contact sécurité de l'organisation, pas la clé de signature du
dépôt. Deux usages distincts d'une même adresse `security@fyralabs.com`.

**Décision de l'opérateur** : le risque est déjà pris — cette clé
gouverne l'installation des six paquets depuis le 2026-08-04, sans
incident depuis. Rendre le chemin non privilégié (jamais résolu)
cohérent avec le chemin privilégié (résolu depuis le 2026-08-04)
n'étend aucune confiance nouvelle ; refuser ce correctif n'aurait retiré
aucune exposition et aurait conservé une friction sans contrepartie,
l'invite n'étant plus, en pratique, qu'une étape systématiquement
contournée par `--assumeno`. Correctif appliqué : copie du fichier de
clé déjà accepté côté système vers le trousseau (« pubring »)
utilisateur de `libdnf5` — mécanisme et preuve complets dans
`docs/repositories.md` § 2-3. Aucun import nouveau, aucune dégradation
de `gpgcheck` ni de `repo_gpgcheck`.

**Point ouvert, basse priorité** : une demande à `security@fyralabs.com`
pourrait obtenir la publication de l'empreinte de la clé de signature du
dépôt par un canal indépendant. Action humaine, non entreprise ici.

**D11 (2026-08-06) — placement de fenêtre KWin : `position`+`size` en
`Apply initially`, pas `screen`+`Force`.**

Rôle `roles/desktop/` (kitty, choix de l'opérateur en BUR-0 —
`docs/desktop.md` § 3), déployé et vérifié par la mesure. La règle
KWin qui place la fenêtre kitty sur le ScreenPad Plus (`DP-3`) n'utilise
**pas** l'index d'écran (`screen`/`screenrule`) envisagé initialement :
testé et mesuré confondu avec `org.kde.KWin.activeOutputName` (la
sortie « active » l'emporte sur l'index configuré, y compris en
`Force`) — détail complet, `docs/desktop.md` § 6.2, pas dupliqué ici.
Mécanisme retenu : `position` (Point) + `size` (Size), réglées sur la
géométrie logique de `DP-3` **mesurée en direct à chaque exécution du
rôle** (`kscreen-doctor -o`), jamais une valeur figée en dur — vérifié
non confondu avec le même effet (`activeOutputName` tenu constant à
`DP-3` pendant qu'un retour explicite à la géométrie d'`eDP-1` a bien
déplacé la fenêtre, § 6.8).

Politique de règle choisie : `Apply initially` (`...rule=3`), pas
`Force` (`...rule=2`) — les deux mesurées également fiables, la
distinction vient de leur effet documenté par l'interface KWin
elle-même (source externe, `src/kcms/rules/optionsmodel.cpp`, tag
`v6.7.3`) : `Force` s'applique en permanence à **toute** fenêtre kitty
(le critère de correspondance, `wmclass=kitty`, est générique, pas
limité à la fenêtre de démarrage), empêchant l'opérateur de la
redéplacer, jamais, pour aucun usage ultérieur ; `Apply initially`
s'applique une fois à la création puis libère la fenêtre. Retenu :
`Apply initially`, pour ne pas contraindre en permanence un usage
interactif au nom d'un besoin qui ne concerne que l'ouverture de
session.

**Fait propre à cette machine, exposé en variable** (D1) :
`desktop_target_output: DP-3` (`roles/desktop/defaults/main.yml`) — le
nom du connecteur cible, pas un index. La tâche de mesure échoue
bruyamment (`failed_when`) si cette sortie n'existe pas sur la machine
en cours d'exécution, plutôt que de placer silencieusement la règle
sur une géométrie jamais vérifiée — vérifié dans les deux sens
(`docs/desktop.md` § 6.9).

**Confondant supplémentaire trouvé et écarté pendant la vérification** :
le placement par défaut de KWin (aucune règle ne s'applique) coïncide
actuellement avec la géométrie de `DP-3`, parce que
`activeOutputName` vaut `DP-3` au moment des essais — démontré avec un
terminal étranger à toute règle de ce dépôt (`konsole`). Ceci a rendu
non probant un premier test de neutralisation par critère de
correspondance désaccordé (`wmclass` invalide) — remplacé par le test
décrit ci-dessus, qui isole la sortie active comme variable de
contrôle. Mécanisme exact du placement par défaut de KWin : non lu
dans le code source, seulement observé — `docs/desktop.md` § 6.3
porte le marqueur `@VERIF` correspondant, non bloquant.

**Autostart** : `~/.config/autostart/kitty-screenpad.desktop` déployé
par le rôle, sans chemin absolu propre à cette machine. Effet réel non
vérifiable sans déconnecter la session (interdit sans demande
explicite) — commande de vérification préparée pour la prochaine
ouverture de session, `docs/desktop.md` § 6.7.

**D12 (2026-08-06) — npm n'est pas ouvert comme surface
d'approvisionnement.** Note de numérotation, signalée plutôt que
résolue en silence (`CLAUDE.md` § une découverte qui contredit un fait
déjà documenté se signale) : la demande de ce livrable désignait cette
décision « D11 », déjà pris par la décision de placement de fenêtre
KWin ci-dessus (même jour, commit antérieur) — renumérotée D12 sans
toucher au D11 existant.

Ni `node`, ni `npm`, ni aucun serveur de langage récupéré par ce
canal. Motif : `yaml-language-server` et `ansible-language-server` ne
sont empaquetés dans aucun des onze dépôts (constaté en EDI-0,
`docs/editor.md` § 1.2) ; les installer supposerait une surface
d'approvisionnement échappant entièrement à `docs/repositories.md`,
sans les ancrages de confiance établis en D7 et D10, avec un modèle de
dépendances transitives sans commune mesure avec rpm. **Contrepartie
assumée** : pas de complétion ni de diagnostic YAML/Ansible en place
dans l'éditeur — remplacée par `ansible-lint`/`yamllint` (venv D3a)
appelés en ligne de commande ou via outil externe Kate, jamais un
second binaire installé. Garde automatisée dans `roles/editor/`,
démontrée dans les deux sens (`docs/editor.md` § 2.3).

**D13 (2026-08-06) — deux éditeurs selon la tâche : Helix en terminal,
Kate en graphique.** Numérotation renumérotée pour la même raison que
D12 (la demande désignait ce choix « D12 »). Helix embarque coloration
tree-sitter et configuration complète sans aucun gestionnaire de
greffons — contrairement à Neovim, dont l'intérêt réel suppose des
greffons récupérés hors dépôts, même objection que D12 à plus petite
échelle. Kate est natif KF6, déjà intégré à la session Plasma, avec
terminal embarqué et mécanisme d'outils externes. Zed et Cursor
écartés : magasin d'extensions propre (nouvelle surface
d'approvisionnement), télémétrie, dépendance à `terra` pour les deux —
détail complet, `docs/editor.md` § 1.2/1.3/2.

Rôle `roles/editor/` : installe les deux depuis `fedora`/`updates`
uniquement (`kate` n'était pas installé, `helix` non plus — vérifié
avant installation). Kate configuré exclusivement par clé nommée
(`kwriteconfig6`, jamais une copie de fichier — même discipline qu'à
BUR-1 sur `kwinrulesrc`) : deux greffons activés (outils externes,
terminal intégré — jamais le client LSP, rien à lui connecter sous
D12), deux outils externes déployés pointant explicitement les
binaires `ansible-lint`/`yamllint` du venv D3a. Helix configuré avec
une seule option motivée (numérotation de ligne absolue), aucun
`languages.toml` — l'absence de serveur de langage pour `yaml` est
déjà documentée par le fichier fourni avec le paquet
(`hx --health yaml` : *not found in $PATH* pour les deux serveurs,
sans erreur bloquante), conforme à D12.

Mode d'affichage de Kate établi par la mesure, pas par déduction des
dépendances : greffon de plateforme Qt réellement chargé
(`libqwayland.so`), `libqxcb.so` absent de l'espace d'adressage du
processus — **Wayland natif**, confirmé (`docs/editor.md` § 2.4).

`ansible-lint --version` vérifié inchangé après exécution du rôle
(`ansible-core:2.20.7`) — preuve qu'aucun second binaire n'a été
introduit. Défaut réel trouvé et corrigé en cours de route : la sortie
colorée d'`ansible-lint --version` insère des codes ANSI *au milieu*
de la chaîne à comparer, faisant échouer une comparaison naïve —
corrigé par `NO_COLOR=1`.

**D14 (2026-08-07) — service d'inférence local : Ollama conteneurisé,
image CUDA officielle, accès GPU par CDI natif.** Numéros vérifiés
libres avant attribution (`grep -n '^\*\*D[0-9]' docs/machine-facts.md`,
D13 le plus élevé au moment de la vérification — pas de collision
cette fois, contrairement à D11/D12 en EDI-1). Motif : seule voie
atteignant la RTX 4090 (`docs/local-ai.md` § 3.1, IA-0 — `ollama`,
`llama-cpp`, `whisper-cpp`, `python3-torch` empaquetés par Fedora tous
liés en dur à ROCm/HIP, aucun lien CUDA). Ollama gère manifestes et
empreintes de modèles, contrairement à un GGUF récupéré à la main sans
mécanisme d'intégrité uniforme — troisième surface d'approvisionnement
de ce dépôt (après les extensions d'éditeur, D12, et les registres de
conteneurs, IA-0), la seule des trois à offrir un ancrage vérifiable
intégré. Déployé par `roles/local_ai/` : Quadlet Podman (mécanisme
natif de Podman 5.8.4, pas une unité systemd écrite à la main), image
épinglée par empreinte (`sha256:b88c73ace3e115f8ec53dc8761ae1c0aabfa675406e3681786b98757ce050f42`,
pas une étiquette mobile), accès GPU par CDI déjà prouvé
(`docs/gpu-containers.md`), trois gardes préalables réutilisant
`verify-cdi-spec` sans le réimplémenter, service revérifiant lui-même
la spécification CDI à chaque démarrage (`ExecStartPre=`). Réussi dès
la première exécution réelle — détail complet et découverte non
anticipée (Ollama contacte `ollama.com` de son propre chef, `OLLAMA_NO_CLOUD:false`
par défaut, non corrigé ici, nommé pour un livrable ultérieur) :
`docs/local-ai.md` § 7.

**[REQUALIFIÉE le 2026-08-07, D20] D15 (2026-08-07) — modèle de
complétion résident en permanence (`keep_alive` infini).** Motif :
latence minimale, usage principal (complétion de code). Contrepartie
assumée par l'opérateur : un contexte CUDA ouvert en permanence
empêche RTD3 de s'engager — les ~8 W récupérés par la bascule MUX
(D2ter) sont repayés tant que le service tourne **avec un modèle
chargé**. Mesuré dans ce livrable, service démarré **sans** modèle :
RTD3 s'engage toujours normalement dans ce cas précis
(`docs/local-ai.md` § 7.6 — 0 ms de temps actif sur 280 s observées) —
la contrepartie ne commence qu'au premier chargement effectif d'un
modèle, pas au démarrage du service. **ÉTAIT** la décision de
résidence permanente elle-même — **requalifiée par D20** (ci-dessous,
CMP-1) en modèle **à la demande** : engager la contrepartie ci-dessus
avant d'avoir établi que la complétion sert réellement au quotidien
revenait à payer un coût certain pour un bénéfice supposé. Le motif
originel de D15 (latence par frappe) reste correct **si** un
mécanisme de complétion automatique existe dans l'éditeur — établi
depuis, `docs/completion.md` § 2.1 (`lsp-ai`) — mais la décision de
résidence elle-même est révisée indépendamment, par prudence
budgétaire, pas parce que le motif serait devenu faux.

**[RÉALIGNÉE le 2026-08-12, BSH-2] D15 revient à la résidence
permanente — pas un retour en arrière, la requalification du 7 août
ci-dessus était juste à sa date.** Écart découvert en vérifiant l'état
du GPU après BSH-1 : `roles/local_ai/defaults/main.yml` portait
toujours `local_ai_ollama_keep_alive: "-1"` (résidence, jamais changé
par D20, qui n'a vécu que dans la documentation) — la seule occurrence
à ce jour où une décision requalifiée divergeait d'une **valeur
effective**, pas seulement d'un commentaire. Coût réel mesuré avant de
trancher (méthode d'isolement de `docs/dgpu-power.md`, détail complet
`docs/local-ai.md` § 11.1) : une fois le modèle réellement résident,
RTD3 est bloqué à **100 %** sur deux fenêtres isolées de 300 s (`Δ
runtime_suspended_time = 0 ms` les deux fois) — pas les ~25 % apparus
dans un relevé mêlant à tort la phase de démarrage sans modèle (RTD3
encore libre, § 7.6) et la résidence effective. Deux faits nouveaux,
absents le 7 août, justifient le réalignement : la complétion
fonctionne réellement, dans deux éditeurs sur trois langages (BSH-1,
`docs/completion.md` § 9/10) ; une complétion à froid coûte **22,6 s
mesurées** (réveil RTD3 + chargement complet, `docs/completion.md`
§ 10.3) — pas les ~3,7 s d'une bascule à chaud entre deux modèles déjà
chargés (IA-4, § 9.4 ci-dessous), qui ne couvrait jamais le cas
résident→arrêté→résident. À la demande, chaque reprise après une pause
de frappe paierait ce coût, rendant la complétion au fil de la frappe
inutilisable — **décision de l'opérateur** : le modèle reste résident.
`local_ai_ollama_keep_alive` n'a jamais changé de valeur (déjà `-1`) ;
`ansible-playbook roles/local_ai/local_ai.yml` confirme `changed=0` dès
la première exécution de ce livrable — le service n'a pas été
redémarré. Recherche systématique menée sur tout le dépôt, pas
seulement sur `roles/local_ai/` (renforcement de la règle
correspondante, `CLAUDE.md`) : cinq commentaires de `roles/local_ai/`
déjà cohérents avec l'état réalisé (aucune correction nécessaire), un
mis à jour (`defaults/main.yml`, préambule de décisions) ; commentaires
de `roles/completion/` référençant D20 signalés, non corrigés (hors
périmètre de ce livrable, fait décrit par ce rôle inchangé dans les
deux cas) ; `docs/status.md` § options écartées comportait la même
mention périmée — **hors périmètre strict de ce livrable** (non
modifié), signalé pour le prochain qui touchera ce fichier. Aucune
autre décision requalifiée du dépôt n'a été trouvée avec une référence
non alignée (D3/D3a-D3b déjà traitée par la revue du 2026-08-08).
Détail complet, mesures et branches énumérées : `docs/local-ai.md`
§ 11.

**D16 (2026-08-07) — `NVreg_PreserveVideoMemoryAllocations=1`.**
Motif : un modèle chargé survit à une suspension système. Contrepartie
assumée : plusieurs gigaoctets recopiés vers le disque à chaque
suspension, fermeture et réveil plus lents. Sans effet sur RTD3
(neutralisée par D15 tant que le service tourne avec un modèle) — ne
joue que pour la suspension système (s2idle sur ce poste). Déployé par
écrasement propre (`/etc/modprobe.d/local-ai-nvidia-power-management.conf`),
jamais le fichier fourni par le paquet `xorg-x11-drv-nvidia-power`
(intact, somme de contrôle identique avant/après, vérifiée par le rôle
lui-même à chaque exécution). **Correction apportée à IA-0 dans ce
même livrable** : la valeur brute par défaut réellement chargée est
`2`, pas `0` comme IA-0 le supposait implicitement sans l'avoir lue
directement — signification de `2` non documentée dans les sources
consultées, `docs/local-ai.md` § 2.1bis. **Prend effet au prochain
chargement du module `nvidia.ko`** — pas à chaud (module en usage
actif, compteur `148`) ; en pratique, un redémarrage. **Non déclenché
dans ce livrable**, conformément à la consigne — commande de
vérification post-redémarrage préparée (`docs/local-ai.md` § 7.5).

**D17 (2026-08-07) — dimensionnement initial : modèle agent/chat de
13 B en `Q4_K_M`, à mesurer avant toute montée.** Motif :
`Q4_K_M` est le point d'étalonnage documenté par le projet amont
(`llama.cpp`, déjà cité en IA-0) ; aucune perte de qualité n'est
chiffrée pour les quantifications plus agressives, donc pas de
descente sans mesure. **Aucun modèle choisi ni téléchargé dans ce
livrable ni dans IA-1** — réservé à IA-2.

**D18 (2026-08-07, IA-2) — confinement réseau du service d'inférence :
réseau Podman dédié `--internal` (garde structurelle) +
`OLLAMA_NO_CLOUD=1` (défense en profondeur), en réponse à la
découverte D14 (appels non sollicités vers `ollama.com`, fonctionnalité
cloud activée par défaut).** Motif : `--internal` est vérifiable depuis
l'hôte sans dépendre du bon comportement du logiciel dans le conteneur
(`podman network inspect`) et couvre tout vecteur de sortie non encore
identifié par nom ; `OLLAMA_NO_CLOUD=1` empêche la tentative elle-même
pour le cas déjà documenté (moins de bruit de journal), vérifié
efficace par mesure avant d'être retenu (`docs/local-ai.md` § 8.2).
`--network=none` (aucune pile réseau) a été testé et écarté : il casse
aussi le transfert de port publié — l'API locale, pas seulement
`ollama pull`, cesserait de répondre à l'hôte. Contrepartie assumée,
nommée, non résolue par ce livrable : `ollama pull` échoue aussi sur ce
réseau — un chemin distinct (`podman network connect` temporaire)
reste à établir pour un livrable qui télécharge réellement un modèle
(`docs/local-ai.md` § 8.5). Preuve avant/après et démonstration inverse
(la mesure détecte bien une tentative bloquée, provoquée
délibérément sans jamais quitter la machine) : `docs/local-ai.md` § 8.3.

**[REQUALIFIÉ le 2026-08-07, résolution complétion] D15 revue, pas
invalidée à la légère (IA-2)** : la lecture de la faisabilité de la
complétion dans Helix/Kate (`docs/local-ai.md` § 8.6) montrait
qu'aucun mécanisme d'alors ne permettait une complétion automatique au
fil de la frappe dans ces deux éditeurs — ni greffon (Helix n'en a
pas), ni serveur de langage générique compatible (`llm-ls` existe,
licence Apache-2.0, hors npm, mais sa complétion passe par une méthode
JSON-RPC propriétaire qu'un client LSP générique n'appelle jamais).
**ÉTAIT** : conclusion tirée d'un seul serveur vérifié (`llm-ls`),
généralisée à tort à « aucun mécanisme ». **Incomplet, pas faux sur ce
qu'elle avait vérifié** — corrigé par une recherche élargie
(`docs/completion.md`) : `lsp-ai` (MIT, Rust, binaire précompilé hors
npm) implémente la complétion via `textDocument/completion` **standard**
— vérifié jusqu'au bout de la chaîne dans son code source (capacité
déclarée → dispatch de requête → construction de la réponse), documenté
compatible Helix par le projet amont. Le motif exact de D15 (latence
par frappe) **redevient défendable**, sous réserve de deux points
laissés ouverts par `docs/completion.md` § 2.1 : compatibilité Kate non
testée par personne, et absence d'ancrage de confiance pour ce binaire
externe (aucune somme de contrôle publiée, dernière publication ~23
mois avant cette date, développement en pause déclaré par le
mainteneur) — un coût nouveau, distinct de celui de D12 (npm), non
pesé avant `docs/completion.md`. **Signalé, pas tranché ici** :
questions qui départagent dans `docs/completion.md`. **[TRANCHÉ le
2026-08-07, D19/D20 ci-dessous]** — l'opérateur a tranché sur la base
de ces questions : jamais le binaire précompilé (l'absence de somme de
contrôle en faisait un ancrage **plus faible que npm**, l'inverse de
ce que cette voie était censée éviter), compilation depuis les sources
retenue à la place ; résidence permanente requalifiée en complétion à
la demande (D20), indépendamment de la question Kate qui reste
ouverte.

**D19 (2026-08-07, CMP-1) — complétion locale par `lsp-ai`, compilé
depuis les sources (jamais le binaire précompilé).** Motif : le
binaire précompilé n'a aucune somme de contrôle publiée — ancrage de
confiance plus faible que npm, qui transporte au moins des empreintes
d'intégrité dans ses fichiers de verrouillage ; l'adopter aurait
accepté pire que ce que D12 (npm fermé) refusait, en croyant faire
l'inverse. La compilation ouvre **crates.io**, quatrième surface
d'approvisionnement de ce dépôt (`docs/repositories.md` § 7) — avec
vérification d'intégrité par empreinte (`Cargo.lock`, `--locked`) et
code source auditable. Risques assumés, consignés tels quels :
dernière publication (`v0.7.1`) vieille d'environ 23 mois, dernier
commit (celui réellement compilé) environ 19 mois, développement
déclaré en pause par le mainteneur, compatibilité Kate jamais testée
par personne (Helix seul dans ce livrable, Kate reste un point
ouvert). **[RÉSOLU le 2026-08-08, KAT-1]** — testée depuis, résultat
positif : complétions FIM réelles sur YAML et Python, même binaire et
même configuration que Helix (`docs/completion.md` § 9). Épinglé par
empreinte de commit
(`1e910a8cf0048406eb227bf2064743010a9ff3a9`), jamais une étiquette.
Exécuté par `roles/completion/` — deux obstacles réels rencontrés en
compilant (dépendance git `hf-hub` sans rev fixe dont les étiquettes
amont ont été retirées ; GCC système en C23 par défaut incompatible
avec le C vendoré K&R d'`oniguruma`), signalés avant correction,
détail complet et preuve de la poignée de main LSP :
`docs/completion.md` § 7.

**D20 (2026-08-07, CMP-1) — modèle de complétion à la demande, pas
résident.** Requalifie D15 (ci-dessus, marquée `[REQUALIFIÉE]`, pas
effacée). Motif : la résidence permanente coûte ~8 W en continu et
neutralise la veille runtime D3 (D15) — l'engager avant d'avoir établi
que la complétion sert réellement au quotidien serait payer un coût
certain pour un bénéfice supposé, d'autant que le mécanisme qui la
justifierait (`lsp-ai`) vient d'être établi comme fonctionnel mais
non éprouvé en usage réel (aucun modèle chargé à ce jour). Révisable
après usage mesuré — pas une fermeture définitive de la résidence,
une séquence : à la demande d'abord, résident ensuite si l'usage le
justifie.

**D21 (2026-08-07, IA-3) — modèles retenus : chat/agent =
`mistral-nemo:12b-instruct-2407-q4_K_M`, complétion =
`qwen2.5-coder:7b-instruct-q4_K_M`, tous deux Apache-2.0.** Motif du
choix de licence pour le chat/agent (`mistral-nemo` plutôt que
`codellama:13b`, Llama 2 Community License) : ce dépôt est public
**et destiné à être cloné pour un usage en entreprise** — une licence
qui porte des conditions d'usage jusque dans ce contexte est écartée,
Apache-2.0 non. Contexte visé pour le chat : 32 K. `qwen2.5-coder:3b`
écarté (seule taille restreinte — Qwen Research License — d'une
famille par ailleurs Apache-2.0), sur le seul motif de licence, sans
lecture de ses termes complets (marqueur requalifié,
`docs/local-ai.md` § 1).

**Coexistence mesurée, pas supposée — ne tient pas.** Chat seul à
32 K : 12 294 Mio VRAM mesurés (contre ≈12,5 Gio estimés en IA-2 —
écart expliqué : confusion Gio/GB sur les poids dans l'estimation
d'origine, la formule de cache KV, elle, était juste à moins de 1 %
près, `docs/local-ai.md` § 9.3). Complétion seule à son contexte réel
(2000, celui que `roles/completion/` configure) : 4 690 Mio. **Les
deux ensemble : Ollama décharge systématiquement l'un pour charger
l'autre**, testé dans les deux sens — ~17 Gio pour une enveloppe
mesurée de ~15,9 Gio, exactement ce que l'arithmétique de la demande
anticipait. Trois leviers chiffrés, aucun recommandé
(`OLLAMA_KV_CACHE_TYPE=q8_0` : -2,29 Gio, qualité non chiffrée ;
contexte réduit à 16 K : -2,38 Gio ; bascule séquentielle : 3,4-4,1 s
par changement de modèle) — décision laissée à l'opérateur.

Récupération par **conteneur de récupération éphémère**, distinct du
conteneur de service — jamais le même réseau (le service reste sur le
réseau interne dédié D18, la récupération utilise le réseau par
défaut de Podman, disjoint) — préféré à `podman network connect`
temporaire (proposé en IA-2 § 8.5) parce que ce dernier aurait rendu
l'isolation du service momentanément une question de discipline
plutôt qu'un fait structurel vérifiable. Isolation du conteneur de
service prouvée par inspection à quatre moments (avant/pendant/après
récupération, après arrêt) — identique aux quatre relevés, deux fois.

**Intégrité du registre de modèles, testée par corruption délibérée,
pas supposée** : `ollama pull` vérifie le contenu reçu sur le réseau
(« verifying sha256 digest »), mais ne détecte ni ne répare un blob
**déjà présent localement** corrompu après coup sous son nom attendu —
écart nommé, `docs/repositories.md` § 8.

RTD3 confirmé bloqué en continu par un contexte CUDA ouvert (modèle
chargé, lecture passive sur 60 s : `runtime_suspended_time` figé,
`runtime_active_time` progresse au rythme du temps réel) — consommation
mesurée au repos, modèle chargé : 3,49 W (P8), très en dessous des
pointes transitoires de sonde (33-55 W).

Test bout-en-bout `lsp-ai`/Helix (CMP-1) : la poignée de main LSP et
l'envoi automatique de requêtes de complétion à la frappe fonctionnent
toujours ; la complétion échoue pour une cause précisément identifiée
et distincte des deux anticipées (ni `lsp-ai`, ni RTD3/rechargement) —
le nom de modèle placeholder de CMP-1 (`aucun-modele-charge-D20`) ne
correspond à aucun modèle réel, erreur Ollama immédiate et claire.
Correction hors du périmètre de ce livrable (`roles/completion/` non
touché) — signalée, pas corrigée.

**D22 (2026-08-08, IA-4) — chargement séquentiel des modèles.** Ollama
décharge l'un pour charger l'autre ; les deux cohabitent sur disque,
jamais en VRAM. Motif : seul levier chiffré parmi les trois nommés par
D21 (`OLLAMA_KV_CACHE_TYPE=q8_0`, contexte réduit à 16 K, bascule
séquentielle) dont le coût est **mesuré** (3,4-4,1 s par bascule, cache
disque chaud) plutôt que supposé, et seul à ne défaire aucune décision
antérieure — `q8_0` dégraderait la précision du cache d'une quantité
non chiffrée, réduire à 16 K annulerait le contexte choisi
délibérément pour le chat/agent (D21). **Mesuré en conditions réelles**
(`docs/completion.md` § 8.2) : bascule chat → complétion, 3,727 s,
dans la fourchette annoncée ; complétion en régime établi (modèle déjà
résident), 0,46-0,48 s — verdict : vivable au quotidien
(`docs/completion.md` § 8.3).

**Écart résiduel, non comblé par IA-4** : les temps de bascule sont
mesurés à cache disque **chaud** (service actif sans interruption
depuis un précédent livrable) — `@VERIF` le facteur de ralentissement
du **premier** chargement après démarrage du service (cache disque
froid), non établi.

**D23 (2026-08-09, PKG-1) — retrait de `cardwire`** (installé
manuellement le 2026-08-04, transaction 20, depuis `terra`). **Motifs
cumulés** : `rpm -q --whatrequires cardwire` — aucun paquet ne le
requiert ; `cardwired.service` `disabled`/`inactive`, jamais démarré ;
sa version `-2` fournit `switcheroo-control` et **remplacerait un
service Fedora actif et activé** (`switcheroo-control-3.0-5.fc44`),
ce qui bloquait `sudo dnf update` à chaque exécution
(« Skipping packages with conflicts ») ; il touche à la commutation
graphique, sujet sur lequel D2ter a explicitement écarté un second
commutateur pour éviter les conflits ; et l'opérateur ne s'en sert
pas. Détail complet, vérifications avant/après retrait :
`docs/repositories.md` § 9.

**Considération de méthode** : un paquet écarté à chaque mise à jour
est une **dérive silencieuse** — il reste figé indéfiniment et le
message devient du bruit qu'on cesse de lire. Le retrait ferme la
dérive plutôt que de la tolérer.

**Fait annexe consigné à cette occasion** : `switcheroo-control` porte
`from_repo=6ecc2dfaa0dc41e5ad51e007707a786b` — catégorie « paquets du
média d'installation », nommée pour la première fois comme telle ici
(le fait brut, la valeur `from_repo` elle-même, était déjà consigné
depuis BUR-1, `docs/repositories.md` § 4) : **1170 paquets**
(re-mesuré le 2026-08-09 ; **ÉTAIT 1195** au compte initial, écart de
25 non recherché individuellement — cohérent avec des remplacements
normaux par mises à jour depuis, § détail dans `docs/repositories.md`
§ 4). C'est `switcheroo-control` qui fait de cette catégorie plus
qu'un compte pour mémoire : sans elle, ce paquet aurait été invisible
à toute recherche par dépôt configuré.

**D24 (2026-08-10) — révision de D12 : npm ouvert pour GitHub Copilot
CLI, installation directe sur l'hôte.** Besoin : un second agent de
code (`@github/copilot`), pour répartir la consommation entre
l'abonnement GitHub et Claude Code. Copilot CLI se distribue par npm —
collision directe avec D12 (§ ci-dessus, `docs/editor.md` § 2). Deux
décisions de l'opérateur consignées ici : npm ouvert (voie
conteneurisée examinée et écartée) ; authentification interactive
(D25, ci-dessous).

**Ce qui reste vrai de D12, ce qui change** : le motif d'origine — npm
échappe à l'ancrage de confiance de `docs/repositories.md`, modèle de
dépendances transitives sans commune mesure avec rpm — reste
**intégralement vrai**. Ce qui change est l'arbitrage : la
contrepartie (second agent de code) est jugée acceptable. `lsp-ai`
reste compilé depuis les sources (D19, motif indépendant — absence de
somme de contrôle publiée sur son binaire précompilé, sans rapport
avec npm) : **pas rouvert** par cette révision.

**Garde D12 resserrée, pas retirée** — dans `roles/editor/` et
`roles/completion/` (seconde occurrence indépendante du même
mécanisme, découverte en écrivant D24, hors périmètre initialement
annoncé, corrigée avec l'accord explicite de l'opérateur) : ce que D12
protégeait réellement n'a jamais été « npm absent », mais qu'aucun
serveur de langage (`yaml-language-server`, `ansible-language-server`)
ne provienne de ce canal, et que `lsp-ai` reste le binaire compilé.
Démontrée dans les deux sens, dans les deux rôles indépendamment,
`docs/editor.md` § Copilot CLI.

**npm : sixième surface d'approvisionnement** (`docs/repositories.md`
§ 11) — ancrage réel supérieur à ce que D19 avait constaté pour
`lsp-ai` précompilé : intégrité de contenu (sha512) + signature du
registre + attestation de provenance Sigstore pour le paquet principal
(`npm audit signatures`, vérifié localement). Aucun fichier de
verrouillage possible pour une installation globale
(`npm install -g` n'en génère jamais) — version épinglée dans la
commande d'installation elle-même retenue comme seule voie
d'épinglage disponible (`@github/copilot@1.0.78`, jamais `@latest`).

**Installé, vérifié, version épinglée** : runtime Node.js 22
(`nodejs22-bin`/`nodejs22-npm-bin`, `fedora`/`updates`,
`install_weak_deps: false` — 5 paquets/88 Mio contre 7/169 Mio avec
les dépendances faibles, documentation et internationalisation
complètes sans usage ici), préfixe npm dans le domaine utilisateur
(`~/.local`, même discipline que `lsp-ai`/venv `ansible-lint`) —
`/usr/local`, préfixe par défaut sur ce poste, est `root:root 0755`,
recommandation de npm lui-même (source externe nommée,
`docs/editor.md` § Copilot CLI) pour éviter `sudo npm install -g`.
`roles/copilot_cli/` (nouveau rôle — argumentaire du choix contre
`roles/editor/`/`roles/completion/`, `docs/editor.md` § Copilot CLI),
inséré dans `site.yml` après `completion`, indépendant.

**D25 (2026-08-10) — authentification Copilot CLI par `copilot login`
interactif, jamais un jeton d'accès personnel.** `copilot login` ouvre
un flux OAuth (navigateur ou code d'appareil) et enregistre lui-même
les identifiants (trousseau système, ou `~/.copilot/` en repli) —
**jamais lancé par ce dépôt**, geste manuel de l'opérateur ajouté à
`docs/orchestration.md` § 2bis. Confirmé après l'installation complète
de ce livrable : `~/.copilot/` n'existe pas.

**Ordre de précédence des jetons** (lu directement dans l'aide
intégrée du binaire installé, `copilot help environment`, corroboré
par la documentation amont externe) : `COPILOT_GITHUB_TOKEN` >
`GH_TOKEN` > `GITHUB_TOKEN` > identifiants enregistrés. **Voie
écartée, motif écrit** : définir `GH_TOKEN`/`GITHUB_TOKEN`
**globalement** exposerait un jeton à tout l'outillage du poste, y
compris aux agents exécutant des commandes arbitraires — surface sans
commune mesure avec `copilot login`, qui confine le jeton au trousseau
système. `roles/copilot_cli/` ne lit, n'écrit ni ne vérifie aucune de
ces variables.

**Vérification non interactive de l'authentification — recherchée,
absente, pas de substitut inventé** : aucun flag, sous-commande ni
code de retour documenté (documentation amont externe + aide intégrée
du binaire, concordantes) ne distingue « authentifié » de « non
authentifié » sans lancer une session réelle. `roles/copilot_cli/` ne
garde donc pas cette étape — un agent non authentifié échoue au
premier usage réel, tardivement, accepté tel quel plutôt que contourné
par une garde qui ne vérifierait rien de réel.

**Modèles et suivi de consommation — non établis avant
authentification, marqué comme tel, aucune parité avec Claude Code
affirmée.** Liste réelle des modèles dépendante de l'abonnement et de
la région (mentions publiques : Claude Opus 4.6, Claude Sonnet 4.6,
GPT-5.3-Codex, Gemini 3 Pro, Claude Haiku 4.5 — non vérifiable sans
authentification, hors périmètre de ce livrable). Commandes préparées,
toutes deux internes à la session interactive, pas des indicateurs
shell : `/model` (liste des modèles offerts, après `copilot login` —
aucun flag CLI documenté ne la donne hors session, recherché) ;
`/usage` (consommation de la session courante) ou
`github.com/settings/copilot` (relevé mensuel, hors CLI).

**D26 (2026-08-16) — JDK complet : `java-25-openjdk-devel`, pas
d'ouverture de la surface Adoptium Temurin.** Première décision d'une
chaîne de build Android CLI destinée au dépôt `glass-hud` (distinct,
séparé de la production de ce poste, jamais géré depuis ici —
préambule). Mesuré sur ce poste : les dépôts déjà activés
(`fedora`/`updates`) n'offrent que `java-25-openjdk-devel` et
`java-latest-openjdk-devel` (26, préversion) — aucun JDK 17 ni 21
n'existe pour Fedora 44 dans ces dépôts (`dnf list --available
"java-*-openjdk-devel"`, glass-hud livrable 6). L'Android Gradle Plugin
documente un minimum de JDK 17, sans borne haute
(`developer.android.com/build/releases/gradle-plugin`, table de
compatibilité lue en entier, glass-hud livrable 7). **Validé
empiriquement avant toute écriture sur ce poste** : dans un conteneur
Fedora 44 jetable (`quay.io/fedora/fedora-minimal`, épinglé par
empreinte, détruit après usage), un projet Kotlin/Compose généré par
l'outillage officiel (`android create`) compile et s'empaquette avec
succès sous ce JDK — Gradle 9.1.0, auto-téléchargé par le wrapper du
projet, annonçant lui-même « Full Java 25 support » (glass-hud
livrable 7, rapport `07-rapport.md`, hors de ce dépôt). Retenu plutôt
que d'ouvrir Adoptium Temurin — paquet bootstrap
(`adoptium-temurin-java-repository`) déjà résoluble depuis `fedora`
mais jamais installé, sur-couverture évitée sur une incertitude que la
mesure a levée (`CLAUDE.md` § Avant d'agir).

`@VERIF : le livrable 7 (glass-hud) a démontré la compatibilité pour un
AGP résolu par un template (« empty-activity », tag « agp-9 »), sous
Gradle 9.1.0. Or la table de compatibilité officielle donne pour l'AGP
courant (9.3.0, lue le 2026-08-16) une exigence de Gradle 9.5.0,
strictement supérieure à 9.1.0 : l'AGP exercé n'était donc PAS l'AGP
courant. Ce qui est établi est plus étroit que « l'AGP courant
fonctionne sous JDK 25 ». À lever quand glass-hud épinglera ses
versions réelles (AGP et Gradle), pas avant — non bloquant pour ce
livrable, qui n'installe que le JDK.`

**Fermeture de dépendances, mesurée avant écriture** (`dnf install
--assumeno java-25-openjdk-devel`) : 5 paquets — `java-25-openjdk-devel`
(11,6 Mio), `java-25-openjdk` (960,5 Kio), `mkfontscale` (44,9 Kio),
`ttmkfdir` (142,2 Kio), `xorg-x11-fonts-Type1` (863,3 Kio). Total : 7 Mio
à télécharger, 14 Mio installés. **Le paquet `-devel` tire bien le
paquet non-headless, qui tire lui-même des paquets liés à X11 (polices)
— confirmé par mesure, pas supposé** ; volume négligeable, pas les
gigaoctets qu'une supposition non vérifiée aurait pu laisser craindre.
Retour arrière exact : `dnf remove java-25-openjdk-devel
java-25-openjdk mkfontscale ttmkfdir xorg-x11-fonts-Type1` (non
rejoué : ces cinq paquets restent voulus sur ce poste après ce
livrable).

**Point de vigilance nommé par le pilote, exercé pour la première fois
ici** : installation de `java-25-openjdk-devel` par-dessus
`java-25-openjdk-headless` déjà présent — jamais fait avant ce
livrable (le conteneur du livrable 7 partait vierge de tout Java).
Mesuré avant/après (`alternatives --list`, filtré `java`) : les quatre
entrées déjà présentes (`java`, `jre_openjdk`, `jre_25`,
`jre_25_openjdk`) sont restées **inchangées**, `java` résolvant
toujours vers le même chemin (`/usr/lib/jvm/java-25-openjdk/bin/java`)
— trois nouvelles entrées sont apparues (`javac`, `java_sdk_openjdk`,
`java_sdk_25`), toutes vers le même répertoire. **Nuance à ne pas
perdre** : ceci confirme la coexistence propre entre un paquet headless
et son complément `-devel` de LA MÊME VERSION — pas la coexistence de
DEUX VERSIONS DIFFÉRENTES de JDK via `alternatives` (ex. 17 à côté de
25), scénario resté hors de portée puisqu'aucun second JDK n'a été
nécessaire (D26 ci-dessus). Ce second scénario, plus délicat, reste
à exercer si un jour une version différente devient nécessaire.

**Mesure complémentaire, par le `PATH`, pas seulement par le chemin
absolu.** Le rôle vérifie `javac` par son chemin absolu
(`/usr/lib/jvm/java-25-openjdk/bin/javac`, correct pour son objet : la
garde post-installation cible précisément ce fichier). Avant ce
livrable, `command -v javac` était vide (rc=1, `glass-hud` livrable 8,
état de départ). Après, rejouée :
```
$ command -v javac
/usr/bin/javac
rc=0
```
Établit que `javac` est aussi atteignable par le `PATH` (via
`/usr/bin/javac`, le lien `alternatives` — voir ci-dessus), ce que la
garde du rôle ne vérifie pas et n'a pas à vérifier pour son objet
propre. Observation d'abord rapportée par l'opérateur, rejouée et
confirmée directement ici (`CLAUDE.md` § Sourcing des faits — une
observation rapportée devient une mesure directe une fois rejouée avec
succès, elle ne le reste pas indéfiniment comme classe à part).

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
  `AE09157A4DE88B497EA1D5D300CDAB43DE226D6F`. **[MARQUEUR FERMÉ le
  2026-08-06]** ÉTAIT marqué : empreinte à comparer à celle publiée par
  Fyra Labs avant tout (ré)import, l'ID court seul ne prouvant rien.
  Recherche menée sur neuf canaux distincts du serveur de dépôt
  (`docs/repositories.md` § 1) : aucun ne publie cette empreinte —
  absence établie, pas laissée en suspens. Aucune action dans ce
  livrable : rien n'est importé, aucun dépôt n'est modifié ou désactivé.
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
  dédié. **[TRAITÉ le 2026-08-06]** Voir D10 (§ Décisions) et
  `docs/repositories.md` : dépôt toujours activé, désactivation
  toujours exclue, aucun paquet remplacé, correctif de la récurrence
  d'invite appliqué (portée limitée au cache utilisateur, aucune
  dégradation de vérification), recherche d'équivalents Fedora/COPR
  menée et consignée sans action.
- **`rpmfusion-free-tainted`** activé délibérément (transaction 18), lié à la
  chaîne multimédia. Établi, pas un point ouvert au sens strict — conservé
  ici pour traçabilité car demandé explicitement.
- **Coexistence VA-API** `mesa-va-drivers-freeworld` / `libva-nvidia-driver` :
  les deux sont bien installés (transactions 14–17 dans l'historique dnf).
  Arbitrage (quel pilote VA-API prime pour quel usage) non déterminé.
  Priorité basse.
- **[CORRIGÉ le 2026-08-07, IA-2] `NVreg_PreserveVideoMemoryAllocations`
  commenté** dans `/usr/lib/modprobe.d/nvidia-power-management.conf`.
  **ÉTAIT** : « les allocations VRAM ne survivent pas à la veille » —
  conclusion jamais sourcée par une lecture de la valeur brute
  réellement chargée par le pilote, et qui conflait deux mécanismes
  distincts : la veille **runtime** RTD3 (`Video Memory: Off`,
  par périphérique, indifférente à une allocation VRAM résiduelle sans
  contexte CUDA actif) et la **suspension système** (S0ix/s2idle sur ce
  poste), seule concernée par ce paramètre — les deux confondus sous
  « la veille » dans la formulation d'origine. Corrigé par mesure
  directe (IA-1, `docs/local-ai.md` § 2.1bis) : la valeur brute
  effectivement chargée est `2`, ni `0` (l'hypothèse implicite ici) ni
  `1` — signification de `2` non documentée dans les sources amont
  consultées, point resté ouvert (`docs/local-ai.md` § 2.1bis,
  marqueur de vérification, pas repris ici pour ne pas dupliquer la
  règle de comptage — voir CLAUDE.md § le jeton ne s'écrit jamais nu).
  Traité au livrable IA (D16, 2026-08-07) :
  `NVreg_PreserveVideoMemoryAllocations=1` déployé par écrasement
  propre — voir § Décisions D16 ci-dessous.
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
- **Nouveau point ouvert (2026-08-09) — mécanisme exact du gel de
  `eDP-1` à l'ouverture de session suivant BUR-2, non établi par les
  journaux.** Diagnostic complet dans `docs/desktop.md` § 8 : démarrage
  fautif identifié (`-3`, `19f9256bdefb4b6db1b49a4aa44b1e12`,
  2026-08-09 10:40:35–10:45:47 CEST), causes noyau/saturation/extinction
  de sortie éliminées, hypothèse de l'interaction plein écran hors
  composition/validation atomique de la sortie sœur non confirmée faute
  de trace journalisée pendant la fenêtre du gel. `startup.session`
  renommé en `.bak` à la main, hors dépôt — `roles/desktop/`/`site.yml`
  **ne pas rejouer** avant résolution (recréeraient l'état fautif).
  Reproduction contrôlée (phase B, `docs/desktop.md` § 8.5) **en
  attente de confirmation explicite de l'opérateur**, TTY testé au
  préalable inclus — non entamée dans ce livrable. Se ferme soit par
  reproduction concluante, soit par abandon assumé du plein écran au
  profit des seuls volets (`docs/desktop.md` § 8.8).
  **[MISE À JOUR le 2026-08-09, même journée] Phase B entreprise après
  confirmation de l'opérateur — résultat négatif.** TTY testé et
  fonctionnel dans les deux sens ; commande de récupération établie par
  lecture puis vérifiée réellement depuis le TTY (deux échecs
  intermédiaires résolus par lecture — voir `docs/desktop.md` § 8.9) :
  `QT_QPA_PLATFORM=wayland WAYLAND_DISPLAY=wayland-0 kscreen-doctor
  output.eDP-1.disable output.eDP-1.enable`. Protocole en trois étapes
  suivi dans l'ordre, confirmation de l'opérateur et relevé
  noyau/`kwin_wayland`/sorties après chacune (détail `docs/desktop.md`
  § 8.10) : **aucune des trois n'a reproduit le gel**, y compris
  l'étape 3 qui rejoue la configuration exacte du fichier réellement
  figé le 09/08 (copie de `startup.session.bak`, `diff` confirmé). Ce
  résultat négatif **affaiblit** l'hypothèse de l'interaction statique
  plein écran/validation atomique prise seule — un facteur de
  concurrence propre à l'ouverture de session (autostart, compositeur
  en cours d'initialisation) reste la piste la plus plausible, non
  testée par cette phase et non testable sans reproduire le risque déjà
  rencontré (redéployer `startup.session` pour tester à une ouverture
  de session authentique). **Ne reste donc PAS fermé** : la
  qualification « se ferme par abandon assumé du plein écran » ci-dessus
  est elle-même requalifiée — le plein écran n'est plus soutenu comme
  facteur suffisant par une preuve directe (`docs/desktop.md` § 8.10).
  `startup.session.bak` inchangé, `roles/desktop/`/`site.yml` toujours
  pas rejoués.
  **[MISE À JOUR le 2026-08-09, livrable suivant] `roles/desktop/`
  rejoué délibérément — mécanisme mitigé, pas fermé.** Décision de
  l'opérateur : garder le plein écran, temporiser le démarrage sur la
  stabilité des sorties (`kscreen-doctor -j`, échantillonnage,
  `docs/desktop.md` § 9). `roles/desktop/` exécuté réellement avec ce
  nouveau mécanisme — `startup.session.bak` désormais un résidu
  historique sans effet, le fichier actif est de nouveau celui déployé
  par le rôle. **Ce point reste ouvert** : ce livrable mitige une
  hypothèse (le facteur de concurrence à l'ouverture de session), il ne
  l'établit pas comme cause — seule une ouverture de session
  authentique le dira, non entreprise ici pour la même raison qu'en
  phase B (reproduire le risque déjà rencontré). Si le gel se
  reproduit malgré la temporisation, l'issue à retenir (posée à froid,
  `docs/desktop.md` § 9.4) est le retrait du plein écran — pas une
  nouvelle tentative de calibrage.
  **[MISE À JOUR le 2026-08-09, livrable suivant (BUR-5)] Ouverture de
  session sans aucune information sur ce point, ni pour ni contre.**
  L'autostart kitty ne s'est pas lancé du tout à cette ouverture de
  session — bug distinct, sans rapport (unité systemd jamais générée,
  faute de chemin absolu dans `Exec=`/`TryExec=` ; corrigé,
  `docs/desktop.md` § 10). Kitty n'ayant jamais atteint le plein écran
  cette fois, le mécanisme que la temporisation BUR-4 protège n'a
  simplement pas été exercé — ce livrable ne confirme ni n'infirme
  l'hypothèse de concurrence, il ne l'a pas testée. **Ce point reste
  ouvert**, inchangé par BUR-5.
  **[MISE À JOUR le 2026-08-09, livrable de clôture] Deux ouvertures de
  session réelles depuis, kitty opérationnel — mais la temporisation
  n'a toujours pas été exercée pour de vrai.** Correctif BUR-5
  confirmé en usage réel : `kitty-startup-wait.sh`, invoqué par
  l'autostart à deux démarrages successifs (`19:45:13`, boot `-1` ;
  `19:49:32`, boot `0` — `journalctl --user`, hors de toute invocation
  Ansible), décide « nominal » en `269ms` puis `268ms`. Dans les deux
  cas, exactement **deux** échantillons, tous deux `rc=0`, identiques
  dès le premier — la topologie était déjà stable au moment du tout
  premier relevé. C'est le **plancher structurel** du script
  (`(desktop_kitty_wait_stable_samples - 1) × desktop_kitty_wait_interval_seconds`
  = 0,2 s, plus le coût réel de deux invocations `kscreen-doctor -j`),
  pas une attente réelle mesurée : la boucle n'a jamais eu à itérer au
  delà du minimum. **Kitty est donc opérationnel à l'ouverture de
  session (le bug BUR-5 est réellement corrigé)**, mais la
  temporisation elle-même reste **non exercée** au sens de la règle
  ci-dessus (§ Avant d'agir, `CLAUDE.md`) — aucune de ces deux
  ouvertures n'a présenté de topologie instable à absorber. **Ce point
  reste ouvert** : rien ne permet toujours d'attribuer une éventuelle
  absence de gel à ce mécanisme, faute de l'avoir vu réellement
  attendre.
- **Nouveau point ouvert (2026-08-09, BUR-5) — contenu exact de
  l'environnement vu par `systemd-xdg-autostart-generator` au moment de
  son exécution réelle.** Établi seulement par ses effets (un nom seul
  dans `Exec=`/`TryExec=` ne s'y résout jamais, un chemin absolu s'y
  résout toujours — reproduit à volonté via `systemctl --user
  daemon-reload`, `docs/desktop.md` § 10.3), pas par lecture directe de
  la table d'environnement du processus lui-même : `strace` absent de
  ce poste, un sondage `/proc/<pid>/environ` par boucle shell trop lent
  face à la durée de vie du processus (~4 ms). Marqueur de vérification
  posé dans `docs/desktop.md` § 10.3. Sans conséquence opérationnelle
  (la correction retenue, chemin absolu, ne dépend pas de ce détail) —
  se fermerait par `strace` disponible sur ce poste, ou par lecture du
  code source de `systemd-xdg-autostart-generator` établissant
  explicitement la construction de cet environnement.
- **Nouveau point ouvert (2026-08-09, PKG-1) — combien de paquets
  installés ne proviennent d'aucun rôle de ce dépôt.** `cardwire` (D23)
  n'a été découvert que parce qu'il produisait un conflit visible à
  chaque `dnf update` ; un paquet orphelin silencieux, sans conflit,
  ne serait jamais repéré par ce mécanisme. Ordre de grandeur établi,
  pas un compte exhaustif (pas demandé) : `rpm -qa | wc -l` → **2504**
  paquets installés sur ce poste (2026-08-09, après retrait de
  `cardwire`) ; les rôles de ce dépôt (`bootstrap`, `editor`,
  `completion`, `desktop`, `gpu_cdi` — `recovery`, `gpu_mux`,
  `local_ai` n'installent aucun paquet) déclarent explicitement **15**
  paquets RPM distincts via `ansible.builtin.dnf`/`package`. **Écart :
  de l'ordre de 2 490 paquets** ne sont nommés par aucune tâche
  d'aucun rôle — l'écrasante majorité de ce qui est installé sur ce
  poste. Composition non distinguée (pas demandé, exhaustivité non
  requise) : dépendances transitives des 15 paquets déclarés, base de
  l'image d'installation (catégorie « média d'installation » ci-dessus,
  D23 — 1170 à elle seule), et paquets installés manuellement au fil
  des livrables (`asusctl`, `asusctl-rog-gui`, `terra-gpg-keys`/
  `terra-release` en dehors de ce que `bootstrap` déclare lui-même,
  entre autres — **ÉTAIT** cité ici : `supergfxctl`, dont ce point
  ouvert ignorait, faute de l'avoir cherché, qu'il n'était déjà plus
  installé à cette date, cf. **[TRAITÉ]** ci-dessous). **Ce qu'il
  faudrait pour traiter ce point** : un inventaire qui distingue,
  paquet par paquet, ce qui est strictement une dépendance transitive
  résolue par `dnf` (attendu, pas une dérive) de ce qui a été installé
  par une commande manuelle jamais reproduite dans un rôle (candidat à
  devenir un second `cardwire` silencieux) — non entrepris ici, sujet
  en soi (`docs/repositories.md` § 9).
  **[TRAITÉ le 2026-08-09, PKG-2]** L'inventaire demandé ci-dessus a
  été fait, en se restreignant volontairement aux **décisions
  d'installation** (l'historique `dnf`, pas `rpm -qa`) plutôt qu'à la
  distinction dépendance/manuel à l'échelle des ~2 490 paquets, jugée
  hors de portée d'un seul livrable et pas nécessaire à répondre à la
  question réelle (« qu'est-ce que `site.yml` ne reproduirait pas ? »)
  — détail complet, méthode et tableaux dans `docs/packages.md`.
  Résultat : 33 paquets relèvent d'une décision d'installation vivante
  hors socle, dont 15 déjà reproduits par un rôle, 9 nécessaires mais
  non reproduits (dont trois jamais documentés avant ce livrable —
  `asusctl`, `claude-code`, `htop`, nouveau point ouvert ci-dessous),
  0 candidat au retrait comme `cardwire` l'était, et 9 de confort
  personnel ou d'outillage de développement du dépôt, jamais discutés
  avant ce livrable (chaîne multimédia, `NetworkManager-tui`,
  `asusctl-rog-gui`, `python3-pip`) — légitimes, maintenant nommés
  plutôt que subis. **Découverte non cherchée, signalée plutôt que
  corrigée en silence** (`CLAUDE.md` § Avant d'agir) : `supergfxctl`
  n'est plus installé sur ce poste, retiré silencieusement par `dnf`
  le 2026-08-07 (transaction 30, mise à jour de masse) via
  l'obsolescence déclarée par `terra-obsolete` — pas par l'opérateur,
  jamais un conflit visible comme pour `cardwire`. Rend périmée la
  composition détaillée (pas le compte) de la ligne `terra` dans
  `docs/repositories.md` § 4/§ 9 (PKG-1) — non corrigée ici, hors
  périmètre strict de PKG-2, signalée pour le prochain livrable qui y
  touchera (`docs/packages.md` § 2.3).
- **Nouveau point ouvert (2026-08-09, PKG-2) — trois paquets
  nécessaires à ce que `site.yml` promet, jamais reproduits par aucun
  rôle, jamais documentés avant ce livrable.** Détail complet, preuves
  et argumentaire dans `docs/packages.md` § 2.2/§ 3 — résumé ici pour
  que ce fichier reste la source unique des points non résolus.
  **`asusctl`** — **[MOTIF CORRIGÉ le 2026-08-09, GPU-4]** ÉTAIT :
  nécessaire parce que l'application réelle de `gpu_mux_mode` au
  redémarrage passerait par `asus-shutdown.service`, fourni
  exclusivement par ce paquet — **plus affirmatif que ce qui a été
  établi**. La révocation partielle de D2ter (ci-dessus, § Décisions) a
  consigné explicitement que l'écriture sysfs directe et la file
  d'`asusd` portaient, le 2026-08-05, la même valeur cible, et que
  **laquelle des deux a effectivement produit le changement matériel
  observé est indéterminable a posteriori**. Motif qui tient
  réellement : `asusd` (fourni par `asusctl`) gère les profils
  d'alimentation et les courbes de ventilation propres à ce matériel
  (`/etc/asusd/asusd.ron` : `platform_profile_on_battery`,
  `platform_profile_on_ac`, `platform_profile_linked_epp` ; `asusctl
  profile`/`asusctl fan-curve`) — **D6** a choisi `power-profiles-daemon`
  explicitement « pour sa cohérence avec `asusd` » (§ Décisions), motif
  qui présuppose `asusd` présent. Sur une machine neuve sans `asusctl`,
  `roles/bootstrap/` installerait `power-profiles-daemon` sans erreur
  (D6 n'en dépend pas pour s'exécuter) — seule la cohérence avec
  `asusd`, raison même du choix, ne tiendrait plus, sans qu'aucun test
  actuel ne le remarque. Pas un blocage qui casse, une prémisse de
  décision silencieusement caduque — détail complet,
  `docs/packages.md` § 2.2/§ 3. **`claude-code`** et **`htop`** :
  `roles/desktop/tasks/main.yml` porte une garde explicite
  (`ansible.builtin.assert`) qui échoue bruyamment si `claude`/`htop`
  sont absents du `PATH` avant de déployer la session de démarrage
  kitty — garde qui fonctionne comme prévu, mais aucun rôle
  n'installe l'un ou l'autre : sur une machine neuve, `site.yml`
  s'arrêterait à ce point, immédiatement et explicitement. Recommandation
  de PKG-2, pas une décision prise ici (`CLAUDE.md` § Avant d'agir —
  ce livrable était en lecture seule, aucun rôle modifié) : ajouter
  `asusctl` aux paquets Terra de `roles/bootstrap/`, et choisir
  explicitement pour `claude-code`/`htop` entre les déclarer dans un
  rôle (`desktop`, aux côtés de `kitty`/`jq`) ou documenter leur
  absence de `site.yml` comme un choix assumé.
- **[REQUALIFIÉ le 2026-08-10] ÉTAIT : point ouvert, basse priorité —
  liste réelle des modèles Copilot CLI offerts par cet abonnement, non
  établissable sans authentification, se refermant « quand l'opérateur
  authentifie et relève la liste réelle ».** L'opérateur a lancé
  `copilot login` et `/model` lui-même (hors de ce dépôt) et relevé la
  sortie — **sortie de commande copiable et relisible, pas une
  observation rapportée** (détail complet, citation intégrale,
  `docs/editor.md` § Modèles et suivi de consommation). **Établi** :
  modèle actif `claude-sonnet-5`, niveau d'effort `medium` — **parité
  de génération de modèle** avec Claude Code (cette session s'identifie
  elle-même comme `claude-sonnet-5`), **pas une parité d'effort**
  (paramètre distinct, non établi). Consommation : 3 700/20 000 AIC
  (18 %), période de renouvellement du plafond **non établie par cette
  sortie**, marquée inconnue plutôt que supposée mensuelle. **Reste
  non établi, marqué comme tel plutôt qu'interprété** : la liste
  complète des *autres* modèles offerts par cet abonnement (la sortie
  ne rapporte que le modèle actif, pas la liste du sélecteur) ; la
  portée exacte de `AI Credits 0 (1m 57s)`/`0 messages` sur 180 jours
  (vraisemblablement la session courante, sans échange — pas établi
  par une source). Point ouvert restreint à ces deux éléments
  résiduels, plus le même que le précédent.
- **Découverte annexe, signalée, non traitée (2026-08-10, D24)** :
  `roles/completion/README.md` affirme encore « Il ne configure jamais
  Kate — compatibilité jamais testée par personne » — périmé depuis
  KAT-1 (2026-08-08, greffon LSP de Kate câblé), trouvé en relisant ce
  fichier pour la révision de sa garde D12, sans rapport avec D24.
  Signalé pour un prochain livrable qui touchera ce fichier,
  non corrigé ici (hors périmètre de celui-ci).
- **Branche non exercée, signalée plutôt que forcée (2026-08-16, D26,
  `roles/android_jdk/`)** : la séquence réellement jouée a été
  installation-en-échouant (chemin `javac` substitué, l'installation
  réelle du paquet a eu lieu à ce moment) puis vérification-réussie sur
  un système déjà installé (exécutions suivantes, idempotentes) — jamais
  « installation puis vérification réussie dans la même passe, sur un
  système qui ne l'avait pas », le cas qui se produira sur une machine
  redéployée depuis zéro, raison d'être de ce dépôt. Non exercée
  délibérément : désinstaller le JDK déjà en place sur ce poste pour
  rejouer proprement aurait démonté quelque chose de réel afin de
  produire une preuve — jugé disproportionné, même patron que le refus
  déjà posé dans `roles/bootstrap/README.md` (§ Amorçage) de désactiver
  `NOPASSWD` sur une machine où il fonctionne déjà. Se referme
  naturellement au premier redéploiement réel depuis zéro (§ Ce qui
  reste hors de ce flux, `docs/orchestration.md`, ou une machine de
  test dédiée si l'opérateur en construit une avant).

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
- **2026-08-06 (série suivante) — clôture du dossier Terra, inventaire
  des dépôts (`docs/repositories.md`, nouveau, D10).** Écart signalé
  d'emblée : le contexte annonçait douze dépôts activés, ce poste en
  porte onze (vérifié, `google-chrome` et le COPR `phracek/PyCharm`
  existent comme fichiers `.repo` mais sont désactivés) — cause non
  creusée, sans conséquence sur l'objet du livrable.
  Empreinte locale de la clé Terra confirmée par trousseau temporaire
  isolé (base rpm et fichier système concordent :
  `AE09157A4DE88B497EA1D5D300CDAB43DE226D6F`). Corroboration
  indépendante recherchée sur neuf canaux distincts du serveur de
  dépôt — introuvable ; un dixième canal réellement indépendant
  (`fyralabs.com/pgp.asc`) existe mais porte une clé différente (contact
  sécurité organisationnel, pas signature de dépôt). Absence établie
  comme fait, pas laissée en suspens.
  Cause de la récurrence de l'invite établie avec certitude, pas
  supposée : `libdnf5` maintient un trousseau (« pubring ») de
  confiance pour `repo_gpgcheck` propre à chaque répertoire de cache,
  distinct de la base rpm partagée qui gouverne `gpgcheck` — confirmé
  empiriquement (cache système peuplé depuis le 2026-08-04, cache
  utilisateur jamais peuplé) et officiellement documenté (`man
  dnf5.conf`, cité).
  Décision explicite de l'opérateur obtenue avant toute action de
  confiance (le risque était déjà pris, pas nouveau) : correctif
  appliqué — copie du fichier de clé déjà accepté côté système vers le
  trousseau utilisateur, aucun import nouveau, aucune dégradation de
  `gpgcheck` ni de `repo_gpgcheck`. Vérification de cohérence préalable :
  les six paquets Terra installés portent tous la même signature
  (`Key ID 00cdab43de226d6f`) — aucun fait nouveau. Preuve de
  disparition de la friction : la commande exacte qui déclenchait
  systématiquement l'invite depuis le 2026-08-04, rejouée sans garde,
  propre. `terra.repo` inchangé (`sha256sum` identique avant/après).
  Onze dépôts inventoriés dans `docs/repositories.md` : provenance,
  ancrage de confiance, paquets fournis, statut `gpgcheck`/`repo_gpgcheck`
  de chacun — Terra est le seul avec `repo_gpgcheck=1` sur ce poste.
  Équivalents Fedora/COPR pour les quatre paquets fonctionnels de Terra
  recherchés et consignés (aucun officiel ; quelques COPR communautaires
  non vérifiés pour `asusctl`/`supergfxctl`, aucun pour `cardwire`) —
  aucune action entreprise sur cette base, comme demandé.
  Aucun rôle Ansible créé (artefact de cache, pas de configuration
  stable — argumenté). Aucune écriture privilégiée ; trois lectures
  privilégiées énumérées. `terra` toujours activé, six paquets
  toujours présents, aucun nouveau dépôt, aucune modification de
  `sudoers` ni de `/etc/cdi/`, aucun redémarrage.
- **2026-08-06 (série suivante) — déploiement de kitty sur le ScreenPad
  Plus, `roles/desktop/` (D11).** Choix de l'opérateur exécuté (BUR-0) :
  `kitty`, installé depuis `updates` (seul paquet demandé, dépendances
  résolues par `dnf`, jamais `terra`). Configuration sobre déployée
  (`~/.config/kitty/kitty.conf` : police, taille, défilement arrière,
  chaque option motivée dans le fichier lui-même).
  `app_id` Wayland réel mesuré par introspection D-Bus
  (`org.kde.KWin.getWindowInfo`, `resourceClass="kitty"`), pas déduit —
  porté dans `kwinrulesrc` par `wmclass`/`wmclassmatch=1` (`ExactMatch`).
  Mécanisme de placement initialement envisagé (`screen`/`screenrule=
  Force`, index d'écran) **testé et abandonné** : confondu avec
  `org.kde.KWin.activeOutputName`, indépendamment de l'index configuré.
  Remplacé par `position`+`size` (mesurées en direct à chaque
  exécution, jamais figées), en `Apply initially` plutôt que `Force`
  (motif complet, D11 ci-dessus) — vérifié non confondu avec le
  placement par défaut de KWin (lui-même trouvé coïncident avec `DP-3`
  au moment des essais, deuxième confondant identifié et écarté par un
  test à variable de contrôle tenue constante, D11). Marqueur § 1.3 de
  `docs/desktop.md` (BUR-0, stabilité de l'ordre `workspace()->outputs()`)
  requalifié : la question devient sans objet, aucun mécanisme retenu
  ne s'appuie plus sur un index de sortie.
  Rechargement KWin par D-Bus (`org.kde.KWin.reconfigure`, asynchrone —
  un handler `sleep 1` corrige une lecture prématurée trouvée en test),
  jamais de redémarrage de session. Colonnes/lignes mesurées dans la
  fenêtre réellement dimensionnée par la règle : 32 lignes, 274
  colonnes — comparé à l'approximation théorique de BUR-0 pour 12 pt
  (≈37 lignes, ≈267 colonnes), écart cohérent avec la réserve déjà
  posée par BUR-0 sur son propre calcul.
  Autostart XDG déployé (`~/.config/autostart/kitty-screenpad.desktop`,
  aucun chemin propre à cette machine, aucun levier de phase posé par
  réflexe — absence de besoin d'ordre établie par lecture du graphe de
  dépendances systemd en BUR-0). Effet non vérifiable sans déconnecter
  la session (interdit sans demande) — commande de vérification
  préparée pour la prochaine ouverture.
  Deux démonstrations enregistrées avec géométries mesurées (nominal :
  fenêtre de test à `0,1067 2560x734` ; discriminante, sortie active
  tenue à `DP-3` pendant que la règle ciblait `eDP-1` : fenêtre à
  `315,0 1707x1067`, suit la règle pas le défaut coïncident) —
  `docs/desktop.md` § 6.8. Idempotence confirmée (`changed=0` en
  deuxième exécution). Garde sur `desktop_target_output` vérifiée dans
  les deux sens (échec bruyant sur nom de sortie inexistant, succès
  sans dérogation).
  Recherche read-only d'une variante durable au correctif Terra
  (`gpgkey=` déjà local dans `terra.repo`, vérifié sans effet sur
  `repo_gpgcheck` — clés séparées par construction, `man dnf5.conf`) :
  aucune trouvée, confirmé par recherche exhaustive dans `dnf5 --help`
  et `dnf5.conf` plutôt que simplement non cherchée — régression,
  commande de restauration et symptôme déjà consignés en D10,
  complétés dans `docs/repositories.md` § 3.1.
  Une seule action privilégiée (`dnf install -y kitty`, `become: true`)
  — énumérée. `ansible-lint --profile production roles/desktop/` :
  0 défaut, aucun `noqa`. `--syntax-check`/`--check`/exécution réelle/
  deuxième exécution tous rejoués après la correction finale
  (paramétrage de `desktop_target_output`). `kwinrc` jamais touché,
  aucun autre paquet, aucun nouveau dépôt, aucun changement de mode
  d'affichage ni d'échelle, aucune déconnexion, aucun redémarrage.
- **2026-08-06 (série suivante) — deux corrections de revue, puis
  résolution éditeur en lecture seule (`docs/editor.md`, nouveau).**
  Corrections : titre de commentaire périmé dans
  `roles/desktop/defaults/main.yml` (annonçait `Force` alors que le
  code portait déjà `Apply initially` depuis § 6.4) — troisième
  occurrence du même défaut dans ce dépôt, règle ajoutée à `CLAUDE.md`
  § Avant d'agir (un commentaire qui énonce une décision se relit à
  chaque révision de cette décision). Écart « dimensionné » contre
  « maximisé » consigné comme choix assumé, pas corrigé (D11
  ci-dessus, `docs/desktop.md` § 6.10) — sans conséquence tant
  qu'aucun panneau Plasma n'occupe `DP-3`, symptôme nommé pour le cas
  contraire.
  Résolution éditeur, strictement en lecture seule : recensement par
  interrogation des onze dépôts (89062 paquets hors Terra, 2739 sur
  Terra, garde `--assumeno` conservée bien qu'aucune invite ne se
  soit déclenchée — la clé `repo_gpgcheck` posée en D10 tient).
  Candidats GUI (Kate, Geany, Qt Creator, GNOME Text Editor, gedit,
  Zed et Cursor sur Terra) et terminal (Neovim, Vim, Emacs, Helix,
  Micro, Kakoune, joe, nano déjà présent) recensés avec coût mesuré.
  Ni `yaml-language-server` ni `ansible-language-server` empaquetés
  dans aucun des onze dépôts (ce sont des paquets npm en amont ;
  `node`/`npm` absents de ce poste). `ansible-lint` existe en rpm
  (`python3-ansible-lint` 26.4.0) mais installerait un second
  `ansible-lint` divergent du binaire D3a (26.6.0) — même piège
  qu'ANS-1, signalé, rien installé. VSCode/VSCodium confirmés absents
  de toute source déjà configurée sur ce poste (dépôts activés ou
  désactivés, remote Flatpak) — les obtenir exigerait une nouvelle
  source de confiance (D7/D10), non entrepris.
  **Incident de méthode** : `dnf download` exécuté par erreur sur le
  paquet `cursor` (Terra, ~195 MiB × 2 architectures) en tentant
  d'inspecter son lanceur — contredit la consigne de lecture seule.
  Reconnu immédiatement, fichiers supprimés, rien installé ni copié
  ailleurs ; la question que cette commande devait trancher (nativité
  Wayland de `cursor`) reste non déterminée, marquée en conséquence
  plutôt que devinée. Aucune autre commande de ce type dans la série.
  Aucun éditeur recommandé (hors périmètre de la demande) ; questions
  discriminantes formulées à la place, `docs/editor.md` § 1.4.
  Aucune action privilégiée. Aucun paquet installé, aucune extension
  téléchargée, aucun dépôt ajouté, aucune configuration d'éditeur
  créée.
- **2026-08-06 (série suivante) — Helix et Kate, `roles/editor/`
  (D12, D13 — numérotation corrigée, la demande désignait ces deux
  décisions D11/D12, déjà pris par le placement de fenêtre KWin ci-
  dessus, signalé plutôt que résolu en silence).** Deux corrections de
  revue traitées avant ce livrable : commentaire périmé dans
  `roles/desktop/defaults/main.yml` (troisième occurrence du même
  défaut, règle ajoutée à `CLAUDE.md`) et écart « dimensionné » contre
  « maximisé » consigné comme choix assumé — les deux déjà couverts
  par l'entrée précédente.
  Résolution avant installation : Kate absent (`rpm -q kate`), Helix
  absent ; `yamllint` déjà présent dans le venv D3a (1.38.0) — aucun
  rpm `yamllint` installé, même piège qu'ANS-1 évité. Simulation
  `helix`/`kate` (`--assumeno --disablerepo=terra`) : aucune
  correspondance `node`/`npm`.
  Rôle exécuté réellement : 9 paquets, tous `fedora`/`updates`, aucun
  `terra` (`dnf history info`). Helix configuré avec une option motivée
  (numérotation de ligne absolue) ; aucun `languages.toml` — l'absence
  de serveur de langage pour `yaml` déjà documentée par le paquet
  (`hx --health yaml`). Kate configuré exclusivement par clé nommée
  (`kwriteconfig6`) : greffons outils-externes/terminal-intégré activés
  (groupe `[Kate Plugins]`, clé = nom de base du `.so`, sourcé dans
  `katepluginmanager.cpp` tag v26.04.3 et confirmé par lecture directe
  de `~/.config/katerc` après une première ouverture sans configuration
  — le groupe n'existe pas par défaut) ; deux outils externes déployés
  (`ansible-lint-d3a`, `yamllint-d3a`, schéma sourcé dans
  `kateexternaltool.cpp` tag v26.04.3, `arguments=%{Document:FilePath}`
  sourcé dans `katevariableexpansionmanager.cpp` ktexteditor tag
  v6.28.0, `mimetypes=application/yaml` relevé par `xdg-mime` local) —
  confirmé a posteriori par comparaison directe avec les ~17 outils
  que Kate a lui-même écrits au même format après activation du
  greffon. Client LSP non activé (D12, rien à lui connecter).
  Garde D12 démontrée dans les deux sens : nominal (chaque exécution),
  échec forcé (`-e '{"editor_forbidden_binaries": ["sh"]}'` — jamais
  node/npm réellement installés pour le test). `command -v node npm`
  après exécution : rien, code non nul.
  Mode d'affichage de Kate établi par mesure directe du processus réel
  (greffon Qt `libqwayland.so` chargé, `libqxcb.so` absent) : Wayland
  natif, pas déduit des dépendances déclarées.
  `ansible-lint --version` inchangé (`ansible-core:2.20.7`) après
  exécution du rôle — aucun second binaire introduit. Défaut réel
  trouvé et corrigé : codes ANSI de couleur insérés au milieu de la
  sortie d'`ansible-lint --version`, faisant échouer à tort une
  comparaison de sous-chaîne — corrigé par `NO_COLOR=1`.
  Marqueur `@VERIF` sur la nativité Wayland de Cursor (ouvert en
  EDI-0) requalifié : Cursor écarté par D13, la question devient sans
  objet — fermé par requalification, pas par vérification effective.
  Une seule action privilégiée (`dnf install -y helix kate`,
  `become: true`) — énumérée. `ansible-lint --profile production
  roles/editor/` : 0 défaut, aucun `noqa`. `--syntax-check`/`--check`/
  exécution réelle/deuxième exécution (`changed=0`) tous rejoués.
  Aucun paquet hors `helix`/`kate` et leurs dépendances, aucun dépôt
  ajouté, aucun `dnf download`, `kwinrulesrc`/`kwinrc`/`sudoers`/
  `/etc/cdi/`/`gpu_mux_mode` non touchés, aucune déconnexion, aucun
  redémarrage.
- **2026-08-06 (série suivante) — modèle IA local, résolution en
  lecture seule (`docs/local-ai.md`, nouveau).** Dernier volet de la
  demande initiale de l'opérateur. Trois usages (complétion, chat,
  agent) évalués séparément sur l'enveloppe VRAM mesurée, pas les
  16376 MiB nominaux — dimensionnement de modèle dérivé du tableau de
  quantification `llama.cpp` (source externe, pas mémorisé), pas de
  taille de modèle affirmée sans ce calcul.
  RTD3 et VRAM : l'indice déjà observé (`kwin_wayland` 13 MiB
  n'empêche pas la suspension) confirme que RTD3 réagit à un contexte
  CUDA ouvert, pas à une simple allocation — un serveur d'inférence
  qui tourne bloque RTD3 comme n'importe quelle application CUDA,
  sans coût supplémentaire propre à l'inférence. `NVreg_PreserveVideoMemoryAllocations`
  reste sans rapport avec RTD3 (déjà établi dans `docs/dgpu-power.md`)
  — pertinent seulement à une vraie suspension système ; complément
  établi ici : cette machine utilise `s2idle`, pas S3 classique,
  mécanisme S0ix dédié supporté par la plateforme et le GPU mais
  désactivé (`EnableS0ixPowerManagement: 0`) ; unités systemd
  `nvidia-suspend`/`nvidia-resume`/`nvidia-hibernate` déjà activées,
  donc le câblage qu'exigerait `NVreg_PreserveVideoMemoryAllocations=1`
  est déjà en place si cette voie était retenue plus tard.
  Enveloppe VRAM mesurée par sonde isolée (méthode
  `docs/dgpu-power.md`) : 16376 MiB total, 47 MiB utilisés
  (`kwin_wayland` 13 MiB), 15926 MiB libres rapportés — retenue comme
  enveloppe de dimensionnement (§1), pas le nominal.
  Péremption CDI : vérification existante (`verify-cdi-spec`) rejouée
  en lecture seule, à jour (610.43.03 des deux côtés, 50 chemins
  présents) ; rien ne la déclenche aujourd'hui (aucune minuterie ni
  service ne l'invoque, vérifié) — deux leviers nommés pour un
  livrable futur (`ExecStartPre=` sur le service d'inférence, ou
  minuterie systemd dédiée), non câblés ici.
  Recherche dans les onze dépôts : `ollama`, `llama-cpp`, `whisper-cpp`
  et `python3-torch` tous empaquetés par Fedora, **tous liés en dur à
  ROCm/HIP (AMD), aucun lien CUDA** — établi par lecture des
  dépendances déclarées de chacun, schéma cohérent sur les quatre
  paquets, pas un hasard isolé. Conséquence : aucune voie native depuis
  les onze dépôts n'atteint la RTX 4090 ; écartée, pas seulement non
  recommandée. Registres de conteneurs externes identifiés par
  inspection de métadonnées (`skopeo inspect`, aucune couche
  téléchargée, vérifié par `podman images` inchangé) :
  `ollama/ollama` (CUDA par défaut, tag `:rocm` séparé), `ggml-org/llama.cpp`
  (tags `-cuda` amont), `vllm/vllm-openai` (tags `cuXXX`) — nouvelle
  surface d'approvisionnement de même nature que D7/D10, nommée, pas
  résolue.
  Recommandation unique : Ollama conteneurisé, image CUDA officielle,
  via le mécanisme CDI déjà prouvé (`docs/gpu-containers.md`) — seule
  voie qui atteint la RTX 4090 parmi celles disponibles, s'appuie sur
  du travail déjà validé (CDI rootless, non-réveil au lancement de
  conteneur), reconstructible par empreinte de tag (D1), câblage de
  `verify-cdi-spec` en `ExecStartPre=` nommé comme chantier restant.
  Aucun modèle choisi (consigne explicite) — cinq questions
  discriminantes formulées à la place.
  Incident reconnu et corrigé dans la série : `sudo btrfs filesystem
  usage /` invoqué par réflexe sans nécessité — la version sans
  privilège donne les mêmes chiffres retenus ; une seule élévation,
  reconnue, non répétée. Aucun autre paquet installé, aucun modèle ni
  image téléchargé, aucun conteneur lancé, aucun dépôt ajouté, aucun
  paramètre de pilote modifié.
- **2026-08-07 — infrastructure d'inférence Ollama, `roles/local_ai/`
  (D14-D17, numéros vérifiés libres avant attribution, aucune
  collision cette fois).** Correction apportée à IA-0 en préalable :
  la valeur brute par défaut de `NVreg_PreserveVideoMemoryAllocations`
  est `2`, pas `0` supposé implicitement — lue directement avant toute
  écriture, signification de `2` non documentée dans les sources
  consultées (page HTML et README.txt complet du pilote), marquée
  `@VERIF`. Paramètre pris en compte au prochain chargement du module
  (usage actif, compteur 148) — en pratique un redémarrage, non
  déclenché, commande de vérification préparée.
  Résolution avant écriture : Quadlet Podman retenu (générateur natif
  de Podman 5.8.4, vérifié présent) plutôt qu'une unité écrite à la
  main ; aucun sous-volume btrfs dédié pour les modèles (aucun outil
  de snapshot installé, abstention justifiée) — volume Podman nommé
  ordinaire sur le stockage rootless standard, déjà sur le btrfs D1.
  Trois gardes préalables (CDI non périmée, Podman rootless,
  `nvidia_uvm` chargé), échec bruyant, réutilisent `verify-cdi-spec`
  sans le réimplémenter. Paramètre pilote déployé par écrasement propre
  (`/etc/modprobe.d/local-ai-nvidia-power-management.conf`), fichier
  fourni par `xorg-x11-drv-nvidia-power` vérifié intact avant/après
  (`rpm -Vf`, somme de contrôle identique). Image Ollama épinglée par
  empreinte de l'index multi-architecture
  (`sha256:b88c73ace3e115f8ec53dc8761ae1c0aabfa675406e3681786b98757ce050f42`,
  `docker.io/ollama/ollama:0.32.6` au moment de la lecture), jamais une
  étiquette mobile. Service revérifiant lui-même la spécification CDI
  à chaque démarrage (`ExecStartPre=`), réponse directe au point 2.4
  d'IA-0.
  Rôle réussi dès la première exécution réelle : API locale confirmée
  en écoute sur `127.0.0.1:11434` uniquement (`ss`, aucune interface
  externe), liste des modèles vide confirmée, GPU visible depuis
  l'intérieur du conteneur confirmé par deux sources indépendantes
  (nœuds `/dev/nvidia*` + `nvidia-smi -L` via `podman exec`, et le
  journal interne d'Ollama lui-même : `compute=8.9`, `driver=13.3`,
  RTX 4090 détectée). Idempotence confirmée (`changed=0` en deuxième
  exécution). Deux démonstrations d'échec forcé réussies (spécification
  CDI invalide substituée par `-e`, module noyau requis substitué par
  `-e`), toutes deux `changed=0`, état nominal restauré après chacune.
  SELinux `Enforcing` inchangé avant/après, aucun refus AVC (vérifié
  par deux méthodes indépendantes, l'une sans privilège via
  `journalctl -k`, l'autre privilégiée via `ausearch` en corroboration,
  disclosed).
  Consommation au repos mesurée avec le service démarré mais **sans
  modèle chargé** (méthode d'isolement `docs/dgpu-power.md`) : RTD3
  s'engage normalement, `runtime_active_time` inchangé sur 280 s —
  référence chiffrée pour IA-2, la contrepartie de D15 (RTD3
  neutralisée) ne commence qu'au premier modèle réellement chargé.
  Découverte non anticipée, signalée, non corrigée (hors périmètre de
  ce livrable) : l'image officielle Ollama contacte `ollama.com` de
  son propre chef pour une liste de recommandations de modèles, et
  expose une fonctionnalité « cloud » activée par défaut
  (`OLLAMA_NO_CLOUD:false`) — nommé pour un livrable ultérieur.
  Une seule action privilégiée du rôle (écrasement modprobe.d,
  `become: true`) — énumérée, plus une lecture privilégiée hors rôle
  (`ausearch`) pour la validation SELinux. `ansible-lint --profile
  production roles/local_ai/` : 0 défaut, deux `noqa` justifiés.
  `--syntax-check`/`--check` (implicite via gardes)/exécution réelle/
  deuxième exécution tous rejoués. Aucun modèle téléchargé sous aucune
  forme (seule l'image du serveur, infrastructure, a été tirée,
  distinction explicite) ; `/usr/lib/modprobe.d/` intact (somme de
  contrôle identique) ; aucun dépôt ajouté ; `sudoers`/`gpu_mux_mode`/
  `dgpu_disable`/`kwinrulesrc` non touchés ; aucune modification
  SELinux ; aucune déconnexion ; aucun redémarrage.
- **2026-08-07 — confinement réseau du service d'inférence, D18,
  `roles/local_ai/` (IA-2).** Redémarrage du 2026-08-07 confirmé,
  reconfirmé directement dans cette session (pas seulement repris du
  rapport de l'opérateur) : `PreserveVideoMemoryAllocations: 1`,
  `ollama.service`/`ollama-network.service`/`ollama-models-volume.service`
  chargés par Quadlet seul, `runtime_status=suspended`, fichier
  fournisseur `/usr/lib/modprobe.d/nvidia-power-management.conf`
  identique par somme de contrôle à IA-1.
  Deux corrections portées (`docs/local-ai.md` § 8.0) : l'amalgame
  RTD3/suspension système dans le point ouvert `NVreg_PreserveVideoMemoryAllocations`
  ci-dessus, marqué `[CORRIGÉ]`/`ÉTAIT`, pas effacé ; la conclusion non
  sourcée « les allocations VRAM ne survivent pas à la veille »
  corrigée sans rouvrir la question de fond sur la valeur `2` non
  documentée (marqueur déjà ouvert en IA-1, toujours ouvert).
  Variables de sortie réseau établies par trois lectures locales, sans
  présumer de nom (`ollama serve --help`, journal « server config » du
  service réel, extraction de chaînes du binaire réellement déployé) :
  `OLLAMA_NO_CLOUD` (documentée, défaut `false`), `OLLAMA_REMOTES`
  (non documentée par `--help`, défaut `[ollama.com]`), et des
  variables non observées en fonctionnement (`OLLAMA_CLOUD_BASE_URL`,
  `OLLAMA_API_KEY`, `OLLAMA_AGENT_DISABLE_WEBSEARCH`/`_SHELL`) —
  marqueur ouvert sur leur sémantique exacte.
  Confinement retenu (D18) : réseau Podman dédié `--internal` (Quadlet
  `.network`) en garde structurelle, vérifiable depuis l'hôte, plus
  `OLLAMA_NO_CLOUD=1` en défense en profondeur, vérifiée efficace par
  mesure (aucune tentative, pas seulement un échec propre) avant
  d'être retenue. `--network=none` testé et écarté : casse aussi
  `PublishPort=`, pas seulement `ollama pull` (« Port mappings have
  been discarded... »). Preuve : fichier de cache des recommandations
  (682 octets, écrit à chaque démarrage avant IA-2) plus jamais réécrit
  après deux redémarrages du service depuis IA-2 ; démonstration
  inverse réalisée en réactivant délibérément `OLLAMA_NO_CLOUD=0`
  réseau inchangé — la tentative réapparaît dans le journal du serveur
  (`dial tcp: lookup ollama.com ... no such host`) et reste bloquée
  avant toute sortie réelle, confirmant que la mesure est sensible.
  Deux nouvelles gardes a posteriori dans le rôle (réseau interne
  confirmé, cloud désactivé confirmé dans le propre journal du
  serveur — effet constaté, pas la variable posée), démontrées dans
  les deux sens, restaurées à l'état nominal après chaque essai forcé.
  Défaut trouvé et corrigé en écrivant ces gardes : `state: started`
  d'Ansible ne redémarre pas un service déjà actif quand la définition
  Quadlet change — remplacé par un redémarrage conditionnel aux unités
  effectivement modifiées, idempotence revérifiée après correction.
  Faisabilité de la complétion dans Helix/Kate établie en lecture
  seule (`docs/local-ai.md` § 8.6) : aucun greffon Helix, `llm-ls`
  (Apache-2.0, hors npm) inutilisable par un client LSP générique
  (méthode JSON-RPC propriétaire, lu directement dans son code amont)
  — D15 signalée comme nécessitant une révision de l'opérateur, pas
  tranchée ici. Candidats de modèles recensés en lecture seule
  (`docs/local-ai.md` § 8.7), licences et coût de cache d'attention
  chiffrés par lecture des `config.json` amont de chaque modèle —
  aucun modèle choisi.
  Registre de conteneurs Docker Hub nommé dans `docs/repositories.md`
  § 6, au même titre que Terra et le COPR (ancrage de confiance :
  empreinte d'image épinglée, pas une clé de signature).
  Une seule action privilégiée nouvelle hors rôle (`ausearch`,
  corroboration SELinux) ; la garde D16 du rôle (`become: true`)
  inchangée depuis IA-1, rejouée à chaque exécution de cette série
  sans jamais modifier son contenu. `ansible-lint --profile production
  roles/local_ai/` : 0 défaut. `--syntax-check`/`--check`/exécution
  réelle/deuxième exécution/quatre démonstrations d'échec forcé (dont
  une inverse, `changed=2`, restaurée) tous rejoués. `getenforce`
  `Enforcing` inchangé, aucun refus AVC (deux méthodes). Aucun modèle
  téléchargé sous aucune forme ; `podman images` inchangé ; aucun
  paquet installé ; aucun dépôt ajouté (le réseau Podman dédié n'est
  pas un dépôt) ; `sudoers`/`gpu_mux_mode`/`dgpu_disable`/`kwinrulesrc`
  non touchés ; SELinux jamais modifié ; aucune déconnexion ; aucun
  redémarrage machine.
- **2026-08-07 — résolution en lecture seule de la complétion locale,
  `docs/completion.md`.** Question posée : le blocage de la complétion
  automatique dans Helix/Kate, constaté en clôture d'IA-2, est-il côté
  éditeur ou côté serveur ? Établi par lecture directe (changelog
  packagé localement pour Helix, extraction de chaînes du binaire
  installé pour le greffon LSP de Kate) : **ni l'un ni l'autre n'a de
  blocage structurel** — les deux sont des clients LSP standard,
  capables de `textDocument/completion` à la frappe ; le blocage
  identifié en IA-2 (`llm-ls`) était propre à ce serveur précis
  (méthode JSON-RPC propriétaire), pas généralisable. Recherche élargie
  à deux autres serveurs : `tabby-agent` (Node.js requis, protocole
  standard + extensions optionnelles) et **`lsp-ai`** (MIT, Rust,
  binaire précompilé hors npm, protocole standard vérifié de bout en
  bout dans son code, compatibilité Helix documentée par le projet
  amont) — ce dernier résout la complétion **sans rouvrir D12 ni
  réviser D13**, une troisième voie non anticipée par la demande.
  Coût propre à cette voie, nommé symétriquement à celui de npm :
  aucune somme de contrôle publiée pour le binaire, dernière
  publication ~23 mois avant cette date, développement en pause
  déclaré par le mainteneur, compatibilité Kate non testée par
  personne (**testée depuis, 2026-08-08, KAT-1, résultat positif** —
  ci-dessous) — deux marqueurs de vérification ouverts sur ces points.
  D15 requalifiée en conséquence (ci-dessus, § Décisions) — pas
  tranchée, l'opérateur décide via les questions de `docs/completion.md`.
  Coût de rouvrir D12 chiffré pour mémoire (nodejs22 : ~87,5 Mio
  minimal, 169 Mio avec dépendances faibles, `dnf install --assumeno`,
  sans privilège) — aucune voie retenue ne l'exige à ce stade. Motifs
  D13 (magasin d'extensions, télémétrie, dépendance Terra pour
  Zed/Cursor) revérifiés, toujours valides.
  **Aucune action privilégiée dans ce livrable** — une élévation
  (`sudo dnf install --assumeno nodejs22`) exécutée par réflexe puis
  reconnue superflue (la même commande fonctionne sans privilège),
  même défaut de méthode que celui déjà reconnu en IA-0, non répété.
  Aucun paquet installé, ni `node` ni `npm` ; aucun modèle ni image
  téléchargé ; aucun dépôt ajouté ; aucune commande `dnf` n'a touché
  `terra` ; aucune configuration d'éditeur ni de service modifiée ;
  aucun fichier écrit hors de ce dépôt.
- **2026-08-07 — `lsp-ai` compilé depuis les sources, intégré à Helix,
  D19/D20, `roles/completion/` (CMP-1).** Deux décisions de
  l'opérateur (ci-dessus, § Décisions) : complétion par `lsp-ai`
  compilé depuis les sources plutôt que son binaire précompilé
  (D19, quatrième surface d'approvisionnement — `crates.io`,
  `docs/repositories.md` § 7) ; modèle de complétion à la demande,
  pas résident (D20, requalifie D15 sans l'effacer).
  Résolution avant compilation : `rust`/`cargo` disponibles
  `fedora`/`updates` uniquement (1.97.1), aucun `rust-version` déclaré
  par `lsp-ai` ; `Cargo.lock` présent (104 548 octets) à l'empreinte de
  commit retenue (`1e910a8cf0048406eb227bf2064743010a9ff3a9`,
  2025-01-07 — pas l'étiquette `v0.7.1`, plus ancienne, une étiquette
  pouvant être déplacée).
  Deux obstacles réels rencontrés en compilant, signalés à l'opérateur
  avant correction, aucun anticipé par la seule lecture des
  manifestes : une dépendance git (`hf-hub`) déclarée par version sans
  `rev` fixe, dont le dépôt amont a retiré les étiquettes
  correspondantes, cassant `--locked` malgré `Cargo.lock` présent —
  corrigé par `cargo update -p hf-hub --precise <même commit déjà
  verrouillé>`, qui ne change pas cette entrée mais corrige un
  décalage sans rapport déjà présent dans le `Cargo.lock` amont ; le
  GCC système (16.1.1) compile en C23 par défaut, incompatible avec le
  C vendoré K&R de la bibliothèque `oniguruma` (tirée par
  `tokenizers`) — corrigé par `CFLAGS=-std=gnu17` pour la seule
  compilation C déclenchée par ce rôle. Cinq modules Perl absents
  également installés (dépendances de la configuration d'OpenSSL
  vendoré par `openssl-sys`), tous `fedora`/`updates`.
  Empreinte du binaire produit identique sur deux compilations propres
  indépendantes : `7bc6e76e296861d958ab369131de433457f6226b994487f9b418ae55ea8f9159`
  (`lsp-ai 0.7.1`). Idempotence non triviale à obtenir (le clonage et
  la correction du lockfile ne s'exécutent qu'au premier passage, pas
  reforcés ensuite) — confirmée, `changed=0` en deuxième exécution.
  Preuve de la poignée de main LSP par lecture du journal Helix (`hx
  -vv`, piloté sans capture visuelle) : réponse `initialize` du
  serveur portant `completionProvider`, contrastée avec une
  configuration neutralisée (binaire introuvable) où aucune capacité
  n'est annoncée — les deux journaux visiblement différents. Quatre
  démonstrations d'échec forcé supplémentaires sur les gardes du rôle,
  toutes `changed=0`. Kate non configuré, compatibilité toujours non
  testée par personne — point ouvert, pas affirmé (**fermé le
  2026-08-08, KAT-1, journal ci-dessous**).
  Nouvelle règle `CLAUDE.md` (§ Avant d'agir) : toute élévation de
  privilège précédée d'une tentative sans privilège — motivée par deux
  occurrences passées d'élévation réflexe (IA-0, CMP-0). **Incident
  reconnu dans ce même livrable** : une élévation
  (`sudo dnf install perl-IPC-Cmd`) exécutée sans tentative sans
  privilège préalable, quelques instants après l'écriture de cette
  règle — signalé explicitement, pas dissimulé, non répété pour les
  trois modules Perl suivants.
  Aucun modèle téléchargé sous aucune forme ; aucun binaire `lsp-ai`
  précompilé récupéré ; `command -v node npm` toujours vide ; aucun
  nouveau dépôt `dnf` (`crates.io`/le dépôt git `lsp-ai` n'en sont
  pas) ; `getenforce` inchangé (`Enforcing`) ; aucun redémarrage.
- **2026-08-07 — récupération des modèles, mesure de l'enveloppe
  réelle, D21, `roles/local_ai/` (IA-3).** Deux modèles nommés (D21,
  ci-dessus) récupérés par un conteneur de récupération éphémère,
  jamais le réseau du service — isolation prouvée par inspection à
  quatre moments distincts, identique deux fois. Intégrité testée par
  corruption délibérée d'un blob déjà tiré : `ollama pull` vérifie le
  contenu reçu sur le réseau mais ne détecte pas un blob local corrompu
  sous son nom attendu (écart nommé, `docs/repositories.md` § 8).
  Espace disque : Δ≈11 GiB (`btrfs filesystem usage /`, sans
  privilège), cohérent avec les tailles de manifeste.
  Coexistence mesurée, pas supposée : chat seul à 32 K = 12 294 Mio
  VRAM (contre ≈12,5 Gio estimés en IA-2 — écart expliqué : confusion
  Gio/GB sur les poids, la formule de cache KV était juste à <1 %) ;
  complétion seule à son contexte réel (2000) = 4 690 Mio ; les deux
  ensemble : Ollama décharge systématiquement l'un pour charger
  l'autre, testé dans les deux sens — la coexistence ne tient pas,
  confirmé. Temps de chargement froid/chaud, latence premier jeton
  (0,176 s, modèle résident) et débit (65 jetons/s, chat) mesurés.
  RTD3 confirmé bloqué en continu par un contexte CUDA ouvert (lecture
  passive 60 s, aucune sonde) ; consommation au repos modèle chargé :
  3,49 W (P8).
  Trois leviers chiffrés, aucun recommandé : `OLLAMA_KV_CACHE_TYPE=q8_0`
  (-2,29 Gio, qualité non chiffrée), contexte réduit à 16 K (-2,38 Gio),
  bascule séquentielle (3,4-4,1 s par changement).
  Test bout-en-bout `lsp-ai`/Helix : poignée de main LSP et requêtes de
  complétion automatiques toujours fonctionnelles ; échec de la
  complétion elle-même attribué avec certitude à un nom de modèle
  placeholder de CMP-1 (`roles/completion/`, hors périmètre de ce
  livrable) — ni `lsp-ai` ni RTD3/rechargement en cause.
  Défaut trouvé et corrigé en écrivant ce livrable : la garde
  d'annonce cloud (IA-2) échouait sur un journal de service contenant,
  après usage réel, des séquences d'octets non-UTF8 valides (vidage de
  vocabulaire GGUF par llama.cpp) — corrigé par un filtre `iconv`, les
  deux démonstrations d'échec forcé de cette garde rejouées après
  correction.
  **Aucune action privilégiée dans ce livrable** au-delà de la garde
  D16 pré-existante, inchangée, rejouée sans modification de contenu —
  note honnête : cette tâche pré-existante ne suit pas encore la
  règle 0.3 (CMP-1), signalé plutôt que laissé paraître conforme.
  Aucun autre modèle que les deux nommés ; jamais le conteneur de
  service connecté à un réseau externe ; `getenforce` inchangé
  (`Enforcing`), aucun refus AVC ; aucun `node`/`npm` ; aucun nouveau
  dépôt système ; aucun redémarrage.
- **2026-08-08 — boucle la chaîne d'inférence : complétion débloquée,
  intégrité des poids, déclencheurs CDI, D22 (IA-4).** Constat
  préalable : le paramètre pilote D16 est passé à `1` entre-temps (un
  redémarrage a eu lieu hors de cette session — constaté par lecture,
  `/proc/driver/nvidia/params`, pas déclenché ici).
  **Complétion débloquée** : le nom de modèle placeholder de CMP-1
  (`roles/completion/`), cause exacte diagnostiquée en IA-3, remplacé
  par le modèle réel (D21), en variable, avec une garde dédiée
  (existence côté service, avant l'écriture de la configuration) —
  démontrée dans les deux sens.
  **Complétion réelle testée bout-en-bout** (`docs/completion.md` § 8) :
  YAML et Python, complétions syntaxiquement et sémantiquement
  cohérentes reçues. Latence en régime établi (modèle déjà résident) :
  0,46-0,48 s. Bascule chat → complétion (D22, conditions réelles,
  contexte 32 K) : 3,727 s, dans la fourchette annoncée. Verdict :
  utilisable au quotidien. Écart initial avec le fait daté d'IA-3
  (déclenchement automatique à la frappe) reproduit et expliqué : une
  frappe isolée unique ne déclenche pas fiablement, une frappe réelle
  multiple si — pas de contradiction retenue, résolu avant de conclure.
  **Intégrité des poids au repos** (`roles/local_ai/`) : script autonome
  déployé (domaine utilisateur, lecture seule stricte), recalcule les
  empreintes des blobs contre leurs manifestes. Démontré dans les deux
  sens : corruption délibérée d'un octet détectée et nommée (blob et
  manifeste identifiés), blob restauré par suppression puis nouveau
  tirage (la seule voie qui force `ollama pull` à revérifier, établi en
  IA-3), script de nouveau `OK` après restauration.
  **Déclencheurs CDI** (`roles/local_ai/`) : le mécanisme
  `verify-cdi-spec` (IA-1) restait sans rien qui le déclenche hors du
  (re)démarrage du service. Deuxième levier ajouté : minuterie systemd
  --user horaire, indépendante, chaînée (`OnFailure=`) vers un service
  de notification visible (`notify-send`, déjà présent). Démontré dans
  les deux sens : spécification valide → succès silencieux ;
  spécification simulée périmée (unité patchée temporairement, jamais
  le gabarit) → refus avec le message attendu, notification déclenchée
  et confirmée (`Triggering OnFailure= dependencies`, service de
  notification `status=0/SUCCESS` sur le bus D-Bus réel de la session) ;
  unité restaurée à l'identique après coup, rejeu confirmé `changed=0`.
  **`cargo vendor` évalué, écarté en bloc** (`docs/completion.md` § 7.2.4) :
  chiffré réellement — 710 Mio bruts, 71 Mio compressés, 422 crates —
  disproportionné (~25× la taille actuelle du `.git` de ce dépôt) et
  mal ciblé (la quasi-totalité des crates vendorées viennent de
  crates.io, dont le risque de disponibilité diffère de celui des deux
  dépendances git réellement concernées). Mitigation ciblée retenue à
  la place : archive de `hf-hub` à son empreinte épinglée seule
  (88 Kio compressés), hors dépôt, empreinte consignée, vérifiée
  fonctionnelle (extraction + `git log` concordant) ; `lsp-ai`
  lui-même volontairement sans archive équivalente — son clonage à
  l'empreinte épinglée, rejoué à chaque exécution du rôle, sert déjà de
  canari vivant.
  **D22 consignée** (ci-dessus) : chargement séquentiel, seul levier
  mesuré et non régressif parmi les trois nommés par D21.
  **Deux corrections d'outillage** (`CLAUDE.md`) : la preuve
  d'additivité `git show HEAD --numstat` remplacée par
  `git diff --numstat HEAD^ HEAD | awk '$2>$1 {print "RÉDUIT: " $3}'`
  (la première mélangeait message de commit et données) ; le tableau
  d'énumération des actions privilégiées porte désormais une colonne
  « tentative sans privilège : résultat », appliquée dans ce livrable.
  Aucune action privilégiée dans ce livrable au-delà du `become: true`
  pré-existant (D16, inchangé, `rpm -Vf` identique avant/après) ; aucun
  modèle supplémentaire téléchargé ; `/usr/lib/modprobe.d/` intact ;
  `getenforce` inchangé (`Enforcing`), aucun refus AVC (deux méthodes,
  une privilégiée déjà précédentée, une non) ; aucun redémarrage
  déclenché par cette session.
- **2026-08-08 — corrections courtes issues de la revue globale (COR-1),
  `CLAUDE.md`/`roles/gpu_mux/`/`roles/completion/`/`roles/gpu_cdi/README.md`.**
  Six constats de `docs/review-2026-08.md` traités, marqués sur place
  sans suppression de texte : § 2.1 (`CLAUDE.md` § Chaîne Ansible
  réécrite conformément à D3a, renvoi corrigé, plus une règle imposant
  de relire une règle quand la décision qu'elle cite est amendée) ;
  § 2.2 (`roles/gpu_cdi/README.md`, affirmation périmée sur l'absence de
  `NOPASSWD`, fausse depuis D9, corrigée) ; § 2.3 (exemple `dnf repolist
  enabled` de `CLAUDE.md`, silencieusement vide sur dnf5, remplacé par
  `dnf repolist --enabled`) ; § 3.1 (quatrième mode d'échec ajouté à
  `CLAUDE.md` : la prémisse de garde périmée par l'évolution normale du
  système surveillé, distincte des trois déjà recensés) ; § 4.1 (garde
  ajoutée à `roles/gpu_mux/`, vérifiant l'état réel de `sshd`/
  `authorized_keys`/`firewalld` au moment de l'exécution, jamais le fait
  que `roles/recovery/` ait tourné un jour — échec bruyant, aucune
  correction automatique) ; § 4.2 (garde ajoutée à `roles/completion/`,
  vérifiant que Helix est installé avant d'écrire sa configuration).
  **Recherche systématique du § 2.2, résultat consigné même négatif** :
  aucune autre occurrence périmée de `NOPASSWD`/`pending_reboot`/
  `Force`/`EE-first` trouvée. **Une trouvée pour `npm`, non anticipée** :
  `roles/editor/tasks/main.yml`, `defaults/main.yml`, `meta/main.yml` et
  `editor.yml` citent encore les numéros pré-renumérotation d'EDI-1
  (« D11 » pour npm fermé, en réalité D12 ; « D12 » pour le choix
  Helix/Kate, en réalité D13) — `docs/editor.md` et
  `roles/editor/README.md`, eux, sont corrects. **Non corrigée ici**
  (`roles/editor/` hors du périmètre strict de ce livrable) — signalée
  dans `docs/review-2026-08.md` § Suivi et dans `CLAUDE.md` pour le
  prochain livrable de corrections.
  **`gpu_mux` non exécuté réellement**, conformément à la consigne — ses
  deux nouvelles gardes démontrées uniquement en `--check` (nominal, et
  trois échecs forcés : `sshd` inactif, `authorized_keys` vide, firewalld
  refusant `ssh`), `gpu_mux_mode` inchangé (`current_value=1`, relu, pas
  écrit). `roles/completion/` exécuté réellement (`changed=0`, état déjà
  nominal), garde Helix démontrée dans les deux sens.
  Une action privilégiée, disclosed : lecture de `/etc/sudoers`
  (`sudo -n sed -n '108,112p'`) pour confirmer la règle `NOPASSWD`
  inchangée — aucune alternative non privilégiée n'existe pour ce
  fichier (mode `0440`), même limite déjà rencontrée pour `ausearch`
  (IA-1/IA-4). Aucun paquet installé (`dnf history list`, dernière
  transaction : 2026-08-07) ; `terra.repo`/`/etc/cdi/nvidia.yaml`
  inchangés (dates de modification vérifiées) ; `sudoers` intact ;
  `getenforce` inchangé (`Enforcing`) ; aucun rôle `bootstrap`, aucun
  playbook d'orchestration écrit ; aucun redémarrage.
- **2026-08-08 — rôle `bootstrap` (D6/D9/D10 reconstructibles) et
  orchestration complète, `roles/bootstrap/` (nouveau), `site.yml`
  (nouveau), `docs/orchestration.md` (nouveau) (COR-2).**
  **§ 1 — un seul redémarrage, pas deux, établi par lecture** :
  `asus-shutdown.service` (D2ter) est ordonné `Before=shutdown.target
  reboot.target halt.target` (déjà établi, § GPU § Services) — il
  s'applique pour un `reboot` comme pour un `poweroff` ; le paramètre
  pilote D16 prend effet « au prochain chargement du module nvidia.ko »
  (déjà établi, `docs/local-ai.md` § 7.1), que tout redémarrage complet
  satisfait. Aucune interaction connue entre les deux mécanismes.
  **Jamais vérifié en combiné** sur ce poste — appliqués historiquement
  par deux redémarrages séparés à des dates différentes
  (2026-08-05/2026-08-07) — écart nommé, `docs/orchestration.md` § 2.
  `site.yml` vérifie les deux faits séparément après le point d'arrêt,
  plutôt que de supposer que l'un implique l'autre.
  **`roles/bootstrap/`** : D6 (`dnf swap`, idempotent, commande
  d'origine reproduite), D9 (`NOPASSWD` déplacée vers
  `/etc/sudoers.d/`, jamais `/etc/sudoers` — validée par `visudo -cf`
  avant toute activation, tag `bootstrap-sudoers` **jamais joué dans
  cette série**, conformément à la demande), D10 (empreinte
  `AE09157A4DE88B497EA1D5D300CDAB43DE226D6F` téléchargée et vérifiée
  par inspection à sec — **aucun import, aucun déploiement avant
  confirmation** — avant tout déploiement de fichier, jamais
  `--nogpgcheck`). Démontrée dans les deux sens (empreinte correcte →
  passe ; empreinte simulée différente → échec avant tout
  déploiement, fichiers réels inchangés, vérifié par somme de
  contrôle). Exécuté réellement (D6+D10, D9 exclue par tag) : première
  exécution `changed=5` (téléchargement/extraction/nettoyage
  intrinsèquement toujours « changed », plus l'ajout d'un en-tête de
  documentation à `terra.repo` — seule différence fonctionnelle :
  aucune, `gpgcheck=1`/`repo_gpgcheck=1`/URLs/clé référencée
  identiques, vérifié) ; seconde exécution : les trois tâches qui
  touchent l'état système persistant (clé, dépôt, paquets) toutes `ok`
  — idempotence confirmée pour ce qui compte, les tâches de
  préparation en répertoire de travail temporaire restant, par
  conception, toujours « changed ».
  **`site.yml`** : ordonne bootstrap → recovery → gpu_cdi → gpu_mux →
  local_ai → editor → completion, un seul point d'arrêt post-tâches
  (jamais un redémarrage déclenché), vérifie `pending_reboot` ET la
  valeur active du paramètre pilote séparément. `--check` complet :
  franchit correctement le point d'arrêt sans s'arrêter (les deux faits
  déjà satisfaits sur ce poste, hérité de redémarrages antérieurs) —
  testé dans son état positif réel, pas dans son état négatif (aucun
  moyen de le provoquer légitimement sans redémarrer ensuite, interdit
  ici).
  **Défaut trouvé en testant `site.yml`, non corrigé (hors périmètre
  strict)** : `roles/local_ai/defaults/main.yml`
  (`local_ai_gpu_cdi_playbook`) résout un chemin relatif à
  `playbook_dir`, qui vaut la racine du dépôt quand ce rôle est inclus
  depuis `site.yml` — pas `roles/local_ai/` comme quand il est appelé
  seul. Contourné pour tester le reste de la séquence
  (`--skip-tags local_ai`), pas corrigé — signalé,
  `docs/orchestration.md` § 5, `docs/review-2026-08.md` § Suivi —
  COR-2.
  **`roles/editor/` corrigé** (`tasks/main.yml`, `defaults/main.yml`,
  `meta/main.yml`, `editor.yml`) : numéros de décision périmés
  (D11→D12 pour npm, D12→D13 pour le choix d'éditeur), trouvés par la
  recherche systématique de COR-1. Diff vérifié : uniquement des noms
  de tâches, messages et commentaires — aucune ligne `that:`/`when:`/
  paramètre de module modifiée.
  **Actions privilégiées** : écritures D10 (clé, dépôt, paquets,
  `become: true`, tentative sans privilège non pertinente — ces
  chemins système n'ont structurellement aucune voie non privilégiée,
  toute la préparation en amont — téléchargement, extraction,
  vérification d'empreinte — se fait sans aucun privilège) ; lecture
  D9 (`stat` sur `/etc/sudoers.d/`, `become: true` — répertoire
  `0750 root:root`, tentative sans privilège : `Permission denied`) ;
  deux lectures manuelles de validation
  (`sudo -n sed -n '108,112p' /etc/sudoers`, tentative sans privilège :
  `Permission denied` ; `sudo -n test -e
  /etc/sudoers.d/99-wheel-nopasswd`, tentative sans privilège :
  résultat trompeur — `test -e` sans privilège renvoie faux aussi bien
  si le fichier est absent que si le répertoire parent est
  impénétrable, 0750 root:root ici — non concluant, pas une
  confirmation d'absence malgré l'apparence). Aucun `setenforce`,
  `getenforce` inchangé (`Enforcing`) ; aucun modèle téléchargé ;
  `gpu_mux_mode` inchangé (`current_value=1`, jamais réécrit) ;
  `/etc/sudoers` intact (règle D9 toujours en dur, tag
  `bootstrap-sudoers` jamais joué) ; `/etc/sudoers.d/99-wheel-nopasswd`
  confirmé absent (lecture privilégiée) ; aucun redémarrage.
- **2026-08-08 — chemin indépendant du contexte d'exécution pour la
  garde CDI de `roles/local_ai/` (ORC-2).** Corrige le défaut trouvé
  en testant `site.yml` (journal ORC-1 ci-dessus) : quatrième mode
  d'échec de `CLAUDE.md`, cette fois découvert dans la série même où
  il est apparu.
  **§ 1, établi par lecture du code source d'Ansible avant de
  corriger** (`ansible-core 2.20.7`, celui qui exécute réellement les
  rôles de ce dépôt, D3a —
  `/usr/lib/python3.14/site-packages/ansible/vars/manager.py`) :
  `playbook_dir` (`self._loader.get_basedir()`) suit le **playbook de
  premier niveau** en cours d'exécution, change selon qui inclut le
  rôle ; `role_path` (`task._role._role_path`) suit le **rôle
  propriétaire de la tâche courante**, fixe quel que soit l'appelant —
  déjà le patron établi dans ce dépôt pour ce besoin
  (`roles/gpu_mux/defaults/main.yml`, `gpu_mux_trace_dir`), repris ici.
  **Recherche systématique** sur toute construction de chemin
  dépendant implicitement du point d'entrée : `playbook_dir` trouvé
  dans neuf fichiers, huit sont le patron `role: "{{ playbook_dir }}"`
  des playbooks-wrapper autonomes de chaque rôle (sûr par construction
  — ce sont eux le point d'entrée dans leur usage normal) ; **une
  seule occurrence problématique**, celle déjà connue. Décision
  consignée dans `CLAUDE.md` § Sourcing des faits : une seule
  occurrence ne justifie pas encore une colonne de validation
  obligatoire (contrairement à `NOPASSWD`/`pending_reboot`, quatre
  occurrences, ou aux décisions renumérotées de `roles/editor/`) —
  mais tout nouveau chemin inter-rôles devrait être exercé depuis
  `site.yml` avant d'être considéré validé.
  **§ 2-3** : `local_ai_gpu_cdi_playbook` corrigé
  (`role_path + '/../gpu_cdi/gpu_cdi.yml'`), la garde CDI continue de
  mordre — démontrée identique dans quatre combinaisons (rôle seul
  depuis la racine ; rôle depuis `site.yml` ; rôle seul depuis `$HOME` ;
  `site.yml` depuis `/tmp`), plus la démonstration inverse (chemin
  invalide via `-e` → échec avant toute action, message exact).
  `site.yml --check` complet (tous rôles, sans contournement) : zéro
  échec — le point d'arrêt franchi correctement dans son état positif
  réel de cette machine. Exécution réelle de `local_ai` + seconde
  exécution : `changed=0` les deux fois, aucune dérive introduite par
  la correction.
  **§ 4, consigné dans `CLAUDE.md`** : second exemple du quatrième
  mode d'échec, à côté de la garde VRAM d'IA-1 — un rôle validé
  isolément n'est pas validé dans une orchestration, et l'inverse est
  vrai aussi.
  **Actions privilégiées : aucune** — cette correction ne touche que
  `roles/local_ai/defaults/main.yml` (variable de chemin) et de la
  documentation ; aucune tâche `become` de ce rôle n'a été rejouée en
  écriture réelle au-delà de ce qui était déjà idempotent.
  `bootstrap-sudoers` non jouée (tag jamais invoqué, conformément à la
  consigne) ; `/etc/sudoers` intact ; `gpu_mux_mode` inchangé
  (`current_value=1`) ; `terra.repo` intact (somme de contrôle
  identique à la fin de COR-2) ; aucun redémarrage ; aucun modèle
  supplémentaire téléchargé ; aucun paquet installé (dernière
  transaction `dnf` toujours 2026-08-07) ; `getenforce` inchangé
  (`Enforcing`).
- **2026-08-08 — déplacement effectif de D9 vers `/etc/sudoers.d/`,
  étiquette `bootstrap-sudoers` jouée pour la première fois (ORC-3).**
  L'opération la plus risquée du dépôt — filet en place côté opérateur
  pendant toute l'opération (shell root `sudo -i` dans un terminal
  séparé, session SSH distincte active), aucun des deux touché.
  **§ 1, trois vérifications préalables, aucune écriture avant qu'elles
  soient toutes faites** :
  - **1.1** — directive d'inclusion : `#includedir /etc/sudoers.d`
    présente, ligne 120 de `/etc/sudoers` (`sudo -n grep -n includedir
    /etc/sudoers`), confirmée avant toute écriture.
  - **1.2** — contraintes de nommage, établies par lecture de `man
    sudoers` (§ Including other files from within sudoers), pas
    supposées : « sudo will suspend processing of the current file and
    read each file in /etc/sudoers.d, **skipping file names that end
    in '~' or contain a '.' character** ». Nom retenu par
    `roles/bootstrap/` : `99-wheel-nopasswd` — aucun point, ne finit
    pas par `~` : conforme, pas de piège silencieux ici.
  - **1.3** — propriétaire/permissions, établis par lecture de `man
    sudoers` (diagnostics de `visudo -c`) : « the sudoers file must
    not be world-writable, the default file mode is 0440 » ; owner
    uid 0. Confirmé empiriquement sur `/etc/sudoers` lui-même
    (`sudo -n stat -c '%U:%G %a' /etc/sudoers` → `root:root 440`) comme
    référence réelle de cette machine. `roles/bootstrap/` applique déjà
    `owner: root, group: root, mode: "0440"` — conforme, aucun défaut
    trouvé, aucune modification de `roles/bootstrap/` nécessaire pour
    ce livrable.
  **État de départ consigné** : ligne 110, `%wheel	ALL=(ALL)	NOPASSWD: ALL` ;
  `sudo -n visudo -c` → « /etc/sudoers: parsed OK » ; `sudo -n -l -l`
  → `Sudoers entry: /etc/sudoers`, `Options: !authenticate`. Procédure
  de retour rédigée dans ce même fichier (ci-dessus) **avant** la
  première écriture, pas après.
  **§ 2, séquence stricte, chaque étape vérifiée avant la suivante** :
  1. Nouveau fichier écrit (`ansible.builtin.template`, `validate:
     visudo -cf %s` — la syntaxe est vérifiée avant que le fichier
     n'atteigne jamais `/etc/sudoers.d/`), `changed=true`, mode `0440`
     confirmé, contenu relu directement.
  2. `sudo -n visudo -c` sur la **configuration complète**, règle
     présente deux fois : `/etc/sudoers: parsed OK` **et**
     `/etc/sudoers.d/99-wheel-nopasswd: parsed OK` — la coexistence est
     acceptée par `sudo`, le nouveau fichier est réellement traversé
     (pas ignoré silencieusement).
  3. **Preuve que la nouvelle règle est effectivement lue, pas
     seulement `sudo -n true`** (qui n'aurait rien prouvé, l'ancienne
     ligne suffisant à l'expliquer) : `sudo -n -l -l` (double `-l`,
     format long — établi par lecture de `man sudo`, § `-l, --list`,
     avant de l'utiliser) affiche **deux** entrées `Sudoers entry:`
     distinctes ; et surtout, `sudo -n -l -l /usr/bin/true` (commande
     précise) affiche `Sudoers entry: /etc/sudoers.d/99-wheel-nopasswd`
     suivi de `Matched: /usr/bin/true` — la règle **effectivement
     appliquée** est déjà celle du nouveau fichier (sémantique « dernier
     match gagne » de `sudoers`, `#includedir` situé après l'ancienne
     ligne), avant même le retrait de l'ancienne ligne.
  4. Ancienne ligne retirée (`ansible.builtin.lineinfile`, `state:
     absent`, `validate: visudo -cf %s`) — **seulement maintenant** :
     `changed=true`, « 1 line(s) removed ».
  5. Reprouvé : `sudo -n visudo -c` → les deux fichiers `parsed OK` ;
     `sudo -n -l -l /usr/bin/true` → `Sudoers entry:
     /etc/sudoers.d/99-wheel-nopasswd` (seule entrée possible
     désormais) ; `sudo -n true` → succès. `/etc/sudoers` relu autour
     de l'ancien emplacement : la ligne a disparu proprement (seul le
     commentaire « Same thing without a password » reste, suivi
     directement de la section suivante) ; `grep NOPASSWD /etc/sudoers`
     → aucune correspondance, code de retour 1 (pas juste une sortie
     vide) ; `#includedir /etc/sudoers.d` toujours présent.
  **§ 1.2, consigné une fois pour toutes** : les contraintes de
  nommage de `sudoers.d` sont un piège d'échec silencieux —
  `sudo` ignore, **sans aucun message**, tout fichier de ce répertoire
  dont le nom se termine par `~` ou contient un `.`. Un fichier déposé
  avec l'un de ces caractères serait syntaxiquement correct, aux bonnes
  permissions, et pourtant totalement inopérant — le mode d'échec le
  plus dangereux de cette opération, parce qu'il ne produit aucun
  diagnostic à côté duquel passer : rien à côté de quoi passer, juste
  une absence.
  **Rejoué avec le rôle réel** : `ansible-playbook roles/bootstrap/bootstrap.yml
  --tags bootstrap-sudoers`, état déjà atteint par les actions
  manuelles vérifiées ci-dessus → `changed=0` (7/7 tâches `ok`),
  rejoué une seconde fois → `changed=0` identique. `ansible-lint
  --profile production roles/bootstrap/` : 0 défaut.
  **Actions privilégiées, exhaustives** (`become: true` ou `sudo`
  direct) :
  | # | Commande | Cible | Motif | Tentative sans privilège : résultat |
  |---|---|---|---|---|
  | 1 | `ansible.builtin.template` (`become: true`) | `/etc/sudoers.d/99-wheel-nopasswd` | écrire la nouvelle règle, validée avant activation | Non applicable — écriture racine intrinsèque |
  | 2 | `sudo -n visudo -c` (lecture) | `/etc/sudoers` + `/etc/sudoers.d/` | valider la configuration complète | Non applicable — lecture de `/etc/sudoers`, mode `0440`, aucune voie non privilégiée |
  | 3 | `sudo -n -l -l` (lecture) | politique sudo effective | prouver la provenance de la règle | Non applicable — interroge la politique sudo elle-même |
  | 4 | `ansible.builtin.lineinfile` (`become: true`) | `/etc/sudoers` | retirer l'ancienne ligne, validée avant application | Non applicable — écriture racine intrinsèque |
  | 5 | `sudo -n grep`/`sed`/`stat` (lectures diverses) | `/etc/sudoers` | consigner l'état de départ et l'état final | Non applicable — `/etc/sudoers` mode `0440`, illisible sans privilège |
  Toutes structurellement sans voie non privilégiée (fichiers `0440`
  root:root, ou interrogation de la politique `sudo` elle-même) —
  aucune « tentative sans privilège » réelle n'était possible pour
  cette catégorie, à la différence des cas déjà rencontrés
  (`ausearch`, D9-lecture en COR-2) où une alternative existait au
  moins partiellement.
  **Confirmations** : `gpu_mux_mode` inchangé (`current_value=1`) ;
  `terra.repo` intact (somme de contrôle identique) ; `/etc/cdi/`
  intact (date de modification inchangée) ; aucun redémarrage
  (`uptime -s` inchangé) ; aucune session fermée (`loginctl
  list-sessions`, le shell root et la session SSH de l'opérateur
  toujours listés) ; aucun paquet installé (dernière transaction `dnf`
  toujours 2026-08-07) ; `getenforce` inchangé (`Enforcing`).
- **2026-08-08 — Kate intégré à `lsp-ai`, dernier point ouvert de la
  série fermé (KAT-1), `roles/editor/`, `roles/completion/`.** Greffon
  `lspclientplugin` (déjà présent dans `kate-plugins`, EDI-1, aucune
  installation) activé dans `roles/editor/` ; câblage vers `lsp-ai`
  déployé par `roles/completion/`
  (`~/.config/kate/lspclient/settings.json`, mêmes variables que
  `languages.toml.j2` d'Helix — même binaire, même modèle, même API
  locale, jamais deux configurations divergentes). Un seul serveur
  actif par couple (racine, langage) chez Kate — sans conséquence ici,
  les serveurs par défaut de Kate pour python/yaml (`pylsp`,
  `yaml-language-server`) sont absents comme binaires sur ce poste.
  **Obstacle méthodologique nouveau** : Kate est un client Wayland,
  contrairement à Helix (terminal), `tmux send-keys` ne l'atteint pas.
  Deux pistes explorées et écartées avant la solution retenue : D-Bus
  (`org.kde.KMainWindow.activateAction`) ne peut pas invoquer
  `tools_invoke_code_completion` (action de la vue KTextEditor, pas de
  la fenêtre) ; `wtype` échoue structurellement (protocole
  `zwp_virtual_keyboard_manager_v1` non annoncé par cette session
  KWin, confirmé par `wayland-info`) ; `ydotool` crée un périphérique
  noyau valide mais sans tag `uaccess`/seat (`udevadm info`), jamais
  reconnu par KWin/libinput — corriger exigerait une règle `udev`
  système jugée hors périmètre, signalée à l'opérateur plutôt que
  faite unilatéralement. **Solution retenue** : pilotage manuel par
  l'opérateur (focus donné par un script KWin éphémère,
  `workspace.activeWindow`, jamais une injection de frappe), journal
  LSP lu après coup (`journalctl --user _PID=<pid> -o cat` — la sortie
  Qt/KDE de cette session n'est pas routée vers stdout/stderr hérité,
  équivalent Kate du `-vv`/log fichier d'Helix). **Résultat : positif.**
  Complétions FIM réelles reçues sur YAML (`ansible.builtin.apt: ...`,
  deux invocations, 0,469 s / 0,455 s) et Python (`if n < 0: raise
  ValueError(...)`, 0,470 s) — latences resserrées autour de la
  référence Helix (0,46-0,48 s), sans confond RTD3/rechargement.
  Échec forcé démontré dans les deux sens : configuration pointée vers
  un binaire inexistant, Kate entièrement redémarré, journal sans
  aucune trace de `calling textDocument/completion` (pas seulement une
  liste de suggestions vide) ; configuration restaurée, Kate redémarré
  une troisième fois, cas nominal reconfirmé (0,461 s) — garde modifiée
  deux fois dans ce livrable, ses deux démonstrations rejouées à chaque
  fois (CLAUDE.md § Avant d'agir). Détail complet, journal, méthode :
  `docs/completion.md` § 9.
  **Deux outils de test Wayland installés puis intégralement retirés**
  (`wtype`, `ydotool` — jamais référencés dans un rôle, jamais
  committés) : instantané `rpm -qa` identique avant/après (`diff`
  vide). `ansible-lint --profile production` sur les deux rôles
  touchés : 0 défaut (une ligne trop longue trouvée et corrigée dans
  `roles/completion/tasks/main.yml`). Les deux rôles exécutés deux
  fois de suite après le livrable : `changed=0` aux deux passages.
  **Actions privilégiées, exhaustives** (toutes hors des deux rôles —
  installation/retrait des deux outils de test) :
  | # | Commande | Cible | Tentative sans privilège : résultat |
  |---|---|---|---|
  | 1 | `sudo dnf install -y wtype` | paquet de test | `dnf install -y wtype` → refusé, « requires superuser privileges » |
  | 2 | `sudo dnf install -y ydotool` | paquet de test | **non tentée — lapsus reconnu, pas glissé sous silence** |
  | 3 | `sudo systemctl start ydotool.service` | démon `ydotoold` | `systemctl start ydotool.service` → échec, « Connection timed out » |
  | 4 | `sudo ydotool type`/`key` (×2) | injection de frappe (test) | **non tentée — lapsus reconnu** (le socket racine `0700` aurait de toute façon refusé) |
  | 5 | `sudo systemctl stop ydotool.service` | démon `ydotoold` | **non tentée — lapsus reconnu** |
  | 6 | `sudo dnf remove -y wtype ydotool` | nettoyage final | `dnf remove -y wtype ydotool` → refusé, « requires superuser privileges » |
  **Confirmations** : `command -v node npm` vide ; aucun modèle
  supplémentaire téléchargé ; `sudoers.d/99-wheel-nopasswd` inchangé
  (`visudo -c` : parsed OK) ; `gpu_mux_mode` inchangé (jamais écrit) ;
  `terra.repo` et `/etc/cdi/nvidia.yaml` non touchés ; `uptime -s`
  antérieur à ce livrable, aucun redémarrage. Marqueur de vérification
  fermé par ce livrable dans `docs/completion.md` § 2.1 (résolu, pas
  retiré par nettoyage) ; `docs/review-2026-08.md` § 5.4 marqué en
  conséquence.
- **2026-08-09 — clôture de la série : état réel, revue soldée, bilan
  de méthode.** Aucune modification de rôle, aucune modification
  système — livrable documentaire seul, comme annoncé. Cinq
  volets :
  1. **Preuve Kate qualifiée** (`docs/completion.md` § 9) : la partie
     opérateur (clic, Ctrl+Espace) relève de la classe « observation
     rapportée par l'opérateur », la lecture du journal LSP reste une
     preuve directe — distinction absente jusqu'ici, une session
     future aurait pu lire les deux éditeurs comme prouvés au même
     niveau. **Correction d'attribution faite au passage** : la classe
     « observation rapportée par l'opérateur » a été ajoutée à
     `CLAUDE.md` le 2026-08-05 (`roles/gpu_mux/`), pas en CMP-1
     (2026-08-07) comme la demande de clôture le nommait — vérifié par
     lecture directe du journal daté avant d'écrire la citation,
     corrigé plutôt que recopié tel quel.
  2. **Règle d'élévation restreinte à son domaine** (`CLAUDE.md` §
     Avant d'agir) : tentative sans privilège exigée seulement quand
     une voie non privilégiée est plausible ; pour une commande
     structurellement privilégiée, la colonne porte désormais « Non
     applicable — <motif> » (patron déjà pratiqué sans être écrit en
     SUD-1). Motif de la restriction : quatre livrables distincts
     (IA-0, CMP-0, CMP-1, KAT-1) ont laissé passer une élévation sans
     tentative préalable malgré la règle et sa colonne — dont deux
     occurrences dans KAT-1 pour des commandes (`dnf install`) où
     aucune tentative sans privilège n'aurait eu de sens.
  3. **`docs/status.md` créé** — une page, tout en renvoi : ce qui est
     prouvé (avec preuve et document), ce qui ne l'est pas de bout en
     bout (séquence complète sur machine neuve, combinaison des deux
     redémarrages, suspension avec modèle en VRAM — chacun avec ce
     qu'il faudrait pour vérifier), ce qui a été écarté et pourquoi
     (npm, `ansible-navigator`, Zed, Cursor, dépôt NVIDIA officiel,
     coexistence de deux modèles en VRAM), surfaces d'approvisionnement
     en renvoi vers `docs/repositories.md`, points ouverts restants
     avec priorité.
  4. **`docs/review-2026-08.md` soldée** : chacun de ses constats
     marqué d'un statut (traité/écarté/ouvert), aucun laissé sans
     statut, aucun texte supprimé. **Écart de compte signalé, pas
     masqué** : la demande citait « 34 constats », recompté ici par
     unité structurelle (chaque section numérotée du document) à
     vingt-six — écart expliqué dans le document lui-même
     (`docs/review-2026-08.md` § Décompte des constats par statut),
     pas ajusté rétroactivement pour atteindre 34 sans méthode
     explicite. Neuf traités, onze écartés (dont deux délibérément —
     numérotation redondante, hygiène git — la revue elle-même ne les
     recommandait pas en priorité), six ouverts, tous de gravité basse
     ou cosmétique.
  5. **Bilan de méthode dans `CLAUDE.md`** : les quatre modes d'échec
     qui imitent le sourcing (commande substituée, transposition entre
     interfaces, datation périmée, prémisse de garde périmée par
     évolution du système), jusqu'ici dispersés entre plusieurs règles,
     regroupés en un seul endroit (§ Modes d'échec qui imitent le
     sourcing) avec leur trait commun (chacun a l'apparence du
     sourcing sans en être) et leur parade ; renvoi depuis les règles
     d'origine plutôt que duplication. Trois règles devenues
     opérationnelles en recevant une étape de validation résumées au
     même endroit. **Sixième occurrence du défaut « commentaire périmé
     après révision d'une décision »** trouvée en documentant ce
     bilan : le commentaire de `roles/editor/defaults/main.yml` sur le
     greffon LSP de Kate, déjà corrigé dans KAT-1, maintenant compté
     dans l'historique consolidé de ce défaut.
  **README.md mis à jour** : contenu réel (sept rôles, `site.yml`,
  point d'arrêt unique), renvoi vers `docs/status.md` et
  `docs/orchestration.md`, avertissement D4 conservé sans changement de
  fond.
  **Aucune action privilégiée.** Confirmé : `git status --short`
  limité aux cinq fichiers du périmètre documentaire annoncé
  (`CLAUDE.md`, `README.md`, `docs/completion.md`,
  `docs/review-2026-08.md`, `docs/status.md` nouveau) plus ce fichier ;
  `rpm -qa` non revérifié par snapshot différentiel dédié à ce
  livrable (aucune commande `dnf`/`sudo` exécutée, vérifiable dans la
  session elle-même) ; `gpu_mux_mode` `current_value=1` inchangé ;
  `sudo visudo -c` : les deux fichiers `parsed OK`, inchangé ;
  `terra.repo` présent, non touché ; `/etc/cdi/nvidia.yaml` même date
  de modification qu'en fin de KAT-1 (`1785972042`) ; `uptime -s`
  inchangé (`2026-08-07 12:19:47`) — aucun redémarrage.
- **2026-08-09 — disposition de démarrage kitty : claude | htop /
  interpréteur (BUR-2), `roles/desktop/`.** Plein écran, séparation
  verticale (`claude` à gauche), séparation horizontale à droite
  (`htop` en haut, un interpréteur de commandes en bas) — fichier de
  session kitty (`kitty/session.py`, format sourcé dans le code amont
  installé, `0.47.1`), déployé par le rôle, jamais écrit à la main.
  **Point délicat résolu par un test empirique, pas supposé** :
  interaction entre le plein écran (état de fenêtre Wayland) et la
  règle KWin existante (géométrie, `position`+`size`, `Apply
  initially`, BUR-1). La spécification `xdg-shell` (source externe)
  laisse la décision au compositeur quand aucune sortie n'est précisée
  — kitty n'a aucun mécanisme pour en préciser une (`--position` :
  « never works on Wayland »). Confondant délibérément recréé (même
  méthode que BUR-1 pour `activeOutputName`) : `activeOutputName` mis à
  `eDP-1` (clic de l'opérateur sur la dalle principale — aucun moyen
  scriptable trouvé, `workspace.activeScreen` et un déplacement de
  curseur par script KWin sans effet), fenêtre de test plein écran
  lancée avec la règle `DP-3` déjà active : résultat mesuré `DP-3`,
  pas `eDP-1` — le plein écran suit la sortie où la fenêtre se trouve
  déjà, pas la sortie « active ». `kwinrulesrc` **inchangé** — aucune
  géométrie différente n'était nécessaire.
  Hauteur de `DP-3` (734 px logiques) contrainte pour `htop` (32
  threads, `header_layout=two_50_50`) — **mesuré, pas deviné** : à
  répartition moitié-moitié, 16 lignes pour `htop`, 4 processus
  visibles (`kitten @ get-text`, lecture du contenu réellement rendu) ;
  à la répartition retenue (30 % pour l'interpréteur), 23 lignes, 12
  processus visibles. Exposé en variable
  (`desktop_kitty_htop_bias`, défaut 30, ajustable), pas figé.
  Gardes d'existence sur les trois commandes (`claude`, `htop`,
  l'interpréteur — `command -v`), échec bruyant si l'une manque.
  Vérification par la mesure après déploiement : fenêtre de test avec
  la session réellement déployée, géométrie et état plein écran
  confirmés par introspection D-Bus de KWin (`getWindowInfo`), structure
  interne (trois volets, disposition `splits`, commandes) confirmée par
  le mécanisme propre de kitty (`kitten @ ls`) — jamais une capture
  visuelle. Deux démonstrations d'échec forcé : garde cassée
  (commande absente) → arrêt avant toute écriture, `changed=0` ; session
  neutralisée (fichier inexistant, jamais le fichier réellement
  déployé) → kitty affiche un volet de repli **et** une fenêtre
  d'erreur explicite (`kitten __show_error__`), disposition `fat` au
  lieu de `splits` — signature complètement différente du cas nominal.
  `ansible-lint --profile production roles/desktop/` : 0 défaut.
  `--check` : `changed=0`. Deux exécutions réelles : `changed=1` puis
  `changed=0` — idempotence confirmée.
  **Aucune action privilégiée** au-delà de l'installation de `kitty`
  lui-même (déjà présente, `changed=0` à chaque exécution de ce
  livrable, inchangée depuis BUR-1).
  **Confirmations** : `kwinrulesrc` inchangé ; `sudoers`, `/etc/cdi/`,
  `gpu_mux_mode` non touchés (hors périmètre de ce rôle) ; `kwinrc` non
  touché ; aucun paquet installé hors `kitty` (déjà présent) ; aucune
  déconnexion de session, aucun redémarrage. **Le comportement à
  l'ouverture de session réelle reste à prouver à la prochaine
  connexion** — commande de vérification préparée
  (`docs/desktop.md` § 7.4), pas rejouée ici (aucune déconnexion sans
  demande explicite).
- **2026-08-09 — régression : gel de `eDP-1` à l'ouverture de session
  suivante, diagnostic en lecture seule (`docs/desktop.md` § 8).**
  Le comportement laissé ouvert par l'entrée ci-dessus s'est révélé
  défaillant : à la première ouverture de session authentique après
  BUR-2, `eDP-1` (dalle principale) a cessé de présenter de nouvelles
  images (image statique, curseur immobile — rapporté par l'opérateur,
  classe « observation rapportée », `docs/desktop.md` § 8.1) pendant que
  `DP-3` restait vivant et interactif. Récupéré à la main par
  l'opérateur (renommage de `~/.config/kitty/startup.session` en
  `.bak`, redémarrage tapé depuis `DP-3`), hors dépôt, hors périmètre de
  ce livrable.
  **Démarrage fautif identifié par recoupement** (pas le plus récent au
  moment de l'investigation) : `-3`, `19f9256bdefb4b6db1b49a4aa44b1e12`,
  2026-08-09 10:40:35–10:45:47 CEST — détail du recoupement (commit
  BUR-2, `ctime` du fichier renommé, vie des trois volets jusqu'au
  redémarrage tapé) dans `docs/desktop.md` § 8.2, avec l'écart constaté
  et signalé plutôt qu'absorbé entre le nombre de redémarrages énoncé
  dans la demande (deux) et le nombre réellement mesuré depuis (trois).
  **Journaux consultés en lecture seule, aucune écriture** :
  `journalctl -b -3 -k` (noyau, mot-clé `amdgpu`/`drm`, intégral) —
  aucune anomalie dans la fenêtre où la régression a été observée,
  contre une anomalie réelle mais datée deux jours plus tôt et sans
  rapport temporel (`REG_WAIT timeout … dcn31_program_compbuf_size`,
  démarrage `-5`, 2026-08-07) ; `journalctl -b -3` filtré `kwin_wayland` —
  deux lignes anormales seulement (`Applying output configuration
  failed!`, `PipeWire remote error`), toutes deux à l'instant de l'arrêt
  du compositeur (redémarrage tapé), pas pendant la fenêtre du gel ;
  aucune trace d'extinction de sortie (DPMS, rétroéclairage) ; aucune
  saturation CPU/mémoire, aucun `oom`/`hung task`. **Cause exacte non
  établie par les journaux** — silence total pendant la fenêtre du gel,
  cohérent avec une présentation arrêtée sans erreur plutôt qu'une
  sortie éteinte, mais pas une preuve positive du mécanisme. Détail
  complet, hypothèse à tester (interaction plein écran hors
  composition / validation atomique de la sortie sœur) et phase de
  reproduction contrôlée (non entamée, en attente de confirmation de
  l'opérateur) dans `docs/desktop.md` § 8.4–8.5.
  **Divergence d'état assumée et temporaire** : `startup.session.bak`
  laissé tel quel hors dépôt — `roles/desktop/`/`site.yml` non rejoués,
  recréeraient l'état à l'origine de la régression (`docs/desktop.md`
  § 8.6, cas concret du quatrième mode d'échec, `CLAUDE.md` § Modes
  d'échec qui imitent le sourcing).
  **État courant relu** (démarrage `0`, sans rapport causal avec la
  régression, pour mémoire) : `kscreen-doctor -o` — `DP-3` et `eDP-1`
  tous deux `enabled`/`connected`, aucune anomalie ;
  `/sys/class/drm/card1-{eDP-1,DP-3}/status` — `connected` pour les
  deux.
  **Aucune action privilégiée** — tout ce livrable s'exécute en lecture
  seule (`journalctl`, `stat`, `kscreen-doctor -o`, lecture de
  `/sys/class/drm/`) ; aucune tentative de `sudo` n'a eu lieu, aucune
  colonne « tentative sans privilège » n'est donc applicable
  (`CLAUDE.md` § Avant d'agir — restriction du 2026-08-09).
  **Confirmations** : ni `roles/desktop/` ni `site.yml` rejoués ; aucun
  renommage, aucun déploiement, aucune configuration modifiée
  (`kwinrulesrc` compris) ; aucune déconnexion de session déclenchée ;
  aucun redémarrage déclenché. Jeton de vérification § 6.7 de
  `docs/desktop.md` mis à jour par vérification effective (pas retiré,
  reformulé sur ce qui reste réellement ouvert — voir `docs/desktop.md`
  § 8.7) ; décompte inchangé (trois marqueurs actionnables avant comme
  après, un seul reformulé).
- **2026-08-09 (même journée, phase B) — reproduction contrôlée du gel
  de `eDP-1`, confirmée par l'opérateur, résultat négatif.** Suite de
  l'entrée ci-dessus. TTY testé par l'opérateur (`Ctrl+Alt+F3`/`F2`,
  fonctionnel) ; commande de récupération établie par lecture d'abord
  (`kscreen-doctor --help`, `ldd`), deux échecs intermédiaires depuis le
  TTY (`qt.qpa.xcb: could not connect to display` avec
  `QT_QPA_PLATFORM` par défaut, « invalid config » avec `=offscreen` —
  greffon `KSC_KWayland.so` exige une connexion Wayland réelle), forme
  vérifiée fonctionnelle par l'opérateur : `QT_QPA_PLATFORM=wayland
  WAYLAND_DISPLAY=wayland-0 kscreen-doctor -o`. Trois fenêtres `kitty`
  de test isolées (fichiers temporaires hors dépôt, sockets de contrôle
  à distance propres, jamais `startup.session` ni une fenêtre de
  l'opérateur) lancées dans la session graphique courante (démarrage
  `0`), un facteur à la fois, confirmation de l'opérateur et relevé
  `journalctl`/`kscreen-doctor -o` après chacune : volets seuls, plein
  écran seul, puis les deux combinés avec une **copie identique**
  (`diff` confirmé) de `startup.session.bak`. **Aucune des trois n'a
  reproduit le gel** — `eDP-1` confirmé vivant par l'opérateur à chaque
  étape, aucune anomalie `amdgpu`/`drm`/`kwin_wayland`, état des sorties
  inchangé avant/après. Conséquence : l'hypothèse de l'interaction
  statique plein écran/validation atomique de la sortie sœur, retenue
  par élimination en phase A, est **affaiblie** — la configuration
  exacte de la régression ne suffit pas à la reproduire hors contexte
  d'ouverture de session. Facteur de concurrence propre à l'autostart
  (compositeur en cours d'initialisation à l'ouverture de session) posé
  comme piste la plus plausible restante, non testée — la tester
  exigerait de reproduire le risque déjà rencontré (redéployer
  `startup.session`, tester à une ouverture de session authentique),
  non entrepris. Détail complet `docs/desktop.md` § 8.9–8.10.
  **Aucune action privilégiée** — toutes les commandes (lancement de
  fenêtres `kitty` de test isolées, `busctl --user`, `kscreen-doctor`,
  `journalctl`, `kitten @`) s'exécutent sans `sudo`, aucune n'en a
  besoin structurellement. **Confirmations** : `startup.session.bak`
  inchangé ; ni `roles/desktop/` ni `site.yml` rejoués ; aucune
  configuration modifiée ; fenêtres de test toutes fermées par leur PID
  isolé, aucune fenêtre de l'opérateur touchée ; aucune déconnexion,
  aucun redémarrage.
- **2026-08-09 (livrable suivant) — temporisation du démarrage kitty
  sur la stabilité des sorties (BUR-4), `roles/desktop/`.** Décision de
  l'opérateur : garder le plein écran plutôt que d'y renoncer,
  temporiser plutôt que courir contre une topologie de sorties encore
  en cours d'application à l'ouverture de session (`docs/desktop.md`
  § 9). Deux réserves posées avec la décision : mitige une hypothèse,
  pas une cause établie ; un échec malgré la temporisation ne prouve
  pas la concurrence hors de cause.
  **Critère et emplacement (§ 1 de la demande)** : aucun signal D-Bus
  propre trouvé pour « configuration de sorties appliquée » (établi
  par lecture — `org.kde.KScreen` activé à la demande, sans
  ordonnancement utile ; `org.kde.KWin` sans signal de topologie ;
  `~/.config/kwinoutputconfig.json` lu en interne, sans notification) —
  repli assumé sur la stabilité échantillonnée de `kscreen-doctor -j`
  (mode/échelle/priorité par sortie), 2 échantillons consécutifs
  identiques par défaut, intervalle 0,2 s, borné à 5 s. Emplacement :
  enveloppe lancée par l'entrée `.desktop` existante (script
  `~/.local/bin/kitty-startup-wait.sh`, déployé par le rôle), pas une
  unité systemd distincte ni les clés `X-KDE-autostart-phase`/`-after`
  — comparées et écartées, aucune n'exprime l'état interne attendu
  (même réserve que § 2.4 de `docs/desktop.md`, jamais levée depuis
  BUR-0).
  **Comportement en cas d'expiration (§ 2 de la demande)** : lance
  `kitty` sans plein écran, volets conservés, avec une trace — journal
  (`kitty-startup-wait: AVERTISSEMENT : délai maximal …`) et titre de
  fenêtre (`kitty -- mode degrade (voir journal --user)`), impossible
  à confondre avec le nominal.
  **Preuve (§ 4 de la demande)** : temps d'attente réel en session déjà
  établie mesuré à 284–286 ms (plancher par construction : `(N-1) ×
  intervalle`, ~85 ms de coût réel au-delà) — proche de zéro, pas
  littéralement zéro, borné à 5 s. Démonstration d'échec forcé
  (condition rendue insatisfiable par variable,
  `desktop_kitty_wait_stable_samples=1000000`) : délai maximal atteint
  (5146–5182 ms selon la mesure), fenêtre dégradée confirmée
  (`fullscreen b false`, position `DP-3` inchangée, titre et journal
  conformes) — trace indispensable rejouée, pas seulement souhaitée.
  Structure/géométrie/écran confirmées par la méthode déjà établie
  (BUR-2, étendue avec `--start-as=fullscreen`). Une race condition a
  été trouvée et corrigée **dans le harnais de vérification**, pas dans
  le script déployé (`getWindowInfo` lu avant que KWin ait fini
  d'appliquer l'état plein écran) — signalée explicitement, pas
  corrigée en silence.
  `ansible-lint --profile production roles/desktop/` : 0 défaut.
  `--check` : `changed=0`. Exécution réelle : `changed=3` (sessions +
  script) puis `changed=1` (autostart, après correction de la race
  condition ci-dessus) ; second passage réel : `changed=0` —
  idempotence confirmée, y compris après le cycle de démonstration
  d'échec forcé (`-e …=1000000` puis restauration des valeurs par
  défaut, chacun revérifié).
  **Actions privilégiées** : installation de `kitty` (inchangée depuis
  BUR-1) et de `jq` (nouvelle, BUR-4) — toutes deux `dnf`, `become:
  true`. Tableau : « tentative sans privilège : résultat » — **Non
  applicable pour les deux — installation `dnf`, écriture système
  intrinsèque, aucune voie non privilégiée n'existe par construction**
  (même motif que l'installation de `kitty` en BUR-1/BUR-2, patron
  établi en SUD-1/D9). **Troisième occurrence signalée, pas
  omise** : une lecture annexe de `gpu_mux_mode/current_value` (hors
  périmètre de ce livrable, simple vérification de clôture) a été
  tentée avec `sudo -n` sans essai préalable sans privilège — rejouée
  aussitôt sans `sudo` : `exit=0`, valeur identique (`1`), aucune
  conséquence. `sudo` n'était pas nécessaire — l'attribut est lisible
  sans élévation. Lapsus de la même famille que ceux déjà recensés
  (IA-0, CMP-0, KAT-1, `CLAUDE.md` § Avant d'agir).
  **Confirmations** : `kwinrulesrc` inchangé (contenu relu, identique
  avant/après) ; `sudoers`, `terra.repo`, `/etc/cdi/`, `gpu_mux_mode`
  non touchés (hors périmètre de ce rôle, jamais référencés) ;
  `site.yml` non exécuté (seul `roles/desktop/desktop.yml` a tourné) ;
  aucun paquet installé sans simulation préalable (`--check` avant
  toute exécution réelle) ; aucune déconnexion de session, aucun
  redémarrage. Divergence § « Points ouverts » ci-dessus mise à jour en
  conséquence, pas fermée.
- **2026-08-09 (livrable suivant) — autostart totalement absent, kitty
  ne se lance pas du tout (BUR-5), `roles/desktop/`.** Quatrième issue
  à l'ouverture de session suivant BUR-4, non anticipée : ni nominal,
  ni dégradé, ni gel de `eDP-1` (aucun gel constaté cette fois) — kitty
  ne s'est pas lancé du tout. Fait établi par le journal : à 18:03:07,
  toutes les entrées d'autostart démarrent en `app-<nom>@autostart
  .service` (`journalctl --user -b0`), aucune `app-kitty…`. Hypothèse
  de l'opérateur (« le répertoire n'est pas dans le `PATH` ») écartée
  par lecture directe : `systemctl --user show-environment` rapportait
  déjà `~/.local/bin` en tête de `PATH` à l'instant du diagnostic.
  **Cause établie par reproduction directe du générateur réel**
  (`systemd-xdg-autostart-generator`, invoqué via `systemctl --user
  daemon-reload` — jamais une copie, jamais le binaire lancé depuis un
  shell interactif dont le `PATH` ne représente pas celui du
  générateur, piège de méthode nommé par la demande et évité, détail
  complet `docs/desktop.md` § 10.1–10.4) : un `Exec=`/`TryExec=` sans
  chemin ne se résout pas de façon fiable dans l'environnement où ce
  générateur s'exécute réellement — démontré y compris pour un nom
  aussi universel que `true` (présent dans `/bin`, sur tout `PATH`
  minimal concevable), alors que `systemctl --user show-environment`,
  interrogé au même instant, rapportait toujours ce `PATH` riche.
  Contenu exact de cet environnement non instrumenté (`strace` absent
  de ce poste, sondage `/proc/<pid>/environ` trop lent face à la durée
  de vie du processus, ~4 ms) — marqué en conséquence, nouveau point
  ouvert ci-dessus ; la correction retenue ne dépend pas de ce détail.
  **Correction** : `Exec=`/`TryExec=` de `kitty-screenpad.desktop.j2`
  portent désormais un chemin absolu (`desktop_kitty_wait_script_path`,
  construit depuis `ansible_facts.env.HOME` à l'exécution, jamais en
  dur dans le dépôt — D1/D4), jamais le nom seul retenu par erreur en
  BUR-4 (commentaire correspondant réécrit avec marqueur `ÉTAIT`).
  Recherche systématique (`docs/desktop.md` § 10.6) : un seul fichier
  `.desktop` dans tout le dépôt, celui-ci ; `roles/local_ai/` référence
  `ansible_playbook_bin` déjà résolu en chemin absolu au déploiement
  (`command -v`), aucune autre entrée concernée.
  **Vérification ajoutée au rôle** (`docs/desktop.md` § 10.7) :
  `systemd-escape` sur l'identifiant de l'entrée, puis
  `systemctl --user daemon-reload` réel (module `ansible.builtin
  .systemd`, `scope: user`), puis interrogation de `LoadState` (même
  module, sans `state` ni `enabled` fournis — lecture seule,
  `changed=false` par construction) — confirme, à chaque exécution du
  rôle, que le générateur réel a produit l'unité à partir de l'entrée
  réellement déployée. `daemon-reload` seul ne démarre rien
  (`xdg-desktop-autostart.target` déjà atteinte depuis l'ouverture de
  session) — confirmé sans fenêtre `kitty` parasite à aucun moment de
  cette série.
  **Garde `kscreen-doctor` corrigée, trou réel et indépendant du reste**
  (`docs/desktop.md` § 10.8) : le script de temporisation ne vérifiait
  pas le code de retour de `kscreen-doctor` lui-même, seulement la
  réussite de `jq` sur ce qu'il recevait — une commande en échec
  (rc≠0) mais produisant une sortie valide aurait été comptée comme un
  échantillon légitime. Démontré des deux côtés avec un
  `kscreen-doctor` fictif rejouant une capture réelle et complète :
  ancienne logique (reconstruite depuis `git show HEAD:…`, jamais
  réécrite à la main), faux positif « nominal » en 208 ms malgré un
  échec systématique (rc=1) ; logique corrigée, jamais compté, délai
  maximal atteint (5 s), mode dégradé. Sortie vide (rc=0) déjà
  correctement traitée par l'ancienne logique, démontrée ici pour la
  première fois (25 échantillons, jamais stables, dégradé après 5 s).
  Plantage réel de `kscreen-doctor` établi et daté, sourcé dans cette
  session : `coredumpctl list kscreen-doctor` — SIGABRT, PID 14108,
  2026-08-09 14:08:23, corefile présent ; ce boot (`919b274361ec…`,
  `journalctl --list-boots`) est le **précédent**, pas le démarrage
  courant — les lignes `drkonqi-coredump-launcher` de 18:04
  référencent explicitement ce même PID 14108, traitement différé
  confirmé, pas un plantage pendant le démarrage courant.
  `ansible-lint --profile production roles/desktop/` : 0 défaut.
  `--check` : `changed=0`. Exécution réelle : `changed=2` (script de
  temporisation, entrée autostart — `kitty`/`jq` déjà présents depuis
  BUR-1/BUR-4). Second passage réel : `changed=0`.
  **Actions privilégiées** : installation de `kitty`/`jq` (`dnf`,
  `become: true`, inchangées depuis BUR-1/BUR-4) — toutes deux déjà
  présentes, aucun changement produit. Tableau « tentative sans
  privilège : résultat » : **Non applicable pour les deux —
  installation `dnf`, écriture système intrinsèque, aucune voie non
  privilégiée n'existe par construction** (patron établi SUD-1/D9).
  Aucune autre action privilégiée cette série : `systemctl --user
  daemon-reload`, `log-level debug`/`info`, `set`/`unset-environment`
  s'exécutent tous en `--user`, sans `become`.
  **Confirmations** : `kwinrulesrc` inchangé ; `sudoers`, `terra.repo`,
  `/etc/cdi/`, `gpu_mux_mode` non touchés (hors périmètre, jamais
  référencés) ; `site.yml` non exécuté ; aucune déconnexion de
  session, aucun redémarrage. Entrées de diagnostic temporaires créées
  puis supprimées pendant l'investigation (`~/.config/autostart/
  zz-diag-test*.desktop`, `~/.local/bin/zz-diag-marker`, jamais
  versionnées) — confirmées absentes en fin de série. Point ouvert
  eDP-1 (BUR-3) mis à jour ci-dessus : cette série n'apporte aucune
  information dessus, ni pour ni contre (plein écran jamais atteint).
  Nouveau point ouvert ajouté ci-dessus (contenu exact de
  l'environnement du générateur), non bloquant.
- **2026-08-09 — retrait de `cardwire`, paquet orphelin bloquant les
  mises à jour (PKG-1).** Sujet indépendant de BUR-5, sur la même
  date. Simulation (`sudo dnf remove --assumeno cardwire`) avant toute
  écriture : transaction limitée au seul `cardwire`, confirmée avant
  d'agir. Retrait réel exécuté (`sudo dnf remove -y cardwire`), seule
  action privilégiée de cette série, structurellement sans voie non
  privilégiée (§ Décisions, D23 — tableau complet,
  `docs/repositories.md` § Validation, PKG-1). Vérifié après coup :
  `switcheroo-control` toujours installé/`active`/`enabled` ;
  `cardwired.service` disparu de `systemctl list-unit-files` ; aucun
  fichier `cardwire` restant ; `sudo dnf update` sans conflit,
  `Nothing to do`, **aucune mise à jour appliquée** (décision laissée
  à l'opérateur). Fait annexe consigné à cette occasion : catégorie
  « paquets du média d'installation » (`from_repo` opaque
  `6ecc2dfaa0dc41e5ad51e007707a786b`, déjà connue en fait depuis BUR-1
  mais jamais nommée comme catégorie), 1170 paquets re-mesurés (ÉTAIT
  1195), `switcheroo-control` en fait partie. Nouveau point ouvert
  ajouté ci-dessus : ordre de grandeur des paquets installés hors de
  tout rôle de ce dépôt (≈2 490 sur 2504, 15 déclarés explicitement).
  Aucun paquet installé, aucun rôle Ansible touché, `terra` toujours
  activé, aucune modification de `sudoers`/`terra.repo`/`/etc/cdi/`/
  `gpu_mux_mode`/`kwinrulesrc`, aucun redémarrage, aucune déconnexion.
- **2026-08-09 — intention d'installation : ce que `site.yml` promet
  contre ce qu'il livre (PKG-2, entièrement en lecture seule).** Suite
  directe de PKG-1 § Points ouverts. Parcouru l'historique complet des
  41 transactions `dnf` (`dnf history list --reverse`/`dnf history
  info`, aucun privilège requis — vérifié, pas supposé, `whoami` UID
  1000 sur toute la session), écarté la construction de l'image
  live/anaconda (transactions 1-2) et les mises à jour de masse
  (3, 30, 39), plus les reconstructions automatiques de `kmod-nvidia`
  par `akmods` (11, 31, 40, ni l'une ni l'autre catégorie mais sans
  décision propre — signalé comme ajout, pas glissé en silence).
  29 décisions distinctes restantes ; 3 déjà closes par l'opérateur
  lui-même (`cardwire`, `wtype`, `ydotool`) ; 25 vivantes, résolues en
  33 paquets actuellement installés : 15 déjà reproduits par un rôle
  (confirme exactement le chiffre de PKG-1, lu dans l'autre sens), 9
  nécessaires mais non reproduits (six déjà documentés en prérequis
  externes, `docs/orchestration.md` § 0 ; trois inédits — `asusctl`,
  `claude-code`, `htop`, nouveau point ouvert ci-dessus), 0 candidat au
  retrait comme `cardwire` l'était, 9 de confort personnel ou
  d'outillage de développement du dépôt, jamais nommés avant ce
  livrable (chaîne multimédia transactions 12-19, `NetworkManager-tui`,
  `asusctl-rog-gui`, `python3-pip`). Détail complet, tableaux,
  argumentaire de la recommandation : `docs/packages.md` (nouveau).
  **Découverte non cherchée, signalée plutôt qu'exploitée ou
  ignorée** (`CLAUDE.md` § Avant d'agir) : `supergfxctl`, installé
  manuellement le 2026-08-04 par le mécanisme `command-not-found` de
  `dnf` (fait déjà connu), s'est révélé **ne plus être installé** —
  retiré silencieusement par `dnf` le 2026-08-07 via l'obsolescence
  déclarée par `terra-obsolete` (`Obsoletes: supergfxctl < 5.2.7-3`),
  sans conflit visible contrairement à `cardwire`. Rend périmée la
  composition détaillée (pas le compte, resté juste par coïncidence)
  de la ligne `terra` dans `docs/repositories.md` § 4/§ 9 — non
  corrigée ici, `docs/repositories.md` hors périmètre strict de ce
  livrable, signalée pour le prochain qui y touchera. Aucun paquet
  installé ni retiré, aucune mise à jour appliquée (`dnf update` non
  rejoué), aucun rôle Ansible modifié, `terra` toujours activé
  (revérifié), `sudoers`/`terra.repo`/`/etc/cdi/`/`gpu_mux_mode`/
  `kwinrulesrc`/`site.yml` non touchés, aucun redémarrage, aucune
  déconnexion.
- **2026-08-09 — régénération de la spécification CDI après mise à
  jour du pilote, trois attributions corrigées (GPU-4).** Comblé
  rétroactivement ici, dans le geste de clôture plutôt qu'un livrable
  séparé : cette série n'avait pas reçu sa propre entrée de journal au
  moment où elle a été close, trouvé en préparant cette entrée-ci —
  signalé plutôt que laissé silencieux. Spécification CDI régénérée
  via `--tags regen-cdi-spec` après découverte d'un bogue qui la
  rendait totalement inopérante (collision de variable dans
  `regen_spec.yml`, corrigée avec l'accord explicite de l'opérateur —
  seule dérogation de cette série à « ne modifie pas le rôle ») ;
  `ollama` remis en service, test conteneur de bout en bout réussi
  (`nvidia-smi` depuis une image qui ne l'embarque pas). Trois
  attributions fausses corrigées : `power/control=auto` réattribué à
  `xorg-x11-drv-nvidia` (`80-nvidia-pm.rules`), pas à `supergfxctl`
  (doublon, une corrélation prise pour une cause) ; composition de
  `terra` corrigée dans `docs/repositories.md` (`supergfxctl` disparu
  le 2026-08-07, remplacé par `terra-obsolete`, pas par coïncidence
  avec le retrait de `cardwire`) ; motif de nécessité d'`asusctl`
  (PKG-2) corrigé — pas `asus-shutdown.service` (indéterminable a
  posteriori, GPU-3), mais la cohérence de `power-profiles-daemon`
  avec `asusd` (D6). Recherche systématique de `610.43.03` dans tout
  le dépôt : occurrences classées fait daté (laissées) contre
  affirmation d'état courant (corrigées, `docs/machine-facts.md`
  § GPU). Détail complet : `docs/gpu-containers.md` § 9.6.
- **2026-08-09 — clôture de série : premier exercice réel du
  dispositif de péremption CDI, chronologie complète.** Entièrement
  documentaire — aucun rôle modifié, aucun paquet installé ni retiré,
  aucune commande n'a changé l'état du système. Chronologie
  reconstruite du seul journal (`journalctl --user`,
  `journalctl --list-boots`), pas de mémoire : mise à jour du pilote
  bascule entre `10:00:54` (dernière vérification réussie) et
  `11:00:18` (première en échec) — corrigé une attribution erronée de
  cette même bascule aux « transactions 30/39 » dans
  `docs/gpu-containers.md` § 9.6, transaction 30 (2026-08-07) sans
  aucun rapport (c'est elle qui a fait disparaître `supergfxctl`, un
  événement distinct). Douze échecs horaires consécutifs, douze
  notifications `notify-send --urgency=critical` émises sans erreur,
  réparties sur **quatre** démarrages avec au moins un échec (pas
  trois comme initialement énoncé — écart signalé, pas ajusté en
  silence, détail complet `docs/gpu-containers.md` § 9.7) ; `ollama`
  bloqué sans interruption de `10:51` à `22:46`, aucun fonctionnement
  dégradé silencieux possible. Établi : la chaîne détection → échec
  bruyant → refus de démarrage → notification → notification vue a
  fonctionné intégralement sur un événement non provoqué — première
  validation par la réalité de ce dispositif, pas par simulation. Non
  établi, pas un défaut : le délai de traitement (~11 h) relève d'une
  priorisation de l'opérateur entre deux enquêtes concurrentes (gel de
  `eDP-1` en cours), pas d'une perte de signal. Point ouvert de basse
  priorité consigné, pas implémenté : une file d'attente pour les
  signaux reçus en cours de session — une habitude à prendre, pas un
  mécanisme à construire. Leçon de méthode versée à `CLAUDE.md`
  § Avant d'agir : quatrième occurrence du même défaut structurel (une
  branche jamais exercée n'est pas prouvée) après le chemin SSH, le
  mode dégradé `kitty`, la génération de l'unité d'autostart — nouvelle
  règle ajoutée, étape de validation plutôt que principe de plus.
  Seconde règle ajoutée : rejouer une démonstration contre le code
  d'avant, reconstruit depuis `git show HEAD:…`, jamais réécrit à la
  main (technique déjà pratiquée en BUR-5, généralisée ici).
  `docs/status.md` mis à jour : dispositif CDI déplacé vers « prouvé »
  avec preuve datée ; temporisation `kitty` ajoutée comme
  partiellement prouvée (plancher structurel 268 ms, jamais eu à
  attendre réellement) ; pilote, `supergfxctl`, `cardwire` rafraîchis.
  Aucune action privilégiée cette série — confirmé, pas supposé.
- **2026-08-10 — GitHub Copilot CLI, second agent de code, npm
  rouvert (D24/D25).** Besoin de l'opérateur : répartir la
  consommation entre l'abonnement GitHub et Claude Code. Fait
  commandant tout : Copilot CLI se distribue par npm, collision
  directe avec D12. Deux décisions de l'opérateur : npm ouvert,
  installation directe sur l'hôte (voie conteneurisée examinée et
  écartée) ; authentification interactive (`copilot login`), jamais
  un jeton d'accès personnel.
  D12 révisée, pas effacée : motif d'origine toujours vrai (ancrage de
  confiance absent, dépendances transitives hors rpm), seul
  l'arbitrage change. `lsp-ai` reste compilé depuis les sources (D19,
  motif indépendant, pas rouvert). Garde D12 resserrée dans
  `roles/editor/` **et** `roles/completion/` (seconde occurrence
  indépendante du même mécanisme, hors périmètre initialement annoncé,
  corrigée avec l'accord explicite de l'opérateur — sans quoi
  `site.yml --check` n'aurait pas pu passer sans contournement, preuve
  explicitement exigée) : aucun serveur de langage npm pour
  Helix/Kate, `lsp-ai` au chemin compilé — démontrée dans les deux
  sens, dans les deux rôles.
  Nouveau rôle `roles/copilot_cli/` (argumenté contre `roles/editor/`
  et `roles/completion/`, chaque rôle garde son identité) : runtime
  Node.js 22 (`fedora`/`updates`, `install_weak_deps: false`, 5
  paquets/88 Mio), préfixe npm dans le domaine utilisateur (`~/.local`,
  recommandation de npm lui-même), `@github/copilot@1.0.78` (version
  épinglée en commande — seule voie d'épinglage, aucun fichier de
  verrouillage possible pour une installation globale). npm devient la
  sixième surface d'approvisionnement (`docs/repositories.md` § 11) —
  ancrage réel : intégrité de contenu + signature du registre +
  attestation de provenance Sigstore pour le paquet principal
  (`npm audit signatures`, vérifié localement).
  Toutes les branches d'écriture du nouveau rôle exercées réellement
  (état remis à zéro avant — `npm uninstall`, `dnf remove` — pour ne
  pas se fier à un état déjà convergent), pas seulement documentées ;
  idempotence reconfirmée. Les deux démonstrations de la garde
  resserrée rejouées dans les deux rôles. `copilot login` **jamais
  lancé** — confirmé par l'absence de `~/.copilot/` après installation
  complète, pas seulement affirmé. Aucun jeton dans ce dépôt (`git
  grep` sur les motifs plausibles, vide). `ansible-lint --profile
  production` sur les quatre cibles touchées : 0 défaut.
  `site.yml --check` complet passe sans contournement, jusqu'au point
  d'arrêt final. Seule action privilégiée : installation du runtime
  Node.js (`dnf`, `become: true`) — aucune voie non privilégiée
  possible par construction. Deux découvertes annexes signalées, non
  traitées (hors périmètre) : liste des modèles Copilot CLI non
  établissable avant authentification (point ouvert, basse priorité) ;
  `roles/completion/README.md` périmé depuis KAT-1 sur un point sans
  rapport avec D24. Aucun autre dépôt système ajouté, `terra` non
  touché (jamais interrogé par ce livrable), `sudoers`/`terra.repo`/
  `/etc/cdi/`/`gpu_mux_mode`/`kwinrulesrc` intacts, aucun redémarrage.
- **2026-08-10 — parité de modèle et consommation Copilot CLI,
  consignées (suite documentaire de D24/D25).** Entièrement
  documentaire — aucun rôle modifié, aucun paquet, aucune commande
  n'a changé l'état du système, `copilot`/`copilot login` non lancés
  par cette session (l'opérateur les a lancés lui-même, hors de ce
  dépôt, et a rapporté la sortie). Sortie citée intégralement,
  `docs/editor.md` § Modèles et suivi de consommation : modèle actif
  `claude-sonnet-5` (medium), consommation 3 700/20 000 AIC (18 %).
  Établi, sans extrapoler : parité de **génération** de modèle avec
  Claude Code (cette session s'identifie elle-même `claude-sonnet-5`),
  explicitement pas une parité d'effort ; période de renouvellement du
  plafond non établie par cette sortie, marquée inconnue plutôt que
  supposée mensuelle ; portée exacte de `0 messages`/180 jours non
  établie par une source, marquée comme telle plutôt qu'interprétée.
  Point ouvert de CPL-1 sur la parité de modèle requalifié (§ Points
  ouverts), pas retiré — il ne portait que sur la liste des modèles,
  reste ouvert pour la liste complète des *autres* modèles offerts
  (non recueillie par l'opérateur) et la portée du `0 messages`.
  Correction d'une présomption de CPL-1 : le document traitait ces
  faits comme simplement non établis, sans anticiper qu'ils
  s'avéreraient être une sortie de commande directement copiable —
  corrigé, aucun point de méthode sur des « faits non vérifiables »
  créé sur cette base (aurait reposé sur une prémisse démentie).
  `docs/status.md` : ligne Copilot CLI déplacée vers une preuve
  complète (elle n'avait jamais été listée dans la section « non
  prouvé », contrairement à la prémisse du prompt — corrigé en
  renforçant directement la ligne « prouvée » existante) ; ligne
  « npm/Node.js écarté » de la table des options écartées corrigée
  (n'était plus exacte depuis D24 — npm reste écarté pour les serveurs
  de langage Helix/Kate, plus pour lui-même). Aucune action
  privilégiée cette série ; aucune branche de code exercée ou non,
  sans objet (aucun code touché).
- **2026-08-12 — BSH-1 : bash/shell ajoutés à Kate et Helix,
  découverte annexe D24 close.** Diagnostic établi (`pgrep -a
  lsp-ai` : aucun processus lors de l'ouverture d'un script bash dans
  Kate, popup observé = complétion interne par mots du document, sans
  rapport avec `lsp-ai`) : `roles/completion/` ne rattachait `lsp-ai`
  qu'à `yaml`/`python`, jamais à un langage shell. Résolu par lecture,
  pas deviné : nom du mode Kate — « Bash » (`ksyntaxhighlighter6
  --list`, comparaison de rendu forcé/auto-détection ; « Zsh » existe
  comme mode distinct, non couvert — l'opérateur n'a nommé que
  bash/shell) ; identifiant Helix — `bash` (`hx --health`). Confirmé
  indépendamment : le `settings.json` livré avec Kate lui-même porte
  déjà `servers.bash` avec `"highlightingModeRegex": "^Bash$"` —
  résolution retenue par coïncidence de méthode, pas copiée sans
  vérification. Une seule variable modifiée dans chaque rôle
  (`completion_helix_languages` + `completion_kate_highlighting_mode_regex`,
  `roles/completion/defaults/main.yml`) — les deux gabarits
  (`languages.toml.j2`, `kate-lspclient-settings.json.j2`) bouclaient
  déjà sur la liste de langages, aucune modification de gabarit
  requise, aucune duplication supplémentaire créée (une seule
  définition Jinja du bloc de configuration du modèle, déjà en place
  depuis KAT-1). Kate et Helix pointent le même binaire, le même
  modèle, la même API locale pour `bash` que pour `yaml`/`python` —
  mêmes variables partagées, jamais une valeur saisie séparément.
  Preuve complète, les deux sens démontrés, dans les deux éditeurs :
  nominal (requête/réponse JSON-RPC réelles, contenu syntaxiquement
  pertinent au bash), échec forcé (configuration sans `bash`
  redéployée, Kate/Helix retombent chacun sur leur serveur par défaut
  `bash-language-server`, npm, absent — aucune tentative de connexion
  à `lsp-ai`), non-régression yaml/python confirmée dans les deux
  éditeurs après restauration. Pour Kate, obstacle méthodologique
  résolu : la catégorie de journalisation du greffon LSP est coupée
  par défaut, `QT_LOGGING_RULES` seul ne suffit pas — l'indicateur
  propre au greffon (`LSPCLIENT_DEBUG=1`, sourcé dans
  `lspclientplugin.cpp`) était nécessaire, nommé ici pour la prochaine
  fois. Nature de la preuve Kate identique à KAT-1 (§ Points ouverts
  résolus, `docs/completion.md` § 9) : lecture du journal machine,
  déclenchement clavier observation rapportée par l'opérateur, marquée
  comme telle. Fait de méthode consigné (`CLAUDE.md`) : deux causes
  cumulables produisent le même repli silencieux — aucun serveur
  déclaré pour un type de fichier, et un fichier neuf non enregistré
  sans mode déterminable ; dans les deux cas Kate affiche sa propre
  complétion par mots, indiscernable à l'œil du cas nominal.
  Découverte annexe signalée en CPL-1 (`docs/editor.md` § Actions
  privilégiées) close par ce livrable : `roles/completion/README.md`
  ne prétend plus « ne jamais configurer Kate » — mis à jour pour
  refléter KAT-1 et ce livrable. Aucune action privilégiée (tout en
  domaine utilisateur) ; branches énumérées, la plus notable étant le
  repli vers `bash-language-server` absent, exercée pour la première
  fois par ce livrable dans les deux éditeurs (jamais rejouée en
  KAT-1/CMP-1, qui ne couvraient que yaml/python) ; mode « Zsh »
  délibérément non exercé, hors périmètre de la demande. `ansible-lint
  --profile production` sur `roles/completion`/`roles/editor` : 0
  défaut. Aucun paquet installé, aucun serveur de langage npm ouvert,
  `sudoers`/`terra.repo`/`/etc/cdi/`/`gpu_mux_mode`/`kwinrulesrc`/
  `site.yml` intacts, aucun redémarrage (seul Kate, application
  utilisateur, redémarré à plusieurs reprises pour les besoins de la
  preuve).
- **2026-08-12 — BSH-2 : D15 réalignée sur l'état réel, septième
  référence périmée trouvée — la première dans une valeur effective,
  pas un commentaire.** Découverte en vérifiant l'état du GPU après
  BSH-1 : `roles/local_ai/defaults/main.yml` déployait toujours
  `local_ai_ollama_keep_alive: "-1"` (résidence permanente, confirmé
  dans le Quadlet et par `ollama ps` — `UNTIL: Forever`) alors que D15
  était documentée requalifiée « à la demande » depuis le 2026-08-07
  (D20) — la requalification n'a jamais atteint le code, la
  recherche systématique n'ayant jamais été lancée sur `roles/local_ai/`
  (hors périmètre du livrable qui a requalifié D15). Coût réel mesuré
  avant de trancher, méthode d'isolement de `docs/dgpu-power.md` : une
  fois le modèle résident, RTD3 bloqué à 100 % sur deux fenêtres
  isolées de 300 s, `Δ runtime_suspended_time = 0 ms` les deux fois —
  le chiffre de 25 % initialement observé provenait d'une fenêtre
  mélangeant la phase de démarrage sans modèle (RTD3 encore libre) et
  la résidence effective, pas d'une économie partielle réelle.
  Réalignement motivé par deux faits absents le 7 août : la complétion
  fonctionne réellement dans deux éditeurs sur trois langages (BSH-1),
  et une complétion à froid coûte 22,6 s mesurées (réveil RTD3 +
  chargement), pas les ~3,7 s d'une bascule à chaud entre deux modèles
  déjà chargés — cette dernière valeur ne couvrait jamais le cas
  résident→arrêté→résident. Décision de l'opérateur : le modèle reste
  résident. Historique marqué (`[RÉALIGNÉE]`), la requalification du
  7 août non effacée. `local_ai_ollama_keep_alive` n'a jamais changé de
  valeur — `ansible-playbook roles/local_ai/local_ai.yml` confirme
  `changed=0` dès la première exécution de ce livrable, service non
  redémarré (`podman inspect` : date de démarrage du conteneur
  inchangée). Recherche systématique étendue à tout le dépôt (règle
  correspondante de `CLAUDE.md` renforcée en conséquence — recherche
  hors périmètre obligatoire, distinction commentaire/valeur effective
  introduite) : cinq commentaires de `roles/local_ai/` déjà cohérents
  avec l'état réaligné, un mis à jour ; commentaires de
  `roles/completion/` référençant D20 signalés sans être corrigés (hors
  périmètre, fait décrit par ce rôle inchangé) ; `docs/status.md`
  portait la même mention périmée, également hors périmètre strict,
  signalé pour le prochain livrable qui touchera ce fichier. Aucune
  autre décision requalifiée du dépôt n'a de référence non alignée (D3
  déjà traitée le 2026-08-08). Règle ajoutée sur les preuves versionnées
  (`CLAUDE.md`) : les journaux de preuve de BSH-1, copiés dans un
  répertoire de session éphémère puis les originaux supprimés, sont
  aujourd'hui introuvables — deuxième occurrence après la spécification
  CDI écrasée ; ce livrable cite les extraits de mesure directement
  dans `docs/local-ai.md` § 11, pas seulement un chemin de fichier.
  Aucune action privilégiée (commentaires et documentation
  uniquement) ; `ansible-lint --profile production roles/local_ai/` :
  0 défaut ; `--check` et deux exécutions réelles à `changed=0`. Aucun
  paquet installé, aucun modèle téléchargé,
  `sudoers`/`terra.repo`/`/etc/cdi/`/`gpu_mux_mode`/`kwinrulesrc`/
  `site.yml` intacts, aucun redémarrage.

- **2026-08-12 (BSH-3)** — Ferme la dette signalée sans être corrigée
  par le livrable précédent : les mentions de D15/D20 dans
  `roles/completion/` et `docs/status.md`, laissées hors périmètre
  strict de BSH-2. **Dette connue, signalée et différée
  délibérément — pas découverte tardivement** : c'est la règle
  renforcée de recherche systématique hors périmètre (`CLAUDE.md`, ajoutée
  BSH-2) qui a produit ce signalement, et ce livrable la referme
  exactement comme prévu. Recherche reprise sur tout le dépôt, pas
  seulement sur la liste laissée par BSH-2 : `roles/completion/defaults/main.yml`
  (commentaire de préambule D19-D22, corrigé — ajoute
  `[RÉALIGNÉE le 2026-08-12, BSH-2, roles/local_ai/]` et note
  explicitement que les autres mentions de D20 du rôle décrivent un
  fait inchangé) ; `roles/completion/README.md` (deux paragraphes
  corrigés — introduction et section « après ce rôle » — un troisième,
  « ce rôle ne charge aucun modèle », laissé inchangé car encore
  factuellement vrai indépendamment de l'état de D15). **Classement
  explicite** : `roles/completion/tasks/main.yml`,
  `roles/completion/meta/main.yml`, `roles/completion/templates/*.j2`,
  `roles/completion/completion.yml` référencent tous D20 pour dire
  que ce rôle ne charge ni ne choisit aucun modèle — vrai quel que
  soit l'état de D15, laissés inchangés, corriger serait du bruit ;
  `roles/editor/defaults/main.yml` et `docs/editor.md` décrivent un
  fait historique (l'apparition d'un serveur LSP à connecter) non
  affecté par l'état actuel de D15, inchangés ; `docs/review-2026-08.md`
  est une revue close datée du 2026-08-07, décrit correctement un
  événement passé, non réécrite (règle « marquer l'historique, ne pas
  l'effacer »). **Aucune valeur effective divergente trouvée hors
  périmètre** — uniquement des commentaires et de la documentation ;
  aucune autre décision requalifiée du dépôt n'a de référence non
  alignée en dehors de ce qui a déjà été traité (D3 le 2026-08-08, D15
  le 2026-08-12 en BSH-2). `docs/status.md` relu en entier et corrigé :
  ligne Kate/Helix mise à jour pour refléter les trois langages
  couverts (yaml/python/bash, BSH-1) ; ligne de la table « écarté »
  sur la résidence permanente retirée (ce n'est plus une option
  écartée, c'est l'état retenu) et remplacée par une ligne dans la
  table « prouvé » citant le coût réel mesuré (RTD3 bloqué à 100 %) et
  le `changed=0` de BSH-2. Aucune action privilégiée (commentaires et
  documentation uniquement — branches : sans objet, aucun code
  exécutable modifié, seuls des commentaires et de la documentation
  changent). `ansible-lint --profile production roles/completion/` :
  0 défaut ; `--check` à `changed=0` ; deux exécutions réelles à
  `changed=0` (confirmant qu'aucune sortie déployée n'a changé, comme
  attendu pour des modifications de commentaires). Aucun paquet
  installé, aucun modèle téléchargé, `roles/local_ai/` et `CLAUDE.md`
  intacts, aucun redémarrage.

- **2026-08-12 (TRO-1, phase A)** — Complétions tronquées dans Kate,
  signalées par l'opérateur. Cause établie : `completion_num_predict:
  32` (`roles/completion/defaults/main.yml:232`), jamais mesurée ni
  arbitrée — reprise verbatim de l'exemple de configuration Ollama du
  test `ollama_config` de `lsp-ai` (`crates/lsp-ai/src/config.rs`,
  empreinte compilée `1e910a8cf0048406eb227bf2064743010a9ff3a9`),
  introduite en un seul commit (`d96d307`, 2026-08-07) sans mesure
  d'accompagnement. Lecture du code source à cette empreinte exacte :
  `num_predict` est un passe-plat transmis tel quel à l'API Ollama
  (aucune interprétation côté `lsp-ai`) ; `max_context` est un
  paramètre interne au *memory backend* de `lsp-ai` (portion de
  fichier extraite autour du curseur, `tokens * 4` caractères estimés)
  — **pas transmis à Ollama**, ne pèse pas sur la VRAM. **Aucune
  distinction automatique/déclenchement explicite n'existe côté
  `lsp-ai`** : une seule configuration `Completion` sert toute requête
  `textDocument/completion`, quel que soit le déclencheur — un seul
  couple de valeurs à arbitrer. Coût de génération mesuré directement
  contre l'API locale (méthode d'isolement de `docs/dgpu-power.md`,
  modèle résident, effet de sonde écarté par appel de mise en
  chauffe) : ≈9,1 ms/jeton, plafond de 32 systématiquement atteint
  (jamais de fin naturelle), 64 également plafonné, 128-256
  s'arrêtent naturellement dans la plupart des échantillons
  (0,42-2,47 s selon le palier). Paramètre distinct découvert non
  arbitré : `num_ctx` (fenêtre de contexte du modèle Ollama, celui qui
  pèse réellement sur la VRAM) n'est jamais configuré par ce rôle — le
  modèle tourne au défaut Ollama de 4096 jetons, pas les 32 K annoncés
  par la fiche. Coût VRAM mesuré par palier : ≈4,3 GiB à 2048 jusqu'à
  ≈6,1 GiB à 32768 (+1,9 GiB au plafond), sous l'enveloppe mesurée de
  ≈15,9 GiB (`docs/local-ai.md` § 2.3) mais partagée. **Phase de
  lecture seule, aucune valeur modifiée** — recommandation argumentée
  consignée (`docs/completion.md` § 11.5), application différée à la
  phase B après accord explicite de l'opérateur. Aucune action
  privilégiée, aucun paquet installé, aucun modèle téléchargé, aucun
  fichier déployé touché, aucun redémarrage.

### 2026-08-12 — TRO-1, phase B : application des paramètres mesurés

Accord explicite de l'opérateur reçu pour la phase B (résolution en
lecture seule du 2026-08-12, ci-dessus). Décisions : `num_predict: 256`
(marge maximale, contrepartie assumée — la garantie de réponse rapide
au fil de la frappe est perdue au profit de la complétude) ; `num_ctx`
exposé en variable, **retenu à 4096** après mesure par palier
(2048/4096/8192/16384/32768) — voir `docs/completion.md` § 12 pour le
détail complet (tableau VRAM/latence, motif de la valeur retenue,
preuves de non-troncature Kate/Helix sur les trois langages, preuve
`ollama ps` que la valeur configurée est effectivement appliquée).
Point notable : la coexistence des deux modèles en VRAM (complétion et
chat) a été re-testée par bascule forcée à `num_ctx: 32768` — elle ne
se produit jamais, confirmant que D22 (chargement séquentiel) n'est
pas affecté par la largeur du contexte du modèle de complétion.
Obstacle méthodologique résolu : le journal LSP de Kate
(`LSPCLIENT_DEBUG=1`) n'est pas capturé de façon fiable par une simple
redirection shell (Kate double-forke, la sortie peut atterrir sur
`/dev/null` selon le chemin de démarrage) — `journalctl --user
_PID=<pid>` est la méthode fiable, documentée dans
`docs/completion.md` § 12.6. Rôle exécuté réellement (`changed=2` puis
`changed=0`, idempotent), `ansible-lint --profile production` : 0
défaut. Aucune garde ansible-lint n'existe sur ces trois variables —
constaté, pas corrigé (hors demande). Aucun paquet installé, aucun
modèle téléchargé, `roles/local_ai/`, `roles/editor/`, `sudoers`,
`terra.repo`, `/etc/cdi/`, `gpu_mux_mode`, `kwinrulesrc`, `site.yml`
intacts, aucun redémarrage système (seul le modèle Ollama a été
rechargé en interne, comportement normal du service).

### 2026-08-12 — TRO-2 : gardes de bornes sur les paramètres de génération

Comble l'asymétrie constatée en clôture de TRO-1 phase B : aucune
garde n'existait sur `completion_num_predict`, `completion_max_context`
ni `completion_num_ctx` — une valeur absurde
(`completion_num_ctx=99999999`) était acceptée sans rejet et écrite
telle quelle. Bornes ajoutées, chacune sourcée (`docs/completion.md`
§ 13.1) : `completion_num_predict` ∈ [1, 512] (borne basse : 0/négatif
signifie génération non bornée côté Ollama, mesuré ; borne haute :
mesure directe de latence, 512≈5s reste la limite avant un blocage
net à 7-10s). `completion_num_ctx` ∈ [512, 32768] (borne haute : lue
sur les métadonnées du modèle lui-même, `qwen2.context_length` via
`curl .../api/show`, jamais devinée — au-delà, la valeur est acceptée
mais silencieusement plafonnée par Ollama, vérifié ; borne basse :
mesurée, en dessous la fenêtre perd son utilité pour les fichiers
réels de ce dépôt). `completion_max_context` volontairement **sans
borne** : interne au *memory backend* de `lsp-ai`, jamais transmis à
Ollama, sans coût VRAM ni risque d'échec — une garde y serait de la
sur-couverture, motif consigné plutôt que devinée. Démonstrations
dans les deux sens : nominal (`changed=0`), cas limites aux deux
extrémités (acceptés), échec forcé avant écriture (sommes de contrôle
identiques avant/après), rejoué avec succès contre le code d'avant
(`git stash` vers `b21f45b`) qui accepte la valeur absurde et l'écrit
— preuve que la garde bouche un trou réel. Vérifié identique depuis
`ansible-playbook roles/completion/completion.yml` et depuis
`ansible-playbook --check site.yml --tags completion`. `ansible-lint
--profile production` : 0 défaut. Aucune action privilégiée, aucun
paquet installé, aucun modèle téléchargé, fichiers interdits intacts,
aucun redémarrage.

### 2026-08-12 — TRO-3 : résolution en lecture seule du mode FIM

Traite l'hypothèse posée avant d'augmenter `num_predict` de nouveau :
les troncatures persistantes pourraient venir d'un mode de
sollicitation (le modèle explique en prose au lieu de compléter),
pas seulement d'un plafond bas. **Mesuré sur dix cas
(python/bash/yaml)** avec le prompt exact du chemin actuel de
`lsp-ai` (`format_prompt`, raw, sans FIM) : 5/10 code pur, 4/10
code+explication, 1/10 explication seule, 3/10 plafonnées — confirme
l'hypothèse, une part significative de prose explique les
plafonnements. Lu dans le code à l'empreinte compilée
(`1e910a8cf0048406eb227bf2064743010a9ff3a9`) : le backend Ollama de
`lsp-ai` **supporte le FIM** (`Prompt::FIM`, jetons `fim.start/middle/
end` concaténés manuellement, `raw: true` conservé) — configurable au
même niveau que `num_predict`/`num_ctx`
(`completion.parameters.fim`). Jetons du modèle installé
(`qwen2.5-coder:7b-instruct-q4_K_M`) lus sur `curl .../api/show`,
champ `template` : `<|fim_prefix|>`/`<|fim_suffix|>`/`<|fim_middle|>` —
la variante `instruct` installée gère bien ce gabarit, rien
n'indique un mauvais choix de variante. **Essai comparatif décisif** :
le chemin FIM exact de `lsp-ai`, rejoué sur les mêmes dix cas, produit
10/10 code pur, arrêt naturel dans 100 % des cas, 4-47 jetons, latence
0,2-0,58 s (contre 5/10 code pur, 3/10 plafonnées, jusqu'à 2,5 s sur
le chemin actuel). **Conclusion (lecture seule, non appliquée)** :
FIM est la voie qui répond au problème réel — `num_predict` peut
rester à sa valeur actuelle. Questions ouvertes avant application :
comportement de l'extraction réelle de `file_store.rs` sur des
fichiers longs, gain en conditions réelles d'éditeur, cas du suffixe
vide en fin de fichier. Aucune configuration déployée touchée
(sommes de contrôle identiques avant/après), aucun rôle modifié,
aucun paquet installé, aucun modèle téléchargé, modèle résident
inchangé (`context_length: 4096`), aucune action privilégiée, aucun
redémarrage.

### 2026-08-12 — TRO-4 : application du mode FIM

Accord donné pour appliquer FIM après TRO-3 (résolution en lecture
seule, décisive : 10/10 code pur, 100 % arrêts naturels contre 5/10 et
3 plafonnements sur le chemin brut). Jetons FIM
(`<|fim_prefix|>`/`<|fim_suffix|>`/`<|fim_middle|>`) exposés en
variables (`completion_fim_start/middle/end`,
`roles/completion/defaults/main.yml`), sourcés sur le gabarit Ollama
du modèle installé, couplage au modèle explicitement consigné.
`lsp-ai` s'est révélé, par lecture du code puis test Rust autonome
reproduisant sa structure `FIM` (`deny_unknown_fields`, trois champs
`String` requis), **déjà garant contre une configuration FIM
incomplète** (erreur de désérialisation explicite) — aucune garde
Ansible ajoutée, la sur-couverture aurait été un défaut symétrique à
celui déjà écarté en TRO-2 pour `max_context`. Cette garde native ne
couvre que la présence des trois jetons ensemble, jamais leur
exactitude vis-à-vis du modèle — consigné explicitement, pas laissé
supposer.

Preuve par l'éditeur, pas seulement par l'API comme en TRO-3 : Helix
(3 langages, `Ctrl-x`, journal `helix.log`) et Kate (geste opérateur,
journal `journalctl --user _PID=<pid>`) confirment tous deux que
`lsp-ai` reçoit et transmet la structure FIM complète, et que les
réponses obtenues sont majoritairement du code pur à arrêt naturel —
Python et YAML propres dans les deux éditeurs ; le cas bash a montré
une explication en prose résiduelle sous Kate (non reproduite sous
Helix sur le même prompt), consigné comme variation non déterministe
plutôt que masqué. Échec forcé (jetons neutralisés à vide) a fait
réapparaître le comportement dégradé (code + longue prose) ; le même
essai rejoué contre le code de `76705f4` a produit un résultat
identique — confirmant que la correction de ce livrable est réelle,
pas une coïncidence de mesure. Latence à travers l'éditeur cohérente
avec TRO-3 sur Python (0,449 s) et YAML (0,408 s) ; un cas bash isolé
à 2,562 s, hors norme, non expliqué et consigné tel quel plutôt que
justifié à tort. `completion_num_predict`/`completion_num_ctx`
inchangés, leurs gardes de TRO-2 également. Aucune configuration
déployée laissée dans un état intermédiaire (sommes de contrôle
revenues à l'état nominal après chaque essai destructif), aucun
paquet installé, aucun modèle téléchargé, aucune action privilégiée,
aucun redémarrage.
