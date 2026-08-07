# roles/local_ai

Rôle Ansible ciblant `localhost`. Met en place l'infrastructure
d'inférence locale (D14-D17, `docs/machine-facts.md` § Décisions) :
Ollama conteneurisé, image CUDA officielle épinglée par empreinte,
accès GPU par CDI natif déjà prouvé (`docs/gpu-containers.md`), servi
par Quadlet Podman. **Ne télécharge aucun modèle** — seule l'image du
serveur (infrastructure) est tirée ; le choix du modèle et les mesures
sont réservés à un livrable ultérieur (IA-2). Résolution complète,
sources citées et démonstrations sont dans
[`docs/local-ai.md`](../../docs/local-ai.md) § 7 — ce README ne
duplique pas ce contenu.

## Ce que ce rôle fait, dans l'ordre

1. **Trois gardes préalables**, échec bruyant, avant toute écriture :
   spécification CDI présente et non périmée (réutilise
   `roles/gpu_cdi`, tag `verify-cdi-spec` — ne le réimplémente pas) ;
   Podman rootless ; module noyau `nvidia_uvm` chargé.
2. Déploie le paramètre pilote D16
   (`NVreg_PreserveVideoMemoryAllocations=1`) par écrasement propre
   dans `/etc/modprobe.d/local-ai-nvidia-power-management.conf` —
   **jamais** `/usr/lib/modprobe.d/nvidia-power-management.conf`
   (fichier fourni par le paquet `xorg-x11-drv-nvidia-power`, vérifié
   intact avant/après par `rpm -Vf`, à chaque exécution).
3. Déploie un volume Podman nommé (Quadlet `.volume`) pour les modèles
   et une unité Quadlet `.container` pour le service Ollama — image
   épinglée par empreinte, accès GPU par `--device nvidia.com/gpu=all`
   (CDI), API publiée sur `127.0.0.1:11434` uniquement, `keep_alive`
   infini (D15).
4. Active et démarre le service (systemd utilisateur généré par
   Quadlet — jamais une unité `.service` écrite à la main).
5. Vérifie : API répond, liste des modèles vide, API en écoute locale
   uniquement, GPU visible depuis l'intérieur du conteneur (pas
   déduit depuis l'hôte).

## Ce que ce rôle ne fait jamais

- Il ne télécharge **aucun modèle**, sous aucune forme.
- Il ne modifie jamais `/usr/lib/modprobe.d/nvidia-power-management.conf`
  (fichier fourni par un paquet) — écrasement propre uniquement.
- Il ne touche jamais `sudoers`, `gpu_mux_mode`, `dgpu_disable`,
  `kwinrulesrc`.
- Il ne modifie jamais SELinux (pas de `setenforce`, pas de mode
  permissif).
- Il n'ajoute aucun dépôt.
- **Il ne redémarre jamais la machine, ne déconnecte jamais la
  session** — si `NVreg_PreserveVideoMemoryAllocations` exige un
  redémarrage pour prendre effet (c'est le cas, module en usage actif),
  ce rôle le documente et s'arrête, il ne le déclenche pas
  (`docs/local-ai.md` § 7.5).

## Découverte non anticipée, à connaître avant d'activer ce rôle

L'image officielle Ollama contacte `ollama.com` de son propre chef peu
après le démarrage (liste de « recommandations de modèles », pas un
modèle), et expose une fonctionnalité « cloud » activée par défaut
(`OLLAMA_NO_CLOUD:false`). Ni provoqué ni corrigé par ce rôle — nommé
en détail dans `docs/local-ai.md` § 7.3.

## Variables

Voir `defaults/main.yml` — chaque variable y porte son motif en
commentaire, en particulier `local_ai_ollama_image_digest` (empreinte
épinglée, pas une étiquette), `gpu_cdi_verify_spec_path` (nom repris
délibérément de `roles/gpu_cdi`, substituable pour la démonstration
d'échec forcé) et `local_ai_required_kernel_module`/
`local_ai_podman_rootless_expected` (substituables pour les deux
autres démonstrations d'échec forcé).

## Utilisation

```
ansible-playbook --syntax-check roles/local_ai/local_ai.yml
ansible-playbook --check roles/local_ai/local_ai.yml
ansible-playbook roles/local_ai/local_ai.yml                 # écriture réelle

# Démonstrations d'échec forcé (§ docs/local-ai.md § 7.3) :
ansible-playbook roles/local_ai/local_ai.yml \
  -e gpu_cdi_verify_spec_path=/tmp/chemin-invalide.yaml
ansible-playbook roles/local_ai/local_ai.yml \
  -e local_ai_required_kernel_module=module_absent_garanti
```

## Vérification post-redémarrage (D16, non déclenché par ce rôle)

```
cat /proc/driver/nvidia/params | grep -i PreserveVideoMemoryAllocations
# attendu après redémarrage : PreserveVideoMemoryAllocations: 1
```
