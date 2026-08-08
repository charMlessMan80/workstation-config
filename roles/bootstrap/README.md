# roles/bootstrap

Rôle Ansible ciblant `localhost`. Rend reconstructibles trois décisions
jusqu'ici manuelles, jamais scriptées (revue globale,
[`docs/review-2026-08.md`](../../docs/review-2026-08.md) § 7.1, COR-2) :
**D6** (`power-profiles-daemon` remplace `tuned-ppd`), **D9** (`sudo`
sans mot de passe pour `wheel`), **D10** (dépôt `terra`) —
[`docs/machine-facts.md`](../../docs/machine-facts.md) § Décisions pour
le texte complet de chacune. Deux améliorations délibérées par rapport
à l'installation d'origine, détaillées dans `tasks/main.yml` :

- **D9** vit dans `/etc/sudoers.d/`, jamais `/etc/sudoers` — un fichier
  dédié, appartenant à aucun paquet, survit à toute mise à jour de
  `sudo` (contrairement à `/etc/sudoers`, qui pourrait recevoir un
  `.rpmnew` écartant la règle en silence).
- **D10** n'utilise jamais `--nogpgcheck` — l'empreinte de la clé
  (`AE09157A4DE88B497EA1D5D300CDAB43DE226D6F`, décision de l'opérateur)
  est vérifiée avant tout import ou déploiement, échec bruyant sinon.

## Amorçage — avant que D9 n'existe

Ce rôle a besoin de `become` fonctionnel dès sa première tâche
privilégiée (D9, D10) — sur une machine qui n'a pas encore D9,
`become` demandera un mot de passe :

```
ansible-playbook --ask-become-pass roles/bootstrap/bootstrap.yml
```

Une fois D9 en place (migration effectuée, voir plus bas),
`--ask-become-pass` n'est plus nécessaire — `become` fonctionne sans
invite. Ce rôle ne teste pas les deux chemins en désactivant `NOPASSWD`
sur une machine où il fonctionne déjà (risque jugé disproportionné pour
la preuve) : il s'appuie sur le mécanisme `become` d'Ansible lui-même,
déjà conçu pour fonctionner indifféremment avec ou sans invite de mot
de passe (CLAUDE.md § déterminer ce que l'outil gère déjà) — pas
réimplémenté ici.

## Ce que ce rôle fait, dans l'ordre

1. **D6** : relève les paquets installés, échoue bruyamment si
   `tuned-ppd` et `power-profiles-daemon` coexistent (état non prévu),
   sinon reproduit `dnf swap tuned-ppd power-profiles-daemon
   --allowerasing` (commande d'origine, transaction 7) — idempotent, ne
   rien fait si `power-profiles-daemon` est déjà seul en place.
2. **D9, diagnostic** (toujours exécuté) : confirme que `sudo -n true`
   fonctionne avant toute action, relève si la règle vit déjà dans
   `/etc/sudoers.d/`.
3. **D9, migration** (tag `bootstrap-sudoers`, **jamais joué sans
   invocation explicite** — voir plus bas) : déploie la règle dans
   `/etc/sudoers.d/99-wheel-nopasswd`, validée par `visudo -cf` avant
   d'être écrite ; revérifie `sudo -n true` ; retire l'ancienne ligne de
   `/etc/sudoers`, elle aussi validée par `visudo -cf` avant
   application ; revérifie `sudo -n true` une dernière fois. Chaque
   étape a son message d'échec avec la procédure de retour exacte.
4. **D10** : télécharge le paquet `terra-gpg-keys` depuis un dépôt
   temporaire non approuvé (`dnf download`, aucun privilège, aucune
   vérification de signature en jeu à ce stade), extrait la clé
   (`rpm2cpio`/`cpio`, toujours sans privilège), relève son empreinte
   par inspection à sec (**aucun import dans aucun trousseau**), et
   échoue bruyamment si elle ne correspond pas à l'empreinte épinglée —
   **avant tout déploiement**. Si elle correspond : déploie le fichier
   de clé vérifié et `terra.repo` (`gpgcheck=1`/`repo_gpgcheck=1` dès ce
   fichier), puis installe `terra-release`/`terra-gpg-keys`
   normalement (la clé étant déjà en place, aucune invite).

## Ce que ce rôle ne fait jamais

- Il ne touche jamais `/etc/sudoers` en dehors du tag `bootstrap-sudoers`
  (D9, migration) — jamais par défaut.
- Il n'utilise jamais `--nogpgcheck`.
- Il n'installe jamais de modèle, ne bascule jamais `gpu_mux_mode`, ne
  redémarre jamais la machine, ne modifie jamais SELinux (aucun
  `setenforce`), n'installe jamais `node`/`npm`.
- Il ne teste jamais la garde d'empreinte Terra en corrompant la clé
  réellement déployée — uniquement par substitution de la valeur
  *attendue* (`-e bootstrap_terra_key_fingerprint_expected=...`), sans
  jamais changer ce qui est réellement vérifié.

## La migration D9 — le tag `bootstrap-sudoers`

**C'est l'opération la plus risquée de ce dépôt** : une erreur de
syntaxe dans une règle `sudo` peut verrouiller l'accès `sudo` de cette
machine. Jamais jouée par une exécution normale de ce rôle — invocation
explicite requise :

```
ansible-playbook roles/bootstrap/bootstrap.yml --tags bootstrap-sudoers
```

**Ne pas exécuter sans avoir lu `tasks/main.yml`** (section « D9 §
migration ») et confirmé une voie de récupération indépendante de
`sudo` sur cette machine (TTY racine, console de récupération, session
parallèle déjà privilégiée). La séquence est conçue pour qu'aucune
étape ne laisse la machine sans `sudo` fonctionnel — voir les messages
d'échec de chaque garde pour la procédure de retour exacte à chaque
point d'arrêt possible.

## Utilisation

```
ansible-playbook --syntax-check roles/bootstrap/bootstrap.yml
ansible-playbook --check roles/bootstrap/bootstrap.yml
ansible-playbook roles/bootstrap/bootstrap.yml --skip-tags bootstrap-sudoers   # D6 + D10 uniquement
ansible-playbook roles/bootstrap/bootstrap.yml --tags bootstrap-sudoers       # D9 seule, en connaissance de cause

# Démonstration d'échec forcé (garde d'empreinte Terra, § D10) :
ansible-playbook roles/bootstrap/bootstrap.yml --skip-tags bootstrap-sudoers \
  -e bootstrap_terra_key_fingerprint_expected=0000000000000000000000000000000000000000
```

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire. Substituable pour la démonstration d'échec forcé, sans
jamais changer l'empreinte réellement épinglée :
`bootstrap_terra_key_fingerprint_expected`.
