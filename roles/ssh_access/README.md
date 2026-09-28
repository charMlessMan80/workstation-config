# roles/ssh_access

Rôle Ansible ciblant `localhost`. Autorise **une** clé publique SSH dans
`~/.ssh/authorized_keys` de ce poste — à ce jour, celle du PC Windows
personnel de l'opérateur (décision D30, `docs/machine-facts.md`
§ Décisions). Conception relevée en lecture seule avant écriture (R1a),
démontrée en R1b.

## Ce que ce rôle fait

- **Lit** l'état de `sshd` (actif et activé) et échoue s'il ne l'est pas
  — garde G4. Ne le démarre ni ne l'active jamais : c'est le mandat de
  [`roles/recovery/`](../recovery/), qui le précède dans `site.yml`.
- Refuse les valeurs-sentinelles des `defaults/` (G1), une clé qui n'est
  pas une ligne `ssh-ed25519` complète lisible par `ssh-keygen` (G2), une
  empreinte déclarée différente de l'empreinte calculée (G3).
- Copie le trousseau vers `authorized_keys.avant-R1b` **avant** la
  première écriture (`force: false` : jamais écrasée), et refuse d'écrire
  si le trousseau, hors la ligne de cette clé, ne concorde plus avec cette
  sauvegarde (sha256).
- Ajoute (ou retire, `ssh_access_state: absent`) la ligne de la clé avec
  `ansible.builtin.lineinfile`, options
  `no-agent-forwarding,no-X11-forwarding,no-port-forwarding`, mode `0600`,
  fichier candidat validé par `ssh-keygen -lf` avant remplacement.
- Vérifie après écriture : empreintes = avant ± cette clé, rien d'autre
  (G5) ; mode `0600` ; `restorecon -n -v` ne propose rien (G6).

G1 à G4 et la garde de sauvegarde échouent **avant** l'écriture ; G5 et
G6 portent sur l'après.

## Ce que ce rôle ne fait jamais

- Il ne touche ni `sshd_config` ni `sshd_config.d/`, ni le pare-feu, ni
  NetworkManager, ni l'état du service `sshd`. L'authentification par
  mot de passe reste donc ce que `sshd` en dit.
- Il ne crée pas `authorized_keys` s'il est absent.
- Aucune tâche `become`.
- Il ne prouve pas qu'une connexion depuis la machine détentrice de la
  clé fonctionne : seule une connexion réelle, depuis elle, le prouve.

## Clé et empreinte : hors du dépôt

`site.yml` est lancé sans inventaire (`/etc/ansible/hosts` entièrement
commenté) : l'hôte est le `localhost` implicite, qui charge
`host_vars/localhost/` **à côté de `site.yml`**. Y créer, en mode `0600` :

```
host_vars/localhost/ssh_access.local.yml
```

avec `ssh_access_pubkey` (la ligne complète du `.pub`) et
`ssh_access_pubkey_sha256` (sortie de `ssh-keygen -lf <fichier>.pub`,
deuxième champ). Le motif `/host_vars/localhost/*.local.yml` de
`.gitignore` l'exclut du suivi (vérifier : `git check-ignore -v` doit
répondre rc=0). Aucune clé, aucun commentaire de clé dans le dépôt
(`CLAUDE.md` § Dépôt public).

## Utilisation

```
ansible-playbook site.yml --tags ssh_access --check --diff
ansible-playbook site.yml --tags ssh_access
```

## Retour arrière

Ciblé (retire la seule ligne de cette clé, rejoue toutes les gardes) :

```
ansible-playbook site.yml --tags ssh_access -e ssh_access_state=absent
```

Complet (restaure le trousseau d'avant la première écriture) :

```
cp -p ~/.ssh/authorized_keys.avant-R1b ~/.ssh/authorized_keys
restorecon -v ~/.ssh/authorized_keys
```

La garde de sauvegarde refuse toute écriture si le trousseau a changé
hors de cette clé depuis la sauvegarde : examiner l'écart, puis renommer
l'ancienne sauvegarde pour qu'une nouvelle soit prise.

## Branches exercées et non exercées (R1b, 2026-09-28)

Démonstrations sur une copie du trousseau dans un répertoire temporaire,
puis sur le vrai fichier (`--check --diff`, exécution réelle, rejeu).
Une branche non exercée n'est pas prouvée (`CLAUDE.md` § Avant d'agir).

| Garde ou étape | Exercé | Non exercé |
|---|---|---|
| G4 (sshd) | passe : réel ; échoue : forcé par `ssh_access_force_sshd_active=false` | sshd réellement arrêté |
| G1 (sentinelle) | passe ; échoue | — |
| G2 (forme) | passe ; échoue : clé RSA, clé tronquée | clé sur plusieurs lignes |
| G3 (empreinte) | passe ; échoue : une lettre changée | — |
| Sauvegarde | création et concordance ; divergence → refus ; `--check` sans sauvegarde → garde ignorée (vrai fichier) | `--check` avec sauvegarde déjà présente |
| Écriture | ajout, sans changement, retrait : sur la copie ; ajout et rejeu `changed=0` : vrai fichier | retrait sur le vrai fichier |
| G5 (après) | passe ; échoue : état « avant » substitué (`ssh_access_force_fingerprints_before`) | altération réelle du fichier pendant l'exécution |
| Mode `0600` | passe | échec |
| G6 (SELinux) | passe : vrai fichier ; échoue : copie sous `/tmp`, sans étiquette par défaut | mauvaise étiquette sous `~/.ssh` |
| Trousseau absent (`create: false`) | — | échec sur fichier absent |
| Connexion réelle | **par clé seule depuis le PC Windows, réussie le 2026-09-28 à 10:20:46 +02:00** (journal sshd, empreinte de la clé Windows) | — |

## Démonstration d'échec forcé

Substituables jamais utilisés en usage réel (`defaults/main.yml`) :
`ssh_access_force_sshd_active`, `ssh_access_force_sshd_enabled` (G4),
`ssh_access_force_fingerprints_before` (G5). Pointer
`ssh_access_authorized_keys_path` sur une copie du trousseau pour
démontrer sans toucher au vrai fichier — G6 y échoue alors par
construction si la copie n'a pas d'étiquette SELinux par défaut (sous
`/tmp`, par exemple).
