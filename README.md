# workstation-config

Ce dépôt décrit et reproduit la configuration d'un poste de développement
personnel : ASUS ROG Zephyrus Duo 16 GX650PY (Ryzen 9 7945HX, 61 Gio RAM,
RTX 4090 Laptop), Fedora 44 KDE Plasma sous Wayland. Il rassemble
l'automatisation (Ansible, bash, Python, Jinja), les conteneurs Podman et la
configuration d'un modèle IA local exécuté sur le GPU.

La cible de production réelle de l'utilisateur est Ansible Automation
Platform 2.6 sur RHEL 9 — **cette cible n'est pas gérée depuis ce dépôt**.
Le contenu Ansible ici sert de banc d'édition et de test local (voir
décision D3 dans [`docs/machine-facts.md`](docs/machine-facts.md)) ; il ne
fait pas autorité pour la production.

## Avertissements

- **Machine de développement personnelle.** Aucune donnée professionnelle,
  aucun secret (clé privée, jeton, mot de passe), aucun adressage
  d'infrastructure interne, aucun nom de serveur ou de domaine
  professionnel n'est présent dans ce dépôt. Le hostname et le nom
  d'utilisateur de ce poste personnel, en revanche, sont assumés et
  apparaissent tels quels (décision D4, amendée le 2026-08-04 dans
  `docs/machine-facts.md`) : les expurger casserait la traçabilité des
  faits sourcés sans rien protéger, l'identité de l'auteur figurant de
  toute façon dans chaque commit. Si vous trouvez un secret ou une donnée
  d'entreprise, c'est un bug — ouvrez une issue.
- **Le dépôt est public par choix.** Il documente une machine réelle à un
  instant donné ; certains constats (`docs/machine-facts.md`) sont datés et
  peuvent être périmés. Chaque fait y porte la commande qui l'a produit —
  fiez-vous à la commande, pas seulement au chiffre affiché.
- **Réutilisation en entreprise.** Ce dépôt peut être cloné et adapté, à
  condition de substituer les variables spécifiques à cette machine
  (matériel, chemins, identifiants de dépôts) par les vôtres. Rien ici n'est
  prévu pour être exécuté tel quel contre une infrastructure de production —
  voir en particulier `docs/machine-facts.md` pour ce qui est spécifique au
  matériel ASUS ROG (attributs `asus-armoury`, bascule MUX GPU).

## Contenu

- [`CLAUDE.md`](CLAUDE.md) — règles persistantes pour les sessions d'agent
  travaillant sur ce dépôt, chacune motivée.
- [`docs/machine-facts.md`](docs/machine-facts.md) — inventaire sourcé du
  poste : matériel, système, dépôts, GPU, stockage, affichage, conteneurs,
  chaîne Ansible, décisions prises et points encore ouverts.
