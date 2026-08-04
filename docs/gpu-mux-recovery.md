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

`@VERIF : preuve d'une connexion SSH entrante réussie depuis la machine
distante vers ce poste. Elle ne peut être établie que depuis cette machine
distante elle-même, en s'y connectant réellement — hors de portée de ce
poste et de ce rôle. À vérifier avant de compter sur ce chemin en
conditions réelles, pas après.`

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

## Voir aussi

- [`roles/recovery/`](../roles/recovery/) — prépare et vérifie
  automatiquement les préconditions du chemin SSH ci-dessus.
- [`docs/machine-facts.md`](machine-facts.md) — décisions D2bis/D2ter,
  attributs `asus-armoury`, topologie DRM complète.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing et gardes MUX
  persistantes.
