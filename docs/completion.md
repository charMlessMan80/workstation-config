# Complétion locale dans l'éditeur — éditeur ou serveur ? (résolution en lecture seule, 2026-08-07)

**Ce document ne corrige, n'installe et ne télécharge rien.** Toutes les
commandes citées ci-dessous sont des lectures, exécutées le 2026-08-07,
sur ce poste, ou des requêtes vers de la documentation amont publique.
Aucun paquet installé, ni `node` ni `npm`, aucun modèle ni image
téléchargé, aucun dépôt ajouté, aucune configuration d'éditeur ni de
service modifiée.

**Note de numérotation, signalée plutôt que suivie en silence**
(`CLAUDE.md` § une découverte qui contredit un fait déjà documenté se
signale) : la demande de ce livrable désigne la fermeture de npm
« D11 ». C'est déjà arrivé une fois pour cette même décision (EDI-1),
et elle a été renumérotée **D12** parce que D11 était déjà pris
(placement de fenêtre KWin, `docs/machine-facts.md`). Ce document
utilise **D12** partout, pas D11 — la décision visée est la même,
seul le numéro diffère.

## Résumé, pour ne pas enterrer la conclusion

**Un mécanisme fonctionne avec Helix et, structurellement, avec Kate —
sans rouvrir D12, sans changer d'éditeur.** Ce n'est ni la voie « npm »
ni la voie « changer d'éditeur » anticipées par la demande : c'est une
**troisième surface**, un binaire externe (`lsp-ai`), avec son propre
coût, nommé § 5. **Ceci contredit directement une conclusion d'IA-2**
(`docs/local-ai.md` § 8.6, `docs/machine-facts.md` § D15) qui affirmait
qu'aucun mécanisme ne permettait de complétion automatique au fil de la
frappe dans ces deux éditeurs — cette conclusion était **incomplète**,
pas fausse sur ce qu'elle avait vérifié (`llm-ls` est bien inutilisable,
pour le motif qu'elle donne), mais elle n'avait pas cherché plus loin
qu'un seul serveur. Signalé ici explicitement, corrigé § 6, pas effacé.

## 1. Où se situe le blocage : éditeur ou serveur ?

### 1.1 — Helix (25.07.1, version installée sur ce poste)

**Aucun système de greffons, établi par trois lectures locales
convergentes, pas une seule** :
```
$ rpm -q helix
helix-25.07.1-11.fc44.x86_64
$ hx --help
[...] -c, --config <file>  -g, --grammar {fetch|build}  --health [...]
# aucun indicateur lié à un greffon, un script, ou une extension
$ grep -i 'plugin\|wasm\|lua\|scripting' /usr/share/licenses/helix/LICENSE.dependencies
(rien)
$ grep -ci 'plugin\|scripting\|inline completion\|ghost text\|copilot\|ai completion\|inlineCompletion' /usr/share/doc/helix/CHANGELOG.md
0
```
Le fichier `CHANGELOG.md` **packagé avec ce poste** (2947 lignes,
historique complet du projet jusqu'à cette version) ne mentionne à
aucun endroit un système de greffons, un moteur de script, ni une
fonctionnalité de complétion IA/« ghost text ». Recoupé par une source
externe, nommée : discussions officielles du projet,
[helix-editor/helix#10131](https://github.com/helix-editor/helix/discussions/10131)
et [#8887](https://github.com/helix-editor/helix/issues/8887) —
« Helix does not yet have a plugin system » ; l'extension LSP 3.18
`inlineCompletionProvider` (le mécanisme qui donnerait un vrai « texte
fantôme ») est discutée mais non implémentée dans les sources
consultées.

**Ce que Helix sait faire, établi par le même changelog** : la
complétion, pour Helix, c'est exclusivement `textDocument/completion`
LSP standard (menu de complétion, déclenché automatiquement à la
frappe ou manuellement) — confirmé par des dizaines d'entrées du
changelog (tri, snippets, complétion multi-curseurs, complétion
incomplète LSP, etc.), toutes dans ce cadre standard. Une entrée
confirme le comportement « à la frappe » par défaut existe déjà comme
fonctionnalité mûre : *« This improves the behavior of autocompletions.
For example navigating in insert mode no longer automatically triggers
completions »* — implique que **taper** du texte déclenche bien la
complétion automatiquement (le changement corrige seulement le cas où
la navigation, pas la frappe, la déclenchait à tort).

**Conclusion pour Helix : le blocage n'est pas « Helix ne sait pas
faire de complétion pilotée par un modèle »** — Helix sait consommer
`textDocument/completion` de n'importe quel serveur conforme, y
compris à la frappe. Le blocage identifié en IA-2 était spécifique à
**un** serveur (`llm-ls`, protocole propriétaire) — voir § 2.

### 1.2 — Kate (26.04.3, version installée sur ce poste)

**Le greffon client LSP n'est pas générique au point d'exclure
n'importe quelle extension de protocole — mais il n'implémente lui-même
que le protocole standard**, établi par lecture directe du binaire
réellement déployé (pas une hypothèse sur son comportement) :
```
$ rpm -q kate
kate-26.04.3-1.fc44.x86_64
$ rpm -ql kate-plugins | grep -i lsp
(métadonnées, .mo de traduction)
$ find / -iname '*lspclient*.so' 2>/dev/null
/usr/lib64/qt6/plugins/kf6/ktexteditor/lspclientplugin.so
$ strings -a /usr/lib64/qt6/plugins/kf6/ktexteditor/lspclientplugin.so | grep -i 'completion\|executeCommand\|custom' | sort -u
[...] LSPClientCompletion  LSPCompletionItem  completionProvider
KTextEditor::CodeCompletionModelControllerInterface [...]
```
Le greffon pilote le modèle de complétion standard de KTextEditor
(`CodeCompletionModelControllerInterface`) à partir de `LSPCompletionItem`
— la structure de données du protocole LSP standard. **Aucune chaîne
de méthode personnalisée trouvée** (pas de `llm-ls/`, pas de `tabby/`,
pas de dispatcher générique vers une méthode arbitraire) — le greffon
Kate est, comme Helix, un client LSP standard, ni plus ni moins
capable. Recoupé par une source externe, nommée :
[kate-editor.org/posts/kate-language-server-protocol-client](https://kate-editor.org/posts/kate-language-server-protocol-client/) —
aucune mention de méthodes personnalisées ou de scripting du greffon
LSP. Aucun greffon empaqueté dans les onze dépôts au-delà de ceux déjà
listés en D13 (`cmaketoolsplugin`, `externaltoolsplugin`,
`katexmltoolsplugin`, `lspclientplugin`) n'apporte de capacité
supplémentaire — vérifié, `rpm -ql kate-plugins`.

**Conclusion pour Kate : même situation qu'Helix.** Le client est
générique-standard, pas extensible par script ni par méthode
personnalisée — mais il consomme `textDocument/completion` de
n'importe quel serveur conforme, sans distinction avec Helix sur ce
point précis.

**Réponse à la question posée : le blocage n'est pas structurellement
côté éditeur.** Les deux éditeurs acceptent la complétion LSP standard,
à la frappe. Le blocage établi en IA-2 concernait un serveur précis
qui n'expose pas cette voie standard — généraliser cette conclusion à
« aucun mécanisme ne fonctionne » était l'erreur, corrigée § 6.

## 2. Serveurs de complétion recensés — le protocole est le point décisif

| Serveur | Distribution | Protocole de complétion | Backend local (Ollama) | FIM | Licence |
|---|---|---|---|---|---|
| `llm-ls` (Hugging Face) | Rust, `cargo install` ou binaires précompilés (GitHub Releases) | **Propriétaire** — `llm-ls/getCompletions`, lu dans `crates/llm-ls/src/main.rs` (IA-2, reconfirmé) | Oui | Oui | Apache-2.0 |
| `tabby-agent` (TabbyML) | **Node.js v18**, agent JS parlant LSP au serveur Tabby | **Standard** (`textDocument/completion` **et** `textDocument/inlineCompletion`, LSP 3.18) + extensions `tabby/*` optionnelles (contexte enrichi, pas requises pour la complétion de base) | Le serveur Tabby lui-même se configure avec un backend OpenAI-compatible (Ollama en fait partie) | Oui | Apache-2.0 (agent), Apache-2.0 (serveur) |
| **`lsp-ai` (SilasMarvin)** | Rust, `cargo install lsp-ai` **ou** binaire précompilé (GitHub Releases, `lsp-ai-x86_64-unknown-linux-gnu.gz`, présent depuis la v0.6.0) | **Standard** — `completion_provider: CompletionOptions::default()`, dispatché sur le type `Completion` de `lsp_types::request` (lu directement dans `crates/lsp-ai/src/main.rs` **et** `transformer_worker.rs` : le texte généré par le backend est inséré comme `textEdit` d'un `CompletionItem` standard, aucune méthode personnalisée) | Oui, backend Ollama documenté explicitement | Oui | MIT |
| `ai-lsp` (tommoa) | **npm/bun** (`bun install`, TypeScript) | Standard visé (« API compatible avec copilot-language-server »), mais projet expérimental, inachevé | Multi-fournisseurs annoncé | Non établi | Apache-2.0 |
| `llm-lsp` (RubyGems) | **Ruby** (`gem install`) — dépendance nouvelle, ni npm ni Fedora | Non vérifié en détail (nom seul établi, hors périmètre : dépendance Ruby, pas npm, mais surface nouvelle quand même) | Ollama et compatibles OpenAI annoncés | Oui (FIM annoncé) | Non établi |
| `minuet-ai.nvim` | Lua, greffon Neovim | **Ne s'applique pas à Helix/Kate** — consomme les sources de complétion internes de Neovim, pas le protocole LSP | Oui | Oui | Non vérifié — hors sujet |

**Aucun des six trouvés n'est empaqueté dans les onze dépôts activés** :
```
$ dnf repoquery --available 2>/dev/null | grep -iE '^lsp-ai|^llm-ls|^tabby|^ai-lsp'
(vide)
```

**Le point décisif, vérifié serveur par serveur, pas supposé** : `llm-ls`
est le seul des trois candidats sérieux (`llm-ls`, `tabby-agent`,
`lsp-ai`) dont la complétion **elle-même** exige une méthode non
standard — confirmé en lisant son code, pas sa réputation. `tabby-agent`
et `lsp-ai` utilisent tous deux `textDocument/completion` standard pour
la complétion de base ; `tabby-agent` ajoute des extensions
personnalisées **optionnelles**, `lsp-ai` n'en a aucune sur ce chemin.
**Ce n'est donc pas vrai de tous** — la généralisation depuis `llm-ls`
seul, faite en IA-2, ne tenait pas.

### 2.1 — `lsp-ai`, le candidat qui fonctionne sans npm ni changement d'éditeur

**Compatibilité Helix explicitement documentée par le projet amont**
(source externe, nommée) : le README de
[github.com/SilasMarvin/lsp-ai](https://github.com/SilasMarvin/lsp-ai)
liste « VS Code, NeoVim, Emacs, Helix, Sublime » comme éditeurs
supportés, avec des captures d'écran de démonstration dans Helix.
**Kate n'y est pas nommé** — absence de mention, pas incompatibilité
établie : le mécanisme (LSP standard) est le même que celui vérifié
§ 1.2 pour Kate, mais personne ne l'a documenté comme testé avec Kate
spécifiquement. `@VERIF : compatibilité réelle de lsp-ai avec le
greffon LSP client de Kate — non testée par le projet amont (silence,
pas un échec documenté) ni par ce livrable (lecture seule, aucun
binaire téléchargé).`

**Ce que `lsp-ai` exigerait, nommé avant d'être pesé** :
- **Pas de npm, pas de Node** — Rust, statique dans l'esprit, aucune
  mention de dépendance JavaScript dans le code source lu.
- **Pas dans les onze dépôts** — obtenu soit par `cargo install`
  (exige un compilateur Rust, absent aujourd'hui : `command -v cargo
  rustc` → vide, seul `rust-srpm-macros` — un paquet de macros de
  construction RPM, pas une chaîne d'outils utilisable — est présent),
  soit par un binaire précompilé téléchargé directement depuis GitHub
  Releases (`lsp-ai-x86_64-unknown-linux-gnu.gz`, correspond à
  l'architecture de ce poste, présent dans la release `v0.7.1`, publiée
  **2024-09-24T14:49:55Z** — établi par l'API GitHub, source externe
  citée précisément).
- **Aucune somme de contrôle ni signature publiée à côté du binaire** —
  vérifié en listant les noms d'actifs de la release (12 fichiers, tous
  des archives binaires par plateforme, aucun `.sha256`/`.asc`/`.sig`).
  L'ancrage de confiance serait donc TLS-vers-GitHub seul — plus faible
  que l'empreinte de contenu épinglée retenue pour l'image Ollama
  (`docs/repositories.md` § 6), du même ordre que ce qui a été discuté
  pour Terra (D10) sans corroboration indépendante trouvée à l'époque.
  Recherche de corroboration indépendante non menée dans ce livrable
  (lecture seule, aucun téléchargement) — à faire si cette voie est
  retenue, même discipline que D7/D10.
- **Fraîcheur** : dernière publication le 2024-09-24, soit **environ
  23 mois** avant la date de ce livrable (2026-08-07) — établi par
  soustraction simple, pas approximé. Le mainteneur déclare lui-même,
  dans le README amont : « Development is not necessarily done, but no
  new features are currently being developed for it » — projet en mode
  maintenance, pas activement développé, sans être formellement
  archivé (aucune bannière d'archivage trouvée).

## 3. Si le blocage avait été côté éditeur — D12/D13, motifs revérifiés

**Sans objet pour la voie `lsp-ai` (§ 2.1)** — elle fonctionne avec
l'éditeur déjà en place. Répondu quand même, tel que demandé, pour
statuer sur les motifs D13 : **ils tiennent toujours**, rien de trouvé
dans cette série ne les affaiblit.
```
$ grep -n 'Zed et Cursor écartés' docs/machine-facts.md
974:**D13 (2026-08-06) — deux éditeurs selon la tâche [...]
981: [...] Zed et Cursor écartés : magasin d'extensions propre (nouvelle
surface d'approvisionnement), télémétrie, dépendance à `terra` pour les deux
```
- **Magasin d'extensions propre** : toujours vrai, non revérifié
  différemment ici — aucune source consultée dans ce livrable ne le
  change.
- **Télémétrie** : idem, `docs/editor.md` § 1.3 fait toujours foi.
- **Dépendance à `terra`** : idem — et Terra reste le seul dépôt de ce
  poste sans corroboration indépendante de sa clé (D10), un motif qui
  s'applique à Zed et Cursor de façon inchangée.

**Nuance nommée pour être complète, sans rouvrir D13** : Zed propose
une fonctionnalité native de prédiction d'édition avec un fournisseur
Ollama configurable (mécanisme propre à l'éditeur, pas LSP) — non
creusé davantage ici, puisque les trois motifs d'exclusion ci-dessus
restent valides indépendamment de cette capacité. `@VERIF : détail
exact du mécanisme de prédiction d'édition natif de Zed avec Ollama —
non lu dans le code ou la documentation à la version précise, cité de
mémoire d'une recherche antérieure, à vérifier si D13 est un jour
rouverte pour un autre motif.`

## 4. Coût de rouvrir npm — chiffré, pour mémoire, aucune voie recensée ne l'exige strictement

Aucun des candidats retenus § 2 n'exige npm pour fonctionner avec
Helix ou Kate (`lsp-ai` n'en a pas besoin ; `tabby-agent` en aurait
besoin mais son avantage — extensions `tabby/*` optionnelles — n'est
pas nécessaire pour la complétion de base). Chiffré quand même, comme
demandé, pour le cas où `tabby-agent` serait un jour préféré :

```
$ dnf repoquery --whatprovides nodejs
nodejs22-1:22.23.1-2.fc44.x86_64   (+ variante updates)
$ dnf install --assumeno nodejs22
Installing: nodejs22 (304.5 KiB)
Installing dependencies: nodejs22-bin (142.1 KiB), nodejs22-libs (79.2 MiB),
  nodejs22-npm-bin (12.7 KiB)
Installing weak dependencies: nodejs22-docs (49.8 MiB),
  nodejs22-full-i18n (31.6 MiB), nodejs22-npm (8.1 MiB)
Transaction Summary: Installing 7 packages
Total size of inbound packages is 36 MiB. Need to download 36 MiB.
After this operation, 169 MiB extra will be used.
Operation aborted by the user.
```
**~87,5 Mio installés pour un `node`+`npm` minimal** (sans documentation
ni jeu de données i18n complet), **169 Mio avec les dépendances
faibles**, dominés par `nodejs22-libs` (moteur V8) à lui seul ~79 Mio.
`tabby-agent` lui-même ajouterait ses propres dépendances npm
transitives, non chiffrées ici (jamais installé) — c'est cette couche,
pas seulement `nodejs`/`npm`, qui motivait D12.

**Ce que retirer cette surface coûterait ensuite, nommé avant d'ouvrir**
(demande § 4) : `dnf remove nodejs22` retirerait le paquet rpm et ses
dépendances rpm, mais **rien de ce que `npm install`/`tabby-agent`
installerait lui-même** — le modèle de dépendances npm vit hors de la
base rpm (typiquement sous `~/.npm`, `~/.local/lib/node_modules`, ou un
répertoire choisi par l'outil), invisible à `dnf`/`rpm`, donc non
retiré par la même commande. Une porte qui se referme mal : un simple
`dnf remove` ne suffirait pas à annuler l'ouverture.

## 5. Coût de la voie « binaire externe » (`lsp-ai`) — nommé symétriquement au coût npm

Puisque cette voie n'était pas anticipée par la demande, son coût
mérite le même traitement que celui de npm (§ 4), pas moins :

- **Nouvelle surface d'approvisionnement**, hors `docs/repositories.md`
  (ni un dépôt `dnf`, ni un registre de conteneurs comme § 6 de ce
  fichier) — un binaire téléchargé directement depuis GitHub Releases,
  catégorie déjà rencontrée (D7 : COPR, IA-1 : registre de conteneurs)
  mais jamais pour un exécutable autonome sans aucun gestionnaire de
  paquets. Ancrage de confiance le plus faible retenu à ce jour sur ce
  poste (§ 2.1 : TLS seul, aucune empreinte publiée).
- **Un seul mainteneur, projet en pause de développement depuis ~23
  mois** — risque de dérive avec les futures versions d'Ollama non
  couvert par une communauté de maintenance active, contrairement à un
  paquet Fedora ou une image officielle épinglée par empreinte.
- **Retrait** : plus simple que npm — un seul binaire, un seul fichier
  à supprimer, aucune dépendance transitive propre (Rust statique dans
  l'esprit) — porte qui se referme proprement, à l'inverse de npm § 4.
- **Pas de mécanisme de détection de péremption** comparable à
  `verify-cdi-spec` (`roles/gpu_cdi/`) — rien n'alerterait si le binaire
  cesse de fonctionner avec une future version d'Ollama.

## 6. Les trois issues, honnêtement pesées

**Ce que les faits établissent, sans forcer une conclusion qu'ils ne
soutiendraient pas** :

1. **Un mécanisme fonctionne avec Helix, structurellement avec Kate —
   c'est l'issue que les faits soutiennent le plus directement.** Pas
   par npm (D12 reste fermée, rien ne l'exige), pas par changement
   d'éditeur (D13 reste valide) — par un troisième chemin, un binaire
   externe (`lsp-ai`) au coût de confiance propre, nommé § 5, distinct
   de celui envisagé par la demande. **Deux réserves empêchent de
   trancher ici** : Kate n'est pas documenté comme testé (marqueur
   § 2.1) et le binaire n'a aucun ancrage de confiance équivalent à ce
   qui existe déjà sur ce poste pour les autres surfaces
   d'approvisionnement (D7, D10, `docs/repositories.md` § 6) — à
   construire, pas improvisé, si cette voie est retenue.
2. **Si `lsp-ai` est écarté** (fraîcheur, absence d'ancrage, Kate non
   confirmé) **et qu'aucune autre voie sans npm n'apparaît** — la
   question redevient D12 (`tabby-agent`, coût chiffré § 4) : rouvrir
   npm pour un résultat cette fois **prévisible** (protocole standard
   vérifié, pas un pari), pas pour l'espoir vague qui motivait la
   position de l'opérateur en tête de ce livrable.
3. **Si ni l'un ni l'autre n'est retenu** — abandon de la complétion
   résidente en connaissance de cause, D15 révisée dans ce sens,
   RTD3 et les ~8 W préservés. Rappel du cadrage (demande) : la
   complétion était le plus faible des trois usages identifiés en
   IA-0, Claude Code couvre déjà le travail agentique — cette issue
   reste légitime, pas un échec.

**Aucune de ces trois n'est à exclure d'office par les faits établis
ici** — mais l'issue 1, sous réserve de ses deux points ouverts, est
celle qui a le plus de matière derrière elle : un mécanisme réel,
vérifié par lecture de code jusqu'au bout de la chaîne (capacité
déclarée → dispatch de requête → construction de la réponse), pas une
affirmation de documentation.

## Questions qui départagent

1. **L'ancrage de confiance d'un binaire GitHub sans somme de contrôle
   publiée est-il acceptable pour ce poste**, au même titre que Terra
   (D10, risque assumé sans corroboration) ou plus strictement (un
   registre de conteneurs avec empreinte épinglée, `docs/repositories.md`
   § 6) ? Cette réponse seule tranche l'issue 1.
2. **La fraîcheur de `lsp-ai` (dernière publication il y a ~23 mois,
   développement en pause déclaré par le mainteneur) est-elle un motif
   suffisant pour l'écarter**, indépendamment de son protocole standard
   vérifié ?
3. **Si l'issue 1 est écartée, rouvrir D12 pour `tabby-agent`
   (coût chiffré § 4, ~87,5 à 169 Mio, porte qui se referme mal) est-il
   préférable à l'abandon (issue 3) ?**
4. **Une vérification de compatibilité réelle avec Kate spécifiquement
   (marqueur § 2.1) doit-elle précéder toute décision**, ou la preuve
   structurelle (protocole standard, § 1.2) suffit-elle à l'opérateur ?

## Validation — résolution en lecture seule (2026-08-07)

**Commandes exécutées, toutes non modifiantes, chacune justifiée :**

| # | Commande | Nature | Sortie/code |
|---|---|---|---|
| 1 | `rpm -q helix helix-parsers`, `hx --version`, `hx --help`, `hx --health` | lecture locale, binaire installé | sortie normale |
| 2 | `grep`/`wc -l` sur `/usr/share/doc/helix/CHANGELOG.md`, `LICENSE.dependencies` | lecture de fichiers packagés localement | sortie normale, un `grep -c` à `0` (recherché, pas une absence de résultat inattendue) |
| 3 | `rpm -q kate kf6-ktexteditor`, `rpm -ql kate-plugins`, `find` sur `lspclientplugin.so` | lecture locale | sortie normale |
| 4 | `strings -a` sur `lspclientplugin.so` | lecture du binaire installé, extraction de chaînes — pas d'exécution | sortie normale |
| 5 | `dnf repoquery --available`/`--whatprovides` (×4, `lsp-ai`/`llm-ls`/`tabby`/`ai-lsp`, `nodejs`, `npm`) | métadonnées `dnf`, aucun dépôt Terra touché | trois requêtes vides (paquets absents, résultat attendu et recherché), une avec résultats (`nodejs22`) |
| 6 | `dnf install --assumeno nodejs22` (sans `sudo`, vérifié fonctionner sans privilège) | simulation de transaction, gardée par `--assumeno` — **aucun `dnf` touchant `terra` dans ce livrable** | « Operation aborted by the user. », code de sortie non nul par construction (`--assumeno`), pas un échec |
| 7 | `command -v cargo rustc`, `command -v node npm`, `rpm -qa \| grep -iE '^rust\|^cargo\|^node\|^npm'` | lecture locale | vide (rien trouvé), résultat attendu |
| 8 | Lectures externes, nommées : GitHub (`helix-editor/helix` discussions, `SilasMarvin/lsp-ai` code source ×2 fichiers + wiki + releases API, `tommoa/ai-lsp`, `TabbyML/tabby`), `kate-editor.org` | requêtes HTTP GET, documentation/code amont public | sortie normale |

**Incident de méthode reconnu, pas répété** : `sudo dnf install
--assumeno nodejs22` a été exécuté une première fois par réflexe,
avant de vérifier que la version sans privilège donnait le même
résultat (§ tableau, ligne 6) — même défaut que celui déjà reconnu en
IA-0 pour `btrfs filesystem usage /`, corrigé dans la même série,
non répété.

**Énumération des actions privilégiées : aucune**, dans la version
finale retenue de chaque commande — la seule élévation de cette série
(`sudo dnf install --assumeno nodejs22`) était superflue, reconnue
ci-dessus, et sa version non privilégiée est celle citée § 4.

**Confirmations finales** : aucun paquet installé ; `command -v node
npm` toujours vide ; aucun modèle ni image téléchargé ; aucun dépôt
ajouté ; aucune configuration d'éditeur ni de service modifiée ; aucun
fichier écrit hors de ce dépôt ; **aucune commande `dnf` n'a touché
`terra`** dans ce livrable (aucune des recherches de paquets ne
portait sur Terra — vérifié, seuls `fedora`/`updates` apparaissent
dans les résultats `nodejs22`).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : deux
marqueurs actionnables dans ce document — compatibilité réelle de
`lsp-ai` avec le greffon LSP de Kate (§ 2.1), détail exact du
mécanisme de prédiction d'édition natif de Zed avec Ollama (§ 3).

## Voir aussi

- [`docs/local-ai.md`](local-ai.md) § 8.6 — la résolution d'IA-2,
  requalifiée par ce document sans être effacée.
- [`docs/machine-facts.md`](machine-facts.md) — D12/D13 (npm fermé,
  choix d'éditeur), D15 (modèle de complétion résident), requalifiée
  en dernier par ce livrable.
- [`docs/editor.md`](editor.md) — résolution complète D12/D13, motifs
  d'exclusion de Zed/Cursor détaillés § 1.2/1.3/2.
- [`docs/repositories.md`](repositories.md) § 6 — traitement de
  l'ancrage de confiance d'une surface d'approvisionnement externe,
  patron applicable à `lsp-ai` si cette voie est retenue.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici, en
  particulier la garde sur les découvertes qui contredisent un fait
  déjà documenté.
