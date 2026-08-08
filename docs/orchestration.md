# Orchestration — reconstruction complète d'un poste neuf

`site.yml`, à la racine du dépôt, rejoue l'ensemble des rôles de ce
dépôt dans l'ordre de leurs dépendances réelles, reconstitué par la
revue globale ([`docs/review-2026-08.md`](review-2026-08.md) § 7.2) et
vérifié ici par une exécution `--check` complète (§ Ce qui a été
vérifié, plus bas). Ce document décrit la séquence **entière**, y
compris ce qui précède `site.yml` — Ansible ne peut rien faire sur une
machine qui n'a ni Fedora, ni `git`, ni `ansible-core`.

## 0. Ce qui doit être vrai avant la première commande Ansible

Rien de ce qui suit n'est scripté par ce dépôt — ce sont des actions
manuelles, ou de futurs candidats à un rôle qui n'existe pas encore
(voir § Ce qui reste hors d'Ansible, plus bas).

1. **Fedora 44 installé**, KDE Plasma (Wayland), sur le matériel cible
   (ASUS ROG Zephyrus Duo 16 GX650PY ou équivalent — les chemins
   `/sys/class/firmware-attributes/asus-armoury/` supposés par
   `roles/gpu_mux/` sont spécifiques à ce matériel, `CLAUDE.md` §
   Matériel spécifique).
2. **Un compte utilisateur membre du groupe `wheel`**, avec un mot de
   passe fonctionnel — c'est ce que l'installateur Fedora fait déjà
   pour le premier compte créé (fait externe à ce dépôt, jamais
   vérifié ici sur une machine autre que celle-ci). Sans ce compte,
   même `--ask-become-pass` (§ 2) n'a rien à authentifier.
3. **RPM Fusion (`free` et `nonfree`) activés** — aucun rôle de ce
   dépôt ne le fait ; `roles/gpu_cdi/` installe le toolkit CDI mais
   suppose le pilote NVIDIA (`akmod-nvidia`, `xorg-x11-drv-nvidia*`,
   fournis par RPM Fusion nonfree) déjà présent. Procédure Fedora
   standard, externe à ce dépôt :
   ```
   sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
     https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
   ```
4. **Pilote NVIDIA propriétaire installé et chargé** (`akmod-nvidia`,
   `xorg-x11-drv-nvidia-cuda`, `nvidia-modprobe`, `nvidia-persistenced`
   au minimum — inventaire complet des paquets réellement présents sur
   ce poste, `rpm -qa | grep -i nvidia`, `docs/machine-facts.md` §
   GPU) :
   ```
   sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
   ```
   Suivi d'un redémarrage (externe à ce dépôt — celui-ci est **avant**
   toute exécution Ansible, distinct du point d'arrêt de `site.yml`
   décrit plus bas) pour que le module `nvidia.ko` charge réellement.
   Sans cette étape, `roles/gpu_cdi/` échouerait dès sa première garde
   (`nvidia-smi` absent).
5. **`git` et `ansible-core` installés** (`sudo dnf install git
   ansible-core`), puis ce dépôt cloné :
   ```
   git clone <url-de-ce-dépôt> workstation-config
   cd workstation-config
   ```

## 1. Amorçage — `become` avant que D9 n'existe

Sur une machine neuve, aucune règle `NOPASSWD` n'existe encore — la
toute première invocation de `site.yml` doit fournir le mot de passe
explicitement :

```
ansible-playbook --ask-become-pass site.yml
```

`roles/bootstrap/` (premier rôle de la séquence) écrit la règle
`NOPASSWD` dans `/etc/sudoers.d/` (D9) — **mais seulement sous le tag
`bootstrap-sudoers`**, jamais par défaut à l'intérieur d'une exécution
normale sur *cette* machine précise (voir
[`roles/bootstrap/README.md`](../roles/bootstrap/README.md) : la
prudence retenue en `COR-2` était spécifique à une machine qui a déjà
une règle fonctionnelle à préserver). **Sur une machine réellement
neuve**, il n'y a rien à préserver — invoquer `site.yml` avec le tag
explicite dès le premier passage est la voie normale :

```
ansible-playbook --ask-become-pass site.yml --tags bootstrap-sudoers,recovery,gpu_cdi,gpu_mux,local_ai,editor,completion
```

(équivalent à jouer tous les tags sauf aucun — explicite ici pour ne
pas dépendre d'un défaut qui pourrait changer ; sur une machine neuve,
lister tous les tags disponibles, `bootstrap-sudoers` inclus). Une fois
D9 en place, les invocations suivantes n'ont plus besoin de
`--ask-become-pass`.

## 2. Séquence — un seul passage, un seul point d'arrêt

`site.yml` exécute, dans l'ordre, sans redémarrer lui-même :

1. **`bootstrap`** — D6, D9, D10 (§ 1 ci-dessus pour le tag
   `bootstrap-sudoers`). Aucune dépendance amont au-delà de `become`.
2. **`recovery`** — chemin de retour SSH. **Doit précéder `gpu_mux`** :
   sa garde 0a/0b/0c (`sshd` actif+activé, `authorized_keys` non vide,
   `firewalld` autorise `ssh`) vérifie l'état réel au moment de
   l'exécution, pas le souvenir que `recovery` ait tourné un jour
   (`docs/review-2026-08.md` § 4.1, COR-1) — l'ordre est ce qui rend
   cette garde utile plutôt que vide.
3. **`gpu_cdi`** — dépôt COPR, toolkit, spécification CDI. Dépend du
   pilote NVIDIA (§ 0.4), pas d'un autre rôle de ce dépôt.
4. **`gpu_mux`** — écrit `gpu_mux_mode` (D2bis/D2ter). Dépend de
   `recovery` (garde 0, point 2). N'écrit jamais `dgpu_disable`, ne
   redémarre jamais. `pending_reboot` passe à `1`.
5. **`local_ai`** — écrit le paramètre pilote D16
   (`/etc/modprobe.d/`), déploie le service Ollama par Quadlet. Dépend
   de `gpu_cdi` (`verify-cdi-spec`, gardé explicitement par ce rôle
   lui-même). Le service démarre et répond immédiatement — rien en
   aval n'attend le redémarrage, seule la **valeur** du paramètre
   pilote l'attend.
6. **`editor`** — Helix, Kate (D12/D13). Indépendant de tout ce qui
   précède, placé ici par regroupement.
7. **`completion`** — `lsp-ai` compilé, rattaché à Helix. Dépend de
   `editor` (garde COR-1 : Helix installé) et de `local_ai` (service
   joignable, modèle de complétion présent sur disque — ce dernier
   suppose `--tags pull-models,untagged` de `local_ai` déjà joué une
   fois, **jamais** dans ce flux, § 4 plus bas).

**Puis, `post_tasks`** : relève `pending_reboot`
(`gpu_mux_mode`) et la valeur active de
`NVreg_PreserveVideoMemoryAllocations` (D16). Si l'un des deux
n'est pas dans l'état attendu, **le playbook s'arrête** (`ansible.builtin.fail`,
jamais un redémarrage déclenché) avec le message exact de ce qu'il
faut vérifier après redémarrage manuel. Sinon (les deux déjà
satisfaits — c'est le cas sur cette machine, § Ce qui a été vérifié),
il rapporte la reconstruction terminée.

**Redémarrage manuel, décision et déclenchement de l'opérateur.**
Après redémarrage, vérifier :
```
cat /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value   # attendu 1
cat /proc/driver/nvidia/params | grep -i PreserveVideoMemoryAllocations                  # attendu : 1
```
Puis relancer `ansible-playbook site.yml` sans argument particulier —
chaque rôle qui précède le point d'arrêt est idempotent (`changed=0`
si déjà appliqué, à l'exception de quelques opérations de lecture/
préparation intrinsèquement toujours « changed » par nature, jamais
d'écriture système répétée — voir
[`roles/bootstrap/README.md`](../roles/bootstrap/README.md)) : rien de
déjà fait n'est rejoué, seul le point d'arrêt est réévalué, et cette
fois franchi.

### Un seul redémarrage, pas deux — établi par lecture, pas supposé

**Question posée par la revue** (`docs/review-2026-08.md` § 7.2) : les
deux redémarrages requis (`gpu_mux_mode`, D2bis/D2ter ; paramètre
pilote, D16) peuvent-ils être combinés en un seul ? **Oui**, établi par
lecture des deux mécanismes de prise d'effet, pas par exécution :

- `gpu_mux_mode` est appliqué par `asus-shutdown.service`, ordonné
  `Before=shutdown.target reboot.target halt.target` (établi,
  `docs/machine-facts.md` § GPU § Services, `systemctl cat
  asus-shutdown.service`) — il s'exécute pour un `reboot` comme pour un
  `poweroff`, pas seulement ce dernier.
- Le paramètre pilote D16 prend effet « au prochain chargement du
  module `nvidia.ko` » (établi, `docs/local-ai.md` § 7.1) — tout
  redémarrage complet décharge puis recharge ce module, ce qui suffit.

Les deux mécanismes appartiennent à des sous-systèmes indépendants
(routage MUX au niveau firmware contre comportement du pilote NVIDIA à
la suspension système) — aucune interaction connue, documentée ni
plausible entre eux.

**Écart résiduel, nommé plutôt que masqué** : cette combinaison n'a
**jamais été vérifiée empiriquement** sur ce poste — les deux
changements ont historiquement été appliqués par deux redémarrages
séparés, à des dates différentes (2026-08-05 pour D2bis/D2ter,
2026-08-07 pour D16). C'est pourquoi `site.yml` vérifie les **deux**
faits séparément après le redémarrage plutôt que de supposer que l'un
implique l'autre — si un jour l'un des deux échoue alors que l'autre
réussit, ce point d'arrêt le distinguerait plutôt que de le masquer.

## 3. Étiquettes — rejouer un rôle seul

```
ansible-playbook site.yml --tags gpu_mux        # un seul rôle
ansible-playbook site.yml --tags local_ai,completion   # plusieurs
ansible-playbook --check site.yml                # simulation complète
```

## 4. Ce qui reste hors de ce flux, délibérément

- **`--tags pull-models,untagged` de `roles/local_ai/`** — jamais dans
  `site.yml` (natif au tag `never` déjà porté par ces tâches,
  `roles/local_ai/tasks/main.yml` — comportement Ansible standard sur
  les tags `never`, rien ajouté ici pour le garantir). Choix et
  téléchargement de modèle restent un geste délibéré, distinct de la
  reconstruction de l'infrastructure (voir `roles/local_ai/README.md`).
- **Le déplacement D9 sur une machine qui a DÉJÀ une règle
  fonctionnelle** (tag `bootstrap-sudoers` sur une machine autre que
  neuve) — prudence spécifique à COR-2, § 1 ci-dessus.

## 5. Ce qui a été vérifié, et ce qui ne l'a pas été

**Vérifié, dans cette série** : `--syntax-check` complet ;
`ansible-lint --profile production` sur `site.yml` et tous les rôles
qu'il inclut, 0 défaut ; `--check` complet — `bootstrap`, `recovery`,
`gpu_cdi`, `gpu_mux`, `local_ai`, `editor`, `completion` s'enchaînent
**tous**, `local_ai` compris (§ ci-dessous), sans erreur en
simulation ; le point d'arrêt post-redémarrage a été exercé dans son
état **nominal réel** de cette machine (`pending_reboot=0`,
`PreserveVideoMemoryAllocations: 1` — les deux déjà satisfaits ici,
héritage de redémarrages antérieurs à cette série) et a correctement
**laissé passer** sans s'arrêter, plutôt que de s'arrêter à tort.

**[CORRIGÉ le 2026-08-08, ORC-2] ÉTAIT : trouvé en testant, non
corrigé (hors périmètre strict de COR-2)** —
`roles/local_ai/defaults/main.yml` (`local_ai_gpu_cdi_playbook`)
résolvait un chemin via `playbook_dir` en supposant que ce rôle est
**toujours** invoqué comme playbook autonome (`roles/local_ai/local_ai.yml`,
où `playbook_dir` vaut `roles/local_ai/`) — faux quand ce rôle est
inclus depuis `site.yml` (`playbook_dir` vaut alors la racine du
dépôt), ce qui faisait échouer `--check site.yml` complet.
**Corrigé** : `playbook_dir` remplacé par `role_path` (répertoire du
rôle **propriétaire de la tâche courante**, stable quel que soit
l'appelant — établi par lecture du code source d'Ansible avant de
corriger, `docs/machine-facts.md` § Décisions, journal ORC-2).
Démontré dans les trois contextes d'exécution requis (rôle seul
depuis la racine du dépôt, rôle depuis `site.yml`, l'un des deux
depuis un répertoire de travail différent) — la garde CDI réussit
identiquement dans les quatre combinaisons testées — plus la
démonstration inverse (chemin délibérément invalide via `-e` : la
garde casse bruyamment, message exact, avant toute action). Recherche
systématique sur toute construction de chemin dépendant implicitement
du point d'entrée d'exécution, dans tous les rôles : rien trouvé
au-delà de cette occurrence.

**Jamais vérifié, et ne peut pas l'être depuis cette machine** :
- La séquence **complète** de bout en bout, sur une machine qui n'a
  aucun état préalable — cette machine a déjà D6/D9/D10 satisfaites,
  `gpu_mux_mode`/le paramètre pilote déjà à leur valeur cible, et un
  historique d'exécutions antérieures de chaque rôle. Rien ici ne
  prouve que la séquence réussirait sur une machine qui n'a **jamais**
  vu aucun de ces rôles.
- Le comportement du point d'arrêt dans son état **négatif réel**
  (`pending_reboot=1` ou paramètre pilote non conforme, produit par un
  vrai `gpu_mux`/`local_ai` venant de s'exécuter) — seulement exercé
  dans son état positif (rien à faire) sur cette machine, faute de
  pouvoir provoquer légitimement l'état négatif sans redémarrer
  ensuite (interdit dans ce livrable).
- L'amorçage réel avec `--ask-become-pass` sur un compte qui n'a
  **aucune** règle `sudo` préalable — cette machine a `wheel` avec
  accès sudo de toute façon (au minimum interactif), `--ask-become-pass`
  n'a jamais été testé comme **seule** voie d'accès privilégié.
- Les prérequis § 0 (RPM Fusion, pilote NVIDIA, `git`/`ansible-core`)
  — déjà en place ici depuis longtemps, jamais rejoués depuis zéro.

**Ce qu'il faudrait pour vérifier la séquence complète** : une machine
jetable (VM ou matériel de test, idéalement avec le même GPU pour les
rôles `gpu_mux`/`gpu_cdi`/`local_ai` — une VM sans RTX 4090 ne
prouverait que la moitié de la séquence) provisionnée Fedora 44 nu,
sans aucun des prérequis § 0 — un livrable distinct, avec l'accord
explicite de l'opérateur avant toute exécution qui redémarre ou
modifie des paquets système pour de vrai. **Ce document ne présente
pas ce qui précède comme une reconstruction prouvée** — seulement
comme une séquence cohérente, vérifiée autant que possible sans
redémarrer ni provisionner de machine neuve.

## Voir aussi

- [`docs/review-2026-08.md`](review-2026-08.md) § 7 — le constat qui a
  motivé ce document, et le graphe de dépendances reconstitué par
  lecture qui en est le matériau de départ.
- [`docs/machine-facts.md`](machine-facts.md) § Décisions — D2bis/D2ter,
  D6, D9, D10, D14-D22, texte complet de chaque décision reconstruite
  ici.
- [`roles/bootstrap/README.md`](../roles/bootstrap/README.md) — D6/D9/D10,
  amorçage, tag `bootstrap-sudoers`.
