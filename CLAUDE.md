# Règles persistantes — poste de développement

Ce dépôt décrit et reproduit la configuration d'un poste de développement
personnel (ASUS ROG Zephyrus Duo 16 GX650PY, Fedora 44 KDE/Wayland). Il sera
public. La cible de production réelle est AAP 2.6 sur RHEL 9, gérée
ailleurs — pas depuis ce dépôt.

Chaque règle ci-dessous porte sa raison. Une règle sans motif se fait
« optimiser » par la session suivante : ne retire ni n'assouplis une règle
sans comprendre pourquoi elle a été posée.

## Sourcing des faits

- **Ne jamais affirmer sans source lue sur la machine — ou sans source
  externe faisant autorité, nommée précisément et marquée comme externe.**
  Si un fait n'a été confirmé ni par une commande exécutée dans la session
  en cours, ni par une référence externe recevable, il se marque
  `@VERIF : <où le confirmer>` plutôt que d'être énoncé comme acquis. Une
  référence externe est recevable si c'est de la documentation amont, le
  code source du projet concerné, ou une ABI/spécification officielle
  (ex. `Documentation/ABI/testing/sysfs-platform-asus-wmi` pour la
  sémantique de `gpu_mux_mode`) — nommée avec assez de précision pour être
  retrouvée, et signalée comme externe dans le fait consigné, jamais
  fondue avec une lecture locale. Ce qui reste interdit sans exception :
  la mémoire et la plausibilité. Motif : cette règle, avant amendement,
  interdisait formellement ce qui a pourtant été fait légitimement pour
  fermer le marqueur `gpu_mux_mode` — corriger l'écart entre la règle
  écrite et la pratique correcte plutôt que de laisser la règle se faire
  contourner en silence la prochaine fois.
- **Reproduire une commande, c'est exécuter la commande exacte, pas une
  variante — y compris ses options de formatage et ses redirections.**
  `-o cat`, `--no-pager`, un pipe en bout de chaîne : ce sont elles qui
  déterminent ce qui est observable, pas un détail cosmétique. Motif : une
  variante qui échoue autrement, ou qui réussit, produit un diagnostic qui
  a toutes les apparences du sourcing — code de retour, message, trace —
  mais qui porte sur un autre objet que celui annoncé. C'est un défaut plus
  difficile à repérer qu'une affirmation non sourcée, parce qu'il ressemble
  à une vérification : sur ce dépôt, `journalctl -b0 -g` sans motif et
  `dnf repolist` sans argument ont d'abord été pris pour des reproductions
  valides de `journalctl -b0 -g 'drm\|amdgpu\|nvidia'` et
  `dnf repolist enabled` — deux commandes différentes, deux diagnostics
  faux. Un `-o cat` qui supprime `-- No entries --`, ou un pipe qui
  substitue le code de retour du dernier maillon à celui de la commande
  qu'on croit observer, sont la même erreur sous une autre forme.
- **Un chiffre non sourcé ne se pose pas.** Version, taille, quantité de
  VRAM, priorité de dépôt — chaque nombre dans `docs/machine-facts.md` porte
  la commande qui l'a produit. Motif : un chiffre approximatif recopié de
  mémoire devient indiscernable d'un chiffre vérifié, et personne ne
  revérifie ce qui a l'air déjà écrit noir sur blanc.
- **Une commande sans sortie n'établit rien.** Si une commande renvoie du
  vide, ce n'est pas un résultat « négatif » à consigner tel quel — c'est une
  absence d'information. Vérifier le code de retour, pas seulement stdout : un
  code non nul avec stdout vide n'est pas la même chose qu'un code nul avec
  stdout vide. Motif : dnf5, systemctl et consorts peuvent échouer
  silencieusement sur stdout tout en signalant l'échec sur le code de
  retour ou sur stderr ; ne regarder que stdout fait passer un échec pour un
  fait.
- **Préférer l'échec bruyant au repli silencieux.** Une variable commentée
  dans un fichier de config (ex. `NVreg_PreserveVideoMemoryAllocations`) est
  une variable *absente*, pas une variable dont la valeur par défaut
  s'applique silencieusement sans qu'on l'ait choisie. Motif : confondre
  « commenté » et « valeur par défaut acceptée consciemment » fait perdre la
  trace d'une décision qui n'a jamais été prise.
- **Un marqueur `@VERIF` ne se retire qu'après vérification effective,
  jamais par nettoyage.** Reformuler un paragraphe, migrer un jeton,
  réorganiser une section : aucune de ces opérations n'est une
  vérification, et aucune ne justifie qu'un marqueur disparaisse. Motif :
  la garde équivalente vivait dans le préambule de `docs/machine-facts.md`
  et a disparu par effet de bord lors de la migration du jeton
  `À VÉRIFIER` vers `@VERIF` — ce paragraphe contenait l'ancien jeton, donc
  réécrit avec le reste, sans que quiconque ait décidé de retirer la garde
  elle-même. Une règle qui ne vit que dans un fichier de faits est
  vulnérable à toute réécriture de ce fichier ; sa place est ici, dans un
  fichier de règles, pas dans celui qu'elle est censée contraindre.
  Corollaire : un compteur de marqueurs qui baisse doit s'expliquer par des
  vérifications faites, pas par des reformulations — si l'explication
  manque, traiter la baisse comme suspecte jusqu'à preuve du contraire.

## Avant d'agir

- **Avant d'écrire du code ou de la configuration, déterminer ce que l'outil
  gère déjà lui-même.** Consulter l'aide intégrée, la doc du produit ou son
  comportement par défaut avant de réimplémenter une vérification, une
  interdiction ou un réglage qu'il applique déjà nativement. Motif : la
  sur-couverture coûte autant que le manque — réécrire ce qu'un
  installateur fait déjà, ouvrir un port purement local, coder à la main une
  garde que le produit impose lui-même, sont des efforts gaspillés qui
  ajoutent une source de divergence à maintenir plutôt que d'en retirer une.
- **Vérifier qu'une commande annoncée comme passive l'est réellement, avant
  de l'exécuter.** Une lecture peut déclencher un effet de bord (import de
  clé, écriture de cache, invite interactive) sans le signaler à l'avance.
  Motif : sur ce poste, `dnf repoinfo terra` a déclenché une invite d'import
  de clé GPG alors qu'elle était censée être une simple consultation —
  l'effet de bord doit être anticipé, ou à défaut vérifié immédiatement
  après coup, pas découvert plus tard au milieu d'une série de commandes.
- **Le gestionnaire `command-not-found` de dnf installe sur réponse
  machinale.** Tester `command -v X >/dev/null && X ...` avant d'invoquer un
  binaire dont la présence n'est pas confirmée. Motif : `supergfxctl` a été
  installé par ce mécanisme lors de l'inventaire initial sans jamais avoir
  été un choix de conception — le prompt interactif de PackageKit se
  franchit facilement sans y prêter attention.
- **Un livrable borné par un prompt s'arrête et demande validation.** Ne pas
  enchaîner sur la suite d'une tâche non demandée une fois le périmètre
  annoncé atteint. Motif : sur une machine de dev sans garde-fou
  d'entreprise, l'enchaînement silencieux est la façon la plus rapide de
  sortir du périmètre convenu.
- **Toute garde se démontre dans les deux sens.** Une contrainte annoncée
  (ex. « dgpu_disable doit rester à 0 ») se vérifie à la fois par le cas
  nominal (la garde n'interfère pas quand tout va bien) et par l'échec forcé
  (la garde déclenche bien le bon message quand la condition est violée).
  Motif : une garde qui n'a été testée que dans le sens qui marche n'est pas
  une garde, c'est une supposition.

## Dépôt public (D4)

- **Interdits, sans exception : secrets (clés privées, jetons, mots de
  passe), données d'entreprise, adressage d'infrastructure interne (IP,
  sous-réseaux, VLAN), noms de serveurs ou domaines professionnels,
  identifiants de comptes de service. Acceptés explicitement : hostname et
  nom d'utilisateur de ce poste personnel, identité de l'auteur dans
  l'historique git.** Voir décision D4 (amendée le 2026-08-04) dans
  `docs/machine-facts.md`. Motif : la formulation d'origine de D4 (« aucune
  adresse interne, aucun nom d'hôte réel ») était inapplicable en pratique
  — violée dès le premier commit sans que la revue le détecte, parce que le
  hostname et les chemins `$HOME` sont indissociables d'un inventaire
  sourcé par des commandes réelles. Expurger ces identifiants dégraderait
  la traçabilité sans rien protéger, puisque l'identité complète de
  l'auteur figure déjà dans chaque commit.
- **L'adressage IP reste interdit en dur, y compris en RFC 1918.** Une IP
  privée (`10.x`, `172.16-31.x`, `192.168.x`) n'est pas neutre au sens de
  D4 : elle décrit un réseau réel, même local. Toute valeur de ce genre
  passe par une variable avec un exemple RFC 5737 (`192.0.2.0/24`,
  `198.51.100.0/24`, `203.0.113.0/24` — plages réservées à la
  documentation) dans `defaults/`, jamais en dur dans un fichier versionné
  — voir `roles/recovery/defaults/main.yml` (`recovery_remote_host`) pour
  le patron à reproduire. Motif : contrairement au hostname ou au nom
  d'utilisateur de ce poste, une adresse IP décrit une topologie réseau,
  potentiellement partagée avec d'autres machines non couvertes par
  l'acceptation explicite ci-dessus — la distinction n'est pas cosmétique.

## Matériel spécifique — GPU / MUX

- **`dgpu_disable` doit rester à `0`.** Le pilote refuse la commutation MUX
  si la dGPU est désactivée. Motif : confondre `dgpu_disable` (coupe la
  carte) avec `gpu_mux_mode` (choisit à quelle carte le panneau est câblé)
  coupe CUDA en croyant seulement changer d'affichage — les deux attributs
  vivent sous `/sys/class/firmware-attributes/asus-armoury/attributes/` et
  se ressemblent.
- **Après écriture de `gpu_mux_mode`, la relecture renvoie l'ancienne valeur
  jusqu'au redémarrage.** La confirmation avant reboot est le champ
  `pending_reboot` (au niveau `attributes/`, pas un fichier séparé sous
  `gpu_mux_mode/`), pas `current_value`. Motif : lire `current_value` juste
  après une écriture et le trouver inchangé n'est pas un échec de l'écriture
  — c'est le comportement attendu tant que la machine n'a pas redémarré ;
  paniquer ou réessayer l'écriture sur cette seule base est une erreur de
  méthode observée sur ce poste.

## `docs/machine-facts.md`

- **Se met à jour en dernier dans une série.** Motif : documenter un état
  avant que toutes les actions de la série soient terminées revient à
  documenter un état qui n'existe pas encore.
- **Ne pas y consigner ce qui se lit dans git** (état de merge, position par
  rapport à `origin`, historique de commits). Motif : git est déjà la source
  de vérité pour son propre état ; le dupliquer dans un fichier statique crée
  une deuxième source qui se périme dès le commit suivant.
- **Marquer l'historique plutôt que l'effacer** — `[RETIRÉ le …]`, `ÉTAIT`,
  dates explicites sur chaque décision retirée ou remplacée. Motif : une
  décision retirée sans trace laisse quelqu'un (humain ou session future)
  redécouvrir le même piège et reprendre la même décision, ou pire,
  l'inverse sans connaître le motif d'origine.
- **`git diff --stat` avant tout commit touchant ce fichier : une édition
  qui ajoute ne doit jamais réduire un fichier.** Motif : ce fichier grossit
  par nature (chaque série ajoute des faits datés) ; une réduction de
  taille lors d'une édition censée être additive signale presque toujours
  une suppression accidentelle plutôt qu'un nettoyage voulu.

## Chaîne Ansible

- L'`ansible-core` système sert à l'édition, pas à l'exécution ni à la
  vérification — voir décision D3 dans `docs/machine-facts.md`. Ne pas
  valider une construction Ansible comme correcte simplement parce que
  l'`ansible-core` local (plus récent que la cible AAP 2.6) l'accepte.
