# État réel du dépôt — 2026-08-09

Une page, pour quelqu'un qui reprend ce dépôt sans avoir lu les
quarante livrables qui l'ont produit. Elle ne recopie rien — chaque
ligne renvoie au document qui porte la preuve ou la décision complète.
`docs/machine-facts.md` porte l'histoire (décisions datées, journal de
chaque série) ; ce document répond à une seule question, **où en
est-on aujourd'hui**, et se périme dès qu'un futur livrable change
l'un des faits qu'il résume — à tenir à jour à ce moment-là, pas avant.

**Rafraîchi le 2026-08-09 (livrable de clôture) — écrit plus tôt le
même jour, déjà périmé sur plusieurs points avant la fin de la
journée.** Pilote NVIDIA : `610.57.04` (était `610.43.03` au moment de
la première rédaction). `supergfxctl` : disparu (retiré silencieusement
par obsolescence Terra le 2026-08-07, `docs/repositories.md` § 10).
`cardwire` : retiré (D23, `docs/machine-facts.md` § Décisions). Kitty :
opérationnel à l'ouverture de session réelle (bug BUR-5 corrigé) — voir
tableau ci-dessous pour la nuance sur ce qui reste non prouvé.

**Rafraîchi une seconde fois le 2026-08-10** : GitHub Copilot CLI
(D24/D25) authentifié par l'opérateur, modèle et consommation
confirmés par sortie de commande — déplacé vers une preuve complète,
voir la ligne dédiée ci-dessous.

## Ce qui est en place et prouvé

| Composant | Décision(s) | Preuve | Document |
|---|---|---|---|
| Bascule GPU MUX (panneau câblé dGPU/iGPU) | D2bis/D2ter | Bascule effectuée et vérifiée par mesure post-redémarrage (topologie DRM, `pending_reboot`) | `docs/gpu-mux-recovery.md`, `docs/machine-facts.md` § Décisions, journal 2026-08-05 |
| Chemin de retour SSH avant toute bascule risquée | `roles/recovery/` | Trois gardes (sshd actif+activé, clé publique non vide, `firewalld` autorise ssh), démontrées dans les deux sens | `roles/recovery/README.md` |
| Toolkit conteneur NVIDIA + spécification CDI | D7 | Dépôt COPR corroboré (empreinte de clé, sans corroboration indépendante — écart assumé, § écarté ci-dessous n'applique pas ici, c'est un risque **accepté**, pas une voie écartée). **Chaîne détection → régénération prouvée de bout en bout par un événement réel non provoqué (2026-08-09, GPU-4)** : mise à jour du pilote `610.43.03` → `610.57.04`, péremption détectée, `ollama` bloqué (`ExecStartPre` refusé), notifications émises, régénération via `regen-cdi-spec` (bogue de collision de variable trouvé et corrigé au passage — cette branche n'avait jamais réellement écrit depuis sa création), test conteneur de bout en bout réussi. **Nature de la preuve, précisée** : avant cet événement, seule la branche « déjà à jour » de `verify-cdi-spec` avait été exercée (d'où la formulation précédente de cette ligne, correcte pour ce qu'elle couvrait mais silencieuse sur la branche de régénération, jamais testée en conditions réelles jusqu'ici) | `docs/gpu-containers.md` § 9.6-9.7, `docs/repositories.md` § 1-3 |
| Temporisation du démarrage `kitty` sur la stabilité des sorties | BUR-4/BUR-5 | **Partiellement prouvée.** Opérationnelle à deux ouvertures de session réelles (`269 ms`, `268 ms`) — le bug BUR-5 qui empêchait l'autostart de se lancer est réellement corrigé. Mais ces deux temps sont le **plancher structurel** du script (deux échantillons `rc=0` identiques dès le premier relevé, jamais d'itération au-delà du minimum) : le mécanisme n'a **jamais eu à attendre** une topologie réellement instable. Rien ne permet de lui attribuer une éventuelle absence de gel de `eDP-1` — voir la nuance Kate ci-dessous pour un patron similaire (preuve réelle mais partielle, pas à lire comme équivalente à une preuve complète) | `docs/desktop.md` § 9-10, `docs/machine-facts.md` § Points ouverts (BUR-5) |
| Service d'inférence local (Ollama conteneurisé, réseau confiné) | D14/D18 | Confinement réseau prouvé par inspection à quatre reprises (deux fois deux) ; service joignable, `verify-cdi-spec` réutilisé sans réimplémentation | `docs/local-ai.md` |
| Paramètre pilote `NVreg_PreserveVideoMemoryAllocations=1` | D16 | Écrit, redémarrage effectué, valeur active confirmée `1` après (journal daté 2026-08-08, IA-4) | `docs/machine-facts.md` § Décisions, D16 |
| Modèles retenus, chargement séquentiel | D21/D22 | Récupération isolée par conteneur éphémère (jamais le réseau du service), intégrité vérifiée, coexistence en VRAM mesurée **ne pas** tenir (§ écarté ci-dessous) | `docs/machine-facts.md` § Décisions, D21/D22 |
| Complétion locale, Helix | D19/D20 | Bout en bout **entièrement machine** : session `tmux` détachée pilote la frappe, `hx -vv` produit le journal, complétions FIM réelles YAML/Python, latence 0,46-0,48 s modèle chargé, échec forcé démontré | `docs/completion.md` § 7-8 |
| Complétion locale, Kate | KAT-1 (2026-08-08) | Bout en bout, mais preuve **mixte** — voir la nuance ci-dessous, pas au même niveau que Helix | `docs/completion.md` § 9 |
| Éditeurs (Helix terminal, Kate graphique) | D12/D13 | Installés, greffons Kate configurés par clé nommée (jamais copie de fichier), garde D12 **resserrée** (D24 : aucun serveur de langage npm, `lsp-ai` au chemin compilé — npm lui-même n'est plus interdit) démontrée dans les deux sens, dans `roles/editor/` et `roles/completion/` indépendamment | `docs/editor.md` § Copilot CLI |
| GitHub Copilot CLI, second agent de code | D24/D25 | `@github/copilot@1.0.78` installé (version épinglée, domaine utilisateur `~/.local`), runtime Node.js 22 depuis `fedora`/`updates`. Authentification établie par l'opérateur (`copilot login`, jamais scriptée) ; modèle actif confirmé `claude-sonnet-5` (medium) — **parité de génération de modèle** avec Claude Code, pas d'effort ; consommation lisible (3 700/20 000 AIC, 18 %, période de renouvellement inconnue) — **preuve par sortie de commande copiable**, datée 2026-08-10 | `docs/editor.md` § Copilot CLI |
| Amorçage (D6/D9/D10 scriptés) | `roles/bootstrap/` | `power-profiles-daemon`, dépôt `terra` (clé vérifiée par inspection hors ligne avant tout usage privilégié), règle `NOPASSWD` — les trois reconstructibles par ce rôle | `roles/bootstrap/README.md`, `docs/orchestration.md` § 7.1 |
| Règle `sudo` sans mot de passe déplacée en `/etc/sudoers.d/` | D9 (déplacée) | Séquence stricte à cinq étapes, provenance de la règle effective prouvée (`sudo -n -l -l`, pas seulement `sudo -n true`) avant tout retrait de l'ancienne ligne | `docs/machine-facts.md` § Décisions, D9, journal 2026-08-08 (SUD-1) |
| Séquence de reconstruction complète, un seul point d'arrêt | `site.yml` | `--check` complet enchaîne les huit rôles sans erreur sur **cette** machine (`copilot_cli` ajouté, D24) ; point d'arrêt exercé dans son état nominal réel | `docs/orchestration.md` |
| Chaîne Ansible de ce dépôt | D3a | `ansible-core` système fait autorité pour l'exécution ET la vérification (pas seulement l'édition) | `docs/ansible-chain.md`, `docs/machine-facts.md` § Chaîne Ansible |

### Nuance sur la preuve Kate, à ne pas lire comme équivalente à Helix

`docs/completion.md` § 9 qualifie explicitement cette différence,
ajoutée à la clôture de cette série : pour Helix, les deux moitiés de
la démonstration (frappe et lecture du journal) sont **machine**,
rejouables sans opérateur. Pour Kate, seule la lecture du journal LSP
est une preuve directe — le déclenchement (clic, Ctrl+Espace) relève
de la classe **« observation rapportée par l'opérateur »**
(`CLAUDE.md` § Sourcing des faits) : recevable, mais **pas rejouable
de la même façon** si la voie Kate se met un jour à défaillir — il
faudrait de nouveau un opérateur au clavier, pas seulement relancer
une session détachée.

### Nuance sur la preuve de la temporisation `kitty`, même patron que Kate

Preuve réelle (deux ouvertures de session authentiques, pas une
invocation Ansible), mais **partielle** pour une raison différente de
Kate : ici, ce n'est pas la classe de la source qui est en cause (le
journal `systemd --user` est une preuve directement rejouable), c'est
le **cas exercé**. Les deux occurrences observées ont trouvé une
topologie déjà stable dès le premier échantillon — la boucle
d'attente n'a jamais eu à itérer, encore moins à atteindre la borne de
5 s. La branche qui justifie l'existence même de ce mécanisme (une
sortie qui met plusieurs centaines de millisecondes à se stabiliser)
reste **non exercée** au sens de la règle ajoutée à `CLAUDE.md`
§ Avant d'agir à cette même clôture — une branche jamais empruntée
n'est pas prouvée, quel que soit le nombre de fois où la branche
opposée (ici : « déjà stable ») a réussi.

## Ce qui est en place mais non prouvé de bout en bout

1. **La séquence de reconstruction complète, sur une machine qui n'a
   jamais vu aucun de ces rôles.** `site.yml --check` a été exercé
   sur **cette** machine, qui a déjà D6/D9/D10 satisfaites,
   `gpu_mux_mode`/le paramètre pilote déjà à leur valeur cible, et un
   historique d'exécutions antérieures de chaque rôle. Rien ne prouve
   que la séquence réussirait sur une machine réellement neuve — y
   compris l'amorçage `--ask-become-pass` sur un compte sans aucune
   règle `sudo` préalable, jamais testé comme seule voie d'accès.
   **Pour vérifier** : une machine jetable (VM ou matériel de test,
   idéalement avec le même GPU pour `gpu_mux`/`gpu_cdi`/`local_ai` —
   une VM sans RTX 4090 ne prouverait que la moitié), Fedora 44 nu,
   aucun des prérequis § 0 satisfait, accord explicite de l'opérateur
   avant toute exécution qui redémarre ou modifie des paquets système
   pour de vrai (`docs/orchestration.md` § 5).
2. **La combinaison des deux redémarrages requis en un seul**
   (`gpu_mux_mode`, D2bis/D2ter ; paramètre pilote, D16). Établie
   **par lecture** des deux mécanismes de prise d'effet (ordre de
   `asus-shutdown.service`, rechargement du module `nvidia.ko`), pas
   par exécution — les deux changements ont toujours été appliqués par
   deux redémarrages séparés à des dates différentes sur cette
   machine. **Pour vérifier** : appliquer les deux changements dans la
   même session, un seul redémarrage, confirmer les deux états
   attendus après (`docs/orchestration.md` § « Un seul redémarrage,
   pas deux »).
3. **Le comportement d'une suspension système avec un modèle chargé en
   VRAM.** D16 vise justement à préserver l'allocation VRAM à la
   suspension (`NVreg_PreserveVideoMemoryAllocations=1`), mais aucun
   livrable de cette série n'a chargé un modèle puis suspendu la
   machine pour l'observer. **Pour vérifier** : charger un modèle
   (`--tags pull-models` puis une complétion ou un message de chat
   réel), suspendre (S0ix/s2idle), reprendre, vérifier l'état du
   pilote et l'absence de rechargement forcé du modèle.

## Ce qui a été écarté, et pourquoi

| Écarté | Motif (une ligne) | Décision |
|---|---|---|
| `npm`/Node.js comme surface de récupération de serveurs de langage pour Helix/Kate | Aucun serveur YAML/Ansible empaqueté dans les onze dépôts ; un modèle de dépendances transitives hors contrôle rpm pour un besoin finalement couvert autrement (`lsp-ai`) — **toujours écarté** (D12 resserrée), y compris depuis que npm lui-même est ouvert pour un usage distinct (Copilot CLI, D24, `docs/repositories.md` § 11) | D12 |
| `ansible-navigator` | Aucune capacité manquante établie pour la chaîne de ce dépôt (ni pour la chaîne AAP différée) — sur-couverture pure | D3a/D3b |
| Zed | Magasin d'extensions propre (nouvelle surface d'approvisionnement), télémétrie, dépendance à `terra` | D13 |
| Cursor | Mêmes motifs que Zed ; nativité Wayland jamais confirmée (marqueur fermé par requalification, pas par vérification — Cursor n'a jamais été installé) | D13 |
| Dépôt NVIDIA officiel (à la place du COPR) | COPR jugé équivalent avec des vérifications OpenPGP supplémentaires, un cran au-dessus du dépôt officiel sur ce point précis | D7 |
| Coexistence de deux modèles en VRAM (chat + complétion résidents) | Mesurée, pas supposée — ne tient pas (chat seul à 32 K de contexte occupe déjà 12 294 Mio) ; chargement séquentiel retenu à la place | D22 |
| Résidence permanente du modèle de complétion | Coûte ~8 W en continu et neutralise la veille runtime (RTD3) pour un bénéfice non encore mesuré à l'origine de la décision | D15, requalifiée par D20 |
| Binaire `lsp-ai` précompilé (à la place de la compilation depuis les sources) | Aucune somme de contrôle publiée — ancrage de confiance plus faible que ce que fermer `npm` (D12) était censé éviter | D19 |

## Surfaces d'approvisionnement et ancrage de confiance

Détail complet, ancrage réel et écart résiduel assumé pour chacune :
[`docs/repositories.md`](repositories.md). Résumé :

| Surface | Ancrage | § |
|---|---|---|
| Dépôts `dnf` (fedora/updates/rpmfusion) | Signature du projet Fedora, chaîne standard | § 4 |
| Terra (Fyra Labs) | Empreinte de clé corroborée localement, **sans corroboration indépendante trouvée** — risque assumé, pas écarté | § 1-3 |
| COPR (toolkit conteneur NVIDIA) | Empreinte de clé de projet COPR — mécanisme distinct du dépôt Fedora principal | § 1 note |
| Registre de conteneurs (image Ollama) | Empreinte de contenu épinglée | § 6 |
| `crates.io` (compilation `lsp-ai`) | `Cargo.lock` + `--locked`, code source auditable | § 7 |
| Dépôt git amont `lsp-ai` | Épinglé par empreinte de commit, jamais une étiquette (les étiquettes peuvent être déplacées) | § 7 |
| Registre de modèles Ollama | Empreinte de contenu, intégrité testée après récupération | § 8 |
| npm (`@github/copilot`, D24) | Intégrité de contenu (sha512) + signature du registre + attestation de provenance Sigstore pour le paquet principal (`npm audit signatures`, vérifié localement) — pas de fichier de verrouillage possible pour une install globale, version épinglée en commande à la place | § 11 |

## Points ouverts restants

**27 occurrences du jeton de vérification** recensées hors
`CLAUDE.md` (0 par construction) et hors `docs/review-2026-08.md`
(8 occurrences, toutes non actionables — citations ou explications du
jeton lui-même, aucune sous forme nue) — **environ dix-neuf questions
distinctes encore actionables** une fois les renvois consolidés
(inventaire complet, non repris ici : `docs/review-2026-08.md` § 5).
Aucune de priorité haute — la revue globale n'en a trouvé aucune de
niveau 1 (faux aujourd'hui) parmi les marqueurs eux-mêmes.

**Priorités, reprises de `docs/review-2026-08.md` § « Ce qu'il
faudrait traiter en premier »** (statut détaillé de chacun des
constats de la revue : `docs/review-2026-08.md`, marquage 2026-08-09) :

1. **Pratiques constantes de ce dépôt jamais écrites comme règles**
   (patron de garde substituable par `-e <var>_expected=...`, chemins
   résolus dynamiquement, domaine `~/.local/bin/` pour tout binaire
   compilé, structure README « ce que le rôle fait / ne fait
   jamais ») — basse priorité, aucune n'a produit de défaut, seulement
   un risque de dérive si personne ne les nomme.
2. **Un seul mécanisme de re-vérification périodique** (minuterie CDI)
   sur quatre catégories de valeurs épinglées équivalentes (clé COPR,
   image Ollama, commit `lsp-ai`, commit `hf-hub`) — basse priorité,
   aucune dérive observée à ce jour.
3. **Couplage manuel de valeurs dupliquées entre `roles/local_ai/` et
   `roles/gpu_cdi/`/`roles/completion/`** (chemin CDI, adresse
   Ollama) — identiques aujourd'hui, risque déjà nommé dans les rôles
   eux-mêmes, pas un défaut caché.
4. **Marqueur `docs/desktop.md` § 6.7** (effet réel de l'entrée
   autostart) — la fenêtre d'observation s'est présentée au moins une
   fois depuis (redémarrage confirmé postérieur), jamais ressaisie.
5. **Deux constats cosmétiques délibérément non traités** : numérotation
   redondante des surfaces d'approvisionnement (`docs/repositories.md`),
   hygiène git (27 commits linéaires sur `main`, jamais de branche par
   série malgré la méthode annoncée) — coût de rétrofit élevé
   (réécriture d'historique) pour un bénéfice de lisibilité seul,
   écartés explicitement par la revue elle-même, pas oubliés.

## Voir aussi

- [`docs/machine-facts.md`](machine-facts.md) — l'histoire complète,
  décisions datées (D1-D25) et journal de chaque série.
- [`docs/review-2026-08.md`](review-2026-08.md) — l'audit complet,
  statut de chaque constat.
- [`docs/orchestration.md`](orchestration.md) — la séquence de
  reconstruction, ce qui a été vérifié et ce qui ne l'a pas été.
- [`docs/repositories.md`](repositories.md) — surfaces
  d'approvisionnement, détail complet.
- [`CLAUDE.md`](../CLAUDE.md) — règles persistantes, bilan de méthode
  de cette série (§ Modes d'échec qui imitent le sourcing).
