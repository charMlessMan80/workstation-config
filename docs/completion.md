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
**[RÉALIGNÉE le 2026-08-12, BSH-2]** — D15 est revenue à la résidence
permanente, sur la base de deux faits établis depuis D20 : la
complétion fonctionne réellement dans deux éditeurs sur trois
langages (ce document), et une complétion à froid coûte 22,6 s
mesurées (§ 10.3 ci-dessous), pas les ~3,7 s d'une bascule à chaud entre
deux modèles déjà chargés — détail complet, mesure isolée du coût
réel : `docs/local-ai.md` § 11. Sans effet sur ce rôle
(`roles/completion/`), qui n'a jamais chargé ni choisi de modèle dans
un cas comme dans l'autre.
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

**Aucun des six trouvés n'est empaqueté dans les dépôts activés (`docs/repositories.md` § 4)** :
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

## 10. BSH-1 — bash/shell ajoutés à Kate et Helix (2026-08-12)

**Diagnostic établi, pas supposé.** L'opérateur avait ouvert un script
bash dans Kate sans voir aucune proposition du modèle : `pgrep -a
lsp-ai` ne montrait aucun processus, et aucune trace au journal.
`~/.config/kate/lspclient/settings.json` (KAT-1) ne déclarait que
`^Python$` et `^YAML$` — aucune expression ne correspond à un script
shell. Le popup observé était la complétion interne de Kate par mots
du document (mécanisme sans serveur de langage) — même mode d'échec
silencieux que celui déjà nommé pour Kate en général (§ 9), sous une
forme nouvelle : ici, ce n'est pas une configuration cassée, c'est une
configuration qui n'a jamais existé pour ce type de fichier.

### 10.1 — Résolution : le nom du mode n'est pas devinable

**Kate.** Le greffon client LSP fait correspondre `highlightingModeRegex`
au **nom du mode de coloration syntaxique** (`doc->highlightingMode()`,
sourcé dans `lspclientservermanager.cpp`, tag v25.04.3, fonction
`_languageId`), pas à l'extension de fichier. Établi par mesure sur ce
poste, pas deviné : `ksyntaxhighlighter6 --list` (paquet
`kf6-syntax-highlighting-6.28.1`, celui que Kate lie réellement) liste
deux définitions distinctes couvrant ce domaine — **« Bash »** et
**« Zsh »**. Comparaison de rendu forcé (`-s Bash` / `-s Zsh`) contre
l'auto-détection (diff des sorties HTML sur un même contenu) confirme :
un fichier `.sh` (quel que soit son shebang, `#!/bin/sh` comme
`#!/bin/bash`) résout vers « Bash » ; un fichier `.bash` résout aussi
vers « Bash » ; seul `.zsh` résout vers « Zsh ». L'opérateur écrit
« bash et shell », jamais zsh : **une seule entrée ajoutée**, pas deux,
pour ne pas multiplier les entrées inutiles.

Confirmation indépendante, deuxième source concordante : le fichier
`settings.json` livré avec Kate lui-même (`addons/lspclient/
settings.json`, tag v26.04.3, celui installé sur ce poste — même
source qu'en KAT-1 § 9.1) porte déjà une entrée `servers.bash` avec
`"highlightingModeRegex": "^Bash$"` et pointe `bash-language-server`
(absent de ce poste, D12 l'interdirait de toute façon). Même regex,
même clé `bash`, retenus ici par coïncidence de méthode — vérifiés
indépendamment, pas copiés sans contrôle.

**Helix.** Mécanisme différent — identifiant de langage déclaré dans
`languages.toml`, pas une expression sur un nom de mode. `hx --health` (25.07.1, version installée) liste `bash`
comme langage connu (tree-sitter/highlight/textobject/indent tous ✓
sur ce poste) — un seul identifiant, Helix ne distingue pas bash/sh/zsh
en langages séparés comme le fait Kate. La lacune y était **exactement
la même** qu'ailleurs : `hx --health` rapportait déjà
`bash-language-server` non trouvé (comportement par défaut de Helix,
serveur npm que D12 exclut), et aucun serveur `lsp-ai` n'y était
rattaché avant ce livrable — confirmé, § 10.3 (échec forcé Helix)
ci-dessous.

### 10.2 — Configuration : une seule source, factorisation déjà en place

Les deux gabarits (`templates/languages.toml.j2`,
`templates/kate-lspclient-settings.json.j2`) bouclent déjà sur
`completion_helix_languages`/`completion_kate_languages` — la
duplication du bloc de configuration du modèle dans le fichier Kate
rendu (un bloc complet par langage, JSON ne permettant pas de
référencer un même sous-objet depuis plusieurs clés) est **déjà
générée depuis une unique définition Jinja** (`lsp_ai_options`) dans le
gabarit, jamais recopiée à la main. Ajouter `bash` n'a exigé **aucune
modification des deux gabarits** — seulement `roles/completion/
defaults/main.yml` :
- `completion_helix_languages` : `bash` ajouté (troisième élément,
  après `yaml`, `python`) — `completion_kate_languages` en hérite
  directement (variable réutilisée, jamais une seconde valeur saisie).
- `completion_kate_highlighting_mode_regex` : `bash: "^Bash$"` ajouté.

Kate et Helix continuent de pointer le même binaire
(`{{ completion_bin_dir }}/{{ completion_bin_name }}`), le même modèle
(`{{ completion_ollama_model }}`) et la même API locale
(`{{ completion_ollama_generate_url }}`/`{{ completion_ollama_chat_url }}`)
— aucune valeur saisie séparément pour `bash`, mêmes variables
partagées qu'pour `yaml`/`python`.

### 10.3 — Preuve : Helix, entièrement automatisée

Méthode identique à § 7.4/§ 8 (`hx -vv`, session `tmux` détachée,
lecture du journal `~/.cache/helix/helix.log` après coup, aucune
capture visuelle).

**Nominal**, fichier `.sh` enregistré (`/tmp/bsh-demo/backup.sh`,
`#!/bin/bash`) :
```
$ grep -n '"languageId":"bash"' helix-full.log
978:...{"textDocument":{"languageId":"bash",...,"uri":"file:///tmp/bsh-demo/backup.sh","version":0}}}
$ grep -n 'textDocument/completion\|"id":1,"result"' helix-full.log
988:  -> textDocument/completion ... "id":1  (requête, 02:49:59.367)
1114: <- {"id":1,"result":{...,"items":[{"label":"ai - \n# Author: John Doe\n# Date: 2023-10-05\n# Description: A Bash script to check if a",...}]}}  (réponse, 02:50:21.985)
```
**Latence 22,6 s, pas une absence de réponse** — distinguée
explicitement (rappel de cadrage de la demande) : le service tournait
mais le modèle n'était pas résident au moment de cette requête
(rechargement séquentiel, D22) ; contenu réellement reçu, cohérent
avec un script bash (pas un accusé de réception vide), confirmant que
la réponse a bel et bien abouti, lentement.

**Échec forcé.** Configuration Helix redéployée sans `bash`
(`-e completion_helix_languages='["yaml","python"]'`), même fichier
rouvert :
```
$ grep -n 'lsp-ai\|bash-language-server' helix-forced-failure.log
969: Language server not found for `source.bash` bash-language-server
     command 'bash-language-server' not found: cannot find binary path
```
`pgrep -a lsp-ai` : aucun processus. Sans notre entrée, Helix retombe
sur son serveur par défaut pour `bash` (`bash-language-server`, npm —
exactement ce que D12 exclut), introuvable sur ce poste : **aucune
tentative de connexion à `lsp-ai`**, contrairement au cas nominal. Le
résultat diffère qualitativement entre les deux configurations — la
démonstration n'est pas vide de sens.

**Non-régression, YAML et Python, même session** — requêtes et
réponses réelles obtenues sur les deux, avant et après le test bash
(fichiers `check.yml`/`check.py`, journal unique `helix-full.log`) :
```
YAML : "->textDocument/completion" 02:50:56.792 -> "<-...result" 02:50:57.255 = 0,463 s
       item reçu : "--- sync..." (contenu réel)
Python : "->textDocument/completion" 02:51:38.339 -> "<-...result" 02:51:42.937 = 4,598 s
       item reçu : "# Base case for n = 0\n    return 0..." (contenu réel)
```
Configuration réelle (avec `bash`) restaurée, rejouée : `changed=0` à
la seconde exécution du rôle (`ansible-playbook
roles/completion/completion.yml` deux fois de suite après
restauration).

### 10.4 — Preuve : Kate, même méthode mixte que KAT-1 (§ 9)

**Nature de la preuve, qualifiée comme en § 9** : la lecture du journal
est une preuve machine, rejouable ; le déclenchement (clic + Ctrl+Espace)
relève de la classe « observation rapportée par l'opérateur »
(`CLAUDE.md` § Sourcing des faits) — marquée ici, datée (2026-08-12),
attribuée (l'opérateur, à la demande explicite de cette session).

**Obstacle méthodologique nouveau, résolu** : la catégorie de
journalisation du greffon LSP (`katelspclientplugin`, déclarée via
`ecm_qt_declare_logging_category`) est **désactivée par défaut** —
confirmé en lisant `lspclientplugin.cpp` (tag v25.04.3) : `qCInfo`/
`qCDebug` de cette catégorie sont explicitement coupés sauf si la
variable d'environnement **`LSPCLIENT_DEBUG=1`** est positionnée (« pour
ne pas spammer l'utilisateur en debug output par défaut » — commentaire
du code amont). `QT_LOGGING_RULES` seul (essayé d'abord, deux variantes,
sans effet) ne suffit pas : ce n'est pas un mécanisme de catégorie Qt
générique, c'est un indicateur propre au greffon. Nommé ici pour la
prochaine fois qui en aurait besoin.

**Nominal**, Kate relancé avec `LSPCLIENT_DEBUG=1`, fichier `.sh`
enregistré, clic + Ctrl+Espace :
```
$ grep -n 'language id\|calling textDocument/completion\|adding completions' kate-journal-bash-nominal.log
1:language id  "bash"
199:calling textDocument/completion
220:adding completions  1
```
Réponse JSON-RPC réelle relue dans le même journal :
```
{"jsonrpc":"2.0","id":2,"result":{"isIncomplete":false,"items":[{"filterText":"if [",
"label":"ai -  -z \"$SRC\" ] || [ -z \"$DEST\" ]; then\n    echo \"Usage: ...",...}]}}
```
`language id "bash"` confirme que Kate résout bien `^Bash$` vers la
même clé `bash` que Helix, sans les distinguer par accident.

**Distinguer la complétion du modèle de celle par mots du document —
la question posée par la demande.** Aucun moyen visuel fiable : les
deux popups se ressemblent, c'est précisément ce qui a trompé
l'opérateur au départ. **Seul le journal LSP distingue les deux, de
façon binaire** : la ligne `calling textDocument/completion` suivie
d'un `adding completions` avec un item `"label":"ai - ..."` atteste la
voie modèle ; leur absence totale (voir échec forcé ci-dessous)
n'exclut pas que Kate affiche malgré tout un popup — celui, alors,
n'est que la complétion interne par mots.

**Échec forcé.** Configuration Kate redéployée sans `bash`
(même substitution `-e` que pour Helix — `completion_kate_languages`
hérite de `completion_helix_languages`), Kate **entièrement redémarré**
(processus tué, relancé — pas un simple rechargement de document, même
exigence qu'en KAT-1), même geste répété :
```
$ cat kate-journal-bash-forced-failure.log
language id  "bash"
language id  "bash"
language id  "bash"
language id  "bash"
```
**Quatre lignes au total, rien d'autre** — ni `calling initialize`, ni
`calling textDocument/completion`, ni aucune tentative de connexion à
`lsp-ai` (`pgrep -a lsp-ai` : aucun processus). Kate reconnaît toujours
le mode « Bash » (`language id "bash"`, sa propre entrée par défaut
existe toujours, pointant `bash-language-server`), mais ne tente même
pas de démarrer ce serveur par défaut — cohérent avec
`AllowedServerCommandLines` (`~/.config/katerc` § `[lspclient]`,
`roles/editor/`), qui n'autorise que la commande `lsp-ai --stdio`
explicitement déployée par ce rôle. **Le résultat diffère
qualitativement du cas nominal (4 lignes contre plus de 200,
aucun échange JSON-RPC contre un échange complet)** — même verdict
qu'en § 9.4 : la démonstration n'est pas vide de sens.

**Non-régression, YAML et Python, Kate relancé à chaque fois (même
processus de débogage), configuration réelle restaurée d'abord** :
```
$ grep -n 'language id\|calling textDocument/completion\|adding completions' kate-journal-yaml-regression.log
1:language id  "yaml"
202:calling textDocument/completion
223:adding completions  1

$ grep -n 'language id\|calling textDocument/completion\|result' kate-journal-python-regression.log
2:language id  "python"
203:calling textDocument/completion
233:{"jsonrpc":"2.0","id":2,"result":{...,"items":[{...,"label":"ai - 2:\n        return n\n    else:\n        return fib(n-1) + fib(n-2)...",...}]}}
```
Kate relancé une dernière fois sans variable de débogage
(fonctionnement normal restauré, pas de journalisation superflue en
usage courant).

### 10.5 — Fait de méthode consigné

**Deux causes distinctes, cumulables, aucune ne produit d'erreur
visible :**
1. **Aucun serveur déclaré pour ce type de fichier** (le cas résolu par
   ce livrable) — Kate et Helix retombent chacun sur leur propre
   comportement par défaut, qui pointe un serveur npm absent de ce
   poste (D12) ; l'échec est silencieux, seul le journal LSP le montre
   (§ 10.3/10.4 ci-dessus).
2. **Un fichier neuf, jamais enregistré, n'a pas de mode déterminé** —
   ni Kate ni Helix ne peuvent résoudre un `highlightingMode`/langId
   sans extension ni contenu stable, donc aucun serveur ne démarre,
   **y compris pour des langages par ailleurs couverts** (yaml,
   python). Un fichier `.sh` fraîchement créé mais non sauvegardé
   n'aurait montré aucune complétion réelle même après ce livrable —
   pas un défaut de configuration, une limite structurelle de la
   détection de mode elle-même.

Dans les deux cas, Kate affiche sa propre complétion par mots du
document — un **repli silencieux visuellement indiscernable du
nominal**, la forme la plus coûteuse d'échec silencieux : elle ne
déclenche même pas de soupçon, contrairement à un message d'erreur qui
inviterait à chercher plus loin.

### 10.6 — Énumération des branches, exercées et non exercées

| Conditionnel | Branche exercée | Branche non exercée |
|---|---|---|
| `completion_forbidden_lsp_servers` (garde 1/4) | Aucun serveur npm présent → garde passe | Échec forcé non rejoué dans ce livrable (déjà démontré en CMP-1/KAT-1, logique de la garde inchangée ici) |
| `completion_lsp_ai_which.rc` (garde 1/4bis) | lsp-ai résout au chemin attendu | — (inchangé, non touché par ce livrable) |
| `completion_toolchain_precheck.rc` (tentative sans privilège Rust) | `rc == 0` (déjà présent, pas d'élévation) | `rc != 0` (installation `dnf`) non exercée dans ce livrable — chaîne Rust déjà installée depuis CMP-1 |
| `completion_helix_binary_name`/`completion_kate_binary_name` (gardes 4/4bis) | Helix et Kate trouvés | Absence non rejouée ici (logique inchangée) |
| Boucle `for lang in completion_kate_languages`/`completion_helix_languages` | **Trois langages** (`yaml`, `python`, `bash`) — la branche `bash` est neuve, exercée pour la première fois | — (aucune, la boucle couvre tous les éléments par construction) |
| Assertion de couverture exacte des langages Kate (`servers.keys() == completion_kate_languages`) | Passe avec les trois langages | Échec (langages manquants ou en trop) non exercé dans ce livrable |
| Fallback Kate/Helix vers le serveur par défaut (`bash-language-server`, absent) | **Exercée pour la première fois par ce livrable** (§ 10.3/10.4, échec forcé) — jamais rejouée avant, y compris lors de KAT-1/CMP-1 qui ne couvraient que yaml/python | — |
| Mode « Zsh » (`.zsh`) | **Non exercée, délibérément** — hors périmètre de la demande (bash/shell nommés, pas zsh) | reste `@VERIF` si un jour requis : aucune entrée `zsh` déployée par ce rôle |

### 10.7 — Tableau des actions privilégiées

**Aucune élévation dans ce livrable.** Toutes les commandes exécutées
(`ksyntaxhighlighter6`, `hx --health`, `ansible-playbook
roles/completion/completion.yml`, lecture de journaux `journalctl
--user`, lancement de Kate/Helix en domaine utilisateur) relèvent du
domaine utilisateur — aucune n'exige `sudo`. Non applicable au sens de
la règle (aucune action structurellement privilégiée à énumérer).

### 10.8 — Confirmations finales

Aucun paquet installé (`ksyntaxhighlighter6` faisait déjà partie de
`kf6-syntax-highlighting`, dépendance déjà présente de Kate ; aucune
commande `dnf install` de ce livrable). Aucun serveur de langage npm
ajouté — `bash-language-server` reste absent, confirmé par le
comportement même de l'échec forcé (§ 10.3/10.4). `sudoers`,
`terra.repo`, `/etc/cdi/`, `gpu_mux_mode`, `kwinrulesrc`, `site.yml`
intacts (aucune tâche de ce livrable n'y touche). Aucun redémarrage de
la machine — seul Kate (application utilisateur) a été redémarré à
plusieurs reprises, jamais la session ni le système.

`ansible-lint --profile production roles/completion roles/editor` :
0 défaut. `ansible-playbook --check roles/completion/completion.yml` :
`changed=0`. Exécution réelle : `changed=2` (première écriture avec
`bash`), puis `changed=0` à la seconde exécution — idempotence
confirmée.

## 11. TRO-1 — phase A, résolution en lecture seule des paramètres de génération (2026-08-12)

**Constat de l'opérateur** : les complétions dans Kate sont tronquées
— réponses coupées en cours de génération. **Cause établie** :
`completion_num_predict: 32` (`roles/completion/defaults/main.yml:232`)
plafonne la génération à 32 jetons, environ une ligne courte, propagé
aux deux éditeurs et présent six fois dans le fichier Kate déployé
(deux blocs par serveur, trois langages). Cette section est
**purement en lecture** — aucune valeur n'est modifiée ici ; la
correction relève de la phase B, après accord explicite de
l'opérateur.

### 11.1 — A.1 : ce que `lsp-ai` permet réellement (lecture du code à
l'empreinte compilée `1e910a8cf0048406eb227bf2064743010a9ff3a9`,
`completion_lsp_ai_commit`)

Code source cloné et vérifié à cette empreinte exacte (pas la branche
courante amont).

- **`num_predict`** (`crates/lsp-ai/src/transformer_backends/ollama.rs`) :
  transmis tel quel dans `options` du corps JSON envoyé à
  `/api/generate` — `lsp-ai` ne l'interprète ni ne le borne lui-même,
  c'est un passe-plat vers l'API Ollama, qui applique sa propre
  sémantique (nombre maximal de jetons **générés** par cette requête,
  pas une longueur de fichier ni de contexte).
- **`max_context`** (`crates/lsp-ai/src/memory_backends/file_store.rs:228-262`,
  `crates/lsp-ai/src/memory_backends/mod.rs:22-28`) : **ce n'est pas un
  paramètre transmis à Ollama** — c'est un paramètre interne au
  *memory backend* de `lsp-ai` (`file_store`), qui borne la portion du
  fichier ouvert que `lsp-ai` extrait autour du curseur avant de
  construire le prompt. Converti en caractères par
  `tokens_to_estimated_characters()` (`crates/lsp-ai/src/utils.rs:34-36`) :
  **`tokens * 4`**, une estimation grossière et fixe, pas un compte de
  jetons réel du tokenizer du modèle. `max_context: 2000` signifie donc
  concrètement environ **8000 caractères** de fichier extraits autour
  du curseur (`file_store.rs:233-234` : `max_length / 2` de chaque
  côté), pas 2000 jetons envoyés au modèle.
- **Distinction automatique/explicite (question décisive, établie, pas
  supposée)** : **aucune distinction n'existe côté `lsp-ai`.** Une
  seule structure de configuration `Completion`
  (`crates/lsp-ai/src/config.rs:339-348`) sert la requête LSP standard
  `textDocument/completion` (`crates/lsp-ai/src/main.rs:207-213`,
  type `Completion` de `lsp-types`) — qu'elle soit déclenchée au fil de
  la frappe par l'éditeur ou par une combinaison de touches explicite
  ne change rien côté serveur, qui reçoit le même type de requête dans
  les deux cas et applique toujours les mêmes `completion.parameters`.
  `lsp-ai` distingue en revanche `Completion` de deux autres types de
  requêtes non liées à cette question : `Generation` et
  `GenerationStream` (custom, non utilisées par la configuration
  actuelle, `GenerationStream` explicitement non implémentée —
  `anyhow::bail!("GenerationStream is not yet implemented")`,
  `ollama.rs:227`). **Conséquence pour la phase B** : il n'existe pas
  de paramètre séparé « frappe courante » / « déclenchement explicite »
  à arbitrer côté `lsp-ai` — une seule paire de valeurs
  `num_predict`/`max_context` s'applique à toute complétion, quel que
  soit le déclencheur.
- **Autres paramètres pertinents non exposés par la configuration
  actuelle** : `fim` (`crates/lsp-ai/src/config.rs:172` et suivants,
  structure `FIM` avec `start`/`middle`/`end`) — **non configuré** dans
  `roles/completion/defaults/main.yml` (aucune occurrence de `fim`,
  vérifié par recherche). Sans lui, `do_chat_completion`
  (`ollama.rs`, branche `Prompt::FIM` sans `params.fim`) échouerait —
  or la complétion fonctionne (§ 9-10), ce qui signifie que le chemin
  réellement emprunté est `Prompt::ContextAndCode` sans `messages`
  (branche `get_completion` directe, prompt brut construit par
  `format_prompt`), pas un FIM structuré avec balises dédiées. Aucune
  séquence d'arrêt (`stop`) n'apparaît dans `config.rs` ni dans
  `OllamaRunParams` — non supporté par cette version, donc rien à
  arbitrer de ce côté. `keep_alive` existe et est déjà couvert par D15
  (résidence, `roles/local_ai/`), hors périmètre de ce livrable.

### 11.2 — A.2 : origine de la valeur 32

**Non mesurée, non arbitrée — reprise verbatim d'un exemple amont.**
`git log --all -S"num_predict" -- roles/completion/` ne montre qu'un
seul commit introduisant cette valeur : `d96d307` (« lsp-ai compilé
depuis les sources, intégré à Helix », 2026-08-07), avec le
commentaire déjà présent à l'époque : « valeurs de départ documentées
par le projet amont (`docs/completion.md` § 2, exemple de
configuration Ollama cité), pas inventées ». Confirmé en source :
`crates/lsp-ai/src/config.rs:548-556` (test `ollama_config`) porte
exactement `"max_context": 1024` et `"num_predict": 32` dans son
propre exemple de configuration Ollama — la valeur du dépôt (`2000`
pour `max_context`, `32` pour `num_predict`) a bien été copiée de là,
avec un seul chiffre changé (`max_context` porté à 2000). Aucune
mesure de latence ni de troncature n'accompagne ce commit ni aucun
commit ultérieur touchant ces deux variables (`git log --all
-S"max_context"` : même commit unique). **Ce fait explique
directement pourquoi personne ne l'a remise en question** : elle
portait l'apparence d'une valeur documentée et raisonnable, alors
qu'elle n'a jamais été mesurée contre un cas d'usage réel.

### 11.3 — A.3 : coût de la génération, mesuré directement contre l'API locale

Méthode d'isolement de `docs/dgpu-power.md` : appels HTTP directs à
`http://127.0.0.1:11434/api/generate` (`curl`/`urllib` Python), jamais
via `lsp-ai` ni un éditeur — isole la génération du reste de la chaîne
(analyse LSP, rendu éditeur). Modèle `qwen2.5-coder:7b-instruct-q4_K_M`
déjà résident (`ollama ps`/`curl .../api/ps` : `expires_at` en 2318,
D15). Un appel de mise en chauffe (`num_predict=8`, 0,230 s) précède
chaque série pour écarter l'effet de sonde de la toute première
requête après une pause du script lui-même (constaté : 0,209-0,230 s
en première position, cohérent avec le régime établi, pas un
rechargement — `size_vram` de `/api/ps` inchangé avant/après). Prompt
identique pour toutes les mesures (fonction Python avec docstring,
~90 caractères). `eval_count`/`eval_duration` proviennent de la
réponse JSON d'Ollama elle-même (jetons réellement générés, durée de
génération pure, hors chargement).

| `num_predict` demandé | `eval_count` (jetons réellement générés) | `eval_duration` | latence murale mesurée |
|---|---|---|---|
| 32 | 32 (plafond atteint) | 0,292 s | 0,424-0,439 s |
| 64 | 64 (plafond atteint) | 0,588-0,592 s | 0,726-0,730 s |
| 128 | 41-106 (arrêt naturel avant le plafond, variable selon l'échantillon) | 0,373-0,984 s | 0,508-1,122 s |
| 256 | 161-248 (arrêt naturel avant le plafond, variable) | 1,495-2,321 s | 1,636-2,473 s |

**Lecture** : à 32 (valeur actuelle), le plafond est systématiquement
atteint — c'est la preuve directe de la troncature, la génération
n'atteint jamais de fin naturelle. À 64, toujours systématiquement
plafonné. À partir de 128, le modèle s'arrête naturellement avant le
plafond dans la plupart des échantillons (fin de fonction, point
d'arrêt logique) — la variabilité de `eval_count` reflète le contenu
généré, pas une instabilité de mesure. **Relation approximativement
linéaire** entre jetons générés et durée : ≈9,1 ms/jeton
(0,292 s / 32), cohérent à 64 (0,588 s / 64 ≈ 9,2 ms/jeton) et au-delà.
Référence de la demande (0,46-0,48 s) mesurée par l'éditeur (Helix,
§ 7) contre 0,424-0,439 s mesuré ici en direct : l'écart (~30-50 ms)
est cohérent avec le coût ajouté par la chaîne LSP (analyse du
fichier, sérialisation), pas une divergence de méthode.

### 11.4 — A.4 : `max_context`, ce que 2000 représente et coût VRAM d'un élargissement

**`max_context` n'est pas un paramètre Ollama** (§ 11.1) — il ne pèse
donc **rien** sur la VRAM au sens propre du terme : c'est une fenêtre
d'extraction de texte côté `lsp-ai`, mesurée en caractères estimés
(`tokens * 4`, fixe, `utils.rs:34-36`), pas en jetons du tokenizer
réel du modèle. `2000` ⇒ ≈8000 caractères extraits autour du curseur
(`file_store.rs:233-234`, moitié de chaque côté) — pour un fichier
Python ou YAML de taille courante de ce dépôt (quelques centaines de
lignes), cela couvre largement plus qu'un écran, mais peut tronquer un
fichier de plusieurs milliers de lignes.

**Le paramètre qui pèse réellement sur la VRAM est `num_ctx`
(la fenêtre de contexte du modèle Ollama lui-même)**, distinct de
`max_context` de `lsp-ai` et **non exposé du tout** par la
configuration actuelle (aucune occurrence de `num_ctx` dans
`roles/completion/`) — le modèle tourne donc avec le **défaut Ollama
de 4096 jetons**, confirmé par mesure directe (`curl .../api/ps` après
un appel sans `num_ctx` explicite : `"context_length": 4096`, pas les
32 K annoncés par la fiche du modèle — ce chiffre est un plafond
supporté, pas la valeur active). **Coût VRAM mesuré par palier**
(`nvidia-smi --query-gpu=memory.used`, un appel de mise en chauffe par
palier, `curl .../api/ps` pour `size_vram`/`context_length` après
coup) :

| `num_ctx` | VRAM mesurée (`nvidia-smi`) | `size_vram` rapporté par Ollama |
|---|---|---|
| 2048 | 4743 MiB | 4 628 519 320 (≈4,31 GiB) |
| 4096 (défaut actif, confirmé) | 4857 MiB | 4 748 056 984 (≈4,42 GiB) |
| 8192 | 5225 MiB | 5 133 943 438 (≈4,78 GiB) |
| 16384 | 5689 MiB | 5 620 482 702 (≈5,23 GiB) |
| 32768 (plafond annoncé du modèle) | 6617 MiB | 6 593 561 230 (≈6,14 GiB) |

**Lecture** : le cache d'attention croît avec `num_ctx`, confirmé —
d'environ 4,3 GiB à 2048 jetons jusqu'à environ 6,1 GiB au plafond
annoncé de 32 K, soit **+1,9 GiB environ** pour l'écart le plus large
mesuré. Sous l'enveloppe mesurée de ≈15,9 GiB (`docs/local-ai.md`
§ 2.3), largement supportable en isolation, mais partagé avec le
bureau et d'éventuels autres modèles chargés séquentiellement (D21) —
un `num_ctx` élargi reste un arbitrage à trancher par l'opérateur,
pas une évidence sans coût.

### 11.5 — Recommandation, sans application

- **`completion_num_predict`** : la valeur de 32 est la cause directe
  et mesurée de la troncature — elle plafonne systématiquement avant
  toute fin naturelle de génération dans les deux paliers testés qui
  correspondent à l'usage réel (courtes complétions). Une valeur entre
  128 et 256 couvrirait la plupart des complétions de ligne/bloc courts
  sans dépasser ~1-2,5 s de latence mesurée — au-delà du seuil «
  utilisable au fil de la frappe » évoqué par la demande (la référence
  de 0,46-0,48 s), mais cohérent avec un usage de complétion
  déclenchée plutôt qu'au fil de chaque caractère (aucune distinction
  possible côté `lsp-ai`, § 11.1 — le même paramètre s'applique aux
  deux). **Question à trancher par l'opérateur** : accepter une
  latence plus élevée (jusqu'à ~2,5 s à 256) en échange de complétions
  rarement tronquées, ou choisir une valeur intermédiaire (64,
  ~0,73 s) qui réduit mais n'élimine pas la troncature sur les
  complétions plus longues qu'une ligne.
- **`completion_max_context`** : ne pèse pas sur la VRAM (ce n'est pas
  un paramètre Ollama) — augmenter cette valeur est presque gratuit en
  coût (juste plus de texte extrait et envoyé en prompt, coût CPU/JSON
  marginal) et améliore la pertinence de ce que le modèle voit du
  fichier. **`num_ctx`, en revanche, n'est actuellement pas configuré
  du tout** — le modèle tourne au défaut Ollama (4096 jetons), jamais
  arbitré ni documenté jusqu'ici. **Question distincte à trancher par
  l'opérateur, hors du périmètre initial du signalement (troncature de
  sortie) mais découverte par cette lecture** : faut-il exposer et
  fixer `num_ctx` explicitement (coût mesuré ci-dessus, +1,9 GiB au
  plafond), ou laisser le défaut Ollama, qui n'a jamais causé de
  symptôme constaté ?

### 11.6 — Confirmations finales (phase A)

**Aucune valeur modifiée** — `roles/completion/defaults/main.yml`,
`roles/completion/README.md` et tout fichier déployé sont restés
inchangés par cette phase (vérifié : `git status --short
roles/completion/` vide avant commit de cette section, seul
`docs/completion.md` et `docs/machine-facts.md` diffèrent). Aucun
fichier déployé (Kate, Helix) touché. Aucun paquet installé, aucun
modèle téléchargé — le clone de lecture de `lsp-ai` à l'empreinte
compilée a eu lieu hors du dépôt (`/tmp`), jamais commis, jamais
compilé.

**Actions privilégiées** : non applicable — aucune élévation, aucune
commande `sudo` dans cette phase (lecture de code source public,
appels HTTP locaux vers un service déjà démarré, lecture `nvidia-smi`
sans privilège).

**Branches exercées/non exercées** : sans objet — aucun code Ansible
exécuté ni modifié dans cette phase, uniquement lecture de code source
tiers et mesures directes contre l'API.

## 12. TRO-1 — phase B, application des paramètres mesurés (2026-08-12)

**Décisions de l'opérateur** (après accord explicite sur la phase A) :
`completion_num_predict: 256` (marge maximale, contrepartie assumée) ;
`completion_num_ctx` exposé en variable, valeur cible à trancher par
mesure (§ 12.2).

### 12.1 — `num_predict: 256`, motif et mesure

Consigné en commentaire dans `roles/completion/defaults/main.yml` :
≈9,1 ms/jeton mesuré (§ 11.3), plafonnement systématique à 32 et 64
(jamais d'arrêt naturel dans l'échantillon mesuré), arrêt naturel dans
la plupart des cas à 128 et 256. **La garantie de réponse rapide au
fil de la frappe est perdue au profit de la complétude** — assumé
explicitement par l'opérateur, pas une conséquence non déclarée.

### 12.2 — `num_ctx` : paliers mesurés avant de figer

**Emplacement exact vérifié par lecture, pas déduit** : `num_ctx` se
transmet dans les `options` de la requête Ollama, exactement au même
niveau que `num_predict`
(`crates/lsp-ai/src/transformer_backends/ollama.rs`, `options:
params.options` — les deux sont des clés arbitraires du même objet
`HashMap<String, Value>` passées telles quelles). Posé donc au même
niveau dans les deux gabarits (`languages.toml.j2`,
`kate-lspclient-settings.json.j2`) — vérifié un seul point d'écriture
par gabarit, jamais recopié.

**VRAM et latence de traitement du prompt, par palier**, mesurées
directement contre l'API (isolement de `docs/dgpu-power.md`, un appel
de mise en chauffe par palier pour écarter l'effet de rechargement du
modèle) :

| `num_ctx` | VRAM mesurée (`nvidia-smi`) | `size_vram` Ollama | `prompt_eval_duration` (13 jetons de prompt) | Rechargement si changé (une fois) |
|---|---|---|---|---|
| 4096 (retenu) | 4857 MiB | 4 748 056 984 (≈4,42 GiB) | 0,0095 s | — |
| 8192 | 5225 MiB | 5 133 943 438 (≈4,78 GiB) | 0,0096 s | ≈6,5 s |
| 16384 | 5689 MiB | 5 620 482 702 (≈5,23 GiB) | 0,0095 s | ≈6,5 s |
| 32768 (plafond du modèle) | 6617 MiB | 6 593 561 230 (≈6,14 GiB) | 0,0096 s | ≈6,5 s |

**Lecture** : l'effet sur le traitement du prompt est **négligeable**
(≈9,5-9,6 ms quel que soit le palier, sur un prompt court — le coût
dépend de la taille du prompt, pas du plafond `num_ctx` configuré).
Le coût réel est en VRAM : **+1,9 GiB environ** entre 4096 et 32768.

**Marge pour le modèle de chat, mesurée par coexistence forcée** (le
modèle de complétion chargé à 32768, puis le modèle de chat sollicité
directement) : la coexistence **ne se produit jamais** — Ollama
décharge intégralement le modèle de complétion avant de charger le
modèle de chat (D22, chargement séquentiel confirmé par échantillonnage
VRAM à 150 ms pendant la bascule : 6637 MiB → 47 MiB → montée
progressive → 7841 MiB stable, jamais de palier intermédiaire montrant
les deux modèles ensemble). **Conséquence directe** : élargir
`num_ctx` du modèle de complétion **ne réduit jamais** la VRAM
disponible pour le modèle de chat quand celui-ci est actif seul — les
deux ne se chevauchent jamais en VRAM. Le seul risque réel d'un
élargissement porte sur la **fenêtre de complétion elle-même** face au
budget total mesuré (≈15,9 GiB, `docs/local-ai.md` § 2.3), sans lien
avec la coexistence.

**Valeur retenue : 4096 (inchangée)**. Motif : le rechargement mesuré
(§ 3, ≈6,5 s) n'a pas d'incidence en régime établi (une seule fois,
au changement de valeur, pas à chaque requête), et l'effet sur la
latence de traitement du prompt est négligeable — l'arbitrage
« coexistence avec le chat » invoqué par la demande ne joue en fait
pas de rôle (les deux modèles ne se chevauchent jamais). **Mais aucun
fichier de ce dépôt n'approche 4096 jetons de contexte extrait** (les
fichiers Python/YAML/bash du dépôt font quelques centaines de lignes
au plus, `max_context: 2000` ≈ 8000 caractères couvre déjà largement
ce qui est utile, § 11.4) — élargir `num_ctx` au-delà de 4096
n'apporterait donc **aucun bénéfice mesurable sur l'usage réel de ce
poste**, contre un coût VRAM certain (+1,9 GiB au plafond). Rester à
4096 est la conclusion retenue : ni la coexistence, ni la latence ne
motivent un changement, et rien ne motive le coût VRAM en l'absence
de fichiers assez longs pour en profiter.

### 12.3 — Rechargement nécessaire, comment le déclencher sans redémarrer le service

Confirmé par mesure : changer `num_ctx` sur un modèle déjà résident
**exige son rechargement** — un appel avec un `num_ctx` différent de
celui actuellement chargé déclenche un `load_duration` de ≈6,5 s (contre
≈0,13-0,16 s pour un rappel avec la même valeur, `docs/completion.md`
§ 12.2 mesures brutes). **Ce rechargement est automatique et
transparent** : Ollama recharge le modèle dès la première requête qui
porte un `num_ctx` différent de la session en cours — aucune commande
d'arrêt/relance du service n'est nécessaire, aucun redémarrage.
Concrètement pour ce rôle : la prochaine complétion envoyée par
`lsp-ai` après un changement de configuration recharge le modèle
d'elle-même (le fichier de configuration déployé porte la nouvelle
valeur, la requête suivante la transmet). Rien à déclencher
manuellement par l'opérateur.

### 12.4 — Application : six occurrences Kate, une source

Vérifié après exécution réelle du rôle :
```
$ python3 -c "
import json
d=json.load(open('/home/mahieumi/.config/kate/lspclient/settings.json'))
for lang,s in d['servers'].items():
    print(lang, s['initializationOptions']['completion']['parameters'])
"
bash {'max_context': 2000, 'options': {'num_ctx': 4096, 'num_predict': 256}}
python {'max_context': 2000, 'options': {'num_ctx': 4096, 'num_predict': 256}}
yaml {'max_context': 2000, 'options': {'num_ctx': 4096, 'num_predict': 256}}
```
**Six occurrences confirmées identiques** (2 clés × 3 langages) —
toutes générées par le même gabarit
(`kate-lspclient-settings.json.j2`, structure `lsp_ai_options`
partagée, référencée une seule fois puis répétée par la boucle
`for lang in completion_kate_languages`) à partir des mêmes deux
variables (`completion_num_predict`, `completion_num_ctx`), jamais
recopiées à la main. Côté Helix, un seul bloc
(`languages.toml.j2`) : `num_predict = 256` / `num_ctx = 4096`,
confirmé identique par lecture de `~/.config/helix/languages.toml`.

### 12.5 — Preuve : troncature disparue, latence après changement

**Kate, complétion Python réelle, réponse complète** (journal
`journalctl --user _PID=<pid>`, `LSPCLIENT_DEBUG=1`, fichier
`fib.py` enregistré, docstring `"""Return the nth Fibonacci
number."""` suivi d'une ligne vide, clic + Ctrl+Espace) :
```
$ grep -n 'language id\|calling textDocument/completion\|adding completions' kate-journal-py-final.log
1:Aug 12 10:59:49 Zephyrus-MM kate[70274]: language id  "python"
200:Aug 12 11:00:32 Zephyrus-MM kate[70274]: calling textDocument/completion
220:Aug 12 11:00:33 Zephyrus-MM kate[70274]: adding completions  1
```
Réponse JSON-RPC réelle relue dans le même journal :
```
{"jsonrpc":"2.0","id":2,"result":{"isIncomplete":false,"items":[{"filterText":"","kind":1,
"label":"ai -     # Base cases\n    if n == 0:\n        return 0\n    elif n == 1:\n
return 1\n    \n    # Recursive case\n    else:\n        return fibonacci(n-1) +
fibonacci(n-2)\n        \nprint(fibonacci(7))\n```\n\nThis will output `13`, which is the
seventh Fibonacci number. The function uses recursion to calculate the value, with a base
case for when `n` is 0 or 1, and a recursive case that adds together the two preceding
numbers in the sequence.", ...}]}}
```
**Arrêt naturel confirmé** (le texte se termine par une phrase
complète, pas coupée en cours de mot) — à l'ancienne valeur de 32,
cette même requête aurait été tronquée après le premier commentaire
`# Base cases` sans jamais atteindre le corps de la fonction. Réponse
reçue en **≈1 s** (11:00:32 → 11:00:33), pas d'attente perceptible
malgré la marge de 256 : ce cas particulier s'est arrêté naturellement
bien avant le plafond.

**Latence mesurée après changement, plusieurs appels, distinguant
arrêt naturel de plafonnement** (méthode d'isolement de
`docs/dgpu-power.md`, API directe, modèle résident à `num_ctx: 4096`) :

| Appel | `eval_count` | `done_reason` | Latence murale |
|---|---|---|---|
| 1 | 111 | `stop` (arrêt naturel) | 1,187 s |
| 2 | 234 | `stop` (arrêt naturel) | 2,354 s |
| 3 | 94 | `stop` (arrêt naturel) | 1,045 s |
| 4 | 108 | `stop` (arrêt naturel) | 1,160 s |
| 5 | 256 | `length` (plafonné) | 2,564 s |

**Comparé à la référence de 0,46-0,48 s** (mesurée à l'ancienne valeur
de 32, systématiquement plafonnée) : la nouvelle valeur dépasse
largement ce seuil dès qu'une réponse va au-delà d'une trentaine de
jetons — **1 à 2,6 s selon le contenu généré, jamais aussi rapide
qu'avant**. C'est la contrepartie assumée par l'opérateur (§ 12.1) :
la garantie de réponse rapide au fil de la frappe est perdue au
profit de la complétude. Fonctionnellement toujours cohérent avec un
usage de complétion déclenchée (Ctrl+Espace/Ctrl+x), pas un
affichage continu à chaque caractère tapé.

**`num_ctx` effectivement appliqué**, prouvé par `ollama ps`
(`curl .../api/ps`) après exécution réelle du rôle :
```
{"context_length": 4096}
```
confirmé inchangé (valeur retenue = valeur déjà active avant ce
livrable, aucune bascule nécessaire pour la valeur finale).

### 12.6 — Non-régression, trois langages, deux éditeurs

**Helix** (`hx -vv`, session `tmux` pilotée, bout en bout machine,
même méthode que § 7-8, § 10.3) :
- Python (`fib.py`, Ctrl+x) : `textDocument/completion` → réponse
  reçue, complétion FIM réelle.
- Bash (`greet.sh`, Ctrl+x) : deux requêtes envoyées, une réponse
  arrêtée naturellement (script Debian complet), une plafonnée à 256
  (`done`/coupure en cours d'explication, `"1. **"` final) — cohérent
  avec la mesure § 11.3/12.5, pas une régression.
- YAML (`config.yaml`, Ctrl+x) : `textDocument/completion` → réponse
  reçue, structure GitHub Actions générée (`on:`/`jobs:`/`steps:`).

**Kate**, preuve mixte comme en § 9/10.4 (lecture du journal = preuve
machine ; déclenchement clic+Ctrl+Espace = observation rapportée par
l'opérateur, marquée, datée 2026-08-12, attribuée) :
- Python : § 12.5 ci-dessus.
- Bash (`greet.sh`) :
  ```
  $ grep -n 'language id\|calling textDocument/completion\|adding completions' kate-journal-sh-final.log
  1:Aug 12 11:01:17 Zephyrus-MM kate[71411]: language id  "bash"
  197:Aug 12 11:01:26 Zephyrus-MM kate[71411]: calling textDocument/completion
  217:Aug 12 11:01:27 Zephyrus-MM kate[71411]: adding completions  1
  ```
- YAML (`config.yaml`) :
  ```
  $ grep -n 'language id\|calling textDocument/completion\|adding completions' kate-journal-yaml-final.log
  1:Aug 12 11:02:08 Zephyrus-MM kate[71536]: language id  "yaml"
  200:Aug 12 11:02:33 Zephyrus-MM kate[71536]: calling textDocument/completion
  220:Aug 12 11:02:35 Zephyrus-MM kate[71536]: adding completions  1
  ```

**Obstacle méthodologique nouveau, résolu** : `LSPCLIENT_DEBUG=1`
associé à une redirection shell classique (`kate ... > fichier.log
2>&1 &`) **n'a pas capturé la sortie de façon fiable** dans plusieurs
tentatives — Kate double-forke en interne (le PID rendu par le shell
n'est pas le processus final, `ps -ef`/`pgrep` le confirment), et
selon le chemin exact de démarrage (session D-Bus déjà active ou non),
la sortie peut atterrir sur `/dev/null` au lieu du fichier attendu
(vérifié : `/proc/<pid>/fd/1` pointait vers `/dev/null` dans les
tentatives infructueuses, vers le fichier attendu dans les
concluantes — aucune règle fiable identifiée pour prédire laquelle des
deux se produit). **Contournement retenu** : `journalctl --user
_PID=<pid>` — Kate journalise systématiquement via le journal
utilisateur systemd, indépendamment de la redirection shell, quel que
soit le chemin de démarrage. Nommé ici pour la prochaine fois qui en
aurait besoin, à la suite de la découverte équivalente de KAT-1/BSH-1
sur la variable d'environnement elle-même (§ 9.3, § 10.4).

### 12.7 — Échec forcé

**Aucune garde n'existe** sur `completion_num_predict`,
`completion_max_context` ni `completion_num_ctx` — vérifié par lecture
de `roles/completion/tasks/main.yml` (aucune tâche `assert` ne les
mentionne). Démontré : `-e completion_num_ctx=99999999` est accepté
sans erreur et écrit tel quel dans les deux configurations déployées
(`changed=2`, aucun rejet). **Ce n'est pas un défaut de ce livrable**
— la demande demande de dire l'absence de garde plutôt que d'en
inventer une, ce qui est fait ici : une valeur hors bornes serait
silencieusement acceptée jusqu'à échouer côté Ollama lui-même (limite
matérielle de VRAM ou erreur de l'API), pas interceptée par ce rôle.

### 12.8 — Branches exercées et non exercées

Branches du rôle touchées par ce livrable :
- **Écriture Helix** (`languages.toml.j2`) : exercée (exécution réelle,
  `changed=2` puis `changed=0`).
- **Écriture Kate** (`kate-lspclient-settings.json.j2`) : exercée,
  mêmes exécutions.
- **Garde de valeur hors bornes sur les nouveaux paramètres** :
  **non exercée par une garde du rôle, parce qu'aucune garde
  n'existe** (§ 12.7) — la seule branche observable est celle,
  triviale, où la valeur est acceptée telle quelle.
- **Rechargement automatique du modèle sur `num_ctx` changé** :
  exercée directement contre l'API (§ 12.2/12.3, mesuré), **jamais
  exercée via une exécution réelle du rôle qui changerait
  effectivement `completion_num_ctx` du dépôt** — la valeur retenue
  (4096) est restée celle déjà active, donc le chemin réel
  « le rôle change `num_ctx`, le service se recharge en conséquence
  au prochain appel » n'a pas été traversé par ce livrable. Signalé
  comme branche non exercée, pas affirmé comme prouvé.

### 12.9 — Actions privilégiées

Non applicable — aucune élévation, aucune commande `sudo` dans ce
livrable (édition de fichiers du dépôt, exécution `ansible-playbook`
sans privilège, appels HTTP locaux, pilotage d'éditeurs en tant
qu'utilisateur courant).

### 12.10 — Confirmations finales

`ansible-lint --profile production roles/completion/` : 0 défaut.
`ansible-playbook --check` : `changed=0` (avant application).
Exécution réelle : `changed=2` (les deux fichiers de configuration
déployés changent, `num_predict`/`num_ctx` désormais présents/modifiés).
Seconde exécution : `changed=0` — idempotence confirmée. Aucun paquet
installé, aucun modèle téléchargé (le modèle de complétion et le
modèle de chat étaient déjà récupérés par `roles/local_ai/`, jamais
par ce rôle). `roles/local_ai/`, `roles/editor/`, `sudoers`,
`terra.repo`, `/etc/cdi/`, `gpu_mux_mode`, `kwinrulesrc`, `site.yml`
intacts (aucune commande de ce livrable ne les touche). Aucun
redémarrage système — seul le modèle Ollama a été rechargé en interne
(comportement normal du service, pas un redémarrage de service ni de
machine) ; Kate a été relancé plusieurs fois par l'opérateur pour la
démonstration, jamais la session ni le système.

## 13. TRO-2 — gardes de bornes sur les paramètres de génération (2026-08-12)

**Motif** : la démonstration de la phase B précédente (§ 12.7) a
montré qu'aucune garde n'existait sur `completion_num_predict`,
`completion_max_context` ni `completion_num_ctx` — une valeur
manifestement absurde (`completion_num_ctx=99999999`) était acceptée
sans rejet et écrite telle quelle dans les deux configurations
déployées. Ce livrable comble l'asymétrie : le dépôt garde partout
ailleurs, ces trois variables étaient les seules exceptions.

### 13.1 — Bornes retenues, chacune avec son motif sourcé

**`completion_num_predict`** :
- **Borne basse = 1.** Établie par mesure directe contre l'API : `0`
  et les valeurs négatives (`-1`, `-2`) signifient « génération non
  bornée » côté Ollama, pas « aucune complétion ». Vérifié :
  `num_predict=-1` a produit une réponse de **1334 jetons** en un seul
  appel ; `num_predict=0` s'est arrêté à 65 jetons dans un essai mais
  sans plafond garanti (dépend de l'arrêt naturel du modèle, jamais
  borné par la valeur elle-même). C'est le risque exact qu'une borne
  basse doit écarter — une génération non bornée coûterait un temps
  arbitraire, pas seulement quelques secondes de plus.
- **Borne haute = 512.** Mesurée directement contre l'API (isolement
  de `docs/dgpu-power.md`, prompt volontairement long pour forcer
  l'atteinte du plafond) :

  | `num_predict` | Latence murale mesurée | `done_reason` |
  |---|---|---|
  | 256 (retenu, § 12.1) | jusqu'à ≈2,5 s | `length` au plafond |
  | 384 | ≈3,81 s | `length` au plafond |
  | 512 | ≈5,03 s | `length` au plafond |
  | 768 | ≈7,50 s | `length` au plafond |
  | 1024 | ≈9,93 s | `length` au plafond |

  512 reste la plus grande valeur qui ne franchit pas le palier où
  l'attente devient un blocage net (7 à 10 s, mesuré à 768/1024) —
  au-delà, aucune valeur ne resterait défendable au fil de la frappe,
  y compris pour une complétion déclenchée explicitement plutôt
  qu'automatique (§ 11.1 — aucune distinction n'existe côté `lsp-ai`
  entre les deux). 512 reste le double de la valeur retenue (256),
  laissant une marge d'ajustement sans ouvrir sur un temps d'attente
  qui rendrait la fonctionnalité inutilisable en pratique.

**`completion_num_ctx`** :
- **Borne haute = 32768.** Établie **par lecture**, pas devinée :
  relevé directement sur les métadonnées du modèle chargé,
  ```
  $ curl -s http://127.0.0.1:11434/api/show \
      -d '{"model":"qwen2.5-coder:7b-instruct-q4_K_M"}' \
      | python3 -c "import json,sys;d=json.load(sys.stdin);print(d['model_info']['qwen2.context_length'])"
  32768
  ```
  Vérifié qu'au-delà, la valeur demandée est acceptée par l'API sans
  erreur mais **silencieusement plafonnée** : `num_ctx=99999999` a
  produit un modèle chargé dont `ollama ps`/`api/ps` continue de
  rapporter `context_length: 32768` — la valeur au-delà de la borne ne
  produit donc pas l'effet demandé, ce qui motive la borne haute
  (empêcher une configuration qui semble demander plus que ce que le
  modèle peut réellement offrir, sans jamais le signaler).
- **Borne basse = 512.** Établie par mesure : en dessous, la part du
  contexte consommée par le prompt système et le gabarit de `lsp-ai`
  (~30 jetons mesurés pour un prompt minimal, `prompt_eval_count`)
  devient une fraction significative de la fenêtre totale. Testé à
  1/2/4/64/128/256 : le modèle continue de répondre sans erreur
  (aucun échec technique), mais la portion de fichier réellement
  visible s'effondre bien avant que la complétion perde en cohérence
  syntaxique — la complétion perd son **sens utilitaire** pour ce
  dépôt (fichiers de quelques centaines de lignes), pas parce qu'elle
  échoue. 512 est la valeur à partir de laquelle un fragment de
  fichier significatif reste représenté dans la fenêtre, cohérente
  avec la borne basse de `num_predict` (les deux partagent la même
  logique de « en dessous, la fonctionnalité perd son sens plutôt que
  d'échouer bruyamment »).

**`completion_max_context` : aucune borne posée, délibérément.**
Établi en phase A (§ 11.1) et reconfirmé ici : ce paramètre est interne
au *memory backend* `file_store` de `lsp-ai` — jamais transmis à
Ollama, converti en une estimation de caractères
(`tokens_to_estimated_characters`, `tokens * 4`,
`crates/lsp-ai/src/utils.rs:34`), sans coût VRAM, sans risque d'échec
côté service. Aucun fait mesuré ne motive une borne : une valeur trop
grande ne coûte qu'un peu plus de texte extrait du fichier local (pas
de VRAM, pas de requête réseau, pas de risque de dépassement de
capacité du modèle puisque c'est `num_ctx` qui borne réellement ce
qui est transmis). Poser une borne ici serait de la sur-couverture —
mieux vaut l'absence de garde, motivée, que trois gardes dont une
arbitraire.

### 13.2 — Ce que la garde empêche, et ce qu'elle ne juge pas

La garde protège contre une valeur **manifestement hors domaine** — un
ordre de grandeur erroné (`99999999`, une faute de frappe qui ajoute
des zéros), une valeur qui déclenche un comportement différent de
celui voulu (`0`/négatif = non borné). **Elle ne juge jamais de la
pertinence d'un réglage plausible mais mal choisi** : une valeur comme
`completion_num_predict=500` ou `completion_num_ctx=600` passerait la
garde sans encombre, alors que ce ne sont pas nécessairement de bons
réglages pour ce poste. Le message de succès de la garde le dit
explicitement : « domaine admissible, aucun jugement sur la
pertinence du réglage » — pour éviter de laisser croire que la garde
valide un choix de configuration, seulement qu'il n'est pas absurde.

### 13.3 — Emplacement, avant toute écriture

La garde s'exécute immédiatement après la garde 3/4bis (existence du
modèle côté service) et **avant** toute tâche de clonage/compilation
et **avant** toute tâche `template` qui écrirait `languages.toml` ou
le fichier de configuration Kate — vérifié par lecture de l'ordre des
tâches dans `roles/completion/tasks/main.yml`. Aucune configuration
n'est jamais écrite, même transitoirement, avant que cette garde ait
réussi.

**Vérifiée dans les deux contextes d'exécution** — le piège du
contexte s'est déjà présenté ailleurs dans ce dépôt
(`local_ai_gpu_cdi_playbook`, `playbook_dir` vs `role_path`) :

```
$ ansible-playbook roles/completion/completion.yml -e completion_num_ctx=99999999
[...] fatal: [localhost]: FAILED! => { "assertion": "completion_num_ctx | int <= completion_num_ctx_max", ... }

$ ansible-playbook --check site.yml --tags completion -e completion_num_ctx=99999999
[...] fatal: [localhost]: FAILED! => { "assertion": "completion_num_ctx | int <= completion_num_ctx_max", ... }
```
Même échec, même message, dans les deux cas.

### 13.4 — Démonstrations dans les deux sens

**Nominal** : valeurs courantes (`num_predict=256`, `num_ctx=4096`),
exécution réelle, `changed=0` — rien ne change, la garde n'interfère
pas.
```
$ ansible-playbook roles/completion/completion.yml
[...]
localhost : ok=38 changed=0 unreachable=0 failed=0 skipped=5 rescued=0 ignored=0
```

**Cas limites, aux deux extrémités — les valeurs à la borne passent** :
```
$ ansible-playbook roles/completion/completion.yml -e completion_num_predict=1 -e completion_num_ctx=512
localhost : ok=38 changed=2 unreachable=0 failed=0 skipped=5 rescued=0 ignored=0

$ ansible-playbook roles/completion/completion.yml -e completion_num_predict=512 -e completion_num_ctx=32768
localhost : ok=38 changed=2 unreachable=0 failed=0 skipped=5 rescued=0 ignored=0
```
(`changed=2` attendu ici : la valeur limite diffère de la valeur
retenue par défaut, donc les deux fichiers déployés se réécrivent —
pas un échec, la garde inclusive laisse bien passer les bornes
elles-mêmes.) Rôle rejoué ensuite avec les valeurs par défaut pour
restaurer l'état nominal (`changed=2` de nouveau, retour à
`num_predict=256`/`num_ctx=4096`).

**Échec forcé, par garde — avant écriture, message complet, sommes de
contrôle identiques avant/après** :
```
$ sha256sum ~/.config/helix/languages.toml ~/.config/kate/lspclient/settings.json
0f3ef4fa9242d09c4e57ffd008635a80182d4063f6b226fd171e3d14bf377efb  .../languages.toml
ba50e497a5e560265d219f20271e39ae426ec835bc4e0278312d771deb5d9f3d  .../settings.json

$ ansible-playbook roles/completion/completion.yml -e completion_num_ctx=99999999
[...]
fatal: [localhost]: FAILED! => {
    "assertion": "completion_num_ctx | int <= completion_num_ctx_max",
    "msg": "completion_num_predict=256 (domaine admissible : [1, 512]) ou
      completion_num_ctx=99999999 (domaine admissible : [512, 32768]) est
      hors domaine. Bornes et motif : completion_num_predict_min=1 (0 et
      les valeurs négatives signifient « génération non bornée » côté
      Ollama...) ; completion_num_predict_max=512 (mesuré : au-delà, la
      latence dépasse largement ce qui reste utilisable au fil de la
      frappe...) ; completion_num_ctx_min=512 (en deçà, une fraction
      significative du contexte système/prompt lsp-ai est déjà
      consommée, la complétion perd son sens) ; completion_num_ctx_max=
      32768 (fenêtre de contexte maximale déclarée par le modèle
      lui-même, qwen2.context_length, relevée via curl .../api/show —
      jamais devinée). Ce rôle s'arrête avant d'écrire quoi que ce soit."
}
localhost : ok=13 changed=0 unreachable=0 failed=1 skipped=2 rescued=0 ignored=0

$ sha256sum ~/.config/helix/languages.toml ~/.config/kate/lspclient/settings.json
0f3ef4fa9242d09c4e57ffd008635a80182d4063f6b226fd171e3d14bf377efb  .../languages.toml   # identique
ba50e497a5e560265d219f20271e39ae426ec835bc4e0278312d771deb5d9f3d  .../settings.json   # identique
```
`changed=0`, `failed=1` : la garde échoue avant toute tâche d'écriture
(le décompte `ok=13` s'arrête bien avant les tâches de compilation et
de déploiement, `skipped=2` couvre les tâches de nettoyage
conditionnelles). Sommes de contrôle strictement identiques avant et
après l'échec.

**Validation par le code d'avant (`git show HEAD:...`, rejoué contre le
code tel qu'il était avant ce livrable, commit `b21f45b`)** — même
échec forcé, contre le rôle sans la nouvelle garde :
```
$ git stash   # remise en état du code au commit b21f45b (avant ce livrable)
$ ansible-playbook roles/completion/completion.yml -e completion_num_ctx=99999999
[...]
localhost : ok=37 changed=2 unreachable=0 failed=0 skipped=5 rescued=0 ignored=0

$ grep num_ctx ~/.config/helix/languages.toml
num_ctx = 99999999
```
**La version d'avant accepte la valeur absurde et l'écrit
intégralement** (`changed=2`, `num_ctx = 99999999` déployé tel quel) —
c'est la preuve que la garde ajoutée dans ce livrable bouche un trou
réel, pas qu'elle ajoute du zèle à un mécanisme déjà correct. Code
restauré (`git stash pop`), rôle rejoué avec les valeurs nominales pour
revenir à l'état correct (`num_predict=256`/`num_ctx=4096`).

### 13.5 — Non-régression

Après restauration des valeurs nominales, sommes de contrôle des deux
fichiers déployés identiques à l'état d'avant ce livrable :
```
0f3ef4fa9242d09c4e57ffd008635a80182d4063f6b226fd171e3d14bf377efb  languages.toml
ba50e497a5e560265d219f20271e39ae426ec835bc4e0278312d771deb5d9f3d  settings.json
```
**Une vérification fonctionnelle complète (3 langages × 2 éditeurs)
n'a pas été rejouée** : aucune valeur de complétion elle-même n'est
modifiée par ce livrable (seules des gardes sont ajoutées, jamais
déclenchées en usage nominal), et la preuve à trois langages/deux
éditeurs vient d'être établie dans le livrable précédent (§ 12.5-12.6)
sur des fichiers déployés strictement identiques (sommes de contrôle
ci-dessus). Une vérification fonctionnelle légère aurait ajouté un
geste opérateur (Kate) sans rien contrôler de plus que la somme de
contrôle ne prouve déjà.

### 13.6 — Rappel de méthode consigné

**La lecture du journal LSP de Kate passe par `journalctl --user
_PID=<pid>`, pas par une redirection shell classique** —
`LSPCLIENT_DEBUG=1 kate ... > fichier.log 2>&1` peut échouer
silencieusement (Kate se re-dédouble en interne, la sortie peut
atterrir sur `/dev/null` selon le chemin de démarrage exact, sans
règle fiable identifiée pour prédire lequel des deux se produit).
Consigné en détail avec l'exemple complet en § 12.6 de ce document —
retrouvable par une recherche de « journalctl --user _PID » dans ce
fichier.

### 13.7 — Branches exercées et non exercées

- **Garde de bornes, cas nominal (valeur dans le domaine)** : exercée
  (§ 13.4, exécution nominale, `changed=0`).
- **Garde de bornes, valeurs aux deux extrémités (bornes inclusives)** :
  exercée pour les quatre bornes (`num_predict`∈{1,512},
  `num_ctx`∈{512,32768}).
- **Garde de bornes, échec (valeur hors domaine haute)** : exercée
  (`num_ctx=99999999`, `num_predict=0`, `num_predict` hors domaine
  testé aussi via l'échec combiné).
- **Garde de bornes, échec par valeur hors domaine basse** : **non
  exercée directement en dessous de la borne basse** (ex.
  `num_ctx=100` ou `num_predict=-1` n'ont pas été rejoués contre le
  rôle complet, seulement contre l'API Ollama directement pour établir
  le motif de la borne, § 13.1) — la logique de l'assertion est
  symétrique (`>= min` et `<= max` dans la même expression `that:`) et
  couvre les deux bornes par construction, mais le chemin d'exécution
  réel du rôle avec une valeur sous la borne basse n'a pas été
  traversé. Signalé comme branche non exercée plutôt qu'affirmé
  couvert par analogie.
- **Exécution depuis `site.yml --tags completion`** : exercée pour le
  cas d'échec (§ 13.3), **non exercée pour le cas nominal ni les cas
  limites** depuis ce point d'entrée — seule l'exécution directe du
  rôle (`ansible-playbook roles/completion/completion.yml`) a été
  utilisée pour ces cas, `site.yml` uniquement pour confirmer que le
  contexte d'exécution ne change pas le comportement de la garde en
  échec.

### 13.8 — Actions privilégiées

Non applicable — aucune élévation, aucune commande `sudo` dans ce
livrable (ajout de tâches `assert`, appels HTTP locaux vers l'API
Ollama sans privilège, exécutions `ansible-playbook` en tant
qu'utilisateur courant).

### 13.9 — Confirmations finales

`ansible-lint --profile production roles/completion/` : 0 défaut.
`ansible-playbook --check` : aucun changement signalé. Exécution
réelle nominale : `changed=0` (les valeurs déployées étaient déjà
correctes, la garde ne modifie aucun comportement en usage normal).
Aucun paquet installé, aucun modèle téléchargé (aucune commande de ce
livrable n'installe ni ne télécharge quoi que ce soit — uniquement des
appels API en lecture et des exécutions du rôle existant).
`roles/local_ai/`, `roles/editor/`, `sudoers`, `terra.repo`,
`/etc/cdi/`, `gpu_mux_mode`, `kwinrulesrc`, `site.yml` intacts (aucune
commande de ce livrable ne les touche — `site.yml` a été **lu et
exécuté en `--check`/`--tags completion`** pour la démonstration du
contexte d'exécution, jamais modifié). Aucun redémarrage système ; le
modèle Ollama a été rechargé plusieurs fois en interne au fil des
essais de valeurs (comportement normal du service, jamais un
redémarrage de service ni de machine).

## 14. TRO-3 — résolution en lecture seule du mode FIM (2026-08-12)

**Question posée** : les troncatures persistantes viennent-elles
réellement d'un plafond trop bas, ou d'un mode de sollicitation qui
fait produire au modèle de la prose explicative plutôt qu'une
complétion de code — auquel cas augmenter `num_predict` ferait passer
une explication tronquée en explication complète, pas mieux.
**Phase de lecture seule : aucune configuration déployée, aucun rôle,
aucune valeur modifiés.**

### 14.1 — Caractérisation du problème réel, dix essais

Établi **avec le prompt exact que `lsp-ai` construit dans le chemin
actuel** : `format_prompt()` (`crates/lsp-ai/src/utils.rs:61`) produit
`"{context}\n\n{code}"`, avec `context` vide dans la configuration de
ce dépôt (backend `file_store` seul, pas de `context` supplémentaire),
donc en pratique `"\n\n" + code_avant_curseur`. Envoyé à
`/api/generate` avec `"raw": true` (`ollama.rs:87` — le chemin actuel
n'utilise ni `messages`, ni le champ natif `suffix` d'Ollama, ni
`fim`).

**Dix cas représentatifs** (python/bash/yaml, fonctions vides, docstrings,
boucles, commentaires-question, configuration), même prompt exact,
`num_predict: 256`/`num_ctx: 4096` (valeurs déployées) :

| Cas | Classement | Arrêt | Jetons |
|---|---|---|---|
| python-fib-docstring | code pur | naturel | 93 |
| python-empty-func | code+explication | naturel | 165 |
| python-class-method | code pur | naturel | 118 |
| python-loop | code pur | naturel | 30 |
| bash-greet | code+explication | naturel | 93 |
| bash-loop | code+explication | **plafonné** | 256 |
| yaml-workflow | **explication seule** | naturel | 76 |
| yaml-config | code+explication | naturel | 254 |
| python-comment-question | code pur | **plafonné** | 256 |
| python-docstring-only | code pur | **plafonné** | 256 |

**Proportion sur ces dix essais** : 5/10 code pur, 4/10 code+explication,
1/10 explication seule — soit **la moitié des réponses contient de la
prose**, et les trois cas plafonnés (30 %) le sont tous alors qu'ils
génèrent une suite de texte discursif plutôt qu'un arrêt naturel après
quelques lignes de code. **Conclusion de cette mesure** : le problème
n'est pas uniquement un plafond trop bas — une part significative des
réponses est du texte explicatif que le modèle continue de produire
jusqu'au plafond, augmenter `num_predict` ferait passer ces cas d'une
explication tronquée à une explication complète, pas à une complétion
de code.

### 14.2 — Support FIM de `lsp-ai`, avec le moteur Ollama

Établi par lecture du code à l'empreinte compilée
(`1e910a8cf0048406eb227bf2064743010a9ff3a9`) :

- **`lsp-ai` supporte un mode FIM générique**, indépendant du backend :
  `Prompt::FIM(FIMPrompt { prompt, suffix })`
  (`crates/lsp-ai/src/memory_backends/mod.rs:50`), construit par
  `file_store.rs:261-278` (`PromptType::FIM` — extrait un préfixe et un
  suffixe autour du curseur, chacun borné par `max_context / 2`
  caractères estimés).
- **Le backend Ollama prend en charge ce mode** :
  `do_chat_completion` (`ollama.rs:196-211`) traite explicitement
  `Prompt::FIM(fim) => match &params.fim { Some(fim_params) => ... }` —
  il construit une chaîne unique
  `format!("{start}{prompt}{middle}{suffix}{end}", ...)` à partir des
  jetons fournis dans `params.fim`, puis l'envoie **telle quelle** à
  `get_completion` (même chemin `raw: true` que le mode actuel — le
  FIM de `lsp-ai` avec Ollama est une **concaténation de chaîne
  manuelle**, pas un appel au champ natif `suffix` de l'API Ollama).
- **Déclenchement** : `get_prompt_type()` (comportement par défaut,
  `transformer_backends/mod.rs:47-56`, non redéfini par le backend
  Ollama) bascule en `PromptType::FIM` **si et seulement si** la clé
  `"fim"` est présente dans les paramètres de la requête LSP — sinon
  `PromptType::ContextAndCode` (chemin actuel). Un seul interrupteur :
  la présence de `parameters.fim` dans la configuration.
- **Emplacement exact de configuration**, vérifié par lecture (test
  intégré `llama_cpp_config`, `config.rs:517-524`, structure générique
  — `Kwargs`, un objet JSON arbitraire fusionné dans `options` avant
  d'atteindre le backend, même mécanisme que `num_predict`/`num_ctx`) :
  ```toml
  [language-server.lsp-ai.config.completion.parameters.fim]
  start = "..."
  middle = "..."
  end = "..."
  ```
  **Même niveau** que `max_context`/`num_predict`/`num_ctx` — sous
  `completion.parameters`, pas sous `completion.parameters.options`
  (`fim` est un champ propre de `OllamaRunParams`, pas une clé
  arbitraire d'`options` comme `num_predict`/`num_ctx`,
  `ollama.rs:19-20`).

### 14.3 — Jetons FIM du modèle, sourcés, et vérification de la variante

**Établis par lecture des métadonnées du modèle côté Ollama**
(`curl .../api/show`, champ `template` — gabarit Go exposé par Ollama
lui-même, pas deviné) :
```
{{- if .Suffix }}<|fim_prefix|>{{ .Prompt }}<|fim_suffix|>{{ .Suffix }}<|fim_middle|>
{{- else if .Messages }} ... (mode chat/instruct)
{{- else }} ... (mode instruct sans historique)
{{- end }}
```
**Trois jetons identifiés, sourcés directement sur ce modèle** :
`<|fim_prefix|>`, `<|fim_suffix|>`, `<|fim_middle|>` — le gabarit
bascule lui-même en mode FIM dès qu'un champ `Suffix` est fourni dans
la requête, **indépendamment de `raw`**. C'est la clé de la mesure
§ 14.1/14.4 : le chemin actuel de `lsp-ai` (raw, sans `suffix` natif)
ne bénéficie jamais de ce gabarit, quel que soit le paramétrage
`num_predict`/`num_ctx`.

**Variante installée** : `qwen2.5-coder:7b-instruct-q4_K_M` — une
variante **`instruct`**, orientée dialogue/chat, pas une variante
`base` dédiée au remplissage au milieu à l'origine. **Fait notable,
établi par mesure** (§ 14.1, § 14.4) : malgré cela, le gabarit livré
par Ollama pour ce modèle **gère explicitement le cas `.Suffix`** avec
les mêmes jetons FIM que la famille de base — et la mesure confirme
que la variante `instruct` produit des complétions de code pures et
courtes dès que ce chemin est emprunté (§ 14.4, 9/10 code pur). **Rien
n'indique ici que la variante installée soit mal choisie** pour le
FIM — la question ouverte par la demande (« la variante n'est peut-être
pas la bonne ») ne se pose pas dans les faits sur ce modèle précis,
constaté plutôt que supposé.

### 14.4 — Essai comparatif, par l'API seulement, mêmes dix cas

**Chemin FIM natif Ollama** (paramètre `suffix` de l'API `/api/generate`,
sans `raw`, gabarit du modèle lui-même applique les jetons — **pas
le chemin exact de `lsp-ai`**, une comparaison intermédiaire) :

| Cas | Classement | Arrêt | Jetons | Latence |
|---|---|---|---|---|
| python-fib-docstring | code pur | naturel | 27 | 0,399 s |
| python-empty-func | code pur | naturel | 19 | 0,328 s |
| python-class-method | code pur | **plafonné** | 256 | 2,549 s |
| python-loop | code pur | naturel | 6 | 0,199 s |
| bash-greet | code pur | naturel | 9 | 0,223 s |
| bash-loop | code pur | naturel | 33 | 0,447 s |
| yaml-workflow | code+explication | naturel | 136 | 1,422 s |
| yaml-config | code pur | naturel | 8 | 0,232 s |
| python-comment-question | code pur | naturel | 9 | 0,219 s |
| python-docstring-only | code pur | naturel | 6 | 0,211 s |

**Chemin FIM exact de `lsp-ai`** (concaténation manuelle
`<|fim_prefix|>{prefix}<|fim_suffix|>{suffix}<|fim_middle|>`, `raw:
true` — reproduit précisément `ollama.rs:196-211`, mêmes dix cas,
mêmes suffixes) :

| Cas | Classement | Arrêt | Jetons | Latence |
|---|---|---|---|---|
| python-fib-docstring | code pur | naturel | 33 | 0,456 s |
| python-empty-func | code pur | naturel | 12 | 0,238 s |
| python-class-method | code pur | naturel | 27 | 0,389 s |
| python-loop | code pur | naturel | 39 | 0,519 s |
| bash-greet | code pur | naturel | 9 | 0,226 s |
| bash-loop | code pur | naturel | 25 | 0,372 s |
| yaml-workflow | code pur | naturel | 47 | 0,581 s |
| yaml-config | code pur | naturel | 4 | 0,175 s |
| python-comment-question | code pur | naturel | 9 | 0,218 s |
| python-docstring-only | code pur | naturel | 7 | 0,194 s |

**Comparaison** :

| | Chemin actuel (`ContextAndCode`, raw) | Chemin FIM exact `lsp-ai` |
|---|---|---|
| Code pur | 5/10 | **10/10** |
| Code+explication | 4/10 | 0/10 |
| Explication seule | 1/10 | 0/10 |
| Plafonnements (`length`) | 3/10 | **0/10** |
| Jetons médians | ≈124 | ≈18 |
| Latence médiane | non mesurée (éditeurs), API : ≈1-2,5 s à 256 jetons | **0,2-0,58 s** |

**Le chemin FIM exact de `lsp-ai` élimine la troncature sur les dix
cas testés** — non pas en autorisant plus de jetons, mais en
produisant des réponses **naturellement courtes** (4 à 47 jetons,
arrêt naturel dans 100 % des cas), parce que le modèle reçoit un
signal structurel (jetons FIM + suffixe réel du fichier) qui le
contraint à compléter plutôt qu'à dialoguer.

### 14.5 — Conclusion et options

**Issue 1 retenue par les faits : FIM est configurable et améliore
nettement les réponses.** `num_predict` peut rester à sa valeur
actuelle (256, TRO-1) — le mécanisme qui produisait les troncatures
observées par l'opérateur (réponses discursives qui plafonnent) est
contourné par le mode FIM, pas par un plafond plus haut. Augmenter
`num_predict` sans passer par FIM aurait bien réduit la fréquence des
plafonnements bruts, mais aurait laissé passer des réponses de
prose complètes plutôt que des complétions de code — ce n'est pas ce
que l'opérateur attend d'un outil de complétion.

**Ce que cette mesure ne tranche pas** : la configuration FIM n'a
jamais été déployée ni testée en conditions réelles d'éditeur (Kate,
Helix) — seulement par appel direct à l'API, avec des préfixes/suffixes
choisis à la main plutôt qu'extraits par le mécanisme réel de
`file_store.rs` (bornage par `max_context / 2` de chaque côté du
curseur, comportement différent d'un simple découpage manuel sur des
exemples courts). Les valeurs actuelles de `completion_max_context`
(2000) et `completion_num_ctx`/`completion_num_predict` (TRO-1/TRO-2)
n'ont pas été réévaluées dans ce contexte — un préfixe/suffixe FIM
généré par `file_store.rs` à partir d'un fichier réel de ce dépôt
pourrait se comporter différemment des dix cas courts testés ici.

**Questions qui départagent, pour un éventuel livrable d'application** :
- L'extraction réelle de préfixe/suffixe par `file_store.rs` (bornée
  par `max_context`) produit-elle des résultats comparables sur des
  fichiers réels de ce dépôt (quelques centaines de lignes), pas
  seulement sur les dix extraits courts de ce livrable ?
- Le gain de latence mesuré ici (0,2-0,58 s contre 1-2,5 s) se
  confirme-t-il en conditions réelles d'éditeur (Kate/Helix, y compris
  le coût de la poignée de main LSP) ?
- Le passage en FIM change-t-il le comportement pour un curseur en fin
  de fichier (suffixe vide) — cas fréquent en pratique, pas testé ici ?

### 14.6 — Actions privilégiées

Non applicable — uniquement des appels HTTP en lecture vers l'API
Ollama locale et la lecture de fichiers/code source, aucune élévation.

### 14.7 — Branches exercées et non exercées

Sans objet — aucun code du dépôt n'est modifié par ce livrable (lecture
seule stricte). Aucune branche de `roles/completion/` n'est exercée ni
modifiée.

### 14.8 — Confirmations finales

Sommes de contrôle des fichiers déployés, avant et après ce livrable —
**identiques**, aucune écriture :
```
0f3ef4fa9242d09c4e57ffd008635a80182d4063f6b226fd171e3d14bf377efb  languages.toml
ba50e497a5e560265d219f20271e39ae426ec835bc4e0278312d771deb5d9f3d  settings.json
```
Aucun rôle modifié (`git status` ne montre aucune modification sous
`roles/`). Modèle résident inchangé : `ollama ps`/`api/ps` rapporte
toujours `context_length: 4096` après les essais de ce livrable —
aucun appel de ce livrable n'a demandé un `num_ctx` différent de la
valeur actuellement chargée. Aucun paquet installé, aucun modèle
téléchargé, aucun redémarrage.

## 15. TRO-4 — application du mode FIM (2026-08-12)

Fait la suite de TRO-3 (§ 14, lecture seule) : accord explicite donné
pour appliquer le mode FIM. Rappel du motif retenu, à ne pas rouvrir
sans nouvelle mesure : le chemin FIM a produit 10/10 complétions de
code pur avec arrêt naturel (4-47 jetons, 0,2-0,58 s) contre 5/10 sur
le chemin brut (3 plafonnements, jusqu'à 2,5 s) — § 14.4-14.5.
**Conséquence pour la demande initiale de l'opérateur** (augmenter
`num_predict` contre les troncatures) : ce livrable ne le fait pas, et
ne doit pas le faire. Avec FIM, les complétions de code s'arrêtent
seules bien en deçà de 256 jetons (§ 15.3.1) — augmenter le plafond
n'aurait fait que produire des **explications complètes** là où le
problème réel était des **explications tronquées** : pas un mieux,
juste un défaut différent. `completion_num_predict` (256) et
`completion_num_ctx` (4096), avec leurs gardes de TRO-2, restent
inchangés dans ce livrable — vérifié § 15.7.

### 15.1 — Jetons FIM : source et couplage au modèle

Trois jetons exposés en variables dans
`roles/completion/defaults/main.yml` : `completion_fim_start`,
`completion_fim_middle`, `completion_fim_end`. Valeurs sourcées par
lecture directe du gabarit exposé par Ollama pour le modèle installé
— pas devinées, pas reprises d'une documentation générique sur la
famille Qwen :
```
curl -s http://127.0.0.1:11434/api/show \
  -d '{"model":"qwen2.5-coder:7b-instruct-q4_K_M"}' \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['template'])"
```
Extrait pertinent du gabarit : `{{- if .Suffix }}<|fim_prefix|>{{
.Prompt }}<|fim_suffix|>{{ .Suffix }}<|fim_middle|>`. Valeurs
retenues, identiques à celles déjà validées à la main en TRO-3 :
```
completion_fim_start:  "<|fim_prefix|>"
completion_fim_middle: "<|fim_suffix|>"
completion_fim_end:    "<|fim_middle|>"
```

**Couplage à consigner, explicitement** : ces trois valeurs sont
propres au modèle retenu (`completion_ollama_model`, D21), pas à
`lsp-ai` — un changement de modèle (D21 à rouvrir) invaliderait ces
jetons **sans qu'aucune erreur ne le signale nécessairement à
l'écriture de la configuration** (le champ reste un `String` Rust
ordinaire côté `lsp-ai`, aucune valeur n'est rejetée pour son
contenu — seulement pour sa présence, § 15.2). Une nouvelle lecture du
gabarit du modèle (même commande que ci-dessus) est donc nécessaire à
chaque changement de modèle, pas une simple relecture de ce document.

### 15.2 — Garde : `lsp-ai` guarde déjà la présence, pas la valeur

Établi **par lecture du code source à l'empreinte compilée**
(`1e910a8cf0048406eb227bf2064743010a9ff3a9`, `/tmp/lsp-ai-src`) puis
**confirmé empiriquement** par un test Rust autonome reproduisant
exactement la structure `FIM` de `lsp-ai`
(`crates/lsp-ai/src/config.rs:172-176` : `{ start: String, middle:
String, end: String }`, tous trois requis, sans `Option` ni
`#[serde(default)]`, struct annoté `#[serde(deny_unknown_fields)]`) :

```rust
#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
struct FIM { start: String, middle: String, end: String }

// {"start": "...", "middle": "..."}  (end manquant)
// → Err(Error("missing field `end`", line: 0, column: 0))

// {"start": "...", "middle": "...", "end": "..."}
// → Ok(FIM { ... })
```

**Conclusion : `lsp-ai` rejette déjà, avec une erreur explicite, une
configuration FIM où l'une des trois clés est absente** — le champ
`fim: Option<FIM>` de `OllamaRunParams`
(`crates/lsp-ai/src/transformer_backends/ollama.rs:19-20`) est
désérialisé par `serde_json::from_value(params)?`
(`ollama.rs:226`) : si la clé `"fim"` est absente du JSON, `fim`
devient `None` (chemin sans FIM, valide) ; si elle est présente mais
incomplète, la désérialisation entière échoue et l'erreur remonte
jusqu'à la réponse LSP — un échec bruyant, jamais un repli silencieux.

**Ce que cette garde ne peut pas faire, et que ce livrable ne tente
pas de faire** : `String` accepte une chaîne vide ou un jeton erroné
sans la moindre erreur — `lsp-ai` garantit la **présence structurelle
des trois clés ensemble**, jamais l'**exactitude de leur valeur**
vis-à-vis du modèle réellement chargé. C'est la même distinction que
celle déjà posée en TRO-2 entre « valeur dans le domaine admissible »
et « valeur correcte » — ici transposée à la présence plutôt qu'aux
bornes numériques.

**Décision : aucune garde Ansible ajoutée.** L'outil garde déjà,
loudly, l'incohérence structurelle que ce livrable devait empêcher —
en ajouter une au niveau du rôle serait de la sur-couverture pour un
cas déjà couvert, contrairement à la garde de bornes de TRO-2 qui,
elle, comblait une lacune réelle (`lsp-ai` n'a jamais borné
`num_predict`/`num_ctx`).

**Démonstration, dans les deux sens** (§ 4 du cadrage, transposé à la
présence plutôt qu'aux bornes) :
- **Absence complète de la clé `fim`** (état d'avant ce livrable,
  `76705f4`) : chemin valide, aucune erreur — § 15.5.3.
- **Présence incomplète** (un jeton non défini) : testé en supprimant
  `completion_fim_end` de `roles/completion/defaults/main.yml` (aucune
  autre variable ne le fournit) et en relançant le rôle :
  ```
  TASK [... Déployer languages.toml ...] ***
  [ERROR]: Task failed: 'completion_fim_end' is undefined
  fatal: [localhost]: FAILED! => {"changed": false,
    "msg": "Task failed: 'completion_fim_end' is undefined"}
  ```
  Échec **avant toute écriture** — Ansible refuse de rendre un modèle
  Jinja2 référençant une variable non définie, la tâche
  `ansible.builtin.template` échoue en amont de toute opération sur le
  fichier cible. Sommes de contrôle des deux fichiers déployés,
  identiques avant et après cet essai :
  ```
  616eaf2ebc0e92ef4aac3f4d27973bdebde36f8ab6ac0d8740e2300715b2ad61  languages.toml
  a84d2c46f28cc02389ea4947512abcaa3b1b2b312d11c46083e0e43b8e103cf7  settings.json
  ```
  Variable restaurée immédiatement après l'essai
  (`diff` contre la sauvegarde : aucune différence).
- **Présence complète mais chaînes vides** (`-e completion_fim_start=""
  -e completion_fim_middle="" -e completion_fim_end=""`) : **accepté
  sans rejet**, écrit tel quel (`start = ""` etc., § 15.5.4) — c'est
  exactement la limite décrite ci-dessus : `lsp-ai` garde la
  structure, pas le contenu. Utilisé délibérément comme mécanisme de
  neutralisation pour la démonstration d'échec forcé (§ 15.5.4),
  pas comme un défaut à corriger.

### 15.3 — Configuration : une seule source, six occurrences côté Kate

`languages.toml.j2` (Helix) : nouveau bloc
`[language-server.lsp-ai.config.completion.parameters.fim]`, au même
niveau que `max_context` (`completion.parameters`), **pas** sous
`.options` — `fim` est un champ propre de `OllamaRunParams` côté
`lsp-ai` (`ollama.rs:19-20`), pas une clé arbitraire transmise à
Ollama comme le sont `num_predict`/`num_ctx` (établi par lecture du
code, § 14.2, confirmé en pratique § 15.4.1).

`kate-lspclient-settings.json.j2` : le dictionnaire `lsp_ai_options`
partagé (§ 9, KAT-1) reçoit la même clé `fim`, générée depuis les
mêmes trois variables — toujours la source Jinja2 unique, jamais
recopiée pour les trois langages Kate. Comptage confirmé après
déploiement :
```
grep -c fim ~/.config/kate/lspclient/settings.json
18
```
18 occurrences du jeton `fim` (6 par langage × 3 langages : la clé
`fim` elle-même + `start`/`middle`/`end`, 2 fois chacune —
`initializationOptions` et `settings` portent le même bloc,
duplication déjà actée en KAT-1, § 9.3, pas ajoutée par ce livrable)
— toutes générées depuis `completion_fim_start/middle/end`, aucune
valeur saisie séparément.

### 15.4 — Preuve que la requête porte réellement la structure FIM

TRO-3 avait comparé les deux chemins en construisant les requêtes à la
main (§ 14.4). Ce livrable prouve que `lsp-ai`, une fois configuré,
envoie réellement la même structure — pas seulement que le fichier de
configuration contient le bloc `fim`.

#### 15.4.1 — Helix : configuration reçue par `lsp-ai`

Journal Helix (`~/.cache/helix/helix.log`, lancé avec `hx -vv`),
message `Using custom LSP config` envoyé à l'initialisation :
```
helix_lsp::client [INFO] Using custom LSP config: {"completion":
{"model":"completion-model","parameters":{"fim":{"end":"<|fim_middle|>",
"middle":"<|fim_suffix|>","start":"<|fim_prefix|>"},"max_context":2000,
"options":{"num_ctx":4096,"num_predict":256}}}, ...}
```
Confirme que la structure `fim` complète (les trois jetons) atteint
bien `lsp-ai` via `initializationOptions`/`workspace/didChangeConfiguration`
— pas seulement écrite dans le fichier, réellement transmise au
serveur. La requête HTTP que `lsp-ai` envoie ensuite à Ollama
(`ollama.rs:196-211`, concaténation manuelle
`{start}{prefix}{suffix}{end}` avec `raw:true`) n'est pas elle-même
journalisée côté Helix (elle sort du processus `lsp-ai`, pas de
`helix`) — mais la **nature des réponses obtenues** (§ 15.4.2, code
pur, arrêts naturels dans la fourchette 4-47 jetons de TRO-3) est la
preuve indirecte que la requête a bien pris la forme FIM : le chemin
brut (§ 15.5.3, TRO-3 § 14.4) produit un tout autre profil de réponse
sur les mêmes fichiers.

#### 15.4.2 — Helix : complétions réelles sur les trois langages

Fichiers de test préparés (`/tmp/tro4-test/{test.py,test.sh,test.yaml}`),
pilotage par `tmux`, déclenchement `Ctrl-x` en fin de dernière ligne.

**Python** (`def fibonacci(n): ... for _ in range(2, n + 1):`) —
requête à 12:17:19.295, réponse à 12:17:19.744, **latence 0,449 s** :
```
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":{"isIncomplete":false,
"items":[{"filterText":"def fibonacci(n):","kind":1,
"label":"ai - \n    \"\"\"Return the nth Fibonacci number using an
iterative approach.\"\"\"\n    # Base cases: if n is 0 or 1, return n
itself.", ...}]}}
```
Code pur (docstring + commentaire, pas de dialogue), arrêt naturel.

**Bash** (`#!/bin/bash\ncount_files() {\n    local dir="$1"`) —
requête à 12:17:53.713, réponse à 12:17:56.275, **latence 2,562 s**
(hors norme des 0,2-0,58 s API directe de TRO-3, à consigner
honnêtement plutôt qu'à masquer — § 15.4.4) :
```
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":{"isIncomplete":false,
"items":[{"filterText":"#!/bin/bash","kind":1,
"label":"ai - \n# Function to count the number of files in a
directory recursively\n\n# Importing necessary packages\nimport
os\n\n# Define the function\ncount_files() {\n    local
dir=\"$1\"\n ... }]}}
```
Code pur (aucune prose explicative en fin de réponse), mais une
dérive notable : `import os` — Python glissé dans un script bash, un
défaut de nature différente de la troncature/prose visée par ce
livrable, pas corrigé ici (hors périmètre : aucune modification des
jetons ou du modèle n'est demandée par ce cas isolé).

**YAML** (`services:\n  web:\n    image: nginx\n    ports:`) —
requête à 12:18:14.275, réponse à 12:18:14.683, **latence 0,408 s** :
```
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":{"isIncomplete":false,
"items":[{"filterText":"services:","kind":1,
"label":"ai - \n  db:\n    image: mysql\n    environment:\n
MYSQL_ROOT_PASSWORD: example\n  redis:\n    image: redis", ...}]}}
```
Code YAML pur, arrêt naturel, aucune trace de prose.

#### 15.4.3 — Point d'attention YAML, dit franchement

TRO-3 avait relevé qu'un cas YAML manuel laissait passer de la prose
(§ 14.4). **Sur ce test-ci, FIM produit un résultat propre pour
YAML** (§ 15.4.2 ci-dessus) — mais un seul cas ne suffit pas à
conclure à un gain systématique : le remplissage au milieu est conçu
pour du code, et YAML/Jinja n'a pas la même structure syntaxique
qu'un langage de programmation portant des jetons de bloc explicites.
**Conclusion honnête** : le gain FIM est **confirmé sur ce cas précis**,
**pas généralisable** à tout contenu YAML à partir d'un seul essai —
contrairement à Python et bash, testés de façon répétée à travers
TRO-3 et ce livrable.

#### 15.4.4 — Latence à travers l'éditeur, comparée à TRO-3

| Cas | TRO-3 (API directe) | TRO-4 (Helix, éditeur complet) |
|---|---|---|
| Python | 0,2-0,58 s | 0,449 s |
| Bash | 0,2-0,58 s | **2,562 s** (hors norme) |
| YAML | 0,2-0,58 s | 0,408 s |

Python et YAML restent dans la fourchette API directe malgré le coût
additionnel du magasin de mémoire et du transport LSP — cohérent avec
l'observation de TRO-1 (§ 12.5) que ce coût est faible. Le cas bash
est un écart, pas expliqué par ce livrable : une seule mesure, pas
reproduite plusieurs fois, ne permet pas de trancher entre une
variation ponctuelle du service (aucune preuve d'un rechargement de
modèle en cours, `ollama ps` non revérifié à cet instant précis — un
marqueur `@VERIF` serait de mise si une conclusion en dépendait, mais
aucune n'est tirée ici) et une propriété du prompt bash en particulier.
Consigné sans explication forcée plutôt que masqué.

### 15.5 — Kate : preuve par geste opérateur

Fichiers de test préparés dans `/tmp/tro4-kate/` (mêmes contenus que
§ 15.4.2), Kate lancé par la session avec `LSPCLIENT_DEBUG=1`,
`lsp-ai` confirmé démarré (`pgrep -a lsp-ai` → pid actif) avant de
solliciter le geste de l'opérateur (complétion sur les 3 fichiers,
un par un). Journal lu par `journalctl --user _PID=<pid kate>
--no-pager` — **jamais** par redirection de sortie standard, Kate se
re-dédouble en interne et la redirection ne suit pas le processus
réel (établi au prix d'une dizaine de tentatives lors du livrable
KAT-1, § 9.4 ; **rappel de méthode pour toute session future** :
c'est la seule voie fiable observée à ce jour sur ce poste).

#### 15.5.1 — Configuration FIM reçue par `lsp-ai` (Kate)

```
"start": "<|fim_prefix|>"
"start": "<|fim_prefix|>"
calling textDocument/completion
```
Occurrences multiples de `<|fim_prefix|>` dans le journal — la
configuration porte bien la structure FIM avant chaque requête de
complétion, cohérent avec Helix (§ 15.4.1).

#### 15.5.2 — Complétions réelles (Kate)

**Python** — réponse reçue :
```
{"jsonrpc":"2.0","id":2,"result":{"isIncomplete":false,"items":
[{"filterText":"","kind":1,"label":"ai -         a, b = b, a +
b\n    return b\n\n# Example usage:\nn = 10\nprint(f\"Fibonacci
number at position {n} is: {fibonacci(n)}\")", ...}]}}
```
Code pur, arrêt naturel — cohérent avec Helix.

**YAML** — réponse reçue :
```
{"jsonrpc":"2.0","id":2,"result":{"isIncomplete":false,"items":
[{"filterText":"","kind":1,"label":"ai -       - \"80:80\"", ...}]}}
```
Code YAML pur, très court, arrêt naturel.

**Bash** — réponse reçue :
```
{"jsonrpc":"2.0","id":2,"result":{"isIncomplete":false,"items":
[{"filterText":"","kind":1,"label":"ai -     find \"$dir\" -type f |
wc -l\n}\n\nusage() {\n ... \n```\n\nThis script defines a function
`count_files` ... The script then checks if exactly one argument
(the directory) is provided and if it exists, counts the number of
files and prints it.", ...}]}}
```
**À dire franchement** : ce cas Kate porte encore une explication en
prose après le code, malgré la configuration FIM active et confirmée
(§ 15.5.1) — contrairement au même cas testé sous Helix (§ 15.4.2, où
le résultat bash était du code pur sans prose). Deux exécutions
distinctes du même modèle sur un prompt structurellement identique
peuvent produire des sorties différentes (génération non
déterministe côté Ollama, absence de graine fixée) — **ce n'est pas
une régression de la configuration**, la structure FIM est bien
transmise dans les deux cas (§ 15.4.1/15.5.1), mais un résultat non
garanti à 100 % même une fois FIM actif, cohérent avec le taux de
10/10 mesuré par TRO-3 sur des essais à la main (un taux élevé,
jamais annoncé comme absolu).

#### 15.5.3 — Distinction complétion du modèle / complétion par mots

Rappel établi en TRO-1 (§ 12.4) : la complétion du modèle et celle,
interne, de Kate par mots du document sont visuellement
indiscernables à l'écran. Cette limite n'est pas levée par ce
livrable — **la preuve continue de reposer exclusivement sur le
journal LSP** (présence de `calling textDocument/completion` et de la
réponse `result` associée), jamais sur la présence d'un popup.

#### 15.5.4 — Échec forcé (Kate/Helix, config neutralisée)

Configuration FIM neutralisée par trois chaînes vides (`-e
completion_fim_start="" -e completion_fim_middle="" -e
completion_fim_end=""`, § 15.2 — accepté sans rejet par `lsp-ai`,
`start = ""` etc. écrits dans les deux fichiers déployés) puis testé
sous Helix sur le cas Python :
```
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":{"isIncomplete":false,
"items":[{"filterText":"def fibonacci(n):","kind":1,
"label":"ai -  \n    if n==0: \n        return 0\n    elif n==1: \n
return 1\n    else: \n        return(fibonacci(n-1)+fibonacci(n-2))\n
\nprint(fibonacci(5)) # Output will be 5\n\n# In this program, we
have used a recursive function to calculate the nth Fibonacci
number.\n# A Fibonacci number is a sequence of numbers where each
number after the first two is the sum of the two preceding ones. \n
# The first two numbers are usually defined as 0 and 1. \n\n# In
this case, when n=5, the output will be 5 because the Fibonacci
sequence is 0, 1, 1, 2, 3, 5 and so on.", ...}]}}
```
550 caractères, code + longue explication en prose — le comportement
dégradé est bien revenu. **Le résultat n'est pas identique dans les
deux cas** (FIM actif contre FIM neutralisé) : la démonstration
prouve donc quelque chose de réel. Configuration restaurée
immédiatement après l'essai (`ansible-playbook
roles/completion/completion.yml`, sommes de contrôle revenues à
`616eaf2e.../a84d2c46...`).

#### 15.5.5 — Validation par le code d'avant (`76705f4`)

Rôle restauré à l'état exact de `76705f4` (`git checkout 76705f4 --
roles/completion/`, confirmé : `grep -c fim` sur les trois fichiers
concernés → `0` partout, aucune trace de FIM), rôle réexécuté,
même cas Python testé sous Helix :
```
lsp-ai <- {"jsonrpc":"2.0","id":1,"result":{"isIncomplete":false,
"items":[{"filterText":"def fibonacci(n):","kind":1,
"label":"ai -  \n    if n==0: \n        return 0\n    elif n==1: \n
        return 1\n    else: \n        return(fibonacci(n-1)+
fibonacci(n-2))\n\nprint(fibonacci(5)) # Output will be 5\n\n# In
this program, we have used a recursive function to calculate the
nth Fibonacci number.\n# A Fibonacci number is a sequence of numbers
...", ...}]}}
```
550 caractères, comportement dégradé identique à celui obtenu par
neutralisation (§ 15.5.4) — confirme que le code d'avant ce livrable
produit bien le défaut que ce livrable corrige, et que la
configuration FIM de ce livrable est la cause réelle de
l'amélioration, pas une coïncidence. Rôle restauré ensuite à l'état
de ce livrable (`git checkout HEAD -- roles/completion/`, puis
`git stash pop` pour les modifications non commitées de la session),
sommes de contrôle des fichiers déployés revenues à
`616eaf2e.../a84d2c46...` après réexécution.

### 15.6 — Non-régression : trois langages, deux éditeurs

Confirmée par construction des essais eux-mêmes (§ 15.4.2, § 15.5.2) :
python/bash/yaml produisent tous une réponse de complétion exploitée
par Kate et par Helix, aucune erreur de handshake, aucun échec de
démarrage de `lsp-ai` observé sur aucun des six essais (3 langages ×
2 éditeurs).

### 15.7 — Confirmation : `num_predict`/`num_ctx` inchangés

```
grep -E "^completion_num_predict|^completion_num_ctx" \
  roles/completion/defaults/main.yml
completion_num_ctx: 4096
completion_num_predict: 256
```
Valeurs identiques à celles fixées en TRO-1 phase B et bornées en
TRO-2 — ce livrable ne touche ni les valeurs ni leurs gardes
(`completion_num_predict_min/max`, `completion_num_ctx_min/max`,
inchangées, `git diff` ne montre aucune modification de ces lignes).

### 15.8 — Branches exercées et non exercées

| Branche | Exercée | Comment |
|---|---|---|
| Clé `fim` absente (chemin d'avant, `Option<FIM> = None`) | Oui | § 15.5.5, reconstruit depuis `76705f4` |
| Clé `fim` présente, complète (chemin nominal) | Oui | § 15.4, § 15.5.1-15.5.2 |
| Clé `fim` présente, incomplète (un jeton non défini) | Oui | § 15.2, `completion_fim_end` retiré, échec Ansible avant écriture |
| Clé `fim` présente, complète mais valeurs vides (accepté par `lsp-ai`) | Oui | § 15.2, § 15.5.4, neutralisation délibérée |
| Exécution du rôle seul (`completion.yml`) | Oui | tous les essais ci-dessus |
| Exécution depuis `site.yml` | **Non** | ce livrable n'a pas rejoué `site.yml` — le rôle `completion` y est inclus sans changement structurel depuis KAT-1/BSH-1 (aucune variable de portée changée, `role_path`/`playbook_dir` non concernés ici, contrairement au cas `local_ai_gpu_cdi_playbook`, TRO-1 note technique) ; branche non exercée, signalée plutôt qu'omise |
| Complétion multi-essai sur le même cas (déterminisme du résultat) | Partiellement | un seul essai par cas et par éditeur ; § 15.5.2 (bash Kate contre bash Helix) montre déjà que deux exécutions du même prompt peuvent diverger |

### 15.9 — Actions privilégiées

| Action | Tentative sans privilège : résultat |
|---|---|
| Aucune action privilégiée dans ce livrable | Non applicable — modification de fichiers utilisateur (`roles/completion/`, configurations déployées sous `$HOME`), lecture de code source local, appels HTTP locaux ; aucune écriture sous `/etc/`, aucun `sudo` |

### 15.10 — Confirmations finales

Aucun paquet installé, aucun modèle téléchargé (modèle résident
inchangé : `qwen2.5-coder:7b-instruct-q4_K_M`, aucun nouveau
téléchargement). `completion_num_predict`/`completion_num_ctx`
inchangés (§ 15.7). Fichiers interdits intacts (`roles/local_ai/`,
`roles/editor/`, `sudoers`, `terra.repo`, `/etc/cdi/`, `gpu_mux_mode`,
`kwinrulesrc`, `site.yml` — `git status` ne montre aucune modification
en dehors du périmètre annoncé). Aucun redémarrage.

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
  livrable (§ 9, KAT-1, pour la partie Kate ; § 10, BSH-1, pour
  bash/shell).
- [`roles/editor/`](../roles/editor/) — greffon LSP client de Kate
  activé (KAT-1, § 9.1-9.2).
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici, en
  particulier la garde sur les découvertes qui contredisent un fait
  déjà documenté, et la règle 0.3 (élévation précédée d'une tentative
  sans privilège), ajoutée par ce livrable.
