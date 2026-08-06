# Chaîne Ansible EE-first (D3) — résolution en lecture seule

**Ce document ne configure rien.** Toutes les commandes citées ont été
exécutées le 2026-08-06, en lecture seule : aucun paquet installé, aucune
image téléchargée (au sens couches d'image — seules des métadonnées de
manifeste/config ont été interrogées, précision § Validation), aucun EE
construit, aucune authentification à un registre, aucun dépôt ajouté,
aucun fichier écrit hors de ce dépôt. La décision entre les options du
§ 4 vient dans un livrable séparé, après revue.

Faits déjà établis, non revérifiés ici : `ansible-core` 2.20.7 sur
Python 3.14 (système) ; `ansible-builder` 3.1.0-8.fc44 disponible dans
Fedora ; `ansible-navigator`, `ansible-runner`, `ansible-lint` absents
des dépôts activés ; `pipx` 1.15.0-2.fc44 et `python3-pip` 26.0.1-2.fc44
disponibles mais non installés ; Podman 5.8.4 rootless, `crun`,
`overlay`.

## 1. D'où viennent les Execution Environments ?

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

### 1.6 — Conclusion sur D3 : la fidélité annoncée n'est pas atteignable telle qu'énoncée

D3 ne nomme pas explicitement d'où viendrait l'EE de référence, mais son
intention (« vérifier dans l'image d'Execution Environment ») implique
la fidélité à AAP 2.6. **Cette fidélité n'est atteignable par aucune
image déjà construite et publiquement accessible** (§ 1.2, § 1.3) — la
seule image alignée sur AAP 2.6 est derrière une authentification que
D4 interdit d'utiliser sur ce poste. Elle reste atteignable **par
construction locale** avec `ansible-builder` (§ 1.4), à partir d'une
base entièrement publique. **D3 n'est pas invalidée dans son intention,
mais sa formulation actuelle (qui présume `ansible-navigator` comme
moyen d'exécution, § 2, sans jamais avoir résolu la provenance de l'EE)
doit être amendée** — recommandation détaillée § 4, décision différée à
l'opérateur.

## 2. `ansible-navigator` : ce qu'il apporte réellement ici

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

## 3. Le problème Python 3.14

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

**Aucun `@VERIF` nécessaire ici** : les deux versions (2.16 par défaut,
2.18 en option) sont établies par une source externe publique nommée
précisément, sans authentification — pas besoin du bundle professionnel
de l'opérateur pour ce fait précis.

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
poste **satisfait** ce contrôle (`2.20.7 >= 2.20.0`) — ce qui explique
que la chaîne système fonctionne pour du lint générique, mais confirme,
chiffres à l'appui, ce que D3 affirmait qualitativement : **cet
`ansible-core` système ne peut pas être substitué à celui d'AAP 2.6 pour
valider du lint fidèle à la production, parce que la version qui
satisferait Python 3.14 et celle qui correspond à AAP 2.6 sont
mutuellement exclusives sur cette machine.**

## 4. Options, coûts, recommandation

Contraintes intégrées : D1 (reconstructible depuis ce dépôt — écarte
toute image `latest` non épinglée par empreinte comme cible finale) ;
D4 (aucun identifiant, aucune donnée d'entreprise — écarte tout chemin
qui authentifie contre `registry.redhat.io`) ; `roles/recovery`,
`gpu_mux`, `gpu_cdi` sont des rôles d'amorçage à repasser au lint une
fois la chaîne établie, pas à ce livrable.

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

## Recommandation

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

## Validation

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

**Décompte du jeton de vérification, `CLAUDE.md` exclu (§ 0.1)** —
piège méthodologique rencontré en écrivant cette section même :
compter ce jeton *dans un fichier qui décrit ce comptage* est
auto-référentiel — écrire le nombre trouvé, ou même l'exemple de
commande, ajoute une occurrence de plus et rend le chiffre qu'on vient
d'écrire déjà faux. Pour ne pas reproduire ici, sous une autre forme, le
défaut exact du § 0.1 (confondre une mention du jeton avec un marqueur
actionnable), ce document ne fige aucun chiffre en dur : le compte brut
final, mesuré après la dernière écriture de ce fichier, est donné dans
le rapport de livrable qui accompagne la poussée, pas ici.
Ventilation qualitative, stable indépendamment du compte exact :
- `docs/ansible-chain.md` porte **un seul marqueur actionnable** (§ 4,
  version `ansible-core` réellement en production, bornée) — le reste
  des occurrences du jeton dans ce fichier sont des mentions en prose
  (cette section, le renvoi dans « Voir aussi », la phrase confirmant
  qu'aucun marqueur n'était nécessaire au § 3.1).
- `docs/machine-facts.md` porte **cinq marqueurs actionnables** après
  ce livrable : trois déjà présents avant lui et non revérifiés ici
  (clé Terra, déclenchement d'`asus-shutdown.service`, canal `claude
  doctor`), un nouveau (§ 0.2, écart de taille CDI), et un nouveau qui
  remplace l'ancien marqueur D3 fermé par ce livrable (version
  `ansible-core` cible, 2.16 ou 2.18) — l'ancien marqueur D3 est retiré
  de sa forme active et conservé en historique (§ D3, note du
  2026-08-06), pas compté une deuxième fois.

**Confirmations finales** : aucun paquet installé (`rpm -q` négatif sur
les quatre outils recherchés) ; aucune image téléchargée au sens couches
(seules des métadonnées interrogées, précision ci-dessus) ; aucun EE
construit ; aucune authentification à un registre (`--no-creds` partout,
`auth.json` absent aux trois emplacements, avant et après) ; aucun dépôt
`dnf`/`yum` ajouté ; aucun fichier écrit hors de ce dépôt.

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — D3 (chaîne Ansible
  EE-first), D4 (dépôt public), dette technique sur l'outillage Ansible.
- [`CLAUDE.md`](../CLAUDE.md) — règle sur le comptage `@VERIF` excluant
  ce fichier (§ Sourcing, ajoutée dans ce livrable) ; règle « avant
  d'agir : qu'est-ce que l'outil fait déjà lui-même ? ».
