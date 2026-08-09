# roles/desktop

Rôle Ansible ciblant `localhost`. Installe `kitty` (choix de
l'opérateur, `docs/desktop.md` BUR-0), le configure sobrement, et
l'ouvre placé et dimensionné sur le ScreenPad Plus (`DP-3`) à
l'ouverture de session. Résolution complète, méthode de mesure et
démonstrations sont dans [`docs/desktop.md`](../../docs/desktop.md) —
ce README ne duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Installe `kitty` (dépôts `fedora`/`updates` uniquement — un seul
   paquet demandé, ses dépendances sont résolues par `dnf`).
2. Déploie `~/.config/kitty/kitty.conf` (police, taille, défilement
   arrière — rien de plus, `roles/desktop/files/kitty.conf`).
3. Relève en direct la géométrie logique de la sortie cible
   (`desktop_target_output`, `kscreen-doctor -o` — jamais figée en
   dur), et échoue bruyamment si cette sortie n'existe pas sur la
   machine en cours.
4. Écrit une règle dans `~/.config/kwinrulesrc`, **clé par clé**
   (`kwriteconfig6`, jamais une copie de fichier) : correspondance sur
   l'`app_id` réel de kitty (`wmclass=kitty`, `wmclassmatch=1` —
   `ExactMatch`), placement par `position`/`size` réglés sur la
   géométrie mesurée à l'étape précédente, tous deux en `Apply
   initially` (`...rule=3` — l'opérateur peut redéplacer ou
   redimensionner n'importe quelle fenêtre kitty après sa création ;
   `Force` a été écarté précisément parce qu'il l'en aurait empêché en
   permanence, pour toute fenêtre kitty, pas seulement celle de
   démarrage — `docs/desktop.md` § 6.4).
5. Demande à KWin de relire sa configuration par D-Bus
   (`org.kde.KWin.reconfigure`) — jamais un redémarrage de session.
6. **Vérifie par la mesure**, pas par confiance : lance une fenêtre
   `kitty` de test, relève sa géométrie réelle par introspection D-Bus
   (`org.kde.KWin.getWindowInfo`), échoue bruyamment si elle ne
   correspond pas à la géométrie mesurée à l'étape 3.
7. **Vérifie l'existence de `claude`, `htop` et l'interpréteur de
   commandes** (`command -v`, BUR-2) — échec bruyant si l'un manque.
8. Déploie deux dispositions de démarrage — `claude` à gauche, `htop`
   en haut à droite, un interpréteur en bas à droite dans les deux
   cas :
   - **nominale** (`~/.config/kitty/startup.session`, BUR-2) — le
     fichier ne porte plus l'état plein écran depuis BUR-4 (déplacé au
     script de temporisation, point 10, § motif ci-dessous) ;
   - **dégradée** (`~/.config/kitty/startup-degraded.session`, BUR-4) —
     mêmes volets, sans plein écran, avertissement imprimé dans le
     volet shell.
   Puis **revérifie par la mesure** : lance une fenêtre de test avec la
   session nominale et `--start-as=fullscreen` (la combinaison que le
   script applique en cas nominal), confirme l'état plein écran et la
   géométrie (`getWindowInfo`, comme à l'étape 6) et la structure
   interne — trois volets, disposition `splits`, bonnes commandes
   (`kitten @ ls`, mécanisme propre à kitty).
9. Installe `jq` (dépôts `fedora`/`updates`) et déploie le script de
   temporisation `~/.local/bin/kitty-startup-wait.sh` (BUR-4) — attend
   que `kscreen-doctor -j` (mode/échelle/priorité par sortie) soit
   stable sur plusieurs échantillons consécutifs, borné à un délai
   maximal, puis lance la session nominale en plein écran si la
   condition est satisfaite à temps, sinon la session dégradée. Motif
   complet : `docs/desktop.md` § 8 (gel de `eDP-1` constaté après BUR-2)
   et § 9 (ce mécanisme). Puis **revérifie par la mesure** : lance le
   script réellement déployé, confirme que la décision « nominal »
   survient quasi immédiatement (session déjà établie) et que la
   fenêtre qui en résulte est plein écran sur `DP-3`.
10. Déploie `~/.config/autostart/kitty-screenpad.desktop`
    (`Exec=kitty-startup-wait.sh`, résolu par `PATH` — aucun chemin
    absolu propre à cette machine).

## Ce que ce rôle ne fait jamais

- Il ne touche jamais `kwinrc` (réglages généraux KWin, distinct de
  `kwinrulesrc` — voir `docs/desktop.md` § 5 sur le mélange
  réglages/état trouvé dans ce fichier sur ce poste).
- Il ne modifie jamais `sudoers`, `/etc/cdi/`, `gpu_mux_mode`,
  `dgpu_disable`.
- Il ne change jamais de mode d'affichage ni d'échelle.
- **Il ne redémarre jamais la machine, ne déconnecte jamais la
  session.**
- Il n'installe aucun autre paquet que `kitty` et `jq` (BUR-4, requis
  par le script de temporisation pour lire `kscreen-doctor -j`).

## Mécanisme de placement — pas un index d'écran

Une première version de ce rôle visait `screen`/`screenrule=Force`
(index d'écran KWin). **Testé et abandonné** le 2026-08-06 : cette
propriété s'est révélée confondue avec `org.kde.KWin.activeOutputName`
(la sortie « active » l'emporte sur l'index configuré) — voir le motif
complet dans `defaults/main.yml` et `docs/desktop.md` § 6.2/6.3. Le
mécanisme retenu (`position`+`size`, mesurés en direct) contourne cette
confusion — démontré par une mesure qui isole le seul facteur qui
compte : la sortie active reste inchangée pendant qu'on change la cible
de la règle, et la fenêtre suit la règle, pas la sortie active.

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, en particulier `desktop_target_output` (nom du connecteur
DRM de la sortie cible, **propre à cette machine**, exposé en variable
plutôt qu'en dur pour qu'une machine reconstruite avec un nommage
différent fasse échouer la mesure au lieu de placer silencieusement la
règle sur une géométrie jamais vérifiée) et `desktop_kitty_wmclass`
(`app_id` réel, relevé par introspection D-Bus, pas déduit du nom du
paquet).

**Disposition de démarrage (BUR-2)** : `desktop_kitty_session_cwd`
(répertoire de travail des trois volets, défaut `$HOME`) ;
`desktop_kitty_claude_cmd`/`_htop_cmd`/`_shell_cmd` (noms de commande) ;
`desktop_kitty_vsplit_bias` (gauche/droite, défaut 50) ;
`desktop_kitty_htop_bias` (haut/bas dans la colonne de droite, défaut
30 — **mesuré**, pas deviné : à 50/50 l'en-tête de `htop` (32 threads
sur ce poste) ne laisse que 4 lignes de processus visibles sur `DP-3`,
à 30 il en laisse 12 — détail complet et méthode de mesure,
`docs/desktop.md` § 7.2).

**Temporisation du démarrage (BUR-4)** : `desktop_kitty_wait_interval_seconds`
(intervalle entre deux échantillons, défaut 0,2 s) ;
`desktop_kitty_wait_stable_samples` (nombre d'échantillons consécutifs
identiques exigé, défaut 2 — le plancher de latence du cas nominal vaut
par construction `(N-1) × intervalle`) ; `desktop_kitty_wait_max_seconds`
(délai maximal avant repli dégradé, défaut 5 s — approximation assumée,
aucun signal D-Bus propre trouvé pour « configuration de sorties
appliquée », `docs/desktop.md` § 9.1). Aucune de ces trois variables
n'est figée en dur dans le script — voir
`roles/desktop/templates/kitty-startup-wait.sh.j2`.

## Utilisation

```
ansible-playbook --syntax-check roles/desktop/desktop.yml
ansible-playbook --check roles/desktop/desktop.yml
ansible-playbook roles/desktop/desktop.yml                 # écriture réelle

# Démonstration d'échec forcé (garde sur le nom de sortie, § docs/desktop.md § 6.8) :
ansible-playbook -e desktop_target_output=sortie-inexistante roles/desktop/desktop.yml

# Démonstration d'échec forcé (garde sur les commandes de la disposition de démarrage, BUR-2) :
ansible-playbook -e desktop_kitty_htop_cmd=htop-inexistant-garanti roles/desktop/desktop.yml

# Démonstration du repli dégradé (condition d'attente rendue insatisfiable
# par variable, BUR-4, docs/desktop.md § 9.4) — déploie un script dont le
# critère de stabilité ne peut jamais être atteint dans le délai imparti :
ansible-playbook -e desktop_kitty_wait_stable_samples=1000000 roles/desktop/desktop.yml
# Revenir aux valeurs par défaut ensuite :
ansible-playbook roles/desktop/desktop.yml
```
