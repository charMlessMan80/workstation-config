# roles/completion

Rôle Ansible ciblant `localhost`. Compile
[`lsp-ai`](https://github.com/SilasMarvin/lsp-ai) depuis les sources
(D19, `docs/machine-facts.md` § Décisions) — jamais le binaire
précompilé (aucune somme de contrôle publiée à côté, ancrage de
confiance plus faible que ce que npm aurait apporté). Rattache le
binaire compilé à Helix pour les langages listés (D19/D20, révise la
décision D12/EDI-1 de n'inscrire aucun serveur de langage — motif
détaillé dans `templates/languages.toml.j2`). **Ne charge et ne choisit
aucun modèle** (D17/D20 : complétion à la demande, pas résidente).
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

## Ce que ce rôle ne fait jamais

- Il ne télécharge **aucun binaire `lsp-ai` précompilé**, sous aucune
  forme (D19 — seule la compilation depuis les sources est retenue).
- Il ne charge et ne choisit **aucun modèle** (D17/D20).
- Il n'installe jamais `node`, `npm`, ni aucun serveur de langage par
  ce canal (D12).
- Il n'installe rien hors du domaine utilisateur, à l'exception de la
  chaîne de compilation Rust (`fedora`/`updates` uniquement, jamais
  `terra`).
- Il ne configure jamais Kate — compatibilité jamais testée par
  personne (point ouvert, `docs/completion.md` § 2.1).
- Il ne modifie jamais SELinux, `sudoers`, aucun dépôt.
- Il ne redémarre jamais la machine, ne déconnecte jamais la session.

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire. Substituables pour les démonstrations d'échec forcé, sans
jamais changer l'état réel du système :
`completion_forbidden_binaries`, `completion_rust_check_binaries`,
`completion_ollama_url_check`.

## Utilisation

```
ansible-playbook --syntax-check roles/completion/completion.yml
ansible-playbook --check roles/completion/completion.yml
ansible-playbook roles/completion/completion.yml                 # écriture réelle

# Démonstrations d'échec forcé (docs/completion.md § 3) :
ansible-playbook roles/completion/completion.yml \
  -e completion_forbidden_binaries='["sh"]'
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
(D20/D22) : le premier appel de complétion paie le chargement (ou une
bascule si un autre modèle est déjà chargé, IA-3 § 9.3/9.4), pas un
échec de ce rôle.
