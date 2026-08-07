# Surface d'approvisionnement logiciel — dépôts activés, ancrages de confiance

Première vue d'ensemble des dépôts `dnf` activés sur ce poste — elle
manquait. Objet immédiat : fermer le dossier Terra (marqueur ouvert
depuis le 2026-08-04, trois invites d'import de clé sur des commandes
réputées passives, la dernière dans `docs/desktop.md`). Consigné comme
**D10** dans `docs/machine-facts.md`.

**Écart avec le contexte de cette demande, signalé avant d'aller plus
loin** (`CLAUDE.md` § Avant d'agir) : la demande évoque « les douze
dépôts activés ». **Vérifié, ce poste en compte onze** :
```
$ dnf repo list --enabled
claude-code, copr:...group_ai-ml:nvidia-container-toolkit, fedora,
fedora-cisco-openh264, rpmfusion-free, rpmfusion-free-tainted,
rpmfusion-free-updates, rpmfusion-nonfree, rpmfusion-nonfree-updates,
terra, updates
```
`google-chrome` et le COPR `phracek/PyCharm` existent bien comme
fichiers `.repo` sur ce poste mais portent `enabled=0` — désactivés, pas
comptés (`grep ^enabled /etc/yum.repos.d/google-chrome.repo` →
`enabled=0`). Onze correspond au compte déjà établi au livrable BUR-0
(non revérifié à l'époque, revérifié ici). Cause de l'écart non établie
— pas creusée davantage, sans conséquence sur l'objet de ce livrable.

## 1. Corroboration de l'empreinte Terra

**Empreinte locale, base rpm** — trousseau temporaire isolé, jamais le
trousseau système (même méthode qu'au livrable CDI-1) :
```
$ rpm -q gpg-pubkey --qf '%{name}-%{version}-%{release} %{summary}\n' | grep -i terra
gpg-pubkey-ae09157a4de88b497ea1d5d300cdab43de226d6f-698a7357 Terra 44 <security@fyralabs.com> public key
$ rpm -q <ce-paquet> --qf '%{DESCRIPTION}\n' > /tmp/.../rpmdb-key.asc
$ gpg --homedir <tmp isolé> --no-default-keyring --import /tmp/.../rpmdb-key.asc
$ gpg --homedir <tmp isolé> --no-default-keyring --list-keys --with-fingerprint --with-colons
fpr:::::::::AE09157A4DE88B497EA1D5D300CDAB43DE226D6F:
```
**Empreinte du fichier déclaré par `terra.repo`**
(`gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-terra44`, déposé par le
paquet `terra-gpg-keys`) — même méthode, trousseau isolé distinct :
```
$ gpg --homedir <tmp isolé 2> --no-default-keyring --import /etc/pki/rpm-gpg/RPM-GPG-KEY-terra44
fpr:::::::::AE09157A4DE88B497EA1D5D300CDAB43DE226D6F:
```
**Les deux concordent, à l'identique** — le fichier système et
l'entrée de la base rpm portent la même clé. Ceci n'établit que la
**cohérence interne** de ce poste (le fichier n'a pas divergé de ce qui
a été importé) — pas l'identité du signataire.

**Recherche de corroboration indépendante — neuf canaux distincts du
serveur de dépôt (`repos.fyralabs.com`/`tetsudou.fyralabs.com`),
chacun nommé, y compris les échecs, pour qu'une série future ne les
rejoue pas sans le savoir :**

| # | Canal | Résultat |
|---|---|---|
| 1 | Recherche web de l'empreinte complète | Aucune occurrence trouvée |
| 2 | `docs.terrapkg.com/reference/faq/` | HTTP 404 |
| 3 | `developer.fyralabs.com/terra/faq` | Contenu vide/inaccessible (site rendu côté client, non extrait) |
| 4 | `developer.fyralabs.com/terra/installing` | Idem, y compris via `curl` brut (HTTP 200 mais aucun texte pertinent dans le HTML statique) |
| 5 | `wiki.fyralabs.com/Terra` | Page accessible, aucune empreinte mentionnée |
| 6 | `github.com/terrapkg/packages` — structure du dépôt | Aucun dossier `keys/`/`gpg/`, `SECURITY.md` inaccessible (404) |
| 7 | `github.com/terrapkg/packages` — `README.md` (branche par défaut `frawhide`, vérifiée via l'API GitHub) | Une seule mention GPG : la commande officielle d'installation elle-même (`--nogpgcheck --repofrompath`) — **aucune empreinte publiée, aucune mention de vérification indépendante** |
| 8 | Fil Fedora Discussion (« HowTo: Install Fyra Lab's Terra… ») | Décrit le téléchargement automatique de la clé, aucune empreinte à comparer, aucun commentaire de vérification |
| 9 | `fyralabs.com/.well-known/security.txt` + `fyralabs.com/pgp.asc` | Canal réellement indépendant du serveur de dépôt (site principal de l'organisation) — **mais porte une clé différente** : `00AEBB9B28D1D0E97D0713B390C30D0FE8C19CC3` (« Fyra Labs Security », contact sécurité organisationnel), **pas** la clé de signature du dépôt. Ne corrobore pas — et ne doit pas être confondu avec une alerte : deux usages distincts d'une même adresse `security@fyralabs.com`, propriété notée pour éviter une fausse alerte future. |

**Conclusion, formulée sans détour** : **aucune corroboration
indépendante trouvée.** À la différence du COPR (CDI-1), ce n'est pas
une impossibilité structurelle — Fyra Labs est une organisation tierce
identifiable, avec un site, un `security.txt`, une organisation
GitHub — mais un examen actif de ces canaux ne montre pas que
l'empreinte du dépôt Terra y soit publiée. C'est une lacune de
publication chez l'éditeur, pas une propriété du modèle.

**Décision de l'opérateur, consignée** : le risque est **déjà pris** —
cette clé gouverne l'installation des six paquets Terra présents sur ce
poste (dont `asusctl`, sur lequel reposent D2bis/D2ter) depuis le
2026-08-04, sans confirmation caduque depuis. Rendre le chemin non
privilégié cohérent avec le chemin déjà privilégié n'étend aucune
confiance nouvelle. Détail complet en D10, `docs/machine-facts.md`
§ Décisions.

**Vérification de cohérence demandée avant toute action, faite** : les
six paquets installés depuis `terra` portent tous la même signature.
`rpm -qi` affiche un champ « Signature » vide pour chacun (affichage
`rpm` ne rend pas les signatures EdDSA dans ce résumé — pas un défaut
de signature), mais la requête précise le confirme :
```
$ rpm -q asusctl asusctl-rog-gui cardwire supergfxctl terra-gpg-keys terra-release \
    --qf '%{SIGPGP:pgpsig}\n'
EdDSA/SHA256, ..., Key ID 00cdab43de226d6f   # (× 6, identique)
```
`00CDAB43DE226D6F` = le même identifiant court que l'empreinte
corroborée localement. Aucun paquet signé par une autre clé, aucun
paquet non signé — aucun fait nouveau, rien qui aurait dû arrêter la
suite.

## 2. Cause de la récurrence de l'invite — établie, pas supposée

**`terra.repo` active les deux vérifications, `gpgcheck` et
`repo_gpgcheck`** — seul dépôt de ce poste dans ce cas :
```
$ grep -E '^gpgcheck|^repo_gpgcheck' /etc/yum.repos.d/terra.repo
gpgcheck=1
repo_gpgcheck=1
```
(Comparaison avec les dix autres dépôts activés : tous ont
`repo_gpgcheck=0` — § 4. Terra est, sur ce point précis, **plus
strictement vérifié que les dépôts Fedora officiels eux-mêmes**.)

**Hypothèse d'un livrable précédent (jamais vérifiée) : un `dnf` non
privilégié utilise-t-il un cache et un trousseau distincts d'un `dnf`
privilégié ? Vérifiée ici, confirmée — avec le mécanisme exact.**

Établi par lecture locale, `man dnf5.conf`, section `repo_gpgcheck`
(source faisant autorité, externe) :
> « OpenPGP keys for this check are stored separately from OpenPGP
> keys used in package signature verification. Furthermore, they are
> also stored separately for each repository. […] This means that
> DNF5 may ask to import the same key multiple times. For example,
> when a key was already imported for package signature verification
> and this option is turned on, it may be needed to import it again
> for the repository. »

Confirmé empiriquement, chemin exact identifié :
```
$ sudo find /var/cache/libdnf5 -path "*terra*/pubring*" -exec ls -la {} \;
-rw-r--r--. 1 root root 514 Aug  4 14:45 /var/cache/libdnf5/terra-<hash>/pubring/00CDAB43DE226D6F.pub

$ find ~/.cache/libdnf5 -iname pubring
(vide)
```
**`libdnf5` maintient un trousseau de confiance (« pubring ») propre à
chaque dépôt, à l'intérieur du répertoire de cache — pas dans la base
rpm partagée, malgré ce que `gpgcheck` (lui, basé sur la base rpm
partagée) laisse penser.** Le cache **système** (`/var/cache/libdnf5/`,
racine) porte ce pubring, peuplé lors de la transaction 6 du
2026-08-04 (`dnf install asusctl`, interactive, acceptée par un humain
à ce moment). Le cache **utilisateur** (`~/.cache/libdnf5/`, arborescence
entièrement distincte) n'a **jamais** eu ce pubring peuplé — chaque
session de cette série étant non interactive (pas d'entrée standard),
l'invite s'est systématiquement résolue par un refus implicite (fin de
flux), jamais par une confirmation, jamais persistée.

**C'est la cause complète** : pas un trousseau système corrompu, pas
une clé absente — un trousseau *utilisateur* jamais rempli, pour un
mécanisme que `gpgcheck` (déjà résolu, base rpm partagée) ne laisse pas
deviner.

## 3. Correctif — le plus étroit possible, appliqué

**Voie retenue : copier le fichier de clé déjà accepté côté système
vers le pubring utilisateur — pas un nouvel import, aucune dégradation
de vérification.**

**Emplacements système touchés, nommés avant d'être touchés** :
- Lecture (aucune écriture) : `/var/cache/libdnf5/terra-7cd12bafb77e2892/pubring/00CDAB43DE226D6F.pub`
  — déjà présent, mode `0644`, lisible sans privilège.
- Écriture : `~/.cache/libdnf5/terra-7cd12bafb77e2892/pubring/00CDAB43DE226D6F.pub`
  — créé (le répertoire `pubring/` n'existait pas), dans l'arborescence
  de cache **personnelle** de l'utilisateur, jamais `/etc`, jamais
  `/var`, jamais la base rpm.

```
$ mkdir -p ~/.cache/libdnf5/terra-7cd12bafb77e2892/pubring
$ cp /var/cache/libdnf5/terra-7cd12bafb77e2892/pubring/00CDAB43DE226D6F.pub \
     ~/.cache/libdnf5/terra-7cd12bafb77e2892/pubring/00CDAB43DE226D6F.pub
$ sha256sum /var/cache/libdnf5/.../00CDAB43DE226D6F.pub ~/.cache/libdnf5/.../00CDAB43DE226D6F.pub
10043133db81d3c94f31fada37664e05077c18f41a8b86e818516275027f17b3  (les deux, identiques)
```
**Aucune commande `rpm --import`, aucun `dnf` avec confirmation
interactive, aucune modification de `terra.repo`** — une copie de
fichier, entre deux emplacements de cache, tous deux à l'intérieur du
compte de l'utilisateur (le second, le seul écrit, ne requiert aucun
privilège).

**Preuve que la friction a disparu** — la commande exacte qui
déclenchait systématiquement l'invite depuis le 2026-08-04, rejouée
**sans garde** :
```
$ dnf --repo=terra --refresh repoinfo terra
Updating and loading repositories:
 Terra 44   100% | ...
Repositories loaded.
Repo ID              : terra
[...]
Repodata info        : Available packages : 4676 ... Revision : 1786011211
```
Aucune invite, aucun avertissement `Signing key not found`. Rejoué
aussi avec les deux commandes historiquement fautives d'autres
livrables (`dnf list --available`, `dnf repoquery --file`) — même
résultat, propre.

**Preuve qu'aucune vérification n'a été dégradée** :
```
$ sha256sum /etc/yum.repos.d/terra.repo   # avant ET après le correctif
6a4a96963c4d1b33264c47deced3fa614f8b4fb0d7dc2b945077481112b63b78   # identique
$ grep -E '^gpgcheck|^repo_gpgcheck' /etc/yum.repos.d/terra.repo
gpgcheck=1
repo_gpgcheck=1   # les deux, inchangés
```

**Pourquoi une action ponctuelle, pas un rôle Ansible** : le fichier
touché est un **artefact de cache**, pas un fichier de configuration
stable — un `dnf clean all` ou une suppression de `~/.cache/libdnf5`
le ferait disparaître, et le nom du répertoire (`terra-7cd12bafb77e2892`)
est un hash dérivé de la configuration du dépôt, pas garanti stable
dans le temps ni entre versions de `libdnf5` (non vérifié au-delà de
cette machine, à cet instant). Un rôle Ansible qui viserait ce chemin
recréerait une dépendance à un détail d'implémentation interne de
`libdnf5`, plus fragile que la commande ponctuelle documentée ici — et
si le cache est un jour vidé, la même commande, rejouée manuellement,
résout la même situation en quelques secondes. Pas de rôle créé par
réflexe.

### 3.1 — Variante durable envisagée (BUR-1 point 0) : aucune trouvée

**Question posée** : `gpgkey=` pointant vers un fichier local plutôt
qu'une URL rendrait-il l'import `repo_gpgcheck` durable, ou existe-t-il
un autre mécanisme `dnf5` documenté pour ça ? Recherche en lecture
seule, sans présumer de nom d'option.

**`gpgkey=` de `terra.repo` pointe déjà vers un fichier local** —
vérifié directement, pas supposé :
```
$ cat /etc/yum.repos.d/terra.repo
[terra]
...
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-terra$releasever
repo_gpgcheck=1
...
```
Et pourtant l'invite s'est reproduite trois fois (§ ci-dessus, historique
depuis le 2026-08-04) : `gpgkey=file://...` gouverne l'import utilisé
par `gpgcheck` (vérification de signature de paquet, contre la base rpm
partagée, déjà stable — § 2), **pas** l'import utilisé par
`repo_gpgcheck` (vérification de signature des métadonnées de dépôt).
La note de `man dnf5.conf` déjà citée au § 2 le dit explicitement :
« *OpenPGP keys for this check [repo_gpgcheck] are stored separately
from OpenPGP keys used in package signature verification* » — un
`gpgkey=` local ne change donc rien au fait que `repo_gpgcheck`
importe et conserve sa propre clé, dans le pubring par-dépôt sous
`~/.cache/libdnf5/`, indépendamment de la source (URL ou fichier) de
`gpgkey=`. Vérifié par lecture directe du fichier de dépôt et par la
documentation amont — pas déduit par supposition.

**Recherche exhaustive d'un mécanisme `dnf5` alternatif** — aucune
option `dnf5.conf` ni sous-commande de gestion de clé trouvée en dehors
de celles déjà connues :
```
$ dnf5 --help | grep -iE 'key|trust|import|gpg'
  --no-gpgchecks         disable OpenPGP signature checking (if RPM policy allows)
  --nogpgcheck           Alias for '--no-gpgchecks'
$ dnf5 repo --help | grep -iE 'key|trust|import|gpg'
(rien)
$ man dnf5.conf   # options pkg_gpgcheck, localpkg_gpgcheck, repo_gpgcheck
# — les trois seules options liées à la vérification OpenPGP ; aucune
# ne porte de mécanisme de persistance distinct du cache par-dépôt.
```
`--no-gpgchecks`/`--nogpgcheck` désactive la vérification — c'est la
dégradation que ce point exclut explicitement, pas une variante
durable.

**Conclusion** : aucune variante durable et non dégradante n'existe
dans `dnf5` pour ce cas précis — confirmé, pas seulement non trouvé
faute de chercher. Le correctif § 3 (copie de cache à cache, aucun
import, aucune dégradation) reste la réponse correcte, et **restera à
rejouer** à chaque fois que le cache utilisateur perd cette clé. Les
trois conditions demandées par ce point sont déjà consignées au § 3 :
régression (`dnf clean all`, suppression de `~/.cache/libdnf5`,
changement de `terra.repo` — ex. bascule de version Fedora qui change
le nom de version dans l'URL/le hash de répertoire), commande de
restauration exacte (le bloc `mkdir -p`/`cp` § 3), symptôme
reconnaissable (invite d'import de clé GPG, ou message
`Signing key not found`/avertissement de signature, sur une commande
`dnf`/`dnf5` non privilégiée touchant `terra` — disparaît une fois la
copie refaite, § 3 « Preuve que la friction a disparu »).

## 4. Inventaire des onze dépôts activés

| Dépôt | Provenance | Ancrage de confiance réel | Fourni sur ce poste | `gpgcheck` | `repo_gpgcheck` |
|---|---|---|---|---|---|
| `fedora` | Fedora Project, miroir (`metalink`) | Clé Fedora 44 officielle, `/etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-44-x86_64`, déposée par `fedora-repos`/`fedora-gpg-keys` (chaîne de confiance de l'image d'installation) | 140 paquets | 1 | 0 |
| `updates` | Fedora Project, miroir | Même clé que `fedora` | 1015 paquets | 1 | 0 |
| `fedora-cisco-openh264` | Cisco (binaire OpenH264, hébergé par Fedora) | Même clé que `fedora` | 2 paquets | 1 | 0 |
| `rpmfusion-free` | RPM Fusion, miroir | Clé RPM Fusion Free, `/etc/pki/rpm-gpg/RPM-GPG-KEY-rpmfusion-free-fedora-44`, déposée par `rpmfusion-free-release` | 8 paquets | 1 | 0 |
| `rpmfusion-free-tainted` | RPM Fusion, miroir | Même clé que `rpmfusion-free` | 1 paquet | 1 | 0 |
| `rpmfusion-free-updates` | RPM Fusion, miroir | Même clé que `rpmfusion-free` | 8 paquets | 1 | 0 |
| `rpmfusion-nonfree` | RPM Fusion, miroir | Clé RPM Fusion Nonfree, `/etc/pki/rpm-gpg/RPM-GPG-KEY-rpmfusion-nonfree-fedora-44` | 0 paquet direct constaté | 1 | 0 |
| `rpmfusion-nonfree-updates` | RPM Fusion, miroir | Même clé que `rpmfusion-nonfree` | 10 paquets (dont pilote NVIDIA) | 1 | 0 |
| `terra` | Fyra Labs (infrastructure propre) | TLS + acceptation de clé le 2026-08-04, **corroboration indépendante introuvable** — D10 | 6 paquets, dont `asusctl` | 1 | **1** |
| `copr:…ai-ml:nvidia-container-toolkit` | COPR Fedora, groupe `@ai-ml` | TLS vers l'infrastructure COPR + empreinte épinglée — D7 | 2 paquets | 1 | 0 |
| `claude-code` | Anthropic, `downloads.claude.ai` | TLS + clé `https://downloads.claude.ai/keys/claude-code.asc` — D5 | 1 paquet | 1 | *(non défini, § note)* |

**Note sur `claude-code`** : `repo_gpgcheck` n'est pas défini dans son
fichier `.repo` — pas revérifié ici quel défaut `dnf5` lui applique en
l'absence de valeur explicite ; sans conséquence pour ce livrable, D5
ferme déjà ce dépôt.

**`6ecc2dfaa0dc41e5ad51e007707a786b` (1195 paquets)** : n'est **pas**
un dépôt actuellement configuré — c'est la valeur `from_repo` héritée de
l'installation initiale (image live/anaconda), conservée telle quelle
dans la base rpm pour les paquets installés à ce moment (confirmé par
échantillon : `Box2D`, `ModemManager`, `NetworkManager-libreswan` —
paquets de base Fedora sans rapport avec un dépôt tiers). Mécanisme
exact non creusé davantage — sans rapport avec l'objet de ce livrable.
`@commandline` (3 paquets) : paquets installés par chemin de fichier
direct (`dnf install <chemin.rpm>`), pas par un dépôt.

## 5. Équivalents Fedora/COPR pour les paquets `terra` — fait consigné, rien entrepris

**Recherche menée, résultat seul, aucune proposition** :
```
$ dnf --disablerepo=terra repoquery --available asusctl asusctl-rog-gui cardwire supergfxctl
(aucun résultat — ni Fedora, ni RPM Fusion, ni les COPR déjà activés)
```
Recherche élargie à l'ensemble des projets COPR (API publique COPR,
recherche par nom, lecture seule) :
- `asusctl` : plusieurs COPR communautaires non officiels trouvés
  (`bahram-f73/asusctl`, `yuikotegawa/asus-linux`,
  `valokardin/asus-linux`, entre autres) — non vérifiés, non comparés,
  non retenus.
- `supergfxctl` : même constat, projets liés à `asus-linux` trouvés
  dans plusieurs COPR communautaires.
- `cardwire` : aucun résultat dans la recherche COPR.

**Aucune action entreprise sur cette base** — information consignée
pour une décision future, comme demandé.

## 6. Registres de conteneurs — surface distincte de `dnf`, IA-2

Les onze dépôts ci-dessus (§ 4) sont tous des dépôts `dnf`. **Une
surface d'approvisionnement d'une autre nature existe déjà sur ce
poste, jamais nommée ici avant IA-2** (`docs/local-ai.md`) : les
registres de conteneurs OCI, interrogés par `podman pull`/`skopeo`, pas
par `dnf`. Nommée ici « au même titre que Terra et le COPR », comme
demandé, avec le même traitement (ancrage de confiance explicite).

| Registre | Provenance | Ancrage de confiance réel | Utilisé pour |
|---|---|---|---|
| Docker Hub (`docker.io`) | Docker, Inc. — infrastructure de registre publique | TLS vers `registry-1.docker.io` + **empreinte d'image épinglée** (`sha256:b88c73...`, `skopeo inspect`, jamais une étiquette mobile comme `:latest`) — `roles/local_ai/defaults/main.yml` | `docker.io/ollama/ollama` (D14) |

**Différence avec Terra/COPR (§ 1, § 4)** : l'ancrage retenu ici n'est
pas une clé de signature de paquet mais une **empreinte de contenu**
(le modèle de confiance natif d'un registre OCI — un tag pointe vers un
digest, le digest identifie le contenu de façon univoque). Pas de
`gpgcheck`/`repo_gpgcheck` équivalent côté registre de conteneurs — la
vérification d'intégrité TLS + digest épinglé au moment du tirage joue
un rôle analogue, documentée dans `docs/local-ai.md` § 7.2 (comment
l'empreinte a été établie et vérifiée avant écriture).

**Récupération de modèles, distincte de l'image du serveur** : le
registre de *modèles* Ollama (adressé par `ollama pull` à l'intérieur
du conteneur, mécanisme distinct du registre OCI ci-dessus bien que du
même écosystème) est une **troisième** surface, nommée mais non
utilisée par ce dépôt — aucun modèle téléchargé (`docs/local-ai.md`
§ 8.5, IA-2 §1) ; le confinement réseau appliqué en IA-2 la ferme
délibérément par défaut, un chemin distinct resterait à établir pour
un livrable qui télécharge réellement un modèle.

## 7. `crates.io` — quatrième surface d'approvisionnement, CMP-1

Nommée « au même niveau d'exigence que Terra, le COPR et les
registres de conteneurs », comme demandé (D19, `docs/machine-facts.md`
§ Décisions) — quatrième surface distincte de ce dépôt après les
extensions d'éditeur (D12/D13 les ferment plutôt que les ouvrir, mais
la catégorie existe), les registres de conteneurs (§ 6) et le registre
de modèles Ollama (§ 6, non utilisé). Ouverte par `roles/completion/`
pour compiler [`lsp-ai`](https://github.com/SilasMarvin/lsp-ai) depuis
les sources — jamais son binaire précompilé (motif complet,
`docs/completion.md` § 2.1, D19).

| Ce qui est récupéré | Provenance | Ancrage de confiance réel | Écart résiduel assumé |
|---|---|---|---|
| Code source de `lsp-ai`, à l'empreinte de commit `1e910a8cf0048406eb227bf2064743010a9ff3a9` | Dépôt git GitHub, `github.com/SilasMarvin/lsp-ai` (clone HTTPS) | TLS vers GitHub + **empreinte de commit épinglée** (jamais une étiquette — une étiquette peut être déplacée, un commit non) | Aucune signature cryptographique de commit (pas de commit signé GPG constaté) — l'ancrage repose sur TLS + immuabilité du graphe de commits Git, pas sur une clé de signataire identifié (contrairement à Terra/COPR, § 1/§ 4) |
| Dépendances transitives (crates.io, ~260 paquets verrouillés) | Registre public `crates.io` (Rust Foundation) | **Empreinte de contenu par paquet**, portée par `Cargo.lock` (`checksum = "..."`, SHA-256 par version publiée) — vérifiée par `cargo build --locked`, qui échoue plutôt que d'accepter un contenu ne correspondant pas à l'empreinte enregistrée | Aucun audit de code indépendant des ~260 paquets — l'ancrage garantit l'intégrité (le contenu tiré correspond à ce que `Cargo.lock` enregistre), pas l'absence de code malveillant dans une dépendance elle-même compromise en amont au moment de la publication |
| Une dépendance git de `lsp-ai`, `hf-hub` (`github.com/huggingface/hf-hub`) | Dépôt git GitHub, tiré transitivement | Empreinte de commit également (`6303587576f8a1ce9f91f8274265a153b89afb6e`) — mais **écart concret rencontré** : cette dépendance est déclarée par simple exigence de version dans `Cargo.toml`, pas par `rev` fixe ; le dépôt amont a retiré les étiquettes correspondantes, cassant la résolution malgré l'empreinte déjà verrouillée (`docs/completion.md` § 7.2.1) | Un dépôt git tiers peut retirer les références (étiquettes, branches) qu'un projet dépendant présumait stables, même quand une empreinte de commit précise reste, elle, immuable et récupérable — écart structurel de l'écosystème, pas propre à `lsp-ai` |

**Différence avec les registres de conteneurs (§ 6)** : `crates.io`
distribue du **code source** compilé localement (pas une image
préconstruite) — l'auditabilité est plus directe (le code compilé est
lisible avant compilation, contrairement aux couches binaires d'une
image de conteneur), au prix d'un temps de compilation et d'une
surface de dépendances transitives bien plus large à faire confiance
(~260 paquets contre une poignée de couches d'image).

**Ce que ce dépôt ne fait jamais avec cette surface** : aucun binaire
précompilé de `lsp-ai` téléchargé (D19, explicite) ; aucune
dépendance ajoutée au-delà de ce que `lsp-ai` déclare lui-même dans
son `Cargo.lock` amont (`--locked` l'impose) ; aucun paquet `cargo`
installé globalement au-delà de la chaîne de compilation elle-même
(`rust`/`cargo`, `fedora`/`updates`).

## 8. Registre de modèles Ollama — cinquième surface, IA-3/D21

Nommé « au même niveau d'exigence que Terra, le COPR et `crates.io` »,
comme demandé (D21, `docs/machine-facts.md` § Décisions) — déjà
identifié mais jamais utilisé en IA-2 (§ 6 ci-dessus, « troisième
surface, nommée mais non utilisée »). Utilisé pour la première fois
par ce livrable, via le conteneur de récupération éphémère
(`docs/local-ai.md` § 9) — jamais le conteneur de service.

**Ancrage de confiance réel** : TLS vers `registry.ollama.ai` +
**empreintes de contenu par manifeste et par blob**, format registre de
distribution OCI/Docker standard (`schemaVersion: 2`,
`mediaType: application/vnd.docker.distribution.manifest.v2+json`) —
même famille de modèle de confiance que les registres de conteneurs
(§ 6), pas une clé de signataire.

**Mécanisme d'intégrité, vérifié par l'outil lui-même, pas supposé** :
```
$ podman exec ollama-recovery ollama pull mistral-nemo:12b-instruct-2407-q4_K_M
pulling dd3af152229f: 100% ▕████████████████▏ 7.5 GB
pulling 438402ddac75: 100% ▕████████████████▏  683 B
pulling 43070e2d4e53: 100% ▕████████████████▏  11 KB
[...]
verifying sha256 digest
writing manifest
success
```
Chaque composant du modèle (poids, gabarit, licence, paramètres) est
un blob adressé par son propre condensé SHA-256 — vérifié directement,
pas déduit du nom de fichier :
```
$ sha256sum models/blobs/sha256-60e05f21...
60e05f2100071479f596b964f89f510f057ce397ea22f2833a0cfe029bfc2463  [...]
```
Le nom du fichier **est** son empreinte de contenu ; `ollama pull`
annonce explicitement une étape « verifying sha256 digest » sur ce
qu'il vient de recevoir.

**Écart résiduel assumé, découvert par un test réel, pas supposé** :
un blob local corrompu **après** téléchargement, dont le nom de
fichier reste inchangé, **n'est pas détecté ni réparé** par un
`ollama pull` ultérieur — l'outil vérifie le contenu **reçu sur le
réseau**, pas le contenu **déjà présent localement sous le nom
attendu**. Démontré : un octet modifié dans le blob de licence d'un
modèle déjà tiré, puis `ollama pull` rejoué sur ce même modèle — la
commande annonce `100%`/`success`, mais le blob corrompu reste
corrompu (empreinte inchangée, incorrecte) après coup. Seule la
**suppression** du fichier avant un nouveau `pull` force un
retéléchargement réel et corrige l'empreinte. **Conséquence pratique**
nommée pour un livrable futur qui s'en soucierait : ce registre protège
contre une corruption **pendant le transfert**, pas contre une
corruption **après écriture** sur le disque local (bit rot,
modification accidentelle) — aucun mécanisme de vérification
périodique de l'intégrité déjà présente sur ce poste (contrairement à
`rpm -Vf` pour les fichiers d'un paquet, § ailleurs dans ce dépôt).

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — D10, D7 (COPR), D5
  (Claude Code), inventaire GPU et Ansible.
- [`docs/gpu-containers.md`](gpu-containers.md) — D7, ancrage de
  confiance du COPR `@ai-ml`, même discipline de sourcing appliquée ici
  à Terra.
- [`docs/local-ai.md`](local-ai.md) — D14, empreinte épinglée de
  `docker.io/ollama/ollama`, confinement réseau (IA-2), conteneur de
  récupération éphémère et mesures d'enveloppe réelle (D21, IA-3).
- [`docs/completion.md`](completion.md) — D19, `crates.io` et
  l'empreinte de commit `lsp-ai` en usage réel (CMP-1).
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing, garde `--assumeno`
  sur les commandes touchant un dépôt à `gpgcheck` incertain.

## Validation

**Actions privilégiées, exhaustives — trois lectures, aucune
écriture** :

| # | Commande | Chemin cible | Motif |
|---|---|---|---|
| 1 | `sudo dnf --repo=terra --assumeno list --available` | lecture réseau/cache, aucune écriture | comparer le comportement privilégié/non privilégié (§ 2) |
| 2 | `sudo dnf --repo=terra --refresh repoinfo terra` | lecture réseau/cache | reproduire la commande décisive côté privilégié, gardée |
| 3 | `sudo find /var/cache/libdnf5 -path "*terra*/pubring*" -exec ls -la {} \;` | lecture répertoire système | confirmer l'existence et le contenu du pubring système |

**Aucune écriture privilégiée dans ce livrable.** Le correctif
lui-même (§ 3) n'a nécessité aucun privilège — lecture d'un fichier
`0644` déjà lisible, écriture dans le cache personnel de l'utilisateur.

**Commandes non modifiantes, hors privilège** : lectures de
`/etc/yum.repos.d/*.repo`, `/etc/pki/rpm-gpg/RPM-GPG-KEY-terra44`,
`rpm -q`/`rpm -qi`, `dnf repo list`, `dnf repoquery` (dont deux
invocations sans garde sur Terra, assumées comme reproduction de
l'incident et sa résolution, pas un nouvel incident — voir § 3) ; `man
dnf5.conf` ; recherches web et lectures de pages publiques (9 canaux,
§ 1) ; requêtes à l'API publique COPR (§ 5).

Ce document ne porte aucun marqueur de vérification actionnable —
aucun point n'est resté sans résolution ; la corroboration a été
activement recherchée et son absence établie comme fait, pas laissée
en suspens.

**Confirmations finales** : `terra` toujours activé
(`dnf repo list --enabled` inclut `terra`) ; aucun paquet désinstallé
ni remplacé (`rpm -q asusctl asusctl-rog-gui cardwire supergfxctl
terra-gpg-keys terra-release` — les six toujours présents, mêmes
versions) ; aucun nouveau dépôt (toujours onze activés) ; aucune
modification de `sudoers` ni de `/etc/cdi/` ; aucun redémarrage ;
`gpgcheck`/`repo_gpgcheck` inchangés sur `terra.repo` (§ 3).
