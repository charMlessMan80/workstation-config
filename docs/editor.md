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
| Cursor | `gtk3` + `libX11` — **nativité Wayland non déterminée** : l'inspection du lanceur (`Exec=` du `.desktop`) aurait nécessité de télécharger le paquet, évité après l'incident ci-dessus. `@VERIF : mode de lancement Wayland/XWayland réel de cursor — nécessite d'inspecter /usr/share/applications/cursor.desktop sans télécharger tout le paquet (introspection de métadonnées de dépôt insuffisante pour lire un contenu de fichier, seulement sa liste) ; ou installer/tester réellement dans un livrable qui en a le mandat.` | Non déterminé |
| Terminaux (Neovim, Vim, Emacs, Helix, Micro, Kakoune, joe) | Rendu délégué au terminal (`kitty`, déjà Wayland natif — BUR-0) | Sans objet : ces binaires n'ouvrent pas de fenêtre propre en usage CLI |

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
   étape manuelle (mécanisme de récupération non documenté dans la
   page consultée, marqué `@VERIF`).
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

## Validation

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

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : trois
marqueurs actionnables dans ce document — nativité Wayland de
`cursor` (§ 1.2, conséquence directe de l'incident signalé), mécanisme
de récupération du serveur YAML par Zed (§ 1.4), texte exact des
conditions d'utilisation du Marketplace Microsoft (§ 1.3). Ce
paragraphe n'ajoute aucune occurrence du jeton lui-même.

**Confirmations finales** : aucun paquet installé (toutes les
installations simulées via `--assumeno`, jamais confirmées) ; aucune
extension téléchargée ; aucun dépôt ajouté (le remote Flatpak `fedora`
et le COPR PyCharm désactivé préexistaient, ni l'un ni l'autre créé
ici) ; aucune configuration d'éditeur créée ; aucun fichier écrit hors
de ce dépôt, à l'exception des deux `.rpm` de l'incident signalé,
supprimés dans la même série.

## Voir aussi

- [`docs/desktop.md`](desktop.md) — `kitty`, déjà installé, méthode de
  mesure de coût/dépendances Wayland dont celle-ci s'inspire.
- [`docs/repositories.md`](repositories.md) — ancrage de confiance des
  onze dépôts, D7/D10, pertinent pour tout dépôt tiers supplémentaire
  qu'un éditeur hors de cette liste exigerait.
- [`docs/ansible-chain.md`](ansible-chain.md) — D3a, le binaire
  `ansible-lint` que tout candidat devrait réutiliser plutôt que
  dupliquer.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici.
