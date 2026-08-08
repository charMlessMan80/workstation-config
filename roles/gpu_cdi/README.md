# roles/gpu_cdi

Rôle Ansible ciblant `localhost`. Rend la RTX 4090 utilisable depuis
Podman rootless par **CDI natif** (décision D7, docs/machine-facts.md §
Décisions), SELinux restant en `Enforcing` (décision D8). Résolution
complète, gardes détaillées et arbitrage de confiance sont dans
[`docs/gpu-containers.md`](../../docs/gpu-containers.md) — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Échoue bruyamment si `gpu_cdi_spec_path` est vide, si `nvidia_uvm`
   n'est pas chargé, si `/dev/nvidia-uvm`/`/dev/nvidia-uvm-tools` sont
   absents, si `nvidia-smi -L` ne liste aucun GPU, ou si Podman n'est pas
   en mode rootless.
2. Relève `containers.conf` et le runtime OCI actif **avant** toute
   installation (pour la comparaison de la tâche 6).
3. Télécharge la clé du COPR dans un **trousseau temporaire isolé**
   (jamais le trousseau système), et échoue bruyamment si son empreinte
   ne correspond pas à celle épinglée par la décision D7
   (`gpu_cdi_expected_key_fingerprint`). Purge le trousseau temporaire
   dans tous les cas.
4. Écrit le fichier de dépôt COPR (`gpgcheck=1`, `includepkgs`), puis
   **relit le fichier réellement écrit sur le disque** pour confirmer
   que `gpgcheck=0` n'y apparaît jamais — pas une confiance dans le
   gabarit source.
5. Installe les paquets retenus (`nvidia-container-toolkit`,
   `nvidia-container-toolkit-selinux`).
6. Relève de nouveau `containers.conf` et le runtime OCI actif, et
   échoue bruyamment si l'un des deux a changé — un hook installé mais
   non référencé doit rester inerte, ce n'est jamais supposé.
7. Génère la spécification CDI dans `/etc/cdi/nvidia.yaml` (**jamais**
   `/run/cdi` ou `/var/run/cdi`, `tmpfs`, disparaît au redémarrage) —
   idempotent via `creates:`.
8. Échoue bruyamment si la spécification générée ne référence pas les
   nœuds UVM — une spécification valide mais incomplète est un faux
   positif, pas un succès.
9. Vérifie que la spécification (celle qui vient d'être générée, par
   défaut) n'est pas périmée : tous les chemins de bibliothèques
   référencés existent encore sur l'hôte, et la version encodée
   (`libcuda.so.<version>`) correspond à la version du pilote réellement
   chargée (`/proc/driver/nvidia/version`). Fait partie du run complet.

## Péremption de la spécification CDI (`docs/gpu-containers.md` § Péremption)

Risque documenté : une mise à jour du pilote NVIDIA
(`rpmfusion-nonfree-updates`) remplace les bibliothèques versionnées sur
le disque sans jamais toucher au contenu de `/etc/cdi/nvidia.yaml` déjà
écrit — le conteneur démarre quand même, le GPU devient invisible,
l'application se rabat sur le CPU, **sans erreur**. Deux tags dédiés,
indépendants du run par défaut :

- `--tags verify-cdi-spec` — vérification seule, jamais d'écriture,
  jamais de `become`, ne réveille jamais le GPU. Utilisable seule ou sur
  une copie (`-e gpu_cdi_verify_spec_path=<chemin>`).
- `--tags regen-cdi-spec` — constate d'abord si la spécification réelle
  est périmée (lecture seule) ; si elle est déjà à jour, n'écrit rien
  (`changed=0`, `sha256sum` inchangé). Si elle est périmée ou absente,
  génère dans un fichier de travail privé, le vérifie, et ne l'installe
  dans `/etc/cdi/nvidia.yaml` que si cette vérification passe — jamais
  d'écriture à partir d'un contenu non vérifié.

Aucun déclencheur automatique (udev/systemd/dnf5) : choix délibéré et
argumenté, voir `docs/gpu-containers.md` § Péremption.

## Ce que ce rôle ne fait jamais

- Il ne touche jamais SELinux (aucun `setenforce`, aucun booléen, aucun
  chargement de module `.pp`) — voir `docs/gpu-containers.md` § SELinux
  pour la procédure séparée, fondée sur des refus AVC réels observés
  après l'exécution de ce rôle, pas anticipés.
- Il n'écrit jamais dans `dgpu_disable`, `gpu_mux_mode`, ne démarre ni
  n'active `supergfxd` ni `asus-shutdown`.
- Il n'ajoute aucun dépôt autre que le COPR retenu par D7.
- Il ne désactive jamais `gpgcheck` (assertion dédiée sur le fichier
  réellement écrit, pas sur le gabarit).
- **Il ne redémarre jamais la machine.**
- Il ne lance aucun conteneur — les tests (nominal, deux échecs forcés)
  sont décrits et exécutés séparément dans `docs/gpu-containers.md` § 3,
  après que ce rôle a tourné pour de vrai.
- En mode `--check`, il ne réalise aucune écriture ni aucune vérification
  qui en dépend (gardées par `when: not ansible_check_mode`) : seules les
  gardes et les lectures s'exécutent.

## Utilisation

Toute exécution réelle écrit dans `/etc/` et exige une élévation de
privilège. **[CORRIGÉ le 2026-08-08, COR-1]** Ce compte porte une règle
`NOPASSWD` depuis D9 (2026-08-05, `docs/machine-facts.md` § Décisions) —
`sudo` n'invite plus de mot de passe, `--ask-become-pass` n'est plus
nécessaire. **ÉTAIT** : « Ce compte n'a pas de règle `NOPASSWD` », avec
`--ask-become-pass` dans les deux commandes ci-dessous — faux depuis D9,
jamais corrigé ici bien que `gpu_cdi.yml` porte le commentaire correct
depuis cette date ; trouvé par la revue globale du 2026-08-08
(`docs/review-2026-08.md` § 2.2).

```
ansible-playbook --syntax-check roles/gpu_cdi/gpu_cdi.yml
ansible-playbook --check roles/gpu_cdi/gpu_cdi.yml
ansible-playbook roles/gpu_cdi/gpu_cdi.yml                     # écriture réelle

# Après une mise à jour du pilote NVIDIA — vérification seule (jamais d'écriture) :
ansible-playbook --tags verify-cdi-spec roles/gpu_cdi/gpu_cdi.yml

# Régénère seulement si la vérification ci-dessus a cassé (sinon changed=0) :
ansible-playbook --tags regen-cdi-spec roles/gpu_cdi/gpu_cdi.yml
```

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, en particulier `gpu_cdi_expected_key_fingerprint` (empreinte
épinglée par la décision D7, à ne jamais mettre à jour sans repasser par
une vérification et une nouvelle décision consignée) et
`gpu_cdi_spec_path` (conçue pour la démonstration d'échec forcé
`-e gpu_cdi_spec_path=""`).
