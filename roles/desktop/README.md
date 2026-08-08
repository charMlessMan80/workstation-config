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
8. Déploie la disposition de démarrage
   (`~/.config/kitty/startup.session`, BUR-2) : plein écran, `claude` à
   gauche, `htop` en haut à droite, un interpréteur en bas à droite —
   puis **revérifie par la mesure** : lance une fenêtre de test avec
   cette session réellement déployée, confirme l'état plein écran et
   la géométrie (`getWindowInfo`, comme à l'étape 6) et la structure
   interne — trois volets, disposition `splits`, bonnes commandes
   (`kitten @ ls`, mécanisme propre à kitty).
9. Déploie `~/.config/autostart/kitty-screenpad.desktop`
   (`Exec=kitty --session startup.session`, aucun chemin absolu propre
   à cette machine).

## Ce que ce rôle ne fait jamais

- Il ne touche jamais `kwinrc` (réglages généraux KWin, distinct de
  `kwinrulesrc` — voir `docs/desktop.md` § 5 sur le mélange
  réglages/état trouvé dans ce fichier sur ce poste).
- Il ne modifie jamais `sudoers`, `/etc/cdi/`, `gpu_mux_mode`,
  `dgpu_disable`.
- Il ne change jamais de mode d'affichage ni d'échelle.
- **Il ne redémarre jamais la machine, ne déconnecte jamais la
  session.**
- Il n'installe aucun autre paquet que `kitty`.

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

## Utilisation

```
ansible-playbook --syntax-check roles/desktop/desktop.yml
ansible-playbook --check roles/desktop/desktop.yml
ansible-playbook roles/desktop/desktop.yml                 # écriture réelle

# Démonstration d'échec forcé (garde sur le nom de sortie, § docs/desktop.md § 6.8) :
ansible-playbook -e desktop_target_output=sortie-inexistante roles/desktop/desktop.yml

# Démonstration d'échec forcé (garde sur les commandes de la disposition de démarrage, BUR-2) :
ansible-playbook -e desktop_kitty_htop_cmd=htop-inexistant-garanti roles/desktop/desktop.yml
```
