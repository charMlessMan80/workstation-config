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
- **`CLAUDE.md` porte des règles, jamais des faits à vérifier — il
  contient zéro marqueur `@VERIF` par construction, et tout comptage de
  validation (`grep -c '@VERIF'`) doit l'exclure.** Les occurrences du
  jeton `@VERIF` dans ce fichier appartiennent aux règles qui le
  définissent ou le documentent (cette règle elle-même, par exemple) —
  ce ne sont pas des marqueurs actionnables sur un fait de ce dépôt.
  Motif : compté par erreur comme marqueurs actionnables dans un rapport
  de livrable (2026-08-06) — précisément les occurrences documentant la
  migration du jeton `À VÉRIFIER` vers `@VERIF` (§ plus bas) ont gonflé
  le compte qu'elles décrivent. C'est le même défaut que celui déjà
  identifié pour la garde équivalente (voir la règle sur le retrait d'un
  marqueur uniquement après vérification effective) revenu par une autre
  porte — le comptage, pas le retrait, cette fois.
- **Le jeton `@VERIF` ne s'écrit jamais nu hors d'un marqueur réel —
  en prose, il s'écrit entre accents graves ou par périphrase.** Un
  marqueur réel a la forme `@VERIF : <où le confirmer>` et porte un fait
  précis à vérifier ; toute autre mention du jeton (expliquer la règle,
  renvoyer à un marqueur ailleurs, décrire un comptage) doit soit
  l'entourer d'accents graves (`` `@VERIF` ``, déjà une forme de mise à
  distance mais encore comptée par `grep -c` — à réserver aux contextes
  où le fichier lui-même n'est pas celui qu'on compte), soit le nommer
  sans l'écrire (« le jeton de vérification », « le marqueur »). Motif :
  la réponse retenue au constat précédent (2026-08-06 : le compte
  s'auto-gonfle dès qu'on écrit le jeton dans le fichier qu'il décrit)
  traitait le symptôme — ne pas figer de chiffre — sans traiter la
  cause, écrire le jeton nu en prose. `grep -c '@VERIF'` doit rester une
  preuve ; un compteur qui inclut ses propres mentions ne mesure plus
  que lui-même.
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
  **Troisième cas de la série (2026-08-08)** : `git show HEAD --numstat`,
  prescrite comme preuve d'additivité pendant une dizaine de livrables,
  mélange les données qu'elle est censée contrôler (le `--numstat`) avec
  des métadonnées sans rapport (le message de commit, inclus par
  défaut par `git show`) — une ligne de prose se parsant par hasard en
  trois champs (`docs/repositories.md § 8.`) a produit un faux positif
  (`RÉDUIT: 8.`) que rien ne distinguait d'un vrai. **Forme correcte,
  qui isole les données du `diff` sans les métadonnées du commit :**
  ```
  git diff --numstat HEAD^ HEAD | awk '$2>$1 {print "RÉDUIT: " $3}'
  ```
  À utiliser désormais partout où une preuve d'additivité est demandée
  — `git show ... --numstat` ne doit plus servir de preuve de contrôle
  sans être immédiatement suivi d'une vérification équivalente à
  celle-ci.
  **Quatrième cas de la série (2026-08-08, revue globale)** : `dnf
  repolist enabled`, citée juste au-dessus comme LA forme correcte par
  contraste avec `dnf repolist` seul, **échoue elle-même silencieusement**
  sur ce système — dnf5 (`5.4.2.1`) traite l'argument positionnel
  `enabled` comme un motif de nom de dépôt (aucun dépôt ne s'appelle
  ainsi), pas comme le mot-clé de filtrage `dnf4` : sortie vide, code
  de retour `0`. Forme qui fonctionne réellement :
  ```
  dnf repolist --enabled
  ```
  **Leçon distincte du défaut lui-même** : les exemples qui illustrent
  une règle doivent être vérifiés au même titre que la règle — on relit
  une règle avant de l'appliquer, presque jamais l'exemple qui
  l'accompagne, en confiance qu'il a été vérifié une fois pour toutes.
  Celui-ci ne l'avait pas été, ou l'a été sur une version de `dnf`
  antérieure sans être revérifié depuis ; il est resté faux dans ce
  fichier sans que personne ne le remarque jusqu'à ce que la revue
  globale du 2026-08-08 le reproduise (`docs/review-2026-08.md` § 2.3).
- **Arbitrage entre reproduction fidèle et vérification de passivité,
  quand une commande a un effet de bord déjà documenté.** La
  reproduction fidèle prime pour le diagnostic — mais une commande dont
  l'effet de bord est **déjà documenté** (ex. l'invite d'import de clé
  GPG sur une commande touchant Terra) se reproduit **avec sa garde**
  (`--assumeno`, `--disablerepo=`), en signalant explicitement que la
  reproduction est de ce fait inexacte et en quoi. Motif : reproduire
  fidèlement un effet de bord déjà connu ne l'établit pas mieux, ça le
  répète — la fidélité de reproduction sert à découvrir ou confirmer un
  écart, pas à retraverser un risque déjà cartographié. Reproduite deux
  fois sur ce dépôt (`dnf repoinfo terra`, puis `dnf repoquery --file
  '*/nvidia-ctk'` sans garde alors que l'effet était déjà documenté) —
  sans conséquence les deux fois, par chance de l'environnement non
  interactif, pas par garde délibérée la seconde fois.
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
- **Un fait mesuré porte sa date ; une mesure ancienne ne sert pas de
  ligne de base pour un changement récent.** Une valeur relevée à un
  instant donné n'atteste que cet instant — la réutiliser comme « état
  juste avant » un événement plus tardif sans revérifier la date
  introduit une erreur silencieuse, même si la valeur elle-même a été lue
  correctement. Motif : c'est le troisième mode d'échec distinct identifié
  dans cette série, à consigner avec les deux autres parce qu'ils se
  ressemblent — chacun **a l'apparence du sourcing** tout en portant sur
  autre chose que l'objet examiné : la **commande substituée** (conclure
  sur une commande en en exécutant une autre — `journalctl -b0 -g` sans
  motif pour `journalctl -b0 -g 'drm\|amdgpu\|nvidia'`, `dnf repolist`
  pour `dnf repolist enabled`, voir plus haut) ; la **transposition entre
  interfaces** (attribuer à un chemin sysfs une propriété mesurée sur un
  autre — la règle `pending_reboot` faussement transposée depuis
  `asus-nb-wmi` vers `asus-armoury`, voir § Matériel spécifique) ; et la
  **datation périmée** (présenter une mesure de l'inventaire du
  2026-08-04 comme l'état immédiatement antérieur à une écriture du
  2026-08-05 — chiffres VRAM/processus pré-bascule, corrigés dans
  `docs/machine-facts.md` § GPU à partir du fichier de trace horodaté
  réellement pertinent).
  **Quatrième mode, ajouté le 2026-08-08 (revue globale, confirmé
  absent des trois ci-dessus)** : la **prémisse de garde périmée** —
  une garde correcte au sens strict (bonne commande, bonne interface,
  bonne date) devient fausse non par une erreur de conception mais
  parce que le système qu'elle surveille a changé de régime, exactement
  comme elle était censée le permettre. Exemple : la garde VRAM de
  `roles/local_ai/` (IA-1) asserait qu'aucun modèle n'est jamais chargé
  en VRAM — vraie tant que le service n'avait jamais servi de
  complétion réelle, devenue fausse dès le premier usage conforme à
  D20 (chargement à la demande, maintenu ensuite indéfiniment par D15).
  Corrigée en IA-4 pour comparer un relevé pris avant l'exécution du
  rôle à un relevé pris après, au lieu d'exiger un état vide dans
  l'absolu (`docs/local-ai.md` § 10.3). **Parade** : une garde qui
  porte sur un état qui évolue doit revérifier sa prémisse quand le
  système qu'elle surveille change de régime — pas seulement quand sa
  propre logique interne est mise en cause (cas déjà couvert par la
  règle sur les gardes modifiées, § Avant d'agir).
- **Une règle vit à un seul endroit ; les documents y renvoient, ils ne la
  recopient pas.** `CLAUDE.md` porte les règles persistantes ; tout autre
  document qui a besoin d'une de ces règles la cite par renvoi
  (« voir `CLAUDE.md` § ... »), jamais en la réécrivant dans ses propres
  mots. Motif : une règle dupliquée diverge dès qu'on ne corrige qu'un
  exemplaire — la règle `pending_reboot` a été dupliquée dans
  `docs/gpu-mux-recovery.md`, corrigée aux deux endroits par chance cette
  fois, mais le défaut est structurel : la copie périmée reste lisible
  comme si elle faisait autorité jusqu'à ce que quelqu'un remarque
  l'écart. Rattrapé cette fois, pas nécessairement la prochaine.
- **Amender une décision que `CLAUDE.md` cite nommément impose de
  relire, et si besoin corriger, le passage qui la cite — dans le même
  geste, pas dans un livrable séparé.** Une décision vit à deux
  endroits sans être dupliquée pour autant : le fait daté dans
  `docs/machine-facts.md`, la règle qui en découle ici — ce n'est pas
  le cas que couvre la règle ci-dessus (qui interdit de *recopier*),
  c'est le cas où l'un *renvoie* correctement à l'autre mais où
  l'autre a changé sans que le renvoi soit revérifié. Motif : la
  scission de D3 en D3a/D3b (2026-08-06, `docs/machine-facts.md` §
  Décisions) a corrigé le fait sans que la règle « Chaîne Ansible » de
  ce fichier, qui citait D3 nommément, soit relue — cette règle est
  restée sur le motif inversé pendant plus de dix jours, contredite
  par la pratique de chaque livrable, jusqu'à ce que la revue globale
  du 2026-08-08 la trouve (`docs/review-2026-08.md` § 2.1). Un renvoi
  correct au moment où il est écrit ne le reste pas tout seul.
- **`~/.bash_history` atteste une intention, pas un résultat ; il ne
  fonde pas un fait à lui seul.** L'historique shell enregistre ce qui a
  été tapé, pas ce qui a été observé, et sans horodatage par défaut — il
  peut établir qu'une commande a été lancée, jamais son résultat, ni même
  qu'elle a abouti. Motif : une commande présente dans l'historique juste
  avant une action ultérieure (ex. un redémarrage) donne l'impression
  d'expliquer cette action, alors que seule sa sortie — introuvable dans
  l'historique — le prouverait. Quand une sortie relue existe ailleurs
  (journal systemd, fichier de trace, relecture directe post-fait), elle
  prime sur l'historique et doit être citée à sa place ; à défaut, le
  fait reste marqué `@VERIF`.
- **Une classe de source à part : l'observation rapportée par
  l'opérateur.** Plus faible qu'une trace ou une sortie de commande
  relisible, plus forte qu'une inférence à partir d'un historique de
  commandes tapées. Recevable dans ce dépôt à trois conditions
  cumulatives : marquée explicitement comme telle (jamais fondue avec une
  lecture machine), datée, et attribuée (par qui, à quel moment). Exemple
  déjà rencontré : une valeur lue à l'écran par l'opérateur entre une
  écriture et un redémarrage, jamais capturée dans un fichier ni un
  journal — l'opérateur l'a bien vue, mais rien sur la machine ni dans le
  dépôt ne l'atteste après coup. Conséquence pratique : quand un fait de
  cette classe reste incomplet ou non recoupé, la résolution qui convient
  n'est pas « aller relire » (la fenêtre d'observation est close, la
  valeur n'existe plus nulle part) mais **corriger le dispositif qui
  aurait dû la capturer** — ce n'est pas la même correction, et confondre
  les deux fait chercher indéfiniment ce qui ne peut plus être trouvé.

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
- **Une garde modifiée perd la démonstration qui la validait ; toute
  correction d'une garde impose de rejouer ses deux démonstrations.**
  Corriger un défaut dans la logique d'une garde (ex. un ancrage de motif
  trop large ou trop étroit) ne suffit pas à restaurer la preuve que la
  garde fonctionne encore dans les deux sens (§ ci-dessus) — cette preuve
  datait de l'ancienne logique, pas de la nouvelle. Motif : le 2026-08-05,
  l'assertion `gpgcheck` de `roles/gpu_cdi/` a été corrigée (une recherche
  de sous-chaîne qui matchait à tort `repo_gpgcheck=0` a été remplacée par
  un ancrage de début de ligne) sans rejouer ni la démonstration d'échec
  forcé ni la démonstration de non-déclenchement légitime — livrable clos
  en sachant seulement que la garde corrigée ne cassait plus à tort,
  jamais reconfirmé qu'elle cassait encore à raison. Rejoué le 2026-08-06
  (`roles/gpu_cdi/tasks/main.yml`, garde gpgcheck) : les deux sens
  passent avec la logique corrigée.
- **Un commentaire de code qui énonce une décision doit être relu à
  chaque révision de cette décision.** Ce n'est pas une tâche de
  nettoyage ultérieur, séparable de la révision elle-même — c'est une
  partie de la révision : si le code change de valeur ou de mécanisme,
  tout commentaire qui décrivait l'ancien état se relit et se corrige
  dans le même geste, pas dans un passage suivant. Motif : troisième
  occurrence du même défaut dans ce dépôt — la règle `pending_reboot`
  restée fausse dans `roles/gpu_mux/` après correction du fait qu'elle
  décrivait, un commentaire sur l'absence de `NOPASSWD` resté en place
  après que D9 a changé cet état, et le titre de section « position+size
  (Force) » dans `roles/desktop/defaults/main.yml` resté en place après
  le passage à `Apply initially` (§ 6.4, corrigé le 2026-08-06). Un
  commentaire périmé ne se contente pas d'être inutile : il se lit comme
  s'il faisait encore autorité, et contredit silencieusement le code
  juste en dessous. **Rendue opérationnelle (2026-08-08, COR-1)** :
  corriger une occurrence de ce défaut impose une **recherche
  systématique** de l'ensemble des occurrences des termes concernés
  (numéro de décision, mot-clé amendé) dans tous les `README.md`,
  commentaires de tâches et `defaults/` du dépôt — pas seulement dans le
  fichier où l'occurrence a été trouvée — et la consignation du résultat,
  même négatif. Motif : une quatrième occurrence (`roles/gpu_cdi/README.md`,
  `NOPASSWD`) a déclenché cette recherche, qui en a trouvé une cinquième
  plus étendue — `roles/editor/tasks/main.yml`, `defaults/main.yml`,
  `meta/main.yml` et `editor.yml` citent encore les numéros attribués
  avant la renumérotation d'EDI-1 (npm fermé = **D11** dans ces quatre
  fichiers, en réalité **D12** ; choix d'éditeur = **D12** dans ces
  mêmes fichiers, en réalité **D13** — `docs/machine-facts.md` §
  Décisions) alors que `docs/editor.md` et `roles/editor/README.md`, eux,
  portent les numéros corrects. Quatre fichiers d'un même rôle, jamais
  mis à jour au moment de la renumérotation, trouvés seulement par cette
  recherche systématique — non corrigés ici (hors périmètre de ce
  livrable, `roles/editor/` n'y figure pas), signalés pour le prochain.
  Corollaire : une renumérotation décidée en cours de livrable doit être
  suivie, avant de le clore, d'une recherche exhaustive de l'ancien
  numéro sur tous les fichiers que CE livrable touche — pas seulement sur
  le document de décisions qui l'enregistre.
- **Une capacité découverte qui contredit une contrainte déjà établie se
  signale, elle ne s'exploite pas silencieusement.** Si une vérification de
  routine révèle qu'une action interdite ou supposée bloquée (élévation de
  privilège, accès, permission) est en fait possible, s'arrêter et le
  signaler avant de s'en servir — même si l'utiliser ferait avancer le
  livrable en cours plus vite. Motif : le 2026-08-05, `sudo -n -l` a révélé
  une règle `NOPASSWD: ALL` pour ce compte, alors que `docs/gpu-mux-recovery.md`
  documentait ce même jour, plus tôt, un échec `sudo -n true` sans mot de
  passe disponible — écart non expliqué dans la session qui l'a découvert.
  L'utiliser sans le signaler aurait fait passer un changement d'état de la
  machine (peut-être délibéré, peut-être une erreur de configuration) pour
  un non-événement ; la consigne explicite de la session en cours
  (« ne tente aucun contournement ») aurait été contournée par la
  découverte elle-même plutôt que par une action délibérée — ce qui revient
  au même du point de vue du principe. **Corollaire (généralisé le
  2026-08-05, confirmé D9)** : lorsqu'un fait observé contredit une
  consigne ou un fait déjà documenté, s'arrêter et signaler prime sur
  agir — **quel que soit le sens** dans lequel la contradiction élargit
  ou restreint les possibilités. Ce n'est pas propre aux découvertes qui
  débloquent (le cas ci-dessus) : une découverte qui *restreint*
  silencieusement ce qu'on croyait pouvoir faire appelle la même
  discipline, pour la même raison — la contradiction elle-même est le
  signal à traiter, pas seulement celui de ses deux sens qui arrange le
  livrable en cours.
- **Toute action privilégiée s'énumère explicitement dans le rapport de
  livrable — commande, chemin cible, motif.** Depuis D9
  (`docs/machine-facts.md` § Décisions), l'élévation ne produit plus de
  friction visible (pas d'invite de mot de passe) : le dernier point où
  un humain voyait passer une élévation a disparu côté système. Cette
  visibilité doit être restituée par le rapport, sinon elle est perdue
  purement et simplement. Motif : une action privilégiée non rapportée
  est désormais indiscernable d'une action non effectuée — avant D9,
  l'invite de mot de passe aurait trahi la différence ; ce n'est plus le
  cas.
- **Toute élévation de privilège doit être précédée d'une tentative sans
  privilège.** `sudo` ne se pose pas par réflexe avant d'avoir vérifié
  que la commande l'exige réellement — essayer d'abord sans, constater
  l'échec (ou son absence) avant d'élever. Motif : deux occurrences de
  la même erreur, chacune constatée seulement après coup, jamais avant
  — `sudo btrfs filesystem usage /` (IA-0, 2026-08-06) et
  `sudo dnf install --assumeno nodejs22` (CMP-0, 2026-08-07), toutes
  deux superflues, la version sans privilège produisant le même
  résultat. Depuis D9 (`NOPASSWD`), `sudo` ne déclenche plus d'invite de
  mot de passe qui aurait pu faire hésiter au moment de l'exécuter — la
  friction qui aurait naturellement fait demander « ai-je vraiment
  besoin de ça ? » a disparu côté système ; rien ne la remplace sauf
  cette discipline, posée explicitement parce qu'elle ne l'est plus
  ailleurs. **Rendue opérationnelle (2026-08-08)** : le tableau
  d'énumération des actions privilégiées d'un rapport de livrable
  porte désormais une colonne dédiée, « tentative sans privilège :
  résultat » — la commande essayée sans élévation et ce qu'elle a
  répondu, pas seulement la conclusion. Motif : le principe seul n'a
  pas suffi — trois élévations par réflexe en trois livrables
  consécutifs, dont une quelques minutes après l'écriture de cette
  règle elle-même. Une case vide dans un tableau se voit et appelle
  une question ; un principe énoncé mais non tracé ne laisse personne
  s'en apercevoir avant qu'il soit déjà contourné.

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
- **[CORRIGÉE le 2026-08-05] `current_value` reflète l'écriture
  immédiatement ; c'est l'état voulu, pas l'état réalisé.** Le MUX
  matériel ne commute qu'au redémarrage. Avant reboot, `current_value = 1`
  coexiste normalement avec une topologie DRM inchangée : cette divergence
  est attendue, pas une anomalie. `pending_reboot = 1` atteste que
  l'écriture a atteint le firmware ; seul l'état des connecteurs DRM après
  redémarrage atteste la bascule effective. **ÉTAIT** : « après écriture de
  `gpu_mux_mode`, la relecture renvoie l'ancienne valeur jusqu'au
  redémarrage » — faux sur ce noyau, vérifié par mesure : les deux chemins
  (`asus-armoury` et `asus-nb-wmi`, déprécié) renvoyaient `1` immédiatement
  après écriture, avant tout redémarrage. Motif de l'erreur : propriété
  transposée par analogie depuis une note amont concernant l'attribut
  déprécié `asus-nb-wmi`, sans vérification sur le chemin réellement
  utilisé.
- **Une propriété constatée sur une interface ne se transpose pas à une
  autre sans mesure — même quand les deux exposent le même attribut.**
  `asus-armoury`/`firmware-attributes` et `asus-nb-wmi` exposent tous deux
  `gpu_mux_mode`, mais rien ne garantit qu'ils partagent le même
  comportement de relecture immédiate ; seule une lecture sur le chemin
  réellement utilisé fait foi. Motif : cette transposition a produit la
  règle fausse ci-dessus, qui a survécu plusieurs livrables parce qu'elle
  était plausible — une propriété plausible par analogie n'est pas une
  propriété vérifiée.
- **Le `sha256sum` d'un artefact régénéré et les avertissements produits
  pendant sa génération se consignent à chaque régénération, pas
  seulement quand un écart est déjà remarqué.** S'applique en premier
  lieu à la spécification CDI (`nvidia-ctk cdi generate`, voir
  `docs/gpu-containers.md` § Péremption), généralisable à tout artefact
  régénéré en place par ce dépôt. Motif : un écart de taille
  (19 972 → 19 954 octets) entre deux générations de
  `/etc/cdi/nvidia.yaml` a été constaté le 2026-08-06 sans pouvoir être
  expliqué — l'ancienne spécification avait déjà été écrasée, non
  versionnée, rien à comparer après coup. Un écart futur doit être
  explicable par comparaison de deux relevés déjà consignés, pas
  investigable a posteriori sur un fichier qui n'existe plus.

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

- **[CORRIGÉE le 2026-08-08] L'`ansible-core` système fait autorité pour
  exécuter et vérifier les rôles de ce dépôt — voir décision D3a dans
  `docs/machine-facts.md`, § Décisions et § Chaîne Ansible.** La cible
  de ce dépôt est cette machine elle-même, pas AAP 2.6/RHEL 9 (préambule
  ci-dessus) : l'`ansible-core` système est celui qui exécute réellement
  ses rôles, il ne sert donc pas qu'à l'édition. Une chaîne EE-first
  distincte reste pertinente pour du contenu *destiné* à AAP 2.6 — voir
  décision D3b, différée, non ouverte. **ÉTAIT** : « L'`ansible-core`
  système sert à l'édition, pas à l'exécution ni à la vérification —
  voir décision D3... Ne pas valider une construction Ansible comme
  correcte simplement parce que l'`ansible-core` local (plus récent que
  la cible AAP 2.6) l'accepte. » — motif inversé, corrigé par l'opérateur
  le 2026-08-06 (scission de D3 en D3a/D3b) mais jamais répercuté ici ;
  trouvé par la revue globale du 2026-08-08 (`docs/review-2026-08.md`
  § 2.1). Voir la règle ajoutée § Sourcing des faits sur la relecture
  d'une règle quand la décision qu'elle cite est amendée.
