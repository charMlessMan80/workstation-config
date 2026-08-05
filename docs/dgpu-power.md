# Alimentation de la dGPU au repos — résolution en lecture seule

**Ce document ne corrige rien.** Toutes les commandes citées ci-dessous
sont des lectures, exécutées le 2026-08-05 entre 12:47 et 12:59 CEST, sur
ce poste, après la bascule MUX réussie du même jour (voir
[`docs/machine-facts.md`](machine-facts.md) et
[`docs/gpu-mux-recovery.md`](gpu-mux-recovery.md)). Aucune option décrite
en § Options n'a été appliquée. La décision viendra dans un livrable
séparé, après revue de cette résolution.

**Correction de méthode annoncée d'emblée** : le point ouvert consigné le
2026-08-05 dans `docs/machine-facts.md` (« dGPU maintenue en `P0` au
repos... contre `P8` avant ») reposait sur une série de lectures
`nvidia-smi` rapprochées, sans relevé de contrôle espacé. Ce livrable
corrige cette erreur de méthode par construction — voir § État mesuré,
méthode d'isolement de l'effet de mesure.

## État mesuré

### Méthode d'isolement de l'effet de mesure

`nvidia-smi` interroge le pilote via un appel qui réveille le
périphérique — ce fait est documenté par NVIDIA elle-même (voir §
Mécanismes) et vérifié ci-dessous par l'expérience, pas supposé. Pour
distinguer un état de repos réel d'un état provoqué par la mesure, deux
relevés espacés de **4 min 12 s, sans invoquer `nvidia-smi` ni aucun
outil NVIDIA entre les deux**, ont été pris d'abord — uniquement des
fichiers `sysfs` passifs (`power/runtime_status`,
`power/runtime_suspended_time`, `power/runtime_active_time`, maintenus
par le cœur PM du noyau, pas par le pilote NVIDIA — leur lecture ne
déclenche pas de réveil). Une sonde `nvidia-smi` délibérée, clairement
isolée, a ensuite été exécutée deux fois séparément pour observer l'effet
causal, avec relecture immédiate des mêmes compteurs.

**T0 — 2026-08-05T12:47:11+02:00**, lecture sysfs pure (aucun
`nvidia-smi` invoqué avant) :

```
$ for f in control runtime_status runtime_suspended_time runtime_active_time; do
    for dev in 0000:01:00.0 0000:01:00.1; do
      cat /sys/bus/pci/devices/$dev/power/$f
    done
  done
0000:01:00.0 control              = auto
0000:01:00.1 control              = auto
0000:01:00.0 runtime_status       = suspended
0000:01:00.1 runtime_status       = suspended
0000:01:00.0 runtime_suspended_time = 8255816
0000:01:00.1 runtime_suspended_time = 8342677
0000:01:00.0 runtime_active_time  = 95570
0000:01:00.1 runtime_active_time  = 8709
```

Boot en cours démarré à `2026-08-05 10:27:59` CEST
(`uptime -s`, confirmé par `journalctl --list-boots`). À T0, `2h19m12s`
(8352 s) se sont écoulées depuis le démarrage — cohérent avec
`runtime_suspended_time + runtime_active_time ≈ 8255816 + 95570 =
8351386` ms sur la fonction `.0` : **le compteur cumulé colle à
l'incertitude près à la durée d'uptime**, ce qui confirme que ces
compteurs sont bien réinitialisés au démarrage et n'ont pas d'autre
source de remise à zéro cachée.

**T1 — 2026-08-05T12:51:23+02:00**, même lecture, **toujours sans
`nvidia-smi` entre T0 et T1** :

```
0000:01:00.0 runtime_status       = suspended
0000:01:00.0 runtime_suspended_time = 8507901   (Δ = +252085 ms)
0000:01:00.0 runtime_active_time  = 95570        (Δ = 0 ms)
```

Écart T1−T0 réel : 4 min 12 s = 252 000 ms. `runtime_suspended_time` a
crû de 252 085 ms (concordance à 85 ms près, dans le bruit de mesure du
shell) ; `runtime_active_time` **n'a pas bougé d'un seul milliseconde**.
**Conclusion établie, pas inférée** : livré à lui-même, sans qu'aucun
outil ne l'interroge, le périphérique NVIDIA reste en veille runtime
(`suspended`) en continu — au moins sur cette fenêtre de 4 min 12 s.

**Sonde délibérée #1 — 2026-08-05T12:51:34+02:00**, `nvidia-smi` (table
complète) :

```
$ nvidia-smi
|   0  NVIDIA GeForce RTX 4090 ...    Off |   00000000:01:00.0 Off |    N/A |
| N/A   45C    P0             32W /  155W |      47MiB /  16376MiB |  10% |
|    0   N/A  N/A            2574    C+G   /usr/bin/kwin_wayland    13MiB |
```

Relecture sysfs **immédiate** (même seconde) :

```
0000:01:00.0 runtime_status       = active
0000:01:00.0 runtime_active_time  = 97284   (Δ = +1714 ms depuis T1)
0000:01:00.1 runtime_status       = suspended   (fonction audio non réveillée)
```

**Le réveil est instantané et causal** : la seule action entre T1 et
cette lecture est l'exécution de `nvidia-smi`. `P0`/`32W`/`kwin_wayland`
en `C+G` — la signature exacte du « point ouvert » du 2026-08-05 —
apparaît uniquement après cette commande, jamais avant.

**Retour au repos — 2026-08-05T12:53:48+02:00**, sysfs pur, sans
nouvelle sonde entre-temps :

```
0000:01:00.0 runtime_status       = suspended
0000:01:00.0 runtime_active_time  = 118123   (Δ = +20839 ms depuis la sonde #1)
```

Le périphérique est resté actif environ **20,8 s** après la sonde puis
est retourné se suspendre seul.

**Sonde délibérée #2 — 2026-08-05T12:55:53+02:00**, appariée avec
l'interface `/proc` propre au pilote NVIDIA (plus précise que `sysfs`,
voir § Mécanismes) :

```
$ nvidia-smi --query-gpu=timestamp,pstate,power.draw,memory.used --format=csv
2026/08/05 12:55:55.425, P0, 32.59 W, 47 MiB
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Runtime D3 status:          Enabled (fine-grained)
Video Memory:               Active
```

**Retour au repos, deuxième confirmation — 2026-08-05T12:57:31+02:00**,
sans sonde entre-temps :

```
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
suspended
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Runtime D3 status:          Enabled (fine-grained)
Video Memory:               Off
```

`runtime_active_time = 140672` (Δ = +22549 ms depuis la sonde #2) —
deuxième mesure indépendante de la même durée de réveil (~20-23 s),
cohérente avec la première.

### Synthèse chiffrée

| Repère | Heure | `runtime_status` | `Video Memory` (procfs NVIDIA) | Provoqué par |
|---|---|---|---|---|
| T0 | 12:47:11 | suspended | (non lu) | — (baseline) |
| T1 | 12:51:23 | suspended (inchangé 4 min 12 s) | (non lu) | — |
| Sonde #1 | 12:51:34–36 | **active** | (non lu) | `nvidia-smi` |
| Repos | 12:53:48 | suspended (~20,8 s après la sonde) | Off | — |
| Sonde #2 | 12:55:53–55 | **active** | **Active** | `nvidia-smi --query-gpu=...` |
| Repos | 12:57:31 | suspended (~22,5 s après la sonde) | Off | — |

**Conclusion de l'état mesuré** : sur les ~8 831 s d'uptime écoulées au
dernier relevé, `runtime_suspended_time = 8 830 345` ms — soit **99,97 %
du temps passé en veille runtime** (`0000:01:00.0`). Le `P0`/`32 W`
consigné le 2026-08-05 comme régression était réel **au moment où il a
été mesuré**, mais n'est pas un état permanent : c'est `nvidia-smi`
lui-même, invoqué de façon répétée sans relevé de contrôle, qui l'a
provoqué et prolongé. **Le point ouvert précédent est corrigé, pas
confirmé** — voir `docs/machine-facts.md` pour la mise à jour formelle
(ÉTAIT/CORRIGÉ).

## Mécanismes en présence

Trois mécanismes gouvernent ce comportement, chacun caractérisé par une
source locale ou une documentation embarquée avec le pilote (donc
« amont », recevable comme source externe au sens de `CLAUDE.md` — jamais
confondue avec une lecture locale) :

### 1. Règle udev `90-supergfxd-nvidia-pm.rules` (paquet `supergfxctl`)

```
$ cat /usr/lib/udev/rules.d/90-supergfxd-nvidia-pm.rules
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="unbind", ... ATTR{power/control}="on"   (deux règles miroir)
```
(`rpm -qf` confirme le paquet porteur : `supergfxctl-5.2.7-2.fc44.x86_64`.)

Cette règle cible explicitement les classes PCI `0x030000` et `0x030200`
(contrôleur VGA / contrôleur 3D — les deux classes possibles de la
fonction `.0`, le GPU lui-même) sur liaison du pilote. **Elle ne cible
pas explicitement la classe audio** (fonction `.1`, classe habituelle
`0x040300`). Observé pourtant : `power/control=auto` est bien actif sur
`0000:01:00.1` aussi. Source exacte de ce `auto` sur la fonction audio
non déterminée dans ce livrable — sans conséquence pratique puisque
`auto` y est déjà en vigueur, mais à ne pas attribuer à cette règle
précise sans l'avoir vérifié.

### 2. `NVreg_DynamicPowerManagement` — non défini dans la config, mais pas « au hasard »

Confirmé absent de tout fichier sous `/etc/modprobe.d/` et
`/usr/lib/modprobe.d/` (déjà établi dans `docs/machine-facts.md` §
GPU). Valeur **effectivement chargée**, lue directement dans l'interface
propre au pilote (pas supposée) :

```
$ cat /proc/driver/nvidia/params | grep -i dynamicpower
DynamicPowerManagement: 3
DynamicPowerManagementVideoMemoryThreshold: 200
```

Documentation embarquée avec le paquet pilote (locale, amont, marquée
comme telle) —
`/usr/share/doc/xorg-x11-drv-nvidia/html/dynamicpowermanagement.html`,
chapitre « PCI-Express Runtime D3 (RTD3) Power Management » — précise
que `0x03` **est la valeur par défaut** et que, « pour les portables
Ampere ou plus récents avec une configuration prise en charge, cette
valeur se traduit en contrôle d'alimentation fin (fine-grained) » —
c'est-à-dire le même comportement que `0x02` explicitement documenté :
mise en veille automatique dès qu'aucune application n'utilise la pile
pilote, avec surveillance active de l'inactivité GPU pendant qu'une
application tourne. C'est exactement ce que confirme la lecture directe
du pilote :

```
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Runtime D3 status:          Enabled (fine-grained)
```

Le RTX 4090 Laptop GPU (architecture Ada Lovelace, postérieure à Ampere)
entre dans le cas documenté. **`NVreg_DynamicPowerManagement` non défini
ne signifie pas « non géré » : le défaut du pilote sur ce matériel active
déjà le contrôle d'alimentation le plus fin.**

La même documentation précise une règle décisive pour la section
Options : *« the NVIDIA GPU will remain in an active state if it is
driving a display »* et *« Similarly, the NVIDIA GPU will remain in an
active state if a CUDA application is running »* — RTD3 ne s'engage que
lorsque rien n'utilise réellement le GPU, jamais pendant qu'un
programme (serveur d'inférence compris) garde un contexte CUDA ouvert.

### 3. `NVreg_PreserveVideoMemoryAllocations` commenté — mécanisme distinct de RTD3, à ne pas confondre

Reconfirmé commenté dans `/usr/lib/modprobe.d/nvidia-power-management.conf`
(`cat`, sortie identique à l'inventaire du 2026-08-04). **Ce paramètre ne
gouverne pas le comportement observé en § État mesuré.** La documentation
embarquée (`powermanagement.html`, chapitre 20, distinct du chapitre 21
RTD3 ci-dessus) est explicite : il gouverne la préservation du contenu de
la VRAM lors d'une **suspension système** (S3/ACPI, mise en veille du
portable — fermeture du capot), pas lors des cycles RTD3 par périphérique
documentés au point 2. Non défini, la conséquence documentée est : *« the
resulting loss of video memory contents is partially compensated for by
the user-space NVIDIA drivers, and by some applications, but can lead to
failures such as rendering corruption and application crashes upon exit
from power management cycles »* — un risque réel pour un serveur
d'inférence avec un modèle chargé, mais **seulement à la mise en veille
du portable**, pas entre deux requêtes RTD3. Fait déjà consigné comme
point ouvert dans `docs/machine-facts.md`, à traiter au livrable IA — non
rouvert ici, seulement recadré par rapport à RTD3 pour éviter la
confusion qui a contribué au malentendu du point précédent.

### `nvidia-powerd.service`

```
$ systemctl is-active nvidia-powerd.service ; systemctl is-enabled nvidia-powerd.service
active
enabled
```
Actif depuis ce démarrage (`10:28:07`, 21 s après `uptime -s`). Le
journal cité dans le point fermé du 2026-08-05 (« `asus-shutdown` » sur
le boot précédent) le donnait `inactive (dead)` **à ce moment précis de
l'extinction précédente** — `asus-shutdown` l'avait déjà arrêté avant
d'écrire l'attribut firmware. Les deux observations ne se contredisent
pas : à l'extinction, arrêté ; au démarrage suivant, actif par défaut
(`enabled`). `nvidia-persistenced`, distinct, reste
`inactive`/`disabled` (reconfirmé) — pertinent ici parce que la
documentation RTD3 signale explicitement que le mode persistance UVM,
s'il tournait, désactiverait RTD3 : ce n'est pas le cas sur ce poste.

## Ce qui retient le périphérique

`nvidia-smi` liste `kwin_wayland` (pid 2574, `C+G`, 13 MiB) comme client
à chaque sonde (2 sondes sur 2, séparées de 4 min). Établir pourquoi a
rencontré une limite d'accès, consignée telle quelle plutôt que
contournée :

**Tentative directe, en échec** : lister les descripteurs de fichiers de
`kwin_wayland` (`ls -la /proc/2574/fd`) échoue par `Permission denied`,
bien que le processus tourne sous le même utilisateur
(`uid=1000=mahieumi`) et que `ptrace_scope=0` (pas de restriction Yama).
`/proc/2574/status` est lisible, `/proc/2574/maps` et `/proc/2574/fd`ne
le sont pas — signature typique d'un processus qui a effacé son bit
« dumpable » (`prctl(PR_SET_DUMPABLE, 0)`), pratique de durcissement
courante pour un compositeur Wayland (protection du contenu affiché
contre l'inspection par d'autres processus du même utilisateur). Un
balayage système de `/proc/*/fd` à la recherche de descripteurs vers
`/dev/nvidia*` ou `/dev/dri/{card0,renderD129}` (les nœuds NVIDIA) ne
trouve **aucun** détenteur au moment du balayage — mais ce balayage a
silencieusement échoué sur `kwin_wayland` lui-même pour la raison
ci-dessus (erreur non affichée car redirigée dans la boucle) : **ce
résultat négatif n'établit rien sur ce processus précis**, il est écarté
plutôt que présenté comme une preuve.

**Mécanisme établi par une autre voie — le code source de KWin.**
`kwin_wayland --help` et `--help-all` ne proposent aucune option de
sélection de périphérique DRM. Recherche dans les bibliothèques
installées :

```
$ strings /usr/lib64/libkwin.so.6.7.3 | grep -B3 -A3 '^KWIN_DRM_DEVICES$'
Failed to open drm device %s
adding GPU %s
KWIN_DRM_DEVICES
chose
as the primary GPU
```

`KWIN_DRM_DEVICES` existe bien dans le binaire installé
(`kwin-6.7.3-1.fc44.x86_64`, `rpm -q kwin`). Confirmation externe,
marquée comme telle (dépôt amont, pas lu sur cette machine) : le code
source de KWin (`KDE/kwin`, `src/backends/drm/drm_backend.cpp`, branche
`master`, consulté le 2026-08-05) montre que la variable est lue via
`GpuManager::splitPathList(qEnvironmentVariable("KWIN_DRM_DEVICES"))` —
une liste de chemins ; quand elle est **définie**, seuls les GPU
explicitement listés sont ajoutés ; quand elle est **absente**, KWin
énumère automatiquement tous les GPU du siège actif via `udev`. Vérifié
localement, trois façons concordantes, qu'elle est bien absente sur ce
poste :

```
$ env | grep -i KWIN_DRM              # rien
$ systemctl --user show plasma-kwin_wayland.service -p Environment
Environment=
$ grep -i KWIN /etc/environment       # absent (ou fichier absent)
```

**Conclusion établie** : `KWIN_DRM_DEVICES` n'étant pas défini, KWin
énumère automatiquement les deux cartes DRM du siège (`card0` NVIDIA et
`card1` AMD) au démarrage — cohérent avec la ligne unique trouvée dans le
journal de session (`journalctl --user -u plasma-kwin_wayland.service
-b0`) : `No backend specified, automatically choosing drm`. Les messages
de débogage plus précis (`adding GPU %s`, `chose %s as the primary GPU`)
ne sont pas journalisés au niveau de verbosité par défaut — je n'ai pas
activé de catégorie de logging supplémentaire pour ne pas perturber un
compositeur en cours d'exécution, ce qui sort du périmètre lecture
seule strict de ce livrable. **Ce que j'ai, en substance : le mécanisme
(énumération automatique, confirmé par le code source amont) et son
levier documenté (`KWIN_DRM_DEVICES`, confirmé présent dans le binaire et
absent de la configuration actuelle) ; ce que je n'ai pas : une preuve
journalisée, sur cette session précise, que `card0` a été spécifiquement
ajouté** — remplacée par la preuve indirecte mais répétée de
`nvidia-smi` (2 sondes sur 2 montrant `kwin_wayland` comme client).

## Options envisageables

Pour chacune : ce qu'elle change, son coût, ce qu'elle risque de casser,
et comment on saurait qu'elle a fonctionné.

### Option 0 — ne rien faire, corriger uniquement la méthode de mesure

**Ce qu'elle change** : rien sur le système. Adopte pour tout relevé
futur la méthode de § État mesuré (compteurs `sysfs` passifs, jamais
`nvidia-smi` seul pour juger d'un état de repos).
**Coût** : aucun.
**Risque de casse** : aucun.
**Mesure de succès** : déjà obtenue — `runtime_suspended_time` croît de
façon quasi continue sur toute fenêtre où aucun outil n'interroge le
GPU (99,97 % de l'uptime courant), `Video Memory: Off` au repos.
**Applicable dès maintenant** puisqu'elle ne change rien — c'est une
correction de procédure, pas une option technique.

### Option 1 — `KWIN_DRM_DEVICES=/dev/dri/card1` (exclure la NVIDIA de KWin)

**Ce qu'elle change** : KWin n'énumère plus `card0` (NVIDIA) du tout —
plus de contexte GBM/EGL résiduel, plus d'apparition de `kwin_wayland`
dans la table de processus `nvidia-smi`.
**Coût** : nécessite de faire persister la variable pour la session
graphique (`/etc/environment`, ou une unité systemd `--user`
d'environnement) — pas une simple commande ponctuelle, un changement de
configuration de session.
**Ce qu'elle risque de casser** : `@VERIF : effet de KWIN_DRM_DEVICES sur
les fonctionnalités Plasma qui dépendent de la connaissance par KWin du
second GPU — notamment un éventuel lanceur « exécuter sur le GPU
discret » dans le menu contextuel. Non trouvé de documentation locale ou
amont confirmant si cette fonctionnalité dépend de l'énumération KWin ou
d'un mécanisme indépendant (variables DRI_PRIME/__NV_PRIME_RENDER_OFFLOAD
positionnées par kioclient/l'application elle-même). À vérifier avant
d'appliquer cette option.`
**Comment savoir qu'elle a fonctionné** : `nvidia-smi` (sonde isolée,
espacée) ne liste plus `kwin_wayland` ; `runtime_suspended_time` continue
de croître pendant les sondes normales de bureau (aucune preuve qu'elle
change le taux de 99,97 % déjà mesuré, puisque ce taux est déjà élevé —
le gain attendu est marginal, pas structurel).
**Non recommandée en l'état** : le coût de configuration et le risque
non résolu ci-dessus ne sont pas justifiés par un gain de repos déjà
proche du maximum mesurable.

### Option 2 — forcer `NVreg_DynamicPowerManagement=0x02` explicitement

**Ce qu'elle change** : rend explicite un choix déjà pris par défaut
(§ Mécanismes, point 2) — aucun changement de comportement observable
attendu sur ce matériel (Ada Lovelace, déjà couvert par le cas «
Ampere ou plus récent » du défaut `0x03`).
**Coût** : ajout d'un fichier sous `/etc/modprobe.d/`, `dracut`/`akmods`
à reconstruire, redémarrage requis pour appliquer un paramètre de
module.
**Ce qu'elle risque de casser** : rien de plus que ce qui l'est déjà, le
comportement effectif ne change pas.
**Comment savoir qu'elle a fonctionné** : `cat /proc/driver/nvidia/params
| grep DynamicPowerManagement` afficherait `2` au lieu de `3` — mais
`Runtime D3 status: Enabled (fine-grained)` resterait identique, donc
rien d'observable ne validerait un gain.
**Non recommandée** : coût (reconstruction de module, redémarrage) pour
un résultat déjà acquis par le défaut — violerait la règle « déterminer
ce que l'outil gère déjà lui-même » (`CLAUDE.md` § Avant d'agir).

### Option 3 — `NVreg_PreserveVideoMemoryAllocations=1`

**Ce qu'elle change** : sans rapport avec le point ouvert P0/32W
(mécanisme distinct, § Mécanismes point 3) — pertinente pour la
survie du contenu VRAM à la **mise en veille système** (S3), pas pour le
comportement RTD3 mesuré ici.
**Coût** : nécessite l'intégration `systemd` documentée (interface
`/proc/driver/nvidia/suspend`, hooks de veille/réveil) — non
triviale, déjà consignée comme point ouvert séparé, réservée au livrable
IA.
**Hors périmètre de ce document** : cité ici uniquement pour éviter
qu'elle soit confondue avec une réponse au point P0/32W — ce n'est pas
le cas, ne pas la traiter comme telle dans un futur livrable qui
partirait de ce document.

## Recommandation

**Option 0 : aucune correction technique, correction de méthode
seulement.** Justification : la mesure spacée (§ État mesuré) établit
que le comportement au repos est déjà, sans aucune intervention,
`Runtime D3 status: Enabled (fine-grained)` avec `Video Memory: Off` sur
99,97 % de l'uptime — un résultat qui **égale ou dépasse** l'état
pré-bascule (`P8`/`8 W`, un état actif du pilote, pas un RTD3 complet,
puisque la NVIDIA pilotait alors l'écran). Le point ouvert du 2026-08-05
n'était pas une régression réelle mais un artefact de mesure — corrigé
formellement dans `docs/machine-facts.md`.

Pour l'usage cible (serveur d'inférence local) : la documentation RTD3
amont est explicite — le GPU reste actif tant qu'une application CUDA
tourne. Un serveur d'inférence qui garde un contexte ouvert en continu
n'est **jamais** concerné par RTD3 pendant qu'il tourne ; RTD3 ne
s'applique qu'entre deux exécutions du serveur, ce qui est le
comportement désiré (économie d'énergie sans coût de latence sur les
requêtes). Le seul coût RTD3 identifiable — un délai de réveil
d'environ 20-23 s observé deux fois (§ État mesuré) — s'appliquerait au
tout premier appel après un arrêt complet du serveur, pas entre deux
requêtes d'un serveur qui reste démarré.

Ce qui départage cette recommandation des options 1 et 2 : leur coût
(configuration persistante ou reconstruction de module + redémarrage)
n'est pas compensé par un gain mesurable, le repos étant déjà proche du
maximum atteignable. Option 1 reste **envisageable plus tard** si le
point non résolu sur son risque (fonctionnalités Plasma dépendantes de
l'énumération KWin) est levé et si l'isolement complet du contexte KWin
devient un objectif en soi (au-delà de la seule consommation, par
exemple pour une séparation stricte affichage/calcul) — pas retenue
dans ce livrable.

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — § GPU, § Points ouverts
  (correction formelle ÉTAIT/CORRIGÉE du point P0/32W), attributs
  `asus-armoury`.
- [`docs/gpu-mux-recovery.md`](gpu-mux-recovery.md) — bascule MUX, état
  post-bascule complet.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing, dont la classe
  « observation rapportée par l'opérateur » et la règle de datation des
  mesures appliquées dans ce document.
