# Règles persistantes — poste de développement

Ce dépôt décrit et reproduit la configuration d'un poste de développement
personnel (ASUS ROG Zephyrus Duo 16 GX650PY, Fedora 44 KDE/Wayland). Il sera
public. La cible de production réelle est AAP 2.6 sur RHEL 9, gérée
ailleurs — pas depuis ce dépôt.

Chaque règle ci-dessous porte sa raison. Une règle sans motif se fait
« optimiser » par la session suivante : ne retire ni n'assouplis une règle
sans comprendre pourquoi elle a été posée.

## Sourcing des faits

- **Ne jamais affirmer sans source lue sur la machine.** Si un fait n'a pas
  été confirmé par une commande exécutée dans la session en cours, il se
  marque `# À VÉRIFIER : <où le confirmer>` plutôt que d'être énoncé comme
  acquis. Motif : une machine de dev évolue (mises à jour, changements
  manuels) ; une mémoire ou une supposition périme silencieusement, une
  commande datée ne ment pas sur ce qu'elle a vu.
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

## Avant d'agir

- **Résolution en lecture seule avant toute édition.** Avant de modifier un
  fichier ou d'exécuter une commande qui change l'état du système, vérifier
  d'abord ce que l'outil fait déjà lui-même (aide intégrée, `--dry-run`,
  option de simulation) plutôt que de le déduire ou de le tester en
  conditions réelles. Motif : cette machine ne porte aucune donnée
  professionnelle mais reste un poste de travail quotidien ; une commande
  supposée passive qui ne l'est pas (ex. import de clé GPG déclenché par un
  simple `repoinfo`) doit être détectée avant d'agir, pas après.
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
