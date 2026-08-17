# roles/android_env

Rôle Ansible ciblant `localhost`. Troisième rôle d'une chaîne de build
Android CLI destinée au dépôt `glass-hud` (distinct, jamais référencé
au-delà de cette phrase) — pose `ANDROID_HOME` et le `PATH`
(`cmdline-tools/latest/bin`, `platform-tools`) pour les shells
interactifs, rien d'autre.

**N'écrit JAMAIS dans `~/.bashrc`, `~/.bash_profile`, `~/.profile`,
`/etc/profile.d/` ni `~/.config/environment.d/`.** La boucle qui charge
`~/.bashrc.d/*` existe déjà nativement dans le gabarit Fedora
(`/etc/skel/.bashrc`, jamais modifié par le pilote sur ce poste —
livrable 10, `docs/machine-facts.md` § Décisions, D28) : ce rôle dépose
un fichier dans un répertoire déjà chargé, il ne touche jamais à un
fichier qui appartient au pilote. Retour arrière : supprimer un fichier,
rien de plus.

Modèle : [`roles/android_sdk/`](../android_sdk/) (squelette identique :
assertion Fedora en première tâche, vérification par l'effet plutôt que
par le code de retour, `check_mode: false` sur les lectures pures).
Écarts assumés, motivés par la nature de ce rôle : `android_sdk`
télécharge une archive et n'a aucune garde sur un fichier du pilote ; ce
rôle ne télécharge rien, mais porte une garde spécifique et centrale —
le CONTENU réel de `~/.bashrc` doit porter la boucle native de
chargement de `~/.bashrc.d/`, faute de quoi le fichier déposé par ce
rôle ne serait jamais chargé et ce rôle rapporterait un succès sans
effet (§ Ce que ce rôle fait, point 3).

## Ce que ce rôle fait

1. Asserte que le poste est Fedora 44 — échoue bruyamment sinon.
2. **Vérifie par l'effet** que la racine du SDK et
   `cmdline-tools/latest/bin` sont réellement en place (`stat`) —
   dépendance réelle sur `roles/android_sdk/`, jamais supposée depuis
   l'ordre de `site.yml`.
3. **Garde centrale** : relève le contenu réel de `~/.bashrc`
   (`android_env_bashrc_path`) et échoue bruyamment si la sous-chaîne
   exacte de la boucle native (`for rc in ~/.bashrc.d/*; do`) n'y
   figure pas — recherche littérale (`grep -F`) sur le contenu, jamais
   une simple vérification d'existence du fichier.
4. Crée `~/.bashrc.d/` s'il n'existe pas déjà (mode `0755`).
5. Dépose `~/.bashrc.d/90-android-sdk.sh` (`ansible.builtin.template`,
   mode `0644`) — contenu **entièrement propriété de ce rôle** (aucun
   contenu du pilote n'y vit jamais), écrasé intégralement à chaque
   exécution, aucune fusion requise.
6. **Vérifie l'effet réel, pas seulement la présence des fichiers** :
   lance un shell interactif non-login dans un environnement assaini
   (`env -i HOME=... bash -ic '...'`) — c'est le mécanisme natif de
   démarrage de bash qui charge `~/.bashrc`, jamais un `source` manuel
   dans la tâche — et échoue bruyamment si `ANDROID_HOME`/`PATH` ne
   portent pas ce que ce rôle vient de déposer.

## Ce que ce rôle ne fait jamais

- Il n'écrit jamais dans `~/.bashrc`, `~/.bash_profile`, `~/.profile`,
  `/etc/profile.d/` ni `~/.config/environment.d/`.
- Il ne pose jamais `ANDROID_SDK_ROOT` — explicitement dépréciée au
  profit d'`ANDROID_HOME` (`developer.android.com/tools/variables`,
  D27, `docs/machine-facts.md` § Décisions).
- Il ne pose jamais `ANDROID_USER_HOME` — laissée à son défaut
  (`$HOME/.android/`, déjà présent sur ce poste, créé par
  `roles/android_sdk/` lors de son exécution).
- Il n'installe aucun paquet, aucun composant SDK (`platform-tools`,
  `platform`, `build-tools`, `adb` système `android-tools`) — voir
  `docs/machine-facts.md` § Décisions, D28, pour la révision de la voie
  `adb` (SDK seul, jamais `android-tools`).
- Il ne fusionne jamais le fichier qu'il dépose avec un contenu
  préexistant — propriété intégrale, écrasement à chaque exécution.

## Ce que la vérification d'effet couvre, et ce qu'elle ne couvre pas

**Couvre** : le chargement réel par un shell interactif non-login —
représentatif d'un terminal `kitty` ouvert par le pilote (`roles/desktop/`,
qui lance un bash interactif non-login par défaut, mesuré livrable 10).

**Ne couvre PAS** :
- Un shell de **connexion** (`~/.bash_profile`) — jamais exercé, aucun
  usage réel identifié sur ce poste qui en dépendrait (kitty n'en lance
  pas, livrable 10).
- Ce qu'un **lanceur graphique KDE** verrait (menu, KRunner, `.desktop`)
  — mécanisme distinct, `~/.config/environment.d/`, non posé par ce
  rôle (point ouvert, `docs/machine-facts.md` § Points ouverts, D28).
- Un `source ~/.bashrc` manuel dans un shell **déjà ouvert avant**
  l'exécution de ce rôle — ce shell ne verra l'effet qu'après un
  nouveau `source` ou un nouveau terminal ; aucune tâche Ansible ne peut
  agir sur un shell interactif que le pilote garde ouvert.

## Utilisation

```
ansible-playbook --syntax-check roles/android_env/android_env.yml
NO_COLOR=1 ~/.venvs/ansible-lint/bin/ansible-lint roles/android_env
ansible-playbook --check roles/android_env/android_env.yml
ansible-playbook roles/android_env/android_env.yml
```

**Ce que `--check` couvre réellement** : les trois assertions de
lecture seule (Fedora 44, dépendance SDK par l'effet, contenu de
`~/.bashrc`) — toutes les trois s'exécutent réellement même sous
`--check` (`check_mode: false` sur les tâches de lecture qui les
alimentent). Il ne couvre PAS la création de `~/.bashrc.d/`, le dépôt
du fichier, ni la vérification d'effet dans un shell réel — ces trois
dépendent d'une écriture (`when: not ansible_check_mode`) : un
`--check` vert sur ce rôle prouve donc davantage que sur
`roles/android_sdk/` (trois gardes de lecture exercées, pas deux), mais
ne prouve toujours rien sur l'effet réel du fichier déposé.

Démonstration d'échec forcé de la garde sur le contenu de `~/.bashrc`
(§ Ce que ce rôle fait, point 3), **sans jamais toucher au `~/.bashrc`
réel du pilote** — contre une copie hors dépôt, délibérément privée de
la boucle native :
```
sed '/^if \[ -d ~\/\.bashrc\.d \]; then$/,/^fi$/d; /^unset rc$/d' ~/.bashrc > /tmp/bashrc-sans-boucle.sh
ansible-playbook roles/android_env/android_env.yml \
  -e android_env_bashrc_path=/tmp/bashrc-sans-boucle.sh
```
Échoue avec le message exact de la garde (`grep rc=1`), **avant tout
dépôt** — `~/.bashrc.d/` reste dans l'état où il était avant cette
tentative (vérifié : absent avant, toujours absent après un échec sur
un poste qui n'a jamais joué ce rôle).

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, y compris la variable substituable pour la démonstration
d'échec forcé (`android_env_bashrc_path`, sans jamais changer le
`~/.bashrc` réel du pilote).

## Idempotence du `PATH` — patron repris, pas inventé

Le fichier déposé (`templates/90-android-sdk.sh.j2`) reprend
**littéralement** le patron déjà présent dans `~/.bashrc` (gabarit
Fedora) pour `$HOME/.local/bin` : une vérification par sous-chaîne
(`[[ "$PATH" =~ "..." ]]`) avant tout ajout, jamais un ajout
inconditionnel. Prouvé par mesure, pas seulement par lecture du
gabarit : sourcer `~/.bashrc` deux fois dans le même shell produit un
`PATH` strictement identique aux deux passages.
