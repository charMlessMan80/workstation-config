# roles/copilot_cli

Rôle Ansible ciblant `localhost`. Installe GitHub Copilot CLI
(`@github/copilot`, version épinglée) comme second agent de code —
décision de l'opérateur D24 (révision de D12), `docs/editor.md` §
Copilot CLI. Résolution complète, sources citées et démonstrations
sont dans [`docs/editor.md`](../../docs/editor.md) — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. Installe le runtime Node.js requis (`nodejs22-bin`,
   `nodejs22-npm-bin` — dépôts `fedora`/`updates` uniquement,
   `disablerepo: terra` explicite, `install_weak_deps: false` pour
   exclure la documentation et l'internationalisation complète, sans
   usage ici).
2. Relève puis, si nécessaire, définit le préfixe npm dans le domaine
   utilisateur (`~/.local`, jamais `/usr` ni `/usr/local`).
3. Relève la version actuellement installée de `@github/copilot` ;
   installe la version épinglée (`copilot_cli_version`) seulement si
   absente ou différente — jamais `@latest`.
4. Relit l'état installé (boucle fermée) et échoue bruyamment si la
   version ne correspond pas à la cible, ou si le binaire n'est pas au
   chemin attendu (`~/.local/bin/copilot`).

## Ce que ce rôle ne fait jamais

- **Il n'authentifie jamais** — `copilot login` reste un geste manuel
  de l'opérateur (flux OAuth interactif, navigateur ou code
  d'appareil). Ce rôle ne lit, n'écrit ni ne vérifie aucun jeton
  (`COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`).
- Il n'installe jamais rien depuis `terra`.
- Il n'écrit jamais dans `/usr` ou `/usr/local` au-delà du runtime
  Node.js lui-même (paquets `dnf`) — le paquet npm va exclusivement
  dans `~/.local`.
- Il n'installe jamais `@latest` ni ne lance `npm update -g`/
  `copilot update` — seule la version épinglée dans `defaults/main.yml`
  est installée, un changement de version est un changement délibéré
  de cette valeur.
- Il ne touche jamais `sudoers`, `/etc/cdi/`, `kwinrc`, `kwinrulesrc`,
  `gpu_mux_mode`.
- **Il ne redémarre jamais la machine, ne déconnecte jamais la
  session.**

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, en particulier `copilot_cli_version` (version épinglée,
seule voie d'intégrité disponible pour une installation npm globale)
et `copilot_cli_npm_prefix` (domaine utilisateur, même discipline que
`completion_bin_dir` de `roles/completion/`).

## Utilisation

```
ansible-playbook --syntax-check roles/copilot_cli/copilot_cli.yml
ansible-playbook --check roles/copilot_cli/copilot_cli.yml
ansible-playbook roles/copilot_cli/copilot_cli.yml                 # écriture réelle

# Après exécution, authentification manuelle (jamais scriptée) :
copilot login
```
