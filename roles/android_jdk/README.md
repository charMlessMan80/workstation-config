# roles/android_jdk

Rôle Ansible ciblant `localhost`. Premier rôle d'une chaîne de build
Android CLI destinée au dépôt `glass-hud` (distinct, jamais référencé
au-delà de cette phrase) — installe le JDK complet, rien d'autre. Choix
du JDK 25 plutôt que l'ouverture d'une nouvelle surface
d'approvisionnement (Adoptium Temurin, JDK 17/21) : voir
`docs/machine-facts.md` § Décisions pour le fait sourcé et sa réserve.

Modèle pris pour ce rôle : [`roles/recovery/`](../recovery/) — patron le
plus proche (assertion en première tâche, `package_facts`, vérification
par l'effet plutôt que par le code de retour de la commande qui le
produit, `check_mode: false` sur les tâches de lecture seule pour qu'une
simulation rapporte l'état réel plutôt qu'un faux silence). Écart
assumé : `recovery` n'installe jamais rien lui-même (il asserte une
présence et échoue sinon) ; ce rôle installe réellement un paquet — le
patron d'installation idempotente vient de
[`roles/editor/`](../editor/) (`ansible.builtin.dnf`, `state: present`,
`disablerepo: terra`, `when: not ansible_check_mode`).

## Ce que ce rôle fait

1. Asserte que le poste est Fedora 44 — échoue bruyamment sinon.
2. Relève les paquets Java déjà installés (`package_facts`).
3. Installe `java-25-openjdk-devel` (dépôts déjà activés, `terra`
   explicitement désactivé pour cette tâche — aucune nouvelle surface
   d'approvisionnement).
4. **Vérifie par l'effet, pas par le code de retour de l'installation** :
   exécute réellement `javac -version` au chemin attendu et asserte que
   la version majeure retournée correspond à celle visée. Une
   installation qui « réussirait » sans que `javac` soit utilisable à ce
   chemin fait échouer le rôle bruyamment, avec le message exact.
5. Relève l'état `alternatives` après coup et le rapporte.

## Ce que ce rôle ne fait jamais

- Il n'installe aucun SDK Android, aucun `cmdline-tools`, aucun Gradle.
- Il ne modifie aucune variable d'environnement, aucun fichier de shell
  (`~/.bashrc`, `~/.profile`, `/etc/profile.d/`).
- Il n'ouvre aucune surface d'approvisionnement au-delà des dépôts
  `fedora`/`updates` déjà activés — jamais `terra` (désactivé
  explicitement pour sa seule tâche d'installation).
- Il ne touche à aucune entrée `alternatives` existante — il les relève,
  il ne les modifie jamais.
- Il ne redémarre jamais la machine (sans rapport avec ce paquet de
  toute façon, aucun service ni module noyau en jeu).

## Utilisation

```
ansible-playbook --syntax-check roles/android_jdk/android_jdk.yml
NO_COLOR=1 ~/.venvs/ansible-lint/bin/ansible-lint roles/android_jdk
ansible-playbook --check roles/android_jdk/android_jdk.yml
ansible-playbook roles/android_jdk/android_jdk.yml
```

Démonstration d'échec forcé de la garde post-installation (§ Ce que ce
rôle fait, point 4), sans jamais toucher au `javac` réel :
```
ansible-playbook roles/android_jdk/android_jdk.yml \
  -e android_jdk_javac_path_expected=/chemin/qui/n/existe/pas
```

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, y compris la variable substituable pour la démonstration
d'échec forcé (`android_jdk_javac_path_expected`, sans jamais changer le
chemin réellement utilisé par le rôle lui-même).
