# roles/editor

Rôle Ansible ciblant `localhost`. Installe Helix (terminal) et Kate
(graphique) — choix de l'opérateur D13, `docs/editor.md` § 2 — et les
configure sobrement. Résolution complète, sources citées et
démonstrations sont dans [`docs/editor.md`](../../docs/editor.md) —
ce README ne duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Installe `helix` et `kate` (dépôts `fedora`/`updates` uniquement,
   `disablerepo: terra` explicite — jamais Terra pour ce rôle).
2. **Garde D12 (resserrée, D24)** : échoue bruyamment si un serveur de
   langage de `editor_forbidden_lsp_servers`
   (`yaml-language-server`/`ansible-language-server` par défaut) est
   présent sur le `PATH`, ou si `lsp-ai` résout ailleurs qu'au binaire
   compilé attendu (`editor_lsp_ai_expected_path`) — npm lui-même est
   ouvert ailleurs (`roles/copilot_cli/`, D24), mais aucun serveur de
   langage ni aucun substitut de `lsp-ai` ne doit en provenir pour cet
   éditeur.
3. Déploie `~/.config/helix/config.toml` (une option : numérotation de
   ligne absolue, pour corréler avec `ansible-lint`/`yamllint`).
   Aucun `languages.toml` déployé — celui fourni par `helix-parsers`
   documente déjà l'absence de serveur de langage, comportement
   attendu par D12.
4. Active deux greffons Kate (`externaltoolsplugin`,
   `katekonsoleplugin`) par clé nommée dans `~/.config/katerc`
   (`kwriteconfig6`, jamais une copie de fichier).
5. Déploie deux outils externes Kate
   (`~/.config/kate/externaltools/{ansible-lint,yamllint}-d3a`),
   chacun pointant explicitement le binaire du venv D3a — jamais un
   second `ansible-lint`/`yamllint` installé.
6. Vérifie qu'`ansible-lint` (venv D3a) reste appelable et rapporte
   toujours `ansible-core:2.20.7` — preuve qu'aucun second binaire n'a
   été introduit par ce rôle.

## Ce que ce rôle ne fait jamais

- Il n'installe jamais aucun serveur de langage npm, ni n'accepte de
  substitut à `lsp-ai` — garde D12 (resserrée, D24), démontrée dans les
  deux sens (`docs/editor.md` § Copilot CLI). npm lui-même n'est plus
  interdit sur ce poste (D24), mais reste hors du périmètre de ce
  rôle : `roles/copilot_cli/` l'installe, jamais celui-ci.
- Il n'installe jamais rien depuis `terra`.
- Il n'écrit jamais de configuration Kate par copie de fichier —
  clés nommées uniquement, même discipline qu'à BUR-1 sur
  `kwinrulesrc`.
- Il ne touche jamais `sudoers`, `/etc/cdi/`, `kwinrc`, `kwinrulesrc`,
  `gpu_mux_mode`.
- **Il ne redémarre jamais la machine, ne déconnecte jamais la
  session.**

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, en particulier `editor_forbidden_lsp_servers` (cible du
premier volet de la garde D12 resserrée), `editor_lsp_ai_expected_path`
(cible du second volet) et `editor_ansible_lint_venv` (binaire faisant
autorité, D3a — jamais un second installé à côté).

## Utilisation

```
ansible-playbook --syntax-check roles/editor/editor.yml
ansible-playbook --check roles/editor/editor.yml
ansible-playbook roles/editor/editor.yml                 # écriture réelle

# Démonstrations d'échec forcé (garde D12 resserrée, docs/editor.md § Copilot CLI) :
ansible-playbook -e '{"editor_forbidden_lsp_servers": ["sh"]}' roles/editor/editor.yml
ansible-playbook -e '{"editor_lsp_ai_expected_path": "/nonexistent"}' roles/editor/editor.yml
```
