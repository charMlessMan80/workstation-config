# roles/gpu_mux

Rôle Ansible ciblant `localhost`. Écrit `gpu_mux_mode` par **écriture
directe dans `current_value`** pour basculer vers Optimus/Hybrid
(décisions D2bis, D2ter révoquée en partie le 2026-08-05 — voir
[`docs/machine-facts.md`](../../docs/machine-facts.md)) après une série de
gardes en lecture seule. Procédure complète, gardes détaillées et résultats
observés sont dans
[`docs/gpu-mux-recovery.md`](../../docs/gpu-mux-recovery.md) — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Échoue bruyamment si `dgpu_disable != 0`.
2. Échoue bruyamment si `gpu_mux_mode` n'existe pas ou si
   `gpu_mux_target_value` n'est pas dans `possible_values`.
3. Échoue bruyamment si `pending_reboot != 0` avant toute écriture.
4. Capture un instantané de l'état avant bascule dans un fichier de trace
   **non versionné** (`trace/`, voir `.gitignore`).
5. Écrit directement la valeur cible dans
   `gpu_mux_mode/current_value` (`become: true`, seule tâche privilégiée du
   rôle — le fichier est `root:root 0644`).
6. Confirme par `pending_reboot` (0 → 1), jamais par une relecture de
   `current_value` — voir `CLAUDE.md` § Matériel spécifique — GPU / MUX
   pour ce que `current_value` reflète exactement avant reboot (règle
   corrigée le 2026-08-05 ; ce README ne la recopie pas, une règle vit à
   un seul endroit).

## Ce que ce rôle ne fait jamais

- Il n'écrit jamais dans `dgpu_disable`.
- Il ne démarre ni n'active `supergfxd` ni `asus-shutdown` — les deux
  restent `disabled` sur ce poste, ce que cette série a explicitement
  vérifié sans les toucher.
- Il n'installe aucun paquet.
- Il ne modifie aucun dépôt, n'importe aucune clé GPG.
- **Il ne redémarre jamais la machine.** La bascule reste *en attente de
  redémarrage* tant qu'un reboot n'a pas été déclenché manuellement — ce
  rôle ne se déclare jamais en succès sur la seule écriture.
- En mode `--check`, il ne réalise jamais l'écriture ni la vérification
  post-écriture (gardées par `when: not ansible_check_mode`) : seules les
  lectures et la capture de trace s'exécutent.

## Utilisation

```
ansible-playbook --check roles/gpu_mux/gpu_mux.yml   # simulation d'abord
ansible-playbook roles/gpu_mux/gpu_mux.yml           # écriture réelle
```

## Variables

Voir `defaults/main.yml`. `gpu_mux_target_value` (par défaut `1`, Optimus/
Hybrid) et `gpu_mux_force_dgpu_disable_value` (indéfinie par défaut,
réservée aux démonstrations d'échec forcé) sont conçues pour être
surchargées via `-e` sans modifier ce fichier.
