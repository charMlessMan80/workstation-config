# Chemin de retour — bascule du MUX GPU (D2bis)

**Ce document ne bascule rien.** Il documente comment revenir à un système
utilisable si la bascule de `gpu_mux_mode` (décision D2bis, voir
[`docs/machine-facts.md`](machine-facts.md)) laisse la dalle principale
noire après redémarrage. Rien ici n'écrit dans `gpu_mux_mode`, ne démarre ni
n'active `supergfxd`, ni ne redémarre la machine — cette procédure se
prépare et se vérifie *avant* la bascule réelle, qui reste un acte manuel et
délibéré, hors du périmètre de ce livrable.

Toute commande citée ci-dessous a été exécutée dans une session en lecture
seule le 2026-08-04 sur ce poste, sauf mention contraire explicite.

## Gardes — à relire avant toute bascule

Deux gardes gouvernent toute écriture dans `gpu_mux_mode`. Leur texte qui
fait autorité — et l'historique de la correction apportée le 2026-08-05 à
la seconde — vit dans `CLAUDE.md` § Matériel spécifique — GPU / MUX, pas
ici (une règle vit à un seul endroit ; ce document y renvoie, il ne la
recopie pas) :

- `dgpu_disable` doit rester à `0`.
- `current_value` reflète l'écriture immédiatement — c'est l'état voulu,
  pas l'état réalisé par le MUX matériel ; c'est `pending_reboot`, pas une
  relecture de `current_value`, qui atteste que l'écriture a atteint le
  firmware.

## Chemin de retour 1 — TTY local (`Ctrl+Alt+F3`)

**Ce qui doit être vrai avant le redémarrage** : un getty doit pouvoir
s'amorcer sur une console virtuelle. Vérifié ce jour :

- `systemctl is-active getty.target` → `active`, `systemctl is-enabled
  getty.target` → `static`.
- `systemctl is-enabled getty@.service` (unité modèle) → `enabled` : les
  instances par tty (`getty@tty2`, `getty@tty3`, …) s'amorcent à la demande
  au changement de console, ce qui explique qu'elles apparaissent
  `inactive`/`disabled` individuellement tant qu'aucun changement de VT n'a
  eu lieu — ce n'est pas une anomalie.
- `/dev/tty1` à `/dev/tty6` présents (`ls /dev/tty[1-6]`).

**Commande** : `Ctrl+Alt+F3` (ou F2/F4/…) depuis la session graphique,
authentification locale normale une fois sur la console.

**Limite non couverte par ce chemin** : rien ci-dessus ne garantit que la
sortie vidéo d'une console virtuelle reste pilotée par un GPU actif après
une bascule MUX ratée — la commutation peut couper l'affichage sur les deux
cartes selon l'état du firmware. Aucune commande locale ne peut établir ce
point sans déclencher la bascule réellement ; ce chemin n'est donc pas
supposé suffisant à lui seul, d'où le chemin 2 ci-dessous, indépendant.

## Chemin de retour 2 — SSH entrant

**Ce qui doit être vrai avant le redémarrage**, sur *ce* poste (celui qui
bascule, pas la machine distante) :

1. `openssh-server` installé.
2. `sshd` actif **et** activé (survit à un redémarrage).
3. Au moins une clé publique dans `~/.ssh/authorized_keys` de ce poste.
4. `firewalld` autorisant le service `ssh` sur la zone active.
5. L'adresse IP de ce poste relevée et **communiquée à l'avance** à la
   personne qui devra s'y connecter — ce document ne contient aucune
   adresse réelle (décision D4) ; relever la sienne avec `ip -brief addr
   show` et la noter hors de ce dépôt.

Le rôle [`roles/recovery/`](../roles/recovery/) vérifie et prépare
automatiquement les points 1 à 4 sur ce poste (voir son
[`README.md`](../roles/recovery/README.md) pour l'usage), et rapporte le
point 5 en sortie de tâche sans jamais l'écrire dans un fichier versionné.

**Commande, une fois connecté** : `ssh utilisateur@IP-de-ce-poste` depuis la
machine distante.

### Direction : ce qui est établi, ce qui ne l'est pas

Une connexion **sortante** réussie depuis ce poste vers la machine distante
(l'échange de clés déjà réalisé) **ne prouve rien** sur la capacité de la
machine distante à se connecter en retour vers *ce* poste. Ce sont deux
directions indépendantes, avec des préconditions différentes de chaque
côté.

Ce qui est établi localement, sur ce poste, à la date de cette procédure :
présence de `sshd`, contenu (non exposé) d'`authorized_keys`, règle
`firewalld` autorisant `ssh` sur la zone active — voir le rôle `recovery`
pour le détail vérifiable à volonté.

**Marqueur fermé le 2026-08-04.** Preuve obtenue, localement, sans dépendre
de la seule parole de la machine distante : `journalctl -u sshd -g
'Accepted'` montre plusieurs connexions acceptées depuis la machine
distante sur ce poste, dont une par clé publique (`Accepted publickey for
<utilisateur> from <machine distante> ... ssh2`, une entrée `Accepted
password` également présente pour un essai antérieur). Adresse source et
empreinte de clé volontairement omises ici (D4 : pas d'adressage réel en
dur ; pas de matière de clé). Ce journal est une trace locale d'un
événement réseau entrant déjà survenu — c'est une preuve directe, pas une
déduction, contrairement à un simple test de connexion sortant.

## ScreenPad Plus (`card1-DP-3`) — survie à la bascule confirmée [FERMÉ le 2026-08-05]

Le ScreenPad Plus est l'écran secondaire de ce Zephyrus Duo. Rattachement
vérifié le 2026-08-04 (avant toute écriture) :

- `/dev/dri/by-path/pci-0000:09:00.0-card -> ../card1` — la carte `card1`
  est bien rattachée au bus PCI `0000:09:00.0`, l'iGPU AMD (voir
  `docs/machine-facts.md` § Matériel).
- `cat /sys/class/drm/card1-DP-3/status` → `connected`.

**Méthode de fermeture — trois relevés comparés, pas un seul :**

1. **Juste avant l'écriture réelle du 2026-08-05** (08:02:55Z, capturé
   automatiquement par `roles/gpu_mux/` dans le fichier de trace non
   versionné `roles/gpu_mux/trace/pre-bascule-20260805T100255.json`, relu
   dans cette session) : `card1-DP-3/status: connected`,
   `/dev/dri/by-path/pci-0000:09:00.0-card -> ../card1`.
2. **Après le redémarrage réel**, dans cette session, par lecture directe :
   `cat /sys/class/drm/card1-DP-3/status` → `connected` ;
   `ls -la /dev/dri/by-path/` → `pci-0000:09:00.0-card -> ../card1`,
   inchangé.
3. La numérotation DRM de la carte NVIDIA a changé entre ces deux relevés
   (`card2` → `card0`, voir `docs/machine-facts.md` § Affichage), mais
   celle de la carte AMD est restée `card1` dans les deux — une
   coïncidence de nommage, pas une preuve à elle seule : c'est le
   rattachement au bus PCI `0000:09:00.0`, constant quel que soit le nom
   `cardN` attribué, qui établit la continuité.

**Marqueur fermé.** Le rattachement au bus PCI de l'iGPU et l'état
`connected` du connecteur `DP-3` sont identiques avant et après la
bascule réelle. Le ScreenPad Plus a survécu à la commutation MUX.

**Le plan de retour ci-dessus (chemins 1 et 2) ne suppose toujours pas que
cet écran reste disponible** — ce constat ferme le doute, il ne change
pas la conception du plan de retour, qui reste volontairement indépendant
de lui.

## Revenir en mode discret (`gpu_mux_mode` → `0`)

Depuis un TTY local ou une session SSH entrante fonctionnelle (chemins
ci-dessus) :

```sh
# Chemin retenu depuis le 2026-08-05 (D2ter révoquée en partie SUR SON
# MOTIF, pas sur son choix — voir « Motif de la révocation, corrigé »
# ci-dessous et docs/machine-facts.md § Décisions) : écriture directe,
# pas `asusctl armoury set`. Motif corrigé : l'écriture directe fournit
# une confirmation vérifiable avant redémarrage (pending_reboot passe à
# 1) ; asusctl laisse pending_reboot à 0 sans jamais dire si la valeur en
# file d'attente sera appliquée — asus-shutdown.service l'a bel et bien
# appliquée le 2026-08-05, alors qu'il est rapporté `disabled` par
# `systemctl is-enabled` (démarré à la demande, pas d'activation
# statique). La préférence tient à la vérifiabilité, pas à une
# impossibilité de la voie asusd.
echo 0 | sudo tee /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value
```

Puis redémarrer :

```sh
sudo systemctl reboot
```

**Après redémarrage**, confirmer avec `pending_reboot` — pas avec une
relecture isolée de `current_value` (voir « Gardes » ci-dessus) :

```sh
cat /sys/class/firmware-attributes/asus-armoury/attributes/pending_reboot
# doit valoir 0 après le redémarrage effectif ; s'il vaut encore 1,
# le changement n'a pas été pris en compte au boot.
cat /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value
# doit valoir 0
```

## Résultats observés — tentative de bascule du 2026-08-04

**État final à l'issue de cette série : écriture envoyée, mise en file
d'attente confirmée par le journal d'`asusd`, mais `pending_reboot` n'est
PAS passé à `1`.** Le rôle `roles/gpu_mux/` s'est donc arrêté en échec
(code retour 2) à sa dernière assertion — c'est le comportement voulu (il
ne se déclare jamais en succès sans preuve), mais ce n'est pas l'issue
prévue par la procédure ci-dessus. Aucun redémarrage n'a eu lieu.

**Résolution avant édition (lecture seule d'abord)** : `asusctl armoury
set --help` confirme la syntaxe `asusctl armoury set <property> <value>` —
pas présumée, lue. `asusctl armoury list` déclare 11 attributs
(`boot_sound`, `charge_mode`, `dgpu_disable`, `gpu_mux_mode`,
`mini_led_mode`, `nv_dynamic_boost`, `nv_temp_target`, `panel_overdrive`,
`ppt_pl1_spl`, `ppt_pl2_sppt`, `ppt_pl3_fppt`), qui correspondent
exactement aux 11 sous-répertoires de
`/sys/class/firmware-attributes/asus-armoury/attributes/` — seul
`pending_reboot` (fichier direct, pas un sous-répertoire) existe côté
sysfs sans être une « propriété » pilotable par `asusctl armoury set`, ce
qui est cohérent avec son rôle de simple indicateur d'état. Aucune
divergence trouvée entre ce qu'`asusctl` déclare piloter et le contenu
réel de `attributes/`. Chemin retenu pour l'écriture : `asusctl armoury
set gpu_mux_mode 1` (pas d'écriture sysfs directe en repli, `asusctl` a
accepté la commande).

**Séquence exécutée par `roles/gpu_mux/`, dans l'ordre :**

1. `dgpu_disable == 0` → passé.
2. `gpu_mux_mode` existe, `1` présent dans `possible_values` (`0;1`) →
   passé.
3. `pending_reboot == 0` avant écriture → passé.
4. Instantané pré-bascule capturé (connecteurs DRM par carte,
   `/dev/dri/by-path/`, `nvidia-smi`, `kscreen-doctor -o`, valeurs des
   trois attributs) dans un fichier de trace non versionné,
   `roles/gpu_mux/trace/pre-bascule-<horodatage>.json`.
5. `asusctl armoury set gpu_mux_mode 1` exécuté — **code de retour 0**,
   aucune erreur signalée par `asusctl`.
6. `pending_reboot` relu après écriture → **toujours `0`**, pas `1`.
   Assertion en échec, rôle arrêté (code retour 2).

**Cause identifiée, en lecture seule, sans nouvelle écriture :**
`journalctl -u asusd` montre, immédiatement après l'écriture :
`[DEBUG asusd::asus_armoury] Queueing GPU attribute gpu_mux_mode = 1 for
delayed apply`. `asusd` ne pousse donc pas la valeur au firmware de façon
synchrone sur `armoury set` pour cet attribut — il la met en file
d'attente. Plus de deux minutes après (relevé à 23:07, écriture à 23:04),
aucune évolution : ni `pending_reboot`, ni `gpu_mux_mode/current_value`,
ni nouvelle ligne dans le journal `asusd`.

**Marqueur fermé le 2026-08-05 — mécanisme d'application de la file
d'attente établi.** Piste confirmée, en lecture seule, sans redémarrer
`asusd` ni écrire quoi que ce soit : `asusd.service` déclare
`Before=asus-shutdown.service` ; l'unité `asus-shutdown.service` (« ASUS
Deferred Shutdown Handler », capacités `CAP_SYS_ADMIN`/`CAP_SYS_MODULE`)
est elle-même ordonnée `Before=shutdown.target reboot.target
halt.target`. Mais `systemctl is-enabled asus-shutdown.service` →
`disabled` : cette unité, seule consommatrice possible de la file
d'attente, ne s'exécute donc jamais dans le cycle d'arrêt sur ce système.
Confirmé par ailleurs : `/var/lib/asusd/` n'existe pas et
`/etc/asusd/asusd.ron` date d'avant la mise en file (pas de persistance
disque non plus) ; `asusctl armoury get gpu_mux_mode` ne restitue aucune
valeur en attente (`current: [(0),1]`, identique à avant l'écriture) —
`asusd` n'expose donc pas cette file ailleurs qu'en mémoire vive.
**Conclusion : la file est purement en mémoire, son unique consommateur
est désactivé, elle n'est donc jamais appliquée.** `pending_reboot` restait
à `0` à raison : rien n'a jamais été écrit dans sysfs par ce chemin.

**Conséquence retenue (voir décision D2ter, révocation partielle du
2026-08-05, dans `docs/machine-facts.md`) : abandon d'`asusctl armoury
set` au profit d'une écriture directe dans `current_value`.** Cette
indirection par `asusd` échangeait une confirmation vérifiable
(`pending_reboot`) contre une intention non persistée — un repli
silencieux caractérisé, au sens de `CLAUDE.md` § Sourcing des faits :
`asusctl` retournait un code de retour 0 sans jamais signaler que la
demande resterait sans suite tant qu'`asus-shutdown` ne tourne pas.
L'écriture directe restaure `pending_reboot` comme preuve exploitable
avant redémarrage — voir `roles/gpu_mux/tasks/main.yml`.

Point non résolu, consigné et compté dans `docs/machine-facts.md` §
Points ouverts plutôt qu'ici (c'est une question sur le comportement
général d'`asus-shutdown.service`, pas spécifique à cette procédure de
retour) : le rôle exact de cette unité (capacités `CAP_SYS_MODULE`,
`SystemCallFilter=@module`) reste incertain — hypothèse non confirmée d'un
déchargement des modules NVIDIA avant commutation, qui rendrait une
écriture directe potentiellement incomplète si ce travail s'avérait
nécessaire.

**Rôle `roles/gpu_mux/` : cas nominal et échecs forcés, tous vérifiés le
même jour, avant l'écriture réelle ci-dessus :**

- `--syntax-check` : passé.
- `--check` : aucune écriture, aucune vérification post-écriture exécutées
  (gardées par `when: not ansible_check_mode`) ; `gpu_mux_mode` et
  `pending_reboot` inchangés après coup.
- Échec forcé (a) — `-e gpu_mux_target_value=9` (hors `possible_values`) :
  assertion cassée avant toute lecture de `pending_reboot` ou toute
  écriture, message exact : « Valeur cible 9 absente de possible_values
  (0;1) ». Code retour 2.
- Échec forcé (b) — `-e gpu_mux_force_dgpu_disable_value=1` (dGPU
  désactivée simulée, jamais écrite) : la lecture réelle de `dgpu_disable`
  est même sautée (`skipping`), l'assertion casse avant toute autre
  action, message exact : « dgpu_disable vaut 1, pas 0 ». Code retour 2.
  `dgpu_disable` réel confirmé toujours à `0` après ce test.

**Confirmations finales de cette série** : `dgpu_disable` toujours `0`
(jamais écrit par ce rôle, ni par les tests) ; `supergfxd` toujours
`inactive`/`disabled` (jamais démarré ni activé) ; aucun paquet installé ;
aucun dépôt modifié ; aucune clé GPG importée ; **aucun redémarrage**.

## Résultats observés — révocation partielle de D2ter et second essai (2026-08-05)

**État final à l'issue de cette série : l'écriture directe n'a pas pu être
tentée sur le matériel — bloquée avant tout accès root par l'absence d'un
moyen d'authentification `sudo` dans cette session.** Ce n'est pas une
garde qui a cassé : c'est une limite d'environnement, distincte, à traiter
différemment (voir plus bas). Aucun redémarrage n'a eu lieu.

**Nettoyage de l'état courant — investigation en lecture seule (garde 2
de la demande).** Sans redémarrer `asusd` ni écrire quoi que ce soit :

- `journalctl -u asusd -g 'watch|mux|inotify'` montre qu'`asusd` fait un
  « Reload » de `gpu_mux_mode` **uniquement à son propre démarrage**
  (`Reload called on GPU attribute gpu_mux_mode: doing nothing` /
  `No saved value for attribute gpu_mux_mode: skipping.`), et que le seul
  `inotify watch` actif qu'il déclare cible son fichier de configuration
  (`Starting inotify watch for asusd config file`), pas les attributs
  sysfs sous `asus-armoury`.
- Établi : aucune preuve, dans le journal, qu'`asusd` surveille ou
  réapplique `gpu_mux_mode/current_value` en continu.
- **Non établi** : je n'ai pas pu inspecter les descripteurs de fichiers
  ouverts par le processus `asusd` (`/proc/<pid>/fd/`, `fdinfo`) faute de
  privilège (`sudo -n` refusé, pas de mot de passe disponible dans cette
  session) — un `inotify_add_watch` actif sur le répertoire
  `gpu_mux_mode/` sans trace dans les journaux de niveau `INFO`/`DEBUG`
  affichés ne peut donc pas être formellement exclu, seulement jugé
  improbable au vu de ce qui précède.
- Conclusion retenue : le risque d'interférence n'a pas été jugé assez
  probable pour justifier l'arrêt prévu par la consigne — mais la limite
  ci-dessus (pas d'accès aux fd du processus) est consignée telle quelle,
  pas comme une certitude.

**Rôle `roles/gpu_mux/` adapté — mécanisme d'écriture seul remplacé,
gardes inchangées :**

- `--syntax-check` : passé.
- `--check` : la nouvelle tâche d'écriture (`ansible.builtin.shell`,
  `become: true`) et la vérification post-écriture restent ignorées sous
  `--check` (`when: not ansible_check_mode`) — aucune tentative
  d'élévation de privilège en simulation.
- Échec forcé (a) — `-e gpu_mux_target_value=9` : casse sur
  `possible_values`, avant toute tentative d'écriture. Rejoué à
  l'identique, même résultat.
- Échec forcé (b) — `-e gpu_mux_force_dgpu_disable_value=1` : casse avant
  la lecture réelle de `dgpu_disable`. Rejoué à l'identique, même
  résultat. `dgpu_disable` réel confirmé toujours à `0`.
- **Pré-vol** : rôle `recovery` ré-exécuté réellement, de nouveau
  entièrement idempotent (`changed=0`). Preuve de connexion entrante
  rejouée : `journalctl -u sshd -g 'Accepted'` montre une nouvelle
  connexion par clé publique acceptée depuis la machine distante, en plus
  de celles de la série précédente.

**Exécution réelle — blocage avant toute écriture.** Toutes les gardes en
lecture seule passent (`dgpu_disable=0`, attribut présent, cible valide,
`pending_reboot=0`), l'instantané pré-bascule est capturé normalement.
La tâche d'écriture elle-même échoue immédiatement :

```
[ERROR]: Task failed: Premature end of stream waiting for become success.
>>> Standard Error
sudo: a password is required
```

Code retour 2, échec propre et immédiat (pas de blocage, pas de tentative
partielle) — le fichier cible est `root:root 0644`
(`ls -la .../gpu_mux_mode/current_value`), et `sudo -n true` avait déjà
établi l'absence de `sudo` sans mot de passe pour ce compte
(`groups=mahieumi,wheel` — membre de `wheel`, mais pas de règle `NOPASSWD`
configurée). Je n'ai ni demandé ni utilisé de mot de passe : ce n'est pas
un blocage que je peux lever depuis cette session sans que tu fournisses
explicitement un moyen d'authentification (`--ask-become-pass` en
exécutant toi-même le rôle, ou une règle `sudoers` `NOPASSWD` limitée à ce
fichier, à ta décision — pas la mienne).

**Confirmations finales de cette série** : `gpu_mux_mode=0` et
`pending_reboot=0` inchangés (aucune écriture n'a pu atteindre le
matériel) ; `dgpu_disable=0` inchangé ; `asus-shutdown` et `supergfxd`
toujours `disabled` (`is-enabled`), jamais démarrés ni activés par cette
série ; aucun paquet installé (`sudo` déjà présent de base,
`sudo-1.9.17-8.p2.fc44`, pas installé par cette série) ; aucun dépôt
modifié ; aucune clé GPG importée ; **aucun redémarrage**.

## Résultats observés — bascule réussie du 2026-08-05

**Écriture réelle exécutée avec succès entre les deux sessions
précédentes.** Le blocage `sudo` documenté dans la section précédente a
été levé côté opérateur, hors de cette série. `~/.bash_history` suggérait
la commande, mais un historique shell atteste une intention tapée, pas un
résultat (voir `CLAUDE.md` § Sourcing des faits) — établi ici par une
source plus forte, le journal systemd persistant, relu dans cette
session :

```
$ journalctl --no-pager -g 'sudo|ansible|gpu_mux_mode' --since "2026-08-05 09:55:00" --until "2026-08-05 10:28:00"
[...]
Aug 05 10:02:58 Zephyrus-MM python3[55695]: ansible-ansible.legacy.command Invoked with cmd=echo 1 > /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value [...]
Aug 05 10:02:58 Zephyrus-MM sudo[55682]: mahieumi : PWD=/home/mahieumi/dev/workstation-config/roles/gpu_mux ; USER=root ; COMMAND=/bin/sh -c 'echo BECOME-SUCCESS-[...] ; /usr/bin/python3 [...]/AnsiballZ_command.py'
Aug 05 10:02:58 Zephyrus-MM sudo[55682]: pam_unix(sudo:session): session opened for user root(uid=0) by mahieumi(uid=1000)
Aug 05 10:02:58 Zephyrus-MM asusd[1472]: [DEBUG asusd::asus_armoury] gpu_mux_mode changed
[... 7 lignes identiques ...]
Aug 05 10:02:58 Zephyrus-MM sudo[55682]: pam_unix(sudo:session): session closed for user root
[...]
Aug 05 10:27:26 Zephyrus-MM audit[56960]: [...] cmd="reboot" exe="/usr/bin/sudo" [...] res=success
```

Ce journal, indépendant de l'historique shell, établit trois faits
distincts : l'écriture directe `echo 1 > .../gpu_mux_mode/current_value`
a réellement été exécutée en tant que root (session `sudo` ouverte puis
fermée à 10:02:58) ; `asusd` a réagi immédiatement au changement (huit
lignes « gpu_mux_mode changed ») ; le redémarrage qui a suivi a lui aussi
été confirmé par l'audit système (`cmd="reboot"`, 10:27:26), pas
seulement par une ligne d'historique.

**Ce que ce journal ne montre pas** : la sortie du `cat pending_reboot`
exécuté juste après l'écriture (10:02:58) — `journalctl` capture
l'invocation de la commande par Ansible, pas son résultat sur stdout.
`@VERIF : valeur exacte de pending_reboot lue immédiatement après
l'écriture directe du 2026-08-05T10:02:58 — irrécupérable
rétroactivement (ni l'historique shell ni le journal systemd n'en
gardent trace). Action pour une prochaine écriture réelle : faire
enregistrer explicitement (register: + assertion ou ajout au fichier de
trace) la sortie de cette lecture, plutôt que de compter sur le log
d'invocation ou l'historique shell seuls.` Ce que la lecture
post-redémarrage ci-dessous établit sans ambiguïté, en revanche, c'est
que la bascule a été prise en compte — état vérifié dans **cette
session**, post-redémarrage, par lecture directe :

```
$ cat /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value
1
$ cat /sys/class/firmware-attributes/asus-armoury/attributes/pending_reboot
0
```

`current_value=1` et `pending_reboot=0` : la bascule est prise en compte,
conforme au tableau ci-dessous.

### Journal du boot précédent — rôle d'`asus-shutdown.service` établi

`journalctl -u asus-shutdown -b -1 --no-pager` (exécuté dans cette
session) montre la séquence complète exécutée à l'extinction précédant ce
redémarrage :

1. Détection d'un réglage GPU en file d'attente (`gpu_mux_mode => 1`).
2. Attente que la dGPU soit inactive (« idle »).
3. Arrêt de `nvidia-powerd.service`, `nvidia-persistenced.service`,
   `nvidia-fabricmanager.service` avant l'écriture du firmware.
4. **Cinq tentatives de `modprobe -r` sur la pile NVIDIA, toutes en
   échec** : `modprobe: FATAL: Module nvidia_drm is in use.` (message
   identique aux cinq tentatives), puis `[WARN] Failed to unload NVIDIA
   modules after 5 attempts before firmware attribute apply`.
5. **L'unité applique quand même** l'attribut différé : `[INFO] Applying
   deferred GPU attribute gpu_mux_mode = 1`.
6. Relâchement de l'inhibiteur d'extinction logind (« Released shutdown
   delay inhibitor ») et sortie.

**Conclusion établie, pas une hypothèse** : le déchargement des modules
NVIDIA **n'est pas nécessaire** au succès de la bascule — l'attribut a
été appliqué malgré l'échec des cinq tentatives de `modprobe -r`, et la
bascule a bel et bien réussi (topologie DRM post-redémarrage confirmée
plus bas et dans `docs/machine-facts.md` § Affichage). Ceci ferme le
point non résolu compté dans `docs/machine-facts.md` § Points ouverts
(« Rôle exact d'`asus-shutdown.service` ») — le doute portait précisément
sur la nécessité de ce déchargement pour une écriture directe dans
`current_value`.

### Motif de la révocation partielle de D2ter, corrigé [MOTIF CORRIGÉ le 2026-08-05]

**ÉTAIT** (voir `docs/machine-facts.md` § Décisions, D2ter — révocation
partielle du 2026-08-05) : « la file d'attente d'`asusd` n'est jamais
consommée car `asus-shutdown` est désactivé ». **Faux** — démenti par le
journal du boot précédent ci-dessus : l'unité a tourné et a appliqué la
valeur en file.

**MOTIF CORRIGÉ** : l'écriture directe dans sysfs reste le chemin
retenu, mais pour une autre raison. Elle fournit une **confirmation
vérifiable avant redémarrage** : `pending_reboot` passe à `1`
immédiatement après l'écriture. La voie `asusctl armoury set` laisse
`pending_reboot` à `0` — l'opération réussit probablement (la file est
bien consommée par `asus-shutdown.service` à l'extinction, comme
démontré ci-dessus), mais rien, avant de redémarrer, ne permet à
l'opérateur de le savoir. La préférence pour l'écriture directe tient
donc à la **vérifiabilité**, pas à une impossibilité de la voie `asusd`.

**Doute résiduel, largement mais pas entièrement levé.** Le 2026-08-05,
l'écriture directe (rôle `roles/gpu_mux/`) et la file d'`asusd` (mise en
file lors de la tentative du 2026-08-04, jamais purgée depuis) portaient
la **même valeur cible** (`gpu_mux_mode = 1`). Le journal systemd
(§ ci-dessus) établit que l'écriture directe a eu lieu à 10:02:58, plus
de vingt minutes **avant** que `asus-shutdown.service` n'applique la
valeur de la file à l'extinction (10:27:31) — la séquence temporelle
pointe donc vers l'écriture directe comme cause réelle du changement,
`asusd` ayant réagi immédiatement (`gpu_mux_mode changed`) à 10:02:58,
et l'application tardive par `asus-shutdown.service` étant redondante
(même valeur, déjà en place). Ce que ni le journal ni aucune autre
source relue ne confirme : l'état de `current_value` entre 10:02:58 et
10:27:26, faute d'une lecture explicitement journalisée à ce moment
(voir le point non résolu sur `pending_reboot` ci-dessus, même limite). La
conclusion « c'est l'écriture directe qui a produit le changement » est
donc solidement **corroborée**, pas strictement **prouvée** — nuance
qui ne change rien à la décision retenue (les deux chemins visaient la
même cible), mais qui reste à formuler avec cette précision.

### Leçon de méthode — `is-enabled` ne dit pas « ne s'exécutera jamais »

`systemctl is-enabled asus-shutdown.service` renvoie `disabled` avant
comme après cette série (reconfirmé dans cette session, code de retour
1). Ce champ signale l'**absence d'activation statique** au démarrage
(pas de lien créé par `systemctl enable`) — il ne dit rien sur un
démarrage **à la demande**, déclenché par un autre service, par D-Bus,
ou par activation par socket. C'est exactement ce qui se produit ici :
`asus-shutdown.service` a tourné pendant l'extinction du 2026-08-04 au
2026-08-05 sans jamais avoir été activé — vraisemblablement démarré par
`asusd` au moment où un attribut est mis en file d'attente (le
vocabulaire d'inhibiteur logind employé dans son propre journal, et le
découpage en phases numérotées orienté vers une exécution ponctuelle
plutôt que vers un service de fond, corroborent cette hypothèse sans la
démontrer). Le mécanisme exact de ce démarrage à la demande reste à
confirmer — point compté et marqué dans `docs/machine-facts.md` §
Points ouverts, pas ici, par cohérence avec le choix déjà fait pour le
point sur le rôle d'`asus-shutdown.service` (question générale sur le
comportement du service, pas spécifique à cette procédure de retour).

### Topologie post-bascule — renvoi

Renumérotation DRM (`card2` → `card0` pour la NVIDIA, `card1` stable pour
l'AMD) et renommage des sorties KScreen (`eDP-2` → `eDP-1`) confirmés
dans cette session — détail complet et sourcé dans
[`docs/machine-facts.md`](machine-facts.md) § Affichage, pour ne pas
dupliquer une deuxième source.

## Commandes de vérification post-redémarrage (à exécuter par l'opérateur, pas par cette série)

Cette liste prépare la validation attendue après un redémarrage réel —
elle n'a pas été exécutée dans cette série et ne doit pas être anticipée.

**[EXÉCUTÉE le 2026-08-05]** Le redémarrage a eu lieu ; la table
ci-dessous est conservée telle quelle pour l'historique (écrite *avant*
que ces valeurs soient connues). Deux écarts entre la prédiction et
l'observation, à noter explicitement plutôt qu'à corriger en silence :

- Ligne `card2-eDP-2` : elle supposait implicitement que la numérotation
  `cardN` resterait stable après bascule (seul l'état du connecteur
  changerait). Ce n'est pas ce qui a été observé — `card2` n'existe plus
  du tout après redémarrage, la carte NVIDIA est renumérotée `card0`
  (voir `docs/machine-facts.md` § Affichage). `card0-eDP-2` est bien
  `disconnected`, ce qui confirme l'esprit de la prédiction (le panneau
  n'est plus piloté par la NVIDIA), mais pas sa forme littérale : le
  chemin `card2-eDP-2` lui-même n'existe plus, il ne se contente pas de
  changer d'état.
- Ligne `nvidia-smi` : la prédiction « sans processus graphique » n'est
  que partiellement vérifiée. `kwin_wayland` (`C+G`, 13 MiB) reste listé
  après bascule — progrès réel par rapport à l'état pré-bascule, mais pas
  la disparition complète anticipée par cette ligne. Voir
  `docs/machine-facts.md` § GPU et § Points ouverts (nouveau point sur
  l'état d'alimentation runtime de la dGPU).

| Commande | Valeur attendue si la bascule a réussi |
|---|---|
| `cat /sys/class/firmware-attributes/asus-armoury/attributes/gpu_mux_mode/current_value` | `1` |
| `cat /sys/class/firmware-attributes/asus-armoury/attributes/pending_reboot` | `0` (remis à zéro une fois le changement pris en compte au boot) |
| `journalctl -u asus-shutdown --no-pager` (sur le boot précédent : `journalctl -u asus-shutdown -b -1`) | trace de l'application de `gpu_mux_mode` pendant l'arrêt précédent — à lire pour confirmer ou infirmer la piste du mécanisme « delayed apply » décrite plus haut |
| `cat /sys/class/drm/card1-eDP-1/status` | `connected` (panneau principal maintenant piloté par l'iGPU AMD) |
| `cat /sys/class/drm/card2-eDP-2/status` | `disconnected` (le chemin direct vers la RTX 4090 n'est plus utilisé pour l'affichage) |
| `cat /sys/class/drm/card1-DP-3/status` | `connected` attendu (survie du ScreenPad Plus) — **si `disconnected`, le point non résolu sur la survie du ScreenPad Plus (section ci-dessus) est infirmé : le documenter comme tel, pas comme un point encore ouvert** |
| `nvidia-smi` | fonctionne, sans processus graphique (`kwin_wayland`, `plasmashell`, etc.) dans la table des processus — seuls des processus de calcul, s'il y en a |
| `kscreen-doctor -o` | modes disponibles sur la sortie du panneau principal ; noter en particulier si `2560x1600@240` est encore listé — sa disparition serait un coût réel de D2ter à consigner comme tel dans `docs/machine-facts.md`, pas une simple curiosité |
| `systemctl is-active supergfxd` / `is-enabled` | `inactive` / `disabled` (inchangé — ce chemin n'utilise jamais supergfxd) |
| `cat /sys/class/firmware-attributes/asus-armoury/attributes/dgpu_disable/current_value` | `0` (inchangé) |

## Voir aussi

- [`roles/recovery/`](../roles/recovery/) — prépare et vérifie
  automatiquement les préconditions du chemin SSH ci-dessus.
- [`docs/machine-facts.md`](machine-facts.md) — décisions D2bis/D2ter,
  attributs `asus-armoury`, topologie DRM complète.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing et gardes MUX
  persistantes.
