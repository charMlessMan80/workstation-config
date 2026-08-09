# Intention d'installation — ce que `site.yml` promet contre ce qu'il livre

**Livrable en lecture seule** (PKG-2, 2026-08-09) : aucun paquet installé
ni retiré, aucune mise à jour appliquée, aucun rôle modifié, aucun
redémarrage. Toutes les commandes de ce document sont des lectures —
`dnf history`, `rpm -q`, `dnf repoquery`, `systemctl`, `rpm -qf`,
`modinfo`, `lsmod` — vérifiées exécutables sans privilège (§ Validation).

**Question posée** : si `site.yml` était rejoué sur une machine neuve
(Fedora 44 KDE nu + les prérequis déjà documentés dans
[`docs/orchestration.md`](orchestration.md) § 0), qu'est-ce qui
manquerait ? PKG-1 a établi le chiffre brut (15 paquets déclarés par les
rôles, 2504 installés) — ce chiffre ne mesure pas la dérive, il mesure
la taille du socle Fedora KDE, déjà hors périmètre d'Ansible par
construction. L'objet de ce livrable est plus étroit : les paquets
**installés délibérément à la main ou par un rôle**, hors socle et hors
mises à jour de masse — c'est là, pas dans `rpm -qa`, que se lit
l'intention.

## 1. Extraire l'intention — historique complet, transaction par transaction

`dnf history list --reverse` (aucun privilège requis, vérifié — `whoami`
→ `mahieumi`, UID 1000, RC=0 sur chaque lecture) donne 41 transactions.
Deux catégories écartées **avant** tout rapprochement, comme demandé,
plus une troisième que j'ajoute et signale comme telle — elle n'est ni
l'une ni l'autre des deux nommées par la demande, mais ne porte pas non
plus de décision propre :

**Transactions 1 et 2 (2026-04-22 13:58/14:00, utilisateur `root`,
`kiwi_dnf5.config`/`image-root`)** — construction de l'image live/
anaconda, antérieures de plus de trois mois à la première transaction de
l'opérateur (transaction 3, 2026-08-04). Fait déjà établi en substance
dans `docs/repositories.md` § 4 (catégorie « média d'installation »,
`from_repo=6ecc2dfaa0dc41e5ad51e007707a786b`) — daté et numéroté ici
pour la première fois par lecture directe de `dnf history info 1`/`2`
plutôt que déduit du seul `from_repo`.

**Transactions 3, 30, 39 — mises à jour de masse** (`dnf update -y`,
1988 paquets ; mise à jour root sans description capturée, 389 paquets,
2026-08-07 ; `dnf update`, 111 paquets, 2026-08-09). Aucune n'exprime un
choix de paquet précis — elles font progresser en bloc ce qui est déjà
installé. Un effet de bord de la transaction 30 mérite d'être signalé à
part : voir § 2.3.

**Ajout signalé, transactions 11, 31, 40** — reconstruction automatique
de `kmod-nvidia` par `akmods`/DKMS après chaque installation de noyau
(`dnf -y install --nogpgcheck --disablerepo=* /tmp/akmods.…/kmod-nvidia-*.rpm`,
exécutée par la machinerie d'`akmod-nvidia`, pas par l'opérateur ni par
un rôle). Ni construction d'image, ni mise à jour de masse au sens
strict de la demande — j'écarte quand même ces trois transactions ici,
en le disant plutôt qu'en les passant sous silence : elles ne portent
aucune décision propre, seulement la trace locale et répétée d'une
décision déjà comptée une fois (`akmod-nvidia`, § 2.2). Détail
technique complet en § 4.2.

**Ce qui reste — 33 transactions, 29 décisions d'installation ou de
retrait distinctes** (certaines transactions couvrent le même paquet,
ex. la paire installation/retrait de `wtype`/`ydotool`, ou une variante
multilib `.i686` de la même intention) :

| # | Date | Ligne de commande | Paquet(s) visé(s) directement |
|---|---|---|---|
| 4 | 2026-08-04 11:40 | `dnf install NetworkManager-tui` | `NetworkManager-tui` |
| 5 | 2026-08-04 12:45 | `dnf install --nogpgcheck --repofrompath terra,… terra-release` | `terra-release` (+ dépendance `terra-gpg-keys`) |
| 6 | 2026-08-04 12:45 | `dnf install asusctl` | `asusctl` |
| 7 | 2026-08-04 12:46 | `dnf swap tuned-ppd power-profiles-daemon --allowerasing` | `power-profiles-daemon` installé, `tuned-ppd` retiré |
| 8 | 2026-08-04 12:47 | `dnf install asusctl-rog-gui` | `asusctl-rog-gui` |
| 9 | 2026-08-04 12:47 | `dnf install <url rpmfusion-free-release> <url rpmfusion-nonfree-release>` | `rpmfusion-free-release`, `rpmfusion-nonfree-release` |
| 10 | 2026-08-04 12:48 | `dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda` | `akmod-nvidia`, `xorg-x11-drv-nvidia-cuda` |
| 12 | 2026-08-04 12:58 | `dnf swap ffmpeg-free ffmpeg --allowerasing` | `ffmpeg` |
| 13 | 2026-08-04 12:58 | `dnf install @multimedia --setopt=install_weak_deps=False --exclude=PackageKit-gstreamer-plugin` | groupe `@multimedia` (`gstreamer1-plugins-bad-freeworld`, `-ugly`, `libheif-freeworld`, `vlc-plugins-freeworld`, entre autres) |
| 14-15 | 2026-08-04 12:59-13:00 | `dnf install mesa-va-drivers-freeworld[.i686]` | `mesa-va-drivers-freeworld` |
| 16-17 | 2026-08-04 13:00-13:01 | `dnf install libva-nvidia-driver[.i686]` | `libva-nvidia-driver` |
| 18 | 2026-08-04 13:01 | `dnf install rpmfusion-free-release-tainted` | `rpmfusion-free-release-tainted` |
| 19 | 2026-08-04 13:01 | `dnf install libdvdcss` | `libdvdcss` |
| 20/41 | 2026-08-04/2026-08-09 | `dnf install cardwire` puis `dnf remove -y cardwire` | `cardwire` — **épisode clos, PKG-1**, exclu du rapprochement § 2 |
| 21 | 2026-08-04 13:53 | (racine, description non capturée par `dnf5` — transaction root sans ligne de commande enregistrée, `Reason: User` sur le paquet de tête) | `ansible-core` |
| 22 | 2026-08-04 13:53 | idem, racine, description non capturée | `supergfxctl` — **épisode distinct, voir § 2.3, retiré depuis par Terra lui-même, pas par l'opérateur** |
| 23 | 2026-08-04 18:14 | `dnf install git` | `git` |
| 24 | 2026-08-04 18:15 | `dnf install claude-code` | `claude-code` |
| 25 | 2026-08-04 19:08 | `dnf install htop` | `htop` |
| 26 | 2026-08-05 14:15 | module Ansible `dnf5` (rôle `gpu_cdi`) | `nvidia-container-toolkit`, `nvidia-container-toolkit-selinux` |
| 27 | 2026-08-06 08:38 | `dnf install -y python3-pip` | `python3-pip` |
| 28 | 2026-08-06 13:31 | `dnf --disablerepo=terra install -y kitty` | `kitty` |
| 29 | 2026-08-06 20:52 | module Ansible `dnf5` (rôle `editor`) | `helix`, `kate` |
| 32 | 2026-08-07 19:27 | module Ansible `dnf5` (rôle `completion`) | `rust`, `cargo` |
| 33 | 2026-08-07 19:34 | module Ansible `dnf5` (rôle `completion`) | `perl-FindBin` |
| 34 | 2026-08-07 19:39 | `dnf install -y --disablerepo=terra perl-IPC-Cmd` | `perl-IPC-Cmd` |
| 35 | 2026-08-07 19:40 | `dnf install -y --disablerepo=terra perl-File-Compare perl-IO-Socket-INET6 perl-Text-Template` | `perl-File-Compare`, `perl-IO-Socket-INET6`, `perl-Text-Template` |
| 36/38 | 2026-08-08 19:54/21:13 | `dnf install -y wtype` puis `dnf remove -y wtype ydotool` | `wtype` — **épisode clos, auto-corrigé par l'opérateur, exclu** |
| 37/38 | 2026-08-08 20:06/21:13 | `dnf install -y ydotool` puis retrait ci-dessus | `ydotool` — **épisode clos, idem** |

Trois épisodes déjà clos par l'opérateur lui-même (`cardwire`,
`wtype`, `ydotool`) ne demandent aucun rapprochement — ils ne sont
plus installés. Ils restent dans cette table par souci d'exhaustivité
de l'historique, pas comptés au § 2.

## 2. Rapprochement

Des 29 décisions du § 1, trois sont des épisodes déjà clos par
l'opérateur lui-même (`cardwire`, `wtype`, `ydotool` — exclus, plus
installés) et une est un cas distinct traité à part parce qu'elle ne
correspond plus à un paquet installé non plus, mais pour une raison
extérieure à l'opérateur (`supergfxctl`, § 2.3). Restent **25 décisions
vivantes**, qui se résolvent en **33 paquets actuellement installés**
une fois les décisions groupées dépliées (`terra-release`+
`terra-gpg-keys` sur une seule décision, par exemple — détail complet
en § 2.5). Ces 33 paquets sont répartis en quatre catégories.

### 2.1 — Reproduit par un rôle (15)

Exactement l'ensemble déclaré par les rôles (PKG-1 avait déjà établi ce
chiffre par lecture des `defaults/`) — confirmé ici par le sens inverse :
chaque paquet de ce chiffre se retrouve dans l'historique `dnf` comme
décision réelle, jamais un doublon.

| Paquet | Rôle | Installé initialement (transaction) |
|---|---|---|
| `terra-release` | `bootstrap` (D10) | 5, manuel puis idempotent depuis |
| `terra-gpg-keys` | `bootstrap` (D10) | 5 (dépendance), déclaré explicitement par le rôle |
| `power-profiles-daemon` | `bootstrap` (D6) | 7, manuel puis idempotent depuis |
| `kitty` | `desktop` | 28, manuel puis idempotent depuis (le rôle n'a jamais eu besoin de l'installer réellement, `kitty` était déjà présent à sa première exécution) |
| `nvidia-container-toolkit` | `gpu_cdi` | 26, par le rôle lui-même |
| `nvidia-container-toolkit-selinux` | `gpu_cdi` | 26, par le rôle lui-même |
| `helix` | `editor` | 29, par le rôle lui-même |
| `kate` | `editor` | 29, par le rôle lui-même |
| `rust` | `completion` | 32, par le rôle lui-même |
| `cargo` | `completion` | 32, par le rôle lui-même |
| `perl-FindBin` | `completion` | 33, par le rôle lui-même |
| `perl-IPC-Cmd` | `completion` | 34, manuel avant que le rôle ne le déclare, idempotent depuis |
| `perl-File-Compare` | `completion` | 35, idem |
| `perl-IO-Socket-INET6` | `completion` | 35, idem |
| `perl-Text-Template` | `completion` | 35, idem |

**Rien à faire.** `jq` (installé par `roles/desktop/`) n'apparaît dans
aucune transaction manuelle ni aucune transaction Ansible : il fait
partie du socle image (transaction 1, 2026-04-22, `Reason: Dependency`)
— le rôle le déclare quand même explicitement (`desktop_jq_package`),
ce qui le rendrait reproductible sur une machine où il manquerait
réellement. Non compté dans les 15 ci-dessus (jamais une décision
d'installation distincte), mentionné pour que l'absence ne soit pas lue
comme un oubli.

### 2.2 — Non reproduit, mais nécessaire (9) — la catégorie critique

Deux sous-groupes : ce que `docs/orchestration.md` § 0 documente déjà
comme prérequis externe (connu, pas une découverte de ce livrable), et
trois paquets dont la nécessité **n'était documentée nulle part avant
ce livrable**.

**Déjà documenté (`docs/orchestration.md` § 0)** :

| Paquet | Pourquoi une reconstruction en manquerait |
|---|---|
| `git` | Cloner ce dépôt (§ 0.5) — rien ne peut s'exécuter sans |
| `ansible-core` | Exécuter `site.yml` lui-même (§ 0.5, D3a) |
| `rpmfusion-free-release` | Prérequis explicite avant tout rôle (§ 0.3) |
| `rpmfusion-nonfree-release` | Idem — porte le pilote NVIDIA (§ 0.3) |
| `akmod-nvidia` | Pilote NVIDIA propriétaire, prérequis de `roles/gpu_cdi/` (§ 0.4) |
| `xorg-x11-drv-nvidia-cuda` | Idem, requis par la garde `nvidia-smi` de `roles/gpu_cdi/` |

**Nouveau, établi par ce livrable — jamais documenté avant** :

- **`asusctl`** — non déclaré par `roles/bootstrap/` (qui installe
  `terra-release`/`terra-gpg-keys` mais pas `asusctl` lui-même), non
  requis par aucun autre paquet installé (`rpm -q --whatrequires
  asusctl` → vide).
  **[MOTIF CORRIGÉ le 2026-08-09, GPU-4]** La version précédente de ce
  paragraphe attribuait la nécessité d'`asusctl` à l'application réelle
  de `gpu_mux_mode` au redémarrage via `asus-shutdown.service` — **plus
  affirmatif que ce qui a été établi**. Un livrable antérieur
  (révocation partielle de D2ter, `docs/machine-facts.md` § Décisions)
  a consigné explicitement que, le 2026-08-05, l'écriture directe dans
  sysfs et la file d'`asusd` (jamais purgée depuis une tentative
  antérieure) portaient **la même valeur cible** — et que **laquelle
  des deux a effectivement produit le changement matériel observé est
  indéterminable a posteriori**. Présenter `asus-shutdown.service`
  comme LE mécanisme d'application allait au-delà de ce que ce fait
  établit.

  Le motif qui tient, sourcé sans dépendre de ce point indéterminé :
  `asusd` (fourni par `asusctl`) gère les profils d'alimentation et les
  courbes de ventilation propres à ce matériel — vérifié directement,
  pas supposé : `/etc/asusd/asusd.ron` porte
  `platform_profile_on_battery`, `platform_profile_on_ac`,
  `platform_profile_linked_epp` ; `asusctl profile` et `asusctl
  fan-curve` existent comme sous-commandes dédiées (`asusctl --help`).
  **D6** (`docs/machine-facts.md` § Décisions) a choisi
  `power-profiles-daemon` plutôt que `tuned-ppd` explicitement « pour
  sa cohérence avec `asusd` » — ce motif présuppose `asusd` présent.
  Sur une machine neuve sans `asusctl`, `roles/bootstrap/` installerait
  et activerait `power-profiles-daemon` sans erreur (D6 ne dépend
  d'aucun autre paquet pour s'exécuter) — mais **sans le partenaire
  ASUS pour lequel il a été choisi plutôt qu'un autre gestionnaire de
  profils** : la prémisse de D6 ne tiendrait plus, sans qu'aucune garde
  actuelle ne le signale. Aucun test, dans `roles/bootstrap/` ni
  ailleurs, ne vérifie la cohérence entre `power-profiles-daemon` et
  `asusd` — silencieux au même titre que l'était l'ancien motif, par un
  mécanisme différent et mieux sourcé.
- **`claude-code`** — non déclaré par aucun rôle, non requis par aucun
  paquet installé. `roles/desktop/tasks/main.yml` contient une garde
  explicite (« Échouer bruyamment si une commande de la session de
  démarrage est absente du PATH », `ansible.builtin.assert: that: item.rc
  == 0`) qui vérifie la présence de `claude` sur le `PATH` **avant**
  de déployer la disposition de démarrage kitty. Sur une machine neuve,
  cette garde **arrêterait `site.yml`** dès l'exécution de `roles/desktop/`
  — échec immédiat et explicite, pas une dégradation silencieuse.
- **`htop`** — même garde, même mécanisme, même conclusion :
  `desktop_kitty_htop_cmd: htop` (`roles/desktop/defaults/main.yml`),
  vérifié par la même assertion. `bash`, le troisième nom vérifié par
  cette garde (`desktop_kitty_shell_cmd`), fait partie du socle Fedora
  de base — pas un gap.

**Asymétrie à noter** : `claude-code`/`htop` échouent tôt et bruyamment
(avant tout redémarrage, avant tout déploiement, garde `assert`
explicite) ; l'absence d'`asusctl` ne ferait échouer aucune garde
existante — `power-profiles-daemon` s'installerait et fonctionnerait
(D6 ne dépend d'aucun paquet pour s'exécuter), seule sa cohérence avec
`asusd` (motif même de D6) ne tiendrait plus, sans qu'aucun test actuel
ne le détecte, ni tôt ni tard. Les trois sont réellement nécessaires,
mais la nécessité d'`asusctl` est de nature différente : pas une
exécution qui casse, une prémisse de décision qui ne tient plus en
silence.

### 2.3 — Non reproduit, non nécessaire (0, à date — un cas déjà refermé de lui-même)

Aucun paquet actuellement installé ne correspond à ce profil parmi les
25 décisions vivantes recensées — `cardwire` en était l'unique
représentant, retiré en PKG-1.

**Fait signalé, pas silencieusement corrigé** (`CLAUDE.md` § Avant
d'agir — une découverte qui contredit un fait déjà documenté se
signale) : `supergfxctl` (transaction 22, installé le 2026-08-04 via le
mécanisme `command-not-found` de `dnf`, jamais un choix de conception —
fait déjà établi dans `CLAUDE.md` § Avant d'agir) **n'est plus installé
sur ce poste** :
```
$ rpm -q supergfxctl
package supergfxctl is not installed
```
Disparu pendant la mise à jour de masse du 2026-08-07 (transaction 30,
écartée du rapprochement ci-dessus comme mise à jour de masse — mais
son contenu mérite d'être lu) :
```
$ dnf history info 30 | grep -i supergfx
  Replaced supergfxctl-0:5.2.7-2.fc44.x86_64   User   @System
```
Cause établie, pas supposée : `terra-obsolete-44-7`, installé dans la
même transaction, déclare `supergfxctl < 5.2.7-3` parmi ses
`Obsoletes` (`dnf repoquery --obsoletes terra-obsolete`) — Terra a
retiré son propre paquet `supergfxctl` du dépôt (probablement remplacé
par le projet `gnome-shell-extension-gpu-switcher-supergfxctl`, visible
dans la même liste d'obsolescence, non vérifié davantage) et `dnf`
l'a désinstallé silencieusement au passage d'une mise à jour de
routine, **sans message de conflit visible** — contrairement à
`cardwire`, dont le blocage a rendu la dérive détectable. Exactement le
risque que ce livrable cherche à mesurer, matérialisé une fois de plus,
cette fois sans qu'aucune action de l'opérateur ni d'aucun rôle n'y
soit pour quoi que ce soit.

**Conséquence directe, hors périmètre strict de ce livrable (PKG-2)** :
`docs/repositories.md` § 4/§ 9 (PKG-1, commit `996fdc7`) affirmait
toujours « `terra` : 5 paquets, dont `asusctl` » en laissant entendre
que les quatre autres sont inchangés depuis avant le retrait de
`cardwire` — **faux** : l'un des cinq est `terra-obsolete` (installé
2026-08-07), pas `supergfxctl` (parti le même jour). Le compte
« 5 » restait juste par coïncidence numérique, sa composition
détaillée ne l'était plus. Non corrigée dans ce livrable-ci —
`docs/repositories.md` était hors périmètre de PKG-2. **[CORRIGÉ le
2026-08-09, GPU-4]** `docs/repositories.md` § 4/§ 10 met désormais la
composition exacte, et nomme le fait de méthode qui en découle
(surface de risque de `terra` non nommée en D10).

### 2.4 — Non reproduit, délibérément hors périmètre (9)

Confort personnel ou outillage de développement du dépôt lui-même —
légitime, à condition d'être **dit**. Aucun signal de dérive (pas de
conflit, pas de service jamais démarré) comparable à `cardwire` n'a été
trouvé pour aucun de ces neuf paquets ; ce n'est pas la même situation,
elle appelle une confirmation d'intention, pas un retrait.

| Paquet | Nature | Note |
|---|---|---|
| `NetworkManager-tui` | Confort — frontend TUI pour NetworkManager | Non requis par aucun paquet, aucun service propre |
| `asusctl-rog-gui` | Confort — interface graphique d'`asusctl` | Dépend d'`asusctl` (§ 2.2), pas l'inverse — `asusctl` fonctionne sans lui |
| `python3-pip` | Outillage de développement **de ce dépôt** | Installé pour la chaîne `ansible-lint` en environnement virtuel (D3a, `docs/ansible-chain.md`) — nécessaire pour *valider* ce dépôt, pas pour ce que `site.yml` *déploie* ; distinction délibérée, pas un oubli |
| `ffmpeg` | Confort — chaîne multimédia | Transaction 12, jamais discutée avant ce livrable |
| `@multimedia` (groupe) | Confort — chaîne multimédia | Transaction 13, `gstreamer1-plugins-bad-freeworld`/`-ugly`, `libheif-freeworld`, `vlc-plugins-freeworld` entre autres |
| `mesa-va-drivers-freeworld` | Confort — accélération vidéo matérielle | Transactions 14-15 |
| `libva-nvidia-driver` | Confort — VA-API sur pilote NVIDIA propriétaire | Transactions 16-17 — coexistence avec `mesa-va-drivers-freeworld` déjà notée comme point ouvert non tranché (`docs/machine-facts.md` § Points ouverts) |
| `rpmfusion-free-release-tainted` | Confort — dépôt hébergeant les codecs à restriction légale | Transaction 18 |
| `libdvdcss` | Confort — décryptage DVD | Transaction 19 |

**Cas déjà connu à traiter explicitement, comme demandé** : la chaîne
multimédia (transactions 12-19) n'est reproduite par aucun rôle et
n'avait jamais été discutée avant ce livrable — elle l'est maintenant.
Aucune recommandation de retrait : rien n'indique une dérive, ces
paquets servent leur fonction déclarée (lecture vidéo/DVD, accélération
matérielle) sur un poste de développement personnel. Ce que ce
livrable ajoute, c'est le fait que **si `site.yml` était rejoué sur une
machine neuve, cette chaîne resterait absente** tant que l'opérateur ne
la réinstalle pas à la main — à nommer explicitement comme un choix
assumé plutôt qu'un oubli qui ressurgirait un jour sous une autre
forme.

### 2.5 — Somme

15 (§ 2.1) + 9 (§ 2.2) + 0 (§ 2.3) + 9 (§ 2.4) = **33** — égal au
chiffre annoncé en tête du § 2. Reconstruit dans l'autre sens, à partir
des 25 décisions vivantes du § 1 (29 décisions au total, moins les
trois épisodes clos et le cas `supergfxctl` traité à part), pour
vérifier que rien n'a été oublié ni compté en trop :

- **18 décisions à un seul paquet** : `NetworkManager-tui`, `asusctl`,
  `power-profiles-daemon` (`tuned-ppd` est retiré dans la même
  décision, il ne s'ajoute pas au compte des paquets *installés*),
  `asusctl-rog-gui`, `ffmpeg`, `@multimedia`, `mesa-va-drivers-freeworld`,
  `libva-nvidia-driver`, `rpmfusion-free-release-tainted`, `libdvdcss`,
  `ansible-core`, `git`, `claude-code`, `htop`, `python3-pip`, `kitty`,
  `perl-FindBin`, `perl-IPC-Cmd` → **18 paquets**.
- **6 décisions à deux paquets** : `terra-release`+`terra-gpg-keys`,
  `rpmfusion-free-release`+`rpmfusion-nonfree-release`,
  `akmod-nvidia`+`xorg-x11-drv-nvidia-cuda`,
  `nvidia-container-toolkit`+`nvidia-container-toolkit-selinux`,
  `helix`+`kate`, `rust`+`cargo` → **12 paquets**.
- **1 décision à trois paquets** : `perl-File-Compare`+
  `perl-IO-Socket-INET6`+`perl-Text-Template` → **3 paquets**.

18 + 6 + 1 = **25 décisions** (concorde avec le § 1). 18 + 12 + 3 =
**33 paquets** (concorde avec la somme des quatre catégories
ci-dessus). Les deux sens de calcul se rejoignent exactement — rien
d'ajusté après coup. Les trois épisodes clos (`cardwire`, `wtype`,
`ydotool`) et le cas `supergfxctl`/`terra-obsolete` (§ 2.3) restent
hors des deux chiffres, pour la raison donnée à chaque endroit.

## 3. Réponse à la question posée

**Si `site.yml` était rejoué sur une machine neuve (Fedora 44 KDE +
prérequis § 0 de `docs/orchestration.md` déjà en place), qu'est-ce qui
manquerait ?**

Trois paquets, avec deux modes d'échec de nature différente :

1. **`claude-code`** et **`htop`** — `roles/desktop/` s'arrêterait
   immédiatement et bruyamment, avant tout redémarrage, avant tout
   déploiement de configuration. Le message serait exact et actionnable
   (« la commande X est absente du PATH »). C'est le comportement voulu
   par la garde — mais la garde protège contre un échec silencieux
   *après* déploiement, pas contre l'absence du paquet lui-même :
   `site.yml` resterait bloqué à ce point tant que l'opérateur n'aurait
   pas installé les deux à la main.
2. **`asusctl`** — **[MOTIF CORRIGÉ le 2026-08-09, GPU-4]** aucune
   garde ne casserait, ni tôt ni tard : `roles/bootstrap/` installerait
   `power-profiles-daemon` (D6) sans erreur, `power-profiles-daemon`
   fonctionnerait. Ce qui manquerait est plus discret qu'un échec de
   playbook — la cohérence avec `asusd` que D6 donne explicitement
   comme motif de ce choix (`docs/machine-facts.md` § Décisions)
   n'aurait plus de sens, `asusd` étant absent, sans qu'aucun test
   actuel ne le remarque. Pas un blocage de `site.yml` comme les deux
   précédents — une prémisse de décision silencieusement caduque.

**Ce que je recommanderais de traiter en premier** : ajouter `asusctl`
à la liste de paquets installés par `roles/bootstrap/` (aux côtés de
`terra-release`/`terra-gpg-keys`, même dépôt, même moment logique de
la séquence) — c'est le seul des trois dont rien, dans l'état actuel
du dépôt, ne remarquerait l'absence : ni un `assert` qui casse (comme
`claude-code`/`htop`), ni un `post_tasks` qui rattrape après coup.
Traiter `claude-code`/`htop` ensuite, par simple
cohérence de méthode : soit les déclarer explicitement dans un rôle
(probablement `desktop`, aux côtés de `kitty`/`jq`), soit documenter
sciemment leur absence de `site.yml` comme un choix assumé — les deux
sont légitimes, mais l'un des deux doit être choisi plutôt que laissé
implicite comme aujourd'hui.

**Rien d'autre n'est critique.** La chaîne multimédia (§ 2.4) est un
choix de confort réel mais non caché une fois nommé ici ; les six
paquets déjà documentés en § 0 d'`orchestration.md` sont un état
connu, pas une découverte. Un rapport qui présenterait la chaîne
multimédia ou `python3-pip` comme urgents inventerait une gravité
qu'ils n'ont pas.

## 4. Deux catégories annexes

### 4.1 — Paquets du média d'installation (~1170)

Nommée en PKG-1 (`docs/repositories.md` § 4), confirmée ici depuis
l'historique plutôt que déduite du seul `from_repo` : ces paquets
proviennent des transactions 1/2 (§ 1 ci-dessus), la construction de
l'image live/anaconda — **pas** une décision de l'opérateur ou d'un
rôle, la toute première chose que ce dépôt écarte explicitement du
rapprochement. **Ce n'est pas un problème pour la reconstructibilité,
et c'est vérifiable plutôt que supposé** : ces ~1170 paquets sont
exactement ce qu'une installation Fedora 44 KDE Plasma standard
apporte d'elle-même (transaction 2 : `@kde-desktop-environment`,
`@kde-apps`, `@multimedia`, `@firefox`, `@standard`, entre autres
groupes explicitement nommés dans sa ligne de commande, lue au § 1) —
une réinstallation de Fedora 44 KDE sur une machine neuve les
ramènerait d'elle-même, par construction de l'image, sans qu'aucun
rôle n'ait à les nommer. Le seul prérequis pour que ce raisonnement
tienne est nommé dans `docs/orchestration.md` § 0.1 (« Fedora 44
installé, KDE Plasma ») — déjà posé comme point de départ de toute
reconstruction, pas un gap supplémentaire.

### 4.2 — Paquets `@commandline` (5)

| Paquet | Reproduit par un rôle ? |
|---|---|
| `kmod-nvidia` (×3, une par version de noyau) | **Non** — sous-produit automatique d'`akmods`/DKMS déclenché par chaque mise à jour de noyau, tant que `akmod-nvidia` est présent (§ 2.2, déjà nécessaire, déjà documenté). Rien à reproduire séparément : reconstruire `akmod-nvidia` + redémarrer suffit à faire réapparaître ces artefacts, ils ne sont pas une décision distincte. |
| `rpmfusion-free-release` | **Non** — prérequis externe § 0.3 d'`orchestration.md` (§ 2.2 ci-dessus), jamais `bootstrap` ni `gpu_cdi` |
| `rpmfusion-nonfree-release` | **Non** — idem |

Aucun des cinq n'est reproduit par `bootstrap` ni par `gpu_cdi` — les
deux `rpmfusion-*-release` parce qu'ils sont un prérequis **avant**
toute exécution d'Ansible par construction (§ 0.3), les trois
`kmod-nvidia` parce qu'ils ne sont jamais une décision en soi, seulement
la trace locale d'une reconstruction automatique déjà couverte par la
nécessité d'`akmod-nvidia`.

## Voir aussi

- [`docs/repositories.md`](repositories.md) § 4, § 9 — PKG-1, catégorie
  « média d'installation », composition de `terra` **actuellement
  périmée** sur un point précis (§ 2.3 ci-dessus), correction différée.
- [`docs/orchestration.md`](orchestration.md) § 0 — prérequis externes
  déjà documentés avant toute exécution de `site.yml`.
- [`docs/machine-facts.md`](machine-facts.md) — D23 (PKG-1), D2bis/D2ter
  (mécanisme d'application de `gpu_mux_mode`), nouveau point ouvert
  ajouté par ce livrable (§ Points ouverts).
- [`roles/desktop/tasks/main.yml`](../roles/desktop/tasks/main.yml) —
  garde d'existence de `claude`/`htop`/`bash` sur le `PATH`.
- [`roles/gpu_mux/tasks/main.yml`](../roles/gpu_mux/tasks/main.yml) —
  écriture directe dans `current_value`, abandon d'`asusctl armoury set`.

## Validation

**Aucune commande de ce livrable n'a requis de privilège** — vérifié
avant, pas supposé : `whoami` → `mahieumi` (UID 1000) sur l'ensemble de
la session, `dnf history list`/`dnf history info` (jusqu'à la
transaction 41 incluse) s'exécutent avec un code de retour 0 sans
`sudo`. La voie non privilégiée existe et a été utilisée du début à la
fin — colonne « tentative sans privilège : résultat » du tableau
ci-dessous documentée en conséquence.

| # | Commande | Tentative sans privilège : résultat | Motif |
|---|---|---|---|
| — | *(aucune action privilégiée dans ce livrable)* | Toutes les lectures (`dnf history`, `rpm -q`/`-qi`/`-qf`, `dnf repoquery`, `systemctl cat`/`is-enabled`/`is-active`, `modinfo`, `lsmod`) ont réussi sans `sudo`, vérifié systématiquement plutôt que supposé | — |

**Commandes non modifiantes, exhaustives** : `dnf history list
--reverse` ; `dnf history info 1` à `41` (41 appels) ; `rpm -q`/`rpm -qi`
sur chaque paquet nommé dans ce document ; `dnf repoquery --installed
--qf ...` (from_repo, plusieurs formes) ; `dnf repoquery --obsoletes
terra-obsolete` (lecture réseau/cache, dépôts déjà activés, aucune
écriture) ; `systemctl cat`/`is-enabled`/`is-active`/`status` sur
`asusd`, `asus-shutdown`, `switcheroo-control`, `supergfxd` (introuvable,
confirmé) ; `rpm -qf` sur les chemins de service et de module noyau ;
`modinfo asus_armoury` ; `lsmod | grep asus` ; `grep`/lecture directe de
`roles/*/tasks/*.yml`, `roles/*/defaults/main.yml`,
`roles/*/README.md`, `docs/orchestration.md`. **Aucune commande sans
sortie, aucun code de retour non nul en dehors des `rpm -q`/`rpm -qf`
attendus « non installé » (RC=1, sortie explicite, jamais silencieuse),
aucune invite déclenchée** — en particulier `dnf repoquery --obsoletes
terra-obsolete`, seule commande de ce livrable touchant réellement
Terra (résolution de dépendances contre le dépôt), n'a déclenché aucune
invite d'import de clé (clé déjà connue du trousseau utilisateur depuis
PKG-1/BUR-1, `docs/repositories.md` § 3).

**Confirmations finales** : `rpm -qa | wc -l` → 2504, inchangé depuis
PKG-1 ; aucun paquet installé ni retiré par ce livrable ; aucune mise à
jour appliquée (`dnf update` non rejoué) ; aucun rôle Ansible modifié ;
`terra` toujours activé (`dnf repo list --enabled` inclut `terra`,
revérifié) ; `sudoers`, `terra.repo`, `/etc/cdi/`, `gpu_mux_mode`,
`kwinrulesrc`, `site.yml` non touchés ; aucun redémarrage, aucune
déconnexion.
