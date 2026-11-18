# roles/cdi_login_notice

Rôle Ansible ciblant `localhost`. Dépose
`~/.bashrc.d/95-cdi-login-notice.sh`, qui, **dans les shells interactifs
seulement**, écrit sur la **sortie d'erreur** un avertissement quand le
**dernier résultat connu** de `local-ai-cdi-verify.service`
(`roles/local_ai/`, garde CDI horaire) dans le journal utilisateur est un
échec. Canal voulu : un message à la connexion, qui fonctionne sans session
graphique (SSH). La notification de bureau existante
(`local-ai-cdi-verify-notify.service`) n'est ni retirée ni modifiée.

Décision du propriétaire du 2026-11-18, **non numérotée** (D28 ne couvre
`~/.bashrc.d/` que pour Android) : option A de la résolution du livrable
12, rôle dédié. Rattachement au point ouvert « chaîne d'alerte » :
`docs/gpu-containers.md` § 9.9.

## Ce que le message dit, et ce qu'il ne peut pas dire

Sans « linger » (décision du propriétaire), la garde ne tourne que
lorsqu'une session existe, et le gestionnaire utilisateur s'arrête entre
deux sessions. Il n'y a donc pas d'**état courant** à afficher : le
fichier lit le **dernier résultat terminé** dans le journal (qui survit aux
redémarrages) et l'écrit comme tel, daté avec son décalage. À la première
connexion après une absence, la minuterie peut rattraper un passage dans la
même seconde (`Persistent=true`) : le message porte alors sur le passage
**précédent**, pas sur celui qui tourne.

Sur échec :

- la date du dernier résultat ;
- « dernier passage lu dans le journal utilisateur, PAS un état courant » ;
- depuis quand : le premier échec après la dernière réussite, et la date
  de cette réussite (ou « aucune réussite dans le journal conservé ») ;
- `journalctl --user -u local-ai-cdi-verify.service` ;
- la régénération prescrite (`docs/gpu-containers.md` § 9.3) :
  `ansible-playbook roles/gpu_cdi/gpu_cdi.yml --tags regen-cdi-spec` ;
- le renvoi à `docs/gpu-containers.md` § 9.9 pour la consignation (témoin,
  `sha256sum`, avertissements — `CLAUDE.md` § Matériel spécifique).

Sur réussite : rien.

Sans aucun résultat lisible (journal vide pour la période, `journalctl` en
échec, dépassement du délai de 2 s) : **« INCONNU »**, avec le code de
retour, et la mention que ce n'est pas un état « à jour ». Choix motivé :
ici, le silence a la forme d'une réussite ; se taire ferait passer une
absence d'information pour un fait (`CLAUDE.md`, « Préférer l'échec
bruyant au repli silencieux », « Une commande sans sortie n'établit
rien »). Coût assumé : sur un poste neuf, ce message apparaît à chaque
shell interactif jusqu'au premier passage horaire de la garde.

## Ce que ce rôle fait

1. Asserte Fedora 44.
2. **Vérifie par l'effet** que l'unité surveillée est déployée
   (`stat ~/.config/systemd/user/local-ai-cdi-verify.service`) —
   dépendance réelle sur `roles/local_ai/`, jamais supposée depuis l'ordre
   de `site.yml`.
3. **Garde** : `~/.bashrc` porte la boucle native de chargement de
   `~/.bashrc.d/` (même garde qu'`android_env`, répétée : ce rôle ne
   suppose pas qu'`android_env` a tourné).
4. Crée `~/.bashrc.d/` s'il manque, dépose le fichier (`0644`), écrasé
   intégralement à chaque exécution.
5. **Assertions de référence** sur le journal (lecture directe par
   `journalctl`, indépendante du fichier déposé) : les résultats sur
   lesquels portent les preuves existent bien.
6. **Preuves par l'effet**, dans des shells réels (`env -i`, seul `HOME`
   conservé, comme `android_env`) :
   1. interactif, référence sur un échec réel → message complet sur la
      sortie d'erreur, sortie standard vide ;
   2. interactif, référence sur une réussite réelle → aucun message ;
   3. **non interactif** chargeant `~/.bashrc` (`BASH_ENV`, drapeaux sans
      `i`, le cas `ssh hôte 'commande'`), référence sur l'échec → aucune
      sortie, ni standard ni d'erreur ; témoin sous `bash -x` : le fichier
      a bien été chargé et sa garde évaluée ;
   4. interactif, référence avant toute entrée de l'unité → « INCONNU » ;
   5. interactif, branche d'échec parcourue → aucune variable ni fonction
      `_cdi_ln_*` ne reste dans le shell.
7. Rapporte ce qu'un shell interactif affiche aujourd'hui (sans date de
   référence) et la durée de ce shell — rapporté, pas asserté.

## Ce que ce rôle ne fait jamais

- N'écrit ni dans `~/.bashrc`, ni dans `~/.bash_profile`, ni dans un autre
  fichier de `~/.bashrc.d/`.
- Aucune élévation (`become`), aucun `systemctl start/stop/enable`, aucun
  `daemon-reload`, ne touche pas aux unités `local-ai-cdi-verify*`.
- Le fichier déposé ne lance jamais la garde, n'appelle ni
  `ansible-playbook`, ni `systemctl`, ni `nvidia-smi` : il ne lit que le
  journal. Pas de `set -e`, pas d'`exit` ; tout le corps tourne dans un
  sous-shell.

## Date de référence

```
CDI_LOGIN_NOTICE_AS_OF='2026-11-17T21:30:00+01:00' bash -i
```

donne « le dernier résultat connu à cette date ». Toute forme
d'horodatage de `systemd.time(7)` est acceptée (`--until` de
`journalctl`). Une valeur invalide fait échouer `journalctl` : le fichier
affiche alors « INCONNU » avec le code de retour.

## Preuves : dépendance au journal de ce poste

Les preuves 1, 2 et 5 portent sur des entrées réelles du journal
utilisateur, relues le 2026-11-18 (`defaults/main.yml`). **Sur un poste
neuf, ou après rotation du journal, ces entrées n'existent pas** : les
assertions de référence (étape 5) échouent **avant** les preuves, avec le
message « RÉFÉRENCE ABSENTE, PREUVES NON EXERCÉES » — ce n'est pas un
défaut du fichier déposé. Deux sorties :

- relire `journalctl --user -u local-ai-cdi-verify.service` et ajuster
  `cdi_login_notice_proof_*` sur un échec et une réussite réels ;
- ou jouer avec `-e cdi_login_notice_journal_proofs=false` : le fichier est
  déposé, les preuves 1, 2 et 5 sont sautées et **un message le signale**
  (jamais en silence). Les preuves 3 et 4 restent jouées : elles ne
  supposent aucune entrée.

Conséquence pour `site.yml` sur une machine neuve : ce rôle s'arrêtera sur
les références tant que la variable n'est pas mise à `false`.

## Utilisation

```
ansible-playbook roles/cdi_login_notice/cdi_login_notice.yml --check --diff
ansible-playbook roles/cdi_login_notice/cdi_login_notice.yml
ansible-playbook site.yml --tags cdi_login_notice
```

**Ce que `--check` couvre** : les gardes de lecture (Fedora, unité
présente, boucle de `~/.bashrc`), les assertions de référence sur le
journal, et le `diff` du gabarit si `~/.bashrc.d/` existe déjà. Il ne
couvre **aucune** preuve par l'effet (le fichier n'est pas déposé).

**Retour arrière** : supprimer `~/.bashrc.d/95-cdi-login-notice.sh`, rien
d'autre.

## Gardes du rôle : démontrées dans les deux sens (2026-11-18, L13b)

Chaque garde a été jouée en nominal (exécutions sans `-e`) et forcée en
échec, par un seul `-e` à chaque fois. Le fichier déployé n'a pas été
réécrit (`stat` identique avant et après chaque exécution). Sorties :
`docs/machine-facts.md` § L13.

| `-e` | Résultat observé |
|---|---|
| `cdi_login_notice_unit_file=/nonexistent` | arrêt sur la garde de l'unité, avant tout dépôt, `failed=1` |
| `cdi_login_notice_bashrc_path=<copie sans la boucle>` | arrêt sur la garde de la boucle (`grep rc=1`), avant tout dépôt, `failed=1` |
| `cdi_login_notice_proof_failure_as_of=2026-08-08T12:00:00+02:00` | arrêt sur les références au journal, « RÉFÉRENCE ABSENTE, PREUVES NON EXERCÉES », **après** la tâche du gabarit (`ok`, pas `changed`), `failed=1` |
| `cdi_login_notice_journal_proofs=false` | aucun échec ; message « PREUVES 1, 2 ET 5 NON EXERCÉES » ; preuves 3 et 4 jouées et vertes |

## Ce que les preuves ne couvrent pas

- Une vraie connexion SSH (interactive ou avec commande) : les preuves
  utilisent `bash -i` et `BASH_ENV`, pas `sshd`. Le chargement de
  `~/.bashrc` par `ssh hôte 'commande'` est une mesure du pilote
  (2026-11-18), pas une preuve de ce rôle.
- Le shell de connexion (`~/.bash_profile`), `scp`/`sftp`, `git` sur SSH.
- Le dépassement du délai (`timeout`, rc=124), une valeur invalide de
  `CDI_LOGIN_NOTICE_AS_OF`, l'absence de `journalctl`.
- La branche « aucune réussite dans le journal conservé ».
- Un passage de la garde en cours au moment de l'ouverture du shell.
