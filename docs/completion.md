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

**[CMP-1, 2026-08-07] Décisions de l'opérateur prises sur la base de ce
qui précède — D19, D20.** Texte complet, motifs et risques assumés :
`docs/machine-facts.md` § Décisions (numéros vérifiés libres avant
attribution). Résumé : **D19** — complétion locale par `lsp-ai`,
**compilé depuis les sources** (jamais le binaire précompilé, dont
l'absence de somme de contrôle publiée en faisait un ancrage de
confiance plus faible que npm) ; **D20** — modèle de complétion **à la
demande, pas résident**, requalifiant la résidence permanente décidée
en IA-1/IA-2 (`docs/machine-facts.md` § D15, marquée `[REQUALIFIÉE]`).
L'exécution de D19 (`roles/completion/`), avec ses obstacles réels et
ses corrections, est documentée § 7 ci-dessous — ajoutée après la
résolution en lecture seule qui suit, pas à sa place.

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
spécifiquement. **[RÉSOLU le 2026-08-08, KAT-1] ÉTAIT** : le marqueur
de vérification portait sur la compatibilité réelle de `lsp-ai` avec
le greffon LSP client de Kate — non testée par le projet amont
(silence, pas un échec documenté) ni par ce livrable (lecture seule,
aucun binaire téléchargé). **Testée depuis, résultat positif** :
complétions FIM réelles reçues sur YAML et Python, même binaire et
même configuration que Helix — § 9 pour la preuve complète.

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

## Questions qui départagent — répondues (CMP-1, D19/D20)

Les quatre questions ci-dessous ont reçu une réponse de l'opérateur
(`docs/machine-facts.md` § D19/D20) — conservées telles quelles pour
la trace, pas effacées.


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

## Validation — CMP-0, résolution en lecture seule (2026-08-07)

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

## 7. Exécution — `lsp-ai` compilé et intégré à Helix (CMP-1, 2026-08-07)

**Aucun modèle chargé ni choisi dans ce livrable** (D17/D20, toujours
en vigueur). Périmètre : `roles/completion/` (nouveau),
ce document, `docs/repositories.md` § 7, `docs/machine-facts.md`
(en dernier), `CLAUDE.md` (règle 0.3 uniquement), écritures utilisateur
sous `~/.config/helix/` et `~/.local/bin/`.

### 7.1 — Résolution avant compilation

**Chaîne Rust, lue avant d'être supposée** :
```
$ dnf repoquery --available --qf '%{name}-%{version}-%{release} | %{repoid}' rust cargo
cargo-1.94.1-1.fc44 | fedora
cargo-1.97.1-1.fc44 | updates
rust-1.94.1-1.fc44 | fedora
rust-1.97.1-1.fc44 | updates
```
`fedora`/`updates` uniquement — **aucune commande touchant `terra`**
n'a été nécessaire pour cette recherche. `lsp-ai` ne déclare aucun
`rust-version` (MSRV) explicite — lu directement dans
`crates/lsp-ai/Cargo.toml` et le `Cargo.toml` racine (`edition = 2021`
délégué au workspace, aucun champ `rust-version`) — la seule
confirmation possible que la version disponible (1.97.1) suffit est la
compilation réelle elle-même (§ 7.2-7.3), pas une comparaison de
numéros.

**Cargo.lock, présent, vérifié avant tout clonage réel** — le rôle
s'arrête si absent (garde dédiée, `roles/completion/tasks/main.yml`).
104 548 octets à l'empreinte retenue.

**Empreinte retenue, pas une étiquette** — la plus récente étiquette
(`v0.7.1`, 2024-09-24T14:49:55Z, établie en CMP-0 § 2.1) **n'est pas**
le dernier état du dépôt : `main` a avancé depuis, jusqu'à
`1e910a8cf0048406eb227bf2064743010a9ff3a9` (2025-01-07T22:17:34Z,
API GitHub, `git/refs/heads/main` puis `commits/<sha>`) — retenue ici
précisément parce qu'une étiquette peut être déplacée, pas un commit.
`Cargo.lock` confirmé présent à ce commit exact (même source).

### 7.2 — Obstacles réels rencontrés, signalés avant correction

Deux catégories d'obstacles, aucune anticipée par la lecture seule de
Cargo.toml — découverts en compilant réellement, pas en lisant les
manifestes.

**7.2.1 — `hf-hub`, dépendance git sans rev fixe, étiquettes amont
retirées.** `cargo build --release --locked` échoue tel quel :
```
error: failed to select a version for the requirement `hf-hub = "^0.3.2"`
candidate versions found which didn't match: 1.1.0
location searched: Git repository https://github.com/huggingface/hf-hub
```
Lu directement : `crates/lsp-ai/Cargo.toml` déclare
`hf-hub = { git = "...", version = "0.3.2" }` — pas de `rev`.
`Cargo.lock` pointe pourtant un commit précis
(`git+https://github.com/huggingface/hf-hub#6303587...`). Confirmé :
le dépôt amont hf-hub a retiré ses étiquettes 0.3.x
(`git ls-remote --tags https://github.com/huggingface/hf-hub` ne
montre que `v0.4.0` et plus récent) — Cargo doit revalider la
correspondance de version à chaque résolution, même verrouillée,
échec **confirmé en ligne et hors ligne** (`--offline` produit la même
erreur). **Signalé à l'opérateur avant toute correction** (le motif de
D19 — l'intégrité portée par Cargo.lock — semblait s'effondrer sur ce
point précis). Deux voies écartées après essai réel : un recouvrement
de source Cargo (`[source]`/`replace-with`) — Cargo exige lui-même un
lockfile déjà cohérent avec le remplacement actif avant de pouvoir
s'en servir (chicken-and-egg, message d'erreur Cargo cité dans
`roles/completion/tasks/main.yml`) ; un `rev=` direct dans
`crates/lsp-ai/Cargo.toml` — change la forme de la dépendance,
invalide l'entrée verrouillée existante, aurait entraîné une
résolution beaucoup plus large (`cargo update --dry-run` montrait des
dizaines de paquets sans rapport proposés à la mise à jour). **Retenu,
vérifié avant d'être encodé dans le rôle** : `cargo update -p hf-hub
--precise <le même commit que Cargo.lock épingle déjà>` — confirmé par
`git diff` : l'entrée `hf-hub` elle-même **ne change pas** ; l'opération
corrige un décalage sans rapport, préexistant dans le Cargo.lock amont
lui-même (auto-version du paquet `lsp-ai` : `"0.7.0"` au lieu de
`"0.7.1"` déclaré par son propre `Cargo.toml` à ce commit — jamais
régénéré par le mainteneur après ce changement), et bascule le format
de fichier de verrouillage de 3 à 4. **262 dépendances non touchées**
(« pass `--verbose` to see 262 unchanged dependencies behind latest »).

**7.2.2 — Modules Perl absents, requis par la configuration d'OpenSSL
vendoré.** `reqwest` (dépendance par défaut, pas seulement du feature
`llama_cpp`) tire `openssl-sys`, qui compile OpenSSL depuis les
sources — son script `Configure`/`configdata.pm` exige plusieurs
modules Perl. Établi par lecture exhaustive des `use`/`require` des
fichiers réellement invoqués (pas une estimation) :
```
$ grep -rhoE '^\s*(use|require)\s+[A-Za-z0-9_:]+' Configure configdata.pm util/perl/OpenSSL | sort -u
```
Cinq modules manquants, trouvés successivement au fil des échecs
(chacun bloquant le suivant, pas tous visibles en une fois) :
`FindBin`, `IPC::Cmd`, `File::Compare`, `IO::Socket::INET6`,
`Text::Template` — paquets `perl-FindBin`, `perl-IPC-Cmd`,
`perl-File-Compare`, `perl-IO-Socket-INET6`, `perl-Text-Template`,
tous `fedora`/`updates` (vérifié, `dnf repoquery --available --qf
'%{repoid}'`), aucun `terra`. Encodés dans le rôle
(`completion_build_extra_packages`), même exception que rust/cargo.

**7.2.3 — GCC système (16.1.1) compile en C23 par défaut, incompatible
avec le C vendoré K&R d'`oniguruma`.** Après les deux corrections
ci-dessus, `onig_sys` (bibliothèque C vendorée, tirée par `onig`, tiré
par `tokenizers` — dépendance directe de `lsp-ai`) échoue :
```
error: too many arguments to function 'func'; expected 0, have 3
```
Établi par lecture directe : `gcc -dM -E -x c /dev/null | grep
__STDC_VERSION__` → `202311L` (C23) — en C23, un prototype à liste
d'arguments vide (`int (*)(void)`, motif historique très répandu dans
le C des années 1990) signifie littéralement « zéro argument »,
contre « non spécifié » dans les normes C antérieures. Le code source
vendoré d'`oniguruma` (fixé par la version épinglée de la crate
`onig_sys`, pas modifiable sans casser l'épinglage) utilise ce motif
partout. **Essayé et insuffisant seul** : suppression ciblée des
avertissements-erreurs (`-Wno-error=incompatible-pointer-types`,
`-Wno-error=implicit-function-declaration`) — repousse l'échec sans le
résoudre, l'erreur de C23 sur les listes d'arguments n'est pas
associée à un avertissement désactivable, c'est un changement de
sémantique du langage. **Retenu, vérifié avant d'être encodé** :
`CFLAGS=-std=gnu17` — restaure le dialecte C par défaut de GCC avant
la bascule à C23, celui sous lequel ce code vendoré a toujours été
écrit et testé en amont. Pas une désactivation de vérification : une
norme de langage, appliquée uniquement à la compilation C déclenchée
par ce rôle (variable d'environnement du seul processus `cargo
build`), sans toucher au compilateur système ni à aucun autre paquet.

**7.2.4 — Reconstructibilité D1 : `cargo vendor` évalué, écarté en
bloc, mitigation ciblée retenue (IA-4, § 4 de la demande).** §7.2.1
établit que l'intégrité n'est pas menacée (`Cargo.lock` porte
l'empreinte du commit `hf-hub` retenu, `cargo build --locked` échoue
plutôt que de dériver silencieusement) — mais que la **disponibilité**
l'est : le dépôt amont `hf-hub` a déjà retiré ses étiquettes 0.3.x, rien
n'empêche un retrait ou une réécriture d'historique qui rendrait le
commit épinglé (`6303587...`) inatteignable. Un même risque, moins
documenté jusqu'ici, touche le dépôt `lsp-ai` lui-même
(`completion_lsp_ai_commit`, épinglé de la même façon).

**`cargo vendor`, chiffré réellement, pas supposé** — exécuté dans le
répertoire de travail à l'empreinte épinglée :
```
$ cd ~/.cache/completion-build/lsp-ai && cargo vendor vendor-eval-tmp
   Vendoring [...] (422 crates)
$ du -sh vendor-eval-tmp
710M    vendor-eval-tmp
$ tar --zstd -cf lsp-ai-vendor-eval.tar.zst vendor-eval-tmp && du -sh lsp-ai-vendor-eval.tar.zst
71M     lsp-ai-vendor-eval.tar.zst
```
**710 Mio bruts, 71 Mio compressés (zstd), 422 crates** — comparé aux
**2,8 Mio** du `.git` de ce dépôt dans son état actuel (`du -sh .git`) :
un facteur ~25 même compressé. **Écarté en bloc** : disproportionné, et
mal ciblé — la quasi-totalité des 422 crates vendorées viennent de
crates.io, dont le modèle de distribution ne partage pas, à ma
connaissance, le risque de `hf-hub`/`lsp-ai` (une version publiée peut
être « yankée » — retirée de la résolution pour de *nouveaux* projets —
mais resterait téléchargeable pour ce qui la référence déjà, à
distinguer d'un retrait de tag ou d'une réécriture d'historique Git,
qui rend un commit précis potentiellement inatteignable) — comportement
exact non lu directement dans la documentation de crates.io au cours de
cette session, marqué `@VERIF` plus bas plutôt qu'affirmé sans réserve.
Vendorer les 420 crates crates.io pour protéger les 2 dépendances git
reviendrait à corriger un risque
localisé par une dépense généralisée.

**Mitigation ciblée retenue à la place** : une archive compressée de
chaque dépôt git à sa **seule empreinte épinglée** (pas tout
l'historique disponible, pas les 420 crates sans rapport), conservée
**hors dépôt** (répertoire personnel, pas ce dépôt Git — même principe
que le stockage des poids de modèle, `docs/local-ai.md` § 5 : la
recette est versionnée, l'artefact volumineux ne l'est pas), avec son
empreinte consignée ici pour que sa présence ou son absence soit
vérifiable :
```
$ mkdir -p ~/.local/share/completion-build-archives
$ cd ~/.cargo/git/checkouts/hf-hub-3c7263c594854a05/6303587
$ tar --zstd -cf ~/.local/share/completion-build-archives/hf-hub-6303587576f8a1ce9f91f8274265a153b89afb6e.tar.zst .git
$ sha256sum ~/.local/share/completion-build-archives/hf-hub-6303587576f8a1ce9f91f8274265a153b89afb6e.tar.zst
1613af4f530f38621013bf87e2fc4bfa2cc12b84bfe199ef4d0ed6d188ec846e  hf-hub-6303587576f8a1ce9f91f8274265a153b89afb6e.tar.zst
$ du -sh hf-hub-6303587576f8a1ce9f91f8274265a153b89afb6e.tar.zst
88K     hf-hub-6303587576f8a1ce9f91f8274265a153b89afb6e.tar.zst
```
**88 Kio** — l'archive ne contient que `.git` (objets + historique,
pas un arbre de travail extrait), **vérifié fonctionnelle, pas
seulement créée** : extraite dans un répertoire vide, `git log -1`
rapporte exactement `6303587576f8a1ce9f91f8274265a153b89afb6e`, `git
rev-parse HEAD` concorde. `lsp-ai` lui-même n'a **volontairement** pas
reçu d'archive équivalente ici : son clonage à l'empreinte épinglée
(`completion_lsp_ai_commit`) est la toute première action de ce rôle,
rejouée à **chaque exécution** — si ce commit devenait inatteignable en
amont, ce rôle échouerait bruyamment dès sa première tâche, sans
attendre une reconstruction complète. C'est un canari vivant, revérifié
à chaque exécution, préférable à une archive statique qui pourrait se
périmer sans que personne ne le remarque — `hf-hub` n'a pas cette
propriété (dépendance résolue par Cargo *pendant* la compilation, pas
clonée explicitement par une tâche de ce rôle), d'où l'archive ciblée
sur ce seul dépôt.

**Régénération** (si l'archive venait à manquer ou si l'empreinte
épinglée changeait) :
```
$ ansible-playbook roles/completion/completion.yml   # peuple ~/.cargo/git/checkouts
$ tar --zstd -cf ~/.local/share/completion-build-archives/hf-hub-<commit>.tar.zst \
    -C ~/.cargo/git/checkouts/hf-hub-*/<commit-court> .git
```
**Risque résiduel assumé, nommé plutôt que corrigé** : cette archive
vit hors dépôt, sur cette seule machine — sa propre disponibilité
dépend de ce poste (D1 ne l'exige pas ailleurs : ce dépôt Git reste la
seule source de vérité versionnée, l'archive est un filet local, pas
un second dépôt).

@VERIF : politique exacte de rétention de crates.io pour une version
« yankée » (une version yankée reste-t-elle indéfiniment
téléchargeable pour qui la référence déjà, ou existe-t-il une voie de
suppression complète ?) — le motif d'exclusion des 420 crates
crates.io ci-dessus en dépend partiellement, à confirmer par lecture de
la documentation crates.io elle-même avant de le considérer clos.

### 7.3 — Rôle exécuté, résultat

```
$ ansible-playbook roles/completion/completion.yml   # première exécution réelle, depuis zéro
[...] changed=6
$ ansible-playbook roles/completion/completion.yml   # deuxième exécution
[...] changed=0
```
**Idempotence non triviale à obtenir** : la correction du décalage de
Cargo.lock (§ 7.2.1) modifie délibérément le clone par rapport au
commit épinglé — `ansible.builtin.git` refuse de revérifier un commit
déjà atteint dès que des modifications locales existent, même sans
rien à faire (« Local modifications exist », constaté avec `force:
true` avant d'être retiré). Corrigé : le clonage et la correction du
lockfile ne s'exécutent que si le répertoire de travail n'existe pas
encore — une fois fait, il reste tel quel, jamais reforcé.

**Empreintes** :
```
$ cargo --version    # chaîne utilisée pour cette compilation
cargo 1.97.1 (c980f4866 2026-06-30) (Fedora 1.97.1-1.fc44)
$ sha256sum target/release/lsp-ai
7bc6e76e296861d958ab369131de433457f6226b994487f9b418ae55ea8f9159
$ ./target/release/lsp-ai --version
lsp-ai 0.7.1
```
**Identique sur deux compilations propres indépendantes** (répertoire
de travail supprimé puis reconstruit entièrement entre les deux) —
même empreinte de binaire les deux fois, cohérent avec une compilation
verrouillée et déterministe.

### 7.4 — Preuve de la poignée de main LSP, par lecture du journal Helix

**Méthode** : `hx -vv` (journal détaillé,
`~/.cache/helix/helix.log`), piloté sans capture visuelle via une
session `tmux` détachée — fichier ouvert, quelques secondes
d'attente, fermeture propre (`:q!`), lecture du journal après coup.
Journal vidé avant chaque essai pour n'observer que les lignes
produites par cet essai précis.

**Nominal** — extraits réels, horodatage `21:55:38` :
```
lsp-ai -> {"jsonrpc":"2.0","method":"initialize","params":{...
  "clientInfo":{"name":"helix","version":"25.07.1 (a05c151b)"},
  "initializationOptions":{"completion":{"model":"completion-model",...},
  "models":{"completion-model":{"chat_endpoint":"http://127.0.0.1:11434/api/chat",
  "generate_endpoint":"http://127.0.0.1:11434/api/generate",
  "model":"aucun-modele-charge-D20","type":"ollama"}}},...}}
lsp-ai <- {"jsonrpc":"2.0","id":0,"result":{"capabilities":{
  "codeActionProvider":{"resolveProvider":true},
  "completionProvider":{},"textDocumentSync":2}}}
lsp-ai -> {"jsonrpc":"2.0","method":"initialized","params":{}}
lsp-ai -> {"jsonrpc":"2.0","method":"workspace/didChangeConfiguration",...}
lsp-ai -> {"jsonrpc":"2.0","method":"textDocument/didOpen","params":{
  "textDocument":{"languageId":"yaml",...}}}
[...]
lsp-ai -> {"jsonrpc":"2.0","method":"shutdown","id":1}
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":null}
lsp-ai -> {"jsonrpc":"2.0","method":"exit"}
```
**`completionProvider":{}` dans la réponse `initialize`** — le serveur
annonce bien une capacité de complétion, exactement ce qui devait être
prouvé, lu dans le journal, jamais capturé visuellement. Poignée de
main complète : `initialize` → réponse avec capacités → `initialized`
→ configuration envoyée → document ouvert → arrêt propre sur `:q!`.

**Échec forcé — configuration Helix neutralisée** (`command` du
serveur pointé vers un binaire inexistant, fichier restauré à
l'identique immédiatement après, `diff` vérifié) :
```
Language server not found for `source.yaml` lsp-ai command
'/home/mahieumi/.local/bin/lsp-ai-binaire-inexistant' not found:
cannot find binary path
```
**Aucune ligne `initialize`, aucune capacité annoncée** — le journal
est visiblement différent du cas nominal, pas identique : la
démonstration prouve quelque chose, conformément à l'exigence de la
demande.

**Second échec forcé — une garde du point 2.1 casse avant toute
action**, trois exemples, tous `changed=0`, aucun n'a modifié le
système réel :
```
$ ansible-playbook roles/completion/completion.yml -e '{"completion_forbidden_binaries": ["sh"]}'
fatal: [...] "Un binaire parmi sh est présent sur le PATH (/usr/bin/sh) [...]"
$ ansible-playbook roles/completion/completion.yml -e '{"completion_rust_check_binaries": ["cargo_absent_garanti"]}'
fatal: [...] "Chaîne Rust attendue introuvable ou non fonctionnelle (cargo_absent_garanti) [...]"
$ ansible-playbook roles/completion/completion.yml -e completion_ollama_url_check=http://127.0.0.1:1/api/tags
fatal: [...] "http://127.0.0.1:1/api/tags ne répond pas (statut : -1) [...]"
```
Plus une quatrième, sur la garde de relecture de configuration (§ 2.4
de la demande), substituant uniquement la valeur *attendue* sans
jamais modifier ce qui est réellement déployé :
```
$ ansible-playbook roles/completion/completion.yml -e completion_ollama_generate_url_expected=http://exemple.invalide/api/generate
fatal: [...] "La configuration déployée [...] ne pointe pas explicitement 127.0.0.1:11434 [...]"
```
Confirmé après coup : le fichier réellement déployé pointe toujours
`127.0.0.1:11434`, jamais modifié par cette démonstration.

### 7.5 — Ce que ce livrable ne peut pas prouver, pas maquillé en succès

Aucun modèle chargé (D17/D20) : `lsp-ai` répondrait `null` ou une
liste vide à une complétion réellement demandée avec le modèle
placeholder `aucun-modele-charge-D20` — non testé ici, et non
testable sans violer D17/D20 ou télécharger un modèle. La poignée de
main LSP et l'annonce de capacité (§ 7.4) sont ce que ce livrable
établit ; la qualité ou même l'existence d'une complétion réelle
reste à démontrer dans un livrable qui charge un modèle.

## Validation — CMP-1 (2026-08-07)

**Actions privilégiées, exhaustives** :

| # | Commande | Cible | Motif | Précédée d'une tentative sans privilège ? |
|---|---|---|---|---|
| 1 | `ansible.builtin.dnf` (`become: true`, rôle) | `rust`, `cargo` | seule installation système accordée par la demande | Oui — `command -v cargo && command -v rustc` (§ rôle) |
| 2 | `ansible.builtin.dnf` (`become: true`, rôle) | `perl-FindBin`, `perl-IPC-Cmd`, `perl-File-Compare`, `perl-IO-Socket-INET6`, `perl-Text-Template` | dépendances de compilation d'`openssl-sys` (§ 7.2.2) | Oui — `perl -MFindBin -MIPC::Cmd -MFile::Compare -MIO::Socket::INET6 -MText::Template -e 1` |
| 3 | `sudo dnf install -y --disablerepo=terra perl-IPC-Cmd` (manuel, hors rôle, diagnostic) | paquet système | débloquer la compilation pour établir le rôle | **Non — règle 0.3 violée**, reconnu ci-dessous |
| 4 | `sudo dnf install -y --disablerepo=terra perl-File-Compare perl-IO-Socket-INET6 perl-Text-Template` (manuel, hors rôle, diagnostic) | paquets système | idem | Oui — `perl -M<module> -e1` pour chacun, avant élévation |

**Incident de méthode reconnu, pas répété** (règle 0.3, ajoutée dans
*ce même livrable*, § 0.3) : l'action privilégiée n°3 a été exécutée
par réflexe, sans tentative sans privilège préalable — alors que la
règle venait d'être écrite quelques instants plus tôt dans cette même
session. `perl -MIPC::Cmd -e1` aurait dû précéder, comme cela a
correctement été fait juste après pour les trois modules suivants
(action n°4). Signalé explicitement plutôt que laissé implicite dans
le tableau — la règle 0.3 existe justement pour des cas comme
celui-ci, elle ne s'applique pas si elle n'est vérifiée que quand
c'est facile.

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/completion/completion.yml   # succès
$ ansible-playbook --check roles/completion/completion.yml          # succès, changed=0
$ ansible-playbook roles/completion/completion.yml                  # succès, changed=6 (première exécution, depuis zéro)
$ ansible-playbook roles/completion/completion.yml                  # succès, changed=0 (deuxième exécution, idempotence confirmée)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/completion/
Passed: 0 failure(s), 0 warning(s) — profil production
```

**Quatre démonstrations d'échec forcé sur les gardes** (§ 7.4
ci-dessus, toutes `changed=0`) **plus deux démonstrations sur la
poignée de main LSP elle-même** (nominal avec capacité de complétion
annoncée ; configuration neutralisée, aucune capacité, § 7.4) — six
démonstrations au total, chacune contrastée avec son cas nominal.

**SELinux** :
```
$ getenforce   # Enforcing, avant et après ce livrable
```

**Confirmations finales** : aucun modèle téléchargé ; **aucun binaire
`lsp-ai` précompilé récupéré, sous aucune forme** (compilé depuis les
sources à chaque fois, empreinte de binaire identique sur deux
compilations propres indépendantes, § 7.3) ; `command -v node npm`
toujours vide ; aucun nouveau dépôt (`crates.io` et le dépôt git
`lsp-ai` ne sont pas des dépôts `dnf`) ; aucune commande `dnf` n'a
touché `terra` ; `getenforce` inchangé ; aucun redémarrage.

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : les deux
marqueurs déjà comptés en CMP-0 restent ouverts, inchangés par ce
livrable — compatibilité réelle de `lsp-ai` avec le greffon LSP de
Kate (§ 2.1, toujours non testée : « Kate seul dans ce livrable »,
demande § 2.6, respecté ; **résolu depuis, 2026-08-08, KAT-1, § 9**),
détail exact du mécanisme de prédiction d'édition natif de Zed avec
Ollama (§ 3, toujours ouvert). Aucun nouveau marqueur
ajouté par ce livrable — les obstacles de compilation (§ 7.2) ont
tous été résolus et vérifiés, pas laissés ouverts.

## 8. IA-4 — complétion réelle bout-en-bout, mesurée

**Troisième cause du blocage, isolée avant ce test (demande IA-4 § 1)** :
ni `lsp-ai` (poignée de main LSP prouvée, § 7.4) ni RTD3 (déjà mesuré
neutralisé par la résidence du modèle, D15) — le nom de modèle
placeholder de CMP-1 (`aucun-modele-charge-D20`), resté en place dans
`languages.toml` après que D21 (IA-3) a choisi et récupéré un modèle
réel, causait `"model 'aucun-modele-charge-D20' not found"` côté
serveur (visible seulement dans le journal LSP, jamais dans
l'interface — exactement le mode d'échec silencieux que la nouvelle
garde de `roles/completion/tasks/main.yml` rend désormais bruyant,
avant même l'écriture de la configuration). Corrigé (§ défauts/main.yml,
`completion_ollama_model`), démontré ci-dessous.

**Méthode** : identique à § 7.4 (`hx -vv`, session `tmux` détachée,
lecture du journal après coup, aucune capture visuelle) — avec, cette
fois, un modèle réel chargé et une complétion réellement demandée.

**Écart initial avec IA-3, résolu avant de conclure, pas laissé en
l'état.** `docs/machine-facts.md` (D21) rapporte l'envoi automatique de
requêtes de complétion pendant la frappe (`triggerKind:1`, observé deux
fois). Mes premiers essais, une **frappe isolée** (un seul caractère)
suivie d'une attente de 20-30 s, n'ont **jamais** déclenché de requête
automatique malgré un `textDocument/didChange` bien émis — en
contradiction apparente avec ce fait déjà documenté et daté
(CLAUDE.md § une découverte qui contredit un fait déjà documenté se
signale, ne se tranche pas à la première lecture). **Reproduit avant de
conclure à un écart réel** : une séquence de frappe plus naturelle
(plusieurs caractères tapés à la suite, ~150 ms d'écart, comme une
vraie frappe) déclenche bien la requête automatique, ~250 ms après le
dernier caractère (`idle-timeout` par défaut de Helix) — confirmé,
cohérent avec IA-3. **Pas de contradiction réelle** : la frappe isolée
et unique ne déclenche pas de façon fiable (`@VERIF : condition exacte`
— non expliqué avec certitude, hypothèse la plus probable : un artefact
du pilotage `tmux` sur une frappe unique, pas un changement de
comportement de Helix ou de la configuration entre IA-3 et ce
livrable), mais la frappe réelle, multiple, le fait de façon
reproductible. Sans conséquence sur les mesures ci-dessous : le
déclenchement **manuel** (`Ctrl-x` en mode insertion, lié par défaut)
produit exactement la même requête `textDocument/completion` — retenu
pour le reste de cet essai pour un contrôle précis du moment de la
requête, condition nécessaire à une mesure de latence propre.

### 8.1 — YAML et Python, modèle déjà chargé

Fichier YAML (contexte Ansible, `ansible.builtin.` en position de
curseur), fichier Python (`if n <` dans une fonction Fibonacci) —
horodatages du journal, requête → réponse :
```
YAML #1 : 13:58:55.571 -> 13:59:00.285  = 4,714 s
Python  : 14:00:04.798 -> 14:00:05.276  = 0,478 s
YAML #2 : 14:02:54.716 -> 14:02:55.180  = 0,464 s
```
**Complétions réellement reçues, pas des accusés de réception vides** —
YAML : `ansible.builtin.synchronize:` avec des paramètres cohérents
(`src`, `dest`, `mode: pull`) ; Python : garde `if n < 0: raise
ValueError(...)` suivie du cas de base `elif n == 0: return 0` —
suggestions syntaxiquement valides et sémantiquement pertinentes au
contexte, pas un texte générique.

**YAML #1 anormalement lente par rapport à YAML #2 et Python, mêmes
conditions (modèle déjà résident, aucune bascule)** — écart nommé, pas
lissé : YAML #1 était la **première génération réelle** demandée à ce
modèle depuis un moment (les précédentes exécutions de ce rôle et de
`roles/local_ai/` n'avaient fait que vérifier la présence du modèle,
`/api/ps`, jamais généré). YAML #2, déclenchée quelques minutes plus
tard sur le même modèle entre-temps sollicité, retombe à 0,464 s —
cohérent avec un coût de réchauffement (allocation de cache KV,
première passe) payé une fois puis amorti, pas avec une latence
normale du chemin complet. Retenu pour "modèle déjà chargé, état
régulier" : **0,46-0,48 s**, pas la valeur du tout premier appel.

### 8.2 — Bascule depuis le modèle de chat (D22), mesurée en conditions réelles

**Écart méthodologique corrigé avant de conclure** : un premier essai
(`/api/generate` sur `mistral-nemo` sans `num_ctx` explicite, défaut
Ollama 4096) a laissé les **deux modèles chargés simultanément**
(`/api/ps` : qwen 4,75 Gio + mistral 7,88 Gio = 12,66 Gio sur 16,38 Gio
— tient dans l'enveloppe). Résultat trompeur : la mesure IA-3 qui fonde
D22 (16 984 Mio requis) porte sur le **contexte réellement visé par
D21 pour le modèle de chat/agent — 32 K**, pas le défaut Ollama. Rejoué
avec `num_ctx=32768` explicite : `mistral-nemo` seul occupe alors
**12 611 548 609 octets (~12 027 Mio) de VRAM à lui seul**, `qwen`
évincé automatiquement (`/api/ps` : un seul modèle listé après coup) —
**confirme D22 dans ses propres termes**, le premier essai ne le
contredisait pas, il mesurait une configuration différente de celle
réellement déployée. Signalé plutôt que tranché à la première lecture
(CLAUDE.md § une découverte qui contredit une contrainte établie se
signale) — l'écart s'est résolu en écart de méthode, pas en fait
nouveau, mais seulement après l'avoir creusé.

**Bascule mesurée, chat → complétion, conditions réelles** (`mistral-nemo`
chargé à 32 K de contexte, comme ci-dessus, puis une complétion Python
réellement demandée) :
```
14:01:49.415 -> 14:01:53.141  = 3,727 s
```
`/api/ps` après coup : seul `qwen2.5-coder` chargé, `mistral-nemo`
évincé — la bascule a bien eu lieu, pas une coïncidence de timing.
Complétion reçue cohérente (`is_prime`, garde `n < 2: return False`
suivie d'une boucle de test de divisibilité). **Dans la fourchette
citée par l'opérateur (3,4-4,1 s)** — mesurée ici, pas seulement
retenue par confiance dans la mesure précédente.

### 8.2bis — Démonstration de la nouvelle garde (§ 1 de la demande), les deux sens

**Échec forcé** — le nom de modèle réellement configuré (`completion_ollama_model`)
n'est jamais substitué, seul l'attendu que la garde relit l'est (même
patron que les gardes existantes) :
```
$ ansible-playbook roles/completion/completion.yml -e completion_ollama_model_expected=absent-garanti
fatal: [...] "Le modèle « absent-garanti » n'existe pas côté service — modèles
  présents : ['qwen2.5-coder:7b-instruct-q4_K_M', 'mistral-nemo:12b-instruct-2407-q4_K_M'].
  [...]" changed=0
```
**Nominal, rejoué juste après** : `ansible-playbook roles/completion/completion.yml`
→ `changed=0`, `languages.toml` toujours `model = "qwen2.5-coder:7b-instruct-q4_K_M"`
— confirmé par relecture directe du fichier déployé, pas seulement par
l'absence d'erreur.

### 8.3 — Verdict

**Utilisable au quotidien** : 0,46-0,48 s en régime établi (modèle déjà
résident, cas dominant si la complétion est l'usage principal en
cours), 3,7 s lors d'une bascule depuis le modèle de chat — perceptible
mais du même ordre qu'un correcteur/analyseur lourd qui se relance,
pas une rupture du flux de travail. Le chargement séquentiel (D22)
reste vivable : le coût mesuré est celui d'une bascule occasionnelle
(changer d'activité, pas chaque frappe), jamais celui du régime normal
de la complétion elle-même.

**Non résolu par ce test, nommé plutôt que passé sous silence** :
le déclenchement automatique par une **frappe isolée unique** reste
`@VERIF` (§ méthode ci-dessus — reproduit et confirmé fiable pour une
frappe réelle, multiple ; jamais reproduit pour une frappe unique).
Sans conséquence sur le verdict ci-dessus (l'ergonomie normale d'écriture
tape plusieurs caractères à la suite, jamais un caractère isolé suivi
d'une longue pause volontaire — le cas qui échoue dans ce test n'est
pas le cas d'usage réel). Le premier chargement après démarrage du
service (cache disque froid, pas seulement VRAM) reste également
`@VERIF` — écart déjà nommé dans la décision D22 elle-même
(`docs/machine-facts.md` § Décisions), non comblé par cet essai (le
service tournait déjà depuis la veille pendant toute cette session).

## 9. KAT-1 — Kate intégré et prouvé (2026-08-08)

**Dernier point ouvert de la série CMP-1/IA-4 : compatibilité réelle de
`lsp-ai` avec le greffon LSP client de Kate (§ 2.1), jamais testée par
personne.** Résultat : **positif** — Kate reçoit des complétions FIM
réelles sur YAML et Python, avec exactement le même binaire et la même
configuration qu'Helix (§ 2 ci-dessous). Aucun risque système engagé
(rappel de cadrage de la demande) ; le résultat aurait été documenté
tel quel s'il avait été négatif.

**[Marquage le 2026-08-09, clôture de série] Nature de la preuve —
mixte, pas du même niveau que Helix.** Pour Helix (§ 7-8), les deux
moitiés de la démonstration sont **machine** : une session `tmux`
détachée pilote le clavier, `hx -vv` produit le journal — bout en
bout automatisé, rejouable sans intervention humaine. Pour Kate
(§ 9.3-9.4 ci-dessous), **seule la lecture du journal LSP est une
preuve directe** (machine, rejouable) ; **le déclenchement — clic puis
Ctrl+Espace — relève de la classe « observation rapportée par
l'opérateur »** (`CLAUDE.md` § Sourcing des faits — **[correction
d'attribution]** ajoutée le 2026-08-05, pas en CMP-1 comme la demande
de clôture le nommait ; `docs/machine-facts.md`, journal daté
2026-08-05, `roles/gpu_mux/`, pas ce document) : plus faible qu'une
trace complète, plus forte qu'une inférence, recevable
ici parce qu'elle remplit les trois conditions posées par cette
classe — **marquée** explicitement comme telle (cette note),
**datée** (2026-08-08, § 9.3-9.4), **attribuée** (l'opérateur, via les
échanges retranscrits § 9.3). Motif de cette distinction : sans elle,
une session future lirait « Kate prouvé » et « Helix prouvé » comme
équivalents, alors qu'une défaillance future de la voie Kate (Ctrl+Espace
qui ne déclenche plus rien, par exemple) ne serait pas rejouable de la
même façon qu'une défaillance Helix — il faudrait de nouveau un
opérateur au clavier, pas seulement une session détachée relancée.
Ce que la preuve établit reste entier (le journal montre une requête
`textDocument/completion` réelle et sa réponse, § 9.4) — seule la
**reproductibilité non assistée** diffère.

### 9.1 — Résolution avant configuration

**Greffon déjà présent, aucune installation nécessaire.** `kate-plugins`
(fournissant `lspclientplugin.so`) est installé sur ce poste depuis
2026-08-06 (`roles/editor/`, EDI-1) — vérifié, `rpm -q kate-plugins`,
`dnf repoquery --requires kate-plugins` (aucune dépendance `node`/`npm`).
La branche « installation si besoin, arrêt si npm tiré » de la demande
ne s'applique donc pas.

**Fichier de configuration et format, sourcés dans le code amont de
Kate à la version exacte installée (`v26.04.3`)**, pas supposés :
- Chemin : `~/.config/kate/lspclient/settings.json` — dérivé de
  `QStandardPaths::AppConfigLocation` + `/lspclient/settings.json`
  (`addons/lspclient/lspclientplugin.cpp`), confirmé aussi par
  inspection directe du système de fichiers (le répertoire existait
  déjà, créé automatiquement au premier chargement du greffon, vide).
- Format : objet JSON `{"servers": {"<langid>": {...}}}`, **fusionné**
  (`json::merge`, pas remplacé) avec les définitions livrées avec Kate
  (`addons/lspclient/settings.json` amont) — une entrée utilisateur
  l'emporte, pour ce seul identifiant de langage, sur celle par défaut.
- Clés par serveur utilisées ici : `command`, `url`,
  `highlightingModeRegex`, `initializationOptions` (passées à la
  requête LSP `initialize`), `settings` (envoyées via
  `workspace/didChangeConfiguration`) — sourcées dans
  `lspclientservermanager.cpp`.

**Deux serveurs pour un même type de fichier — établi, pas présumé** :
Kate n'autorise **qu'un seul serveur actif par couple (racine, langId)**
(`m_servers[root][langId]`, pas de cohabitation simultanée). Ce point,
que la demande signalait explicitement comme potentiellement
structurellement bloquant, s'avère **sans conséquence ici** : les
serveurs par défaut de Kate pour python/yaml (`pylsp`,
`yaml-language-server`) sont **tous deux absents comme binaires sur ce
poste** (`command -v pylsp yaml-language-server` → vide) — aucune
capacité de diagnostic/navigation réelle n'est donc déplacée par
l'ajout de `lsp-ai`. Si l'un des deux avait été présent, la question
serait devenue « complétion ou reste des fonctions LSP », arbitrage
opérateur comme demandé — non atteinte.

### 9.2 — Configuration : même binaire, même configuration qu'Helix

Déployée par `roles/completion/` (jamais un fichier Plasma copié —
même discipline qu'à BUR-1 sur `kwinrc`) :
`roles/completion/templates/kate-lspclient-settings.json.j2`, rendu à
partir des **mêmes variables** que
`roles/completion/templates/languages.toml.j2` (Helix) —
`completion_bin_dir`/`completion_bin_name`, `completion_ollama_model`,
`completion_ollama_generate_url`/`completion_ollama_chat_url`,
`completion_max_context`, `completion_num_predict` —
`completion_kate_languages` est littéralement
`"{{ completion_helix_languages }}"`, pas une seconde liste saisie
séparément. Vérifié par lecture directe du fichier déployé après
exécution du rôle : `command` = `/home/mahieumi/.local/bin/lsp-ai
--stdio`, `generate_endpoint`/`chat_endpoint` =
`http://127.0.0.1:11434/api/{generate,chat}`, `model` =
`qwen2.5-coder:7b-instruct-q4_K_M` — identique, terme à terme, à ce
que `hx --health` et le journal LSP de Helix rapportent (§ 7-8).
**Pas une troisième forme du défaut déjà rencontré deux fois**
(`ansible-lint`/ANS-1, `yamllint`/EDI-1) : une seule source de vérité
pour les deux éditeurs.

La garde d'existence de Helix ajoutée en COR-1
(`roles/completion/tasks/main.yml`) a son équivalent pour Kate :
`command -v kate`, échec bruyant si absent, avant toute écriture.

### 9.3 — Preuve : obstacle méthodologique, deux impasses, la voie qui a marché

**Obstacle nouveau, spécifique à Kate.** Contrairement à Helix (appli
terminal, pilotable par `tmux send-keys`), Kate est un client Wayland
natif — `tmux` ne contrôle que le pseudo-terminal du shell lançant
Kate, jamais sa fenêtre graphique. Deux pistes explorées et
documentées avant la solution retenue, aucune des deux gardée :

**Impasse 1 — D-Bus, action de complétion introuvable au niveau
fenêtre.** Kate expose un service D-Bus riche
(`org.kde.kate-<pid>`, interface `org.kde.KMainWindow`, méthode
`activateAction(s) -> b`). L'action de complétion manuelle,
`tools_invoke_code_completion`, sourcée dans le code amont de
KTextEditor (`src/view/kateview.cpp`, tag `v6.28.0`,
`ViewPrivate::setupActions()`), **appartient à la vue KTextEditor, pas
à la fenêtre principale** — `activateAction s
tools_invoke_code_completion` renvoie `false`, confirmé par appel
direct ; aucun objet D-Bus par document/vue n'est exposé pour la
contourner.

**Impasse 2 — deux outils d'injection de frappe, deux échecs
structurels distincts, chacun consigné et signalé à l'opérateur avant
d'aller plus loin** (aucun outil non prévu par la demande n'a été
installé sans validation) :
- `wtype` (protocole `zwp_virtual_keyboard_manager_v1`) : « Compositor
  does not support the virtual keyboard protocol » — confirmé
  indépendamment par `wayland-info`, le protocole n'est simplement pas
  annoncé par cette session KWin.
- `ydotool` (périphérique noyau via `/dev/uinput`, démon `ydotoold`) :
  le périphérique virtuel est bien créé et valide
  (`/proc/bus/input/devices`, bits `EV_KEY` complets, handler `kbd`)
  mais **sans tag `uaccess`/seat** (`udevadm info`) — KWin/libinput
  s'appuie sur ce tag pour n'ouvrir que les périphériques de la
  session graphique active ; sans lui, aucune frappe n'atteint aucune
  fenêtre. Corriger ça exige une règle `udev` système
  supplémentaire — jugé plus invasif que l'installation des deux
  paquets de test et hors périmètre de ce livrable, signalé à
  l'opérateur plutôt que fait unilatéralement.

**Voie retenue — pilotage manuel par l'opérateur, journal lu en
parallèle.** Kate déjà ouvert et focus (le focus lui-même a demandé un
petit script KWin éphémère, `workspace.activeWindow = <fenêtre Kate>`
via `org.kde.kwin.Scripting`, chargé et déchargé pour ce seul usage —
un geste de mise au premier plan, jamais une injection de frappe).
L'opérateur a cliqué en fin de ligne puis pressé Ctrl+Espace ; le
journal LSP (`journalctl --user _PID=<pid> -o cat`, équivalent Kate du
`~/.cache/helix/helix.log` d'Helix — la sortie Qt/KDE de cette session
n'est pas routée vers stdout/stderr hérité, confirmé par inspection de
`/proc/<pid>/fd`) a été lu **après coup**, jamais une capture visuelle.
**Nature exacte de la preuve, qualifiée en tête de § 9** : le clic et
le Ctrl+Espace sont une observation rapportée par l'opérateur, pas une
trace machine — la lecture du journal qui suit, elle, en est une.

### 9.4 — Résultats mesurés

**YAML** (`ansible.builtin.` en position de curseur), deux
invocations :
```
$ grep -n "textDocument/completion" kate-journal-yaml-completion.txt
2:calling textDocument/completion
26:calling textDocument/completion
```
```
YAML #1 : 22:33:04.529933 -> 22:33:04.999163 = 0,469 s
YAML #2 : 22:33:39.410982 -> 22:33:39.866200 = 0,455 s
```
Complétions réellement reçues (pas des accusés de réception vides) :
`ansible.builtin.apt: name: nginx state: present ... - name: Start and
enable nginx service ansible.builtin.service: name: nginx` —
syntaxiquement valide, cohérent avec le contexte Ansible du fichier,
même registre que les complétions Helix (§ 8.1).

**Python** (`if n <` dans une fonction Fibonacci), une invocation :
```
Python : 22:35:46.944678 -> 22:35:47.414707 = 0,470 s
```
Complétion reçue : `if n < 0: raise ValueError("Fibonacci is not
defined for negative numbers") elif n == 0: return 0 elif n` —
syntaxiquement et sémantiquement pertinente.

**Échec forcé, les deux sens démontrés.** `command` de
`~/.config/kate/lspclient/settings.json` pointé vers un binaire
inexistant (`lsp-ai-BROKEN-KAT1`), Kate **entièrement redémarré**
(quitté via D-Bus puis relancé — pas un simple rechargement de
document, pour forcer une relecture complète de la configuration).
Même geste (clic + Ctrl+Espace) répété deux fois (deux relances, pour
écarter un artefact d'une seule mesure) :
```
$ journalctl --user _PID=<pid> -o cat
language id  "yaml"
language id  "yaml"
language id  "yaml"
language id  "yaml"
```
**Aucune trace de `calling textDocument/completion` ni d'échange
JSON-RPC** — contrairement au cas nominal, le journal ne montre même
pas de tentative de connexion au serveur. Le popup de complétion
générique de Kate (mots déjà présents dans le document, mécanisme
propre à Kate, indépendant du LSP) reste actif — c'est lui que
l'opérateur a vu, sans item `ai`, distinction confirmée par le journal.
**Le journal diffère qualitativement, pas seulement par l'absence
d'un item** — la démonstration n'est pas vide de sens.

**Configuration restaurée, Kate redémarré une troisième fois, cas
nominal reconfirmé** (garde modifiée deux fois dans ce livrable —
la configuration cassée puis restaurée — perd sa démonstration
précédente à chaque modification, CLAUDE.md § Avant d'agir ; rejouée) :
```
Nominal (reconfirmation) : 23:10:49.255306 -> 23:10:49.716750 = 0,461 s
```

**Quatre latences mesurées, modèle déjà chargé** : 0,469 s / 0,455 s /
0,470 s / 0,461 s — resserrées autour de la référence Helix (0,46-0,48
s, § 8.3), sans le confond RTD3/rechargement (~3,7 s, D22/IA-4) à
écarter : le service tournait déjà, aucune inactivité prolongée entre
les mesures.

### 9.5 — Nettoyage et invariants confirmés

`wtype` et `ydotool` étaient des outils de test éphémères, jamais
référencés dans un rôle ni committés — désinstallés en fin de
livrable, état des paquets installés identique à l'instantané
d'avant-livrable (`diff` vide entre les deux relevés `rpm -qa`).
`ydotool.service` (unité système livrée par le paquet) arrêté avant
la désinstallation ; la désinstallation a supprimé l'unité avec le
paquet (`systemctl is-enabled` → `not-found` après coup).

**Tableau des actions privilégiées, tentative sans privilège :
résultat** :

| Action | Commande privilégiée | Tentative sans privilège : résultat |
|---|---|---|
| Installer `wtype` | `sudo dnf install -y wtype` | `dnf install -y wtype` → refusé : « requires superuser privileges » |
| Installer `ydotool` | `sudo dnf install -y ydotool` | **non tentée — lapsus reconnu, pas glissé sous silence** (règle CLAUDE.md § Avant d'agir violée une fois dans ce livrable) |
| Démarrer `ydotool.service` | `sudo systemctl start ydotool.service` | `systemctl start ydotool.service` → échec : « Connection timed out » |
| Frapper via `ydotool` (type/key) | `sudo YDOTOOL_SOCKET=/tmp/.ydotool_socket ydotool ...` | **non tentée — lapsus reconnu** (le socket racine `0700` aurait refusé une connexion non privilégiée, mais la démonstration explicite n'a pas été faite) |
| Arrêter `ydotool.service` | `sudo systemctl stop ydotool.service` | **non tentée — lapsus reconnu** |
| Désinstaller `wtype`/`ydotool` | `sudo dnf remove -y wtype ydotool` | `dnf remove -y wtype ydotool` → refusé : « requires superuser privileges » |

**Confirmations finales** : `command -v node npm` toujours vide ;
aucun modèle supplémentaire téléchargé ; `/etc/sudoers.d/99-wheel-nopasswd`
inchangé (`visudo -c` : parsed OK) ; `gpu_mux_mode` `current_value`
inchangé (jamais écrit par ce livrable) ; `/etc/yum.repos.d/terra.repo`
présent, jamais touché ; `/etc/cdi/nvidia.yaml` jamais réécrit (aucune
tâche de ce livrable n'y touche) ; `uptime -s` antérieur à ce
livrable — aucun redémarrage.

**`ansible-lint --profile production`** sur `roles/editor/` et
`roles/completion/` : 0 défaut (un dépassement de longueur de ligne
trouvé et corrigé dans `roles/completion/tasks/main.yml` avant la
validation finale). `--check` sur les deux rôles : `changed=0`.
Exécution réelle des deux rôles, deux fois de suite : `changed=0` aux
deux passages (l'état était déjà convergent — la configuration
restaurée après l'échec forcé correspondait déjà exactement à ce que
le rôle produit).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : le marqueur
Kate (§ 2.1) est fermé par ce livrable — résolu, pas retiré par
nettoyage. Le second marqueur de ce document (mécanisme de prédiction
d'édition natif de Zed, § 3) reste ouvert, hors périmètre de ce
livrable.

## Voir aussi

- [`docs/local-ai.md`](local-ai.md) § 8.6 — la résolution d'IA-2,
  requalifiée par ce document sans être effacée.
- [`docs/machine-facts.md`](machine-facts.md) — D12/D13 (npm fermé,
  choix d'éditeur), D15 (modèle de complétion résident, requalifiée
  par D20), D19/D20 (ce livrable).
- [`docs/editor.md`](editor.md) — résolution complète D12/D13, motifs
  d'exclusion de Zed/Cursor détaillés § 1.2/1.3/2.
- [`docs/repositories.md`](repositories.md) § 6 (registre de
  conteneurs Ollama) et § 7 (`crates.io`, ce livrable) — ancrages de
  confiance des surfaces d'approvisionnement de ce dépôt.
- [`roles/completion/`](../roles/completion/) — rôle exécuté par ce
  livrable (§ 9, KAT-1, pour la partie Kate), README pour l'usage
  courant.
- [`roles/editor/`](../roles/editor/) — greffon LSP client de Kate
  activé (KAT-1, § 9.1-9.2).
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici, en
  particulier la garde sur les découvertes qui contredisent un fait
  déjà documenté, et la règle 0.3 (élévation précédée d'une tentative
  sans privilège), ajoutée par ce livrable.
