# roles/editor

Rôle Ansible ciblant `localhost`. Installe Helix (terminal) et Kate
(graphique) — choix de l'opérateur D13, `docs/editor.md` § 2 — et les
configure sobrement. Résolution complète, sources citées et
démonstrations sont dans [`docs/editor.md`](../../docs/editor.md) —
ce README ne duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Installe `helix` et `kate` (dépôts `fedora`/`updates` uniquement,
   `disablerepo: terra` explicite — jamais Terra pour ce rôle).
2. **Garde D12** : échoue bruyamment si un binaire de
   `editor_forbidden_binaries` (`node`, `npm` par défaut) est présent
   sur le `PATH` — npm n'est pas une surface d'approvisionnement
   ouverte pour cet éditeur.
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

- Il n'installe jamais `node`, `npm`, ni aucun serveur de langage —
  garde D12, démontrée dans les deux sens (§ 2.3 de `docs/editor.md`).
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
commentaire, en particulier `editor_forbidden_binaries` (cible de la
garde D12) et `editor_ansible_lint_venv` (binaire faisant autorité,
D3a — jamais un second installé à côté).

## Utilisation

```
ansible-playbook --syntax-check roles/editor/editor.yml
ansible-playbook --check roles/editor/editor.yml
ansible-playbook roles/editor/editor.yml                 # écriture réelle

# Démonstration d'échec forcé (garde D12, § docs/editor.md § 2.3) :
ansible-playbook -e '{"editor_forbidden_binaries": ["sh"]}' roles/editor/editor.yml
```
