# roles/completion

Rôle Ansible ciblant `localhost`. Compile
[`lsp-ai`](https://github.com/SilasMarvin/lsp-ai) depuis les sources
(D19, `docs/machine-facts.md` § Décisions) — jamais le binaire
précompilé (aucune somme de contrôle publiée à côté, ancrage de
confiance plus faible que ce que npm aurait apporté). Rattache le
binaire compilé à Helix **et à Kate** (KAT-1) pour les langages listés
(D19/D20, révise la décision D12/EDI-1 de n'inscrire aucun serveur de
langage — motif détaillé dans `templates/languages.toml.j2` et
`templates/kate-lspclient-settings.json.j2`). **Ne charge et ne choisit
aucun modèle** (D17 : dimensionnement initial, D20 historique — voir
note ci-dessous) — le service qui charge réellement le modèle
(`roles/local_ai/`) est **résident en permanence** depuis
`[RÉALIGNÉE le 2026-08-12, BSH-2]` (`docs/machine-facts.md` §
Décisions) : D20 avait requalifié D15 en « à la demande » le
2026-08-07, réalignée depuis sur la résidence. Fait inchangé par ce
réalignement : ce rôle-ci ne charge ni ne choisit de modèle, quelle
que soit la politique de résidence de `roles/local_ai/`.
Résolution complète, sources citées et démonstrations sont dans
[`docs/completion.md`](../../docs/completion.md) — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. **Garde** : aucun binaire `node`/`npm` sur le PATH (D12 reste en
   vigueur), échec bruyant sinon.
2. **Chaîne Rust** : tentative sans privilège d'abord (`command -v`),
   installation de `rust`/`cargo` (`fedora`/`updates` uniquement,
   jamais `terra`) seulement si absente — seule installation système de
   ce rôle, exception explicitement accordée à la chaîne de
   compilation. Garde après coup : la chaîne est bien fonctionnelle.
3. **Garde** : le service d'inférence local (`roles/local_ai/`) répond
   sur `127.0.0.1:11434`, échec bruyant sinon.
4. **Clonage** de `lsp-ai` à l'empreinte de commit épinglée (jamais une
   étiquette), dans un répertoire de travail hors de ce dépôt
   (`~/.cache/completion-build/`). Vérifie que `Cargo.lock` existe à
   cette empreinte — s'arrête sinon, ce fichier porte l'argument
   d'intégrité qui fonde D19.
5. **Compilation** : `cargo build --release --locked` (fonctionnalités
   par défaut uniquement — pas `llama_cpp`/`cuda`/`metal`, aucune
   compilation de `llama.cpp` depuis les sources). `--locked` force
   l'échec plutôt qu'une révision silencieuse hors du fichier de
   verrouillage.
6. **Installation** du binaire compilé dans le domaine utilisateur
   uniquement (`~/.local/bin/lsp-ai`), jamais `/usr/local/`. Garde
   après coup : le binaire installé rapporte sa version.
7. **Configuration Helix** (`~/.config/helix/languages.toml`) : `lsp-ai`
   rattaché aux langages listés, pointant explicitement
   `127.0.0.1:11434` — jamais une API distante. Garde après coup :
   relecture du fichier réellement écrit, pas seulement la variable
   posée.
8. **Configuration Kate** (`~/.config/kate/lspclient/settings.json`,
   KAT-1) : mêmes langages, même binaire, même modèle, même API locale
   que Helix — fichier dédié au client LSP, jamais une copie d'un
   fichier Plasma. Garde après coup : relecture du fichier réellement
   écrit, couvre exactement les langages attendus.

## Ce que ce rôle ne fait jamais

- Il ne télécharge **aucun binaire `lsp-ai` précompilé**, sous aucune
  forme (D19 — seule la compilation depuis les sources est retenue).
- Il ne charge et ne choisit **aucun modèle** (D17/D20).
- Il n'installe jamais aucun serveur de langage npm, ni n'accepte de
  substitut à `lsp-ai` (D12 resserrée, D24) — npm lui-même est ouvert
  ailleurs (`roles/copilot_cli/`), pas ici.
- Il n'installe rien hors du domaine utilisateur, à l'exception de la
  chaîne de compilation Rust (`fedora`/`updates` uniquement, jamais
  `terra`).
- Il ne modifie jamais SELinux, `sudoers`, aucun dépôt.
- Il ne redémarre jamais la machine, ne déconnecte jamais la session.

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire. Substituables pour les démonstrations d'échec forcé, sans
jamais changer l'état réel du système :
`completion_forbidden_lsp_servers`, `completion_lsp_ai_expected_path`,
`completion_rust_check_binaries`, `completion_ollama_url_check`.

## Utilisation

```
ansible-playbook --syntax-check roles/completion/completion.yml
ansible-playbook --check roles/completion/completion.yml
ansible-playbook roles/completion/completion.yml                 # écriture réelle

# Démonstrations d'échec forcé (docs/completion.md § 3) :
ansible-playbook roles/completion/completion.yml \
  -e completion_forbidden_lsp_servers='["sh"]'
ansible-playbook roles/completion/completion.yml \
  -e completion_lsp_ai_expected_path='/nonexistent'
ansible-playbook roles/completion/completion.yml \
  -e completion_rust_check_binaries='["cargo_absent_garanti"]'
ansible-playbook roles/completion/completion.yml \
  -e completion_ollama_url_check=http://127.0.0.1:1/api/tags
ansible-playbook roles/completion/completion.yml \
  -e completion_ollama_model_expected=absent-garanti
```

## Après ce rôle

`lsp-ai` est compilé, installé et rattaché à Helix pour
`{{ completion_helix_languages }}` (voir `defaults/main.yml`),
configuré pour le modèle réel `{{ completion_ollama_model }}` (D21,
`roles/local_ai/`) — vérifié présent côté service avant l'écriture de
la configuration. **Aucun modèle n'est chargé par ce rôle lui-même**
(D22 : la bascule est gérée par Ollama, pas par ce rôle) : le premier
appel de complétion paie le chargement **si le modèle n'était pas déjà
résident** — depuis `[RÉALIGNÉE le 2026-08-12, BSH-2]`
(`docs/machine-facts.md` § Décisions), le modèle de complétion reste
chargé en continu (`roles/local_ai/`, `keep_alive` infini), donc ce
coût ne se paie plus qu'après un redémarrage du service ou une bascule
vers un autre modèle (IA-3 § 9.3/9.4), pas à chaque appel comme au
régime « à la demande » (D20, historique).
