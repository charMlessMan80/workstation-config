# workstation-config

Ce dépôt décrit et reproduit la configuration d'un poste de développement
personnel : ASUS ROG Zephyrus Duo 16 GX650PY (Ryzen 9 7945HX, 61 Gio RAM,
RTX 4090 Laptop), Fedora 44 KDE Plasma sous Wayland. Il rassemble
l'automatisation (Ansible, bash, Python, Jinja), les conteneurs Podman et la
configuration d'un modèle IA local exécuté sur le GPU.

La cible de production réelle de l'utilisateur est Ansible Automation
Platform 2.6 sur RHEL 9 — **cette cible n'est pas gérée depuis ce dépôt**.
L'`ansible-core` système de cette machine fait autorité pour l'exécution et
la vérification des rôles **de ce dépôt** (décision D3a,
[`docs/ansible-chain.md`](docs/ansible-chain.md)) ; une chaîne distincte,
différée, resterait à construire pour du contenu réellement destiné à AAP
2.6 (D3b, non ouverte).

**Pour savoir où en est ce dépôt aujourd'hui** — ce qui est prouvé, ce qui
ne l'est pas encore, ce qui a été écarté et pourquoi — voir
[`docs/status.md`](docs/status.md), une seule page, tenue à jour à la
clôture de chaque série.

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

**Les rôles Ansible que [`site.yml`](site.yml) orchestre**, chacun ciblant
`localhost`, chacun avec son propre `README.md` (« ce que ce rôle fait » /
« ce que ce rôle ne fait jamais ») — table qui liste aussi `roles/desktop/`,
seul rôle du dépôt tenu à l'écart de cet enchaînement :

| Rôle | Fait |
|---|---|
| [`roles/bootstrap/`](roles/bootstrap/) | `power-profiles-daemon`, dépôt `terra` (clé vérifiée hors ligne avant tout usage), règle `sudo` sans mot de passe — les trois préalables jusqu'ici manuels, désormais reconstructibles |
| [`roles/recovery/`](roles/recovery/) | Chemin de retour SSH, préalable à toute bascule GPU risquée |
| [`roles/gpu_cdi/`](roles/gpu_cdi/) | Rend la RTX 4090 utilisable depuis Podman rootless par CDI natif, SELinux `Enforcing` |
| [`roles/gpu_mux/`](roles/gpu_mux/) | Bascule le MUX GPU (panneau câblé dGPU/iGPU) |
| [`roles/local_ai/`](roles/local_ai/) | Infrastructure d'inférence locale : Ollama conteneurisé, réseau confiné |
| [`roles/editor/`](roles/editor/) | Helix (terminal) et Kate (graphique), configurés sobrement |
| [`roles/completion/`](roles/completion/) | `lsp-ai` compilé depuis les sources, câblé à Helix et Kate, même configuration pour les deux |
| [`roles/copilot_cli/`](roles/copilot_cli/) | GitHub Copilot CLI, second agent de code — n'authentifie jamais (geste manuel de l'opérateur) |
| [`roles/android_jdk/`](roles/android_jdk/) | JDK complet pour la chaîne de build Android CLI d'un dépôt distinct (`glass-hud`) |
| [`roles/android_sdk/`](roles/android_sdk/) | Command-line tools du SDK Android dans le domaine utilisateur, licences acceptées — dépend réellement d'`android_jdk`, aucun `platform-tools`/`platform`/`build-tools` |
| [`roles/desktop/`](roles/desktop/) | Kitty positionné sur le ScreenPad Plus à l'ouverture de session — joué par son propre playbook (`roles/desktop/desktop.yml`), jamais par `site.yml` |

**Un point d'entrée unique**, [`site.yml`](site.yml), qui enchaîne les
rôles ci-dessus — à l'exception de `roles/desktop/`, joué séparément —
dans l'ordre de leurs dépendances réelles et s'arrête une seule fois,
avant tout redémarrage — jamais déclenché par le playbook lui-même :

```
ansible-playbook --ask-become-pass site.yml   # première exécution, machine neuve
ansible-playbook site.yml                     # exécutions suivantes
ansible-playbook --check site.yml             # simulation complète
ansible-playbook site.yml --tags gpu_mux      # un seul rôle
```

Détail complet de la séquence, des prérequis hors Ansible (Fedora, pilote
NVIDIA, RPM Fusion), du point d'arrêt et de ce qui a été vérifié ou non :
[`docs/orchestration.md`](docs/orchestration.md).

**Documents de référence**, chacun couvrant une résolution complète, sourcée
et datée :

- [`docs/status.md`](docs/status.md) — état réel du dépôt aujourd'hui, en
  une page, tout en renvoi.
- [`CLAUDE.md`](CLAUDE.md) — règles persistantes pour les sessions d'agent
  travaillant sur ce dépôt, chacune motivée par un incident réel de ce
  dépôt.
- [`docs/machine-facts.md`](docs/machine-facts.md) — inventaire sourcé du
  poste : matériel, système, dépôts, GPU, stockage, affichage, conteneurs,
  chaîne Ansible, des décisions datées et numérotées, et le journal
  complet de chaque série.
- [`docs/orchestration.md`](docs/orchestration.md) — reconstruction
  complète d'un poste neuf, séquence, point d'arrêt, ce qui reste hors
  d'Ansible.
- [`docs/repositories.md`](docs/repositories.md) — surfaces
  d'approvisionnement logicielles activées, ancrage de confiance de
  chacune.
- [`docs/gpu-containers.md`](docs/gpu-containers.md),
  [`docs/gpu-mux-recovery.md`](docs/gpu-mux-recovery.md),
  [`docs/local-ai.md`](docs/local-ai.md),
  [`docs/completion.md`](docs/completion.md),
  [`docs/editor.md`](docs/editor.md),
  [`docs/desktop.md`](docs/desktop.md),
  [`docs/dgpu-power.md`](docs/dgpu-power.md),
  [`docs/ansible-chain.md`](docs/ansible-chain.md) — résolutions
  thématiques complètes, chacune citée depuis `docs/status.md` ou
  `docs/machine-facts.md` au point où elle s'applique.
- [`docs/review-2026-08.md`](docs/review-2026-08.md) — audit global en
  lecture seule de l'ensemble (décisions, rôles, gardes, chaînes de
  valeurs), chaque constat marqué d'un statut (traité, écarté, ou ouvert
  avec sa priorité).
