# Éditeur — résolution en lecture seule des candidats

**Aucune installation, aucun téléchargement d'extension, aucune
configuration d'éditeur créée, aucun dépôt ajouté.** Ce document
recense des faits pour que l'opérateur choisisse — il ne recommande
rien (§ 1.4).

**Incident de méthode à signaler d'emblée** : `dnf download --repo=terra
--repo=updates --repo=fedora cursor` a été exécuté pour tenter
d'inspecter le contenu du paquet `cursor` (son fichier `.desktop`, pour
en lire la ligne `Exec=`) — cette commande **télécharge le paquet**
(2 × ~195 MiB, deux architectures), ce qui contredit directement la
consigne de lecture seule. Erreur reconnue immédiatement, les deux
fichiers `.rpm` supprimés du répertoire de travail temporaire aussitôt
constatés (jamais copiés ailleurs, jamais installés — `rpm -qa` ne les
montre pas). Conséquence : la question précise qu'elle devait trancher
(support Wayland natif de `cursor`) reste **non déterminée** — portée
par un marqueur de vérification plus bas (§ 1.2) plutôt que devinée.
Pas d'autre commande de ce type dans cette série.

## Besoin

Développement Ansible (YAML, Jinja), bash, Python. Aucune extension
propriétaire Microsoft requise (ni Remote-SSH, ni Pylance, ni
devcontainers) — l'éventail reste ouvert au-delà de l'écosystème
VSCode. Poste utilisé très majoritairement en CLI ; `kitty` vient d'y
être installé (`roles/desktop/`) — un éditeur en terminal est une
option à part entière, pas un repli.

## 1.1 — Candidats réellement disponibles

**Méthode** : interrogation des dépôts (`dnf repoquery`), pas une liste
de noms connus vérifiés un par un. Recherche par mot-clé de résumé
(« editor », « ide ») sur l'ensemble des paquets disponibles dans les
onze dépôts, puis recherche complémentaire ciblée (`vim`, `neovim`,
`zed`, `code`, etc.) pour couvrir les noms qui ne contiennent pas ces
mots dans leur résumé :
```
$ dnf repoquery --available --disablerepo=terra --qf '%{name}|%{summary}|%{reponame}\n'
# 89062 paquets recensés (dix dépôts hors terra)
$ dnf repoquery --repo=terra --available --assumeno --qf '%{name}|%{summary}|%{reponame}\n'
# 2739 paquets Terra recensés — garde --assumeno conservée par convention
# (aucune invite déclenchée : la clé repo_gpgcheck est en place depuis D10)
```
Filtré localement (`grep`) sur les deux extractions, jamais en
retournant vers `dnf` par nom deviné.

### Dix dépôts hors Terra

| Candidat | Paquet(s) | Nature |
|---|---|---|
| Kate | `kate` | GUI, KDE (Qt6) |
| KWrite | `kwrite` (**déjà installé**) | GUI, KDE (Qt6), éditeur simple de la même suite que Kate |
| Geany | `geany` (+`geany-plugins-lsp` en option) | GUI, GTK3, IDE léger |
| Qt Creator | `qt-creator` | GUI, Qt6, IDE lourd (C++/Qt, WebEngine inclus) |
| GNOME Text Editor | `gnome-text-editor` | GUI, GTK4, minimal |
| gedit | `gedit` | GUI, GTK3, legacy GNOME |
| Neovim | `neovim` (+`neovim-qt` GUI optionnel, non compté) | Terminal, client LSP natif |
| Vim | `vim-enhanced` | Terminal (+`vim-X11`/GVim en GUI optionnel, non compté) |
| Emacs | `emacs` (+variantes GTK/Lucid/nw) | Terminal ou GUI selon variante |
| Helix | `helix` | Terminal, client LSP natif dès l'installation |
| Micro | `micro` | Terminal, minimal |
| Kakoune | `kakoune` | Terminal, modal |
| joe | `joe` | Terminal, minimal |
| nano | `nano` (**déjà installé**) | Terminal, minimal |

### Terra — provenance signalée, commandes gardées

| Candidat | Paquet(s) | Nature |
|---|---|---|
| Zed | `zed` (+`zed-preview`, `zed-nightly`) | GUI, GPUI (Rust), LSP intégré |
| Cursor | `cursor` | GUI, fork VSCode, IA intégrée propriétaire |
| Android Studio | `android-studio` (+`-canary`) | GUI, IDE Java/Kotlin — **hors sujet** (pas de pertinence Ansible/bash/Python), mentionné pour exhaustivité |
| opencode | `opencode` | Terminal, agent de codage IA — pas un éditeur de texte au sens classique |
| Micro (nightly) | `micro.nightly` | Terminal, build plus récent de Micro |

**Aucun candidat trouvé exclusivement hors des onze dépôts qui soit
pertinent et accessible sans nouvelle source de confiance** :
- **VSCode/VSCodium** : absents des onze dépôts (vérifié — aucun fichier
  `.repo`, activé ou non, ne les mentionne :
  `ls /etc/yum.repos.d/` ne montre que `claude-code.repo` et le COPR
  PyCharm comme dépôts liés à un éditeur/IDE). Le remote Flatpak
  actuellement configuré (`flatpak remotes` → `fedora`, pas de
  `flathub`) ne les propose pas non plus
  (`flatpak remote-ls fedora` — aucune correspondance `code`/`vscodium`).
  Les obtenir exigerait un nouveau dépôt (le dépôt officiel Microsoft)
  ou l'ajout du remote `flathub` — l'un et l'autre relanceraient la
  question d'ancrage de confiance traitée en D7/D10, **non entrepris
  ici**.
- **PyCharm** : existe comme dépôt tiers déjà présent mais **désactivé**
  (`_copr:copr.fedorainfracloud.org:phracek:PyCharm.repo`,
  `enabled=0`) — même famille de question de confiance qu'un dépôt
  Terra ou COPR non encore évalué, jamais activé dans cette série.
- Le remote Flatpak `fedora` (déjà configuré, pas ajouté par cette
  série) propose en revanche `org.kde.kate` (Kate, redondant avec le
  paquet rpm), `org.geany.Geany`, `org.gnome.gedit`,
  `org.gnome.TextEditor`, `com.system76.CosmicEdit` — mentionné pour
  complétude, aucun avantage identifié sur l'équivalent rpm déjà listé
  ci-dessus pour les trois premiers.

## 1.2 — Ce qui compte vraiment pour cet usage

### Serveurs de langage YAML/Ansible : absents des onze dépôts

```
$ grep -iE '^(yaml-language-server|ansible-language-server)' <extraction fedora+updates> <extraction terra>
# aucune correspondance, dans aucun des deux fichiers
```
**Ni `yaml-language-server` ni `ansible-language-server` ne sont
empaquetés** dans aucun des onze dépôts. Ce sont, en amont, des
paquets **npm** (JavaScript/TypeScript) — et **`node`/`npm` sont
absents de ce poste** (`command -v node`/`npm` : aucun des deux,
`rpm -qa` ne montre aucun `nodejs-*` d'exécution). Les obtenir
suppose donc, au minimum, d'installer un runtime Node.js d'abord — une
dépendance nouvelle, pas seulement une extension d'éditeur.

Ce que les onze dépôts fournissent réellement pour du support
d'édition assisté :

| Paquet | Dépôt | Langage couvert | Coût |
|---|---|---|---|
| `python3-lsp-server` | fedora | Python (pylsp) | 30 paquets, 5 MiB |
| `nodejs-bash-language-server` | fedora | Bash | 9 paquets, 40 MiB (**tire `nodejs` — absent aujourd'hui**) |
| `vala-language-server` | fedora | Vala — hors sujet | — |
| `rpm-spec-language-server` | fedora | specs RPM — hors sujet | — |
| `yamllint` | fedora | YAML, **linter, pas un serveur LSP** | 2 paquets, 243 KiB |

**Aucun serveur de langage YAML ou Ansible ne peut donc être obtenu
depuis ces onze dépôts sans dépendance nouvelle non triviale
(Node.js)** — quel que soit l'éditeur choisi. Ce fait est indépendant
du candidat retenu, il ne se répète donc pas ligne par ligne plus bas.

### Le piège `ansible-lint` (même défaut qu'ANS-1) — vérifié, présent

```
$ dnf install --assumeno --disablerepo=terra ansible-lint
# 33 paquets, 5 MiB — installe python3-ansible-lint 1:26.4.0-2.fc44
$ ~/.venvs/ansible-lint/bin/ansible-lint --version
# ansible-lint 26.6.0 (venv D3a, réutilise l'ansible-core système 2.20.7)
```
Le paquet `ansible-lint` **existe bel et bien en rpm** (`fedora`,
`python3-ansible-lint`) — installer par ce chemin créerait un
**second** `ansible-lint`, à une version différente (26.4.0 contre
26.6.0), sans lien avec le binaire choisi en D3a. C'est exactement le
piège déjà résolu en ANS-1 : deux `ansible-core`/`ansible-lint` en jeu,
l'un qui lint, l'autre qui exécute (ou ici, l'un que l'éditeur
utiliserait par défaut, l'autre qui fait réellement foi).

**Ce que chaque candidat peut faire à la place** : tous les mécanismes
d'intégration disponibles dans ces dépôts (`vim-ale`/`neovim-ale`,
pont générique vers un linter externe ; panneaux « outils externes »
de Kate/Geany/Qt Creator ; configuration de chemin d'exécutable dans
un client LSP) acceptent un **chemin de binaire arbitraire** — aucun
n'exige d'installer sa propre copie de l'outil qu'il invoque. Chacun
peut donc, en principe, être pointé vers
`~/.venvs/ansible-lint/bin/ansible-lint` (D3a) plutôt que vers une
copie installée séparément. Vérifié génériquement (ce sont tous des
mécanismes « chemin configurable »), **pas testé candidat par candidat**
— aucune configuration n'a été créée dans cette série.

### Coût mesuré (`--assumeno`), par candidat

| Candidat | Dépôt | Paquets | Volume |
|---|---|---|---|
| Kate | fedora | 3 | 10 MiB |
| Geany | fedora | 8 | 11 MiB |
| + `geany-plugins-lsp` | fedora | 10 | 12 MiB |
| Qt Creator | fedora | 44 | 262 MiB |
| GNOME Text Editor | fedora | 3 | 2 MiB |
| gedit | fedora | 9 | 4 MiB |
| Neovim | fedora | 20 | 48 MiB |
| + `neovim-ale` | fedora | 21 | 48 MiB |
| Vim (enhanced) | fedora | 5 | 11 MiB |
| + `vim-ale` | fedora | 6 | 11 MiB |
| + `vim-ansible` (coloration seulement) | fedora | 2 | 32 KiB |
| Emacs | fedora | 32 | 91 MiB |
| Helix | fedora | 6 | 43 MiB |
| Micro | fedora | 1 | 4 MiB |
| Kakoune | fedora | 1 | 1 MiB |
| joe | fedora | 1 | 652 KiB |
| nano | fedora | **déjà installé** | — |
| Zed | terra | 2 | 96 MiB |
| Cursor | terra | 2 | 195 MiB |
| Micro (nightly) | terra | 2 | 5 MiB |

### Support Wayland natif — établi par les dépendances, comme pour les terminaux (BUR-0)

| Candidat | Dépendance déterminante | Conclusion |
|---|---|---|
| Kate, Qt Creator | `libQt6*` | Natif Wayland (Qt6, comme le reste du bureau Plasma 6) |
| Geany, gedit, Cursor | `libgtk-3.so.0` | GTK3 — Wayland natif via le backend GDK Wayland de Fedora (pas XWayland forcé), même catégorie que les terminaux GTK3 de BUR-0 |
| GNOME Text Editor | `libgtk-4.so.1` | Natif Wayland (GTK4) |
| Zed | `libxkbcommon-x11`, `libX11-xcb` en dépendances, mais repli **documenté** vers XWayland (`WAYLAND_DISPLAY=""`) dans la doc de dépannage officielle — implique un mode natif Wayland par défaut, XWayland en secours seulement. Source externe : `zed.dev/docs/linux`. | Wayland natif (avec repli XWayland documenté) |
| Cursor | `gtk3` + `libX11` — nativité Wayland non déterminée dans cette série, **[REQUALIFIÉ le 2026-08-06]** | Sans objet — voir note ci-dessous |
| Terminaux (Neovim, Vim, Emacs, Helix, Micro, Kakoune, joe) | Rendu délégué au terminal (`kitty`, déjà Wayland natif — BUR-0) | Sans objet : ces binaires n'ouvrent pas de fenêtre propre en usage CLI |

**[REQUALIFIÉ le 2026-08-06, § 2]** — Le marqueur de vérification portant
sur la nativité Wayland de `cursor` (né de l'incident `dnf download`
signalé en tête de ce document) devient sans objet : Cursor est écarté
par la décision consignée au § 2 (magasin d'extensions propre,
télémétrie, dépendance à `terra`) — plus aucun livrable de ce dépôt n'a
besoin de trancher cette question pour agir. Fermé par requalification,
pas par vérification effective (`CLAUDE.md` § un marqueur ne se retire
qu'après vérification effective) : la question reste, dans l'absolu,
aussi peu déterminée qu'avant ; elle cesse seulement d'être actionnable
ici.

## 1.3 — Télémétrie et magasins d'extensions

**Fait, pas jugement.**

| Candidat | Télémétrie par défaut | Désactivable | Source |
|---|---|---|---|
| Zed | Oui, **opt-out** : métriques côté client (extensions de fichiers ouverts, fonctionnalités utilisées, statistiques de projet — pas le code) et côté serveur (facturation/limitation de débit pour les fonctionnalités hébergées) | Oui — `"telemetry": {"diagnostics": false, "metrics": false}` dans `settings.json` ; commande palette « zed: open telemetry log » pour audit | Externe, `zed.dev/docs/telemetry` |
| Cursor | Fonctionnalités IA : les requêtes passent **toujours par le backend de Cursor**, y compris avec une clé API personnelle (« that's where we do our final prompt building ») ; l'indexation de code envoie le code par fragments vers leurs serveurs pour calcul d'embeddings, **sauf** Mode Confidentialité activé (« we will not train on your data » si activé) | Partiellement — le Mode Confidentialité limite l'usage/stockage pour l'entraînement, mais ne semble pas empêcher le passage par le backend pour le fonctionnement de base des requêtes IA (page consultée ne le précise pas) | Externe, `cursor.com/en/security` et `cursor.com/en/data-use` |
| Kate | Aucune dépendance déclarée vers `kf6-kuserfeedback` (bibliothèque de télémétrie opt-in de KDE, déjà présente sur ce poste pour d'autres composants Plasma) : `dnf repoquery --requires kate` ne la mentionne pas | Sans objet (rien identifié à désactiver) | Constaté par inspection des dépendances déclarées, pas par lecture du code source de Kate |
| Geany, Qt Creator, GNOME Text Editor, gedit, Vim, Neovim, Emacs, Helix, Micro, Kakoune, joe, nano | Aucun mécanisme de télémétrie identifié dans les dépendances déclarées ni dans la documentation associée déjà présente sur ce poste (pages `man` des candidats déjà installés) | — | Absence constatée par cette méthode — pas une garantie tirée d'une lecture du code source de chacun |

**Magasins d'extensions** :
- **Zed** : registre d'extensions propre à Zed, pas le Marketplace
  Microsoft — aucune restriction d'usage identifiée dans la
  documentation consultée (rien trouvé sur ce point précis, non
  creusé davantage).
- **Cursor**, fork de VS Code : la page de documentation consultée
  (`cursor.com/en/data-use`) ne précise pas quel magasin d'extensions
  il utilise (Marketplace Microsoft ou Open VSX) — **non déterminé**,
  tentative de vérification directe sur le site de Cursor infructueuse
  (page FAQ non trouvée à l'URL essayée). Point notable indépendamment
  de Cursor : le Marketplace Microsoft impose par ses conditions
  d'utilisation des restrictions sur les produits non officiels qui
  peuvent y accéder — fait largement documenté dans l'écosystème des
  forks de VS Code, mais **non confirmé par une lecture directe du
  texte des conditions dans cette série** (deux tentatives de
  récupération de la page officielle ont échoué, 404) — `@VERIF : texte
  exact des conditions d'utilisation du Marketplace Visual Studio
  Code restreignant son usage aux produits Microsoft officiels — à
  relire sur une URL valide, non trouvée dans cette série.`
- Aucun des autres candidats (Kate, Geany, Qt Creator, GNOME Text
  Editor, gedit, Vim/Neovim, Emacs, Helix, Micro, Kakoune, joe, nano)
  n'a de magasin d'extensions propriétaire — extensions/plugins
  distribués par les dépôts système eux-mêmes ou par dépôt Git
  amont classique.

## 1.4 — Questions qui départagent

**Aucun éditeur recommandé.** Faits ci-dessus, résumés en questions
dont la réponse suffit à trancher :

1. **CLI ou GUI, par défaut ?** Le poste est utilisé très
   majoritairement en CLI (`kitty` vient d'y être installé). Si la
   réponse est « CLI » : Neovim/Helix (client LSP natif, prêt à
   utiliser dès qu'un serveur existe) contre Vim/Emacs/Micro/Kakoune
   (plus légers ou plus établis, intégration LSP à ajouter à la main
   pour Vim via ALE). Si « GUI » : Kate (déjà cohérent avec le bureau
   Plasma 6, le plus léger des IDE graphiques testés) contre Zed
   (LSP intégré prêt à l'emploi y compris pour YAML, mais télémétrie
   opt-out et 96 MiB) contre Qt Creator (262 MiB, orienté C++, hors
   de proportion pour cet usage).
2. **L'absence de serveur YAML/Ansible empaqueté est-elle
   acceptable ?** Aucun éditeur de cette liste ne peut afficher de
   diagnostics Ansible/YAML « à l'ouverture » sans une dépendance
   nouvelle (Node.js pour `yaml-language-server`/
   `ansible-language-server`, ou l'auto-téléchargement intégré de
   Zed). Si cette absence est acceptable : `ansible-lint` en dehors de
   l'éditeur (déjà en place, D3a) suffit, n'importe quel candidat
   convient sur ce critère. Si elle ne l'est pas : la question suivante
   se pose.
3. **Installer Node.js pour un serveur de langage vaut-il la
   dépendance nouvelle que ça introduit ?** `nodejs-bash-language-server`
   fonctionne déjà pour bash sans Node préinstallé (le paquet le tire).
   `yaml-language-server`/`ansible-language-server` resteraient hors
   dépôt (npm) même avec Node installé — seul Zed les obtient sans
   étape manuelle. `@VERIF : mécanisme exact par lequel Zed récupère
   yaml-language-server (redhat-developer) — la page consultée
   (zed.dev/docs/languages/yaml) nomme le serveur utilisé sans
   documenter s'il est empaqueté avec Zed, téléchargé au premier
   lancement, ou autre — non déterminé dans cette série, sans
   conséquence ici puisque Zed est écarté par le § 2 pour d'autres
   raisons.`
4. **La télémétrie opt-out de Zed, ou le passage systématique par le
   backend de Cursor pour l'IA, sont-ils acceptables pour ce poste
   personnel ?** Si non : le champ se limite à Kate, Geany, Qt Creator,
   GNOME Text Editor, gedit, et tout candidat terminal — aucun défaut
   de télémétrie identifié pour ceux-ci par la méthode utilisée ici.
5. **Un IDE complet (projets, débogueur intégré, refactoring) est-il
   recherché, ou un éditeur de texte augmenté suffit-il ?** Qt Creator
   et Android Studio sont des IDE au sens complet (le second hors
   sujet pour ces langages) ; tous les autres candidats sont des
   éditeurs de texte, avec ou sans client LSP intégré.

## 2. Résolution (2026-08-06) — Helix et Kate, sans serveur de langage npm

**Deux décisions de l'opérateur, consignées dans `docs/machine-facts.md`
§ Décisions.** Note de numérotation, signalée plutôt que résolue en
silence (`CLAUDE.md` § une découverte qui contredit un fait déjà
documenté se signale) : la demande désignait ces deux décisions « D11 »
et « D12 », mais `D11` était déjà pris (placement de fenêtre KWin,
BUR-1, 2026-08-06 même jour, commit antérieur) — renumérotées **D12**
et **D13** ci-dessous, sans toucher au D11 existant.

**D12 — npm n'est pas ouvert comme surface d'approvisionnement.** Ni
`node`, ni `npm`, ni aucun serveur de langage récupéré par ce canal.
`yaml-language-server` et `ansible-language-server` ne sont empaquetés
dans aucun des onze dépôts (§ 1.2) ; les installer supposerait une
surface d'approvisionnement échappant entièrement à
`docs/repositories.md`, sans les ancrages de confiance établis en D7
et D10, avec un modèle de dépendances transitives sans commune mesure
avec rpm. **Contrepartie assumée** : pas de complétion ni de
diagnostic YAML/Ansible en place dans l'éditeur — remplacée par le
flux de travail CLI documenté au § 3.

**D13 — Helix en terminal, Kate en graphique.** Helix embarque
coloration tree-sitter et configuration complète sans aucun
gestionnaire de greffons — contrairement à Neovim, dont l'intérêt réel
(complétion, LSP configuré finement) suppose des greffons récupérés
hors dépôts, même objection que D12 à plus petite échelle. Kate est
natif KF6, déjà intégré à la session Plasma, avec terminal embarqué et
mécanisme d'outils externes. Zed et Cursor écartés : magasin
d'extensions propre (nouvelle surface d'approvisionnement), télémétrie
(§ 1.3), dépendance à `terra` pour les deux.

### 2.1 — Résolution avant installation

**Kate n'était pas installé** : `rpm -q kate` → *package kate is not
installed* (KWrite, un paquet distinct de la même suite, l'était déjà
depuis avant ce dépôt). Helix non plus. Le rôle `roles/editor/`
installe donc réellement les deux — `ansible.builtin.dnf` avec
`state: present` est par nature idempotent (rien ne se passe pour un
paquet déjà présent), aucun conditionnel « si absent » écrit à la main.

**`yamllint` : déjà présent dans le venv D3a (1.38.0), aucun paquet
rpm installé.** Même piège qu'à ANS-1 explicitement évité : le rpm
`yamllint` existe (`fedora`, version `1.38.0-1.fc44` — identique par
coïncidence au moment de la vérification, mais rien ne garantit que
les deux restent synchronisés après une mise à jour système
indépendante du venv). Binaire faisant autorité :
`~/.venvs/ansible-lint/bin/yamllint` — même venv que `ansible-lint`
(D3a), pas un second binaire.

**Simulation d'installation de `helix`** (`dnf install --assumeno
--disablerepo=terra helix`) : 6 paquets (`helix`, `libstdc++-devel`,
`gcc-c++`, `helix-parsers`, `helix-themes`, `xsel`), 43 MiB — **aucune
correspondance `node`/`npm`** dans la liste. Même vérification pour
`kate` (3 paquets : `kate`, `kate-krunner-plugin`, `kate-plugins`,
10 MiB) — même résultat.

### 2.2 — Ce que le rôle installe et configure réellement

Exécution réelle, transaction dnf consignée
(`dnf history info`) : 9 paquets, tous depuis `fedora`/`updates`,
aucun depuis `terra` — `helix`, `kate`, `kate-krunner-plugin`,
`gcc-c++`, `xsel`, `helix-parsers`, `helix-themes`, `kate-plugins`,
`libstdc++-devel`.

**Helix** : `~/.config/helix/config.toml` déployé — une seule option,
`line-number = "absolute"` (corrélation directe avec les numéros de
ligne absolus rapportés par `ansible-lint`/`yamllint` en ligne de
commande, § 3). **Aucun `languages.toml` déployé** : le fichier fourni
par `helix-parsers` référence déjà `yaml-language-server` et
`ansible-language-server` pour le langage `yaml` sans qu'aucun des
deux ne soit présent — mesuré, pas supposé :
```
$ hx --health yaml
Configured language servers:
  ✘ yaml-language-server: 'yaml-language-server' not found in $PATH
  ✘ ansible-language-server: 'ansible-language-server' not found in $PATH
Tree-sitter parser: ✓   Highlight queries: ✓   Textobject queries: ✓   Indent queries: ✓
```
C'est exactement le comportement attendu par D12 (absence documentée,
pas corrigée) — écrire un `languages.toml` pour faire disparaître ces
deux lignes irait à l'encontre de la décision plutôt que de la
respecter.

**Kate** : configuration **exclusivement par clé nommée**
(`kwriteconfig6`/`kreadconfig6`, jamais une copie de fichier — même
discipline qu'à BUR-1 sur `kwinrulesrc`), avec relecture préalable de
chaque clé pour l'idempotence (`kwriteconfig6` ne rapporte jamais
lui-même s'il a changé quelque chose).

- **Greffons activés** : `externaltoolsplugin`, `katekonsoleplugin`,
  **[CORRIGÉ le 2026-08-08, KAT-1] `lspclientplugin`** — groupe
  `[Kate Plugins]` de `~/.config/katerc`. Ni le groupe ni les greffons
  n'existent par défaut après installation : vérifié en ouvrant Kate
  une première fois sans aucune configuration (`~/.config/katerc` ne
  contenait alors aucun groupe `[Kate Plugins]`) — `kate-plugins`
  (paquet) ne les active pas lui-même. Groupe et clé sourcés dans le
  code amont, pas supposés : `apps/lib/katepluginmanager.cpp`, tag
  `v26.04.3` (version installée exacte) — groupe littéral `"Kate
  Plugins"`, clé = `KatePluginInfo::saveName()` = nom de base du
  fichier `.so` du greffon (confirmé par lecture directe de
  `/usr/lib64/qt6/plugins/kf6/ktexteditor/*.so`, chaîne `library=`
  intégrée). **ÉTAIT** : « Le greffon client LSP n'est **pas** activé :
  D12 interdit tout serveur de langage à lui connecter, l'activer sans
  rien à lui connecter n'aurait aucun effet utile » — vrai tant
  qu'aucun serveur LSP n'existait sur ce poste (D12 ferme `npm`, pas
  `lsp-ai`, un binaire compilé depuis les sources, D19/D20). Depuis
  CMP-1, `lsp-ai` est quelque chose à lui connecter — le greffon est
  activé, câblé par `roles/completion/`
  (`~/.config/kate/lspclient/settings.json`, jamais ce fichier-ci,
  jamais une copie de fichier Plasma — même discipline qu'à BUR-1),
  complétion FIM réelle confirmée sur YAML et Python
  (`docs/completion.md` § 9, KAT-1). D12 reste en vigueur : aucun
  serveur `npm` n'est ouvert par ce changement.
- **Outils externes** déployés dans `~/.config/kate/externaltools/`
  (un fichier par outil, groupe `[General]`) : `ansible-lint-d3a` et
  `yamllint-d3a`, chacun pointant explicitement
  `~/.venvs/ansible-lint/bin/{ansible-lint,yamllint}` (D3a) — jamais
  un second binaire. Schéma de clés sourcé dans le code amont
  (`addons/externaltools/kateexternaltool.cpp`, tag `v26.04.3`) :
  `category`, `name`, `executable`, `arguments`, `mimetypes`, `save`,
  `output`, `trigger`. `arguments=%{Document:FilePath}` — syntaxe de
  substitution de variable KTextEditor sourcée dans
  `src/utils/katevariableexpansionmanager.cpp` (ktexteditor, tag
  `v6.28.0`, version installée exacte) : délimiteurs `%{`/`}`,
  variable `Document:FilePath` = chemin complet du document courant.
  `mimetypes=application/yaml`, relevé par lecture locale
  (`xdg-mime query filetype`) sur un fichier `.yml` de ce dépôt, pas
  supposé — le MIME moderne de `shared-mime-info` sur ce poste, pas
  l'ancien `text/x-yaml`/`application/x-yaml` d'autres distributions.
  `save=CurrentDocument` (le fichier est sauvegardé avant l'appel —
  sinon l'outil linterait une version obsolète sur disque).
  `output=DisplayInPane` (panneau de sortie, pas d'insertion dans le
  document). `trigger=None` (appel manuel seulement, jamais automatique
  à la sauvegarde — pour ne jamais surprendre l'opérateur par un
  lancement qu'il n'a pas demandé). **Confirmation empirique
  supplémentaire, non planifiée** : après activation du greffon,
  Kate a lui-même peuplé le même répertoire avec ~17 outils intégrés
  par défaut (`Compile and Run cpp`, `git blame`, `GoFmt`, …), au
  format **identique** (mêmes clés, même style de fichier) — validation
  croisée directe que le format choisi ici est le bon, par comparaison
  avec ce que Kate écrit lui-même, pas seulement par lecture de son
  code source.

### 2.3 — Garde D12, démontrée dans les deux sens

**Nominal** — chaque exécution du rôle (`editor_forbidden_binaries:
[node, npm]`, valeur par défaut) :
```
"Aucun des binaires interdits par D12 (node, npm) n'est présent sur le PATH — garde vérifiée."
```
**Échec forcé** — sans jamais installer `node`/`npm` réellement : la
liste des binaires interdits est temporairement redéfinie vers un nom
dont la présence est *certaine* sur ce poste (`sh`), pour prouver que
la garde détecte correctement un binaire listé quand il est présent :
```
$ ansible-playbook -e '{"editor_forbidden_binaries": ["sh"]}' roles/editor/editor.yml
fatal: [localhost]: FAILED! => {"assertion": "editor_forbidden_check.rc != 0", ...,
"msg": "Un binaire parmi sh est présent sur le PATH (/usr/bin/sh) — D12 ... interdit
node/npm comme surface d'approvisionnement ... Ce rôle s'arrête plutôt que de continuer
sur une prémisse violée."}
```
Rejoué sans dérogation immédiatement après : succès, `changed=0`. État
restauré, rien laissé dans l'état de l'échec forcé.

### 2.4 — Mode d'affichage de Kate, établi par mesure

**Ni supposé ni déduit des dépendances déclarées** (contrairement à la
table du § 1.2, qui devait se contenter de ça faute d'installation) —
Kate a réellement été lancé, une fenêtre de test identifiée par PID via
introspection D-Bus KWin (même méthode qu'en BUR-1), puis son processus
inspecté directement :
```
$ lsof -p <pid> | grep -iE 'qxcb|platforms/|wayland'
kate  mem  REG  /usr/lib64/qt6/plugins/platforms/libqwayland.so
kate  mem  REG  /usr/lib64/libQt6WaylandClient.so.6.11.1
kate  mem  REG  /usr/lib64/qt6/plugins/wayland-shell-integration/libxdg-shell.so
kate  DEL  REG  /memfd:wayland-shm  (×2)
$ lsof -p <pid> | grep -c libqxcb
0
```
Le greffon de plateforme Qt réellement chargé est `libqwayland.so`
(Wayland natif) — **`libqxcb.so` (greffon X11/XWayland) n'est chargé
à aucun moment**, occurrence zéro. Conclusion établie par la mesure :
**Kate tourne en Wayland natif**, pas via XWayland. `libX11.so.6` est
bien chargé en mémoire (compatibilité presse-papiers/glisser-déposer
historique, comportement courant d'applications Qt par ailleurs
natives Wayland) mais n'implique pas XWayland — seul le greffon de
plateforme (`platforms/*.so`) en fait foi, pas la simple présence
d'une bibliothèque X11 dans l'espace d'adressage.

Fermeture propre après chaque test, jamais un `kill -9` ni un `pkill`
générique sur le nom du processus (qui aurait pu atteindre une fenêtre
de l'opérateur) : `busctl --user call
org.kde.kate-<pid> /MainApplication org.qtproject.Qt.QCoreApplication quit`,
vérifié par `pgrep` immédiatement après.

## 3. Flux de travail CLI — vérifier un fichier YAML ou un rôle Ansible sans complétion en éditeur

D12 retire la complétion et les diagnostics en place dans l'éditeur —
ce que voici les remplace. **Binaires faisant autorité, les deux dans
le même venv (D3a), jamais un second installé :**
`~/.venvs/ansible-lint/bin/ansible-lint` et
`~/.venvs/ansible-lint/bin/yamllint`.

**Vérifier un fichier YAML isolé (syntaxe, style)** :
```
$ ~/.venvs/ansible-lint/bin/yamllint chemin/vers/fichier.yml
```
Rapporte des avertissements/erreurs de style (indentation, lignes
trop longues, espaces superflus) au format `fichier:ligne:colonne`.

**Vérifier un rôle ou playbook Ansible (sémantique, bonnes pratiques,
pas seulement la syntaxe YAML)** :
```
$ ~/.venvs/ansible-lint/bin/ansible-lint chemin/vers/rôle_ou_playbook
```
Englobe les vérifications `yamllint` en interne (même version que
l'appel autonome, D3a) et ajoute les règles propres à Ansible (idempotence
déclarée, modules dépréciés, `changed_when` manquant, etc. — déjà
rencontrées et corrigées dans `roles/desktop/`, `roles/editor/`).

**Depuis Kate, sans quitter l'éditeur** : menu Outils → Ansible →
« ansible-lint (venv D3a) » ou « yamllint (venv D3a) » (§ 2.2) — lance
le binaire ci-dessus sur le document courant (sauvegardé d'abord),
affiche la sortie dans un panneau. Trigger manuel uniquement, jamais
automatique à la sauvegarde.

**Depuis Helix** : pas d'équivalent en éditeur (D12) — basculer vers
`kitty` (déjà installé, BUR-1) et exécuter la commande `yamllint`/
`ansible-lint` ci-dessus directement, ou utiliser la commande `:sh`
de Helix (`:sh ~/.venvs/ansible-lint/bin/yamllint %`, où `%` est
substitué par Helix avec le chemin du fichier courant — comportement
documenté de Helix lui-même, pas une extension de ce dépôt).

**Ce que `ansible-lint --version` doit toujours rapporter**, avant et
après tout changement touchant `roles/editor/` ou le venv D3a :
`ansible-core:2.20.7` — toute autre valeur signale qu'un second
binaire a été introduit ou que le venv a changé sans que ce dépôt le
documente (garde automatisée dans `roles/editor/tasks/main.yml`,
rejouée à chaque exécution du rôle).

## Validation — EDI-0 (résolution en lecture seule, 2026-08-06)

**Commandes exécutées** — toutes non modifiantes sauf l'incident déjà
signalé en tête de document :

| # | Commande | Nature | Code/sortie |
|---|---|---|---|
| 1 | `dnf repoquery --available --disablerepo=terra --qf ...` | métadonnées, dix dépôts | succès, 89062 lignes |
| 2 | `dnf repoquery --repo=terra --available --assumeno --qf ...` | métadonnées, Terra, gardée | succès, 2739 lignes, aucune invite |
| 3 | `grep`/`sort` locaux sur les deux extractions | filtrage local | succès |
| 4 | `command -v node`/`npm` | test de présence | absence confirmée (code non nul, comportement attendu) |
| 5 | `dnf install --assumeno --disablerepo=terra <paquet>` (×17, un par candidat/complément fedora) | simulation d'installation | succès, aucune installation |
| 6 | `dnf install --assumeno --repo=terra --repo=updates --repo=fedora <paquet>` (×3 : zed, cursor, micro.nightly) | simulation d'installation, Terra gardée | succès, aucune installation, aucune invite |
| 7 | `dnf repoquery --requires <paquet>` (×~10) | dépendances déclarées | succès |
| 8 | `dnf repoquery -l cursor` | liste de fichiers (métadonnées, pas de contenu) | succès |
| 9 | **`dnf download --repo=terra ... cursor`** | **télécharge le paquet — incident, voir tête de document** | succès (mais ne devait pas être exécutée), fichiers supprimés aussitôt |
| 10 | `flatpak remotes` / `flatpak remote-ls fedora` | lecture de remote déjà configuré | succès |
| 11 | `ls /etc/yum.repos.d/`, `cat` sur fichiers `.repo` désactivés | lecture locale | succès |
| 12 | `rpm -q`/`rpm -qa` (plusieurs) | état paquets installés | succès |
| 13 | `~/.venvs/ansible-lint/bin/ansible-lint --version`, `pip list` | lecture d'état du venv D3a | succès |
| 14 | Lecture externe : `zed.dev/docs/telemetry`, `zed.dev/docs/linux`, `zed.dev/docs/languages/yaml`, `cursor.com/en/security`, `cursor.com/en/data-use` | requêtes HTTP GET, documentation amont | succès |
| 15 | Lecture externe : `code.visualstudio.com/api/...`, `visualstudio.microsoft.com/license-terms/...`, `cursor.com/en/docs/faq` | requêtes HTTP GET | **échec (404) à trois reprises** — signalé, pas de fait affirmé à partir de ces tentatives |

**Actions privilégiées : aucune.** Toutes les commandes ci-dessus
s'exécutent sans `sudo` (simulations `dnf --assumeno`, lectures
locales, requêtes réseau anonymes).

**Décompte du jeton de vérification, `CLAUDE.md` exclu, à la clôture
de EDI-0** : trois marqueurs actionnables — nativité Wayland de
`cursor` (§ 1.2), mécanisme de récupération du serveur YAML par Zed
(§ 1.4), texte exact des conditions d'utilisation du Marketplace
Microsoft (§ 1.3). **[MIS À JOUR le 2026-08-06, § 2]** : le premier est
requalifié (Cursor écarté par décision, § 1.2) — deux marqueurs
actionnables restent à la clôture de ce document. Aucun paragraphe
n'ajoute d'occurrence nue du jeton lui-même.

**Confirmations finales** : aucun paquet installé (toutes les
installations simulées via `--assumeno`, jamais confirmées) ; aucune
extension téléchargée ; aucun dépôt ajouté (le remote Flatpak `fedora`
et le COPR PyCharm désactivé préexistaient, ni l'un ni l'autre créé
ici) ; aucune configuration d'éditeur créée ; aucun fichier écrit hors
de ce dépôt, à l'exception des deux `.rpm` de l'incident signalé,
supprimés dans la même série.

## Validation — EDI-1 (déploiement, 2026-08-06)

**Actions privilégiées, exhaustives** :

| # | Commande | Cible | Motif |
|---|---|---|---|
| 1 | `dnf install -y helix kate` (module `ansible.builtin.dnf`, `become: true`, `disablerepo: terra`) | paquets `helix`, `kate` (dépôts `fedora`/`updates`) | seule installation demandée, choix de l'opérateur D13 |

Toutes les autres actions du rôle (garde D12, `kwriteconfig6`/
`kreadconfig6` sur `katerc` et les outils externes, écriture de
`~/.config/helix/config.toml`, appel `ansible-lint --version`)
s'exécutent sans `sudo`.

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/editor/editor.yml   # succès
$ ansible-playbook roles/editor/editor.yml                  # succès, changed>0 (première écriture réelle)
$ ansible-playbook roles/editor/editor.yml                  # succès, changed=0 (idempotence confirmée)
$ ansible-playbook --check roles/editor/editor.yml          # succès, aucune écriture
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/editor/
Passed: 0 failure(s), 0 warning(s) — profil production, aucune dérogation noqa
```

**Garde D12** : démontrée dans les deux sens, § 2.3.

**`command -v node npm` après exécution** :
```
$ command -v node npm; echo "rc=$?"
rc=1
```
Vide, code non nul — confirmé.

**`ansible-lint --version` après exécution** :
```
ansible-lint 26.6.0 using ansible-core:2.20.7 ansible-compat:26.6.0 ruamel-yaml:0.19.1 ruamel-yaml-clib:None
```
`ansible-core:2.20.7` inchangé — confirmé par la garde automatisée du
rôle (§ 2.2) et rejoué manuellement ci-dessus. Défaut réel trouvé et
corrigé en cours de route : `ansible-lint --version` insère des codes
ANSI de couleur *au milieu* de sa propre sortie (entre `ansible-core:`
et `2.20.7`), faisant échouer à tort une comparaison de sous-chaîne
naïve — corrigé en fixant `NO_COLOR=1` dans l'environnement de la
tâche (`--nocolor` documenté comme équivalent par `ansible-lint
--help`).

**Mode d'affichage de Kate** : Wayland natif, établi par mesure
(chargement de `libqwayland.so`, absence de `libqxcb.so` dans l'espace
d'adressage du processus réel) — § 2.4.

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : deux
marqueurs actionnables dans ce document à la clôture de EDI-1 —
mécanisme de récupération du serveur YAML par Zed (§ 1.4), texte exact
des conditions d'utilisation du Marketplace Microsoft (§ 1.3). Le
marqueur sur la nativité Wayland de Cursor, ouvert en EDI-0, est
requalifié (§ 1.2) — Cursor n'a jamais été installé ni testé, ce n'est
pas une fermeture par vérification.

**Confirmations finales** : aucun paquet installé hors `helix`/`kate`
et leurs dépendances (`fedora`/`updates` uniquement, aucune depuis
`terra` — vérifié par `dnf history info`) ; aucun dépôt ajouté ; aucun
`dnf download` dans cette série (l'incident appartient à EDI-0, pas
répété ici) ; `kwinrulesrc` non touché (seul `katerc` et
`~/.config/kate/externaltools/` modifiés, tous deux hors du périmètre
KWin) ; `sudoers`, `/etc/cdi/`, `kwinrc`, `gpu_mux_mode` non touchés ;
aucune déconnexion de session ; aucun redémarrage.

## Validation — KAT-1 (greffon LSP de Kate, 2026-08-08)

**Périmètre de ce document pour KAT-1** : l'activation du greffon
`lspclientplugin` (§ 2, ci-dessus) — le câblage vers `lsp-ai`, la
preuve de complétion réelle et le décompte complet des actions
privilégiées de ce livrable (installation puis retrait de deux outils
de test Wayland, `wtype`/`ydotool`) vivent dans
[`docs/completion.md`](completion.md) § 9, pas ici (une règle ne se
recopie pas à deux endroits — même principe appliqué au contenu d'un
livrable qui touche deux documents).

**Validation Ansible, `roles/editor/`** :
```
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/editor/
Passed: 0 failure(s), 0 warning(s) — profil production
$ ansible-playbook --check roles/editor/editor.yml   # changed=0
$ ansible-playbook roles/editor/editor.yml           # changed=0 (état déjà convergent)
$ ansible-playbook roles/editor/editor.yml           # changed=0 (idempotence reconfirmée)
```

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : inchangé
par ce livrable — les deux marqueurs actionnables de EDI-1 restent
ouverts (mécanisme de récupération du serveur YAML par Zed § 1.4,
texte exact des conditions Marketplace Microsoft § 1.3), hors
périmètre de KAT-1. Le marqueur fermé par ce livrable (compatibilité
Kate/`lsp-ai`) vivait dans `docs/completion.md`, pas ici.

**Confirmations finales** : `sudoers`, `/etc/cdi/`, `kwinrc`,
`kwinrulesrc`, `gpu_mux_mode`, `terra.repo` non touchés ; aucun
redémarrage — détail complet des actions privilégiées et de leurs
tentatives sans privilège dans `docs/completion.md` § 9.5.

## Voir aussi

- [`docs/desktop.md`](desktop.md) — `kitty`, déjà installé, méthode de
  mesure de coût/dépendances Wayland dont celle-ci s'inspire.
- [`docs/repositories.md`](repositories.md) — ancrage de confiance des
  onze dépôts, D7/D10, pertinent pour tout dépôt tiers supplémentaire
  qu'un éditeur hors de cette liste exigerait.
- [`docs/ansible-chain.md`](ansible-chain.md) — D3a, le binaire
  `ansible-lint` que tout candidat devrait réutiliser plutôt que
  dupliquer.
- [`docs/completion.md`](completion.md) § 9 — intégration complète du
  greffon LSP de Kate à `lsp-ai` (KAT-1), preuve, méthode, tableau des
  actions privilégiées.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici.
- [`roles/editor/`](../roles/editor/) — rôle Ansible déployant Helix et
  Kate (§ 2), détails d'exécution dans son propre `README.md`.
