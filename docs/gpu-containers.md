# Accès GPU depuis Podman rootless — résolution en lecture seule

**Ce document ne configure rien.** Toutes les commandes citées ont été
exécutées en lecture seule le 2026-08-05, sur ce poste, après la
résolution alimentation dGPU
([`docs/dgpu-power.md`](dgpu-power.md)). Aucun paquet n'a été installé,
aucune image téléchargée, aucun conteneur lancé, aucune clé GPG importée,
aucun dépôt modifié. La décision entre les options du § 6 viendra dans un
livrable séparé.

Objectif : rendre la RTX 4090 utilisable depuis des conteneurs Podman
rootless — préalable commun aux Execution Environments Ansible (D3) et au
serveur d'inférence local.

## 1. Ce que Podman gère déjà nativement — établi avant toute conclusion

**Podman a un support CDI natif**, confirmé par sa propre documentation,
pas supposé :

```
$ podman --version
podman version 5.8.4
$ zcat /usr/share/man/man1/podman.1.gz | sed 's/\\f[A-Z]//g'
[.SH GLOBAL OPTIONS]
.SS --cdi-spec-dir=path
The CDI spec directory path (may be set multiple times). Default path is /etc/cdi.
```

`podman info --format json | grep -i cdi` ne renvoie rien (`exit=1`) : le
support CDI n'apparaît pas comme un champ déclaré dans `podman info`,
c'est un mécanisme intégré au parseur de `--device`, pas une capacité
qui se signale. `podman run --help` confirme deux options pertinentes,
sans mention explicite de CDI dans leur description :

```
--device stringArray   Add a host device to the container
--gpus strings         GPU devices to add to the container ('all' to pass all GPUs)
                       Currently only Nvidia devices are supported.
```

Répertoires CDI scannés — **lus dans la documentation embarquée, pas
supposés** :

```
$ man -w containers.conf
/usr/share/man/man5/containers.conf.5.gz
$ zcat /usr/share/man/man5/containers.conf.5.gz | grep -B2 -A2 cdi
cdi_spec_dirs=["/etc/cdi", "/var/run/cdi", ...]
Directories to scan for CDI Spec files.
```

Confirmé par le fichier de configuration système lui-même (valeurs
commentées = valeurs par défaut compilées, pas de surcharge active) :

```
$ grep -A3 'Directories to scan for CDI' /usr/share/containers/containers.conf
# Directories to scan for CDI Spec files.
#cdi_spec_dirs = [
#  "/etc/cdi",
#  "/var/run/cdi",
#]
```

**Aucune surcharge utilisateur** : `~/.config/containers/containers.conf`
n'existe pas (`ls` → `No such file or directory`).

**Aucune spécification CDI présente sur ce poste, dans aucun des deux
emplacements scannés :**

```
$ ls -la /etc/cdi/       → No such file or directory
$ ls -la /var/run/cdi/   → No such file or directory
$ ls -la /run/cdi        → No such file or directory (alias de /var/run)
```

**Conclusion établie, pas supposée** : Podman 5.8.4 sait consommer une
spécification CDI nativement dès qu'elle existe dans l'un de ces deux
répertoires — aucun composant tiers n'est nécessaire pour cette partie.
Ce qui manque réellement n'est **pas un consommateur CDI, mais la
spécification elle-même** : rien ne l'a jamais générée sur ce poste (voir
§ 2).

Reste des faits d'inventaire, sourcés :

```
$ podman info   # extraits pertinents
version.APIVersion: 5.8.4
host.security.rootless: true
host.ociRuntime.name: crun (1.28)
store.graphDriverName: overlay
```
Mode rootless confirmé (`security.rootless: true`), runtime OCI `crun`
1.28, pilote de stockage `overlay` sur `btrfs` — cohérent avec
l'inventaire `docs/machine-facts.md` § Conteneurs, non reproduit ici.

## 2. NVIDIA Container Toolkit — disponibilité et provenance

### Dépôts actuellement activés (Terra désactivé pour cette requête)

```
$ dnf --disablerepo=terra --assumeno list --available \
    '*nvidia-container*' '*libnvidia-container*' '*nvidia-ctk*' '*cdi*'
[...]
cdist, jakarta-cdi, libcdio, nbdkit-cdi-plugin, vcdimager, ...
```
**Aucun résultat pertinent.** Les seules correspondances au motif
`*cdi*` sont des paquets sans rapport (Configuration Distribution
`cdist`, le framework Java `jakarta-cdi`, la bibliothèque CD-ROM
`libcdio`) — homonymie du sigle, pas une piste. `nvidia-container-toolkit`,
`libnvidia-container` et `nvidia-ctk` sont absents des dépôts Fedora,
RPM Fusion et `updates` actuellement activés.

### Terra (dépôt tiers déjà activé, clé en point non résolu — traité avec la même prudence)

```
$ dnf --repo=terra --assumeno list --available '*nvidia*'
Terra 44   100%  ...
>>> repomd.xml GPG signature verification error: Signing key not found
Terra 44   100%  ...
>>> repomd.xml GPG signature verification error: Signing key not found
Repositories loaded.
No matching packages to list
exit=1
```
**Deux faits distincts à ne pas confondre** :
1. **Aucun paquet NVIDIA dans Terra** — `*nvidia*`, `*container-toolkit*`
   et `*container-runtime*` ne renvoient rien.
2. **La commande n'a déclenché aucune invite d'import de clé** —
   `--assumeno` en a fait un avertissement (« Signing key not found »)
   au lieu d'un prompt, comportement volontaire de cette série pour ne
   jamais risquer un import accidentel. Cet avertissement, obtenu en
   **session non privilégiée**, est cohérent avec le point ouvert déjà
   consigné dans `docs/machine-facts.md` (asymétrie d'import selon le
   contexte privilégié/non privilégié) — il le corrobore, il ne le
   contredit pas.
`exit=1` sur une recherche sans résultat : comportement attendu de
`dnf list`, pas un échec de la commande elle-même.

### Dépôt tiers nécessaire — NVIDIA officiel, avec un problème de confiance documenté par la communauté

Aucun dépôt actuellement activé sur ce poste ne fournit ce paquet.
Sourcé par recherche externe, marquée comme telle (jamais confondue avec
une lecture locale) :

- **Dépôt officiel NVIDIA**
  (`https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo`,
  cité dans la documentation d'installation NVIDIA, `install-guide.html`,
  consultée le 2026-08-05). **Problème de confiance documenté par la
  communauté Fedora**, pas par cette série : un fil de discussion
  Fedora (« Trustworthy Nvidia Container Toolkit Installation »,
  `discussion.fedoraproject.org`, consulté le 2026-08-05) rapporte un
  avertissement `Warning: skipped OpenPGP checks for 4 packages from
  repository: nvidia-container-toolkit` sur ce dépôt précis, et mentionne
  « a series of longstanding issues with their certificate management ».
  **Ce dépôt a donc un ancrage de confiance objectivement plus faible que
  Terra** (dont le `gpgcheck` fonctionne, seule sa visibilité en session
  non privilégiée est en cause) — à traiter avec plus de prudence que
  Terra, pas la même.
- **Alternative signalée dans ce même fil** : un dépôt COPR maintenu par
  le groupe d'intérêt Fedora AI/ML
  (`copr.fedorainfracloud.org/coprs/g/ai-ml/nvidia-container-toolkit/`).
  Infrastructure COPR = infrastructure Fedora officielle, mais **signature
  par une clé propre au propriétaire du COPR, pas la clé Fedora
  principale** — un dépôt tiers au sens de la confiance, même hébergé par
  Fedora. Contenu exact et détail de la clé **non vérifiés dans cette
  série** : la page du dépôt a renvoyé un mur anti-robot (Anubis/Techaro)
  à la tentative de consultation, aucune clé ni contenu de paquet n'a pu
  être lu. `@VERIF : contenu exact, mainteneur et clé de signature du
  COPR ai-ml/nvidia-container-toolkit — à vérifier via 'dnf copr list'
  ou une consultation directe avant tout ajout, la page web n'ayant pas
  répondu lors de cette série.`
- **Alternative non creusée dans cette série** : le dépôt CUDA officiel
  NVIDIA (mentionné dans le même fil comme ayant potentiellement une
  meilleure hygiène de signature) — hors périmètre de cette résolution,
  à évaluer si l'option toolkit est retenue.

### Ce qu'il faudrait réellement installer, si cette voie est retenue

Point établi par la documentation NVIDIA (`cdi-support.html`, consultée
le 2026-08-05, externe) directement pertinent pour éviter la
sur-couverture : NVIDIA distingue **deux paquets**, pas un seul —
`nvidia-container-toolkit` (complet, inclut le hook d'exécution
`nvidia-container-runtime` et ses dépendances transitives) et
`nvidia-container-toolkit-base` (« avoids installing the container
runtime hook and transitive dependencies » — fournit uniquement
`nvidia-ctk`, l'utilitaire qui génère la spécification CDI). **Puisque
Podman consomme CDI nativement (§ 1), le hook d'exécution du paquet
complet est une redondance pour cet usage** — seul `nvidia-ctk` (paquet
`-base`) serait nécessaire pour produire la spécification que Podman
sait déjà lire seul. Ce distinguo n'a pas pu être vérifié contre les
paquets réels d'un dépôt précis dans cette série (aucun dépôt fournissant
l'un ou l'autre n'a été identifié comme activable sans réserve — voir
ci-dessus) ; il détermine néanmoins déjà la taille de l'option à évaluer
au § 6.

## 3. Périphériques et mappage rootless

```
$ ls -l /dev/nvidia*
crw-rw-rw-  root root  195,   0  /dev/nvidia0
crw-rw-rw-  root root  195, 255  /dev/nvidiactl
crw-rw-rw-  root root  195, 254  /dev/nvidia-modeset
crw-rw-rw-  root root  507,   0  /dev/nvidia-uvm
crw-rw-rw-  root root  507,   1  /dev/nvidia-uvm-tools
/dev/nvidia-caps/nvidia-cap1  cr--------  root root
/dev/nvidia-caps/nvidia-cap2  cr--r--r--  root root

$ ls -l /dev/dri/
crw-rw----+ root video  226, 0  card0     (NVIDIA)
crw-rw----+ root video  226, 1  card1     (AMD)
crw-rw-rw-  root render 226,128 renderD128 (AMD)
crw-rw-rw-  root render 226,129 renderD129 (NVIDIA)
```

**Fait central pour le rootless** : tous les nœuds nécessaires à un
usage **calcul** (`nvidia0`, `nvidiactl`, `nvidia-uvm`, `nvidia-uvm-tools`,
`renderD129`) sont en mode `666` — accessibles à n'importe quel
utilisateur, sans appartenance à un groupe. Aucune règle `udev` ni
`tmpfiles.d` locale n'explique ce mode pour `/dev/nvidia*` (recherché
dans `/usr/lib/udev/rules.d/` et `/usr/lib/tmpfiles.d/`, aucune
correspondance trouvée hormis `10-nvidia.rules`, qui ne fait que
déclencher `nvidia-fallback.service`, sans toucher aux permissions) —
très probablement un mode de création par défaut du pilote lui-même, non
confirmé par lecture de code (pilote propriétaire, binaire). Sans
conséquence pratique ici puisque le mode observé, quelle qu'en soit
l'origine, est déjà celui qui convient.

`card0`/`card1` (nœuds « master », KMS/affichage — **pas nécessaires pour
du calcul**) sont restreints au groupe `video`, dont `mahieumi` ne fait
**pas** partie (`id` → `groups=1000(mahieumi),10(wheel)`). Un accès
existe néanmoins par ACL nominative :

```
$ getfacl /dev/dri/card0
user:mahieumi:rw-
$ udevadm info /dev/dri/card0 | grep TAGS
TAGS=:switcheroo-discrete-gpu:uaccess:master-of-seat:seat:
```
Le tag `uaccess` confirme le mécanisme : `systemd-logind` accorde cette
ACL **pour la durée de la session active sur le siège** (`loginctl` :
`Seat=seat0`, `Type=wayland`, `Active=yes`), pas de façon permanente.
**Distinction à ne pas perdre** : un conteneur lancé depuis un service
`systemd --user` sans session active de siège n'hériterait pas
nécessairement de cette ACL sur `card0`/`card1` — sans conséquence pour
un usage calcul pur (qui n'a besoin que des nœuds `666` ci-dessus), mais
pertinent si un usage futur nécessitait le nœud `card0`.

```
$ grep '^mahieumi:' /etc/subuid /etc/subgid
/etc/subuid:mahieumi:524288:65536
/etc/subgid:mahieumi:524288:65536
```
Plage `subuid`/`subgid` présente et déjà en usage actif — confirmée par
`podman info` (`host.idMappings`, deux entrées uid/gid correspondant à
cette même plage `524288:65536`). Rien à établir de plus ici : ce
mappage fonctionne déjà pour le rootless en général (Podman l'utilise
pour tout conteneur, GPU ou non), il n'est pas spécifique à l'accès GPU.

```
$ lsmod | grep nvidia
nvidia_uvm    2650112  0
nvidia_drm     176128  3
nvidia_modeset 1925120 3 nvidia_drm
nvidia        17956864 148 nvidia_uvm,nvidia_modeset
```
`nvidia_uvm` chargé (0 utilisateur actuellement, chargé via le
`softdep nvidia post: nvidia-uvm` de
`/usr/lib/modprobe.d/nvidia-uvm.conf`, déjà documenté dans
`docs/machine-facts.md` § GPU) — nécessaire pour tout calcul CUDA
(gestion mémoire unifiée), déjà en place, aucune action requise.

## 4. Conditions pour un accès GPU rootless réellement fonctionnel

Établies par ce qui précède, à réunir **toutes** :

1. Les nœuds de périphérique nécessaires existent et sont accessibles en
   lecture/écriture par l'utilisateur non privilégié — **acquis** (§ 3,
   mode `666` sur les nœuds de calcul).
2. `subuid`/`subgid` configurés pour l'utilisateur — **acquis** (§ 3),
   déjà utilisé par tout conteneur rootless sur ce poste.
3. Les bibliothèques utilisateur NVIDIA (`libcuda.so`, `libnvidia-ml.so`,
   etc., versionnées `610.43.03` sur ce poste) doivent être visibles
   **dans** le conteneur — soit embarquées dans l'image, soit injectées
   au lancement. **C'est le point que CDI résout et qu'un montage manuel
   de périphériques ne résout pas** — voir § 6.
4. Le pilote et les modules noyau doivent rester chargés côté hôte
   pendant toute la durée de vie du conteneur — **acquis** (§ 3, modules
   déjà chargés, pas de rapport avec le conteneur lui-même).

### Test de vérification conçu, non exécuté

Un test qui se contente de démarrer un conteneur ne prouve rien — le
mode d'échec dominant est silencieux (l'application se rabat sur le CPU
sans le signaler). Le test doit **échouer bruyamment** si le GPU n'est
pas visible, et ne doit **pas** déléguer cette vérification à un
framework ML (qui bascule sur CPU silencieusement) :

```sh
# Conçu pour une image minimale contenant nvidia-smi (ou un binaire CUDA
# équivalent) — pas exécuté dans cette série, exigerait une image.
podman run --rm --device nvidia.com/gpu=all docker.io/<image-avec-nvidia-smi> \
  sh -c '
    set -e
    OUT="$(nvidia-smi -L)" || { echo "ÉCHEC : nvidia-smi absent ou en erreur (rc=$?)" >&2; exit 1; }
    echo "$OUT" | grep -q "^GPU 0:" || { echo "ÉCHEC : aucun GPU listé — $OUT" >&2; exit 1; }
    echo "OK : $OUT"
  '
```

Points de conception qui rendent ce test **actionnable**, pas
décoratif :
- `set -e` + vérification explicite du code retour de `nvidia-smi` —
  une erreur silencieuse de la commande ne doit pas être avalée.
- L'assertion porte sur le **contenu** de la sortie (`grep -q "^GPU
  0:"`), pas seulement sur le code retour : une sortie vide avec code 0
  ne doit pas passer pour un succès (cohérent avec la règle `CLAUDE.md`
  « une commande sans sortie n'établit rien »).
- Le message d'échec inclut la sortie brute — pas un simple « échec »
  muet, pour rester diagnosticable après coup.
- N'utilise à aucun moment un framework ML comme oracle : `nvidia-smi`
  interroge directement le pilote, il ne peut pas se rabattre sur le CPU
  par conception, contrairement à PyTorch/TensorFlow.

`@VERIF : ce test n'a pas été exécuté — exigerait une image contenant
nvidia-smi ou un binaire équivalent, hors périmètre lecture seule de ce
livrable. À exécuter avant toute déclaration de succès dans un livrable
qui appliquerait une option du § 6.`

## 5. Interaction avec la veille runtime D3 — ce qui est documentable sans lancer de conteneur

Établi dans `docs/dgpu-power.md` : le réveil du périphérique est
provoqué par un **appel qui interroge activement le pilote** (`nvidia-smi`
en a apporté la preuve directe), pas par la simple existence d'un
descripteur de fichier ouvert sur le périphérique. Un bind-mount de
`/dev/nvidia0` dans un espace de noms de montage de conteneur est, du
point de vue du noyau, une opération sur le même périphérique physique
que depuis l'hôte — il n'existe pas de mécanisme connu par lequel le
seul fait de le rendre visible dans un conteneur, sans qu'aucun
processus ne l'ouvre ni ne l'interroge, changerait son état
d'alimentation runtime. **Ce raisonnement s'appuie sur le mécanisme déjà
vérifié dans `docs/dgpu-power.md`, il n'est pas revérifié ici avec un
conteneur réel.**

Deux nuances établissables en lecture seule, sans lancer de conteneur :

- **Documentation NVIDIA déjà citée dans `docs/dgpu-power.md`** (« the
  NVIDIA GPU will remain in an active state ... if a CUDA application is
  running ») s'applique identiquement à un processus dans un conteneur
  qu'à un processus sur l'hôte — un serveur d'inférence conteneurisé qui
  garde un contexte CUDA ouvert bloque RTD3 **exactement comme le ferait
  le même serveur non conteneurisé**. Ce n'est pas un coût introduit par
  la conteneurisation, c'est le coût du workload, déjà connu.
- La spécification CDI générée par `nvidia-ctk` inclut des **hooks**
  (établi par recherche externe, § 6 — pas vérifié sur ce poste faute de
  spécification existante, voir § 1). Un hook qui interroge le pilote au
  **lancement** du conteneur (par exemple pour créer des liens
  symboliques de bibliothèques ou mettre à jour un cache de l'éditeur de
  liens) pourrait réveiller le périphérique **à chaque démarrage de
  conteneur**, indépendamment de l'usage réel du GPU ensuite — un coût
  potentiellement différent pour des conteneurs courts et fréquents
  (Execution Environments) que pour un serveur de longue durée.

`@VERIF : effet réel, mesuré, du lancement d'un conteneur avec
périphériques NVIDIA montés (via CDI ou --device) sur runtime_status
alors qu'aucune charge de travail active n'utilise le GPU. Exigerait de
lancer un conteneur minimal et de relire immédiatement
/sys/bus/pci/devices/0000:01:00.0/power/runtime_status et
/proc/driver/nvidia/gpus/0000:01:00.0/power, avec la méthode d'isolement
de `docs/dgpu-power.md` (relevé avant lancement, relevé immédiatement
après, relevé après un délai sans nouvelle action) — hors périmètre
lecture seule de ce livrable. Ne pas conclure, dans un sens ou l'autre,
sans cette mesure : ni « les conteneurs cassent la veille » ni « les
conteneurs n'ont aucun effet » ne sont établis par ce document.**

## 6. Options, coûts, recommandation

### Option A — CDI natif Podman, spécification générée par `nvidia-ctk` (paquet `-base` seul)

**Apporte** : consommation par Podman sans aucune configuration
supplémentaire (§ 1, déjà acquis) ; injection automatique des
bibliothèques utilisateur NVIDIA correctes dans le conteneur —
confirmé par recherche externe (`cdi-support.html` et recherche croisée,
2026-08-05) : la spécification générée contient une section
`containerEdits` avec les montages des bibliothèques du pilote
(`libnvidia-ml.so.1`, `libcuda.so.<version>`), des fichiers de firmware,
et des `deviceNodes` — **l'image n'a pas besoin d'embarquer des
bibliothèques CUDA utilisateur pré-assorties à la version du pilote
hôte (`610.43.03`)**, contrairement au montage manuel (Option C).
**Coût** : un paquet à installer (`nvidia-ctk`, via un dépôt dont
aucun n'est activé sans réserve sur ce poste — § 2) ; régénération de la
spécification requise à chaque mise à jour du pilote (`akmod-nvidia`),
sans quoi les bibliothèques montées divergeraient de la version chargée.
**Risque** : dépôt tiers à ancrer correctement (§ 2) ; hooks embarqués
dans la spécification dont l'effet sur la veille runtime n'est pas
mesuré (point non résolu, § 5).
**Comment savoir qu'elle fonctionne** : le test conçu au § 4, exécuté
après génération réelle de la spécification.

### Option B — NVIDIA Container Toolkit complet (hook d'exécution + CDI)

**Apporte** : la même chose que l'option A, plus un hook d'exécution
OCI (`nvidia-container-runtime`) redondant dès lors que Podman consomme
CDI nativement (§ 1, § 2).
**Coût** : strictement supérieur à l'option A — mêmes paquets, plus le
hook et ses dépendances transitives, jamais exercés si CDI est le seul
chemin utilisé. Violerait directement la règle « déterminer ce que
l'outil gère déjà lui-même » (`CLAUDE.md` § Avant d'agir).
**Non recommandée** : sur-couverture caractérisée par rapport à
l'option A, pour un gain nul dans ce contexte (Podman seul, pas Docker).

### Option C — Montage manuel des périphériques, sans CDI ni toolkit

**Apporte** : aucune dépendance à un dépôt tiers — `--device` seul
suffit pour les nœuds déjà accessibles (§ 3).
**Coût** : l'image doit embarquer des bibliothèques utilisateur NVIDIA
compatibles avec le pilote hôte `610.43.03` (aucune injection automatique
sans CDI) — coordination manuelle à chaque mise à jour du pilote entre
l'hôte et les images utilisées, pour les deux usages cibles (EE et
serveur d'inférence).
**Risque** : dérive de version entre pilote hôte et bibliothèques de
l'image — un mode d'échec qui peut lui aussi être silencieux (erreur
CUDA peu claire, ou un plantage au lieu d'un message net) si les versions
divergent trop.
**Comment savoir qu'elle fonctionne** : même test qu'en § 4, avec une
image ayant embarqué ses propres bibliothèques.
**Repli possible**, pas recommandée en premier choix : viable seulement
si les images utilisées intègrent déjà une pile CUDA compatible (images
officielles NVIDIA CUDA, dont la version doit alors être choisie et
maintenue en cohérence avec le pilote hôte).

## Recommandation

**Option A** — CDI natif, spécification générée par `nvidia-ctk` (paquet
`-base` uniquement, pas le toolkit complet). Ce qui la départage :

- Contre l'option B : même bénéfice, coût strictement moindre (pas de
  hook OCI redondant avec un consommateur CDI déjà natif dans Podman).
- Contre l'option C : la coordination de version pilote/bibliothèques
  est automatique (régénération de la spec après mise à jour du pilote)
  plutôt que manuelle par image — plus robuste pour un serveur
  d'inférence de longue durée dont la disponibilité importe, et pour des
  Execution Environments dont les images ne devraient pas avoir à
  embarquer une pile CUDA figée.

**Ce qui reste à trancher avant d'appliquer cette option, pas dans ce
livrable** : la source du paquet `nvidia-ctk`/`-base` — aucun dépôt
actuellement activé ne le fournit sans réserve (§ 2). Trois pistes
identifiées, aucune vérifiée jusqu'au bout : le dépôt officiel NVIDIA
(signature non fiable, documenté par la communauté), le COPR Fedora
AI/ML (page inaccessible dans cette série, contenu et clé non vérifiés),
le dépôt CUDA officiel (non creusé). **Les faits ne permettent pas de
choisir entre ces trois pistes** — mesure manquante : contenu et clé de
signature du COPR `ai-ml/nvidia-container-toolkit` (point non résolu, § 2), à
lever avant de choisir la source du paquet, indépendamment du choix déjà
tranché ci-dessus entre CDI natif et alternatives.

## Voir aussi

- [`docs/dgpu-power.md`](dgpu-power.md) — mécanisme RTD3, méthode
  d'isolement de l'effet de mesure, réutilisée en § 5 de ce document.
- [`docs/machine-facts.md`](machine-facts.md) — inventaire Conteneurs,
  attributs `asus-armoury`, point ouvert sur la clé Terra.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici,
  notamment la classe de source externe et la prudence dépôt tiers.
