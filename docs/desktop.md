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

### 1.3 — Stabilité de l'index : établie partiellement, marquée pour le reste

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

## Validation

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

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — état d'affichage
  post-bascule (§ Affichage), inventaire GPU, D1/D4.
- [`docs/ansible-chain.md`](ansible-chain.md) — chaîne Ansible D3a/D3b,
  sans rapport direct mais même discipline de sourcing.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici,
  notamment la garde sur Terra et le jeton de vérification jamais nu.
