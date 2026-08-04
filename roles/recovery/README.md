# roles/recovery

Rôle Ansible ciblant `localhost`. Vérifie et prépare un chemin de retour SSH
sur ce poste avant une bascule du MUX GPU (décisions D2bis/D2ter). La
procédure de récupération complète, les gardes et les points encore
`@VERIF` sont documentés dans
[`docs/gpu-mux-recovery.md`](../../docs/gpu-mux-recovery.md) — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait

- Vérifie que `openssh-server` (nom configurable) est installé. **Ne
  l'installe jamais** : absent, il échoue bruyamment avec le message et la
  commande à lancer manuellement — installer un serveur SSH est une
  décision, pas un effet de bord d'un rôle de récupération.
- Active et démarre `sshd`.
- Asserte qu'au moins une clé publique est présente dans `authorized_keys`
  sur ce poste — échoue bruyamment sinon.
- Relève l'état de firewalld pour le service `ssh` (lecture seule, ne
  modifie jamais la configuration du pare-feu).
- Relève l'adresse IPv4 de ce poste et la rapporte en sortie de tâche.

## Ce que ce rôle ne fait jamais

- Il n'écrit jamais dans `gpu_mux_mode` ni dans aucun attribut
  `asus-armoury`.
- Il ne démarre ni n'active `supergfxd`.
- Il ne redémarre pas la machine.
- Il ne prouve pas qu'une connexion SSH **entrante** depuis la machine
  distante fonctionnerait : seule cette dernière peut le prouver, en
  s'y connectant réellement.

## Utilisation

```
ansible-playbook roles/recovery/recovery.yml
```

En mode simulation d'abord :

```
ansible-playbook --check roles/recovery/recovery.yml
```

## Variables

Voir `defaults/main.yml`. Aucune valeur concrète (IP, utilisateur distant)
n'est versionnée : ce sont des exemples neutres (RFC 5737) à surcharger via
`-e`, un inventaire local ou des `group_vars` — voir `.gitignore` pour les
emplacements déjà exclus du suivi git.
