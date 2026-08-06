# Chaîne Ansible (D3a/D3b) — lint local et EE différé, en lecture seule

**[REQUALIFIÉ le 2026-08-06]** Ce document a d'abord été écrit sous un
cadrage inversé : il traitait la fidélité à AAP 2.6 comme la question
posée par *ce dépôt*. **Ce n'est pas le cas** — la cible de
`workstation-config` est cette machine elle-même (`localhost`, Fedora
44), pas AAP 2.6/RHEL 9, géré ailleurs (préambule de `CLAUDE.md`). D3 a
été scindée en conséquence dans `docs/machine-facts.md` :
- **D3a** — la chaîne réellement active de ce dépôt. L'`ansible-core`
  **système** (2.20.7) fait autorité, puisque c'est lui qui exécute les
  rôles sur la cible réelle. Résolue et appliquée le 2026-08-06, § 0
  ci-dessous.
- **D3b** — une chaîne EE-first à fidélité AAP 2.6, différée, pour le
  jour où ce poste développerait du contenu *destiné* à AAP. **Non
  ouverte.** Tout le contenu ci-dessous à partir du § 1 (écrit sous
  l'ancien cadrage, avant la correction) porte en réalité sur D3b, pas
  sur la chaîne active de ce dépôt — requalifié section par section,
  **aucun fait sourcé n'est supprimé**, seuls les titres et les
  conclusions changent de portemanteau.

**Ce document ne configure rien** au-delà de ce qui est explicitement
marqué comme appliqué (§ 0 uniquement — installation d'`ansible-lint`,
seule action du 2026-08-06 qui écrit sur ce poste). Tout le reste (§ 1 à
§ 4) reste une résolution en lecture seule sur D3b : aucun paquet
installé pour cette partie, aucune image téléchargée (au sens couches
d'image — seules des métadonnées de manifeste/config ont été
interrogées, précision § Validation), aucun EE construit, aucune
authentification à un registre, aucun dépôt ajouté, aucun fichier écrit
hors de ce dépôt. La décision d'ouvrir D3b, et le choix entre les
options du § 4, restent différés à un livrable séparé.

Faits déjà établis, non revérifiés ici : `ansible-core` 2.20.7 sur
Python 3.14 (système) ; `ansible-builder` 3.1.0-8.fc44 disponible dans
Fedora ; `ansible-navigator`, `ansible-runner`, `ansible-lint` absents
des dépôts activés (avant le 2026-08-06, § 0) ; `pipx` 1.15.0-2.fc44 et
`python3-pip` 26.0.1-2.fc44 disponibles ; Podman 5.8.4 rootless, `crun`,
`overlay`.

## 0. D3a — chaîne active de ce dépôt : lint local contre l'`ansible-core` système

**Question posée, propre à D3a** (pas celle des § 1-4, qui portent sur
D3b) : `ansible-lint` dépend d'`ansible-core`. L'installer dans un
environnement isolé (`pipx`, venv classique) y installerait un
**second** `ansible-core`, potentiellement différent du 2.20.7 système
qui exécute réellement les rôles de ce dépôt — un lint qui valide contre
une version différente de celle d'exécution reproduit exactement le
décalage qu'on cherche à éviter, déplacé d'un cran. Établi avant toute
installation, par métadonnées et par un essai `--dry-run` (n'installe
rien, montre seulement ce qui serait fait) :

```
$ python3 -m venv /tmp/.../isolated   # témoin, SANS --system-site-packages
$ /tmp/.../isolated/bin/pip install --dry-run ansible-lint
Collecting ansible-core>=2.16.14 (from ansible-lint)
  Downloading ansible_core-2.21.2-py3-none-any.whl.metadata
Would install ... ansible-core-2.21.2 ansible-lint-26.6.0 ...
```
Un venv isolé **installerait un second `ansible-core` (2.21.2, distinct
du 2.20.7 système)** — exactement le piège nommé par la demande.

```
$ python3 -m venv --system-site-packages /tmp/.../shared
$ /tmp/.../shared/bin/pip install --dry-run ansible-lint
Requirement already satisfied: ansible-core>=2.16.14 in
  /usr/lib/python3.14/site-packages (from ansible-lint) (2.20.7)
Would install ansible-compat-26.6.0 ansible-lint-26.6.0 ... (jamais ansible-core)
```
Avec `--system-site-packages`, `pip` détecte l'`ansible-core` **système**
(métadonnées `ansible_core-2.20.7.dist-info` déjà présentes sous
`/usr/lib/python3.14/site-packages/`, déposées par le paquet `dnf`) comme
satisfaisant la dépendance, et **ne le duplique pas** — seuls
`ansible-lint` et ses dépendances propres (`black`, `yamllint`, etc.,
aucune ne recoupant `ansible-core`) sont installés dans le venv.

**Coût de chaque voie** :
- Isolé (option écartée) : garantit zéro version unique — `ansible-lint`
  validerait contre un `ansible-core` que rien n'exécute réellement sur
  ce poste. Aucune mesure ne rendrait cette voie fiable sans figer
  manuellement une version, ce qui la ramènerait au mur démontré au § 3.2
  pour D3b (sans objet ici puisque D3a vise l'`ansible-core` système, pas
  celui d'AAP — mais la mécanique de résolution `pip` est la même).
- `--system-site-packages` (retenue) : coût nul en fidélité — une seule
  version d'`ansible-core` en jeu, celle qui exécute réellement les
  rôles. Coût réel : dépendance à ce que `dnf` continue de déposer des
  métadonnées `dist-info` standard (déjà le cas, vérifié) ; si un jour
  `ansible-core` était retiré du système, le venv perdrait sa seule
  source et `ansible-lint` cesserait de fonctionner tant qu'il ne serait
  pas réinstallé — comportement voulu, pas un défaut cette voie ne
  masque rien.

**Voie retenue : elle garantit une seule version d'`ansible-core` en
jeu**, mesurée en fin d'installation par la sortie de l'outil
lui-même, pas déduite :
```
$ python3 -m venv --system-site-packages ~/.venvs/ansible-lint
$ ~/.venvs/ansible-lint/bin/pip install ansible-lint
Successfully installed ansible-compat-26.6.0 ansible-lint-26.6.0 ... (17 paquets, aucun ansible-core)

$ ~/.venvs/ansible-lint/bin/pip show ansible-core
Location: /usr/lib/python3.14/site-packages   # le système, pas le venv
Version: 2.20.7

$ ansible --version | head -1                       # système
ansible [core 2.20.7]

$ ~/.venvs/ansible-lint/bin/ansible-lint --version   # venv
ansible-lint 26.6.0 using ansible-core:2.20.7 ansible-compat:26.6.0 ...
```
Les trois lectures s'accordent sur **2.20.7** — la version qui exécute
réellement les rôles de ce dépôt est celle contre laquelle
`ansible-lint` valide. Aucune copie d'`ansible-core` dans le venv,
confirmé (`find ~/.venvs/ansible-lint/lib/python3.14/site-packages
-iname "ansible_core*"` → vide).

**Action réalisée** : `sudo dnf install -y python3-pip` (dépôt `fedora`,
déjà activé — seul prérequis manquant pour faire tourner `pip`),
`python3 -m venv --system-site-packages ~/.venvs/ansible-lint`, `pip
install ansible-lint` dans ce venv. Aucune autre installation.

### 0.2 — Lint des rôles d'amorçage

`recovery`, `gpu_mux`, `gpu_cdi` : écrits sans lint, relus manuellement
(dette posée explicitement au livrable GPU-1). Passés au lint le
2026-08-06 avec l'outil résolu au § 0.1.

**Avant corrections** :
```
$ NO_COLOR=1 ~/.venvs/ansible-lint/bin/ansible-lint roles/recovery roles/gpu_mux roles/gpu_cdi
WARNING  Listing 9 violation(s) that are fatal
Failed: 9 failure(s), 0 warning(s) in 19 files processed of 23 encountered.
Last profile that met the validation criteria was 'min'.
rc=2
```
Neuf constats, un par un — **corrigés, aucun `noqa` posé par ce
livrable** (aucun ne relevait d'une règle inapplicable) :

| # | Règle | Fichier | Constat | Correction |
|---|---|---|---|---|
| 1 | `name[casing]` | `roles/gpu_cdi/gpu_cdi.yml` | Nom de play `gpu_cdi — …` ne commence pas par une majuscule | `GPU CDI — …` (même motif que `Recovery — …`, déjà conforme dans `roles/recovery/recovery.yml`) |
| 2 | `name[casing]` | `roles/gpu_mux/gpu_mux.yml` | Idem, `gpu_mux — …` | `GPU MUX — …` |
| 3 | `schema[meta]` | `roles/gpu_cdi/meta/main.yml` | `galaxy_info.author` manquant (règle non contournable — l'outil l'indique lui-même : « This rule is not skippable ») | `author: Mickael Mahieu` ajouté (identité déjà publique dans chaque commit git, D4) |
| 4 | `schema[meta]` | `roles/gpu_mux/meta/main.yml` | Idem | Idem |
| 5 | `schema[meta]` | `roles/recovery/meta/main.yml` | Idem | Idem |
| 6 | `name[template]` | `roles/gpu_cdi/tasks/main.yml:336` | Jinja `{{ gpu_cdi_spec_dir }}` au milieu du nom de tâche, pas à la fin | Reformulé, Jinja en fin (parenthèses) : « S'assurer que le répertoire cible existe ({{ gpu_cdi_spec_dir }}) » |
| 7 | `name[template]` | `roles/gpu_cdi/tasks/main.yml:346` | Idem, avec en plus du texte statique après le Jinja | Texte statique déplacé avant le Jinja : « Générer la spécification CDI (jamais /run/cdi ou /var/run/cdi, tmpfs) : {{ gpu_cdi_spec_path }} » |
| 8 | `yaml[line-length]` | `roles/gpu_mux/tasks/main.yml:74` | Ligne à 207 caractères (> 160) — `fail_msg` d'une garde `assert` | Repliée sur plusieurs lignes (scalaire replié `>-`, sémantique de rendu identique — vérifié, § ci-dessous) |
| 9 | `yaml[line-length]` | `roles/gpu_mux/tasks/main.yml:257` (254 avant la correction n°8) | Ligne à 186 caractères | Repliée sur plusieurs lignes (chaîne YAML entre guillemets, repliage natif — vérifié) |

**Garde touchée (constat n°8) : les deux démonstrations rejouées**,
conformément à la règle posée au livrable CDI-2 (une garde modifiée
perd la démonstration qui la validait). La tâche visée est l'`assert`
« Échouer bruyamment si gpu_mux_mode est absent ou si la valeur cible
n'est pas dans possible_values » — seul son `fail_msg` a été reformaté
(retour à la ligne), pas sa condition `that:`.
```
$ ansible-playbook --check roles/gpu_mux/gpu_mux.yml
ok: [localhost] => {"msg": "gpu_mux_mode présent, valeur cible 1 valide parmi 0;1."}

$ ansible-playbook --check -e gpu_mux_target_value=99 roles/gpu_mux/gpu_mux.yml
fatal: [localhost]: FAILED! => {"msg": "Valeur cible 99 absente de possible_values (0;1) — vérifier gpu_mux_target_value avant de relancer."}
```
Cas nominal : passe, message inchangé. Échec forcé (valeur hors
`possible_values`) : casse avec le message exact de la branche modifiée
— aucun saut de ligne parasite (vérifié aussi hors ligne, par un rendu
Jinja isolé reproduisant l'expression exacte : le repliage `>-` traite
les retours à la ligne entre jetons comme des espaces, y compris à une
indentation plus profonde, tant qu'ils ne coupent pas un littéral de
chaîne). Le second repliage (constat n°9, un message `debug`
d'information, pas une garde) a été vérifié de la même façon
(`yaml.safe_load` : chaîne repliée strictement identique à l'originale).

**Après corrections** :
```
$ NO_COLOR=1 ~/.venvs/ansible-lint/bin/ansible-lint roles/recovery roles/gpu_mux roles/gpu_cdi
Passed: 0 failure(s), 0 warning(s) in 19 files processed of 23 encountered.
Last profile that met the validation criteria was 'production'.
rc=0
```
Profil `production` atteint (le plus strict des profils d'`ansible-lint`),
`0` défaut. Le ratio « 19 fichiers traités sur 23 rencontrés » est
identique avant et après — comptage interne de l'outil (fichiers non
YAML/Ansible rencontrés en parcourant les rôles : `README.md` ×3,
`.gitignore`, un gabarit `.j2`, deux fichiers de trace JSON — ni erreur
ni périmètre réduit, `rc=0` et « 0 failure(s) » en attestent).

**Constat en marge, non corrigé, hors périmètre** : un commentaire
`# noqa: command-instead-of-module` préexistant dans
`roles/recovery/tasks/main.yml` (avant ce livrable, justifié en ligne
pour l'usage de `firewall-cmd --query-service` via `ansible.builtin.command`)
s'est révélé **inopérant** — testé en le retirant temporairement puis en
relançant le lint sur ce seul fichier : `Passed: 0 failure(s)` malgré
son absence. Cause établie par lecture du code de la règle elle-même
(`ansiblelint/rules/command_instead_of_module.py`, installé dans le
venv) : `firewall-cmd` ne figure pas dans la liste `_modules` de
commandes couvertes par cette règle — le constat que ce commentaire
anticipait ne se produit donc jamais avec ce jeu de règles, avec ou
sans le commentaire. Ni corrigé ni retiré : ce livrable ne modifie que
ce que le lint signale lui-même (consigne explicite), et ce commentaire
n'a été signalé par aucun constat de ce lint — fichier restauré à
l'identique après le test (`diff` confirmé). Consigné ici pour que la
prochaine session n'ait pas à le redécouvrir.

**Idempotence des rôles inchangée après corrections** : les trois
diffèrent uniquement sur des noms/messages (aucune tâche ajoutée,
retirée, ni reconditionnée) — confirmé en comparant `--check` avant/après
via `git stash` (état exact du dépôt avant ce livrable, restauré) :

| Rôle | `ok`/`changed`/`skipped` avant | après |
|---|---|---|
| `recovery` | 10 / 0 / 0 | 10 / 0 / 0 |
| `gpu_mux` | 18 / 1 / 4 | 18 / 1 / 4 |
| `gpu_cdi` | 41 / 0 / 53 | 41 / 0 / 53 |

Le `changed=1` de `gpu_mux` est préexistant (la tâche d'écriture de
`gpu_mux_mode` prédit un changement en `--check`, indépendamment de ce
livrable) — identique dans les deux relevés, pas une régression
introduite ici.

## 1. D3b (différée) : d'où viendraient les Execution Environments ?

### 1.1 — Registre officiel AAP : authentification exigée, démontré sans s'authentifier

```
$ skopeo inspect --no-creds docker://registry.redhat.io/ansible-automation-platform-26/ee-supported-rhel8:latest
[ERROR]: unable to retrieve auth token: invalid username/password: unauthorized:
Please login to the Red Hat Registry using your Customer Portal credentials.
Further instructions can be found here: https://access.redhat.com/RegistryAuthentication

$ skopeo inspect --no-creds docker://registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:latest
[même erreur]
```
`--no-creds` : aucune tentative d'authentification, aucun identifiant
fourni ni stocké. L'échec porte sur le **manifeste lui-même** — même la
liste des tags n'est pas accessible sans compte. Confirmé sans
ambiguïté : **aucune image d'EE officielle AAP n'est accessible sans
compte Red Hat**, quelle que soit l'opération (pas seulement le
téléchargement des couches).

Cohérent avec la configuration locale, jamais modifiée pour cette
recherche :
```
$ grep unqualified-search-registries /etc/containers/registries.conf
unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "docker.io"]
```
`registry.redhat.io` (AAP) n'est **pas** dans la liste de recherche par
défaut — à distinguer de `registry.access.redhat.com` (images RHEL/UBI
publiques, sans rapport avec ce point). Aucun identifiant stocké sur ce
poste, vérifié aux deux emplacements documentés (`man
containers-auth.json`, lu localement, § 1.3) :
```
$ ls /run/user/1000/containers/auth.json ~/.config/containers/auth.json ~/.docker/config.json
No such file or directory (les trois)
```

### 1.2 — Images publiques, sans compte : lesquelles, et laquelle est encore maintenue

**`quay.io/ansible/creator-ee`** (l'image communautaire historiquement
citée pour ce cas d'usage) :
```
$ skopeo inspect --no-creds docker://quay.io/ansible/creator-ee:latest
Created: 2024-02-08T13:57:59Z
Labels: version=39 (Fedora 39), org.opencontainers.image.source=https://github.com/ansible/creator-ee
```
Accessible sans compte — mais son dépôt source (`github.com/ansible/creator-ee`,
lu directement, externe) est **archivé depuis le 2024-08-26**, redirige
vers un successeur. Le tag `latest` de cette image n'a plus bougé
depuis 2024-02 : **base de départ obsolète, à ne pas retenir**.

**`ghcr.io/ansible/community-ansible-dev-tools`** (successeur, cité par
la documentation amont du projet `ansible-dev-tools`,
`docs.ansible.com/projects/dev-tools/container/`, consultée sans
authentification) :
```
$ skopeo inspect --no-creds docker://ghcr.io/ansible/community-ansible-dev-tools:latest
Created: 2026-07-24T07:22:58Z
Labels: org.opencontainers.image.title=fedora-minimal, version=44
        org.opencontainers.image.source=https://github.com/ansible/ansible-dev-tools
```
Accessible sans compte, activement maintenue (construite il y a moins
de deux semaines au moment de cette lecture), **basée sur Fedora 44** —
même version que ce poste.

**Sa base publique elle-même**, citée dans `execution-environment.yml`
du dépôt source (lu via l'API GitHub, externe, sans authentification) :
```
$ skopeo inspect --no-creds docker://quay.io/fedora/fedora-minimal:44
Created: 2026-08-06T06:47:54Z   # reconstruite le jour même de cette lecture
```
Chaîne de provenance entièrement publique et actuelle, de bout en bout :
`quay.io/fedora/fedora-minimal:44` → `ghcr.io/ansible/community-ansible-dev-tools`.

### 1.3 — Ce que l'image communautaire publique ne résout PAS : la fidélité de version

Point décisif, établi par lecture directe du dépôt source
`ansible/ansible-dev-tools` (`pyproject.toml`, externe, sans
authentification) :
```
dependencies: ansible-lint>=26.4.0, ansible-navigator>=26.1.3, molecule>=26.3.0
[tool.tox.env.milestone] ansible-core@ https://github.com/ansible/ansible/archive/milestone.tar.gz
```
`ansible-core` n'est **pas épinglé** dans les dépendances principales du
projet — seul un environnement de test `milestone` le référence, contre
le dépôt git `ansible` (branche de développement). L'image publique
`community-ansible-dev-tools` embarque donc, au moment de sa
construction, la version d'`ansible-core` **la plus récente compatible
avec `ansible-lint`** — exactement le même profil de fraîcheur que
l'`ansible-core` 2.20.7 déjà présent sur ce poste (§ 3). **Consommer
cette image publique ne rapproche pas de la fidélité à AAP 2.6 — elle
déplace seulement le même écart de version dans un conteneur.**

### 1.4 — `ansible-builder` : ce qu'il permet, et son coût réel de fidélité

```
$ dnf --disablerepo=terra info ansible-builder
Version 3.1.0-8.fc44, dépôt fedora, résumé : "A tool for building Ansible
Execution Environments"
$ dnf --disablerepo=terra repoquery -l ansible-builder
/usr/bin/ansible-builder
/usr/lib/python3.14/site-packages/ansible_builder/...
```
Paquetage Fedora pour Python 3.14 système — installable par `dnf`
directement, sans `pip`/`pipx`. Confirmé par documentation amont
(`docs.ansible.com/projects/builder/`, externe) : `ansible-builder`
accepte **n'importe quelle image de base** choisie par l'utilisateur,
pas seulement celles de Red Hat — il n'est pas structurellement lié à
`registry.redhat.io`.

**Coût en fidélité, précisément situé** : `ansible-builder` construit un
EE à partir d'un `execution-environment.yml` qui déclare l'`ansible-core`
et les collections voulus — la version d'`ansible-core` embarquée est
**ce qu'on lui demande d'installer**, pas une propriété de l'outil
lui-même. Cela signifie qu'un EE construit avec `ansible-builder`, à
partir d'une base publique (`quay.io/fedora/fedora-minimal:44` ou
équivalent), **peut épingler exactement `ansible-core==2.16.14` ou
`2.18.9`** (les deux versions documentées pour AAP 2.6, § 3) — sans
jamais consommer l'image officielle ni s'authentifier où que ce soit.
C'est la seule voie identifiée dans cette résolution qui atteint une
fidélité de version réelle sans registre authentifié. Elle a un coût
distinct : construire et maintenir ce `Containerfile`/`execution-environment.yml`
soi-même (non fait dans ce livrable, hors périmètre — § 4).

### 1.5 — Identifiants de registre : où ils vivraient, si le chemin authentifié était un jour retenu

Établi par lecture locale (`man containers-auth.json`, § 1.1), pas
supposé : le fichier primaire est
`${XDG_RUNTIME_DIR}/containers/auth.json` (`/run/user/1000/...` sur ce
poste — **hors `$HOME`, sur `tmpfs`, ne survit pas au redémarrage**),
avec un repli possible sur `~/.config/containers/auth.json` (persistant,
dans `$HOME`, donc **jamais dans ce dépôt** par construction — `git
status` ne verrait ce fichier que si quelqu'un le copiait explicitement
dans l'arborescence versionnée). Géré par `podman login`/`skopeo
login`/`buildah login`, jamais par ce dépôt. Si le chemin authentifié
était un jour retenu (hors périmètre de ce livrable), D4 continuerait
d'exclure catégoriquement ce fichier du dépôt — la contrainte technique
(emplacement hors arborescence) et la règle du dépôt (D4) coïncident
déjà, sans mesure supplémentaire à prendre.

### 1.6 — Conclusion sur D3b : la fidélité annoncée n'est pas atteignable telle qu'énoncée, si D3b est un jour ouverte

**[REQUALIFIÉ le 2026-08-06]** Cette conclusion portait initialement sur
« D3 » tout court, sous le cadrage inversé (§ note en tête de document)
— elle ne s'applique **qu'à D3b**, différée, pas à la chaîne active de
ce dépôt (D3a, § 0, qui n'a besoin d'aucun EE et n'est pas concernée par
la question « d'où viendrait l'EE de référence »).

Pour D3b, si elle est un jour ouverte : son intention (fidélité à AAP
2.6, cadrage EE-first) implique une fidélité de version. **Cette
fidélité n'est atteignable par aucune image déjà construite et
publiquement accessible** (§ 1.2, § 1.3) — la seule image alignée sur
AAP 2.6 est derrière une authentification que D4 interdit d'utiliser sur
ce poste. Elle reste atteignable **par construction locale** avec
`ansible-builder` (§ 1.4), à partir d'une base entièrement publique.
Recommandation détaillée § 4, décision d'ouvrir D3b différée à
l'opérateur.

## 2. `ansible-navigator` : ce qu'il apporte réellement ici

**Analyse valable pour D3a et pour D3b** — contrairement au reste de ce
document (§ 1, § 3, § 4, requalifiés comme portant sur D3b), ce constat
ne dépendait pas du cadrage inversé initial : que la question soit
« exécuter contre l'`ansible-core` système » (D3a) ou « exécuter dans un
EE fidèle à AAP » (D3b), `ansible-navigator` n'apporte de capacité
manquante dans aucun des deux cas. Non requalifié, conservé tel quel.

Documentation amont lue (`docs.ansible.com/projects/navigator/subcommands/`,
externe, sans authentification) — sous-commandes pertinentes pour
l'usage visé (développement local de rôles, `--syntax-check`, lint,
exécution contre `localhost` et des hôtes distants) :

| Sous-commande | Rôle | Au-delà d'un `podman run` direct ? |
|---|---|---|
| `run` | Exécute un playbook dans l'EE | Non — gère les montages de volumes et l'inventaire automatiquement, mais un `podman run -v ...` scripté fait la même chose |
| `exec` | Exécute une commande dans l'EE | Non — équivalent direct |
| `lint` | Lance `ansible-lint` dans l'EE | Non — invoque l'outil déjà présent dans l'image ; `podman run <EE> ansible-lint ...` produit le même résultat |
| `doc`/`config`/`inventory`/`collections`/`images`/`replay`/`welcome` | Consultation interactive (TUI) | Oui — mais sans objet pour un flux scripté, non interactif, tel que celui déjà en usage dans ce dépôt (`roles/recovery`, `gpu_mux`, `gpu_cdi`, tous exécutés en CLI directe) |

**Constat pour cet usage précis** : aucune capacité de `run`/`exec`/`lint`
— les trois sous-commandes réellement utiles au flux de travail décrit
par la demande — n'est indisponible en `podman run` direct. Le reste de
la valeur ajoutée (TUI, `replay`) répond à un usage interactif ou
d'audit de production que ce dépôt n'a pas (la cible de production
réelle est gérée ailleurs — préambule de `CLAUDE.md`).

**Coût d'installation, à charge de la conclusion** : `ansible-navigator`
n'est pas dans les dépôts Fedora activés (déjà établi) ; l'installer
exigerait `pipx` ou un venv — ce qui réintroduit exactement le problème
de fidélité Python/`ansible-core` traité au § 3, pour un outil dont la
valeur ajoutée, sur ce cas d'usage précis, est nulle en capacité et
seulement ergonomique. **Conforme à la règle « une couche de moins vaut
mieux qu'une couche de plus » (`CLAUDE.md` § Avant d'agir)** : rien
n'indique qu'installer `ansible-navigator` soit nécessaire ici.

## 3. Le problème Python 3.14 (spécifique à D3b — sans objet pour D3a)

**[REQUALIFIÉ le 2026-08-06]** Ce mur ne concerne que D3b (fidélité à
AAP 2.6). **Il ne bloque rien pour D3a** : le système porte
`ansible-core` 2.20.7, qui satisfait déjà le contrôle décrit ci-dessous
(`2.20.7 >= 2.20.0`) — c'est exactement pour cela que l'installation du
§ 0.1 fonctionne sans y toucher. Ce mur ne redevient pertinent que si
D3b est ouverte : viser une fidélité aux versions réelles d'AAP 2.6
(2.16/2.18, § 3.1 ci-dessous) sous Python 3.14 s'y heurterait, ce que
cette section démontre.

### 3.1 — Version d'`ansible-core` dans AAP 2.6 (sourcé, sans authentification)

Deux pages publiques Red Hat, accessibles sans compte (marquées
externes) :
- `access.redhat.com/support/policy/updates/ansible-automation-platform-execution-environments`
  (page de politique de support, publique) : tableau 1.2, **« AAP
  version: 2.6 | Default Ansible-core: ansible-core 2.16 »**. EE
  associées : `ee-minimal-rhel8/9`, `ee-supported-rhel8/9` — exactement
  les noms testés sans authentification au § 1.1.
- `redhat.com/en/blog/planning-your-upgrade-path-ansible-automation-platform-26`
  (article public) : confirme un **EE optionnel avec `ansible-core`
  2.18** pour AAP 2.6, présenté comme apportant des changements plus
  significatifs que la version par défaut.

**Aucun jeton de vérification nécessaire ici** : les deux versions (2.16
par défaut, 2.18 en option) sont établies par une source externe
publique nommée précisément, sans authentification — pas besoin du
bundle professionnel de l'opérateur pour ce fait précis.

### 3.2 — Installabilité d'`ansible-lint`/`ansible-core` sous Python 3.14 : requête de métadonnées, aucune installation

**`ansible-lint` lui-même** — métadonnées PyPI de la version publiée
(`pypi.org/pypi/ansible-lint/json`, externe, API publique, aucune
installation) :
```
requires_python: >=3.10
distributions: ansible_lint-26.6.0-py3-none-any.whl (roue pure Python, universelle)
requires_dist (extrait) :
  ansible-core>=2.16.14
  cffi>=1.15.1
  cryptography>=37
  pyyaml>=6.0.1                                  # (les deux branches, <3.14 et >=3.14)
  ruamel-yaml-clib>=0.2.12 ; python_version < "3.14"   # explicitement EXCLU pour 3.14+
```
La roue d'`ansible-lint` est universelle (`py3-none-any`) : rien
n'empêche son **installation** sous Python 3.14 au niveau de pip. Ses
dépendances compilées ont toutes une roue publiée pour `cp314` à la date
de cette lecture (`pypi.org/pypi/<paquet>/json`, extrait des noms de
fichiers) :
```
cryptography 50.0.0 : cp314 présent
cffi        2.1.1  : cp314 présent
pyyaml      6.0.3  : cp314 présent
ruamel.yaml.clib 0.2.15 : cp314 présent (mais explicitement exclu par
                          ansible-lint pour python>=3.14 — § ci-dessus,
                          contrainte posée par le projet, pas une
                          absence de roue)
```
**Aucun blocage d'installation au niveau des roues publiées.**

**Le blocage réel n'est pas là — il est dans un contrôle applicatif à
l'exécution**, trouvé par lecture directe d'une discussion amont
(`github.com/ansible/ansible-lint/issues/4822`, externe, publique) :
```
"Python 3.14 requires ansible-core version >= 2.20.0, and we found 2.19.3."
```
Ce contrôle (porté par `ansible_compat`, dépendance d'`ansible-lint`)
**refuse explicitement l'exécution** d'`ansible-lint` sous Python 3.14
si l'`ansible-core` apparié a moins de `2.20.0` — indépendamment de ce
que pip a accepté d'installer. Ticket fermé « not planned » côté projet
amont : ce n'est pas un bogue transitoire, c'est une incompatibilité
assumée par le projet lui-même.

**Conséquence directe, chiffrée** : `ansible-core` 2.16.14 et 2.18.9
(les deux versions réelles d'AAP 2.6, § 3.1) sont **toutes deux
strictement inférieures à `2.20.0`**. Confirmé par leurs métadonnées
PyPI propres (aucune installation) :
```
ansible-core 2.16.14 : requires_python >= 3.10 (roue py3-none-any, pas de borne haute)
ansible-core 2.18.9  : requires_python >= 3.11 (roue py3-none-any, pas de borne haute)
```
Aucune des deux ne déclare de borne haute sur Python — `pip install`
les accepterait sous 3.14 — mais **`ansible-lint`, apparié à l'une ou
l'autre, refuserait de s'exécuter sous Python 3.14** (le contrôle
`ansible_compat` ci-dessus). Le système `ansible-core 2.20.7` de ce
poste **satisfait** ce contrôle (`2.20.7 >= 2.20.0`) — c'est précisément
ce qui rend D3a possible sans aucun mur (§ 0, § 3 en tête). Mais chiffres
à l'appui : **cet `ansible-core` système ne peut pas être substitué à
celui d'AAP 2.6 pour valider du lint fidèle à *cette* production-là**
(D3b, différée) — **la version qui satisfait Python 3.14 et celle qui
correspond à AAP 2.6 sont mutuellement exclusives sur cette machine.**
Sans rapport avec D3a, qui ne vise jamais la fidélité AAP.

## 4. D3b — options, coûts, recommandation

**Portée requalifiée** : les quatre options ci-dessous évaluent
comment atteindre une fidélité de version à AAP 2.6 (D3b, différée) —
pas comment lint ce dépôt lui-même (D3a, résolu au § 0 par une cinquième
voie, `--system-site-packages`, qui n'était pas dans ce périmètre
initial puisque D3b et D3a n'étaient pas encore distinguées). Contraintes
intégrées : D1 (reconstructible depuis ce dépôt — écarte toute image
`latest` non épinglée par empreinte comme cible finale) ; D4 (aucun
identifiant, aucune donnée d'entreprise — écarte tout chemin qui
authentifie contre `registry.redhat.io`). **[REQUALIFIÉ le 2026-08-06]**
ÉTAIT : « `roles/recovery`, `gpu_mux`, `gpu_cdi` sont des rôles
d'amorçage à repasser au lint une fois la chaîne établie, pas à ce
livrable » — fait depuis, par D3a (§ 0.2), sans attendre D3b.

### Option A — `pipx install ansible-lint`/`ansible-navigator`

**Apporte** : isolation par venv dédié, pas de pollution du site-packages
système, commande simple.
**Coûte** : `pip` résout, par défaut, l'`ansible-core` transitif le plus
récent compatible (`>=2.16.14`, sans borne haute) — très probablement
une version proche de 2.20+, pas 2.16/2.18. Épingler manuellement
`ansible-core==2.16.14` romprait immédiatement le contrôle
`ansible_compat` (§ 3.2) sous Python 3.14 : **`ansible-lint`
refuserait de s'exécuter**. `pipx` ne fige pas non plus le graphe de
résolution dans un fichier versionné — deux installations à des dates
différentes peuvent résoudre des versions différentes, silencieusement.
**Risque** : illusion de fidélité (on croit tester « comme AAP » en
utilisant un `ansible-core` récent, sans base de comparaison figée).
**Mesurable ?** `pipx list` + `<outil> --version` donnent l'état
installé, mais rien ne garantit sa reproduction sans un fichier de
contraintes committé — absent de cette option par nature.

### Option B — venv dédié (`python3 -m venv` + `requirements.txt` épinglé)

**Apporte** : contrôle fin, `requirements.txt` versionnable dans ce
dépôt (reconstructible, D1) — la seule option locale qui pourrait, en
théorie, épingler `ansible-core==2.16.14`.
**Coûte** : le même mur que l'option A, mais démontré plus précisément
ici (§ 3.2) — épingler `ansible-core` à la version réelle d'AAP 2.6
**et** exécuter `ansible-lint` sous Python 3.14 sont deux exigences
**mutuellement exclusives** sur ce poste, pas seulement mal réglées par
défaut. `ansible-playbook` seul (sans `ansible-lint`) pourrait
fonctionner avec `ansible-core==2.16.14` sous Python 3.14 (le contrôle
`ansible_compat` est spécifique à `ansible-lint`, pas à `ansible-core`
lui-même — nuance établie mais non testée ici, aucune installation
réalisée) — mais perdrait alors la fonction de lint qui motive une
bonne partie du besoin.
**Risque** : le même que l'option A, en pire — la fausse impression de
rigueur (fichier de contraintes versionné) masque un mur d'exécution
qui, lui, n'est pas contournable par un simple pin.
**Mesurable ?** Oui pour l'installation (`pip freeze`), mais la mesure
qui compte (`ansible-lint` s'exécute-t-il vraiment avec cet
`ansible-core` ?) échouerait par construction dès qu'on viserait
2.16/2.18.

### Option C — EE conteneurisé seul, `podman run` direct (aucun outil local)

**Apporte** : l'EE embarque son propre Python et son propre
`ansible-core`, totalement découplés de Python 3.14 système — **seule
option qui élimine structurellement le mur du § 3.2**, puisque le
contrôle `ansible_compat` s'évalue contre le Python *du conteneur*, pas
celui de l'hôte. Zéro nouvelle installation sur l'hôte (« une couche de
moins »).
**Coûte** : aujourd'hui, aucune image publique déjà construite n'a la
version d'`ansible-core` d'AAP 2.6 (§ 1.3) — il faudrait construire
soi-même avec `ansible-builder` (§ 1.4), non fait dans ce livrable.
Ergonomie `podman run` (montages, options) à scripter soi-même — un
script de quelques lignes, pas une dépendance nouvelle.
**Risque** : dépendre d'une image `latest` flottante romprait D1 — la
recommandation doit s'accompagner d'un épinglage par empreinte, pas par
tag.
**Mesurable ?** Oui, directement : `podman run <EE> ansible --version`
rapporte la version exacte d'`ansible-core` embarquée, comparable sans
ambiguïté aux chiffres publics du § 3.1.

### Option D — combinaison : `ansible-builder` (déjà disponible par `dnf`) pour construire, `podman run` seul pour exécuter, rien d'autre installé

Reprend l'option C et ferme son seul manque : la construction. Puisque
`ansible-builder` est disponible par `dnf` (§ 1.4, pas de `pip`/`pipx`),
un EE minimal peut être défini (`execution-environment.yml` committé
dans ce dépôt, D1) à partir d'une base publique
(`quay.io/fedora/fedora-minimal:44` ou équivalent, épinglée par
empreinte une fois choisie), avec `ansible-core` et `ansible-lint`
épinglés aux versions réelles d'AAP 2.6 (2.16.14 ou 2.18.9, à trancher
par l'opérateur). `ansible-lint`, exécuté par le **Python du
conteneur** (pas celui de l'hôte), ne rencontre jamais le contrôle
`ansible_compat` bloquant du § 3.2, puisque ce Python n'a aucune raison
d'être 3.14. Aucun `ansible-navigator`, aucun `pipx`, aucun venv.
**Coûte** : le temps de définir et maintenir ce fichier — un artefact de
plus dans le dépôt, mais reconstructible (D1) et sans identifiant (D4).
**Mesurable ?** Oui, comme l'option C, plus une vérification supplémentaire
possible à la construction : le `ansible --version` du résultat doit
correspondre exactement à ce qui a été demandé dans
`execution-environment.yml` — écart détectable immédiatement.

## Recommandation (pour D3b, si elle est un jour ouverte — sans objet pour D3a, déjà résolue au § 0)

**Option D** — `ansible-builder` (déjà disponible par `dnf`, sans
`pip`/`pipx`) pour construire un EE minimal, épinglé à
`ansible-core` 2.16.14 ou 2.18.9 (AAP 2.6, § 3.1), à partir d'une base
publique épinglée par empreinte ; exécution exclusivement par `podman
run` direct, sans `ansible-navigator`. Ce qui la départage :

- Contre A et B : seule option qui ne se heurte pas au mur
  `ansible_compat`/Python 3.14 démontré au § 3.2 — les deux versions
  réelles d'AAP 2.6 sont structurellement incompatibles avec
  `ansible-lint` sous Python 3.14, quel que soit le soin apporté à
  l'épinglage local.
- Contre C seule (consommer une image déjà publiée) : aucune image
  publique actuellement disponible n'a la fidélité de version voulue
  (§ 1.3) — construire soi-même est la seule voie qui l'atteint sans
  authentification.
- Conforme à D1 (reconstructible, à condition d'épingler par empreinte,
  pas par tag flottant), D4 (aucun identifiant, base entièrement
  publique), et à la règle « une couche de moins vaut mieux qu'une
  couche de plus » (pas d'`ansible-navigator`, § 2).

**Ce qui reste à trancher avant d'appliquer cette option, pas dans ce
livrable** : quelle version cible entre `ansible-core` 2.16 (défaut AAP
2.6) et 2.18 (EE optionnel) — les deux sont réelles et documentées
(§ 3.1), le choix dépend de la version réellement utilisée en
production par l'opérateur, non lisible depuis ce poste sans le bundle
professionnel : `@VERIF : version d'ansible-core réellement utilisée en
production AAP 2.6 (2.16 par défaut ou 2.18 optionnel) — à confirmer par
l'opérateur depuis son environnement professionnel (bundle AAP,
console `automation controller`, ou `ansible --version` dans l'EE
réellement déployé), pas depuis ce poste.` Une fois cette version
choisie, la construction elle-même (`execution-environment.yml`,
`ansible-builder`, épinglage par empreinte) est un livrable séparé, hors
périmètre de cette résolution en lecture seule.

## Validation — D3a (ce livrable, 2026-08-06)

**Action privilégiée, unique, énumérée explicitement** (`CLAUDE.md` §
Avant d'agir) :

| # | Commande | Chemin cible | Motif |
|---|---|---|---|
| 1 | `sudo dnf install -y python3-pip` | paquet système, dépôt `fedora` (déjà activé) | seul prérequis manquant pour exécuter `pip` — nécessaire à la résolution du § 0.1 avant toute installation d'`ansible-lint` |

Aucune autre élévation. La création du venv
(`python3 -m venv --system-site-packages ~/.venvs/ansible-lint`) et
l'installation d'`ansible-lint` dedans (`pip install ansible-lint`)
s'exécutent sans `sudo` — écriture dans `$HOME`, pas dans `/usr` ni
`/etc`.

**Commandes non privilégiées, exhaustives** : essais `pip install
--dry-run` (×2, témoin isolé et `--system-site-packages`, § 0.1) ;
création du venv et installation réelle (§ 0.1) ; `pip show
ansible-core`, `ansible --version`, `ansible-lint --version`, `find`
(confirmation absence de copie, § 0.1) ; `ansible-lint` sur les trois
rôles, avant et après corrections (§ 0.2) ; `ansible-playbook
--syntax-check`/`--check` sur les trois rôles, avant (via `git stash`)
et après corrections (§ 0.2) ; test réversible du `noqa` mort dans
`roles/recovery/tasks/main.yml` (retiré temporairement, lint relancé,
restauré — `diff` confirmé identique à l'original) ; lecture du code
source de la règle `command-instead-of-module` dans le paquet installé
(fichier local, déjà présent dans le venv, pas une requête réseau).
Aucune sortie vide sans investigation ; aucun code de retour non nul
non expliqué (le seul rencontré, `rc=2` sur le lint « avant », est
l'échec attendu et documenté § 0.2).

**Confirmations finales D3a** : trois rôles au profil `production`,
`0` défaut ; `--syntax-check` et `--check` identiques avant/après sur
les trois rôles (table § 0.2) ; aucun `noqa` ajouté par ce livrable ;
aucune modification de `sudoers`, `dgpu_disable`, `gpu_mux_mode`,
`supergfxd`, `asus-shutdown`, `/etc/cdi/` ; `terra` non désactivé ;
aucun redémarrage ; aucun EE construit ; aucune image téléchargée ;
aucune authentification à un registre ; aucun nouveau dépôt système (le
seul paquet installé, `python3-pip`, vient d'un dépôt déjà activé).

## Validation — D3b (résolution en lecture seule, premier livrable du 2026-08-06, contenu inchangé)

**Commandes exécutées, toutes non modifiantes** :

| # | Commande | Nature | Sortie/code |
|---|---|---|---|
| 1 | `skopeo inspect --no-creds docker://quay.io/ansible/creator-ee:latest` | requête manifeste/config, anonyme, réseau | succès, JSON |
| 2 | `skopeo inspect --no-creds docker://registry.redhat.io/.../ee-supported-rhel8:latest` | requête manifeste, anonyme, réseau | échec attendu (401, message d'authentification), rc=1 |
| 3 | `skopeo inspect --no-creds docker://registry.redhat.io/.../ee-minimal-rhel9:latest` | idem | échec attendu identique, rc=1 |
| 4 | `skopeo inspect --no-creds docker://ghcr.io/ansible/community-ansible-dev-tools:latest` | requête manifeste/config, anonyme, réseau | succès, JSON |
| 5 | `skopeo inspect --no-creds docker://quay.io/fedora/fedora-minimal:44` | idem | succès, JSON |
| 6 | `man -w containers-auth.json` + lecture locale | lecture de page de manuel déjà installée | succès |
| 7 | `ls` sur les trois emplacements d'`auth.json` | lecture, confirme l'absence | absence confirmée, code non nul attendu (fichiers inexistants) |
| 8 | `grep unqualified-search-registries /etc/containers/registries.conf` | lecture de config déjà présente | succès |
| 9 | `dnf --disablerepo=terra info ansible-builder` | requête de métadonnées de dépôt (rafraîchit le cache dnf, n'installe rien) | succès |
| 10 | `dnf --disablerepo=terra repoquery -l ansible-builder` | idem, liste de fichiers déclarés par le paquet, sans l'installer | succès |
| 11 | `curl https://pypi.org/pypi/<paquet>/json` (×9 : ansible-lint, ansible-core, ansible-navigator, ansible-runner, cryptography, cffi, pyyaml, ruamel.yaml, ruamel.yaml.clib) | requête d'API publique, métadonnées seules | succès, JSON |
| 12 | `curl https://pypi.org/pypi/ansible-core/2.16.14/json` et `.../2.18.9/json` | idem, versions ciblées | succès, JSON |
| 13 | `rpm -q ansible-builder ansible-navigator ansible-lint ansible-runner` | confirmation finale d'absence | code non nul attendu (aucun installé) |
| 14 | `podman images` | confirmation finale, aucune nouvelle image locale | succès, seule l'image `nvidia/cuda` du livrable précédent présente |
| 15 | Recherches web et lectures de pages publiques (Red Hat `access.redhat.com`/`redhat.com`, GitHub `ansible/creator-ee`, `ansible/ansible-dev-tools`, `ansible/ansible-lint#4822`, `docs.ansible.com`) | requêtes HTTP GET, lecture seule | succès, sauf `raw.githubusercontent.com/.../requirements.txt` (404 — fichier inexistant à cet emplacement, pas une erreur d'accès) |

Aucune sortie vide sans investigation : la commande n°7 (recherche
d'`auth.json`) a un code non nul par construction (fichier absent, résultat
attendu et souhaité, pas une commande dont l'absence de sortie serait
ambiguë). La commande n°15 (`requirements.txt`) a été redirigée vers une
lecture différente (`pyproject.toml`, § 1.3) une fois le 404 constaté,
pas laissée comme un vide non expliqué.

**Précision sur « aucune image téléchargée »** : `skopeo inspect` (sans
`--raw` ni `copy`) ne récupère que le manifeste et le petit blob de
configuration JSON (quelques kilo-octets) — jamais les couches d'image
(les plus gros objets, plusieurs centaines de mégaoctets pour ces
images). Aucune commande de copie/tirage (`skopeo copy`, `podman pull`)
n'a été exécutée dans ce livrable.

**Actions privilégiées** : **aucune.** Toutes les commandes ci-dessus
s'exécutent sans `sudo`/`become` — confirmé par leur nature (lectures
réseau anonymes, lectures de fichiers déjà lisibles par l'utilisateur,
requêtes de métadonnées `dnf` qui n'écrivent que dans le cache
utilisateur/système déjà en place).

**Décompte du jeton de vérification, `CLAUDE.md` exclu.** Le livrable
précédent avait rencontré un piège méthodologique en écrivant cette
section même : compter ce jeton *dans un fichier qui décrit ce
comptage* est auto-référentiel — écrire le jeton nu (même dans
l'exemple d'une commande, même entre accents graves) ajoute une
occurrence de plus. **Cause traitée à la racine dans ce livrable**
(règle ajoutée à `CLAUDE.md` : le jeton ne s'écrit jamais nu hors d'un
marqueur réel — périphrase sinon) — cette section elle-même, comme le
reste du document, a été relue et corrigée pour n'écrire le jeton que
dans ses marqueurs réels. Le compte brut égale donc maintenant le
compte de marqueurs actionnables, sans ventilation à faire :
- `docs/ansible-chain.md` porte **un seul marqueur actionnable** (§ 4,
  version `ansible-core` réellement en production, bornée).
- `docs/machine-facts.md` porte **quatre marqueurs actionnables** :
  clé Terra, déclenchement d'`asus-shutdown.service`, canal `claude
  doctor` (les trois déjà présents avant ce livrable, non revérifiés
  ici), et l'écart de taille de la spécification CDI (§ 0.2 du livrable
  du 2026-08-06). L'ancien marqueur D3 (« voie d'installation... ») est
  fermé et conservé en historique, sans le jeton nu ; le marqueur sur la
  version cible d'`ansible-core` pour D3b vit uniquement ici, pas
  dupliqué dans `docs/machine-facts.md` (qui y renvoie par périphrase).

**Confirmations finales** : aucun paquet installé (`rpm -q` négatif sur
les quatre outils recherchés) ; aucune image téléchargée au sens couches
(seules des métadonnées interrogées, précision ci-dessus) ; aucun EE
construit ; aucune authentification à un registre (`--no-creds` partout,
`auth.json` absent aux trois emplacements, avant et après) ; aucun dépôt
`dnf`/`yum` ajouté ; aucun fichier écrit hors de ce dépôt.

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — D3a/D3b (scission du
  2026-08-06), D4 (dépôt public), dette technique sur l'outillage
  Ansible.
- [`CLAUDE.md`](../CLAUDE.md) — règles sur le jeton de vérification : ce
  fichier exclu de tout comptage, jeton jamais écrit nu en prose ; règle
  « avant d'agir : qu'est-ce que l'outil fait déjà lui-même ? ».
