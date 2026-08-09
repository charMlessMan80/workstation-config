# Bureau et terminal — placement multi-écran, résolution en lecture seule

**Ce document ne configure rien.** Toutes les commandes citées ont été
exécutées le 2026-08-06, en lecture seule : aucun paquet installé,
aucune configuration Plasma modifiée, aucune règle KWin créée, aucun
autostart créé, aucun mode d'affichage changé, aucun fichier écrit hors
de ce dépôt. **Aucune option n'est appliquée, aucun terminal n'est
recommandé** — le choix revient à l'opérateur.

Faits déjà établis, non revérifiés ici : Fedora 44 KDE Plasma, session
Wayland ; sorties `DP-3` (ScreenPad Plus, iGPU AMD, `priority 2`) et
`eDP-1` (dalle principale, `priority 1`) depuis la bascule MUX ;
Konsole présent par défaut avec Plasma.

**Contradiction relevée, signalée avant d'aller plus loin** (`CLAUDE.md`
§ Avant d'agir — un fait observé qui contredit un fait déjà documenté
s'arrête et se signale) : le contexte de cette demande affirme
`eDP-1` en mode actif `@60` avec `@240` disponible mais non actif.
**Lecture directe, à l'instant, contredit ce point** :
```
$ kscreen-doctor -o
Output: 2 eDP-1 c5c4f802-3ee8-459a-8724-bd1810d84686
  Modes: 18:2560x1600@240.00*!  19:2560x1600@60.00 ...
```
Le mode `18` porte **à la fois** `*` (courant) et `!` (préféré) — `@240`
est actif, pas `@60`. `docs/machine-facts.md` (§ Affichage, état
post-bascule du 2026-08-05) documentait `@60` actif avec sélection
manuelle requise pour `@240` — cohérent avec le contexte de cette
demande, donc l'écart n'est pas une erreur de lecture de cette série.
**Cause non établie ici** : sélection manuelle faite par l'opérateur
entre les deux relevés (le plus probable, l'ancienne note documentait
justement que cette sélection restait à faire), ou autre changement non
tracé. Aucune action entreprise (aucun mode d'affichage changé, comme
l'exige ce livrable) — juste consigné, avec la date, dans
`docs/machine-facts.md` § Affichage.

## 1. Comment KWin désigne un écran dans une règle de fenêtre

### 1.1 — Par index, jamais par nom ni par UUID — établi par le schéma source, pas supposé

Aucune documentation locale (pas de page de manuel `kwinrulesrc`, le
paquet `kwin-common` ne livre que les traductions et le greffon de
configuration compilé, pas de fichier `.kcfg` pour les règles). Établi
par lecture directe du schéma source de KWin, à la version **exactement
installée sur ce poste** (`kwin-common-6.7.3-1.fc44`, tag `v6.7.3`,
externe, `github.com/KDE/kwin`) :

```
$ rpm -q kwin-common
kwin-common-6.7.3-1.fc44.x86_64
```
```
# src/rulesettings.kcfg, tag v6.7.3 — 81 <entry> au total, exhaustif
<entry name="screen" type="Int">
  <label>Screen number</label>
</entry>
<entry name="screenrule" type="Int">
  <label>Screen number rule type</label>
</entry>
```
**Aucune entrée nommée `output`, `outputName` ou `uuid` n'existe dans ce
schéma** — vérifié sur les 81 entrées, pas seulement cherché par motif.
La seule propriété d'écran disponible dans une règle KWin est un entier
brut : un **index**, pas un nom de sortie (`DP-3`), pas l'UUID stable
que `kscreen-doctor -o` expose par ailleurs.

### 1.2 — Ce que cet index représente, établi par le code d'application de la règle

Lecture directe de `src/rules.cpp`, même tag :
```cpp
// Mémorisation (quand l'utilisateur demande à KWin de retenir l'écran) :
if NOW_REMEMBER (Screen, screen) {
    const int index = workspace()->outputs().indexOf(c->output());
    updated = updated || screen != index;
    screen = index;
}

// Application de la règle à une fenêtre :
LogicalOutput *WindowRules::checkOutput(LogicalOutput *output, bool init) const
{
    int ret = workspace()->outputs().indexOf(output);
    for (Rules *rule : rules) {
        if (rule->applyScreen(ret, init)) break;
    }
    LogicalOutput *ruleOutput = workspace()->outputs().value(ret);
    return ruleOutput ? ruleOutput : output;
}
```
**La valeur stockée est la position dans la liste `workspace()->outputs()`
au moment de la mémorisation — pas un identifiant persistant.** Au
moment de l'application, KWin relit cette même liste et résout l'entier
en écran réel par position. Si l'ordre de cette liste diffère entre le
moment où la règle a été enregistrée et le moment où elle est
réappliquée, la règle place la fenêtre sur le **mauvais écran physique,
sans aucun message d'erreur** — exactement le mode d'échec silencieux
nommé par la demande.

**Réponse au rappel sur l'UUID (`kscreen-doctor -o`)** : KWin ne sait
**pas** l'utiliser dans une règle — le schéma ne prévoit aucun champ
pour un identifiant de ce type (§ 1.1). L'UUID existe dans KScreen
(couche de configuration d'écrans indépendante), pas dans le moteur de
règles de fenêtres de KWin.

### 1.3 — Stabilité de l'index : établie partiellement, marquée pour le reste [REQUALIFIÉ le 2026-08-06, § 6.2/6.3]

**Ce qui est établi, par lecture, sans manipuler d'écran** : l'ordre
d'énumération des connecteurs DRM (niveau noyau, sous la liste KWin,
pas prouvé identique à elle) est **identique sur les trois derniers
démarrages relevés**, y compris de part et d'autre de la bascule MUX du
2026-08-05 :
```
$ journalctl -k -b 0  --no-pager | grep -E 'drm.*(edp|dp-)[0-9]:'
$ journalctl -k -b -1 --no-pager | grep -E 'drm.*(edp|dp-)[0-9]:'
$ journalctl -k -b -2 --no-pager | grep -E 'drm.*(edp|dp-)[0-9]:'
# Les trois : eDP-1, DP-1, DP-2, DP-3, dans cet ordre, sans exception
```
Ce poste n'a par ailleurs **aucun écran externe branché** — `DP-3` et
`eDP-1` sont tous deux des panneaux internes fixes de ce châssis, pas
des moniteurs amovibles. Le risque de branchement à chaud, nommé par la
demande, est réel en général mais **sans objet pour la configuration
actuelle de ce poste précis**.

**Ce qui n'est pas établi** : que l'ordre `workspace()->outputs()` de
KWin (Qt, niveau compositeur) suit fidèlement cet ordre d'énumération
DRM (niveau noyau) — les deux sont vraisemblablement liés mais aucune
lecture de cette série ne le prouve directement, et rien ne garantit
qu'il reste identique après un branchement externe futur (un nouveau
connecteur s'insère dans cette liste à une position non déterminée par
cette lecture). `@VERIF : correspondance exacte entre l'ordre
d'énumération DRM et l'ordre workspace()->outputs() de KWin, et
stabilité de ce dernier lors du branchement d'un écran externe — à
vérifier sans manipuler d'écran réel en comparant, sur plusieurs
sessions, la sortie de checkOutput() via le service D-Bus de KWin
(org.kde.KWin, introspection en lecture seule) ou, plus simplement, en
observant le comportement réel d'une règle une fois qu'une (1) a été
créée et testée dans un livrable ultérieur — non fait ici, ce livrable
n'en crée aucune.`

**[REQUALIFIÉ le 2026-08-06, BUR-1, § 6.2/6.3]** — Le marqueur
ci-dessus portait sur la correspondance entre l'ordre d'énumération
DRM et l'ordre `workspace()->outputs()` de KWin, dans l'hypothèse où
le placement se ferait par **index d'écran**. Cette hypothèse
elle-même s'est révélée fausse à l'usage : la propriété
`screen`/`screenrule` s'est montrée confondue avec
`org.kde.KWin.activeOutputName`, indépendamment de tout ordre
d'énumération — § 6.2. Le mécanisme finalement retenu (`position`+
`size`, § 6.3) ne consulte l'index de sortie à aucun moment. **Le
marqueur ci-dessus reste donc non vérifié dans l'absolu, mais devient
non bloquant** : ce dépôt ne s'appuie plus sur cet index pour quoi que
ce soit, donc son statut non vérifié n'affecte aucune garantie
actuellement en vigueur — à revisiter seulement si un futur livrable
réintroduit une dépendance à `screen`.

## 2. Mécanisme de démarrage

Trois mécanismes distincts, établis par lecture des unités systemd
utilisateur réellement chargées sur ce poste (`systemctl --user`,
lecture seule) :

### 2.1 — Autostart XDG

`~/.config/autostart/*.desktop` (utilisateur, absent actuellement —
vérifié, `ls` échoue) ou `/etc/xdg/autostart/*.desktop` (système,
peuplé). Depuis Plasma 6 sur ce poste, ces fichiers sont importés par le
générateur **systemd natif** (pas un mécanisme propre à Plasma) :
```
$ systemctl --user list-units '*autostart*'
xdg-desktop-autostart.target        loaded active active  Startup of XDG autostart applications
app-org.kde.kalendarac@autostart.service   loaded active running  Calendar Reminders
[...]
$ man -w systemd.special   # xdg-desktop-autostart.target y est documenté, externe
```
Chaque application autostart devient une unité `app-<nom>@autostart.service`
distincte — observable individuellement (démarrée, échouée, etc.).
**Contrôle fin d'ordre disponible et déjà en usage sur ce poste** (pas
supposé — lu dans les fichiers système réels) :
```
$ grep -h "X-KDE-autostart-phase\|X-KDE-autostart-after" /etc/xdg/autostart/*.desktop
org.kde.kalendarac.desktop:X-KDE-autostart-phase=2
org.kde.discover.notifier.desktop:X-KDE-autostart-phase=1
baloo_file.desktop:X-KDE-autostart-phase=0
vboxclient.desktop:X-KDE-autostart-after=panel
```
`X-KDE-autostart-phase` (0/1/2) et `X-KDE-autostart-after=` sont des
clés `.desktop` propres à KDE, en plus de l'ordre systemd — deux leviers
distincts, tous deux déjà exercés par des applications système sur ce
poste.

### 2.2 — Restauration de session Plasma

Mécanisme **distinct** de l'autostart XDG : `ksmserver` (le gestionnaire
de session Plasma) peut rouvrir les fenêtres ouvertes à la dernière
déconnexion. Porté par `plasma-restoresession.service`, lu directement :
```
$ systemctl --user cat plasma-restoresession.service
[Unit]
After=graphical-session.target
[Service]
Type=oneshot
ExecStart=-/usr/lib64/qt6/bin/qdbus org.kde.ksmserver /KSMServer org.kde.KSMServerInterface.restoreSession
```
Ce service tourne **après** `graphical-session.target` — plus tard que
tout le reste de la chaîne (§ 2.3). Contrôlé par une case à cocher dans
les paramètres de session (« se souvenir des applications en cours »),
**pas un fichier qu'on écrit soi-même pour une application donnée** —
inadapté à l'objectif (« un terminal précis, toujours ») : ce mécanisme
rejoue l'état de la dernière session, pas une intention fixe.
`~/.config/session/` (l'ancien emplacement des fichiers d'état de
session sous X11) n'existe pas sur ce poste — la case n'a jamais été
activée ou aucune session n'a encore été sauvegardée.

### 2.3 — Unité systemd utilisateur personnalisée

Un fichier écrit à la main dans `~/.config/systemd/user/`, avec un
`After=`/`PartOf=` choisi explicitement — offre un contrôle d'ordre plus
fin qu'une simple entrée autostart (pas de dépendance à
`X-KDE-autostart-phase`, ordre systemd complet disponible). C'est le
même mécanisme que celui exploité par les `.desktop` d'autostart
eux-mêmes (§ 2.1) — juste écrit directement plutôt que généré.

### 2.4 — Ordre réel par rapport à la disponibilité des écrans, lu dans le graphe de dépendances réel

```
$ systemctl --user show plasma-kwin_wayland.service -p Before
Before=plasma-kcminit.service plasma-core.target plasma-ksmserver.service
       shutdown.target plasma-workspace-wayland.target plasma-kwallet-pam.service

$ systemctl --user cat plasma-ksmserver.service
After=plasma-kwin_wayland.service plasma-kcminit.service

$ systemctl --user cat plasma-workspace.target
Requires=plasma-core.target
Requires=graphical-session.target
Wants=... xdg-desktop-autostart.target ...
Before=graphical-session.target xdg-desktop-autostart.target plasma-restoresession.service
```
**Chaîne établie, par lecture, pas par supposition** : `kwin_wayland`
démarre en tout premier (`Before=` tout le reste) → `ksmserver` démarre
après lui explicitement → `plasma-core.target`/`plasma-workspace.target`
sont requis avant `xdg-desktop-autostart.target` → la restauration de
session Plasma (§ 2.2) tourne encore après. **Les applications
autostart démarrent donc, par construction de la chaîne systemd, après
que le processus KWin est déjà lancé** — favorable au risque nommé par
la demande.

**Ce que cette lecture ne prouve pas** : l'ordre de démarrage des
*processus* (ce que systemd garantit) n'est pas la même chose que
l'ordre de disponibilité de l'**état interne** de KWin (topologie de
sorties entièrement énumérée, `kwinrulesrc` entièrement chargé). KWin
étant premier dans la chaîne et l'énumération des sorties étant l'une
de ses toutes premières responsabilités (nécessaire à son propre rendu),
le risque semble faible sur le papier — mais rien dans cette série ne
le mesure directement, puisque mesurer supposerait de créer la règle et
l'autostart que ce livrable interdit. **Comment le constater, pour un
livrable ultérieur** : comparer, via `journalctl --user`, l'horodatage
du premier message de `plasma-kwin_wayland.service` établissant la
topologie de sorties à l'horodatage de démarrage du premier processus
autostart réel — les deux journaux existent déjà indépendamment,
lisibles sans rien créer de nouveau.

## 3. Candidats terminaux — disponibles dans les onze dépôts activés

**Incident de méthode à signaler** : la première requête de cette
recherche (`dnf list --available <liste de candidats>`) a été lancée
sans `--assumeno`, alors que Terra a un effet de bord déjà documenté
(invite d'import de clé GPG) — violation de la règle `CLAUDE.md` §
Sourcing sur la reproduction gardée d'un effet déjà connu. Vérifié
immédiatement après coup, pas supposé sans risque : aucune clé n'a été
importée (`rpm -q gpg-pubkey` : la clé Terra porte une date d'installation
du 2026-08-04, session antérieure, rien de nouveau à l'horodatage de
cette commande). Corrigé pour la suite de cette série
(`--repo=terra --assumeno` isolé, puis `--disablerepo=terra` pour tout
le reste).

**Terra ne fournit aucun des candidats** (vérifié avec garde, motif
isolé) :
```
$ dnf --repo=terra --assumeno list --available <candidats>
No matching packages to list
```
**Vingt candidats trouvés, tous depuis `fedora`/`updates`** (requête sur
les onze dépôts, Terra désactivé pour cette commande précise puisqu'il
n'apporte rien ici) :

| Paquet | Dépôt | Taille installée | Résumé |
|---|---|---|---|
| `konsole` | fedora + updates | 6.5 MiB | Terminal KDE — **déjà installé, par défaut avec Plasma** |
| `yakuake` | fedora + updates | 2.5 MiB | Terminal KDE déroulant (même moteur que Konsole) |
| `alacritty` | updates | 7.5 MiB | Terminal OpenGL, Rust |
| `kitty` | updates | 14.9 MiB | Terminal GPU, Python/C |
| `foot` | fedora + updates | 875.7 KiB | Terminal **Wayland uniquement** |
| `xterm` | fedora | 1.9 MiB | Terminal X11 historique |
| `rxvt-unicode` | fedora | 2.9 MiB | Terminal X11 historique (urxvt) |
| `st` | fedora | 106.6 KiB | Terminal suckless, X11 |
| `eterm` | fedora | 2.9 MiB | Terminal Enlightenment, X11 |
| `gnome-terminal` | fedora | 8.5 MiB | Terminal GNOME (VTE/GTK3) |
| `xfce4-terminal` | fedora | 2.2 MiB | Terminal XFCE (VTE/GTK3) |
| `terminator` | fedora | 2.8 MiB | Multiplexage de terminaux GNOME (GTK3) |
| `tilix` | fedora | 5.0 MiB | Terminal en tuiles (VTE/GTK3) |
| `terminology` | fedora | 3.5 MiB | Terminal Enlightenment (EFL) |
| `sakura` | fedora | 226.2 KiB | Terminal minimal (VTE/GTK3) |
| `lxterminal` | fedora | 421.9 KiB | Terminal LXDE (VTE/GTK3) |
| `guake` | fedora | 3.2 MiB | Terminal déroulant GNOME (GTK3) |
| `blackbox-terminal` | fedora + updates | 882.5 KiB | Terminal GNOME (VTE/**GTK4**) |
| `deepin-terminal` | fedora + updates | 91.6 MiB | Terminal Deepin (Qt) |
| `cool-retro-term` | fedora | 2.2 MiB | Simulation d'écran CRT (Qt5/QML) |

**Absents des onze dépôts, vérifié explicitement, pas seulement non
cherché** : `wezterm`, `ghostty`, `contour`, `roxterm`.
```
$ dnf --disablerepo=terra list --available wezterm ghostty
No matching packages to list
```

### 3.1 — Support Wayland natif : établi par les dépendances déclarées, pas par mémoire

Vérifié paquet par paquet (`dnf repoquery --requires`, métadonnées de
dépôt, sans installation) — puis, pour les paquets GTK3/GTK4/Qt6/EFL
dont le lien Wayland n'apparaît pas directement (la capacité vient de la
boîte à outils, pas de l'application elle-même), vérifié que la boîte à
outils correspondante dépend bien de `libwayland-client`/`cursor`/`egl` :
```
$ dnf repoquery --requires gtk3 | grep -i wayland    → présent
$ dnf repoquery --requires gtk4 | grep -i wayland    → présent
$ dnf repoquery --requires qt6-qtbase-gui | grep -i wayland → présent
$ dnf repoquery --requires efl | grep -i wayland     → présent
```

- **Wayland natif confirmé, directement ou via la boîte à outils** :
  `konsole`, `yakuake` (le seul à référencer explicitement
  `libKWaylandClient` — intégration KDE-Wayland la plus directe du lot),
  `alacritty` (`libwayland-egl` en dépendance directe), `kitty`
  (Wayland **et** X11, les deux backends), `foot` (**Wayland
  exclusivement — aucune dépendance X11 du tout**, seul candidat dans ce
  cas), `gnome-terminal`, `xfce4-terminal`, `terminator`, `tilix`,
  `sakura`, `lxterminal`, `guake` (via GTK3), `terminology` (via EFL),
  `blackbox-terminal` (via GTK4), `deepin-terminal`, `cool-retro-term`
  (via Qt).
- **X11 exclusivement, aucune boîte à outils Wayland-capable en
  dépendance** — tourneraient sous XWayland, pas nativement : `xterm`,
  `rxvt-unicode`, `st`, `eterm`. Confirmé par l'absence totale de
  dépendance GTK/Qt/EFL, pas seulement l'absence de mention Wayland :
  ```
  $ dnf repoquery --requires xterm rxvt-unicode st eterm | grep -iE "gtk|qt[0-9]|efl"
  (rien)
  ```

### 3.2 — Coût réel, mesuré par simulation d'installation (`--assumeno`, rien installé)

```
$ dnf --disablerepo=terra install --assumeno <candidat>
```
Nombre total de paquets que la transaction *installerait* (candidat +
dépendances neuves — les dépendances déjà présentes sur ce poste, comme
`gtk3`/`gtk4`/`qt6-qtbase-gui`/`kf6-kwindowsystem`, tous trois déjà
installés, ne comptent pas) :

| Paquet | Paquets au total | Note |
|---|---|---|
| `konsole` | 0 (déjà installé) | |
| `yakuake` | 1 | Toutes ses dépendances Qt6/KF6/Wayland déjà satisfaites par l'installation Plasma existante |
| `alacritty` | 1 | |
| `st` | 1 | |
| `rxvt-unicode` | 2 | |
| `foot` | 3 | |
| `xterm` | 3 | |
| `terminology` | 4 | tire `efl` (non installé), `bullet`, `luajit` |
| `sakura` | 4 | |
| `lxterminal` | 4 | |
| `kitty` | 5 | |
| `gnome-terminal` | 5 | |
| `terminator` | 6 | |
| `blackbox-terminal` | 6 | |
| `guake` | 7 | |
| `deepin-terminal` | 7 | |
| `eterm` | 7 | |
| `xfce4-terminal` | 9 | |
| `tilix` | 9 | |
| `cool-retro-term` | 82 | tire une pile **Qt5** complète (`qt5-qtbase`, `qt5-qtdeclarative`, `qt5-qtquickcontrols*`, `qt5-qtsvg`, ...), 47 MiB — seul candidat à faire coexister deux générations de Qt sur ce poste |

**Konsole comme option à part entière** : déjà installé, zéro paquet
supplémentaire, seul candidat dont l'intégration Plasma est native au
sens strict (KF6, pas seulement Qt6 — `libKF6ConfigWidgets`,
`libKF6WindowSystem`, etc., directement dans ses dépendances). C'est
« ce que l'outil fait déjà lui-même » au sens de `CLAUDE.md` § Avant
d'agir, appliqué au bureau plutôt qu'à un rôle Ansible.

## 4. Contrainte propre au ScreenPad Plus (`DP-3`, 3840×1100)

**Mise à l'échelle relevée, pas supposée** :
```
$ kscreen-doctor -o
Output: 1 DP-3 ...
  Modes: 3840x1100@60.02*!
  Geometry: 0,1067 2560x734
  Scale: 1.5
```
Résolution physique `3840x1100`, mise à l'échelle Plasma `1.5` →
surface logique `2560×734` (`3840/1.5=2560`, `1100/1.5≈733,3`, cohérent
avec la géométrie rapportée). `eDP-1` porte la **même** échelle (`1.5`)
— pas de disparité de densité entre les deux sorties à prendre en
compte pour un déplacement de fenêtre entre écrans.

**Estimation du nombre de colonnes, méthode explicitée, pas un
chiffre affirmé sans base** : aucun profil Konsole n'existe encore sur
ce poste (`~/.local/share/konsole/*.profile` : aucun fichier) et
`kdeglobals` ne surcharge pas la police à chasse fixe — la police et la
taille réelles dépendent d'un choix non encore fait par l'opérateur.
Approximation usuelle pour une police à chasse fixe (largeur de
cellule ≈ 0,6 × taille de police en pixels ; conversion point→pixel à
96 ppp, base indépendante de la mise à l'échelle puisque les
applications Qt/GTK disposent leur interface en pixels logiques avant
application du facteur d'échelle) :

| Taille de police | Largeur de cellule ≈ | Colonnes sur 2560 px logiques | Hauteur de ligne ≈ | Lignes sur 734 px logiques |
|---|---|---|---|---|
| 10 pt | 8 px | ≈ 320 | 17 px | ≈ 43 |
| 12 pt | 9,6 px | ≈ 267 | 20 px | ≈ 37 |
| 14 pt | 11,2 px | ≈ 229 | 24 px | ≈ 31 |

**Implication** : un terminal maximisé sur `DP-3` offre, à toute taille
de police raisonnable, **bien plus de colonnes qu'une ligne de texte
n'en utilise couramment** (80–120), pour un nombre de lignes au
contraire réduit (30–45) — le format 3840×1100 n'est pas praticable
comme un unique panneau maximisé de la même façon qu'un écran classique.
**Intérêt direct d'un découpage en volets verticaux**, déjà natif dans
plusieurs candidats du § 3 (Konsole et la plupart des terminaux VTE
proposent un fractionnement horizontal/vertical intégré, sans paquet
supplémentaire) : 3 à 4 volets côte à côte, à ~65–100 colonnes chacun,
correspond à un usage CLI bien plus proche de l'habitude qu'un panneau
unique. Aucune préférence de police n'est établie ni choisie dans ce
livrable — lecture seule, le calcul ci-dessus n'est qu'une méthode
applicable une fois un choix fait par l'opérateur.

## 5. Ce que la configuration devra porter, et où (D1/D4)

### 5.1 — Fichiers identifiés

- **Règle de placement KWin** : `~/.config/kwinrulesrc`. Format INI,
  `[General]\nrules=<liste d'ids>` puis un groupe `[<id>]` par règle
  (`wmclass=`, `screen=<index>`, `screenrule=<mode>`, etc. — schéma
  établi au § 1.1). **Actuellement vide sur ce poste** :
  ```
  $ cat ~/.config/kwinrulesrc
  [General]
  rules=
  ```
- **Autostart** : `~/.config/autostart/<nom>.desktop` (§ 2.1). N'existe
  pas encore (`~/.config/autostart/` absent).

### 5.2 — Versionabilité sous D1/D4 — un exemple concret de mélange trouvé sur ce poste

**`kwinrulesrc`** : un groupe de règle, une fois écrit, ne contient ni
secret ni identifiant réseau — seulement des critères de fenêtre
(`wmclass`) et l'index d'écran (§ 1, un entier, pas un UUID ni un nom de
machine). Versionable sous D4 sans réserve. Sous D1 : une valeur
`screen=N` figée dans le dépôt encoderait un fait propre à cette
machine à un instant donné (§ 1.3, stabilité non garantie) — un rôle
Ansible qui écrirait cette clé devrait le faire en conscience de cette
fragilité, pas la traiter comme une constante portable.

**`~/.config/autostart/*.desktop`** : fichier `.desktop` standard,
`Exec=`/`Name=`/`X-KDE-autostart-phase=`. Aucun contenu propre à cette
machine n'est requis (`Exec=konsole` ou équivalent, pas de chemin
`$HOME` en dur nécessaire). Versionable sans réserve sous D1 et D4.

**Mélange trouvé, en marge de la question posée mais qui illustre
exactement l'avertissement de la demande** : `~/.config/kwinrc` (réglages
KWin généraux, **fichier distinct** de `kwinrulesrc`) contient, lu tel
quel sur ce poste :
```ini
[Desktops]
Id_1=c6ae8d4e-04b9-487d-95b9-ba1df17864be

[Tiling][c6ae8d4e-04b9-487d-95b9-ba1df17864be][c5c4f802-3ee8-459a-8724-bd1810d84686]
tiles={"layoutDirection":"horizontal","tiles":[{"width":0.25},{"width":0.5},{"width":0.25}]}

[Xwayland]
Scale=1.5
```
La section `[Tiling]` est **littéralement indexée par l'UUID du bureau
virtuel et l'UUID de la sortie** (`c5c4f802-...` = `eDP-1`, confirmé en
comparant à `kscreen-doctor -o`) — un réglage de disposition en tuiles
propre à *cette instance* de bureau et *cette* sortie précise, pas
portable tel quel sur une autre machine ni même après une réinitialisation
du profil KScreen. `[Xwayland] Scale=1.5` en revanche est un réglage
générique, portable. **Copier `kwinrc` dans son ensemble dans ce dépôt
mélangerait les deux** — exactement l'avertissement de la demande,
démontré sur un fichier réel de ce poste, pas hypothétique. Toute
écriture future dans `kwinrc` par ce dépôt devrait cibler une clé
précise (`kwriteconfig6 --file kwinrc --group Xwayland --key Scale`),
jamais copier le fichier.

### 5.3 — Cohérence avec le reste du dépôt

Les rôles existants (`gpu_mux`, `gpu_cdi`) n'écrivent jamais de fichier
de configuration en le copiant tel quel — ils ciblent des clés
précises, avec garde et vérification. La même discipline s'appliquerait
ici : un futur rôle `desktop` écrirait des clés nommées dans
`kwinrulesrc`/un fichier `.desktop` d'autostart généré, jamais une copie
brute d'un export de configuration Plasma.

## Validation — BUR-0 (lecture seule, 2026-08-04/05)

**Commandes exécutées, toutes non modifiantes** :

| # | Commande | Nature | Sortie/code |
|---|---|---|---|
| 1 | `kscreen-doctor -o` | lecture d'état d'affichage | succès |
| 2 | `ls`/`cat ~/.config/kwinrulesrc` | lecture fichier utilisateur | succès (fichier vide de contenu utile) |
| 3 | `rpm -qa \| grep -i kwin` | lecture base rpm | succès |
| 4 | `rpm -ql kwin-common kwin` (filtré) | liste de fichiers d'un paquet installé | succès |
| 5 | `man -w kwinrulesrc` / `man -w kwinoutputconfig` | recherche de page de manuel | absence confirmée (code non nul attendu, aucune page) |
| 6 | Lecture externe : `raw.githubusercontent.com/KDE/kwin/v6.7.3/src/rulesettings.kcfg` et `rules.cpp` | requête HTTP GET, code source amont, épinglé à la version exacte installée | succès |
| 7 | `systemctl --user list-units 'plasma-*'`, `show`, `cat` (×8 unités) | lecture d'état systemd utilisateur | succès |
| 8 | `find` / `ls ~/.config/autostart` `/etc/xdg/autostart` | lecture répertoires | succès (absence confirmée pour le premier) |
| 9 | `grep X-KDE-autostart-* /etc/xdg/autostart/*.desktop` | lecture fichiers système | succès |
| 10 | `journalctl -k -b 0/-1/-2 -g ...` | lecture journal noyau, boots déjà existants | succès |
| 11 | `journalctl --list-boots` | lecture journal | succès |
| 12 | `dnf list --available <candidats>` **sans garde** | requête de métadonnées | **incident signalé § 3** — a déclenché l'invite Terra, vérifié après coup sans effet |
| 13 | `dnf --repo=terra --assumeno list` | idem, avec garde | succès, aucune correspondance |
| 14 | `dnf --disablerepo=terra repoquery --requires/--info/install --assumeno` (répété par paquet) | requêtes de métadonnées et simulations, aucune installation | succès |
| 15 | `rpm -q gpg-pubkey`, `find ... -newer` | vérification a posteriori de l'incident n°12 | succès, aucune clé récente |
| 16 | `cat ~/.config/kwinrc`, `ls -la` sur plusieurs fichiers `~/.config/*rc` | lecture | succès |
| 17 | `fc-match monospace` | lecture de configuration fontconfig | succès |

**Actions privilégiées : aucune.** Toutes les commandes ci-dessus
s'exécutent sans `sudo` — confirmé par leur nature (lectures locales
déjà lisibles par l'utilisateur, requêtes `dnf` en mode métadonnées,
lectures réseau anonymes).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** — un seul
marqueur actionnable dans ce document (§ 1.3, stabilité de l'ordre
`workspace()->outputs()` de KWin) ; le jeton n'est écrit nulle part
ailleurs dans ce fichier, y compris en prose (règle `CLAUDE.md` du
livrable précédent appliquée dès l'écriture, pas relue après coup).

**Confirmations finales** : aucun paquet installé (toutes les
installations simulées via `--assumeno`, jamais confirmées) ; aucune
configuration Plasma modifiée ; aucune règle KWin créée
(`kwinrulesrc` relu, inchangé) ; aucun autostart créé ; aucun mode
d'affichage changé (la contradiction relevée en tête de document est
**constatée**, pas **provoquée** par cette série) ; aucun fichier écrit
hors de ce dépôt.

## 6. BUR-1 — déploiement effectif (2026-08-06)

Choix de l'opérateur (§ 3) : **kitty**. Rôle Ansible :
[`roles/desktop/`](../roles/desktop/), playbook `desktop.yml`, détails
d'exécution dans `roles/desktop/README.md` — cette section documente
ce que le rôle a réellement produit et prouvé, pas son fonctionnement
interne (déjà commenté dans le rôle lui-même).

### 6.1 — Identifiant d'application réel (`app_id`), mesuré (point 2)

Fenêtre kitty réelle lancée, identifiée par introspection D-Bus KWin
(`org.kde.krunner1.Match` pour trouver l'UUID, filtré par PID pour
n'identifier que la fenêtre de test — jamais une fenêtre de
l'opérateur), puis interrogée directement :
```
$ busctl --user call org.kde.KWin /KWin org.kde.KWin getWindowInfo s "{5eedf1d9-3114-4305-ba2d-b3bd36317071}"
a{sv} 29 ... "resourceClass" s "kitty" "resourceName" s "kitty" ...
    "pid" i 88286 ... "x" d 0 "y" d 1067 "width" d 2560 "height" d 734
```
`resourceClass="kitty"` — confirmé, pas déduit du nom du paquet ni du
binaire ; cohérent avec `kitty --help` (« `--class, --app-id=[kitty]`
... valeur par défaut déjà "kitty" », documentation externe déjà citée
en BUR-0). **Champ exact dans `kwinrulesrc`** portant ce critère :
`wmclass` (String), avec son mode de correspondance `wmclassmatch`
(Int) — schéma source `rulesettings.kcfg`/`rules.h`, KWin v6.7.3, déjà
sourcé au § 1.1. Valeur retenue : `wmclassmatch=1` (`ExactMatch`) — pas
`SubstringMatch`, pour ne pas matcher accidentellement une autre
application dont le nom contiendrait « kitty ».

### 6.2 — Mécanisme de placement écarté : index d'écran (`screen`)

Première tentative, conforme à l'hypothèse de BUR-0 (§ 1) : règle
`screen=<index>` + `screenrule=Force`, combinée à
`maximizehoriz`/`maximizevert`. **Testé de façon répétée le
2026-08-06 (fenêtres de test isolées par PID, géométrie relevée par
`getWindowInfo`) et abandonné** : la valeur mesurée pour un même index
configuré variait selon `org.kde.KWin.activeOutputName()` — la sortie
« active » (déterminée par le focus/l'interaction en cours, pas par
l'index de la règle) déterminait le placement réel, y compris avec
`placement=PlacementMaximizing` en `Force`. Le détail du raisonnement
et le point exact où la confusion a été identifiée sont conservés dans
`roles/desktop/defaults/main.yml` (commentaire « ÉTAIT » sur
`desktop_kitty_placerule`) — pas dupliqués ici (`CLAUDE.md` § une
règle vit à un seul endroit ; ce principe s'applique par extension à
un fait technique établi une fois, cité par renvoi plutôt que recopié).

### 6.3 — Mécanisme retenu : `position`+`size`, et un second confondant trouvé puis écarté

**Mécanisme retenu** : `position` (Point) et `size` (Size), réglées
sur la géométrie logique de `DP-3` **mesurée en direct** à chaque
exécution du rôle (`kscreen-doctor -o`, jamais figée en dur — variable
`desktop_target_output`, § machine-facts). Ce mécanisme contourne
l'algorithme de placement plutôt que de tenter de l'influencer par un
index.

**Second confondant trouvé pendant la vérification de ce mécanisme,
avant d'affirmer quoi que ce soit** : une tentative de « neutraliser »
la règle en désaccordant son critère de correspondance
(`-e desktop_kitty_wmclass=bogus-nonexistent-app`, censée empêcher la
règle de matcher la fenêtre de test) a échoué à démontrer quoi que ce
soit — la fenêtre de test s'est quand même ouverte exactement à la
géométrie de `DP-3`. Vérifié que ce n'était pas un résidu de fenêtre
précédente (`pgrep -a kitty` : aucun processus), puis reproduit avec
un terminal **totalement étranger à toute règle de ce dépôt**
(`konsole`, jamais mentionné dans `kwinrulesrc`) :
```
$ busctl --user call org.kde.KWin /KWin org.kde.KWin getWindowInfo s "<uuid konsole>"
... "resourceClass" s "org.kde.konsole" ... "x" d 0 "y" d 1067 "width" d 2560 "height" d 733.333
```
Même géométrie, sans aucune règle. Cause : `org.kde.KWin.activeOutputName()`
valait `DP-3` au moment de ces essais — le placement par défaut de
KWin (aucune règle ne s'applique) semble suivre la sortie active,
comme le faisait déjà `screen`/`Force` avant son abandon (§ 6.2), pour
une fenêtre **sans aucune règle cette fois**. `@VERIF : mécanisme exact
du placement par défaut de KWin en l'absence de toute règle
applicable (politique globale, kwinrc [Windows] Placement relevée vide
sur ce poste — donc valeur compilée par défaut, non lue dans le code
source à ce stade) — non lu dans le code source KWin, seulement observé
empiriquement à deux reprises (konsole, kitty à critère désaccordé).`
Ce point n'affecte cependant pas la validité du mécanisme retenu (voir
la démonstration non confondue au § 6.8) : il explique seulement
**pourquoi le test de neutralisation par critère désaccordé ne prouve
rien** — le défaut coïncide avec la cible, masquant toute différence.
Erreur de méthode réelle, documentée plutôt qu'effacée
(`CLAUDE.md` § marquer l'historique).

### 6.4 — Politique de règle : `Apply initially`, pas `Force` (point 3)

Choix demandé explicitement par le point 3 : « appliquer initialement »
(l'opérateur peut redéplacer la fenêtre ensuite) contre « forcer »
(l'en empêche). Les deux valeurs ont été **mesurées fiables** à la
création de la fenêtre de test (géométrie identique dans les deux cas,
`0,1067 2560x734`) — le critère de choix n'est donc pas la fiabilité,
mais l'effet réel de chaque politique, établi par une source externe
autoritaire, pas par supposition : les chaînes descriptives de
l'interface KWin elle-même (`src/kcms/rules/optionsmodel.cpp`, tag
`v6.7.3`, source externe nommée) :
- `Force` : *"The window property will be always forced to the given
  value."*
- `Apply initially` : *"The window property will be only set to the
  given value after the window is created. No further changes will be
  affected."*

`wmclass=kitty` en `ExactMatch` matche **génériquement toute fenêtre
kitty**, pas seulement celle de démarrage. `Force` aurait donc empêché
l'opérateur de déplacer ou redimensionner **n'importe quelle** fenêtre
kitty ouverte ensuite, y compris pour un usage sans rapport avec le
ScreenPad Plus. `Apply initially` positionne chaque nouvelle fenêtre
kitty sur `DP-3` à sa création — l'effet demandé — sans jamais reprendre
la main par la suite. Retenu : `positionrule=3`, `sizerule=3`
(`Rules::Apply`).

### 6.5 — Rechargement de KWin sans redémarrage de session

`org.kde.KWin.reconfigure()` (méthode D-Bus, `/KWin`,
`org.kde.KWin`) — sans réponse (fire-and-forget). Un défaut réel a été
trouvé pendant les tests : une vérification lancée immédiatement après
l'appel pouvait encore lire l'ancienne règle. Corrigé par un second
handler (`sleep 1`), chaîné via `listen:` sur le même topic —
`roles/desktop/handlers/main.yml`. Aucune alternative nécessitant un
redémarrage de session n'a été nécessaire ni utilisée.

### 6.6 — Colonnes/lignes mesurées, comparées à la table théorique de BUR-0 (point 1)

`~/.config/kitty/kitty.conf` déployé par le rôle (`font_family Noto
Sans Mono`, `font_size 12.0`, `scrollback_lines 10000` — chaque option
porte son motif en commentaire dans le fichier lui-même, pas dupliqué
ici). Mesure réelle, pas seulement calculée, dans une fenêtre kitty
dimensionnée par la règle déployée :
```
$ kitty sh -c "stty size > <fichier>; sleep 1"
32 274
```
**32 lignes, 274 colonnes**, à comparer à la ligne « 12 pt » de la
table théorique de BUR-0 (§ 4) : **≈267 colonnes, ≈37 lignes**. Écart :
+7 colonnes (+2,6 %), **-5 lignes (-13,5 %)**. La table théorique de
BUR-0 se qualifiait déjà elle-même d'approximation (largeur de cellule
≈ 0,6 × taille en pixels à 96 ppp, hauteur de ligne ≈ un multiple
générique), « applicable une fois un choix fait par l'opérateur » —
pas une prédiction exacte. L'écart mesuré est cohérent avec cette
réserve : la cellule réelle de Noto Sans Mono à 12 pt est un peu plus
étroite que l'heuristique (2560⁄274 ≈ 9,3 px mesurés contre 9,6 px
supposés) et sa hauteur de ligne un peu plus grande (734⁄32 ≈ 22,9 px
mesurés contre 20 px supposés) — pas une anomalie, la confirmation que
l'approximation générique ne remplace pas une mesure avec la police
réellement choisie.

### 6.7 — Autostart (point 4)

`roles/desktop/templates/kitty-screenpad.desktop.j2` déployé dans
`~/.config/autostart/kitty-screenpad.desktop` — `Exec=kitty`,
`TryExec=kitty`, aucun chemin absolu propre à cette machine.

**Leviers `X-KDE-autostart-phase`/`X-KDE-autostart-after` : volontairement
non utilisés.** Motif de l'abstention (point 4 : « utiliser seulement
si justifiable ») : § 2.4 établit déjà, par lecture du graphe de
dépendances systemd réel, que les applications autostart démarrent
après que `kwin_wayland` (donc son état de sortie et sa lecture de
`kwinrulesrc`) est déjà en place — la seule dépendance pertinente pour
qu'une règle de placement s'applique correctement est déjà satisfaite
par l'ordre par défaut. Aucun besoin d'ordre supplémentaire identifié
vis-à-vis des autres applications autostart de ce poste (contrairement
à `vboxclient.desktop:X-KDE-autostart-after=panel`, qui a un besoin
propre) — donc pas de levier posé par réflexe.

**Ne peut être prouvé qu'à la prochaine ouverture de session** — pas
testé ici (`CLAUDE.md` : ne jamais déconnecter la session sans
demander). Commande de vérification préparée, à exécuter après une
prochaine connexion normale :
```
$ systemctl --user status app-kitty\\x2dscreenpad@autostart.service
$ busctl --user call org.kde.KWin /KWin org.kde.KWin getWindowInfo s "<uuid à retrouver via Match 'kitty'>"
# attendu : "x" d 0 "y" d 1067 "width" d 2560 "height" d 734 (valeurs à
# comparer à `kscreen-doctor -o` relevé au moment du test, pas figées)
```
**[PARTIELLEMENT VÉRIFIÉ le 2026-08-09, § 8.2–8.3/8.7]** Le déclenchement
réel de l'entrée autostart à une ouverture de session authentique est
désormais établi — trois démarrages distincts où
`app-kitty\x2dscreenpad@autostart.service` démarre et positionne des
fenêtres, journal à l'appui. Reste ouvert, faute d'avoir jamais pu
exécuter la commande `busctl getWindowInfo` ci-dessus sur une fenêtre
autostart réellement vivante (la seule fenêtre d'observation, démarrage
`-3`, s'est terminée par la régression documentée § 8 avant toute
lecture) : `@VERIF : géométrie D-Bus (getWindowInfo) d'une fenêtre
autostart à une ouverture de session authentique — commande ci-dessus
toujours prête à l'emploi, jamais exécutée avec succès à ce jour.`

### 6.8 — Démonstrations (point 5), géométries enregistrées

**Nominal** — rôle exécuté sans dérogation, fenêtre de test mesurée :
```
x=0.0, y=1067.0, 2560.0x734.0   (cible DP-3 mesurée : 0,1067 2560x734)
```
Assertion du rôle : succès (`Fenêtre de test mesurée sur DP-3 ...
règle position+size confirmée efficace par la mesure, pas supposée.`).
**Insuffisant seul** — § 6.3 montre que le placement par défaut (sans
aucune règle) coïncide actuellement avec cette même géométrie, parce
que `activeOutputName` vaut `DP-3`. Ce nominal ne discrimine donc pas,
à lui seul, « la règle fonctionne » de « la règle est sans effet et la
coïncidence masque tout ».

**Démonstration discriminante** — la règle est temporairement reciblée
sur la géométrie d'`eDP-1` (`315,0 1707x1067`, mesurée en direct,
`kwriteconfig6` hors rôle) **pendant que `activeOutputName` reste
`DP-3`** (relevé juste avant l'essai, variable de contrôle tenue
constante) :
```
$ busctl --user call org.kde.KWin /KWin org.kde.KWin activeOutputName
s "DP-3"
$ # fenêtre de test kitty lancée, règle ciblant eDP-1
"height" d 1067.33 "width" d 1707.33 "x" d 315 "y" d 0
```
La fenêtre suit la **règle** (eDP-1), pas la sortie active coïncidente
(DP-3) — preuve propre, non confondue, que le mécanisme `position`+
`size` détermine réellement le placement. C'est cette démonstration,
et non la neutralisation par critère désaccordé (§ 6.3, écartée comme
non probante), qui répond au point 5 (« même commande, règle
[donnant une cible différente], doit ouvrir ailleurs — si le résultat
est identique dans les deux cas, la règle ne fait rien »). État
restauré immédiatement après (règle reciblée sur `DP-3`, rôle rejoué,
assertion de nouveau au succès).

**Idempotence** : deuxième exécution du rôle sans dérogation,
`changed=0` (`PLAY RECAP ... changed=0 ... failed=0`).

**Marqueur § 1.3 : fermé par cette double preuve** — pas seulement
requalifié en théorie (§ 1.3 ci-dessus) mais démontré en pratique : le
mécanisme retenu ne dépend d'aucun index de sortie, et son effet a été
isolé de tout confondant connu (sortie active, coïncidence de
géométrie par défaut).

### 6.9 — Fait propre à la machine, exposé et affirmé (point 6)

Le pivot du § 6.2/6.3 élimine l'index d'écran comme fait à figer — il
n'est plus utilisé. Le fait propre à cette machine qui subsiste est le
**nom du connecteur** de la sortie cible, jusqu'ici en dur dans le
script de mesure. Corrigé : `desktop_target_output: DP-3`
(`roles/desktop/defaults/main.yml`, motif en commentaire), interpolé
dans la tâche de mesure (`tasks/main.yml`). **Affirmé, pas supposé** :
la tâche de mesure échoue bruyamment (`failed_when` sur stdout vide)
si `desktop_target_output` ne correspond à aucune sortie connue de
`kscreen-doctor -o` — vérifié dans les deux sens (`CLAUDE.md` § toute
garde se démontre dans les deux sens) :
```
$ ansible-playbook -e desktop_target_output=sortie-inexistante roles/desktop/desktop.yml
... fatal: [localhost]: FAILED! => {... "failed_when_result": true, "stdout": ""}
$ ansible-playbook roles/desktop/desktop.yml   # sans dérogation : succès, DP-3 mesurée
```
Un rebuild de cette machine rejouerait ce rôle avec cette même valeur
par défaut (D1) ; une machine dont le nommage de connecteur diffère
ferait échouer la mesure au lieu de placer silencieusement la règle
sur une géométrie jamais vérifiée.

### 6.10 — Écart assumé : « dimensionné », pas « maximisé »

**Consignation, pas correction** — signalé en revue, non corrigé ici
sur consigne explicite.

La demande initiale (BUR-1) disait **maximisé** — un *état* de fenêtre
(`maximizehoriz`/`maximizevert`, un booléen que KWin applique
lui-même, avec ses éventuelles réservations de bord d'écran gérées en
interne). Ce que le rôle produit est autre chose : une fenêtre
**dimensionnée** aux pixels exacts de la sortie cible (`position`+
`size`, § 6.3), pas mise dans l'état « maximisé » de KWin.

**Motif de l'écart** : le mécanisme d'état (`maximizehoriz`/
`maximizevert`) dépendait, dans la version du rôle envisagée au § 6.2,
du même index d'écran (`screen`/`screenrule`) établi inopérant —
`position`+`size` est la voie qui s'est vérifiée fonctionner (§ 6.8).
L'état de maximisation propre n'a pas été retesté séparément une fois
le pivot fait vers `position`+`size` — l'objectif (occuper visuellement
toute la surface de `DP-3`) était atteint, la distinction d'état n'a
pas été creusée plus loin à ce moment-là.

**Sans conséquence aujourd'hui** : aucun panneau Plasma n'occupe
`DP-3` sur ce poste (vérifié, aucune barre ni widget positionné sur
cette sortie). La géométrie retenue (`0,1067 2560x734`) couvre donc
toute la sortie, panneau compris — parce qu'il n'y a pas de panneau à
éviter.

**Symptôme à nommer, si ça change** : si un panneau Plasma est un jour
ajouté sur `DP-3`, une fenêtre **dimensionnée** aux pixels ne s'ajuste
pas à l'espace restant — elle passe dessous ou dessus du panneau,
contrairement à une fenêtre réellement à l'état **maximisé**, que KWin
aurait automatiquement contrainte à la zone utile. Reconnaissable par
lecture directe : la géométrie mesurée d'une fenêtre kitty coïnciderait
alors avec le rectangle plein de la sortie, pas avec le rectangle
utile (zone de travail moins panneaux) que rapporterait
`org.kde.KWin.activeOutputName`/l'API de géométrie de zone de travail.

**Aucune correction entreprise ici** — consigné pour qu'un livrable
futur qui ajouterait un panneau sur `DP-3` sache où regarder, pas pour
être résolu dans cette série.

## Validation — BUR-1 (déploiement, 2026-08-06)

**Actions privilégiées, exhaustives** :

| # | Commande | Cible | Motif |
|---|---|---|---|
| 1 | `dnf install -y kitty` (module `ansible.builtin.dnf`, `become: true`) | paquet `kitty` (dépôt `updates`) | seule installation demandée, choix de l'opérateur (BUR-0) |

Toutes les autres actions du rôle (`kwriteconfig6`/`kreadconfig6` sur
`~/.config/kwinrulesrc`, écriture dans `~/.config/kitty/`,
`~/.config/autostart/`, appels D-Bus `busctl --user`) s'exécutent sans
`sudo`, dans l'espace utilisateur.

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/desktop/desktop.yml   # succès
$ ansible-playbook --check roles/desktop/desktop.yml          # succès, aucune écriture
$ ansible-playbook roles/desktop/desktop.yml                  # succès, changed>0 (première écriture réelle)
$ ansible-playbook roles/desktop/desktop.yml                  # succès, changed=0 (idempotence confirmée)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/desktop/
Passed: 0 failure(s), 0 warning(s) — profil production, aucune dérogation noqa
```

**Confirmations finales** : aucun paquet installé hors `kitty` (et ses
dépendances résolues par `dnf`, jamais `terra`) ; aucun nouveau dépôt ;
`kwinrc` jamais touché (seul `kwinrulesrc` modifié) ; aucun changement
de mode d'affichage ni d'échelle ; aucune déconnexion de session ;
aucun redémarrage ; `sudoers`, `/etc/cdi/`, `gpu_mux_mode` non touchés
(hors périmètre de ce rôle, jamais référencés par lui).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : trois
marqueurs actionnables dans ce document — § 1.3 (correspondance
DRM/`outputs()`, non bloquante, expliquée), § 6.3 (mécanisme exact du
placement par défaut de KWin, non lu dans le code source), § 6.7
(effet réel de l'autostart, vérifiable seulement à la prochaine
session). Ce paragraphe-ci n'ajoute aucune occurrence du jeton
lui-même — périphrase seulement, conformément à la règle qui l'exige.

## 7. BUR-2 — disposition de démarrage : claude | htop / interpréteur (2026-08-09)

Trois volets dans la fenêtre kitty déjà placée sur `DP-3` par la règle
KWin (§ 6) : plein écran, séparation verticale (`claude` à gauche),
séparation horizontale à droite (`htop` en haut, un interpréteur de
commandes en bas).

### 7.1 — Résolution, § 1 de la demande

**Mécanisme de disposition, sourcé dans le code amont de kitty**
(`kitty/session.py`, `kitty/layout/splits.py`, version installée
`0.47.1` — pas supposé, la documentation packagée ne porte pas le
corps du chapitre « Sessions », seulement un renvoi ; lu à la source
directement, même discipline qu'à EDI-1/KAT-1) :

- **Format du fichier de session** : texte, une directive par ligne,
  commentaires `#` en tête de ligne, analysé par
  `kitty.session.parse_session` — directives reconnues :
  `new_tab`/`new_os_window`, `layout <nom>`, `launch <cmd>`, `focus`,
  `focus_tab`, `enabled_layouts`, `cd`, `title`, `os_window_size`,
  `os_window_class`/`os_window_name`/`os_window_title`,
  `os_window_state`, `resize_window`, `focus_matching_window`,
  `set_layout_state` — toute autre ligne lève `ValueError` (échec
  bruyant, pas un défaut silencieux).
- **Disposition permettant des séparations arbitraires** : `splits`
  (`kitty/layout/splits.py`, classe `Splits`) — seule disposition dont
  l'arbre interne (`Pair`, `one`/`two`, `horizontal`) admet des
  scissions verticales et horizontales composées librement. Les autres
  dispositions embarquées (`tall`, `fat`, `grid`, `stack`, `vertical`,
  `horizontal`) ont une structure fixe, pas de mécanisme de scission
  arbitraire. `enabled_layouts` vaut `*` par défaut
  (`kitty.conf.5`) — toutes les dispositions sont déjà disponibles sur
  ce poste, aucun réglage à ajouter dans `kitty.conf`.
- **Directive de séparation** : `launch --location=vsplit|hsplit
  --bias=N <cmd>` (`kitty/launch.py`, option `--location`, choix
  `vsplit`/`hsplit`/`split`/`default`/…, et `--bias`, pourcentage —
  « la NOUVELLE fenêtre prend N % de l'espace de la fenêtre qu'elle
  scinde, l'originale garde le reste »). Chaque `launch` d'un fichier
  de session est traité par la MÊME fonction `launch()` que celle
  invoquée à l'exécution interactive (`kitty/tabs.py`,
  `Tab._startup`) — le mécanisme de scission fonctionne à l'identique
  en session.
- **Focus explicite** : `focus` (sans argument) marque le DERNIER
  volet ajouté comme celui qui aura le focus au démarrage
  (`Session.focus()`) — placé juste après `launch claude`, avant les
  deux `launch` suivants, il fixe l'index cible sans être affecté par
  les volets ajoutés ensuite (`Tab._startup`, `session_tab.active_window_idx == i`).
  Sans directive `focus`, kitty retiendrait le PREMIER volet créé par
  défaut (`first_window_id`, `tabs.py`) — jamais laissé au hasard ici.
- **Raccrochage au démarrage** : deux mécanismes existent,
  `startup_session <chemin>` dans `kitty.conf` (s'applique à TOUTE
  invocation de kitty, y compris une future fenêtre ouverte
  manuellement — pas souhaité ici) et `--session <chemin>` en ligne de
  commande (s'applique à CETTE seule invocation — retenu, ajouté à
  `Exec=` du fichier `.desktop` d'autostart déjà déployé par ce rôle,
  § 6). « Les chemins relatifs sont résolus par rapport au répertoire
  de configuration de kitty » (`kitty.1` § `--session`) — le fichier de
  session est donc référencé par son seul nom
  (`Exec=kitty --session startup.session`), aucun chemin absolu propre
  à cette machine dans le `.desktop`.

**Interaction plein écran / règle KWin — tranchée par la mesure, pas
supposée (§ 1.2 de la demande, le point délicat).** La règle KWin (§ 6)
impose une géométrie (`position`+`size`, `Apply initially`) ; le plein
écran (`os_window_state fullscreen`, directive de session — sourcée
dans `kitty/utils.py::parse_os_window_state`, appliquée par
`boss.py::add_os_window` via `create_os_window(..., wstate, ...)`) est
un état de fenêtre Wayland distinct. **Lu d'abord, jusqu'à sa limite** :
la spécification `xdg-shell` (source externe, protocole officiel,
`xdg_toplevel::set_fullscreen`) dit exactement : « The output passed by
the request indicates the client's preference as to which display it
should be set fullscreen on. If this value is NULL, it's up to the
compositor to choose which display will be used. » — kitty n'a **aucun
mécanisme** pour indiquer une sortie cible (confirmé, `kitty.1` §
`--position` : « It never works on Wayland » — et aucune autre option
de session ou de config ne porte de nom de sortie pour le plein écran)
: la requête part donc **sans sortie précisée**, et la spécification
elle-même renvoie la décision au compositeur — **pas moyen de trancher
plus loin par la seule lecture.**

**Testé empiriquement, avec le confondant délibérément recréé** (même
méthode que BUR-1 § 6.2/6.3 pour `activeOutputName`) :
1. État de départ vérifié : `busctl --user call org.kde.KWin /KWin
   org.kde.KWin activeOutputName` → `DP-3` (pas un confondant si testé
   tel quel — l'écran actif coïnciderait déjà avec la cible).
2. **Divergence recréée par l'opérateur** (aucun mécanisme scriptable
   trouvé pour forcer `activeOutputName` — ni `workspace.activeScreen`,
   ni un déplacement de curseur par script KWin n'ont eu d'effet ; un
   clic réel sur la dalle principale a été demandé et obtenu) :
   `activeOutputName` → `eDP-1`, confirmé par relecture.
3. Fenêtre de test lancée avec `os_window_state fullscreen` (règle KWin
   déjà active, matche `wmclass=kitty`) :
   ```
   $ busctl --user call org.kde.KWin /KWin org.kde.KWin getWindowInfo s <uuid>
   ... "fullscreen" b true ... "x" d 0 "y" d 1067 "width" d 2560 "height" d 733.333 ...
   ```
   **Résultat : `DP-3`, pas `eDP-1`.** La fenêtre plein écran suit la
   sortie où la règle `Apply initially` l'a déjà placée au moment de sa
   création, pas `activeOutputName`. Cohérent avec la seconde moitié de
   la phrase du protocole (« the compositor is free to allow the
   surface to remain on the output it's currently at ») — confirmé sur
   ce KWin précis, pas seulement plausible par lecture du protocole.
4. Fenêtre de test fermée immédiatement après la mesure (PID isolé,
   jamais une fenêtre de l'opérateur).

**Conséquence retenue** : `os_window_state fullscreen` dans le fichier
de session, sans réserve — la règle KWin (§ 6) continue de déterminer
la sortie, le plein écran ne fait qu'ajouter l'état par-dessus une fois
la fenêtre déjà positionnée.

**§ 1.3 — `htop`** : déjà installé (`htop-3.4.1-3.fc44.x86_64`, dépôt
`fedora` — `dnf repoquery --installed --qf '%{name} %{from_repo}'
htop`). Aucune installation, aucune simulation nécessaire.

### 7.2 — Contrainte de hauteur, § 2 de la demande

`DP-3` : 2560×734 logiques (mesuré § 6, `kscreen-doctor -o`). Police
`kitty.conf` : `Noto Sans Mono`, taille `12.0` — inchangées par ce
livrable. Cellule mesurée en direct (fenêtre plein écran à un seul
volet) : 274 colonnes × 33 lignes sur 2560×734 px, soit environ
9,3×22,2 px logiques par cellule (calcul de vérification seulement —
la mesure ci-dessous porte sur la disposition réelle, pas sur ce
calcul).

**Mesuré directement dans la disposition réelle** (`kitten @ ls`,
colonne `lines`, sur la fenêtre de test du rôle) :

| Répartition (`--bias` du `hsplit`) | Lignes `htop` | Lignes interpréteur | Processus visibles dans `htop` (`kitten @ get-text`) |
|---|---|---|---|
| 50 (moitié-moitié) | 16 | 16 | **4** |
| 30 (retenue par défaut) | 23 | 9 | **12** |

**En-tête `htop` mesuré, pas déduit du seul règlage** (`~/.config/htop/htoprc`,
`header_layout=two_50_50`, `LeftCPUs4`/`RightCPUs4` — grille 4 cœurs
par ligne, 32 threads sur ce poste `nproc`) : quatre lignes de grille
CPU, une ligne `Mem`, une ligne `Swp`/`Load average`, une ligne
`Uptime`, une ligne vide, une ligne d'onglets (`[Main] [I/O]`), une
ligne d'en-tête de colonnes — **dix lignes fixes avant le premier
processus**, plus une ligne de pied de page (`F1Help…`). À 16 lignes
totales, il ne reste que 4 lignes utiles ; à 23, il en reste 12 —
confirmé par lecture réelle du contenu rendu, pas par ce calcul seul :

```
$ kitten @ get-text --match title:htop-pane   # bias=50
    0[...]   4[...]   8[...] 12[...]  16[...]  20[...]  24[...] 28[...]
    1[...]   5[...]   9[...] 13[...]  17[...]  21[...]  25[...] 29[...]
    2[...]   6[...]  10[...] 14[...]  18[...]  22[...]  26[...] 30[...]
    3[...]   7[...]  11[...] 15[...]  19[...]  23[...]  27[...] 31[...]
  Mem[...] Tasks: 149, ...
  Swp[...] Load average: ...
                         Uptime: ...
  [Main] [I/O]
    PID USER ... Command
  19673 mahieumi ... claude          # 1 seul processus affiché...
 465486 mahieumi ... /usr/bin/htop   # ...jusqu'à 4, puis le pied de page
```

**Répartition à 50/50 jugée trop juste** — confirmée par cette lecture,
pas seulement par le calcul. Ratio exposé en variable
(`desktop_kitty_htop_bias`, `roles/desktop/defaults/main.yml`), défaut
retenu **30** (23 lignes pour `htop`, 12 processus visibles, 9 lignes
pour l'interpréteur — suffisant pour une invite de commande) : le
chiffre est donné, pas la décision — l'opérateur ajuste la variable
s'il préfère un autre compromis.

### 7.3 — Implémentation, § 3 de la demande

Fichier versionné, jamais écrit à la main :
[`roles/desktop/templates/kitty-startup.session.j2`](../roles/desktop/templates/kitty-startup.session.j2),
déployé dans `~/.config/kitty/startup.session`. Variables
(`roles/desktop/defaults/main.yml`) : `desktop_kitty_session_cwd`
(répertoire de travail des trois volets, défaut `$HOME` — pas un
sous-répertoire de projet particulier, aucun chemin propre à cette
machine en dur) ; `desktop_kitty_claude_cmd`/`_htop_cmd`/`_shell_cmd`
(noms de commande, résolus par `PATH` à l'exécution) ;
`desktop_kitty_vsplit_bias` (gauche/droite, 50 par défaut) ;
`desktop_kitty_htop_bias` (haut/bas à droite, 30 par défaut, § 7.2).

**Gardes d'existence** (`roles/desktop/tasks/main.yml`) : les trois
commandes (`claude`, `htop`, l'interpréteur) sont vérifiées par
`command -v` avant tout déploiement — échec bruyant et explicite si
l'une manque, plutôt que le mode d'échec typique visé par la demande
(un volet dont la commande n'existe pas se ferme silencieusement au
démarrage, sans aucun message).

Focus par défaut : `claude` — choix explicite (directive `focus`
placée juste après son `launch`, § 7.1), pas laissé au hasard.

### 7.4 — Preuve, § 4 de la demande

Méthode établie dans ce dépôt (aucune capture visuelle) : le rôle
lance une fenêtre de test avec la session **réellement déployée**
(contrôle distant activé pour ce seul lancement, jamais dans
`kitty.conf`), relève sa géométrie et son état plein écran par
introspection D-Bus de KWin (même mécanisme que § 6, `getWindowInfo`)
et sa structure interne par le mécanisme propre de kitty
(`kitten @ ls`, JSON) — puis ferme la fenêtre de test (PID isolé,
jamais une fenêtre de l'opérateur).

**Structure réelle** (`kitten @ ls`, extrait) :
```
layout: splits
  title=claude cmdline=['/usr/bin/claude']
  title=htop   cmdline=['/usr/bin/htop']
  title=shell  cmdline=['/usr/bin/bash', '--posix']
```
(kitty résout chaque commande en chemin absolu par `PATH` avant de la
rapporter — constaté à l'essai, la garde `roles/desktop/tasks/main.yml`
compare donc sur le nom de base, pas sur l'argv brut ; kitty ajoute
`--posix` à `bash` de son propre chef, sans effet sur l'usage
interactif du volet.)

**Géométrie réelle** (`getWindowInfo`) :
```
"fullscreen" b true "x" d 0 "y" d 1067 "width" d 2560 "height" d 733.333
```
Sur `DP-3` (0,1067, 2560×734 mesuré § 6) — le plein écran n'a pas
déplacé la fenêtre, cohérent avec § 7.1.

**Échec forcé (garde § 3)** : commande substituée par une valeur
garantie absente —
```
$ ansible-playbook roles/desktop/desktop.yml -e desktop_kitty_htop_cmd=htop-inexistant-garanti
...
La commande « htop-inexistant-garanti » (volet de la session de
démarrage kitty) est absente du PATH — [...] Ce rôle s'arrête plutôt
que de déployer une session qui échouerait en silence.
PLAY RECAP ... changed=0 ...
```
Arrêt avant toute écriture — `changed=0`, confirmé.

**Second échec forcé (disposition neutralisée)** : `kitty --session
<fichier inexistant>` (test isolé, jamais le fichier réellement
déployé) — sortie de `kitten @ ls` :
```
layout: fat | windows: 2
  ~/dev/workstation-config           cmdline=['/bin/bash', '--posix']
  The startup session was invalid    cmdline=['/usr/bin/kitten', '__show_error__', ...]
```
**Signature complètement différente du cas nominal** — disposition
`fat` au lieu de `splits`, un volet de repli (l'interpréteur par
défaut) accompagné d'une fenêtre d'erreur explicite générée par kitty
lui-même (`kitten __show_error__`, pas un simple silence) plutôt que
les trois volets nommés attendus. La démonstration n'est pas vide de
sens — kitty signale l'échec plutôt que de le masquer, un mode d'échec
plus favorable que celui redouté au § 3 de la demande (qui portait sur
une commande *à l'intérieur* d'un volet, pas sur le fichier de session
lui-même).

**Le comportement à l'ouverture de session réelle ne peut être prouvé
qu'à la prochaine connexion** — pas déclenché ici (aucune déconnexion
sans demande explicite). Commande de vérification, à rejouer à la
prochaine ouverture de session :
```
busctl --user call org.kde.KWin /KWin org.kde.KWin getWindowInfo s <uuid-de-la-fenêtre-autostart>
# attendu : "fullscreen" b true, géométrie DP-3
kitten @ ls   # depuis un volet de cette même fenêtre — attendu : trois volets, disposition splits
```

## Validation — BUR-2 (2026-08-09)

**Résolution en trois points (§ 1 de la demande)** : mécanisme de
session sourcé dans le code amont de kitty (§ 7.1) ; interaction plein
écran/règle KWin **tranchée par un test empirique avec confondant
délibérément recréé**, pas supposée (§ 7.1) ; `htop` déjà installé,
aucune action (§ 7.1).

**Tableau des actions privilégiées** : aucune. Toutes les commandes de
ce livrable (lecture de code source packagé, lecture D-Bus, lancement
de fenêtres de test kitty, `ansible-playbook` sans `become` sur les
tâches ajoutées) s'exécutent sans `sudo` — seule l'installation de
`kitty` lui-même (déjà présente, `changed=0` à chaque exécution de ce
livrable) reste privilégiée dans ce rôle, inchangée depuis BUR-1.

**`ansible-lint --profile production roles/desktop/`** :
```
Passed: 0 failure(s), 0 warning(s) in 8 files processed of 10 encountered.
```
`--check` : `changed=0`. Exécution réelle : `changed=1` (première
écriture de la session et de l'entrée autostart mise à jour) puis
`changed=0` au passage suivant — idempotence confirmée.

**Confirmations finales** : `kwinrulesrc` **inchangé** (aucune tâche de
ce livrable n'y écrit — la règle existante suffisait, § 7.1 conclut que
le plein écran n'a pas besoin d'une géométrie différente) ; `sudoers`,
`/etc/cdi/`, `gpu_mux_mode` non touchés (hors périmètre de ce rôle,
jamais référencés) ; `kwinrc` non touché ; aucune installation de
paquet hors `kitty` (déjà présent) ; aucune déconnexion de session,
aucun redémarrage.

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : inchangé
par ce livrable — les trois marqueurs actionnables de BUR-1 restent
ouverts (§ 1.3, § 6.3, § 6.7), aucun nouveau marqueur ajouté ici
(l'interaction plein écran/règle KWin, seul point qui aurait pu en
porter un, a été tranchée par la mesure plutôt que laissée ouverte).

## 8. Régression — gel de `eDP-1` à l'ouverture de session (2026-08-09)

### 8.1 — Rapport initial

**Classe « observation rapportée par l'opérateur »** (`CLAUDE.md` § Sourcing
des faits) — datée du 2026-08-09, attribuée à l'opérateur de ce poste, non
recoupée par une trace journalisée directe pour la partie visuelle (aucune
capture d'écran, aucun outil de traçage du contenu affiché n'existe sur ce
poste) :

- le découpage en volets et le plein écran, déployés par BUR-2, **ont
  fonctionné sur `DP-3`** à une ouverture de session réelle (pas dans une
  session déjà établie, contrairement à la preuve de BUR-2 elle-même,
  § 7.4 et § 8.7 plus bas) ;
- **`eDP-1` (dalle principale) s'est figée** : image statique, curseur
  immobile ;
- **`DP-3` est resté vivant et interactif** — l'opérateur s'est débloqué en
  tapant dans la fenêtre `kitty` de `DP-3` ;
- aucun raccourci clavier n'a été testé pendant l'épisode ;
- renommage à la main de `~/.config/kitty/startup.session` en
  `startup.session.bak`, puis redémarrage — retour à la normale.

### 8.2 — Indices de démarrage réels, et démarrage retenu pour l'analyse

`journalctl --list-boots` (exécutée telle quelle, sans option de formatage
ajoutée ou retirée) donne, pour les démarrages pertinents :

```
 -5 56a2fe0f26aa4a0492f5f67548ea2e2d Fri 2026-08-07 12:19:48 CEST Sun 2026-08-09 10:37:43 CEST
 -4 97422a135fe14ee9be4b088c8f4fde9b Sun 2026-08-09 10:38:09 CEST Sun 2026-08-09 10:40:04 CEST
 -3 19f9256bdefb4b6db1b49a4aa44b1e12 Sun 2026-08-09 10:40:35 CEST Sun 2026-08-09 10:45:47 CEST
 -2 bb63b22b5f804cffb21dae565d7c92e6 Sun 2026-08-09 10:46:14 CEST Sun 2026-08-09 10:50:10 CEST
 -1 32c1b1de0f9a462d8508f94168f29f75 Sun 2026-08-09 10:50:36 CEST Sun 2026-08-09 11:07:50 CEST
  0 919b274361ec450fbddbd431805c28bb Sun 2026-08-09 12:13:52 CEST — courant
```

**Démarrage retenu pour l'analyse : `-3`, identifiant
`19f9256bdefb4b6db1b49a4aa44b1e12`, du 2026-08-09 10:40:35 au 2026-08-09
10:45:47 CEST — pas `-1`, conformément à l'indication reçue.** Établi par
recoupement de trois faits indépendants, tous lus sur la machine :

1. **`git log -1 --format='%cI' -- roles/desktop`** situe le commit BUR-2 à
   `2026-08-09T01:43:45+02:00` ; les tâches `ansible-legacy.copy`/`file`
   qui écrivent `startup.session` apparaissent dans le journal du
   démarrage `-5` entre `01:36:36` et `01:39:35` — ce démarrage est celui
   du **déploiement**, pas celui de la régression : `journalctl -b -5`
   montre `plasma-kwin_wayland.service` (le compositeur de la session
   réelle, pas celui du greeter) démarré une seule fois, à
   `2026-08-07T12:46:03`, **avant** l'écriture de BUR-2 — l'instance
   `kitty` de `ScreenPad Plus` déjà active depuis cette heure-là
   (`kitty-15445-0.scope`, § 8.3) n'a jamais été relancée dans ce
   démarrage, donc n'a jamais lu la nouvelle session. Le commit
   lui-même le documente : « le comportement à l'ouverture de session
   réelle reste à prouver à la prochaine connexion — aucune déconnexion
   déclenchée ». Le démarrage `-5` se termine par un redémarrage
   délibéré à `10:37:43`, cohérent avec ce constat (tester la session
   réelle exige une ouverture de session neuve).
2. **`stat --format='%z' ~/.config/kitty/startup.session.bak`** donne un
   `ctime` (heure de changement d'inode — un renommage la met à jour,
   contrairement au `mtime`, préservé par `mv` et resté à `01:36:36`,
   heure de l'écriture initiale par Ansible) de
   `2026-08-09 10:45:39.982484656 +0200` — strictement à l'intérieur de
   la fenêtre du démarrage `-3` (`10:40:35`–`10:45:47`), à 5 secondes de
   sa fin.
3. **`journalctl -b -3`** montre les trois volets de la session
   (`kitty-3740-0.scope`, `-1.scope`, `-2.scope`, lancés à `10:41:08`)
   restés vivants jusqu'à `10:45:44` — arrêtés **au même instant** que
   `app-kitty\x2dscreenpad@autostart.service` et que
   `systemd-logind[…]: The system will reboot now!`, **sans** passer par
   la boîte de dialogue graphique de déconnexion KDE (`LogoutPrompt`/
   `org.kde.Shutdown`, absente du journal de ce démarrage) — signature
   d'une commande tapée directement (`reboot`/`systemctl reboot`), pas
   d'un clic dans l'interface. Cohérent avec « débloqué en tapant dans
   kitty ».

Les trois faits se recoupent sur la même minute : renommage à `10:45:39`,
redémarrage tapé à `10:45:44`, dans une session ouverte à `10:41:08` — soit
une fenêtre de session vécue de 4 min 36 s pendant laquelle la disposition
BUR-2 tournait réellement pour la première fois.

**Écart non résolu, signalé plutôt que forcé** (`CLAUDE.md` § Avant
d'agir — contradiction à signaler, pas à absorber en silence) : la demande
énonçait « deux redémarrages ont eu lieu depuis » le démarrage fautif.
Entre la fin de `-3` et le démarrage courant (`0`), **trois** transitions
sont mesurées (`-3→-2`, `-2→-1`, `-1→0`), pas deux. Le démarrage `-4`
(10:38:09–10:40:04, **avant** `-3`) montre une anomalie distincte et plus
courte — la session s'ouvre, les trois volets se lancent à `10:38:47`,
mais `app-kitty\x2dscreenpad@autostart.service` s'arrête de lui-même 35 s
plus tard (`10:39:22`, avant tout redémarrage), et le redémarrage qui suit
passe cette fois par la boîte de dialogue graphique KDE (`LogoutPrompt`
à `10:39:54`) — signature différente de « taper dans kitty ». Si
l'opérateur comptait `-4` comme faisant partie du même épisode que `-3`
plutôt que comme un redémarrage distinct qui le précède, le compte
« deux » se rapprocherait sans se vérifier exactement ; aucune des deux
lectures ne rend le compte exact. Retenu ici : le fait mesurable (`ctime`
du fichier renommé, trois volets vivants jusqu'au redémarrage tapé) prime
sur le décompte rapporté — voir § 8.1.

### 8.3 — Chronologie reconstruite

| Démarrage | Fenêtre | Ce qui s'y passe |
|---|---|---|
| `-5` | 07/08 12:19 → 09/08 10:37 | Session ouverte le 07/08 à 12:46 (`kitty-15445-*`, ScreenPad Plus **sans** la disposition BUR-2, jamais relancée). BUR-2 développé et déployé 01:36–01:43 le 09/08, vérifié par des fenêtres de test isolées (fermées proprement à 01:40:06) — jamais via l'autostart réel. Redémarrage délibéré à 10:37:43 pour tester l'autostart à froid. |
| `-4` | 09/08 10:38 → 10:40 | Première ouverture de session avec BUR-2 actif. Trois volets lancés à 10:38:47 ; l'autostart s'arrête de lui-même à 10:39:22 (35 s), sans lien établi avec `-3`. Redémarrage via la boîte de dialogue KDE à 10:39:54–10:40:02. |
| **`-3`** | 09/08 10:40:35 → 10:45:47 | **Démarrage retenu.** Session ouverte 10:41:07, trois volets lancés 10:41:08, vivants 4 min 36 s. Régression rapportée par l'opérateur (§ 8.1) pendant cette fenêtre. Renommage `startup.session` → `.bak` à 10:45:39, redémarrage tapé à 10:45:44. |
| `-2` | 09/08 10:46 → 10:50 | Fichier de session déjà absent (renommé). Autostart en repli — deux fenêtres seulement (`kitty-3761-0/1.scope`), cohérent avec le second échec forcé déjà documenté § 7.4 (disposition `fat` + fenêtre d'erreur `kitten __show_error__`, pas trois volets nommés). |
| `-1` | 09/08 10:50 → 11:07 | Même repli, session plus longue (16 min 38 s). |
| `0` (courant) | 09/08 12:13 → — | Même repli. `kscreen-doctor -o` (relu pour cette section) : `DP-3` et `eDP-1` tous deux `enabled`/`connected`, aucune anomalie. |

### 8.4 — Phase A, journaux (démarrage `-3` sauf mention contraire)

**A.1 — Noyau, `amdgpu`/affichage.** `journalctl -b -3 -k` (intégral,
toutes priorités, mot-clé `amdgpu`/`drm`) : uniquement la séquence
d'initialisation normale du pilote à 10:40:38–10:40:43 (chargement
firmware, énumération des connecteurs, PSR — rien après). **Aucune**
occurrence de délai de basculement de page, d'échec de validation de
commit atomique, de réinitialisation GPU ou d'erreur de connecteur entre
l'ouverture de session (10:41:07) et l'arrêt (10:45:44) — fenêtre vide de
tout message noyau `amdgpu`/`drm`. Comparé au démarrage `-5` : la seule
anomalie noyau trouvée dans toute la série (`REG_WAIT timeout … dcn31_
program_compbuf_size`, avec trace complète `WARNING` et pile d'appel
`amdgpu_dm_commit_planes`/`amdgpu_dm_atomic_commit_tail`) se produit le
**2026-08-07 à 12:46–12:48**, à l'ouverture de la session longue de `-5`
— deux jours avant BUR-2, sans rapport temporel possible avec cette
régression. Écart entre les deux démarrages : c'est l'écart, néant contre
une anomalie connue mais datée ailleurs.

**A.2 — `kwin_wayland`.** Aucune ligne concernant la présentation directe
au balayage (« direct scanout »), les plans matériels, ou un changement de
mode de rendu au passage en plein écran — kwin, à la verbosité par
défaut, n'émet rien sur ce chemin (nécessiterait `KWIN_DRM_…`/
`QT_LOGGING_RULES` non positionnés sur ce poste). **L'hypothèse de
l'énoncé (fenêtre plein écran présentée hors composition, perturbant la
validation atomique de l'autre sortie de la même carte) reste une
hypothèse — ni confirmée ni infirmée par ce journal**, faute
d'instrumentation. Les deux seules lignes anormales de tout le
démarrage `-3` sont `kwin_wayland[2609]: Applying output configuration
failed!` et `PipeWire remote error: connection error`, toutes deux à
**10:45:44** — au moment même où `plasma-kwin_wayland.service` s'arrête
(déconnexion déclenchée par le redémarrage tapé), pas pendant la fenêtre
où le gel a été observé (10:41–10:45:39). Le plus probable, sans
certitude : un artefact de la perte du rôle maître DRM à l'extinction du
compositeur, pas un indice du mécanisme du gel lui-même.

**A.3 — État des sorties.** Aucune trace, dans tout le démarrage `-3`, de
désactivation de `eDP-1`, de changement de mode forcé, ou de gestion
d'énergie déclenchée (`org_kde_powerdevil[…]: Watching for DPMS state
changes unimplemented` est un message statique de capacité, émis une
fois à l'ouverture de session à 10:41:07, pas un événement). **Absence de
toute trace d'extinction confirme, sans la prouver positivement, la
distinction que l'opérateur décrit** : une sortie éteinte (DPMS off,
rétroéclairage coupé) aurait laissé une trace côté noyau ou
`systemd-backlight` ; une présentation simplement arrêtée (dernier flux
d'images non renouvelé, compositeur par ailleurs vivant) n'en laisse
aucune par construction — silence cohérent avec « image figée, curseur
immobile », pas avec un écran noir.

**A.4 — Autres causes.** `claude`, `htop` et l'interpréteur démarrent
ensemble à 10:41:08 (trois handles systemd distincts,
`kitty-3740-{0,1,2}.scope`). Charge et mémoire mesurées à l'arrêt des
volets (`systemd[…]: … Consumed …`) : `kitty-3740-0.scope` 1,096 s CPU /
381,2 Mo pic sur 4 min 36 s ; `kitty-3740-1.scope` 6,079 s CPU / 12,2 Mo
pic — négligeable. `systemd-oomd` actif tout le long du démarrage, aucun
déclenchement (`journalctl -b -3 -k` : aucune occurrence `oom`/`hung
task`/`soft lockup`/`blocked for more than`). **Éliminées** : saturation
CPU/mémoire, capture d'entrées par l'un des trois volets (le clavier
restait fonctionnel dans `kitty` sur `DP-3`, rapporté par l'opérateur),
OOM-kill.

### 8.5 — Conclusion de la phase A

**Les journaux ne tranchent pas le mécanisme exact du gel — ils éliminent
la plupart des causes énumérées par l'énoncé sans en confirmer
positivement aucune.** Aucune erreur noyau, aucune erreur `kwin_wayland`,
aucune extinction de sortie, aucune saturation ne coïncide avec la fenêtre
où la régression a été observée (10:41:08–10:45:39 dans le démarrage
`-3`) : le gel n'a laissé **aucune trace journalisée**, positive ou
négative, pendant qu'il se produisait — seul un silence total, ce qui est
cohérent avec « présentation arrêtée sans erreur » (A.3) mais ne le
prouve pas. L'hypothèse à tester de l'énoncé (A.2 : interaction entre le
plein écran hors composition d'une sortie et la validation atomique de
l'autre sortie de la même carte `amdgpu`) reste la plus plausible par
élimination des autres causes explicitement recherchées, mais **n'est
confirmée par aucune ligne de journal** — elle resterait à démontrer par
reproduction contrôlée (phase B) pour être autre chose qu'une hypothèse
par défaut.

**[MISE À JOUR § 8.10] Phase B entreprise après confirmation de
l'opérateur (TTY testé au préalable) — résultat négatif sur les trois
étapes, y compris la configuration exacte de la régression.** Ceci
affaiblit l'hypothèse ci-dessus prise isolément (§ 8.10) ; le facteur le
plus plausible restant est propre au contexte d'ouverture de session
(autostart, compositeur en cours d'initialisation), non testé par cette
phase et non testable sans reproduire le risque déjà rencontré.

### 8.6 — Divergence consignée : `startup.session.bak`

**État courant assumé et temporaire** : `~/.config/kitty/startup.session`
a été renommé à la main en `startup.session.bak` par l'opérateur
(§ 8.1–8.2), **hors du dépôt** — aucun commit ne le documente, c'est un
changement d'état de la machine seule. `roles/desktop/` (donc `site.yml`
qui l'inclut) **recréerait** `startup.session` à sa prochaine exécution,
restaurant l'état à l'origine de cette régression. **Ne pas rejouer ce
rôle avant résolution** (abandon du plein écran, ou confirmation par
reproduction contrôlée que le facteur réel est ailleurs).

Cas concret du **quatrième mode d'échec** du bilan de série (`CLAUDE.md`
§ Modes d'échec qui imitent le sourcing) : un rôle correct et idempotent
(`ansible-lint --profile production` : 0 défaut, `changed=0` confirmé en
second passage, § « Validation — BUR-2 » ci-dessus) dont la réexécution
restaurerait un état désormais connu comme défaillant — pas un défaut du
rôle lui-même, un changement de régime de ce qu'il produit.

### 8.7 — Leçon de méthode

BUR-2 avait prouvé structure, géométrie et dimensions par lancement
manuel d'une fenêtre de test **dans une session déjà établie** (§ 7.4) —
et l'avait dit explicitement : « le comportement à l'ouverture de session
réelle reste à prouver à la prochaine connexion ». Le jeton de
vérification associé (§ 6.7, posé lors de BUR-1) portait déjà ce constat
pour le placement seul. Ce qui manquait n'était pas le marqueur — il existait,
il était honnête — mais son **statut** : rien ne le rendait bloquant
avant l'intégration d'un comportement de démarrage automatique
supplémentaire (le plein écran de BUR-2, construit par-dessus). Une
fonctionnalité qui s'exécute à l'ouverture d'une session ne peut pas être
validée depuis une session déjà démarrée ; un tel marqueur doit être
**bloquant** avant toute intégration ultérieure, pas seulement honnête
sur son statut ouvert.

**Mise à jour partielle du marqueur § 6.7** (vérification effective, pas
nettoyage — `CLAUDE.md` § Avant d'agir) : le déclenchement réel de
l'autostart à une ouverture de session authentique est désormais établi
(§ 8.2–8.3, trois démarrages distincts où `app-kitty\x2dscreenpad@
autostart.service` démarre et positionne des fenêtres) — la partie
« jamais testé faute de déconnexion » du marqueur de BUR-1 ne tient plus.
Ce qui reste non vérifié, et donc **toujours ouvert** : aucune
introspection D-Bus (`getWindowInfo`) n'a été faite sur une fenêtre
autostart réellement vivante — la géométrie exacte à l'ouverture de
session authentique (par opposition à la fenêtre de test de § 6.8/7.4)
n'a jamais été lue, la session `-3` s'étant terminée par le gel avant
qu'une telle lecture soit entreprise — jeton de vérification reformulé en
ce sens § 6.7, pas dupliqué ici.

### 8.8 — Issue à garder ouverte

**[REQUALIFIÉE § 8.10]** Écrite avant la phase B, sur l'hypothèse que la
reproduction désignerait le plein écran comme facteur suffisant à lui
seul — ce que la phase B **infirme** (étapes 2 et 3, résultat négatif).
Conservée pour l'historique, plus la réponse par défaut : voir § 8.10
pour ce qui reste réellement ouvert (facteur de concurrence propre à
l'ouverture de session, non testé).

Si une reproduction future, par l'autostart réel, désignait malgré tout
le plein écran comme facteur (dans le contexte d'ouverture de session,
pas hors de lui), **renoncer à celui-ci et conserver la disposition en
volets avec la géométrie imposée par la règle KWin resterait une réponse
légitime** — le coût se limite à la barre de titre visible. Si le
diagnostic pointe plutôt vers un défaut du pilote `amdgpu`/`amdgpu_dm`
qui ne sera pas corrigé depuis ce dépôt, ne pas s'acharner : consigner et
s'arrêter.

### 8.9 — Phase B, préconditions vérifiées (2026-08-09)

Confirmation reçue de l'opérateur pour entamer la reproduction contrôlée
(§ 8.5). Préconditions établies avant tout lancement, comme exigé :

- **Basculement TTY testé par l'opérateur** (`Ctrl+Alt+F3`, retour
  `Ctrl+Alt+F2`) : fonctionne dans les deux sens, confirmé par
  l'opérateur avant toute provocation.
- **Commande de récupération, établie par lecture puis vérifiée
  réellement depuis le TTY** (pas présumée) :
  ```
  QT_QPA_PLATFORM=wayland WAYLAND_DISPLAY=wayland-0 kscreen-doctor output.eDP-1.disable output.eDP-1.enable
  ```
  Deux échecs intermédiaires, chacun résolu par lecture avant nouvel
  essai — pas par tâtonnement : `kscreen-doctor` (`ldd` : lié à
  `libQt6DBus`/`libdbus-1`, hypothèse initiale insuffisante) échoue en
  `qt.qpa.xcb: could not connect to display` depuis un TTY brut (aucun
  `DISPLAY`/`WAYLAND_DISPLAY` hérité de la session graphique) ;
  `QT_QPA_PLATFORM=offscreen` lève cette erreur mais casse la lecture
  réelle (« invalid config ») — `kscreen-doctor` est en réalité lié à
  `libQt6WaylandClient`/`libwayland-client`, greffon `KSC_KWayland.so`
  (trouvé sous `/usr/lib64/qt6/plugins/kf6/kscreen/`), qui a besoin
  d'une connexion Wayland réelle au compositeur — `offscreen` la
  supprime. Forme qui fonctionne, vérifiée par l'opérateur depuis le
  TTY (`kscreen-doctor -o`, lecture seule) : `QT_QPA_PLATFORM=wayland
  WAYLAND_DISPLAY=wayland-0`, en s'appuyant sur `/run/user/1000/
  wayland-0` (socket du compositeur, accessible à `mahieumi` depuis
  n'importe quel TTY de ce même utilisateur — `XDG_RUNTIME_DIR` est par
  utilisateur, pas par session, confirmé par `loginctl show-user
  mahieumi -p RuntimePath` → `/run/user/1000`).
- **État des sorties avant tout lancement** (`kscreen-doctor -o`, ce
  livrable, avant § 8.10) : `DP-3` et `eDP-1` tous deux
  `enabled`/`connected`, `eDP-1` actif à `2560x1600@60.00`, `DP-3` actif
  à `3840x1100@60.02` — identique à l'état courant relevé § « Journal
  des séries » de `docs/machine-facts.md`, aucun changement entre les
  deux relevés.
- **Fenêtres de test isolées, jamais le fichier de session réellement
  déployé** (même discipline que § 7.4/6.8) : fichiers temporaires sous
  le répertoire de travail (non versionnés, jamais dans le dépôt),
  chaque lancement avec un identifiant de contrôle à distance propre —
  jamais de `pkill` sur une instance existante de l'opérateur.

### 8.10 — Phase B, résultat (2026-08-09)

Protocole suivi dans l'ordre prescrit, un seul facteur à la fois,
confirmation de l'opérateur avant chaque étape, relevé de l'état des
sorties et du journal noyau/`kwin_wayland` après chacune. **Aucune des
trois étapes n'a reproduit le gel** — y compris l'étape 3, qui rejoue la
configuration exacte à l'origine de la régression du 09/08.

| Étape | Facteur | Fenêtre de test | Confirmation `eDP-1` | Journal après |
|---|---|---|---|---|
| 1 | Volets seuls, sans plein écran | `layout splits`, 3 volets, `fullscreen b false`, `DP-3` (`getWindowInfo` confirmé) | « eDP-1 reste vivant » | `amdgpu`/`drm`/`kwin_wayland` : rien |
| 2 | Plein écran seul, sans volets ni commandes | `kitty --start-as=fullscreen`, `fullscreen b true`, `DP-3` | « pas de freeze » | `amdgpu`/`drm`/`kwin_wayland` : rien |
| 3 | Les deux, **configuration identique** au fichier réellement figé (copie de `startup.session.bak`, `diff` confirmé) | `fullscreen b true`, `layout splits`, `DP-3` | « aucun problème » | `amdgpu`/`drm`/`kwin_wayland` : rien |

État des sorties (`kscreen-doctor -o`) relevé avant et après chaque
étape : identique dans les trois cas, aucun changement de géométrie ou
de mode. Fenêtres de test fermées par leur PID isolé à chaque fois,
jamais une fenêtre de l'opérateur.

**Conséquence sur l'hypothèse retenue § 8.5.** Le résultat négatif de
l'étape 3 est le fait le plus informatif de cette phase : **la
configuration exacte qui a figé `eDP-1` le 09/08 ne le reproduit pas**
lorsqu'elle est lancée à la main dans une session déjà stable et
établie. Ceci **affaiblit** l'hypothèse d'une interaction statique entre
l'état plein écran et la validation atomique de la sortie sœur — si
cette interaction suffisait à elle seule, l'étape 3 l'aurait déclenchée.
**Différence non testée, et qui reste la piste la plus plausible** :
dans le démarrage fautif (`-3`), la disposition s'est lancée **par
l'autostart, au tout début d'une ouverture de session** (`10:41:08`, une
seconde après le premier message de `org_kde_powerdevil` — § 8.4, A.3),
period où le compositeur et les autres services de session finissent de
s'initialiser ; ici, chaque fenêtre de test a été lancée à la main dans
une session **déjà stable depuis des heures**. Un facteur de
concurrence propre au démarrage de session (compositeur pas encore
totalement établi, plusieurs clients autostart simultanés) n'est ni
confirmé ni exclu par cette phase — seule une reproduction **par
l'autostart réel à une ouverture de session authentique** permettrait de
trancher, ce qui exigerait de redéployer `startup.session` et de courir
le même risque que celui déjà rencontré. **Non entrepris ici** — hors
périmètre de ce livrable sans nouvelle confirmation explicite.

**Correction de l'issue ouverte § 8.8** : renoncer au plein écran n'est
**plus soutenu par une preuve directe** comme réponse au problème — les
étapes 2 et 3 montrent le plein écran, seul et combiné aux volets,
fonctionner sans incident hors contexte d'ouverture de session. Rester
prudent : ceci ne prouve pas non plus que le plein écran est innocent en
contexte d'autostart réel, seulement que ce n'est pas un facteur suffisant
en dehors de ce contexte. `startup.session.bak` reste en l'état,
`roles/desktop/`/`site.yml` **toujours pas rejoués** — cette phase ne
lève pas la divergence consignée § 8.6, elle ne fait que retirer un
argument à une des deux résolutions possibles.

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — état d'affichage
  post-bascule (§ Affichage), inventaire GPU, D1/D4.
- [`docs/ansible-chain.md`](ansible-chain.md) — chaîne Ansible D3a/D3b,
  sans rapport direct mais même discipline de sourcing.
- [`docs/repositories.md`](repositories.md) — dossier Terra (D10),
  y compris le point 0 de BUR-1 (variante durable envisagée pour
  `repo_gpgcheck`, aucune trouvée).
- [`docs/completion.md`](completion.md) § 9 — même discipline de preuve
  sans capture visuelle (`journalctl`, ici `kitten @ ls`/`get-text`),
  même méthode de confondant délibérément recréé pour trancher une
  question de sortie/écran (KAT-1, `activeOutputName`).
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici,
  notamment la garde sur Terra et le jeton de vérification jamais nu.
