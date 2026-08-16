# roles/android_sdk

Rôle Ansible ciblant `localhost`. Deuxième rôle d'une chaîne de build
Android CLI destinée au dépôt `glass-hud` (distinct, jamais référencé
au-delà de cette phrase) — installe les command-line tools du SDK
Android dans le domaine utilisateur et accepte les licences, rien
d'autre.

Deux rôles séparés, décision du pilote : ce rôle installe, un rôle
ultérieur (`android_env`) câblera l'environnement. Motif : `sdkmanager`
n'a pas besoin d'`ANDROID_HOME`/`ANDROID_SDK_ROOT` (résolution par
`--sdk_root` explicite, livrable 7 de glass-hud) — seul l'usage
interactif en a besoin, et un rôle qui pose des fichiers dans le
domaine utilisateur n'a pas la même réversibilité qu'un rôle qui édite
un fichier de configuration personnel.

Modèle : [`roles/android_jdk/`](../android_jdk/) pour le squelette
(assertion Fedora en première tâche, vérification par l'effet plutôt
que par le code de retour, `check_mode: false` sur les tâches de
lecture seule). Écart assumé : `android_jdk` installe un paquet système
(`become: true`) ; ce rôle télécharge et extrait une archive dans le
domaine utilisateur sans aucune élévation nulle part — le patron de
téléchargement vérifié avant écriture vient de
[`roles/gpu_cdi/`](../gpu_cdi/) (clé GPG du COPR), adapté ici avec le
paramètre `checksum` natif de `get_url` plutôt qu'une vérification
manuelle après coup.

## Ce que ce rôle fait

1. Asserte que le poste est Fedora 44 — échoue bruyamment sinon.
2. **Vérifie par l'effet** que `javac` est réellement utilisable
   (`javac -version`) — dépendance réelle sur `roles/android_jdk/`,
   jamais supposée depuis l'ordre de `site.yml`.
3. Relève si les command-line tools sont déjà en place (idempotence).
4. Si absents : télécharge l'archive dans un répertoire temporaire
   isolé, **empreinte sha256 vérifiée avant toute extraction**
   (paramètre natif `checksum` de `get_url` — le fichier n'atteint sa
   destination que si l'empreinte correspond), extrait, puis déplace
   `cmdline-tools/` vers `cmdline-tools/latest/` (arborescence que
   `sdkmanager` attend réellement, établi au livrable 7 de glass-hud).
5. Vérifie par l'effet que `sdkmanager` est réellement en place à ce
   chemin.
6. Relève si les licences sont déjà acceptées (fichier de porte
   `licenses/android-sdk-license`).
7. Si non : accepte les licences (`yes | sdkmanager --licenses
   --sdk_root=...`, jamais `ANDROID_SDK_ROOT` exporté — `sdkmanager`
   l'ignorerait, livrable 7 de glass-hud). **Les sept accords proposés
   par `sdkmanager --licenses` à ce jour sont tous acceptés, aucun
   filtrage** — décision du pilote, `docs/machine-facts.md` § Décisions
   (D27) pour le motif complet :
   - `android-googletv-license`
   - `android-googlexr-license`
   - `android-sdk-arm-dbt-license`
   - `android-sdk-license` (fichier de porte de ce rôle, § ci-dessus)
   - `android-sdk-preview-license`
   - `google-gdk-license`
   - `mips-android-sysimage-license`

   Au moins trois de ces sept ne concernent aucun composant que ce
   poste utilisera (Google TV, systèmes MIPS, Google GDK) — accepté
   quand même, pas filtré : `sdkmanager` ne permet pas un refus
   sélectif adressable en une passe, et une liste d'exclusion serait
   elle-même un compteur périmable à chaque composant futur.
8. **Vérifie par l'effet, jamais par le code de retour** : `sdkmanager
   --licenses` renvoie 0 aussi bien quand rien n'est accepté que quand
   tout l'est (livrable 7) — ce rôle relit `licenses/` après coup et
   échoue bruyamment si le fichier de porte est absent ou vide.

## Ce que ce rôle ne fait jamais

- Il n'installe ni `platform-tools`, ni aucune `platform`, ni aucun
  `build-tools`, ni Gradle — le socle uniquement, la cible d'API
  appartient à `glass-hud`, pas à ce dépôt.
- Il ne modifie aucune variable d'environnement, aucun fichier de
  shell (`~/.bashrc`, `~/.profile`, `/etc/profile.d/`,
  `~/.config/environment.d/`).
- Il ne télécharge ni n'extrait jamais directement dans
  `android_sdk_root` — toujours via un répertoire temporaire isolé,
  déplacé seulement une fois l'empreinte vérifiée.
- Il ne relance jamais l'acceptation des licences ni le téléchargement
  si le fichier de porte / `sdkmanager` sont déjà en place. **Limite
  connue, consignée, non corrigée ici** : cette garde ne compare rien
  au manifeste de licences réellement requis aujourd'hui par
  `sdkmanager` — si Google ajoute un jour une licence pour un
  composant nouveau, ce fichier de porte existera toujours et
  l'acceptation ne sera jamais rejouée. Détail complet et conséquence
  observable : `docs/machine-facts.md` § Points ouverts (2026-08-16,
  D27).

## Utilisation

```
ansible-playbook --syntax-check roles/android_sdk/android_sdk.yml
NO_COLOR=1 ~/.venvs/ansible-lint/bin/ansible-lint roles/android_sdk
ansible-playbook --check roles/android_sdk/android_sdk.yml
ansible-playbook roles/android_sdk/android_sdk.yml
```

**Ce que `--check` couvre réellement** : les deux assertions initiales
(Fedora 44, `javac` utilisable) seulement — un rôle qui télécharge et
extrait ne peut pas simuler grand-chose de plus. Aucune tâche de
téléchargement, extraction, déplacement ou acceptation de licence ne
s'exécute sous `--check` (`when: not ansible_check_mode` sur chacune),
et les deux assertions finales (sdkmanager en place, licences
acceptées) sont donc elles-mêmes sautées sous `--check` — un `--check`
vert sur ce rôle ne prouve rien sur ces deux points.

Démonstration d'échec forcé de la garde d'empreinte, sans jamais
toucher à l'archive réelle :
```
ansible-playbook roles/android_sdk/android_sdk.yml \
  -e android_sdk_cmdline_tools_sha256_expected=0000000000000000000000000000000000000000000000000000000000000000
```

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, y compris la variable substituable pour la démonstration
d'échec forcé (`android_sdk_cmdline_tools_sha256_expected`, sans
jamais changer l'empreinte réellement utilisée par le rôle lui-même).

## Mise à jour de l'archive command-line tools

**Ce rôle a une date de péremption inconnue** (`docs/machine-facts.md`
§ Points ouverts, 2026-08-16, D27) : l'URL et l'empreinte sont épinglées
en dur (`android_sdk_cmdline_tools_build`, `..._url`, `..._sha256`,
`defaults/main.yml`) — volontairement, une empreinte non épinglée ne
protège de rien. Quand Google publiera un nouveau build, l'URL
actuelle rendra probablement 404 et ce rôle échouera bruyamment sur un
redéploiement complet — **comportement attendu**, pas un défaut.
Procédure pour en sortir :

1. **Ouvrir `https://developer.android.com/studio`, section « Command
   line tools only ».** Ne pas s'arrêter à un résumé automatique de
   cette page (outil de résumé, recherche web) : **deux fois** au
   livrable 9, un tel résumé a renvoyé un hôte de téléchargement
   (`edgedl.me.gvt1.com`) différent de celui réellement présent dans
   le HTML servi. Récupérer le HTML brut et le lire directement :
   ```
   curl -s -A "Mozilla/5.0" https://developer.android.com/studio -o page.html
   ```
2. **Trouver la ligne du tableau « Linux »**, pas une autre plateforme :
   ```
   grep -n "commandlinetools-linux" page.html
   ```
   La ligne `<tr><td>Linux</td>...` porte, dans l'ordre, le nom de
   fichier (contient le numéro de build), la taille, puis l'empreinte
   sha256 — dans la cellule `<td>` suivante, pas ailleurs sur la page
   (plusieurs empreintes y coexistent, une par plateforme). **L'URL de
   téléchargement réelle** est le `href` du bouton « Download Android
   Command Line Tools for … » associé à ce même `data-modal-dialog-id`
   (`sdk_linux_download`) — commence par
   `https://dl.google.com/android/repository/`, jamais
   `edgedl.me.gvt1.com` (§ ci-dessus).
3. **Télécharger et recroiser l'empreinte soi-même, avant de faire
   confiance à la page** :
   ```
   curl -sS -o commandlinetools-linux-<build>_latest.zip <url dl.google.com trouvée ci-dessus>
   sha256sum commandlinetools-linux-<build>_latest.zip
   ```
   Comparer à l'empreinte lue à l'étape 2 — **identique obligatoire**
   avant de poursuivre. En cas d'écart : arrêt, ne pas poursuivre, ne
   pas supposer une simple différence de mise en forme.
4. **Mettre à jour `defaults/main.yml`** :
   `android_sdk_cmdline_tools_build` (le numéro trouvé à l'étape 2) et
   `android_sdk_cmdline_tools_sha256` (l'empreinte recroisée à l'étape
   3). `android_sdk_cmdline_tools_url` se recalcule seul (gabarit sur
   le premier).
5. **Point de vigilance propre à ce rôle, pas seulement « changer deux
   variables »** : la garde d'idempotence de téléchargement
   (`android_sdk_sdkmanager_stat.stat.exists`, § Ce que ce rôle ne
   fait jamais) ne compare **que la présence** de
   `cmdline-tools/latest/bin/sdkmanager`, jamais sa version — sur un
   poste où l'ancienne archive est déjà installée, changer seulement
   les variables ci-dessus **ne déclenche aucun retéléchargement** : le
   rôle rapporterait un succès sans rien avoir mis à jour. Pour forcer
   une mise à jour réelle sur un poste existant :
   ```
   rm -rf ~/Android/Sdk/cmdline-tools
   ansible-playbook roles/android_sdk/android_sdk.yml
   ```
   Les licences déjà acceptées ne sont pas affectées par cette
   suppression ciblée (`licenses/` n'est pas touché) — l'acceptation
   ne sera donc, à raison, pas rejouée non plus.
6. **Rejouer la chaîne de validation** (§ Utilisation ci-dessus) avant
   de considérer la mise à jour terminée : `--syntax-check`,
   `ansible-lint`, `--check`, puis exécution réelle.
