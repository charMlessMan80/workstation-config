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
  être lu. **[PARTIELLEMENT RÉSOLU le 2026-08-05]** Contenu et paquets
  établis (§ 7.1-7.2) ; empreinte de clé obtenue mais sa corroboration
  indépendante reste un point non résolu — voir § 7.3 pour le marqueur
  actif, qui remplace celui-ci.
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

**[PARTIELLEMENT LEVÉ le 2026-08-05]** Image obtenue et test tenté (§
8.3, 8.5) — résultat actuel : échec attendu (`unresolvable CDI devices`,
la spécification n'existe pas encore). Reste un point non résolu :
rejouer ce test **après** l'installation réelle (§ 8.6) pour confirmer
qu'il réussit alors, avant toute déclaration de succès pour l'option
retenue au § 6.

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

**[FERMÉ le 2026-08-05, § 8.7]** Mesuré, avec la méthode d'isolement de
`docs/dgpu-power.md` (relevé avant lancement, immédiatement après,
après un délai) : `runtime_status` reste `suspended` et
`runtime_active_time` ne bouge pas d'un milliseconde
(`podman run --rm --device nvidia.com/gpu=all ... echo ...`, sans
invoquer `nvidia-smi` ni aucun outil touchant le GPU à l'intérieur).
**Un lancement de conteneur avec périphériques CDI montés, sans charge
de travail qui interroge activement le GPU, ne réveille pas le
périphérique** — les hooks de la spécification (injection de
bibliothèques, création de nœuds) ne semblent pas interroger le
matériel. Établi par une seule mesure ; le coût réel pour des
conteneurs très fréquents (EE) reste à confirmer sur volume, mais
l'hypothèse « chaque lancement réveille le GPU » est infirmée pour ce
cas simple.

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

## 7. Suite (2026-08-05) — D7/D8, Phase 1 exécutée, **arrêt avant écriture**

**Décisions de l'opérateur, consignées telles quelles** (détail complet
et motif dans `docs/machine-facts.md` § Décisions, D7 et D8) : source du
toolkit = COPR `@ai-ml/nvidia-container-toolkit`, restreint par
`includepkgs` ; SELinux reste en `Enforcing`, résolution par refus réels
observés, pas par ajustement préventif.

Ce qui suit est la Phase 1 de résolution (identification du COPR,
paquets, hypothèse base/complet, empreinte de clé) — **exécutée
entièrement en lecture seule**, avant toute écriture dans `/etc/`. Les
Phases 2 à 5 (rôle `roles/gpu_cdi/`, dépôt écrit, paquet installé,
spécification CDI générée, SELinux, tests) **ne sont pas exécutées dans
cette série** — voir § 7.5 pour le motif exact de cet arrêt, imposé par
la consigne elle-même (§ 1.3 de la demande).

### 7.1 — Identification du COPR et paquets fournis pour `fedora-44-x86_64`

Groupe `@ai-ml`, projet `nvidia-container-toolkit`. Fichier de dépôt tel
que `dnf copr enable @ai-ml/nvidia-container-toolkit` l'installerait —
**lu à sa source publique, jamais activé** :

```
$ curl -s https://copr.fedorainfracloud.org/coprs/g/ai-ml/nvidia-container-toolkit/repo/fedora-44/ai-ml-nvidia-container-toolkit-fedora-44.repo
[copr:copr.fedorainfracloud.org:group_ai-ml:nvidia-container-toolkit]
name=Copr repo for nvidia-container-toolkit owned by @ai-ml
baseurl=https://download.copr.fedorainfracloud.org/results/@ai-ml/nvidia-container-toolkit/fedora-$releasever-$basearch/
type=rpm-md
skip_if_unavailable=True
gpgcheck=1
gpgkey=https://download.copr.fedorainfracloud.org/results/@ai-ml/nvidia-container-toolkit/pubkey.gpg
repo_gpgcheck=0
enabled=1
```
(Récupéré par requête HTTP externe, pas par `dnf copr enable` — aucun
fichier n'a été écrit dans `/etc/yum.repos.d/` pour cette lecture.)
`repo_gpgcheck=0` : seules les signatures des **paquets** sont vérifiées
(`gpgcheck=1`), pas celle du `repomd.xml` lui-même — propriété du modèle
de dépôt COPR par défaut, pas une option choisie par ce projet en
particulier.

Contenu réel pour `fedora-44-x86_64`, build `10501685`
(`nvidia-container-toolkit-1.19.1-1.fc44`, soumis 2026-05-22, état
`succeeded`), lu par accès direct au répertoire de résultats publié
(`download.copr.fedorainfracloud.org/results/@ai-ml/nvidia-container-toolkit/fedora-44-x86_64/10501685-nvidia-container-toolkit/`) :

```
nvidia-container-toolkit-1.19.1-1.fc44.x86_64.rpm
nvidia-container-toolkit-1.19.1-1.fc44.src.rpm
nvidia-container-toolkit-selinux-1.19.1-1.fc44.noarch.rpm
nvidia-container-toolkit-debuginfo-1.19.1-1.fc44.x86_64.rpm
nvidia-container-toolkit-debugsource-1.19.1-1.fc44.x86_64.rpm
nvidia-container-toolkit-operator-extensions-1.19.1-1.fc44.x86_64.rpm
nvidia-container-toolkit-operator-extensions-debuginfo-1.19.1-1.fc44.x86_64.rpm
```
`operator-extensions` = intégration Kubernetes (device plugin), sans
objet pour Podman seul. `debuginfo`/`debugsource` = symboles de
débogage, sans objet. `.src.rpm` = source, non installable tel quel.

### 7.2 — Hypothèse « `-base` suffit » : **vérifiée, et infirmée pour cette source précise**

Contenu réel du paquet binaire `nvidia-container-toolkit` (fichiers,
pas suppositions), obtenu via un dépôt éphémère
(`--repofrompath`, jamais persisté dans `/etc/yum.repos.d/`) :

```
$ dnf repoquery --repofrompath=coprtmp,https://download.copr.fedorainfracloud.org/results/@ai-ml/nvidia-container-toolkit/fedora-44-x86_64/ \
    --repo=coprtmp --assumeno -l nvidia-container-toolkit
/usr/bin/nvidia-cdi-hook
/usr/bin/nvidia-container-runtime
/usr/bin/nvidia-container-runtime-hook
/usr/bin/nvidia-ctk
/usr/bin/nvidia-ctk-installer
/etc/cdi
[...documentation, licences...]
```

**Ce COPR ne construit pas de variante `-base`** — contrairement au
dépôt officiel NVIDIA (§ 2, qui distingue bien les deux). Le seul paquet
binaire principal (`nvidia-container-toolkit`) contient `nvidia-ctk`
**et** les deux hooks d'exécution (`nvidia-container-runtime`,
`nvidia-container-runtime-hook`), inutilisés avec Podman/CDI natif mais
**indissociables au niveau du paquet** — `dnf` ne permet pas d'installer
un sous-ensemble de fichiers d'un paquet. **L'hypothèse de la
résolution précédente (choisir un paquet `-base` pour éviter le hook) ne
s'applique pas à cette source** : le hook sera installé, inerte, quel
que soit le choix. Ce n'est pas une sur-couverture qu'on peut éviter ici
— seulement une sur-couverture qu'on choisit de ne jamais invoquer.

`nvidia-container-toolkit-selinux` (`noarch`) contient un module SELinux
compilé : `/usr/share/selinux/packages/targeted/nvidia-container.pp`.

**[CORRIGÉ le 2026-08-05, § 8.7] ÉTAIT** : « installer ce paquet
n'active pas la politique (un `.pp` sur disque n'est chargé qu'après
`semodule -i` explicite), donc son inclusion dans `includepkgs` ne
préempte pas la Phase 3 » — **faux pour ce paquet précis**, infirmé par
l'exécution réelle : `semodule -l` après installation montre le module
`nvidia-container` chargé et actif. Le script `%post` du paquet
l'enregistre automatiquement via `semodule -i` — comportement vendeur
standard pour un sous-paquet `-selinux`, pas une action que ce rôle ou
cette session a prise. Detail complet, y compris pourquoi ceci reste
conforme à D8, en § 8.7.

### 7.3 — Empreinte de la clé GPG : obtenue, ancrage de confiance corrigé et accepté [RÉSOLU le 2026-08-05]

Empreinte complète, calculée localement à partir du fichier de clé
publié à l'URL déclarée par le dépôt (`.../pubkey.gpg` ci-dessus),
importée dans un **trousseau temporaire isolé** — jamais le trousseau
système, aucune confiance accordée :

```
$ gpg --homedir <tmp isolé> --no-default-keyring --import pubkey.gpg
gpg: key 1C34CABF2CC19B05: public key "@ai-ml_nvidia-container-toolkit
     (None) <@ai-ml#nvidia-container-toolkit@copr.fedorahosted.org>" imported
$ gpg --homedir <tmp isolé> --no-default-keyring --list-keys --with-fingerprint --with-colons
fpr:::::::::0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05:
```

**Empreinte complète : `0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05`.**

**Tentatives de corroboration indépendante, toutes infructueuses** —
consignées pour que la prochaine série n'ait pas à les rejouer à
l'identique sans le savoir :
1. Page du projet COPR (`copr.fedorainfracloud.org/coprs/g/ai-ml/...`,
   deux tentatives, deux onglets) : bloquée par le pare-robot Anubis du
   frontend Fedora, aucun contenu récupéré.
2. Recherche web de l'empreinte complète elle-même
   (`0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05`) : aucune occurrence
   trouvée, sur aucun site.
3. API COPR (`api_3/project`, `api_3/package/list`, `api_3/build/list`) :
   fournit les métadonnées de build, jamais l'empreinte de clé en clair.
4. Aucune seconde publication indépendante identifiée (pas de page wiki
   Fedora dédiée à la SIG `ai-ml` listant cette empreinte, pas de
   mention dans les discussions Fedora trouvées par recherche).

**[MOTIF CORRIGÉ le 2026-08-05] ÉTAIT** : la conclusion de la série
précédente demandait une « corroboration indépendante » de cette
empreinte, faute de quoi elle restait un point non résolu, en s'appuyant
sur le parallèle avec la clé Terra. **Ce parallèle était mal posé** : Terra a
une clé publiée par un tiers identifiable (Fyra Labs) dont on peut en
principe vérifier l'empreinte par un canal distinct de la machine qui
sert le dépôt — l'obstacle y est une asymétrie de visibilité locale, pas
une absence structurelle de second canal. **Une clé de projet COPR n'a
jamais de second canal** : elle est générée et détenue par
l'infrastructure de build Fedora elle-même, pour ce projet précis, et
republiée par cette même infrastructure au même endroit qu'elle sert le
dépôt. Comparer l'empreinte « à la page du projet » est circulaire —
même origine, même ancrage TLS — pour **tout** projet COPR, pas
seulement celui-ci. Demander cette corroboration revenait à demander une
preuve qui n'existe structurellement pour aucun COPR.

**CORRIGÉ — ancrage de confiance, formulé explicitement plutôt que
supposé** : l'ancrage réel est **TLS vers `copr.fedorainfracloud.org`**
(et `download.copr.fedorainfracloud.org`), plus la confiance accordée à
l'infrastructure Fedora et au groupe `@ai-ml` en tant que tel. La
signature GPG garantit l'intégrité entre le build et cette machine ;
elle **n'établit rien de plus sur l'identité du signataire** que ce que
TLS a déjà fourni au moment où le fichier `.repo` et la clé ont été
récupérés. Empreinte relevée, faisant foi pour ce dépôt : **
`0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05`** (méthode § ci-dessus,
trousseau temporaire isolé). Aucune corroboration indépendante n'existe
ni ne peut exister pour une clé de projet COPR — ce n'est pas une lacune
de ce projet précis, c'est une propriété du modèle COPR lui-même.
**Positionnement retenu** : un cran en dessous d'un paquet Fedora
officiel (clé Fedora primaire, publiée et renouvelée par un processus
distinct de tout dépôt particulier), un cran au-dessus du dépôt NVIDIA
officiel (vérifications OpenPGP documentées comme défaillantes par la
communauté, § 2).

**Décision de l'opérateur, consignée** : ce modèle de confiance est
accepté explicitement pour ce projet (voir `docs/machine-facts.md` §
Décisions, D7, motif corrigé). Le marqueur précédent est fermé — il
n'était pas actionnable pour la raison qui vient d'être établie, pas
parce que la question a été éludée.

### 7.4 — Liste `includepkgs` — établie, **non appliquée**

Si le point 7.3 est levé, la liste minimale justifiée serait :

```
includepkgs=nvidia-container-toolkit,nvidia-container-toolkit-selinux
```
- `nvidia-container-toolkit` : seul paquet fournissant `nvidia-ctk`
  (§ 7.2) — nécessaire, avec le hook inerte comme sur-couverture
  incompressible depuis cette source.
- `nvidia-container-toolkit-selinux` : module SELinux vendeur,
  disponible pour la Phase 3 sans être chargé par sa seule installation
  (§ 7.2) — inclus par prudence, pas par anticipation d'un refus non
  encore observé.
- Exclus délibérément : `-debuginfo`, `-debugsource`,
  `-operator-extensions` (Kubernetes, sans objet), `.src` (non
  installable par ce mécanisme).

### 7.5 — Arrêt signalé, conformément à la consigne [levé le 2026-08-05, série suivante]

**Les Phases 2 à 5 n'étaient pas exécutées à la clôture de cette série.** Aucun fichier n'a été écrit
dans `/etc/yum.repos.d/` ni `/etc/cdi/`, aucun paquet installé, aucune
politique SELinux touchée, aucun conteneur lancé, `roles/gpu_cdi/` n'a
pas été créé. Motif : la demande elle-même prescrit cet arrêt précis
(« Si tu ne peux pas établir la correspondance, arrête-toi et
signale-le ») pour le cas exactement rencontré en § 7.3.

**Incident de méthode à consigner** : la reproduction de la commande
citée par l'opérateur (`dnf repoquery --file '*/nvidia-ctk'`, sans
`--disablerepo=terra` ni `--assumeno`, pour corroborer fidèlement —
CLAUDE.md § Sourcing) a déclenché la même invite d'import de clé Terra
que l'incident déjà documenté (`dnf repoinfo terra`, 2026-08-04) :
`Is this ok [y/N]:`. La session n'étant pas interactive (pas d'entrée
standard), l'invite a été résolue par une fin de flux, pas par une
réponse — comportement de repli sûr de `dnf5`, **vérifié après coup, pas
supposé** : `rpm -q gpg-pubkey` (aucune entrée datée du 2026-08-05,
liste identique à avant), et recherche de tout fichier de moins de 15
minutes sous `~/.cache` et `/etc/pki/rpm-gpg` (rien qui ressemble à un
trousseau importé). **Aucune clé n'a été importée**, mais l'invite
elle-même n'aurait pas dû pouvoir apparaître sans un filet de sécurité
explicite — reproduire une commande exacte demandée par l'opérateur ne
dispense pas d'ajouter `--assumeno` quand la commande touche un dépôt à
`gpgcheck` incertain, ce que cette série aurait dû faire d'emblée par
cohérence avec elle-même.

**Chemin retenu, décidé par l'opérateur dans la série suivante (voir §
7.3, motif corrigé)** : le modèle de confiance TLS-seul de COPR est
accepté explicitement pour ce projet — la demande de corroboration
indépendante était elle-même mal posée (aucun projet COPR n'a de second
canal). Phase 1 est close depuis cette décision ; le rôle `roles/gpu_cdi/`
est préparé (§ 8) mais son exécution réelle est **suspendue** — pas
bloquée par l'absence de règle `NOPASSWD` comme prévu, mais par une
découverte inattendue qui change la donne : voir § 8.6.

## 8. Rôle `roles/gpu_cdi/` préparé et validé — exécution réelle suspendue

### 8.1 — Corrections préalables 0.2 et 0.3

**0.2, arbitrage consigné dans `CLAUDE.md` § Sourcing** : la
reproduction fidèle d'une commande dont l'effet de bord est déjà
documenté se fait désormais avec sa garde (`--assumeno`), en signalant
l'écart. Aucune commande touchant Terra n'a été rejouée sans garde dans
cette série.

**0.3, hypothèse vérifiée, pas supposée** : l'état de `containers.conf`
et du runtime OCI actif a été relevé **avant** toute installation, pour
comparaison après coup (§ 8.4 ci-dessous, une fois l'installation
réelle effectuée) :

```
$ sha256sum /usr/share/containers/containers.conf
6b1069352180459572e458cd778961cc22ab144dd2f63605d3bbdd16099f6c85  /usr/share/containers/containers.conf
$ grep -A3 '^\[engine.runtimes\]' /usr/share/containers/containers.conf
[engine.runtimes]
#crun = [
#  "/usr/bin/crun",
#  "/usr/sbin/crun",
$ podman info --format '{{.Host.OCIRuntime.Name}} {{.Host.OCIRuntime.Path}}'
crun /usr/bin/crun
$ ls ~/.config/containers/containers.conf
No such file or directory
```
`[engine.runtimes]` entièrement commenté (aucune surcharge), seul
`crun` actif, aucun `containers.conf` utilisateur. Le rôle
`roles/gpu_cdi/` intègre cette même comparaison comme tâche assertée
(pas seulement consignée en prose) — voir `tasks/main.yml`.

### 8.2 — Rôle construit et validé

`roles/gpu_cdi/` créé : `defaults/main.yml`, `tasks/main.yml`,
`templates/copr-nvidia-container-toolkit.repo.j2`, `meta/main.yml`,
`README.md`, `gpu_cdi.yml`. Huit groupes de tâches dans l'ordre demandé
(gardes, capture avant, épinglage de clé, dépôt, installation, capture
après + assertion, génération CDI, vérification de complétude).

```
$ ansible-playbook --syntax-check roles/gpu_cdi/gpu_cdi.yml
playbook: roles/gpu_cdi/gpu_cdi.yml
$ ansible-playbook --check roles/gpu_cdi/gpu_cdi.yml
[...]
localhost : ok=14  changed=0  unreachable=0  failed=0  skipped=20  rescued=0  ignored=0
```
Toutes les gardes passent réellement en `--check` (`nvidia_uvm` chargé,
nœuds UVM présents, `nvidia-smi -L` liste la RTX 4090, Podman rootless
confirmé) ; toutes les tâches privilégiées ou réseau sont proprement
ignorées (`skipping`), aucune n'a été tentée.

**Épinglage de clé (§ 7.3) opérationnalisé, pas seulement documenté** :
le rôle télécharge la clé dans un répertoire temporaire, l'importe dans
un trousseau isolé (`--no-default-keyring`), et casse si l'empreinte
diffère de `0E6C304C16654ADA1AF6CB8F1C34CABF2CC19B05` — garde contre un
changement de clé futur, pas contre l'absence structurelle d'une
seconde source déjà actée en § 7.3.

### 8.3 — Les trois démonstrations d'échec

**Second échec forcé demandé (garde de vide) — exécuté** :
```
$ ansible-playbook --check -e gpu_cdi_spec_path="" roles/gpu_cdi/gpu_cdi.yml
[...]
TASK [Garde 0 : gpu_cdi_spec_path ne doit jamais être vide...]
fatal: [localhost]: FAILED! => {"assertion": "gpu_cdi_spec_path | length > 0", ...}
localhost : ok=1  changed=0  unreachable=0  failed=1 ...
```
Casse à la toute première tâche du rôle (`ok=1` = la seule collecte de
faits), avant toute lecture, toute garde matérielle, toute écriture.

**Échec forcé (sans `--device`), conçu pour séparer les deux causes** —
exécuté, avec l'image de test (§ 8.5) :
```
$ podman run --rm docker.io/nvidia/cuda:12.6.2-base-ubi9 sh -c '
    if command -v nvidia-smi >/dev/null 2>&1; then
      nvidia-smi -L && exit 1 || exit 0   # présent mais échoue = device en cause
    else
      ls /dev/nvidia* >/dev/null 2>&1 && exit 1 || exit 0   # absent : vérifier la vraie cause
    fi'
OK : nvidia-smi absent (attendu, non injecté sans --device) ET aucun
nœud /dev/nvidia* visible -- échec dû à l'absence de périphérique, pas
à l'absence d'outil
```
**Comment ce test sépare les deux causes** : établi au préalable, sur
*cette même image*, que `nvidia-smi` n'y est **pas** embarqué
(`which nvidia-smi` → absent, `nvidia-smi --version` → « executable
file not found », rc=127) — c'est le pilote/CDI qui l'injecte au
lancement, pas l'image. Sans `--device`, son absence est donc **attendue
et non significative** ; le test ne s'arrête pas là et vérifie la cause
réelle en listant `/dev/nvidia*` directement (présent dans toute image
via `ls`, indépendant de CDI) — c'est cette absence-là, pas celle de
l'outil, qui établit l'échec d'accès GPU. Un test qui se serait arrêté
sur « `nvidia-smi` absent → échec » aurait été exactement le piège
signalé : indiscernable d'un échec d'accès GPU réel.

**Test nominal — tenté dès maintenant pour établir l'état de référence,
échec attendu et confirmé (Phase 1 pas encore appliquée réellement)** :
```
$ podman run --rm --device nvidia.com/gpu=all docker.io/nvidia/cuda:12.6.2-base-ubi9 nvidia-smi -L
Error: setting up CDI devices: unresolvable CDI devices nvidia.com/gpu=all
```
Attendu : `/etc/cdi/nvidia.yaml` n'existe pas encore. Ce test sera rejoué
tel quel une fois l'installation réelle effectuée (§ 8.6) — c'est la
preuve « avant » de la comparaison, pas un échec de ce livrable.

### 8.4 — Terra : paquets installés en provenance de ce dépôt

```
$ dnf repoquery --installed --qf '%{name} from_repo=%{from_repo}\n' | grep -i 'from_repo=terra'
asusctl from_repo=terra
asusctl-rog-gui from_repo=terra
cardwire from_repo=terra
supergfxctl from_repo=terra
terra-gpg-keys from_repo=terra
terra-release from_repo=terra
```
**Six paquets réels en dépendent**, dont `asusctl` — central à toute la
série GPU MUX de ce dépôt (attributs `asus-armoury`). **Terra reste
nécessaire ; sa désactivation n'est pas proposée** — conforme à la
consigne (« si aucun paquet n'en provient, la désactivation sera
proposée comme décision à part » : ce n'est pas le cas ici).

### 8.5 — Image de test téléchargée (exception signalée)

`docker.io/nvidia/cuda:12.6.2-base-ubi9`, seule exception à
l'interdiction de téléchargement, pour les tests § 8.3. Conservée en
cache local Podman pour rejouer le test nominal après l'installation
réelle — pas versionnée, pas un composant du dépôt.

### 8.6 — SELinux, état de référence — et une découverte qui a suspendu l'exécution réelle [levé le 2026-08-05, D9]

**Avant toute installation** :
```
$ getenforce
Enforcing
$ sudo -n ausearch -m avc -ts recent
<no matches>
$ journalctl -t setroubleshoot --no-pager -n 20
[...] SELinux is preventing power-profiles- from {read,open,getattr} access on the file /etc/passwd [...]
```
Aucun refus AVC récent ; les seuls refus journalisés (`setroubleshoot`)
concernent `power-profiles-`, sans rapport avec ce travail — établi
comme référence, pas comme une anomalie à traiter ici.

**Découverte non anticipée, à traiter avant de poursuivre** : la
commande `sudo -n ausearch` ci-dessus **a réussi sans mot de passe**.
Vérification immédiate, par prudence :
```
$ sudo -n true
$ echo $?
0
$ sudo -n -l
[...]
User mahieumi may run the following commands on Zephyrus-MM:
    (ALL) NOPASSWD: ALL
```
**Ceci contredit directement `docs/gpu-mux-recovery.md`**, qui
documente, plus tôt le même jour, un échec exact et sourcé de la même
vérification (« sudo: a password is required », `groups=mahieumi,wheel`
— membre de `wheel`, mais pas de règle `NOPASSWD`). Cause de l'écart
**non établie dans cette session** : changement délibéré des `sudoers`
par l'opérateur entre les deux séries, ou différence de contexte
d'exécution non identifiée — les deux restent possibles, aucune n'est
tranchée ici.

**Ce livrable n'a pas exploité cette capacité.** La consigne de la
demande était explicite (« ne tente aucun contournement ») et présumait
cette règle absente ; la trouver présente ne change pas ce que cette
consigne demande — elle appelle à signaler avant d'agir, pas à profiter
du fait que l'obstacle prévu s'est révélé ne pas en être un (voir la
règle ajoutée à ce sujet dans `CLAUDE.md` § Avant d'agir). L'exécution
réelle du rôle est restée suspendue jusqu'à confirmation explicite de
l'opérateur — obtenue, consignée comme **D9**
(`docs/machine-facts.md` § Décisions). Suite de la résolution, avec
l'exécution réelle : § 8.7.

### 8.7 — D9 confirmée, exécution réelle, toutes les validations restantes

**D9 vérifiée indépendamment, pas seulement reçue** — voir
`docs/machine-facts.md` § Décisions pour le détail (`visudo -c`,
contenu de `/etc/sudoers.d/`, ligne 110 relue directement). Toutes les
lectures de `/etc/sudoers` ci-dessous sont des **actions
privilégiées** (le fichier est `0440 root:root`), énumérées ici
conformément à la règle ajoutée dans `CLAUDE.md` :

| # | Commande | Chemin/cible | Motif |
|---|---|---|---|
| 1 | `sudo -n sed -n '108,112p' /etc/sudoers` | lecture `/etc/sudoers` | corroborer D9 indépendamment du rapport de l'opérateur |
| 2 | `sudo -n visudo -c` | lecture/validation `/etc/sudoers` | corroborer « parsed OK » |
| 3 | `sudo -n ls -la /etc/sudoers.d/` | lecture répertoire | corroborer « vide » |
| 4 | `sudo -n ls -la /etc/sudoers` | lecture métadonnées | dater la dernière modification |
| 5 | rôle, tâche « Écrire le fichier de dépôt COPR » | écriture `/etc/yum.repos.d/_copr:copr.fedorainfracloud.org:group_ai-ml:nvidia-container-toolkit.repo` | dépôt retenu par D7 |
| 6 | rôle, tâche « Installer les paquets retenus » | `dnf install nvidia-container-toolkit nvidia-container-toolkit-selinux` | paquets retenus, § 7.4 — inclut l'exécution, **en tant que root**, du script `%post` du paquet `-selinux` (voir plus bas, correction de la § 7.2) |
| 7 | rôle, tâche « S'assurer que /etc/cdi existe » | création `/etc/cdi` | répertoire cible de la spécification CDI |
| 8 | rôle, tâche « Générer la spécification CDI » | `nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml` | Phase 1 |
| 9 | rôle, tâche « S'assurer que la spécification est lisible » | `chmod 0644 /etc/cdi/nvidia.yaml` | lisibilité rootless |
| 10 | `sudo -n ausearch -m avc -ts recent` (×2, avant et après) | lecture journal d'audit | recherche de refus AVC |
| 11 | `sudo -n semodule -l` / `--list-modules=full` | lecture état politique SELinux | vérifier si le module vendeur s'est chargé |

Aucune autre élévation. Le téléchargement/import/purge de la clé GPG
(§ 7.3, § 8.2) tourne **sans** `become` — répertoire temporaire possédé
par l'utilisateur courant, jamais le trousseau système.

**Défaut réel trouvé par l'exécution réelle, corrigé, pas refactorisé**
(permis explicitement par le périmètre de cette série) : la première
exécution a cassé sur l'assertion `gpgcheck` — `'gpgcheck=0' in contenu`
matchait aussi `repo_gpgcheck=0` (légitime, sous-chaîne). Corrigé par un
ancrage de début de ligne (`regex_search` multiligne) au lieu d'une
recherche de sous-chaîne. Un seul fichier modifié
(`roles/gpu_cdi/tasks/main.yml`), une seule assertion réécrite.

**Exécution réelle, après correction** :
```
$ ansible-playbook roles/gpu_cdi/gpu_cdi.yml
[...]
localhost : ok=33  changed=5  unreachable=0  failed=0  skipped=1 ...
```
Toutes les tâches passent, y compris l'assertion `gpgcheck=1` (ligne
exacte confirmée) et l'assertion `containers.conf`/runtime OCI inchangé
(§ 0.3 — les hooks installés restent inertes, confirmé, pas supposé).

**Seconde exécution — idempotence partielle, expliquée plutôt que
maquillée** :
```
$ ansible-playbook roles/gpu_cdi/gpu_cdi.yml
[...]
localhost : ok=33  changed=3  unreachable=0  failed=0  skipped=1 ...
```
`changed=3`, pas `0`. Les trois tâches concernées sont, dans les deux
exécutions, exactement les mêmes : création du répertoire temporaire de
vérification de clé, téléchargement de la clé, purge du répertoire —
**non idempotentes par conception**, puisque cette vérification est
délibérément éphémère et rejouée intégralement à chaque exécution
plutôt que mise en cache (§ 7.3 : l'épinglage vérifie l'empreinte à
chaque fois, il ne fait pas confiance à un résultat précédent). **Sur
l'état persistant réel** (fichier de dépôt, paquets installés,
spécification CDI, permissions), la seconde exécution ne montre
**aucun** changement (`ok` partout ailleurs) — c'est cette
idempotence-là qui compte, et elle est vérifiée. Présenter un
`changed=0` littéral aurait exigé de cesser de revérifier la clé à
chaque exécution, ce qui aurait dégradé une propriété de sécurité pour
satisfaire un chiffre — non fait.

**`/etc/cdi/` — contenu et preuve des nœuds UVM** :
```
$ ls -la /etc/cdi/
-rw-r--r--. 1 root root 19972 nvidia.yaml
$ grep -A2 'nvidia-uvm' /etc/cdi/nvidia.yaml
- path: /dev/nvidia-uvm
  major: 507
  fileMode: 438
  permissions: rwm
- path: /dev/nvidia-uvm-tools
  major: 507
  minor: 1
```
Fichier lisible par tous (`0644`), les deux nœuds UVM référencés avec
leurs numéros de périphérique réels (majeur 507, cohérent avec
`docs/machine-facts.md` § GPU).

**`containers.conf` après installation, comparé à l'avant (§ 8.1)** :
```
$ sha256sum /usr/share/containers/containers.conf
6b1069352180459572e458cd778961cc22ab144dd2f63605d3bbdd16099f6c85  (identique à l'avant)
$ podman info --format '{{.Host.OCIRuntime.Name}} {{.Host.OCIRuntime.Path}}'
crun /usr/bin/crun  (identique à l'avant)
$ grep -A5 '^\[engine.runtimes\]' /usr/share/containers/containers.conf
[engine.runtimes]
#crun = [ ...   (toujours entièrement commenté)
$ ls ~/.config/containers/containers.conf
No such file or directory  (toujours absent)
```
**Hypothèse 0.3 confirmée par comparaison réelle, pas supposée** : les
hooks installés (`nvidia-container-runtime`,
`nvidia-container-runtime-hook`) n'ont rien modifié dans la
configuration de Podman.

**`dnf repo list --enabled`** :
```
copr:copr.fedorainfracloud.org:group_ai-ml:nvidia-container-toolkit  Copr repo for nvidia-container-toolkit owned by @ai-ml (...)
```
Présent, aux côtés des dix dépôts déjà activés — aucun autre dépôt
ajouté. `includepkgs=nvidia-container-toolkit,nvidia-container-toolkit-selinux`
confirmé dans le contenu réel du fichier (`cat`), pas seulement dans le
gabarit source.

**Test nominal rejoué — succès, et ce que ce succès prouve
précisément** :
```
$ podman run --rm --device nvidia.com/gpu=all docker.io/nvidia/cuda:12.6.2-base-ubi9 nvidia-smi
[...]
|   0  NVIDIA GeForce RTX 4090 ...    Off |   00000000:01:00.0 Off |    N/A |
| N/A   50C    P0             33W /  155W |      47MiB /  16376MiB |  10% |
exit=0
```
Établi en § 7.1/8.3 : cette image **ne contient pas** `nvidia-smi` (
`which nvidia-smi` → absent). Son exécution réussie ici, avec un
résultat cohérent (même GPU, même bus PCI que partout ailleurs dans ce
dépôt), **prouve donc l'injection CDI depuis l'hôte** — le binaire et
l'accès au pilote sont tous deux apportés par la spécification générée,
pas par l'image. Ce n'est pas seulement « le GPU est visible », c'est
« le mécanisme CDI fonctionne de bout en bout ».

**Échec forcé rejoué (sans `--device`) — même séparation des causes,
même résultat** :
```
OK : nvidia-smi absent (attendu, non injecté sans --device) ET aucun
nœud /dev/nvidia* visible -- échec dû à l'absence de périphérique, pas
à l'absence d'outil
exit=0
```

**Refus AVC après le test réel** :
```
$ getenforce
Enforcing
$ sudo -n ausearch -m avc -ts recent
<no matches>
$ journalctl -t setroubleshoot --no-pager --since "16:00"
-- No entries --
```
**Aucun refus AVC, déclaré explicitement plutôt que passé sous
silence.** `getenforce` vaut `Enforcing` avant et après l'ensemble de
cette série. Conformément à D8 : **aucune modification SELinux
appliquée** — ni booléen, ni type, ni ajustement préventif, puisqu'il
n'y avait rien à résoudre.

**Correction d'une hypothèse de la § 7.2, infirmée par l'exécution
réelle** : § 7.2 affirmait « l'installation de
`nvidia-container-toolkit-selinux` n'active pas la politique... un
`.pp` sur disque n'est chargé qu'après `semodule -i` explicite ».
**Faux pour ce paquet précis** : `semodule -l` après installation montre
le module `nvidia-container` chargé et actif — le script `%post` du
paquet l'a enregistré automatiquement via `semodule -i`, comportement
vendeur standard pour un sous-paquet `-selinux`, pas une action prise
par ce rôle ou par cette session. Ceci reste conforme à D8 : aucun
ajustement SELinux n'a été **décidé ou appliqué par cette série** — le
module fait partie du paquet retenu en Phase 1, et son auto-chargement
est le comportement de son propre éditeur, pas une réponse à un refus
observé. Établissement des types/booléens exacts du module : **non
approfondi** — `seinfo` (paquet `setools-console`) est absent et son
installation aurait ajouté un paquet hors du périmètre retenu ; sans
objet ici puisqu'aucun refus AVC n'a été observé à résoudre.

**Confirmations finales de cette série** : `getenforce=Enforcing`
inchangé ; aucun `setenforce` exécuté ; aucun booléen SELinux modifié ;
aucun dépôt autre que le COPR retenu ; `terra` non désactivé ; aucune
modification de `/etc/sudoers` ; aucun redémarrage.

## Voir aussi

- [`docs/dgpu-power.md`](dgpu-power.md) — mécanisme RTD3, méthode
  d'isolement de l'effet de mesure, réutilisée en § 5 de ce document.
- [`docs/machine-facts.md`](machine-facts.md) — inventaire Conteneurs,
  attributs `asus-armoury`, point ouvert sur la clé Terra, décisions D7
  et D8.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici,
  notamment la classe de source externe et la prudence dépôt tiers.
