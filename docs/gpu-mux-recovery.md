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

- **`dgpu_disable` doit rester à `0`.** Le pilote refuse la commutation MUX
  si la dGPU est désactivée. `dgpu_disable` (coupe la carte) et
  `gpu_mux_mode` (choisit à quelle carte le panneau est câblé) vivent tous
  deux sous
  `/sys/class/firmware-attributes/asus-armoury/attributes/` et se
  ressemblent — les confondre coupe CUDA en croyant seulement changer
  d'affichage.
- **Après écriture de `gpu_mux_mode`, `current_value` renvoie l'ancienne
  valeur jusqu'au redémarrage.** La confirmation avant reboot est le champ
  `pending_reboot`, au niveau `attributes/` (fichier direct, pas un fichier
  séparé sous `gpu_mux_mode/`). Relire `current_value` juste après
  l'écriture et le trouver inchangé n'est **pas** un échec de l'écriture —
  c'est le comportement attendu tant que la machine n'a pas redémarré. Une
  vérification par relecture immédiate de `current_value` produit un faux
  négatif ; c'est `pending_reboot` qui fait foi avant reboot.

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

## ScreenPad Plus (`card1-DP-3`) — rattachement DRM établi, survie post-bascule non prouvée

Le ScreenPad Plus est l'écran secondaire de ce Zephyrus Duo. Rattachement
vérifié ce jour :

- `/dev/dri/by-path/pci-0000:09:00.0-card -> ../card1` — la carte `card1`
  est bien rattachée au bus PCI `0000:09:00.0`, l'iGPU AMD (voir
  `docs/machine-facts.md` § Matériel).
- `cat /sys/class/drm/card1-DP-3/status` → `connected`.

Le rattachement au bus de l'iGPU est donc établi. Ce qui ne l'est pas :

`@VERIF : survie effective de l'affichage sur le ScreenPad Plus après une
bascule de gpu_mux_mode. Le rattachement au bus PCI de l'iGPU ne garantit
pas que le firmware maintienne cette sortie active pendant/après une
commutation MUX ratée — cela ne peut être prouvé qu'en basculant
réellement, ce qui est hors périmètre de ce document.`

**Le plan de retour ci-dessus (chemins 1 et 2) ne suppose pas que cet
écran reste disponible.** Il est mentionné ici pour mémoire, pas comme
troisième chemin de secours : TTY local et SSH entrant sont conçus pour
suffire indépendamment l'un de l'autre, sans lui.

## Revenir en mode discret (`gpu_mux_mode` → `0`)

Depuis un TTY local ou une session SSH entrante fonctionnelle (chemins
ci-dessus) :

```sh
# Chemin officiel — cohérent avec D2ter (asusd/asus-armoury, pas supergfxd)
asusctl armoury set gpu_mux_mode 0

# Repli si asusd/asusctl est indisponible pour une raison quelconque :
# écriture directe, chemin complet
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

Piste identifiée pour l'application réelle de cette file d'attente,
établie par lecture de fichiers d'unité systemd, pas du code source
d'`asusd`/`asus-shutdown` — la nuance est portée par le marqueur qui suit :
`asusd.service` déclare `Before=asus-shutdown.service` ; l'unité
`asus-shutdown.service` (« ASUS Deferred Shutdown Handler », active depuis
le démarrage de session, capacités `CAP_SYS_ADMIN`/`CAP_SYS_MODULE`) est
elle-même ordonnée `Before=shutdown.target reboot.target halt.target`.
Cette combinaison est cohérente avec un mécanisme où les attributs mis en
file d'attente par `asusd` sont réellement écrits au firmware par
`asus-shutdown` au moment de l'arrêt/redémarrage, pas avant.

`@VERIF : mécanisme exact d'application de la file d'attente « delayed
apply » côté asusd/asus-shutdown. Établi ici par inférence à partir de
l'ordonnancement et des capacités systemd (asusd.service, asus-shutdown.service),
pas par lecture du code source d'asusd ou d'asus-shutdown — à confirmer
en amont (dépôt asusctl/asusd) ou par observation directe du journal
d'asus-shutdown pendant un redémarrage réel.`

**Conséquence pour la confirmation avant reboot.** La garde documentée
plus haut (« la confirmation avant reboot est `pending_reboot` ») reste la
méthode correcte *si* `pending_reboot` finit par passer à `1` avant le
redémarrage — ce qui n'a pas été observé ici dans la fenêtre de temps
couverte par cette série. Il est possible que `pending_reboot` ne passe à
`1` qu'au moment même de l'arrêt (juste avant que `asus-shutdown` agisse),
auquel cas cette confirmation pré-reboot spécifique n'est tout simplement
pas observable en dehors d'un redémarrage réel sur cette version d'asusd —
à vérifier lors du prochain redémarrage (voir « Commandes de vérification
post-redémarrage » ci-dessous, à exécuter par l'opérateur).

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

## Commandes de vérification post-redémarrage (à exécuter par l'opérateur, pas par cette série)

Cette liste prépare la validation attendue après un redémarrage réel —
elle n'a pas été exécutée dans cette série et ne doit pas être anticipée.

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
