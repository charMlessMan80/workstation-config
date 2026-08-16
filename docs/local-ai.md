# Modèle IA local sur la RTX 4090 — résolution en lecture seule

**Ce document ne corrige, n'installe et ne télécharge rien.** Toutes les
commandes citées ci-dessous sont des lectures, exécutées le 2026-08-06,
sur ce poste. Aucun paquet installé, aucun modèle ni aucune image
téléchargé, aucun conteneur lancé, aucun dépôt ajouté, aucun paramètre
de pilote modifié. La décision d'appliquer une option viendra dans un
livrable séparé, après revue de cette résolution.

**Incident de méthode reconnu d'emblée** : `btrfs filesystem usage /`
a été invoqué une première fois avec `sudo`, par réflexe, sans vérifier
d'abord si la lecture fonctionnait sans privilège — elle fonctionne
(§ 5). Reconnu et corrigé dans la même série, avant d'aller plus loin ;
aucune autre commande de ce livrable n'a demandé de privilège.

**Faits déjà établis, non revérifiés ici** (cités tels quels) : RTX 4090
Laptop, 16376 MiB de VRAM, plafond 155 W, CUDA UMD 13.3, pilote
610.43.03 (`rpmfusion-nonfree-updates`) ; MUX Optimus/Hybrid (Plasma sur
l'iGPU AMD, dGPU réservée au calcul, `kwin_wayland` ~13 MiB résiduels) ;
dGPU en veille runtime D3 fine-grained 99,97 % du temps
(`docs/dgpu-power.md`), réveil sur appel, retour au repos en 20-23 s ;
CDI natif fonctionnel de bout en bout (`docs/gpu-containers.md`,
`/etc/cdi/nvidia.yaml`, 21 chemins, nœuds UVM présents), détection de
péremption sans régénération automatique ; Podman 5.8.4 rootless, crun,
SELinux `Enforcing`, aucun refus AVC ; btrfs `Data,single` sur deux
NVMe (D1) ; D11(D12) npm fermé ; D12/D13 éditeurs ; D4 dépôt public ;
D7/D10 ancrages de confiance des dépôts tiers.

## 1. La tension entre les trois usages — ce qui est réellement atteignable

Trois usages décrits par l'opérateur, trois profils de contrainte
différents sur les mêmes 16376 MiB **partagés avec le bureau** (§ 2.3 :
enveloppe réellement mesurée ≈ 15,9 GiB, pas 16 GiB nominaux).

**Méthode de dimensionnement, sourcée, pas mémorisée** : la taille d'un
modèle GGUF quantifié se calcule à partir des bits par poids documentés
par le projet en amont (`llama.cpp`, tableau de quantification,
`tools/quantize/README.md`, consulté le 2026-08-06) — point d'étalonnage
vérifié : Llama-3.1-8B, `Q4_K_M` = 4,58 GiB (≈4,89 bits/poids), `F16` =
14,96 GiB (16 bits/poids, non quantifié). Extrapolé linéairement (poids
seuls, **hors cache KV**, qui s'ajoute et croît avec la longueur de
contexte et le nombre de requêtes concurrentes — non chiffré ici faute
d'un modèle réellement chargé pour le mesurer) :

| Taille du modèle | `Q4_K_M` (≈4,89 bit/poids) | Tient dans ≈15,9 GiB mesurés ? |
|---|---|---|
| 7 B | ≈4,0 GiB | Oui, large marge |
| 8 B (étalon) | 4,58 GiB (mesuré en amont) | Oui, large marge |
| 13-14 B | ≈7,4-8,0 GiB | Oui |
| 20 B | ≈11,5 GiB | Oui, marge réduite (cache KV à soustraire) |
| 32 B | ≈18,3 GiB | **Non — dépasse la capacité totale de la carte, cache KV non compté** |
| 70 B | ≈40 GiB | Non, hors de portée par un ordre de grandeur |

**Complétion de code (éditeur)** — latence basse, petit modèle
**toujours chaud**. Atteignable : un modèle 1,5-7 B tient large dans
l'enveloppe mesurée, coûte peu de VRAM, laisse de la marge pour autre
chose en parallèle (§ ci-dessous). **Condition non négociable, établie
par les mécanismes RTD3 déjà documentés** (`docs/dgpu-power.md`) : le
serveur doit rester un **processus persistant**, jamais relancé par
requête — RTD3 ne suspend la carte que lorsque plus aucune application
CUDA n'y maintient de contexte ; un serveur relancé à chaque complétion
paierait le réveil de 20-23 s **à chaque frappe**, inutilisable. Un
serveur qui reste démarré ne paie ce coût qu'une fois, au premier appel
après un arrêt (§ 2.2). **Servable**, à cette condition précise.

**Agent autonome (lecture/écriture de fichiers, contexte long)** —
qualité de raisonnement et fenêtre de contexte, latence secondaire.
Ce que les chiffres soutiennent : un modèle jusqu'à ≈13-20 B tient dans
l'enveloppe mesurée avec du contexte raisonnable ; **32 B et au-delà ne
tiennent pas du tout sur cette carte**, cache KV non compté. Un modèle
local de cette taille **n'égale pas** la qualité de raisonnement d'un
modèle frontière (Claude Code, déjà disponible sur ce poste) sur des
tâches agentiques complexes — ne pas le promettre. **Servable pour des
tâches simples, hors-ligne ou pour économiser des jetons sur du travail
répétitif/mécanique** ; **régression probable** face à Claude Code sur
tout ce qui exige du raisonnement long ou une base de code large — à
dire explicitement plutôt que de laisser un compromis décevoir sur les
deux fronts.

**Chat** — entre les deux. Un 7-13 B à `Q4_K_M`/`Q5_K_M` convient pour
un usage conversationnel courant ; qualité en dessous d'un modèle
frontière sur des questions techniques pointues, correcte pour du
brouillon, de la reformulation, du hors-ligne.

**Coexistence** — arithmétique, pas mesurée : un petit modèle de
complétion (≈4 GiB) tenu en permanence **plus** un modèle de
chat/agent de 13 B (≈7,4 GiB) chargé à la demande totalisent ≈11,4 GiB,
sous l'enveloppe mesurée de ≈15,9 GiB, marge de cache KV et de base du
bureau (≈0,45 GiB, § 2.3) incluse. **Plausible arithmétiquement,
non vérifié en pratique** — aucun modèle n'a été chargé dans cette
série. **[RÉSOLU le 2026-08-07, IA-3, § 9.3] ÉTAIT** un marqueur de
vérification ouvert (chargement simultané non mesuré) — **mesuré
réellement** avec les deux modèles finalement retenus (D21) :
la coexistence **ne tient pas**, Ollama décharge le premier modèle
pour charger le second dès que les deux sont sollicités l'un après
l'autre — résultat mesuré, pas déduit de l'arithmétique. Détail
complet, § 9.3.

**Où le modèle local apporte quelque chose de réel, face à Claude Code
déjà disponible** : disponibilité hors-ligne, coût marginal nul par
requête (pas de jetons consommés), latence de complétion locale une
fois le serveur chaud, confidentialité (rien ne quitte la machine).
**Où il serait une régression** : qualité de raisonnement sur des
tâches agentiques complexes, largeur de la base de connaissances,
intégration d'outils déjà en place avec Claude Code — ne pas chercher à
le remplacer sur ces usages.

## 2. Quatre points ouverts

### 2.1 — VRAM et veille : documentable sans charger de modèle, et ce qui ne l'est pas

**Indice exploité, comme demandé** : la carte se suspend alors même que
`kwin_wayland` détient 13 MiB — établi dans `docs/dgpu-power.md`,
confirmé de nouveau ici (§ 2.3). Ce fait **exclut** l'hypothèse « le
GPU refuse de se suspendre tant qu'une allocation VRAM existe, quelle
qu'elle soit » : une allocation seule, sans processus qui interroge
activement le pilote, ne bloque pas RTD3. Ce qui bloque RTD3, documenté
sans ambiguïté (`dynamicpowermanagement.html`, déjà cité dans
`docs/dgpu-power.md`, re-consulté ici) : *« the NVIDIA GPU will remain
in an active state ... if a CUDA application is running »* — un
**contexte CUDA ouvert**, pas une simple allocation mémoire résiduelle
comme celle de `kwin_wayland` (qui n'ouvre pas de contexte de calcul).

**Conséquence pour un serveur d'inférence** : tant qu'il tourne et
garde un contexte CUDA ouvert (le cas normal d'un serveur qui répond à
des requêtes), **RTD3 ne s'engage pas** — la carte reste active, la
VRAM allouée au modèle n'est jamais menacée par une suspension runtime
per-périphérique. RTD3 ne redevient pertinent qu'**entre deux
exécutions du serveur** (serveur arrêté, aucun contexte CUDA nulle
part) — à ce moment, `Video Memory: Off` est **attendu et sans
conséquence**, puisqu'il n'y a justement plus rien à préserver (le
serveur, arrêté, ne tient plus de modèle en mémoire).

**Ce qui reste réellement documentable sans charger de modèle, établi
ici, pas déduit** : le risque documenté par NVIDIA pour
`NVreg_PreserveVideoMemoryAllocations` **ne concerne pas RTD3** —
confirmé par une lecture locale supplémentaire (source amont, déjà
partiellement citée dans `docs/dgpu-power.md`, approfondie ici) :
```
$ grep -B2 -A2 -i PreserveVideoMemoryAllocations /usr/share/doc/xorg-x11-drv-nvidia/html/powermanagement.html
```
Le paramètre gouverne la préservation de la VRAM à la **suspension
système** (S3 classique, ou S0ix — voir plus bas), pas les cycles RTD3
per-périphérique. Sans lui (`NVreg_PreserveVideoMemoryAllocations`
absent, donc `0` par défaut), documentation amont citée : *« the
resulting loss of video memory contents is partially compensated for by
the user-space NVIDIA drivers, and by some applications, but can lead
to failures such as rendering corruption and application crashes upon
exit from power management cycles »* — un risque **réel et documenté**,
mais seulement pour une **vraie suspension système** (fermeture du
capot, mise en veille explicite), jamais entre deux requêtes d'un
serveur qui reste démarré.

**Complément établi dans ce livrable, absent de `docs/dgpu-power.md`**
(cette machine utilise le suspend moderne, pas S3 classique — établi
par lecture locale) :
```
$ cat /sys/power/mem_sleep
[s2idle]
```
Sur ce mode, un mécanisme **distinct** s'applique en théorie —
« S0ix-based power management », même documentation amont, chapitre
« Preserving video memory allocations ». Support matériel confirmé
(lecture procfs, passive — voir méthode d'isolement § 2.3) :
```
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
GPU Hardware Support:
 Video Memory Self Refresh: Supported
 Video Memory Off:          Supported
S0ix Power Management:
 Platform Support:          Supported
 Status:                    Disabled
```
**Support matériel et plateforme présents des deux côtés, mais le
mécanisme n'est pas activé sur ce poste** (`Status: Disabled`) —
confirmé par la valeur effectivement chargée par le pilote :
```
$ cat /proc/driver/nvidia/params | grep -i s0ix
EnableS0ixPowerManagement: 0
S0ixPowerManagementVideoMemoryThreshold: 256
```
`NVreg_EnableS0ixPowerManagement` (paramètre distinct de
`NVreg_PreserveVideoMemoryAllocations`) est absent de la configuration
et vaut `0` par défaut — **non activé**, donc sans effet actuellement.
Le seuil par défaut documenté (256 MiB) serait de toute façon largement
dépassé par n'importe quel modèle de plusieurs gigaoctets, plaçant le
comportement dans la branche « self-refresh » documentée plutôt que
« copie et coupure » — mais **sans objet tant que le mécanisme reste
désactivé**.

**Mécanisme d'intégration déjà en place, découverte non anticipée par
la demande** : les unités systemd que `NVreg_PreserveVideoMemoryAllocations=1`
exige (interface `/proc/driver/nvidia/suspend`, documentation amont,
« systemd Configuration ») sont **déjà présentes et activées** sur ce
poste :
```
$ systemctl list-unit-files | grep -i nvidia
nvidia-hibernate.service                     enabled enabled
nvidia-resume.service                        enabled enabled
nvidia-suspend.service                       enabled enabled
nvidia-suspend-then-hibernate.service        enabled enabled
```
**Ce qui est documentable établi** : l'intégration systemd nécessaire à
`NVreg_PreserveVideoMemoryAllocations=1` est déjà en place — activer ce
paramètre ne demanderait pas de câblage systemd supplémentaire,
contrairement à ce que la documentation amont pourrait laisser
craindre pour un système qui partirait de zéro. **Ce qui exigerait une
mesure réelle, pas documentable ici** : `@VERIF : effet réel d'une
suspension système (S0ix/s2idle) avec un modèle de plusieurs
gigaoctets chargé en VRAM et NVreg_PreserveVideoMemoryAllocations
laissé à sa valeur d'alors — **[CORRIGÉ le 2026-08-07, § 2.1bis]**
« absent/0 » supposé ici sans lecture directe de la valeur brute
effective ; la lecture directe montre `2`, pas `0`, signification non
documentée — voir § 2.1bis. Ce qui reste vrai indépendamment de cette
correction : la carte se suspendrait-elle avec le contenu perdu
(comportement décrit ci-dessus, risque de corruption/plantage au
réveil) ou le serveur d'inférence bloquerait-il la suspension système
lui-même (comportement distinct de RTD3, non documenté pour S0ix dans
les pages consultées) ? Non mesurable sans charger un modèle réel et
déclencher une vraie suspension système — hors périmètre lecture seule
de ce livrable.` **[RÉSOLU en pratique le 2026-08-07, D16]** — voir
§ 2.1bis et § 7 : l'opérateur a tranché en amont de toute mesure
(contrepartie assumée), ce livrable déploie `=1` sans attendre cette
mesure.

### 2.1bis — Correction (IA-1, 2026-08-07) : la valeur brute par défaut n'est pas 0

**Défaut d'IA-0 reconnu, pas seulement corrigé en silence** (CLAUDE.md
§ un fait mesuré porte sa date, une mesure ancienne ne sert pas de
ligne de base sans revérification) : IA-0 raisonnait sur la
conséquence documentée de l'absence du paramètre (perte de contenu
VRAM) sans jamais lire la valeur entière brute réellement chargée par
le pilote pour `NVreg_PreserveVideoMemoryAllocations` spécifiquement —
seul `NVreg_DynamicPowerManagement` (`3`) et
`NVreg_EnableS0ixPowerManagement` (`0`) avaient été lus directement.
Corrigé ici, lecture directe, avant toute écriture de ce livrable :
```
$ cat /proc/driver/nvidia/params | grep -i PreserveVideoMemory
PreserveVideoMemoryAllocations: 2
```
**Ni `0` ni `1`.** Recherche exhaustive de la signification de cette
valeur `2` — `/usr/share/doc/xorg-x11-drv-nvidia/html/powermanagement.html`
(déjà consulté en IA-0) et `/usr/share/doc/xorg-x11-drv-nvidia/README.txt`
(source amont plus complète, non consultée en IA-0) : les deux ne
documentent que la valeur `=1` (« changes the video memory save/restore
strategy to save and restore all video memory allocations ») — **aucune
des deux sources ne documente ce que vaut `0`, `2`, ou toute autre
valeur non nulle différente de `1`.** `@VERIF : signification exacte de
la valeur brute par défaut 2 de NVreg_PreserveVideoMemoryAllocations —
non documentée dans les sources consultées (page HTML, README.txt
complet du pilote 610.43.03). Hypothèse plausible mais non confirmée :
un mode « automatique », par analogie avec NVreg_DynamicPowerManagement
(dont le défaut documenté, 3, signifie explicitement « fin si
supporté, sinon grossier ») — non vérifiée, présentée ici comme
hypothèse, pas comme fait.`

**Conséquence concrète, celle-ci établie sans ambiguïté** : régler
`NVreg_PreserveVideoMemoryAllocations=1` (D16, § 7) change réellement
la valeur chargée (de `2` à `1`), ce n'est **pas** une explicitation
sans effet d'un défaut déjà équivalent (contrairement à l'option 2
écartée dans `docs/dgpu-power.md` pour `NVreg_DynamicPowerManagement`,
où `0x02` explicite et le défaut `0x03` produisaient le même
comportement documenté sur ce matériel). Ici, `=1` déclenche
spécifiquement le comportement documenté sans ambiguïté (sauvegarde et
restauration de toutes les allocations vidéo) — un comportement dont
rien ne garantit qu'il soit déjà celui de la valeur `2` non documentée.

**Complément trouvé dans le README.txt complet, absent d'IA-0** : le
mécanisme systemd (`/proc/driver/nvidia/suspend`) qu'exige
`NVreg_PreserveVideoMemoryAllocations=1` est également requis, dit le
texte amont, « **or advanced CUDA features (such as UVM)** » — pas
seulement pour ce paramètre. Nuance apportée à la lecture d'IA-0 (qui
présentait les unités `nvidia-suspend`/`nvidia-resume` déjà activées
comme si elles anticipaient spécifiquement ce paramètre) : leur
activation s'explique tout aussi bien, sinon mieux, par l'usage déjà
actif d'UVM sur ce poste (CDI, `docs/gpu-containers.md`) — sans
changer la conclusion pratique (le câblage nécessaire est déjà là),
seulement sa cause la plus probable.

### 2.2 — Latence de réveil, reliée à chaque usage

20-23 s, mesurés deux fois indépendamment dans `docs/dgpu-power.md`,
**réveil sur le premier appel qui interroge activement le pilote**
après une période sans aucun contexte CUDA ouvert — pas à chaque
requête d'un serveur qui reste démarré (§ 2.1).

`NVreg_DynamicPowerManagement` non défini dans la configuration, valeur
effectivement chargée déjà établie dans `docs/dgpu-power.md` : `3`
(fine-grained sur cette architecture Ada Lovelace, confirmé par
`/proc/driver/nvidia/gpus/.../power` → `Runtime D3 status: Enabled
(fine-grained)`). Ce paramètre **n'offre pas de levier pour arbitrer**
la latence de réveil elle-même — il choisit *si* RTD3 s'engage
finement ou grossièrement, pas *combien de temps* dure le réveil une
fois qu'il s'engage. Aucun paramètre `NVreg_*` documenté dans les pages
consultées ne configure directement la durée du réveil RTD3 lui-même
(mécanisme PCIe/ACPI, pas un délai logiciel réglable côté pilote).

**Coût par usage** :
- **Complétion** : sensible au dixième de seconde — 20-23 s est
  **rédhibitoire** si payé par requête. Seule architecture viable :
  serveur persistant démarré une fois (à l'ouverture de session,
  comme `kitty` en BUR-1), payant le réveil **une seule fois**, puis
  restant chaud tant qu'il tourne (§ 2.1).
- **Chat** : payable une fois par session de travail si le serveur
  démarre à la demande plutôt qu'en permanence — un utilisateur qui
  ouvre un chat attend rarement moins de quelques secondes de toute
  façon pour la première réponse.
- **Agent** : le moins sensible des trois — une tâche agentique dure
  déjà de plusieurs secondes à plusieurs minutes ; 20-23 s au tout
  premier appel est un coût marginal, pas structurant.

### 2.3 — Enveloppe réelle, mesurée maintenant, méthode d'isolement appliquée

`nvidia-smi` réveille la carte — établi et vérifié à nouveau, sourcé
sur cette même série (pas seulement rappelé de `docs/dgpu-power.md`) :
```
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status   # avant
suspended
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power | head -3    # passif, ne réveille pas — vérifié ci-dessous
Runtime D3 status: Enabled (fine-grained)
Video Memory:      Off
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status   # immédiatement après la lecture procfs ci-dessus
suspended   # inchangé — confirme que CETTE lecture procfs précise est passive
```
**Une seule sonde délibérée, isolée**, pour obtenir un chiffre de VRAM
actuel (la lecture procfs passive ne donne que `Off`/`Active`, pas un
volume en MiB) :
```
$ date -Iseconds ; cat .../runtime_status ; cat .../runtime_suspended_time   # T0, sysfs pur
2026-08-06T23:22:33+02:00  suspended  79193267
$ nvidia-smi --query-gpu=timestamp,memory.total,memory.used,memory.free,power.draw,pstate --format=csv
2026/08/06 23:22:35.512, 16376 MiB, 47 MiB, 15926 MiB, 32.57 W, P0
$ nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
2582, /usr/bin/kwin_wayland, 13 MiB
```
**Réveil provoqué par cette sonde, écarté du résultat comme suit** :
seule cette commande unique a été exécutée entre T0 (suspendu) et la
lecture — le réveil qu'elle cause n'affecte pas le chiffre `memory.used`
lui-même (la VRAM allouée par les clients existants, ici `kwin_wayland`,
ne change pas parce que la carte se réveille pour répondre à la
requête) ; **aucune deuxième sonde rapprochée** n'a été faite pour ne
pas prolonger artificiellement l'état actif (leçon de méthode de
`docs/dgpu-power.md`) — la carte redevient suspendue seule, dans les
20-23 s déjà mesurées, sans nouvelle vérification ici (déjà établi,
pas re-mesuré pour ne pas re-provoquer le réveil sans raison).

**Enveloppe effectivement disponible, mesurée** : **16376 MiB total,
47 MiB utilisés (13 MiB `kwin_wayland`, ≈34 MiB non attribués à un
processus — recouvrement pilote/contexte non détaillé par
`nvidia-smi`), 15926 MiB rapportés libres**. Écart entre
`16376 - 47 = 16329` et les `15926` rapportés libres (~403 MiB) :
réserve interne du pilote/firmware non attribuée à un processus ni
comptée comme libre — non creusé davantage dans ce livrable, sans
conséquence sur la conclusion (l'enveloppe **utilisable** est bien le
chiffre `memory.free` rapporté, pas `total - used`). **Enveloppe
retenue pour le dimensionnement du § 1 : ≈15,9 GiB**, pas 16 GiB
nominaux — écart de ≈450 MiB (≈2,7 %) entre le nominal et le mesuré,
stable et attribuable au poste de bureau (`kwin_wayland`), déjà
identifié à travers plusieurs livrables comme constant (13 MiB).

### 2.4 — Péremption CDI : ce qui devrait déclencher la vérification existante

Mécanisme déjà écrit (`roles/gpu_cdi/tasks/check_spec.yml`,
`verify_spec.yml`) : compare la version du pilote encodée dans
`/etc/cdi/nvidia.yaml` (suffixe de `libcuda.so.<version>`) à la version
réellement chargée (`/proc/driver/nvidia/version`), et vérifie que
chaque `hostPath` référencé existe encore. Rejoué en lecture seule dans
cette série, sans écriture, sans `become` :
```
$ ansible-playbook roles/gpu_cdi/gpu_cdi.yml --tags verify-cdi-spec
"Spécification à jour : version 610.43.03 == version chargée 610.43.03,
 50 chemins de bibliothèques référencés, tous présents sur l'hôte."
$ echo $?
0
```
**À jour actuellement.** L'assertion `ansible.builtin.assert` échoue
(code de sortie non nul) dès que ce n'est plus le cas — exploitable
tel quel par n'importe quel mécanisme de déclenchement externe.

**Rien ne la déclenche aujourd'hui** — vérifié par l'absence de toute
minuterie ou service qui l'invoquerait :
```
$ systemctl list-timers 2>&1 | grep -i cdi     # aucune correspondance
$ systemctl list-unit-files 2>&1 | grep -i cdi # aucune correspondance
```
**Ce dont un service de longue durée devrait se prémunir, établi par
lecture des mécanismes systemd déjà standard sur ce poste (pas un
nouveau mécanisme inventé)** — deux leviers distincts, combinables :

1. **`ExecStartPre=`** sur l'unité systemd du serveur d'inférence
   lui-même (native ou wrapper autour du conteneur) :
   `ExecStartPre=/usr/bin/ansible-playbook <chemin>/gpu_cdi.yml --tags
   verify-cdi-spec`. Un `ExecStartPre` qui échoue **empêche
   `ExecStart` de démarrer** (comportement systemd documenté,
   standard) — le service refuserait de démarrer avec une
   spécification périmée plutôt que de démarrer avec un GPU
   silencieusement invisible (le risque exact nommé par
   `check_spec.yml`). Coût : dépendance à `ansible-playbook` disponible
   au moment du démarrage du service (déjà le cas sur ce poste, D3a).
2. **Minuterie systemd périodique** (`.timer` + `.service` dédié,
   même patron que n'importe quelle autre unité de ce dépôt) qui
   rejoue `--tags verify-cdi-spec` à intervalle régulier et, sur
   échec, notifie plutôt que de rester silencieuse (`OnFailure=` vers
   une unité de notification, ou simplement un journal consulté).
   Complémentaire au levier 1 : couvre le cas où le pilote est mis à
   jour **pendant que le serveur tourne déjà** (le serveur démarré
   avant la mise à jour ne relirait pas `ExecStartPre` avant son
   prochain redémarrage).

**Non implémenté ici** (résolution en lecture seule) — nommé pour un
livrable ultérieur qui aurait le mandat d'écrire une unité systemd.

## 3. Ce qui existe dans les dépôts activés (`docs/repositories.md` § 4)

**Méthode** : réutilise l'extraction de métadonnées déjà faite pour
EDI-0 le même jour (`dnf repoquery --available`, dix dépôts hors Terra
et Terra séparément, gardée par `--assumeno`) — pas re-téléchargée,
recherche ciblée sur les noms/résumés pertinents à l'inférence locale.
Complétée par des vérifications directes sur les paquets trouvés.

### 3.1 — Serveurs d'inférence trouvés, et leur limite décisive

| Paquet | Dépôt | Ce qu'il fournit |
|---|---|---|
| `ollama` | `fedora`/`updates` | Serveur d'inférence, gestion de modèles, API compatible OpenAI |
| `llama-cpp` (+`-devel`) | `fedora`/`updates` | `llama-server` (API compatible OpenAI, `/v1/completions` et `/v1/chat/completions`), `llama-cli`, `llama-quantize`, etc. |
| `whisper-cpp` (+`-devel`) | `fedora`/`updates` | Reconnaissance vocale (Whisper) — hors sujet direct (pas un LLM), mentionné pour exhaustivité |
| `python3-torch` (+`torchvision`/`torchaudio`/…) | `fedora`/`updates` | Bibliothèque bas niveau, pas un serveur — nécessiterait d'écrire soi-même l'intégration |

**Aucune correspondance sur `terra`** pour ces mêmes recherches
(`inference`, `llm`, `gguf`, `cuda`, noms directs) — un seul résultat
tangentiel, `ensu` (« Private, personal LLM app »), **sans dépendance
CUDA/ROCm/Vulkan déclarée** (vérifié, `dnf repoquery --requires`) —
nature exacte de son moteur d'inférence non creusée davantage, hors
sujet probable pour un GPU dédié.

**Limite décisive, établie par lecture des dépendances déclarées, pas
supposée** — **aucun des trois candidats sérieux ci-dessus ne peut
utiliser CUDA** :
```
$ dnf repoquery --requires llama-cpp | grep -iE 'cuda|rocm|hip'
hipblas
libamdhip64.so.7()(64bit)
libhipblas.so.3()(64bit)
rocblas
$ dnf repoquery --requires ollama | grep -iE 'cuda|rocm|hip'
hipblas
libamdhip64.so.7()(64bit)  libhipblas.so.3()(64bit)  librocblas.so.5()(64bit)
$ dnf repoquery --requires python3-torch | grep -iE 'cuda|rocm|hip'
libamdhip64.so.7()(64bit)  libhipblas.so.3()(64bit)  libMIOpen.so.1(...)  librocm_smi64.so.1()(64bit)  [...]
```
**Aucune des trois sorties ne contient `cuda`.** Les trois paquets
dépendent **en dur** de bibliothèques ROCm/HIP (AMD) — `hipblas`,
`rocblas`, `libamdhip64`. Ni option, ni sous-paquet CUDA alternatif
(`dnf repoquery llama-cpp\* ollama\*` ne montre que les paquets
`-devel`, pas de variante `-cuda`). **Ce n'est pas un hasard isolé** :
`python3-torch`, bibliothèque distincte, indépendante, montre
exactement le même schéma — Fedora construit sa pile IA officielle
pour ROCm, pas CUDA, cohérent avec le fait que le kit CUDA propriétaire
n'est pas disponible dans l'environnement de construction de Fedora
(dépendance non redistribuable, même famille de contrainte que D7/D10
sur les dépôts eux-mêmes, appliquée cette fois à la chaîne de
compilation plutôt qu'au dépôt).

**Conséquence directe pour ce poste** : le GPU pertinent ici est une
**RTX 4090 NVIDIA** — ROCm ne pilote jamais de matériel NVIDIA, quel
que soit l'état de l'iGPU AMD. **Installer `ollama`, `llama-cpp` ou
`python3-torch` depuis les dépôts activés (`docs/repositories.md` § 4)
ne donnerait accès à la RTX 4090 sous aucune forme** — au mieux un repli CPU (non vérifié ici,
aucun de ces paquets n'a été installé), jamais un chemin GPU vers cette
carte. `@VERIF : comportement runtime exact de llama-cpp/ollama
empaquetés par Fedora en l'absence d'un périphérique ROCm valide —
repli CPU silencieux, ou échec au démarrage ? Non testé, nécessiterait
d'installer le paquet, hors périmètre lecture seule.`

### 3.2 — Ce qui existe seulement hors des onze dépôts

**npm fermé par D11(D12)** — tout serveur de langage ou outillage
distribué exclusivement par npm est d'emblée exclu, cohérent avec
`docs/editor.md`.

**Conteneurs, provenance externe, jamais téléchargés dans cette série**
— existence confirmée par inspection de métadonnées de registre
(`skopeo inspect`, lit le manifeste distant, **ne télécharge aucune
couche d'image** — vérifié : `podman images` identique avant/après
chaque appel) :

| Image | Registre | Ce qu'elle apporte |
|---|---|---|
| `docker.io/ollama/ollama:latest` | Docker Hub | Ollama avec CUDA (défaut) — nécessite `--gpus=all` (Docker) ou l'équivalent CDI déjà prouvé (`--device nvidia.com/gpu=all`, `docs/gpu-containers.md`) ; tag `:rocm` séparé pour AMD |
| `ghcr.io/ggml-org/llama.cpp:server-cuda` | GitHub Container Registry | `llama-server` du projet amont, construit avec CUDA — même logiciel que le paquet Fedora, backend différent |
| `docker.io/vllm/vllm-openai` | Docker Hub | Serveur haut débit (vLLM), tags `cuXXX` pour la version CUDA — orienté production multi-utilisateurs, VRAM de gestion de lot plus lourde que les deux précédents |

Chacune est une **nouvelle surface d'approvisionnement**, au même titre
qu'un magasin d'extensions d'éditeur (`docs/editor.md` § 1.3) : registre
de conteneurs externe (Docker Hub, GHCR), pas un dépôt `dnf`, donc
**hors du périmètre déjà couvert par `docs/repositories.md`**. Question
d'ancrage de confiance de même nature que D7/D10, mais sur un registre
d'images plutôt qu'un dépôt de paquets — non résolue ici (aucune image
téléchargée), à traiter si un livrable ultérieur retient cette voie.

**Ce qu'elles exigent de télécharger après installation** : les poids
du modèle lui-même, systématiquement — aucune des trois images ne les
embarque (vérifié par la taille modeste du manifeste inspecté,
cohérent avec une image d'exécutable, pas de poids). Ollama ajoute son
propre mécanisme de récupération (`ollama pull`, registre de modèles
adressé par le contenu — voir § 5) ; `llama-server`/`vLLM` chargent un
fichier GGUF/safetensors déjà présent sur le disque, à obtenir
séparément (Hugging Face typiquement) — encore une surface
d'approvisionnement distincte à nommer, pas seulement l'image
elle-même.

**API et formats, établis par lecture de la documentation amont** :
- `llama-server` : `/v1/completions` et `/v1/chat/completions`
  (compatible OpenAI), charge des fichiers `.gguf` directement,
  parallélisme configurable (`--parallel`), mode routeur multi-modèles
  (`--models-dir`, `--models-max`, 4 par défaut).
- `ollama` : API compatible OpenAI sur `/v1` (`/v1/chat/completions`
  confirmé ; pas de point d'entrée dédié à la complétion de code style
  FIM documenté dans la page consultée), déchargement automatique des
  modèles après **5 minutes d'inactivité par défaut**
  (`keep_alive`/`OLLAMA_KEEP_ALIVE`, réglable à `-1` pour un maintien
  indéfini), plusieurs modèles chargés simultanément si la VRAM le
  permet (`OLLAMA_MAX_LOADED_MODELS`, défaut *3× le nombre de GPU*).
  **Directement pertinent pour le § 1** : le comportement par défaut
  (déchargement à 5 min) **contredit** l'exigence « serveur de
  complétion toujours chaud » — `keep_alive=-1` explicite serait
  nécessaire pour le modèle de complétion, faute de quoi chaque
  réveil après une pause de plus de 5 minutes repaierait à la fois le
  rechargement du modèle et, si aucun autre contexte CUDA ne restait
  ouvert entre-temps, le réveil RTD3 (§ 2.2).

## 4. Conteneur ou natif

**Ne pas présupposer le conteneur** — le travail CDI déjà fait
(`roles/gpu_cdi/`) est un coût irrécupérable, pas un argument. Établi
ci-dessus (§ 3.1) : la voie **native depuis les onze dépôts est fermée
pour CUDA**, quel que soit le choix conteneur/natif — ce n'est donc pas
un argument en faveur du conteneur non plus, c'est un fait antérieur
au choix. La vraie question est : **binaire natif hors dépôt** contre
**conteneur hors dépôt** — les deux sortent des onze dépôts, pour la
même raison (aucun n'y offre CUDA).

| Critère | Binaire natif téléchargé (ex. script d'installation `ollama`, ou compilation locale de `llama.cpp` avec CUDA) | Conteneur (image CUDA externe, CDI déjà prouvé) |
|---|---|---|
| Isolation | Aucune — s'exécute directement avec les droits de l'utilisateur, accès complet au reste de `$HOME` | Espace de noms de montage/PID séparé (Podman rootless) — déjà mesuré sans effet négatif sur SELinux (`Enforcing`, aucun refus AVC, `docs/gpu-containers.md` § 8.6) |
| SELinux | Hérite du contexte de session normal, aucune étiquette dédiée | Étiquettes conteneur standard, déjà validées avec CDI sur ce poste |
| Persistance des modèles | Répertoire choisi par l'opérateur, non versionné (D4/D1, § 5) | Volume monté (bind ou nommé), même contrainte D1/D4 — pas de différence structurelle sur ce point précis |
| Démarrage à l'ouverture de session | Unité systemd `--user` (même patron que `kitty`, BUR-1) ou entrée autostart | Unité systemd `--user` lançant `podman run`/`podman-compose`, ou `podman generate systemd` (fonctionnalité native déjà identifiée en BUR-0/EDI-0 comme « ce que l'outil fait déjà ») |
| Interaction avec la veille RTD3 | Identique — RTD3 réagit au contexte CUDA, pas à la conteneurisation (§ 2.1, déjà établi dans `docs/gpu-containers.md` § 5, mesuré sans conteneur réel mais avec le même mécanisme) | Identique, **et déjà mesuré avec un vrai lancement de conteneur** (`docs/gpu-containers.md` § 8.7 : le lancement seul, sans charge GPU active, ne réveille pas la carte) |
| Reconstructibilité D1 | Dépend d'un script d'installation tiers, version non figée sauf à l'épingler soi-même ; pas de mécanisme de vérification d'intégrité standard au-delà de ce que le script offre | Image identifiée par tag **et par empreinte** (`skopeo inspect` donne un `Digest` sha256 vérifiable) — un `Containerfile`/une commande `podman pull <image>@sha256:...` versionnée dans le dépôt reconstruit l'environnement de façon déterministe, seul le contenu du modèle reste hors dépôt (§ 5) |
| Mise à jour du pilote (péremption) | Pas concerné par la péremption CDI (accès direct aux bibliothèques système, pas de spécification à régénérer) — mais alors dépend du câblage `ldconfig`/chemins système, jamais testé ici | Concerné par la péremption CDI (§ 2.4) — mécanisme de détection déjà écrit, mais rien ne le déclenche encore |

**Ce que chaque voie risque de casser** : le binaire natif, en cas de
mise à jour de bibliothèques système (CUDA UMD, glibc) sans mécanisme
de détection dédié, échouerait probablement de façon plus visible
(erreur de lien dynamique au démarrage) que le conteneur, qui
échouerait plus silencieusement (GPU invisible, repli CPU sans erreur —
exactement le risque documenté par `check_spec.yml`).

**Comment on saurait que chacune fonctionne** : les deux se vérifient
identiquement — `nvidia-smi --query-compute-apps` (sonde isolée)
montrant le processus serveur comme client GPU, et une requête de test
retournant une réponse cohérente avec un débit de génération
supérieur à un repli CPU observable. Aucune des deux n'a d'avantage
structurel sur *comment vérifier* — la différence est dans le risque
de silence en cas de dérive (tableau ci-dessus), pas dans la
vérifiabilité elle-même.

**Conclusion de cette section** : le conteneur n'est pas retenu parce
que le travail CDI a déjà été fait — il est retenu (§ 6) parce que
l'empreinte vérifiable du tag d'image et le mécanisme de détection de
péremption déjà écrit, une fois câblé (§ 2.4), donnent une
reconstructibilité et une détection de dérive que le binaire natif
téléchargé n'offre pas nativement.

## 5. Stockage des modèles et D1

**Où les poids devraient résider** : hors du dépôt Git lui-même (D4 —
plusieurs gigaoctets, taille seule suffit à l'exclure, indépendamment
de tout autre motif) et hors de `~/.venvs`/`~/.config` (ni un
environnement Python, ni une configuration). Un répertoire dédié sous
`$HOME` (ex. `~/.local/share/models/` ou équivalent — nom exact laissé
à un livrable d'application), sur le btrfs `Data,single` déjà en place
(D1) — vérifié dans cette série, **sans besoin de privilège** (défaut
corrigé, voir tête de document) :
```
$ df -h /home
/dev/nvme1n1p3  3.8T  8.5G  3.8T   1% /home
$ btrfs filesystem usage /
Data,single: Size:10.01GiB, Used:7.79GiB (77.82%)
```
**3,8 To disponibles** sur le point de montage qui couvre `$HOME` —
aucune contrainte de place pour des poids de quelques gigaoctets à
quelques dizaines de gigaoctets, quel que soit le modèle retenu.

**Comment ils sont re-téléchargeables** : jamais versionnés dans ce
dépôt (D4) ; ce qui EST versionnable et doit l'être pour que la machine
reste reconstructible (D1) — le **nom exact et la source** de chaque
modèle retenu (dépôt Hugging Face, nom de fichier GGUF, ou identifiant
`ollama pull` avec sa quantification), pas le poids lui-même. C'est la
même distinction déjà appliquée à `docs/repositories.md`/`docs/editor.md` :
le dépôt versionne la **recette**, jamais l'artefact volumineux.

**Vérification d'intégrité** :
- **Ollama** : mécanisme de type registre de conteneurs — manifeste et
  blobs adressés par empreinte SHA256 (établi par la même famille de
  mécanisme que celui déjà observé via `skopeo inspect`, § 3.2, propre
  à l'écosystème OCI dont Ollama réutilise le modèle ; comportement
  spécifique d'`ollama pull` non vérifié directement ici, aucun `pull`
  exécuté). **[RÉSOLU le 2026-08-07, IA-3, § 9.1] ÉTAIT** un marqueur de
  vérification ouvert (comportement non testé) — **testé réellement**,
  par corruption délibérée d'un blob déjà tiré (un octet modifié, puis
  restauré) : `ollama pull` vérifie bien le contenu **reçu sur le
  réseau** (« verifying sha256 digest », observé dans sa propre
  sortie), mais **ne détecte ni ne répare** un blob **déjà présent
  localement** sous son nom attendu même s'il a été corrompu après
  coup — seule la suppression du fichier avant un nouveau `pull` force
  une vérification réelle. Écart nommé, `docs/repositories.md` § 8.
- **GGUF téléchargé manuellement** (Hugging Face typiquement) : pas de
  mécanisme d'intégrité uniforme au niveau du format lui-même — dépend
  de ce que la source publie à côté (empreinte SHA256 dans la page du
  dépôt, ou pointeur Git LFS qui porte sa propre validation lors du
  clone). À vérifier au cas par cas selon la source retenue, pas un
  mécanisme unique à documenter ici.

## 6. Options, coûts, recommandation

| Option | Apporte | Coûte | Risque de casse | Comment savoir qu'elle fonctionne |
|---|---|---|---|---|
| **A — Ollama conteneurisé (image CUDA officielle, CDI)** | API compatible OpenAI, gestion de modèles avec empreintes, chargement/déchargement automatique configurable, isolation Podman déjà éprouvée | Image externe (~plusieurs centaines de Mio, non chiffré ici sans téléchargement), câblage systemd à écrire (démarrage de session, § 2.4) | Péremption CDI silencieuse si non câblée (§ 2.4) ; comportement par défaut (déchargement 5 min) inadapté à la complétion sans configuration explicite (§ 3.2) | `nvidia-smi --query-compute-apps` (sonde isolée) montre le conteneur comme client GPU ; requête `/v1/chat/completions` répond ; tag verify-cdi-spec en `ExecStartPre` réussit |
| **B — `llama-server` conteneurisé (image CUDA amont, CDI)** | Contrôle plus fin (contexte, parallélisme, mode routeur), même logiciel que le paquet Fedora déjà connu (juste un backend différent) | Gestion de modèles manuelle (pas de `pull` intégré), un conteneur par modèle sauf mode routeur | Mêmes risques CDI que l'option A | Idem — `nvidia-smi`, requête `/v1/completions` |
| **C — vLLM conteneurisé** | Débit élevé, adapté au multi-utilisateurs | VRAM de gestion de lot significative, orienté production — probablement disproportionné pour un usage mono-utilisateur sur 16 Gio partagés | Configuration la plus complexe des trois, risque de sur-allocation VRAM qui écrase les autres usages (§ 1) | Idem, plus surveillance de la VRAM allouée au cache de lot |
| **D — Binaire natif téléchargé hors dépôt (script officiel ou compilation locale CUDA)** | Pas de couche de conteneurisation, latence de démarrage potentiellement plus faible (non mesuré) | Aucun mécanisme de détection de dérive équivalent à `verify-cdi-spec` ; reconstructibilité D1 plus faible (§ 4) | Échec silencieux plus probable en cas de dérive de bibliothèques système | `nvidia-smi --query-compute-apps` montre le binaire comme client — sinon, pas de garde-fou automatique connu |
| **E — Paquet natif des onze dépôts (`ollama`/`llama-cpp` Fedora)** | Rien lié au GPU (§ 3.1) — **exclue pour l'usage GPU visé** | — | — | — |

**Aucune option non mesurable ne doit être recommandée** (consigne
explicite) — l'option E est explicitement **écartée**, pas seulement
non recommandée : les chiffres (§ 3.1) montrent qu'elle ne peut pas
atteindre l'objectif (accès à la RTX 4090), quel qu'en soit le coût.

## Recommandation

**Option A — Ollama, conteneurisé, image CUDA officielle, via le
mécanisme CDI déjà prouvé.** Argumentation :

1. **Seule voie qui utilise réellement la RTX 4090** parmi celles
   disponibles sans repli CPU silencieux (§ 3.1 exclut toute solution
   native des onze dépôts).
2. **Répond directement à la tension du § 1** : `keep_alive`/
   `OLLAMA_MAX_LOADED_MODELS` permettent, sur le papier, de tenir un
   petit modèle de complétion en permanence (`keep_alive=-1`) tout en
   laissant un modèle de chat/agent plus gros se charger à la demande
   — exactement le compromis à trois usages recherché, avec les
   réglages explicites nécessaires nommés (§ 3.2), pas laissés au
   défaut qui échouerait pour la complétion.
3. **S'appuie sur du travail déjà validé, pas un nouveau risque** :
   CDI rootless prouvé de bout en bout (`docs/gpu-containers.md`),
   comportement RTD3 en présence d'un conteneur déjà mesuré sans
   réveil au lancement seul (§ 8.7 du même document).
4. **Reconstructible sous D1** : image identifiée par tag et empreinte
   vérifiable (`skopeo inspect`), recette versionnable dans ce dépôt ;
   seuls les poids du modèle restent hors dépôt, comme n'importe quel
   artefact volumineux (D4).
5. **La péremption CDI (§ 2.4) devient le principal chantier
   restant** — mécanisme de détection déjà écrit
   (`verify-cdi-spec`), à câbler en `ExecStartPre=` de l'unité qui
   lancerait ce conteneur, pour qu'une mise à jour de pilote fasse
   échouer bruyamment le démarrage plutôt que de laisser tourner un
   serveur qui a silencieusement perdu son GPU.

**Ce qui départage des options B/C** : `llama-server` (B) reste une
alternative raisonnable si le contrôle fin (mode routeur, paramètres
de contexte par requête) s'avère nécessaire à l'usage — pas exclue,
simplement moins immédiatement adaptée à la gestion multi-modèles du
§ 1 sans la construire soi-même. vLLM (C) n'est pas justifié par le
profil d'usage décrit (mono-utilisateur, VRAM partagée avec le
bureau) — son coût de configuration et de VRAM de lot ne trouve pas de
contrepartie ici.

## Questions qui départagent

**Aucun modèle choisi ici** — ces questions suffisent à trancher les
choix restants :

1. **La complétion de code doit-elle rester chaude en permanence
   (coût : VRAM et éveil continu de la carte, plus de consommation),
   ou acceptez-vous un délai de réveil occasionnel pour économiser
   l'énergie ?** Détermine `keep_alive=-1` contre une valeur finie, et
   donc si RTD3 s'engage jamais pendant les heures de travail actives.
2. **Quelle taille de modèle pour l'agent — 13 B (tient large,
   qualité modeste) ou jusqu'à ~20 B (tient serré, cache KV réduit,
   qualité meilleure mais contexte plus court) ?** Le § 1 montre que
   32 B et au-delà sont hors de portée quel que soit l'arbitrage.
3. **Quelle quantification — `Q4_K_M` (le point d'étalonnage utilisé
   au § 1, bon compromis documenté par le projet amont) ou une
   quantification plus agressive (`Q3_K`/`IQ*`, plus de marge VRAM,
   perte de qualité non chiffrée ici) ?**
4. **Ollama (gestion de modèles intégrée, empreintes automatiques,
   API OpenAI) ou `llama-server` (contrôle plus fin, pas de `pull`
   intégré) — la commodité de gestion vaut-elle la perte de contrôle
   fin ?**
5. **Le petit modèle de complétion et le modèle de chat/agent
   doivent-ils pouvoir cohabiter en VRAM (§ 1, plausible
   arithmétiquement, non vérifié), ou un seul modèle actif à la fois
   suffit-il, quitte à payer un rechargement au changement d'usage ?**

## 7. IA-1 — infrastructure déployée (2026-08-07)

**Quatre décisions de l'opérateur**, numéros vérifiés libres avant
attribution (`grep -n '^\*\*D[0-9]' docs/machine-facts.md` — D13 le
plus élevé au moment de la vérification) :

- **D14** — service d'inférence : Ollama conteneurisé, image CUDA
  officielle, accès GPU par CDI (seule voie qui atteigne la RTX 4090,
  § 3.1 — troisième surface d'approvisionnement de ce dépôt, après les
  extensions d'éditeur et les registres de conteneurs, D11/D12).
- **D15** — modèle de complétion résident en permanence (`keep_alive`
  infini) — contrepartie assumée : RTD3 ne s'engage plus tant que le
  service tient un contexte CUDA ouvert.
- **D16** — `NVreg_PreserveVideoMemoryAllocations=1` — contrepartie
  assumée : suspension/réveil système plus lents.
- **D17** — dimensionnement initial 13 B / `Q4_K_M`, à mesurer avant
  toute montée — **aucun modèle choisi ni téléchargé par ce livrable**.

### 7.1 — Résolution avant écriture

**Comment `NVreg_PreserveVideoMemoryAllocations` prend effet, et
comment vérifier qu'il est actif** — établi avant d'écrire quoi que ce
soit : c'est un paramètre de **module noyau**, chargé au moment où
`nvidia.ko` est inséré — pas un fichier `sysfs` réinscriptible à chaud
(vérifié : `/sys/module/nvidia/parameters/PreserveVideoMemoryAllocations`
n'existe pas). Sur ce poste, le module est **en usage actif** par la
session graphique en cours :
```
$ lsmod | grep -E '^nvidia'
nvidia_uvm       ...   0
nvidia_drm       ...   3
nvidia_modeset   ...   3 nvidia_drm
nvidia           ...   148 nvidia_uvm,nvidia_modeset
```
Compteur d'usage `148`, pas `0` — le décharger (`rmmod`) sans arrêter
la session graphique échouerait ou l'interromprait ; **en pratique,
seul un redémarrage applique proprement une nouvelle valeur ici**
(§ 4 de la demande, traité au § 7.5). **Vérification de la valeur
effective** : `/proc/driver/nvidia/params` l'expose bien (déjà utilisé
en IA-0/§ 2.1bis) — méthode retenue par le rôle (§ 7.3) pour relever
l'état avant écriture, et par la commande de vérification préparée
pour après un redémarrage (§ 7.5).

**Mécanisme de service — Quadlet, pas une unité écrite à la main** :
Podman 5.8.4 fournit nativement un générateur systemd utilisateur —
vérifié présent avant de choisir :
```
$ rpm -q podman
podman-5.8.4-1.fc44.x86_64
$ ls /usr/lib/systemd/user-generators/podman-user-generator
/usr/lib/systemd/user-generators/podman-user-generator
```
Des fichiers déclaratifs `.container`/`.volume` déposés dans
`~/.config/containers/systemd/` sont transformés en unités systemd
utilisateur réelles à `daemon-reload` (`man podman-systemd.unit`,
consulté en entier) — retenu, une unité `.service` écrite à la main
appelant `podman run` n'a jamais été envisagée au-delà de cette
vérification (CLAUDE.md § déterminer ce que l'outil gère déjà).

**Où vivent les modèles — pas de sous-volume btrfs dédié** : aucun
outil de snapshot n'est installé sur ce poste, vérifié avant d'écrire
quoi que ce soit :
```
$ rpm -qa | grep -iE 'snapper|timeshift|btrfs-assistant'
(vide)
```
Sans régime de snapshot à en exclure, un sous-volume dédié n'apporte
ni exclusion de snapshot ni quota séparé — **abstention justifiée**,
pas un réflexe : les poids vivent dans un volume Podman **nommé**
ordinaire (unité Quadlet `.volume`), sous l'arborescence de stockage
rootless standard de Podman (`~/.local/share/containers/storage/volumes/`),
déjà sur le btrfs `Data,single` en place (D1).

### 7.2 — Épinglage de l'image par empreinte

```
$ skopeo inspect docker://docker.io/ollama/ollama:0.32.6
Digest: sha256:b88c73ace3e115f8ec53dc8761ae1c0aabfa675406e3681786b98757ce050f42
```
`0.32.6` est, au moment de cette lecture, la version stable numérique
la plus récente (222 étiquettes numériques pures recensées,
`0.32.6` en tête) et pointe vers la **même empreinte** que `:latest` —
vérifié, pas supposé. **Nuance établie en creusant plus loin, absente
d'une lecture superficielle** : cette empreinte est celle de l'**index
multi-architecture** (`skopeo inspect --raw` : `mediaType:
application/vnd.oci.image.index.v1+json`, pas un manifeste
mono-architecture) — Podman résout lui-même la plateforme locale
(amd64) à partir de cet index au tirage, comportement OCI standard.
Épinglage retenu dans `roles/local_ai/defaults/main.yml` :
`docker.io/ollama/ollama@sha256:b88c73...` — jamais une étiquette
mobile.

### 7.3 — Rôle exécuté, résultat

Trois gardes préalables (§ 2 de la demande), toutes échec bruyant,
réutilisant `verify-cdi-spec` sans le réimplémenter — exécution réelle,
première fois :
```
$ ansible-playbook roles/local_ai/local_ai.yml
Garde 1/3 : "Spécification CDI vérifiée à jour (/etc/cdi/nvidia.yaml)."
Garde 2/3 : "Podman rootless confirmé : true."
Garde 3/3 : "Module 'nvidia_uvm' chargé."
[...]
PLAY RECAP : ok=23  changed=5  failed=0
```
**Réussi dès la première exécution réelle** — écrasement modprobe.d
déployé, fichier fournisseur vérifié intact avant/après (`rpm -Vf`
identique), volume et unité Quadlet déployés, service démarré, API
vérifiée locale uniquement, liste des modèles vide confirmée, GPU
visible depuis l'intérieur du conteneur confirmé (détail § 7.4).
Deuxième exécution : `changed=0`, idempotence confirmée.

**Découverte non anticipée, signalée, pas exploitée ni corrigée ici**
(CLAUDE.md § une découverte qui contredit une attente se signale) —
le journal du conteneur (`podman logs ollama`), lu après démarrage :
```
OLLAMA_REMOTES:[ollama.com] ... OLLAMA_NO_CLOUD:false
msg="Ollama cloud disabled: false"
msg="model recommendations cache sleep scheduled" wait=4h31m...
```
**L'image officielle contacte `ollama.com` de son propre chef**, peu
après le démarrage, pour une liste de « recommandations de modèles »
(682 octets, `cache/model-recommendations.json` dans le volume — pas
un modèle, mais un appel réseau sortant non demandé) — et expose une
fonctionnalité « cloud » **activée par défaut**
(`OLLAMA_NO_CLOUD:false`), y compris des entrées « `:cloud` » dans
cette même liste (ex. `glm-5.2:cloud`) qui, si jamais exécutées,
routeraient vers l'infrastructure d'Ollama plutôt que d'inférer
localement. **Rien dans ce livrable n'a demandé ni configuré cela** —
comportement par défaut de l'image, découvert en testant, pas en
lisant sa documentation au préalable. Non corrigé ici (hors périmètre
de ce livrable, quatre décisions précises seulement) — nommé
explicitement pour qu'un livrable ultérieur tranche
(`OLLAMA_NO_CLOUD=1`, ou un pare-feu sortant, ou l'acceptation
délibérée du comportement par défaut).

### 7.4 — Preuve que le conteneur voit le GPU (depuis l'intérieur, pas l'hôte)

```
$ podman exec ollama sh -c 'ls -la /dev/nvidia*; nvidia-smi -L'
crw-rw-rw-. ... /dev/nvidia-modeset
crw-rw-rw-. ... /dev/nvidia-uvm
crw-rw-rw-. ... /dev/nvidia-uvm-tools
crw-rw-rw-. ... /dev/nvidia0
crw-rw-rw-. ... /dev/nvidiactl
GPU 0: NVIDIA GeForce RTX 4090 Laptop GPU (UUID: GPU-bd9088d2-0875-7077-8475-ac1bc4519249)
```
**Corroboré indépendamment par le propre journal interne d'Ollama**
(pas une commande que ce livrable a provoquée — Ollama l'émet de
lui-même à l'initialisation) :
```
msg="inference compute" id=0 library=CUDA compute=8.9
  name=CUDA0 description="NVIDIA GeForce RTX 4090 Laptop GPU"
  driver=13.3 pci_id=0000:01:00.0 type=discrete
  total="15.6 GiB" available="15.3 GiB"
```
`compute=8.9` (Ada Lovelace), `driver=13.3` (CUDA UMD, cohérent avec
le fait déjà établi) — le CDI natif accorde au conteneur un accès
CUDA fonctionnel à la RTX 4090, confirmé par deux sources
indépendantes internes au conteneur, aucune depuis l'hôte.

### 7.5 — Redémarrage

**Non déclenché**, conformément à la consigne. Le paramètre D16 est
déployé (`/etc/modprobe.d/local-ai-nvidia-power-management.conf`) mais
**pas encore actif** — valeur actuellement chargée toujours `2`
(§ 2.1bis), pas `1`. **Commande de vérification post-redémarrage,
préparée, à exécuter par l'opérateur quand il en décidera** :
```
$ cat /proc/driver/nvidia/params | grep -i PreserveVideoMemoryAllocations
# attendu après redémarrage : PreserveVideoMemoryAllocations: 1
```
Le service Ollama, activé (`enabled: true`), redémarrerait
automatiquement à la prochaine ouverture de session (`WantedBy=
default.target`) — pas testé ici, comme l'autostart de `kitty` en
BUR-1, seule la commande de vérification est préparée.

### 7.6 — Consommation au repos, service démarré sans modèle (référence pour IA-2)

Méthode d'isolement de `docs/dgpu-power.md` appliquée : deux relevés
`sysfs` purs, espacés, **aucun `nvidia-smi` entre les deux**, service
Ollama confirmé `active` (sans modèle) tout du long :
```
T0 2026-08-07T10:59:43+02:00  runtime_status=suspended
                               runtime_suspended_time=120797976
                               runtime_active_time=1549856
[280 s, aucune sonde, service actif sans modèle]
T1 2026-08-07T11:04:23+02:00  runtime_status=suspended   (inchangé)
                               runtime_suspended_time=121078216  (Δ=+280240 ms ≈ Δt réel)
                               runtime_active_time=1549856       (Δ=0 ms)
```
**Établi, pas supposé** : un service Ollama démarré, sans aucun modèle
chargé, **n'ouvre pas de contexte CUDA persistant** — RTD3 s'engage
exactement comme si le service ne tournait pas, sur toute la fenêtre
de 280 s observée. Cohérent avec le journal du conteneur (§ 7.3) : la
découverte GPU se fait une fois, brièvement, à l'initialisation, sans
rester active. **Référence pour IA-2** : tant qu'aucun modèle n'est
chargé, ce service a un coût énergétique nul au-delà de son
empreinte mémoire système habituelle (pas de VRAM, pas de veille RTD3
bloquée) — la contrepartie assumée par D15 (RTD3 neutralisée) ne
commence qu'au premier chargement effectif d'un modèle, pas au
démarrage du service lui-même.

## Validation — IA-0 (résolution en lecture seule, 2026-08-06)

**Commandes exécutées, non modifiantes sauf l'incident déjà signalé** :

| # | Commande | Nature | Réveille le GPU ? |
|---|---|---|---|
| 1 | Lecture `docs/dgpu-power.md`, `docs/gpu-containers.md`, `roles/gpu_cdi/tasks/*.yml` | lecture locale, dépôt | non applicable |
| 2 | `grep`/`cat` sur `/usr/share/doc/xorg-x11-drv-nvidia/html/*.html` | lecture documentation amont locale | non |
| 3 | `cat /sys/power/mem_sleep` | lecture sysfs | non |
| 4 | `cat /proc/driver/nvidia/gpus/0000:01:00.0/power` (×2, avant/après isolées) | lecture procfs pilote, passive | **non — vérifié explicitement** : `runtime_status`/`runtime_active_time` inchangés immédiatement avant et après (§ 2.3) |
| 5 | `cat /proc/driver/nvidia/params` | lecture procfs pilote | non (même famille que #4, interface de paramètres statiques) |
| 6 | `systemctl list-unit-files \| grep nvidia` | lecture état systemd | non |
| 7 | `cat /sys/bus/pci/devices/0000:01:00.0/power/{runtime_status,runtime_suspended_time}` | lecture sysfs passive | non |
| 8 | `nvidia-smi --query-gpu=...` (une sonde unique, isolée) | lecture active du pilote | **oui, délibérément et isolée** — voir méthode § 2.3 |
| 9 | `nvidia-smi --query-compute-apps=...` (immédiatement après #8, même fenêtre de réveil) | lecture active | oui, mais ne prolonge pas une deuxième fois le réveil (même fenêtre que #8) |
| 10 | `ansible-playbook roles/gpu_cdi/gpu_cdi.yml --tags verify-cdi-spec` | lecture seule (rejoué), voir motif dans le fichier lui-même | non (n'invoque ni `nvidia-smi` ni aucun outil qui interroge activement le pilote — vérifié par lecture du rôle) |
| 11 | `dnf repoquery --available/--requires/-l` (réutilise l'extraction EDI-0 + requêtes ciblées) | métadonnées `dnf`, Terra gardée par `--assumeno` où pertinent | non applicable |
| 12 | `skopeo inspect docker://...` (×3) | métadonnées de registre distant, **aucune couche téléchargée** — vérifié par `podman images` inchangé avant/après chaque appel | non applicable |
| 13 | Lecture externe : `github.com/ggml-org/llama.cpp` (README quantification, tools/server), `hub.docker.com/r/ollama/ollama`, `ollama.readthedocs.io/en/faq` | requêtes HTTP GET, documentation amont | non applicable |
| 14 | `df -h /home`, `btrfs filesystem usage /` (sans privilège, après correction) | lecture système de fichiers | non applicable |
| 15 | `podman images`, `command -v skopeo` | lecture locale | non applicable |

**Incident signalé, corrigé dans la série** : `sudo btrfs filesystem
usage /` exécuté par réflexe (§ tête de document) — la version sans
`sudo` fonctionne identiquement pour les chiffres retenus ici (seul le
détail par périphérique, non nécessaire, manque). Aucune autre
élévation dans cette série.

**Actions privilégiées : une seule, par erreur, reconnue — pas
répétée.** `sudo btrfs filesystem usage /` (lecture seule malgré le
privilège, aucune écriture) — chemin cible : `/` (racine btrfs), motif
donné au moment de l'exécution : aucun, réflexe non justifié,
corrigé dès constat. Toutes les autres commandes de ce livrable
s'exécutent sans `sudo`.

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : quatre
marqueurs actionnables dans ce document — comportement réel d'une
suspension système S0ix avec modèle chargé (§ 2.1), chargement
simultané réel de deux modèles (§ 1), comportement runtime de
llama-cpp/ollama Fedora sans périphérique ROCm (§ 3.1), comportement
d'`ollama pull` en cas de coupure/corruption (§ 5).

**Confirmations finales** : aucun paquet installé ; aucun modèle ni
aucune image téléchargé (`skopeo inspect` lit des manifestes, jamais
des couches — vérifié par `podman images` inchangé) ; aucun conteneur
lancé (seule la vérification `verify-cdi-spec`, qui n'en lance aucun,
a été rejouée) ; aucun dépôt ajouté ; aucun paramètre de pilote
modifié ; aucun fichier écrit hors de ce dépôt.

## Validation — IA-1 (infrastructure déployée, 2026-08-07)

**Actions privilégiées, exhaustives** :

| # | Commande | Cible | Motif |
|---|---|---|---|
| 1 | `ansible.builtin.template` (`become: true`) | `/etc/modprobe.d/local-ai-nvidia-power-management.conf` | D16, écrasement propre — seule tâche `become: true` du rôle |
| 2 | `sudo -n ausearch -m avc -ts recent` (hors rôle, validation manuelle) | lecture `/var/log/audit/` | corroborer l'absence de refus AVC, même méthode que `docs/gpu-containers.md` § 8.6 — silencieuse sous D9 (NOPASSWD), disclosed ici |

Toutes les autres actions du rôle (les trois gardes, lecture
`/proc/driver/nvidia/params`, `rpm -Vf` (×2), déploiement Quadlet,
`systemctl --user`, `podman exec`, requêtes `curl`/`uri`) s'exécutent
sans `sudo`. `journalctl -k -g 'avc:'` (méthode alternative, sans
privilège) confirme indépendamment l'absence de refus AVC — voir le
relevé complet ci-dessous.

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/local_ai/local_ai.yml   # succès
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=5 (première écriture réelle)
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=0 (idempotence confirmée)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/local_ai/
Passed: 0 failure(s), 0 warning(s) — profil production, deux `noqa` justifiés
(rpm -Vf sans équivalent module ; réutilisation délibérée du nom de
variable gpu_cdi_verify_spec_path, pas un oubli de préfixe)
```

**Les deux démonstrations d'échec forcé** (§ 7.3), avant toute action,
toutes deux `changed=0` :
```
$ ansible-playbook roles/local_ai/local_ai.yml \
    -e gpu_cdi_verify_spec_path=/tmp/chemin-cdi-invalide-inexistant.yaml
fatal: [...] => "Spécification CDI absente, périmée, ou chemin invalide
(/tmp/chemin-cdi-invalide-inexistant.yaml) [...] Ce rôle s'arrête :
déployer le service sur cette base démarrerait avec un GPU
silencieusement invisible [...]"

$ ansible-playbook roles/local_ai/local_ai.yml \
    -e local_ai_required_kernel_module=module_absent_garanti_xyz
fatal: [...] => "Module noyau requis absent : 'module_absent_garanti_xyz'
ne figure pas dans /proc/modules. Ce rôle s'arrête [...]"
```
Service restauré à l'état nominal après chaque essai (rejoué sans
dérogation, `changed=0`) — aucun essai n'a laissé le système dans un
état de test.

**Empreinte de l'image épinglée** : `sha256:b88c73ace3e115f8ec53dc8761ae1c0aabfa675406e3681786b98757ce050f42`
— établie par `skopeo inspect docker://docker.io/ollama/ollama:0.32.6` (§ 7.2).

**Preuve que l'API n'écoute pas sur une interface externe** :
```
$ ss -H -tln sport = :11434
LISTEN 0 128 127.0.0.1:11434 0.0.0.0:*
```
Aucune occurrence de `0.0.0.0:11434` ni `*:11434` — assertion du rôle
sur ce point précis, réussie à chaque exécution (§ 7.3).

**SELinux** :
```
$ getenforce   # avant toute action de ce livrable, et après : Enforcing (inchangé)
$ journalctl -k -g 'avc:' --no-pager -n 20   # -- No entries --  (sans privilège)
$ sudo -n ausearch -m avc -ts recent          # <no matches>      (avec privilège, corroboration)
```
Aucun refus AVC, par les deux méthodes, avant et après le déploiement
du service et son exécution réelle.

**Consommation et veille runtime, service démarré sans modèle** :
§ 7.6 — `runtime_active_time` inchangé (0 ms de delta) sur une fenêtre
de 280 s, service confirmé `active` tout du long. RTD3 s'engage
normalement ; référence chiffrée pour IA-2.

**Aucun modèle téléchargé** : `curl http://127.0.0.1:11434/api/tags`
→ `{"models":[]}`, confirmé à chaque exécution du rôle (assertion
dédiée) ; volume `systemd-ollama-models` inspecté directement — 12 Kio
au total (clés d'identité Ollama générées au premier démarrage +
cache de recommandations, § 7.3 — pas un modèle).

**`/usr/lib/modprobe.d/` intact** :
```
$ sha256sum /usr/lib/modprobe.d/nvidia-power-management.conf
de19aadaad8cdacd4caa5331a87b49cc9a79e36f81badec4932ad26343ea0257
```
Identique avant et après ce livrable (même valeur qu'IA-0/EDI-1) — et
vérifié en continu par le rôle lui-même (`rpm -Vf`, avant/après
écriture, assertion dédiée, § 7.3).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : à la
clôture d'IA-1, cinq marqueurs actionnables dans ce document — les
quatre déjà comptés en IA-0 (§ 2.1 suspension S0ix avec modèle chargé,
§ 1 chargement simultané de deux modèles, § 3.1 comportement runtime
ROCm sans périphérique, § 5 comportement `ollama pull` en cas de
coupure) **plus un nouveau** (§ 2.1bis, signification de la valeur
brute par défaut `2` de `NVreg_PreserveVideoMemoryAllocations`).
Aucun marqueur fermé dans cette série — celui sur `absent/0` au § 2.1
est corrigé (la prémisse était fausse), pas vérifié ; le nouveau
marqueur du § 2.1bis en découle directement.

**Confirmations finales** : aucun modèle téléchargé, sous aucune
forme (seule l'image du serveur — infrastructure — a été tirée,
distinction explicite § 7 en tête) ; aucun conteneur lancé au-delà du
service Ollama lui-même, objet de ce livrable ; aucun dépôt ajouté ;
`/usr/lib/modprobe.d/nvidia-power-management.conf` intact (somme de
contrôle identique) ; `sudoers`, `gpu_mux_mode`, `dgpu_disable`,
`kwinrulesrc` non touchés ; SELinux jamais modifié (`Enforcing`
inchangé, aucun `setenforce`) ; aucun redémarrage, aucune déconnexion
de session.

### 7.7 — Redémarrage, confirmé (2026-08-07, reconfirmé dans cette même
session avant IA-2)

Le redémarrage laissé en attente au § 7.5 a eu lieu. Quatre contrôles,
relus **directement dans la session IA-2**, pas seulement repris du
rapport de l'opérateur :
```
$ cat /proc/driver/nvidia/params | grep -i PreserveVideoMemory
PreserveVideoMemoryAllocations: 1
$ systemctl --user list-units 'ollama*'
ollama-models-volume.service   loaded active exited
ollama.service                 loaded active running
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
suspended
$ sha256sum /usr/lib/modprobe.d/nvidia-power-management.conf
de19aadaad8cdacd4caa5331a87b49cc9a79e36f81badec4932ad26343ea0257   # identique à § 7 ci-dessus
```
D16 est donc réellement actif (`1`, plus `2` non documenté du § 2.1bis) ;
le service est remonté seul par Quadlet (`enabled: true`, § 7.3) sans
action manuelle ; RTD3 s'engage normalement service démarré sans
modèle, cohérent avec § 7.6 ; le fichier fournisseur reste intact.

## 8. IA-2 — confinement réseau, faisabilité de la complétion, candidats de modèles (2026-08-07)

**Aucun modèle téléchargé dans ce livrable.** Périmètre : `roles/local_ai/`
(partie réseau), ce document, `docs/repositories.md` (§ 6, registre de
conteneurs nommé), `docs/machine-facts.md` (en dernier).

### 8.0 — Deux corrections portées depuis IA-0/IA-1

**8.0.1 — Amalgame entre deux mécanismes distincts, dont l'opérateur
est l'auteur du signalement.** `Video Memory: Off` (statut RTD3, veille
**runtime**, par périphérique) et `NVreg_PreserveVideoMemoryAllocations`
(suspension **système**, S0ix/s2idle sur ce poste) sont deux chemins
indépendants — déjà correctement distingués dans le corps de ce
document (§ 2.1, § 2.1bis, § 7.1) depuis IA-1. **Le seul endroit où
l'amalgame subsistait, non corrigé, est un point ouvert de
`docs/machine-facts.md`** (« `NVreg_PreserveVideoMemoryAllocations`
commenté : les allocations VRAM ne survivent pas à la veille ») —
corrigé dans ce même livrable, § Décisions/points ouverts,
`[CORRIGÉ le 2026-08-07]`/`ÉTAIT`, pas effacé (CLAUDE.md § marquer
l'historique plutôt que l'effacer).

**8.0.2 — Conclusion tirée d'une prémisse juste mais jamais lue
directement.** La prémisse (« paramètre commenté = paramètre absent »)
est correcte ; la conclusion comportementale (« les allocations VRAM ne
survivent pas ») ne l'était pas — personne n'avait lu la valeur brute
réellement chargée avant IA-1 § 2.1bis, qui a établi qu'elle vaut `2`,
pas `0`. Le marqueur sur la signification exacte de `2` reste ouvert
(§ 2.1bis) — seule la conclusion comportementale non sourcée est
corrigée, pas la question de fond. Même correction reportée dans
`docs/machine-facts.md` (§ Décisions/points ouverts, en dernier).

### 8.1 — Ce que le service contacte réellement, établi par lecture puis par mesure

**Aucun nom de variable présumé** — trois sources locales, pas de
documentation amont supposée avant d'avoir été lue :

1. **`ollama serve --help` (exécuté dans le conteneur, 0.32.6, source
   la plus autoritative possible : le binaire réellement déployé sur ce
   poste)** — seule variable de sortie réseau **documentée** par le
   binaire lui-même :
   ```
   OLLAMA_NO_CLOUD   Disable Ollama cloud features (remote inference and web search)
   ```
   Valeur par défaut non donnée par `--help` — établie par la lecture
   suivante.
2. **Journal du serveur au démarrage** (`podman logs ollama`,
   `msg="server config"`, émis par Ollama lui-même, pas provoqué) — la
   carte complète des variables **effectivement chargées** à cet
   instant, dont deux gouvernent la sortie réseau et n'apparaissent
   **nulle part dans `--help`** :
   ```
   OLLAMA_NO_CLOUD:false   OLLAMA_REMOTES:[ollama.com]
   ```
   `OLLAMA_NO_CLOUD` par défaut **`false`** (cloud activé) ;
   `OLLAMA_REMOTES` par défaut **`[ollama.com]`** — undocumented par
   `--help`, seule cette lecture du journal réel l'établit.
3. **Chaînes compilées dans le binaire** (`podman cp` du binaire vers
   l'hôte, `strings -a`, sur le fichier réellement exécuté par ce
   service — pas une hypothèse de nommage) — surface plus large que les
   deux lectures précédentes, y compris des variables **absentes du
   journal de démarrage standard** (probablement gardées par un
   commutateur expérimental non activé ici — `OLLAMA_EXPERIMENT`
   apparaît aussi dans cette liste) :
   ```
   OLLAMA_NO_CLOUD  OLLAMA_REMOTES  OLLAMA_CLOUD_BASE_URL  OLLAMA_API_KEY
   OLLAMA_AGENT_DISABLE_WEBSEARCH  OLLAMA_AGENT_DISABLE_SHELL
   ```
   `@VERIF : sémantique exacte et valeur par défaut de
   OLLAMA_CLOUD_BASE_URL, OLLAMA_API_KEY,
   OLLAMA_AGENT_DISABLE_WEBSEARCH, OLLAMA_AGENT_DISABLE_SHELL —
   absentes du journal « server config » standard (§ ci-dessus, donc
   non observées en fonctionnement), leur nom seul est établi
   (extraction de chaînes du binaire réellement déployé), pas leur
   comportement. La documentation officielle de `OLLAMA_NO_CLOUD`
   (« remote inference **and web search** ») suggère que
   `OLLAMA_AGENT_DISABLE_WEBSEARCH` est un sous-bascule de la même
   fonctionnalité cloud plutôt qu'un vecteur de sortie indépendant —
   plausible, non confirmé par une lecture directe de code.` **Sans
   conséquence sur la voie retenue ci-dessous** : c'est précisément
   l'argument en faveur du confinement réseau structurel (§ 8.2) plutôt
   que d'une liste de variables à poser une par une — un vecteur non
   encore identifié par nom (comme ceux-ci) reste bloqué par construction,
   pas par exhaustivité de configuration.

**Trois catégories, statut distinct (comme demandé)** :
- **Récupération de modèles** (`ollama pull`, registre de modèles
  Ollama) — surface d'approvisionnement **attendue et assumée**, nommée
  dans `docs/repositories.md` § 6.
- **Appels de recommandation non sollicités** — confirmés par mesure
  (§ 7.3, § 8.3 ci-dessous) : 682 octets vers `ollama.com`, jamais
  demandés par ce dépôt.
- **Fonctionnalité « cloud »** (`OLLAMA_NO_CLOUD:false` par défaut) — la
  plus grave : ouvre la possibilité que des **requêtes d'inférence**
  quittent la machine, contredisant directement le motif « rien ne
  quitte la machine » d'IA-0 § 1.

### 8.2 — Concevoir le confinement : ce que l'outil fait déjà, ce que Podman offre en propre

**Voie A — variable d'environnement (`OLLAMA_NO_CLOUD=1`), vérifiée
avant d'être retenue, pas posée de confiance** (CLAUDE.md § une
variable posée n'est pas un effet constaté). Testé sur un conteneur
jetable, réseau par défaut (sortie disponible, pour isoler l'effet de
la seule variable) :
```
$ podman run -d --network=pasta -p 127.0.0.1:19999:11434 -e OLLAMA_NO_CLOUD=1 <même image épinglée> serve
$ podman exec <ctr> sh -c 'ls /root/.ollama/cache/'
ls: cannot access '/root/.ollama/cache/': No such file or directory
```
**Aucune tentative, pas seulement un échec propre** — comparé au même
test sans la variable, où `cache/model-recommendations.json` (682
octets) est créé (§ 8.3). `OLLAMA_NO_CLOUD=1` supprime réellement la
tentative, pas seulement l'exécution d'un modèle cloud si une requête
le demandait — vérifié, retenu comme complément.

**Voie B — Podman `--network=none`.** Testé sur la même image, jamais
sur le conteneur de production :
```
$ podman run --rm --network=none -p 127.0.0.1:19999:11434 <image> serve
Port mappings have been discarded because "none" network namespace mode does not support them
```
**Coût disqualifiant, plus large que celui nommé par la demande** : pas
seulement `ollama pull` qui échouerait, **l'API locale elle-même
cesse d'être joignable depuis l'hôte** — `--network=none` supprime
toute la pile réseau du conteneur, y compris le mécanisme de transfert
de port (`pasta`/`rootlessport`) dont `PublishPort=` dépend. Écarté :
casse l'usage même que ce service existe pour servir.

**Voie C — réseau Podman dédié, `--internal`** (`man podman-run` §
`--network`, `man podman-systemd.unit` § Network units, `Internal=`
↔ `--internal`, lu avant d'écrire quoi que ce soit). Testé sur un
conteneur jetable avant de le retenir :
```
$ podman network create --internal ollama-test-internal
$ podman run -d --network=ollama-test-internal -p 127.0.0.1:19999:11434 <image> serve
$ curl -sS http://127.0.0.1:19999/api/tags
{"models":[]}                                                      # PublishPort intact
$ podman logs <ctr> | grep -i recommendations
msg="model recommendations refresh failed" error="Get \"https://ollama.com/api/experimental/model-recommendations\":
  dial tcp: lookup ollama.com on 10.89.0.1:53: no such host"        # sortie bloquée
```
**Les deux propriétés recherchées, vérifiées ensemble** :
`PublishPort=` (hôte → conteneur) et la restriction de sortie
(conteneur → Internet) sont deux mécanismes indépendants dans
Podman/netavark — un réseau `--internal` n'installe simplement aucune
route vers la passerelle par défaut, sans toucher au transfert de port.

**Retenu : les deux voies, en profondeur, pas l'une à la place de
l'autre.** Critère de départage explicite (demande § 1.2) : préférer ce
qui se **vérifie** à ce qui se **configure**. Le réseau interne (voie
C) est la garde **structurelle**, vérifiable depuis l'hôte sans faire
confiance au bon comportement du logiciel à l'intérieur du conteneur
(`podman network inspect ... Internal`) — elle couvre par construction
tout vecteur de sortie non identifié par nom, y compris les variables
au comportement non confirmé du § 8.1. `OLLAMA_NO_CLOUD=1` (voie A) reste un complément
utile : elle empêche la **tentative** elle-même (pas seulement son
échec), donc pas de bruit de journal répété (§ 8.3) — mais elle ne
serait pas suffisante seule, puisqu'elle ne couvre qu'un nom de
variable déjà connu aujourd'hui.

### 8.3 — Preuve : trafic sortant réel du service de production, avant/après, et démonstration inverse

**Avant IA-2 (déjà consigné § 7.3, revérifié ici par horodatage de
fichier plutôt que recopié)** :
```
$ stat --format '%Y %s' <volume>/cache/model-recommendations.json
1786181179 682   # écrit le 2026-08-07 12:46:19, avant tout changement IA-2
```
**Après IA-2 (réseau interne + `OLLAMA_NO_CLOUD=1` déployés,
service redémarré deux fois depuis, à 14:08 et 16:08)** :
```
$ stat --format '%y' <volume>/cache/model-recommendations.json
2026-08-07 12:46:19.007757502 +0200   # mtime INCHANGÉ malgré deux redémarrages
$ podman logs ollama | grep -iE 'ollama.com|recommendation|dial'
(rien)
```
Le fichier n'a **plus jamais été réécrit** depuis IA-2, malgré deux
redémarrages du service qui, avant IA-2, déclenchaient chacun une
tentative (§ 7.3 : `consecutive_failures=0`, cache réécrit à chaque
démarrage).

**Démonstration inverse — la mesure aurait détecté le trafic s'il avait
eu lieu, prouvé en le provoquant réellement, sans jamais quitter la
machine** : `OLLAMA_NO_CLOUD` remis à `0` par `-e` sur le rôle
(réseau interne **inchangé**) :
```
$ ansible-playbook roles/local_ai/local_ai.yml -e local_ai_ollama_no_cloud=0 -e local_ai_cloud_disabled_expected=false
$ podman logs ollama | tail
msg="model recommendations refresh failed" error="dial tcp: lookup ollama.com on 10.89.0.1:53: no such host"
msg="model show cloud cache hydration failed" error="dial tcp: lookup ollama.com on 10.89.0.1:53: no such host"
$ stat --format '%y' <volume>/cache/model-recommendations.json
2026-08-07 12:46:19.007757502 +0200   # toujours inchangé — la tentative a échoué avant tout accès réseau réel
```
**La tentative a bien eu lieu** (la variable seule ne l'empêchait plus)
**et a été bloquée avant de quitter la machine** (résolution DNS
échouée sur le résolveur interne du réseau `--internal`, jamais un
résolveur externe) — la mesure (journal + horodatage de cache) est
donc sensible : elle voit la différence entre « aucune tentative » et
« tentative bloquée ». État nominal restauré immédiatement après
(`local_ai_ollama_no_cloud=1` par défaut, rejoué, `changed=2`
[conteneur + garde], service à nouveau conforme, § 8.4).

### 8.4 — Garde dans le rôle, démonstration dans les deux sens

Deux gardes a posteriori ajoutées (`roles/local_ai/tasks/main.yml`,
« Garde 4/5 » et « Garde 5/5 »), échec bruyant, valeurs attendues
substituables par `-e` sans jamais changer ce qui est réellement
déployé (même patron que les trois gardes préalables déjà en place) :

- **Garde réseau** : `podman network inspect systemd-ollama --format
  {{.Internal}}` comparé à `local_ai_network_internal_expected`
  (défaut `true`).
- **Garde cloud** : le propre journal du serveur (`podman logs ollama`)
  doit contenir `msg="Ollama cloud disabled: true"` — **effet constaté
  par le serveur lui-même**, pas la variable qu'on lui a posée. Défaut
  trouvé et corrigé en écrivant cette garde : `podman logs` relaie le
  flux du conteneur sur **son propre stderr**, jamais son stdout —
  vérifié par un test Python isolé avant correction (
  `subprocess.run(capture_output=True)`, `stderr` contenait le message,
  `stdout` non), la garde lit désormais les deux flux.

**Nominal** : `ansible-playbook roles/local_ai/local_ai.yml` → les deux
gardes `ok`, `changed=0` en deuxième exécution consécutive.
**Échec forcé, deux fois, chacune restaurée** :
```
$ ansible-playbook roles/local_ai/local_ai.yml -e local_ai_network_internal_expected=false
fatal: [...] "Réseau systemd-ollama : Internal attendu = 'false', relevé = 'true'. [...]"
$ ansible-playbook roles/local_ai/local_ai.yml -e local_ai_cloud_disabled_expected=false
fatal: [...] "Le journal du service n'annonce pas 'Ollama cloud disabled: false' [...]"
```
Aucun des deux essais n'a modifié le système réel (`changed=0` sur les
deux, le premier échoue avant toute écriture de service ; le second
étudié § 8.3 ci-dessus, `changed=2`, restauré par une exécution
nominale immédiatement rejouée).

**Défaut corrigé en cours de route, sans rapport avec les gardes** :
`ansible.builtin.systemd: state: started` est un no-op sur un service
déjà actif — la première exécution IA-2 a écrit les nouvelles unités
Quadlet (`changed`) sans jamais redémarrer le conteneur pour les
appliquer, laissant tourner l'ancienne définition (ancien réseau,
sans `OLLAMA_NO_CLOUD`) pendant qu'un premier passage des nouvelles
gardes échouait, à raison, sur cet écart. Corrigé : `state` vaut
désormais `restarted` quand l'une des trois unités (réseau, volume,
conteneur) a changé, `started` sinon — idempotence revérifiée après
correction (`changed=0` en deuxième exécution, § validation).

### 8.5 — Coût pour la récupération de modèles, nommé, pas résolu

Un réseau `--internal` bloque la résolution DNS et toute route de
sortie — `ollama pull` échouerait pour la même raison que
`ollama.com` (§ 8.3). **Non résolu dans ce livrable** (aucun modèle
téléchargé). Chemin distinct identifié, pas implémenté — natif à
Podman, pas un mécanisme à inventer (CLAUDE.md § déterminer ce que
l'outil gère déjà) :
```
$ podman network connect <réseau-avec-sortie> ollama   # temporaire, pour la durée du pull
$ podman network disconnect <réseau-avec-sortie> ollama # revient à l'état confiné
```
Podman permet d'attacher un second réseau à un conteneur déjà en cours
d'exécution — le réseau interne resterait la voie par défaut, un accès
sortant ne serait ouvert que le temps d'un `ollama pull` délibéré, puis
retiré. Alternative plus lourde, nommée pour mémoire : basculer
temporairement `Network=` du fichier `.container` vers le réseau
`pasta` par défaut, redémarrer, tirer le modèle, revenir en arrière —
manuel, visible, pas automatisé par ce rôle.

### 8.6 — Partie B : faisabilité de la complétion dans Helix/Kate (lecture seule)

**Question posée : D15 (modèle de complétion résident) suppose que la
complétion arrive dans l'éditeur — est-ce vrai ?**

**Helix n'a pas de système de greffons.** Établi par lecture de
discussions officielles du projet (source externe, nommée) :
[helix-editor/helix discussion #10131](https://github.com/helix-editor/helix/discussions/10131),
[#8887](https://github.com/helix-editor/helix/issues/8887) — « Helix
does not yet have a plugin system ». Le seul point d'extension
reconnu par l'éditeur est le protocole LSP standard (client déjà
présent, `docs/editor.md` § 2026-08-06) ; l'extension LSP 3.18
`inlineCompletionProvider` (le mécanisme qui permettrait un vrai
« texte fantôme » IA comme dans les éditeurs à greffons) est discutée
mais **non implémentée** dans les versions couvertes par ces sources.

**`llm-ls` (Hugging Face) — le candidat binaire hors npm le plus
sérieux, écarté après lecture de son code, pas de sa réputation.**
Établi par lecture directe du dépôt amont (source externe, nommée) :
[github.com/huggingface/llm-ls](https://github.com/huggingface/llm-ls),
licence Apache-2.0, distribué par binaires précompilés (`Assets`
attachés à chaque publication, ex. 0.5.3) **et** par `cargo install` —
aucune dépendance npm, respecte D11(D12). Compatible d'un serveur
Ollama comme moteur d'inférence, capable de remplissage au milieu
(FIM). **Mais** — lu directement dans
[`crates/llm-ls/src/main.rs`](https://github.com/huggingface/llm-ls/blob/main/crates/llm-ls/src/main.rs) :
```
.custom_method("llm-ls/getCompletions", LlmService::get_completions)
.custom_method("llm-ls/acceptCompletion", ...)
.custom_method("llm-ls/rejectCompletion", ...)
```
**`llm-ls` n'implémente pas `textDocument/completion` standard** — la
complétion passe par une méthode JSON-RPC **propriétaire**,
`llm-ls/getCompletions`, qu'un client LSP générique n'a aucune raison
d'appeler. Seules les extensions dédiées officiellement listées
(`llm.nvim`, `llm-vscode`, `llm-intellij`, `jupytercoder` — le README
amont ne mentionne ni Helix ni Kate) savent l'invoquer, parce
qu'elles portent chacune le code de scripting nécessaire. **Ni Helix
(pas de greffons) ni le greffon client LSP de Kate (générique, pas
scriptable pour des méthodes propriétaires — établi par lecture de
[kate-editor.org/posts/kate-language-server-protocol-client](https://kate-editor.org/posts/kate-language-server-protocol-client/),
aucune mention de méthodes personnalisées) ne peuvent l'appeler.**
`llm-ls`, malgré son nom, n'est donc pas un serveur de langage
générique utilisable par n'importe quel client LSP — c'est un serveur
compagnon d'extensions spécifiques.

**Aucun autre serveur de langage packagé pour le remplissage au milieu
n'a été trouvé** dans les dépôts activés (`docs/repositories.md` § 4 ;
recherche par nom et par résumé, `dnf repoquery`, aucune correspondance sur `llm-ls`,
`tabby`, `continue`, `twinny`, `codeium`, en dehors de faux positifs
sans rapport — paquets Haskell `copilot`, polices).

**Mécanisme natif restant, sans greffon ni serveur LSP dédié** : les
deux éditeurs savent invoquer une commande shell arbitraire et insérer
sa sortie — `:pipe`/`:pipe-append` pour Helix (capacité native,
`docs/editor.md`), le greffon `externaltoolsplugin` déjà **activé**
pour Kate (`docs/editor.md` § 2026-08-06). Une commande `curl` vers
l'API locale d'Ollama (`/api/generate`, déjà en place, § 3.2) suffit à
obtenir une complétion FIM à la demande — respecte D11 (aucun greffon,
aucun npm), fonctionne dès aujourd'hui sans rien installer de plus. Ce
n'est **pas** une complétion automatique au fil de la frappe : c'est un
déclenchement manuel, ponctuel, plus proche d'un remplissage de
gabarit que d'une expérience de complétion.

**Conclusion, formulée sans détour : troisième issue.** Un mécanisme
existe (`:pipe-append`/outil externe → Ollama), il ne viole pas D11 —
mais il ne réalise pas ce que D15 justifiait. Le motif exact de D15
(« latence minimale... le serveur doit rester un processus persistant,
**jamais relancé par requête** — un serveur relancé à chaque
complétion paierait le réveil de 20-23 s à chaque frappe ») suppose
une complétion **automatique, au fil de la frappe** — c'est
précisément ce qu'aucun des deux éditeurs ne peut offrir sans greffon
ni serveur LSP standard, structurellement absents ici (Helix : pas de
greffons ; Kate : greffon LSP générique, `llm-ls` inutilisable en
dehors de ses extensions officielles). Un déclenchement manuel,
espacé de plusieurs minutes, tolère très bien un réveil RTD3 occasionnel
de 20-23 s — l'argument qui justifiait la résidence permanente (D15)
ne tient plus à cette cadence d'usage. **D14 (Ollama conteneurisé,
CDI) reste valide** — l'infrastructure sert aussi le rôle chat/agent,
indépendant de cette question. **C'est D15, précisément, qui doit être
révisée** : la contrepartie qu'elle assume (RTD3 neutralisée, ~8 W en
continu, § 7.6) n'a plus la justification qui l'a motivée tant
qu'aucune complétion automatique n'est réalisable dans l'éditeur. Dit
franchement, pas contourné : à l'opérateur de trancher entre revenir à
un `keep_alive` fini pour le modèle de complétion (§ 8.7 ci-dessous),
ou accepter consciemment le coût pour un usage qui restera manuel et
peu fréquent.

### 8.7 — Partie C : candidats de modèles, lecture seule, aucun téléchargement

**Contrainte technique posée avant tout choix, comme demandé** : la
complétion de code exige un modèle **FIM** (remplissage au milieu,
prompt structuré avec des jetons de préfixe/suffixe/milieu) — un
modèle de conversation générique sans entraînement FIM produirait une
continuation, pas un remplissage cohérent avec le code qui suit le
curseur. Critère de sélection retenu ci-dessous, pas déduit après
coup.

**Rôle complétion — petit, FIM, bash/Python/YAML/Jinja** (source :
[ollama.com/library/qwen2.5-coder](https://ollama.com/library/qwen2.5-coder),
[tags](https://ollama.com/library/qwen2.5-coder/tags),
[qwenlm.github.io/blog/qwen2.5-coder-family](https://qwenlm.github.io/blog/qwen2.5-coder-family/)
— FIM confirmé : « Fill-in-the-Middle mode for testing » ; « more than
40 programming languages ») :

| Modèle | Taille `Q4_K_M` | Contexte | Licence | Note |
|---|---|---|---|---|
| `qwen2.5-coder:1.5b-instruct-q4_K_M` | 986 Mio | 32 K | Apache-2.0 | Le plus léger, marge VRAM maximale |
| `qwen2.5-coder:3b-instruct-q4_K_M` | 1,9 Gio | 32 K | **Qwen Research License** (non-commerciale) | **[REQUALIFIÉ le 2026-08-07, D21] Écarté** — seule taille restreinte d'une famille par ailleurs Apache-2.0, motif du dépôt public suffisant pour exclure sans lire les termes complets (D21, `docs/machine-facts.md` § Décisions). Fermé par requalification de la décision, pas par vérification effective (CLAUDE.md § un marqueur ne se retire qu'après vérification effective) : la portée exacte de cette licence reste, dans l'absolu, aussi peu déterminée qu'avant ; elle cesse seulement d'être actionnable pour ce choix précis. |
| `qwen2.5-coder:7b-instruct-q4_K_M` | 4,7 Gio | 128 K (32 K pour 0.5-3 B) | Apache-2.0 | Le plus proche de la cible « ~4 Gio » de la demande |

**Rôle chat/agent — 13 B, `Q4_K_M`** (sources :
[ollama.com/library/codellama](https://ollama.com/library/codellama),
[tags](https://ollama.com/library/codellama/tags),
[ollama.com/library/mistral-nemo:12b-instruct-2407-q4_K_M](https://ollama.com/library/mistral-nemo:12b-instruct-2407-q4_K_M),
[ollama.com/library/qwen2.5/tags](https://ollama.com/library/qwen2.5/tags)) :

| Modèle | Taille `Q4_K_M` | Contexte | Licence | Note |
|---|---|---|---|---|
| `codellama:13b-code-q4_K_M` | 7,9 Gio | 16 K | Llama 2 Community License (restrictions d'usage, pas OSI) | Seul candidat à **vraiment** 13 B ; FIM également supporté (jetons `<PRE>`/`<SUF>`/`<MID>`), pertinent si le rôle chat/agent sert aussi de repli complétion |
| `mistral-nemo:12b-instruct-2407-q4_K_M` | 7,5 Gio | 128 K | Apache-2.0 | 12 B, pas 13 B — le plus proche par la licence la plus permissive |
| `qwen2.5:14b-instruct-q4_K_M` | 9,0 Gio | 32 K | Apache-2.0 | 14 B, au-dessus de la cible — poids seul déjà proche du plafond avant tout cache KV |

**Coût du cache d'attention — calculé, pas la seule taille des poids,
comme demandé.** Formule standard (attention multi-têtes groupées) :
`octets = 2 (K+V) × couches × têtes_KV × dim_tête × contexte ×
octets/élément`. Paramètres d'architecture lus directement dans le
`config.json` amont de chaque modèle (source externe, nommée) — pas
mémorisés :

- **Qwen2.5-Coder-7B** ([config.json](https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct/raw/main/config.json)) :
  28 couches, 4 têtes KV, dim_tête 128 → **56 Kio/jeton** (fp16, défaut
  Ollama constaté : `OLLAMA_KV_CACHE_TYPE` vide dans le journal de
  production, § 8.1, donc f16 non quantifié). À 4096 jetons (contexte
  par défaut mesuré sur ce poste, § 7.4 : `default_num_ctx=4096`) :
  **≈224 Mio**. À 32 K (plafond du modèle) : **≈1,75 Gio** — poids +
  cache passent de 4,7 Gio à **≈6,45 Gio** au contexte maximal.
- **Mistral-Nemo-12B** ([config.json](https://huggingface.co/mistralai/Mistral-Nemo-Instruct-2407/raw/main/config.json)) :
  40 couches, 8 têtes KV, dim_tête 128 → **160 Kio/jeton**. À 4096
  jetons : **≈625 Mio**. À 32 K : **≈5 Gio** — poids + cache passent de
  7,5 Gio à **≈12,5 Gio**, une fraction significative de l'enveloppe
  mesurée de ≈15,9 Gio (IA-0 § 2.3), **avant même de compter le modèle
  de complétion résident à côté**.

**L'arithmétique de coexistence d'IA-0 (~4 Gio + ~7,4 Gio sous
~15,9 Gio) reste non vérifiée, et ce calcul montre pourquoi elle ne
peut pas être acceptée telle quelle** : à contexte modeste (4096), la
marge est large (≈0,85 Gio de cache cumulé pour les deux rôles) ; à
contexte élevé pour le rôle chat/agent (32 K, plausible pour une tâche
« contexte long » — IA-0 § 1), le cache seul du modèle chat/agent
(≈5 Gio) dépasse la marge que l'arithmétique de poids seuls avait
laissée. **[RÉSOLU le 2026-08-07, IA-3, § 9.3] ÉTAIT** un marqueur de
vérification ouvert (chargement simultané à contexte représentatif non
mesuré) — **mesuré réellement** avec les deux modèles retenus (D21,
mistral-nemo:12b à 32 K + qwen2.5-coder:7b) : la coexistence ne tient
pas, confirmant exactement ce que ce calcul anticipait. Détail complet
§ 9.3, chiffres réels contre estimation ici.

**Questions qui départagent — aucun modèle choisi :**

1. **D15 doit-elle être révisée maintenant (§ 8.6) ?** Si la complétion
   reste manuelle (`:pipe-append`/outil externe), un `keep_alive` fini
   (ex. quelques minutes) redevient défendable — cela change le
   dimensionnement VRAM disponible pour le rôle chat/agent, puisque le
   modèle de complétion ne resterait plus chargé en permanence.
2. **Le modèle chat/agent doit-il rester à 13 B strict
   (`codellama:13b`, licence Llama 2 restrictive) ou une taille voisine
   sous licence plus permissive (`mistral-nemo:12b`, Apache-2.0) est-elle
   acceptable pour un dépôt public (D4) ?**
3. **Quelle longueur de contexte réellement visée pour le rôle
   chat/agent ?** Le calcul ci-dessus montre que c'est ce chiffre, pas
   la taille du modèle, qui décide si la coexistence tient — 4096
   (large marge) et 32 K (marge consommée par le cache seul) ne sont
   pas la même décision.
4. **`qwen2.5-coder:3b`, seule taille de la famille sous licence
   restreinte (Qwen Research License), doit-elle être exclue d'office
   d'un dépôt public, ou sa lecture complète (marqueur de vérification
   ci-dessus) change-t-elle cette réponse ?**
5. **Le chemin de récupération de modèles à travers le réseau interne
   (§ 8.5, `podman network connect` temporaire) convient-il, ou une
   voie plus automatisée doit-elle être conçue avant le prochain
   livrable qui télécharge un modèle ?**

## Validation — IA-2 (2026-08-07)

**Actions privilégiées, exhaustives** — aucune nouvelle par rapport à
IA-1, les mêmes s'exécutent à chaque exécution du rôle (déploiement de
la garde D16, sans changement de contenu depuis IA-1, donc `ok`, jamais
`changed` dans cette série) :

| # | Commande | Cible | Motif |
|---|---|---|---|
| 1 | `ansible.builtin.template` (`become: true`), rejouée à chaque exécution de ce livrable (6 exécutions réelles, 2 démonstrations d'échec forcé sur les nouvelles gardes, 1 démonstration inverse) | `/etc/modprobe.d/local-ai-nvidia-power-management.conf` | D16, inchangé — seule tâche `become: true` du rôle, contenu identique à IA-1 à chaque relecture |
| 2 | `sudo -n ausearch -m avc -ts recent` (hors rôle, validation manuelle) | lecture `/var/log/audit/` | corroborer l'absence de refus AVC, même méthode qu'IA-1 — silencieuse sous D9 (NOPASSWD), disclosed ici |

Toutes les autres actions (déploiement du réseau/volume/conteneur
Quadlet, les cinq gardes, `podman network inspect`, `podman logs`,
`podman cp` du binaire vers l'hôte pour `strings`, requêtes `curl`)
s'exécutent sans `sudo`.

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/local_ai/local_ai.yml   # succès
$ ansible-playbook --check roles/local_ai/local_ai.yml          # succès, changed=0
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=0 (état déjà nominal après convergence, § 8.4)
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=0 (deuxième exécution consécutive, idempotence confirmée)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/local_ai/
Passed: 0 failure(s), 0 warning(s) — profil production
```

**Les quatre démonstrations d'échec forcé** (deux d'IA-1, revérifiées
implicitement par la structure inchangée des gardes 1-3 ; deux
nouvelles d'IA-2, § 8.4), toutes `changed=0` sauf la démonstration
inverse (§ 8.3, `changed=2`, restaurée par une exécution nominale
immédiatement rejouée, revérifiée `changed=0`) :
```
$ ansible-playbook roles/local_ai/local_ai.yml -e local_ai_network_internal_expected=false
fatal: [...] "Réseau systemd-ollama : Internal attendu = 'false', relevé = 'true'. [...]"
$ ansible-playbook roles/local_ai/local_ai.yml -e local_ai_cloud_disabled_expected=false
fatal: [...] "Le journal du service n'annonce pas 'Ollama cloud disabled: false' [...]"
```
Service restauré à l'état nominal après chaque essai, aucun n'a laissé
le système dans un état de test.

**Preuve que l'API n'écoute pas sur une interface externe (inchangé)** :
```
$ ss -H -tln sport = :11434
LISTEN 0 4096 127.0.0.1:11434 0.0.0.0:*
```

**SELinux** :
```
$ getenforce                                  # Enforcing, avant et après ce livrable
$ journalctl -k -g 'avc:' --no-pager -n 20     # -- No entries --
$ sudo -n ausearch -m avc -ts recent           # <no matches>
```

**Aucun modèle téléchargé** : `curl http://127.0.0.1:11434/api/tags` →
`{"models":[]}` ; volume `systemd-ollama-models` toujours à 12 Kio
(clés d'identité + cache, inchangé depuis IA-1 — le cache de
recommandations n'a plus été réécrit depuis IA-2, § 8.3) ; `podman
images` inchangé (les deux mêmes images qu'IA-1, aucune couche
supplémentaire tirée — `podman cp` copie un fichier déjà présent
localement, ne tire rien).

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : huit
marqueurs actionnables dans ce document à la clôture d'IA-2 — les cinq
déjà comptés en IA-1 (§ 2.1 suspension S0ix, § 1 chargement simultané,
§ 3.1 comportement ROCm, § 5 comportement `ollama pull`, § 2.1bis
signification de `2`), **plus trois nouveaux** (§ 8.1 sémantique des
variables non documentées trouvées par extraction de chaînes ; § 8.7
texte complet de la Qwen Research License ; § 8.7 reformulation, avec
un chiffre de cache, du marqueur de coexistence déjà ouvert en IA-0).
Aucun marqueur fermé dans cette série — celui sur la suspension S0ix
(§ 2.1) reste ouvert malgré D16 tranché en amont de sa mesure
(rappelé § 2.1bis) ; celui sur la coexistence est reformulé, pas
résolu.

**Confirmations finales** : aucun modèle téléchargé, sous aucune forme ;
aucun paquet installé ; aucun dépôt ajouté (le réseau Podman dédié
n'est pas un dépôt) ; `/usr/lib/modprobe.d/nvidia-power-management.conf`
intact (non revérifié par somme de contrôle dans ce livrable au-delà de
la garde `rpm -Vf` déjà intégrée au rôle, rejouée à chaque exécution) ;
`sudoers`, `gpu_mux_mode`, `dgpu_disable`, `kwinrulesrc` non touchés ;
SELinux jamais modifié (`Enforcing` inchangé, aucun `setenforce`) ;
aucun redémarrage de la machine, aucune déconnexion de session.

## 9. IA-3 — récupération des modèles, mesure de l'enveloppe réelle (2026-08-07)

**D21** (`docs/machine-facts.md` § Décisions) : chat/agent =
`mistral-nemo:12b-instruct-2407-q4_K_M` (Apache-2.0, contexte visé
32 K) ; complétion = `qwen2.5-coder:7b-instruct-q4_K_M` (Apache-2.0,
FIM confirmé) ; `qwen2.5-coder:3b` écarté (marqueur requalifié § 1,
plus haut dans ce document). **Ce livrable mesure, il ne force pas la
coexistence** — conformément à la demande, l'arithmétique annonçait un
dépassement (~17,4 Gio pour ~15,9 Gio mesurés) et la mesure le
confirme (§ 9.3).

### 9.1 — Récupération par conteneur de récupération éphémère

**Voie retenue, argumentée face à l'alternative** : `podman network
connect` temporaire sur le conteneur de service (proposé § 8.5)
affaiblit la propriété que le confinement réseau établit — pendant la
connexion temporaire, l'isolation cesse d'être un fait structurel
vérifiable et redevient une question de discipline opérationnelle
(« ne pas oublier de déconnecter »). Retenu à la place : un **second**
conteneur, jamais démarré par défaut (`roles/local_ai/`, tag
`pull-models` + `never` — `ansible-playbook ... --tags
pull-models,untagged` requis explicitement), qui partage le volume de
modèles avec le service mais **jamais son réseau** — aucun `Network=`
passé à `podman run`, donc le réseau rootless par défaut de Podman
(pasta, avec sortie) s'applique, entièrement disjoint de
`systemd-ollama`. Le conteneur de service ne reçoit ainsi jamais
d'accès externe, à aucun moment.

**Preuve, par inspection, pas par affirmation** — état réseau du
conteneur de service relevé à quatre moments distincts (avant tout
démarrage du conteneur de récupération, pendant qu'il tourne, après la
récupération des deux modèles, après son arrêt), tâche Ansible dédiée,
échec bruyant si l'un des quatre diffère :
```
$ podman inspect ollama --format '{{json .NetworkSettings.Networks}}'
{"systemd-ollama":{"Gateway":"10.89.0.1","IPAddress":"10.89.0.4",...}}
```
**Identique aux quatre relevés**, dans les deux exécutions réelles de
ce livrable — le conteneur de service n'a jamais été attaché au réseau
par défaut, jamais eu de route de sortie, à aucun moment de la
récupération.

### 9.2 — Récupération et intégrité

**Manifestes, empreintes par blob** (format registre de distribution
OCI/Docker standard, lu directement, pas déduit) :

| Modèle | Empreinte du blob de poids | Taille (octets, manifeste) | Config |
|---|---|---|---|
| `mistral-nemo:12b-instruct-2407-q4_K_M` | `sha256:dd3af152229f92a3d61f3f115217c9c72f4b94d8be6778156dab23f894703c28` | 7 477 204 672 (6,964 GiB — le pilleur d'IA-2 disait « 7,5 Gio », en réalité 7,5 **GB décimaux**, une confusion Gio/GB, § 9.3) | `sha256:4439d5cfc094433a522f8d00558da759ff0aec62909c5b0ba6f14d99f9068fe9` |
| `qwen2.5-coder:7b-instruct-q4_K_M` | `sha256:60e05f2100071479f596b964f89f510f057ce397ea22f2833a0cfe029bfc2463` | 4 683 074 048 (4,362 GiB) | `sha256:d9bb33f2786931fea42f50936a2424818aa2f14500638af2f01861eb2c8fb446` |

**Emplacement** : `~/.local/share/containers/storage/volumes/systemd-ollama-models/_data/models/{manifests,blobs}/` — volume Podman nommé, sur le btrfs `Data,single` (D1), jamais versionné (D4 : taille seule suffit).

**Mécanisme d'intégrité, vérifié par un test réel, pas supposé** —
détail complet et écart résiduel assumé : `docs/repositories.md` § 8.
Résumé : `ollama pull` annonce et effectue une vérification SHA-256 du
contenu **reçu sur le réseau** (« verifying sha256 digest ») ; un blob
**déjà présent localement**, corrompu après coup sans changer de nom,
**n'est ni détecté ni réparé** par un nouveau `pull` — démontré par
corruption délibérée d'un octet, restauration nécessitant la
suppression du fichier avant nouveau `pull`. Les dix blobs des deux
modèles, revérifiés un par un après ce test (`sha256sum` de chaque
fichier comparé à son propre nom) : tous corrects.

**Espace disque, avant/après** (`btrfs filesystem usage /`, sans
privilège) :
```
Avant : Device allocated: 16.02GiB   (/home : 13G utilisés, df -h)
Après : Device allocated: 27.02GiB   (/home : 24G utilisés, df -h)
```
**Δ ≈ 11 GiB**, cohérent avec 6,964 + 4,362 = 11,33 GiB de poids
(le reste : manifestes, métadonnées, clés d'identité déjà présentes).

**D1, rappel** : ces poids ne sont pas versionnés dans ce dépôt — leur
reconstructibilité repose entièrement sur le registre Ollama (nommé,
`docs/repositories.md` § 8) et sur les deux identifiants consignés ici
(nom de modèle + empreinte de blob), pas sur une copie locale
sauvegardée par ce dépôt.

### 9.3 — Mesures VRAM, temps, latence, débit, RTD3

**Méthode d'isolement** (`docs/dgpu-power.md`) appliquée à chaque
relevé : une sonde `nvidia-smi` unique et délibérée par mesure, effet
de sonde nommé (elle réveille le GPU — sans conséquence sur le chiffre
de VRAM lui-même, qui reflète l'état déjà en place). Chargement/
déchargement des modèles par l'API `/api/generate` (`keep_alive`
explicite), pas par `ollama run` (mode interactif, moins contrôlable).

**1. Chat seul, 32 K** — sonde après chargement :
```
memory.used : 12349 MiB total (llama-server : 12294 MiB, kwin_wayland : 13 MiB, reste : réserve pilote)
```
**Écart à l'estimation d'IA-2 (12,5 Gio), expliqué dans les deux
sens** : mesuré 12294 MiB = **12,01 GiB**, environ 0,5 GiB **en
dessous** de l'estimation. Décomposé : poids réels 7 130,8 MiB
(6,96 GiB, § 9.2) + cache KV déduit 5 163,2 MiB (5,04 GiB). Or la
formule de cache KV d'IA-2 prédisait **5,00 GiB** — un écart de moins
de 1 %, la formule était juste. L'écart total vient entièrement d'une
confusion Gio/GB sur les **poids** : IA-2 citait « 7,5 Gio » en
reprenant l'affichage `ollama` (« 7.5 GB », décimal) sans convertir —
7,5 GB décimaux = 6,98 GiB, pas 7,5 GiB. Corrigé ici par mesure
directe.

**2. Complétion seule, contexte 2000** (celui que `roles/completion/`
configure réellement pour `lsp-ai`, pas un contexte arbitraire) :
```
memory.used : 4745 MiB total (llama-server : 4690 MiB)
```
Poids réels 4 362 MiB (4,26 GiB) + cache KV déduit ≈328 MiB — plus
que la formule ne le prédirait pour 2000 jetons seuls (~107 MiB),
écart non creusé davantage ici (buffers de calcul / overhead
d'allocation par défaut probablement inclus, formule KV pure ne les
couvre pas — nommé, pas expliqué en détail).

**3. Les deux ensemble** : **la coexistence ne tient pas — mesuré, pas
supposé.** Charger le modèle chat pendant que la complétion est
chargée **décharge la complétion** (`/api/ps` ne montre plus qu'un
seul modèle après coup, un seul processus `llama-server` visible dans
`nvidia-smi --query-compute-apps`) ; l'inverse est également vrai
(charger la complétion décharge le chat). Testé dans les deux sens,
confirmé les deux fois. **C'est le résultat de la mesure, pas un échec
de la mesure** — exactement ce que l'arithmétique de la demande
anticipait (~17,4 Gio pour ~15,9 Gio mesurés).

**4. Temps de chargement, à froid puis à chaud** :

| Modèle | Froid (`load_duration`) | Chaud (déchargé puis rechargé, `load_duration`) |
|---|---|---|
| Chat (mistral-nemo, 32 K) | 16,19 s | 2,85 s |
| Complétion (qwen2.5-coder, 2000) | 7,60 s | — (non remesuré séparément, voir bascule § 9.4) |

**5. Latence du premier jeton et débit, modèle déjà chargé** (chat,
prompt de 18 jetons, 141 jetons générés) :
```
latence avant premier jeton (load+prompt_eval, déjà résident) : 0,176 s
débit en régime établi : 141 jetons / 2,167 s = 65,05 jetons/s
```

**6. Veille runtime (RTD3), modèle chargé, lecture passive uniquement**
(aucune sonde `nvidia-smi` pendant la fenêtre — même méthode que
`docs/local-ai.md` § 7.6) :
```
T0  runtime_status=active  runtime_suspended_time=36016112  runtime_active_time=1144417
[60 s, aucune sonde, complétion chargée, contexte CUDA ouvert]
T1  runtime_status=active  runtime_suspended_time=36016112 (inchangé)  runtime_active_time=1204421 (Δ=+60004 ms ≈ Δt réel)
```
**Confirmé, pas seulement plausible** : le contexte CUDA ouvert d'un
modèle chargé bloque RTD3 en continu — `runtime_suspended_time` figé,
`runtime_active_time` progresse exactement au rythme du temps écoulé.
Ferme le marqueur ouvert en IA-0 § 2.1 sur ce point précis (l'effet
d'une suspension **système**, distinct, reste ouvert — non testé ici,
aucune suspension déclenchée).

**Consommation, modèle chargé mais au repos (60 s+ sans requête),
sonde unique** :
```
power.draw : 3,49 W   pstate : P8   utilization.gpu : 0 %
```
**Bien en dessous** des ~33-55 W relevés pendant une sonde ou un
chargement actif (état transitoire, pas la consommation stable) — un
modèle chargé et inactif se stabilise en état d'alimentation bas
(P8), malgré RTD3 bloqué. Nuance à distinguer du chiffre « ~8 W » de
D15/D20 (économie de la bascule MUX, un mécanisme différent) — pas
directement comparable, nommé pour ne pas laisser croire à une
équivalence.

### 9.4 — Trois leviers, chiffrés, aucun recommandé

| Levier | Configuration de référence (f16, 32 K) | Avec le levier | VRAM économisée |
|---|---|---|---|
| `OLLAMA_KV_CACHE_TYPE=q8_0` | 12294 MiB | 9946 MiB (mesuré, conteneur de mesure distinct, jamais le service réel) | **2348 MiB (2,29 GiB)** — perte de qualité **non chiffrée**, nommée comme telle, pas évaluée ici |
| Contexte réduit à 16 K | 12294 MiB (32 K) | 9854 MiB (mesuré) | **2440 MiB (2,38 GiB)** |
| Chargement séquentiel (bascule) | — | complétion→chat : 4,11 s ; chat→complétion : 3,42 s (`load_duration`, les deux `déjà en cache disque OS`) | coût par bascule, pas de VRAM à demeure, les deux modèles cohabitent sur disque (§ 9.2) sans jamais cohabiter en VRAM |

**Aucun des trois n'est recommandé ici** — chiffrés pour que
l'opérateur tranche : le premier réduit la VRAM sans réduire le
contexte mais au prix d'une qualité non mesurée ; le second réduit la
VRAM en réduisant directement l'usage possible (32 K → 16 K de fenêtre
de contexte réelle pour le chat) ; le troisième n'économise aucune
VRAM à demeure mais paie quelques secondes par changement d'usage, sans
jamais dépasser l'enveloppe.

### 9.5 — Bout-en-bout : `lsp-ai`/Helix, cause distinguée avant conclusion

CMP-1 est en place (`~/.local/bin/lsp-ai`, `~/.config/helix/languages.toml`)
— testé réellement, `hx -vv` piloté sans capture visuelle (même méthode
que `docs/completion.md` § 7.4), fichier YAML ouvert, complétion
déclenchée automatiquement à la frappe.

**La poignée de main LSP fonctionne** (confirmé de nouveau) et Helix
envoie bien des requêtes `textDocument/completion` automatiques
pendant la frappe (`triggerKind:1`, observé deux fois). **La
complétion échoue**, mais pour une **troisième cause**, distincte des
deux anticipées par la demande (politique `lsp-ai` défaillante, ou
réveil RTD3 + rechargement) :
```
lsp-ai err <- "ERROR lsp_ai::transformer_worker: generating response:
  making Ollama completions request: \"model 'aucun-modele-charge-D20' not found\""
```
**Ni `lsp-ai` ni RTD3/la politique de rechargement n'est en cause** —
`roles/completion/` (CMP-1) pointe délibérément un nom de modèle
placeholder, non résolvable par construction (motif exact,
`docs/completion.md` § doc de `templates/languages.toml.j2` : « pour
que la complétion échoue de façon visible plutôt que de sembler
fonctionner par coïncidence »). Ollama répond une erreur claire et
immédiate (pas de latence de 20 s, pas de timeout) — le mécanisme
entier, jusqu'à l'appel réseau vers Ollama, fonctionne. **Corriger le
nom de modèle dans `roles/completion/` est hors du périmètre strict de
ce livrable** (`roles/local_ai/`, `docs/`, le volume de modèles
uniquement) — signalé ici pour un livrable qui aurait ce mandat, pas
corrigé.

## Validation — IA-3 (2026-08-07)

**Actions privilégiées : aucune, dans ce livrable.** Ni la récupération
des modèles (conteneur rootless, volume utilisateur), ni les mesures
(API HTTP locale, `nvidia-smi`, lectures `/sys`/`/proc`), ni le test
d'intégrité (lecture/écriture dans un volume utilisateur) n'ont
demandé de `sudo`. Seule action `become: true` du rôle,
**pré-existante, inchangée** : le gabarit modprobe.d (D16), rejoué à
chaque exécution de cette série sans modification de contenu (`rpm -Vf`
identique avant/après, comme toujours). **Note honnête sur la règle
0.3** (CLAUDE.md, ajoutée en CMP-1) : cette tâche pré-existante
n'implémente pas la tentative sans privilège préalable que la règle
demande désormais — pas corrigée dans ce livrable (hors périmètre
annoncé), signalée plutôt que laissée paraître conforme par silence.

**Défaut trouvé et corrigé en écrivant ce livrable** : la Garde 5/5
(annonce cloud) lit `podman logs` — après un usage réel du service
(générations réelles pour les mesures), ce journal contient le vidage
brut du vocabulaire GGUF par `llama.cpp`, des séquences d'octets
valides pour un tokeniseur par octets mais pas toujours valides comme
UTF-8 une fois tronquées par le journal, faisant échouer le module
`command` (« Refusing to deserialize an invalid UTF8 string »).
Corrigé par `iconv -f utf-8 -t utf-8 -c`, contenu utile préservé,
vérifié. Les deux démonstrations d'échec forcé de cette garde
rejouées après correction (CLAUDE.md § une garde modifiée perd la
démonstration qui la validait).

**Validation Ansible** :
```
$ ansible-playbook --syntax-check roles/local_ai/local_ai.yml   # succès
$ ansible-playbook --check roles/local_ai/local_ai.yml          # succès, changed=0
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=0 (état déjà nominal)
$ ansible-playbook roles/local_ai/local_ai.yml                  # succès, changed=0 (deuxième exécution consécutive)
$ ansible-playbook roles/local_ai/local_ai.yml --tags pull-models,untagged  # changed=2 (conteneur éphémère, attendu — recréé à chaque fois)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/local_ai/
Passed: 0 failure(s), 0 warning(s) — profil production
```

**Isolation réseau du conteneur de service, prouvée par inspection** —
§ 9.1 : identique aux quatre relevés (avant/pendant/après récupération,
après arrêt du conteneur de récupération), sur les deux exécutions
réelles de la récupération dans ce livrable.

**SELinux** :
```
$ getenforce                                # Enforcing, avant et après ce livrable
$ journalctl -k -g 'avc:' --no-pager -n 5   # -- No entries --
```

**Intégrité, testée pas supposée** — § 9.2 : `ollama pull` vérifie le
contenu reçu sur le réseau (« verifying sha256 digest »), mais ne
détecte pas un blob déjà présent corrompu après coup ; les dix blobs
des deux modèles, revérifiés un par un après ce test, sont tous
corrects.

**Confirmations finales** : les deux modèles nommés par D21, et
seulement ceux-là, sont présents (`ollama list`) ; aucun autre modèle ;
`/usr/lib/modprobe.d/nvidia-power-management.conf` intact ; aucun
nouveau dépôt système ; `sudoers`/`gpu_mux_mode`/`dgpu_disable`/
`kwinrulesrc` non touchés ; SELinux jamais modifié ; aucun redémarrage ;
`command -v node npm` toujours vide.

**Décompte du jeton de vérification, `CLAUDE.md` exclu** : quatre
marqueurs actionnables à la clôture d'IA-3 (huit à la clôture d'IA-2,
un déjà fermé par requalification D21 § 1, trois fermés par mesure
réelle ce livrable — § 9.3, § 9.2) — comportement d'une suspension
**système** avec modèle chargé (toujours non testé, distinct de RTD3),
comportement runtime ROCm sans périphérique, sémantique des variables
Ollama non documentées, sémantique de la valeur `2` de
`NVreg_PreserveVideoMemoryAllocations`.

## 10. IA-4 — déclencheurs CDI, intégrité des poids, garde VRAM corrigée (2026-08-08)

**10.1 — Intégrité des blobs au repos** (§ 9.2 ci-dessus, motif complet).
Script autonome déployé (`roles/local_ai/templates/verify-model-integrity.sh.j2`
→ `~/.local/bin/verify-model-integrity.sh`, domaine utilisateur, lecture
seule stricte — aucune dépendance à Ansible une fois déployé). Recalcule
l'empreinte de chaque blob référencé par chaque manifeste et la compare
au nom du fichier. **Démontré dans les deux sens, sur ce poste, pas en
abstrait** : nominal, `OK — 10/10 blobs vérifiés` ; un octet d'un blob
réel corrompu délibérément (même méthode qu'IA-3, offset 1000, `dd`) →
`ÉCHEC — écart(s) d'intégrité trouvé(s) (9/10 blobs corrects) : CORROMPU
<manifeste> -> <digest attendu> (empreinte réelle : <digest réel>)`,
code de sortie 1 ; restauré (suppression du blob puis nouveau tirage —
la seule voie qui force `ollama pull` à revérifier, établi § 9.2) ;
`OK — 10/10` de nouveau, empreinte du fichier restauré identique à
l'originale.

**10.2 — Déclencheurs de `verify-cdi-spec`** (§ 2.4, IA-1 : le
mécanisme existait sans rien qui le déclenche hors du redémarrage du
service). Second levier ajouté : unités systemd --user **plates**
(pas Quadlet, aucun conteneur) — `local-ai-cdi-verify.service`
(`ExecStart=ansible-playbook … --tags verify-cdi-spec`), `.timer`
(`OnCalendar=hourly`, `Persistent=true`), `OnFailure=` chaîné vers
`local-ai-cdi-verify-notify.service` (`notify-send --urgency=critical`,
libnotify déjà présent). Fréquence horaire justifiée par l'absence de
`dnf-automatic` sur ce poste (vérifié, `systemctl list-unit-files`) —
les mises à jour de pilote sont manuelles, pas continues.

**Démontré dans les deux sens, unité réelle, pas simulée** : nominal
(`systemctl --user start local-ai-cdi-verify.service`) → succès,
`Spécification à jour : version 610.43.03 == version chargée
610.43.03` ; échec forcé (unité **déployée** temporairement patchée
avec `-e gpu_cdi_verify_spec_path=/tmp/spec-invalide-garantie.yaml`,
jamais le gabarit source) → `Job … failed`, code 2, message exact
(« /tmp/spec-invalide-garantie.yaml n'existe pas »), et **la chaîne
`OnFailure=` confirmée déclenchée** (`systemd[…]: … Triggering
OnFailure= dependencies`, `local-ai-cdi-verify-notify.service` :
`status=0/SUCCESS` sur le bus D-Bus réel de la session graphique).
Unité restaurée identique (`diff` vide), rejeu du rôle `changed=0`.
**Limite honnête** : `notify-send` a retourné 0 sur le bus de session
réel — le rendu visuel effectif de la bannière n'a pas été confirmé
autrement (pas de capture visuelle, cohérent avec la méthode déjà
établie pour `lsp-ai`/Helix, mais sans journal équivalent ici pour
attester le rendu à l'écran lui-même).

**10.3 — Défaut trouvé et corrigé en écrivant ce livrable, pas
anticipé** : la garde post-démarrage héritée d'IA-1
(`local_ai_ps_check.json.models == []`) supposait qu'aucun modèle ne
serait jamais chargé en VRAM au moment où ce rôle s'exécute — vrai par
coïncidence tant que le service n'avait jamais servi de complétion
réelle (IA-1 à IA-3). Le premier usage réel de ce livrable
(`docs/completion.md` § 8) charge un modèle qui reste résident
indéfiniment (D15 — le comportement voulu, pas une anomalie) : rejouer
ce rôle après un usage normal du service **échouait** sur cette garde
(constaté en rejouant le rôle après les essais de complétion, pas
anticipé en écrivant le rôle). **Corrigé** : la garde compare désormais
un relevé `/api/ps` pris **avant** toute action de ce rôle à un relevé
pris **après** — elle échoue seulement si CE rôle a fait apparaître un
modèle qui n'y était pas avant, plus jamais si un modèle est
simplement déjà résident par usage normal. **Les deux démonstrations
rejouées avec la logique corrigée** (CLAUDE.md § une garde modifiée
perd la démonstration qui la validait) : nominal (modèle déjà résident
par usage réel de ce livrable) → succès ; échec forcé (une tâche
temporaire, retirée juste après, charge délibérément un second modèle
entre le relevé « avant » et le relevé « après », simulant ce qu'un bug
de ce rôle ferait) → `Modèle(s) apparu(s) en VRAM pendant l'exécution
de ce rôle, qui n'y étaient pas avant : ['mistral-nemo:…']`, `changed=0`
dans les deux cas ; état réel restauré (modèle de démonstration
déchargé, `keep_alive:0`) et rejeu confirmé `changed=0`.

## Validation — IA-4 (2026-08-08)

**Actions privilégiées, exhaustives** :

| # | Commande | Cible | Motif | Tentative sans privilège : résultat |
|---|---|---|---|---|
| 1 | `ansible.builtin.template` (`become: true`, rôle, D16, **pré-existante, inchangée par ce livrable**) | `/etc/modprobe.d/local-ai-nvidia-power-management.conf` | seule écriture système du rôle, fichier fournisseur jamais touché (`rpm -Vf` identique avant/après) | Non applicable — écriture racine intrinsèque (`/etc/modprobe.d/`), aucune voie non privilégiée n'existe pour ce fichier |
| 2 | `sudo -n ausearch -m avc -ts recent` (manuel, hors rôle, validation SELinux) | journal d'audit | recherche de refus AVC | Oui — `journalctl -k -g 'avc:'` (non privilégié) essayé en parallèle, même résultat (aucun refus), les deux méthodes rapportées |

Toutes les autres actions de ce livrable (script d'intégrité,
minuterie CDI, service de notification, redémarrage du service
utilisateur, corruption/restauration délibérée d'un blob) s'exécutent
dans le domaine utilisateur, sans `become`, sans `sudo`.

**Validation Ansible**, `roles/completion/` et `roles/local_ai/` :
```
$ ansible-playbook --syntax-check roles/local_ai/local_ai.yml      # succès
$ ansible-playbook --check roles/local_ai/local_ai.yml             # succès, changed=0
$ ansible-playbook roles/local_ai/local_ai.yml                     # succès, changed=0 (état déjà nominal)
$ ansible-playbook roles/local_ai/local_ai.yml                     # succès, changed=0 (deuxième exécution)
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/local_ai/
Passed: 0 failure(s), 0 warning(s) — profil production
$ ~/.venvs/ansible-lint/bin/ansible-lint --profile production roles/completion/
Passed: 0 failure(s), 0 warning(s) — profil production
```
**Bug trouvé et corrigé en écrivant ce livrable** (§ 10.3) : la garde
VRAM héritée d'IA-1, cassée par le premier usage réel du service — les
deux démonstrations rejouées avec la logique corrigée, `changed=0`
dans les deux cas, état réel restauré après la démonstration d'échec
forcé.

**SELinux** :
```
$ getenforce   # Enforcing, avant et après ce livrable
$ sudo -n ausearch -m avc -ts recent   # aucun refus (précédent : docs/gpu-containers.md § 8.6/8.7)
$ journalctl -k -g 'avc:'              # aucun refus (méthode non privilégiée, même résultat)
```

**Confirmations finales** : aucun modèle supplémentaire téléchargé
(seuls les deux modèles déjà présents depuis IA-3 ont servi aux essais
de complétion et à la démonstration de bascule, `docs/completion.md`
§ 8) ; `/usr/lib/modprobe.d/nvidia-power-management.conf` intact
(`rpm -Vf` identique avant/après, comme à chaque livrable de cette
série) ; aucun nouveau dépôt ; `getenforce` inchangé ; aucun
redémarrage déclenché par cette session (le paramètre pilote D16 est
passé à `1` entre-temps — constaté par lecture, pas déclenché ici).

**Décompte du jeton de vérification, `CLAUDE.md` exclu**
(`grep -c '@VERIF'`) : `docs/completion.md` = 7 occurrences (2
pré-existantes CMP-0/CMP-1, 5 nouvelles ou reformulées par ce
livrable — dont deux paires « annoncé puis marqueur », même patron que
le reste de ce dépôt) ; `docs/local-ai.md` = 4 occurrences, toutes
pré-existantes, inchangées par ce livrable ; `docs/machine-facts.md` =
7 occurrences (6 pré-existantes + 1 nouvelle, D22 : le facteur de
ralentissement du premier chargement à cache disque froid) ;
`docs/gpu-containers.md` = 0, fichier non touché par ce livrable.

## 11. BSH-2 — D15 réalignée sur l'état réel (2026-08-12)

### 11.1 — Écart découvert, mesuré avant d'écrire

Constat de départ (BSH-1, terminé le 2026-08-12, `docs/completion.md`
§ 10) : `roles/local_ai/defaults/main.yml` déployait toujours
`local_ai_ollama_keep_alive: "-1"` (résidence permanente, `OLLAMA_KEEP_ALIVE`
confirmé `-1` côté Quadlet, `ollama ps` affichant `UNTIL: Forever`)
alors que `docs/machine-facts.md` documentait D15 comme requalifiée
« à la demande » depuis le 2026-08-07 (D20). Le code n'avait jamais
suivi la requalification — la valeur effective divergeait de la
décision documentée, pas seulement un commentaire.

**Le coût annoncé de la résidence a été mesuré avant de trancher**,
avec la méthode d'isolement de `docs/dgpu-power.md` § Méthode
d'isolement (relevés `sysfs` purs, espacés, sans invoquer `nvidia-smi`
entre les deux, effet de sonde écarté). État observé au moment de la
mesure : conteneur `ollama` démarré à `2026-08-12T02:38:34+02:00`,
modèle de complétion chargé à `02:50:17+02:00` (`loaded runners
count=1`, journal du service) — soit 703 s après le démarrage du
conteneur, service actif mais sans modèle pendant cet intervalle.

**Relevé A, pendant l'intervalle sans modèle** (aucune sonde entre les
deux, méthode identique à § 7.6) — `runtime_suspended_time` a continué
de croître normalement durant cette phase, cohérent avec § 7.6 (aucun
contexte CUDA sans modèle chargé). **Relevé B, après chargement,
fenêtre isolée de 300 s** :
```
T0 2026-08-12T03:47:56+02:00  runtime_status=active
                               runtime_suspended_time=788040   (figé depuis le chargement)
                               runtime_active_time=3555284
[300 s, aucune sonde nvidia, modèle résident]
T1 2026-08-12T03:52:56+02:00  runtime_status=active
                               runtime_suspended_time=788040   (Δ=0 ms)
                               runtime_active_time=3855288      (Δ=+300004 ms ≈ Δt réel)
```
Rejoué une seconde fois (fenêtre indépendante, 300 s, 03:40:02 →
03:45:47), même résultat : `runtime_suspended_time` inchangé,
`runtime_active_time` progresse au rythme exact du temps écoulé.
**Conclusion établie, pas supposée** : une fois le modèle réellement
résident, RTD3 est bloqué à 100 % pour la fenêtre observée — le
comportement documenté par D15/D16 (`docs/local-ai.md` § 9.3, cité
plus haut) est confirmé, pas contredit.

**Écart apparent expliqué, pas un second coût découvert** : le chiffre
initial de 25 % (`runtime_suspended_time = 788 040 ms` sur 53 min,
soit ≈ 788 040 / (53×60×1000) ≈ 24,8 %) rapportait le compteur cumulé
**depuis le démarrage du conteneur**, une fenêtre qui mélange les 703 s
de service sans modèle (RTD3 encore libre, § 7.6) et le reste en
résidence effective (RTD3 bloqué). `788 040` ms de veille se sont
intégralement accumulés **avant** `02:50:17` — aucune veille
supplémentaire n'a été observée aux deux relevés isolés effectués
**après** ce chargement. **Correction apportée à l'affirmation**, pas
une reconduite : la contrepartie de D15 n'est ni « ~8 W en continu dès
le démarrage du service » (§ 7.6 la réfutait déjà) ni « ~75 % du temps
réel », mais **100 % de blocage RTD3 pendant toute la durée où un
modèle reste effectivement chargé** — le seul chiffre à retenir tant
qu'aucune mesure contraire n'est produite. `docs/status.md` et
`docs/machine-facts.md` § D15/D20 sont corrigés dans ce sens
(ci-dessous, § 11.4).

### 11.2 — Motif du réalignement, pas un retour en arrière

La requalification du 7 août (D20) était juste à sa date : la
complétion n'était pas encore établie en usage réel, et le coût d'un
rechargement à froid n'était pas mesuré — engager la contrepartie RTD3
avant d'avoir ces deux faits aurait été payer un coût certain pour un
bénéfice supposé (motif exact de D20, `docs/machine-facts.md`).

Deux faits nouveaux, établis depuis, justifient le réalignement :
- la complétion fonctionne réellement, dans **deux éditeurs sur trois
  langages** (Kate et Helix, yaml/python/bash — `docs/completion.md`
  § 9/10) ;
- une complétion à froid coûte **22,6 s mesurées** (Helix, BSH-1,
  `docs/completion.md` § 10.3) — réveil RTD3 **plus** chargement complet
  du modèle en VRAM, pas les **~3,7 s** d'une simple bascule entre deux
  modèles déjà chauds (IA-4, `docs/local-ai.md` § 9.4, chargement
  séquentiel). Cette dernière valeur ne couvrait jamais le cas
  résident→arrêté→résident : elle mesurait un échange de modèle **déjà
  en cache disque OS et sans réveil RTD3 à payer** (le service tournait
  déjà, un modèle était déjà chargé) — un cas structurellement plus
  favorable que celui d'un modèle à la demande après une pause de
  frappe, où RTD3 s'est effectivement engagé entre-temps (§ 7.6).

**La mesure de 22,6 s corrige l'estimation antérieure de 3,7 s**, plutôt
que de s'y ajouter : ce ne sont pas deux coûts différents à additionner,
mais deux régimes distincts confondus jusqu'ici sous un seul chiffre — à
la demande, chaque reprise après une pause de frappe paierait le premier
(réveil + chargement), pas le second (bascule à chaud), ce qui rend la
complétion au fil de la frappe inutilisable dans ce régime.

### 11.3 — Décision de l'opérateur

L'opérateur a tranché : le modèle de complétion reste résident. D15 est
réalignée sur cette décision et sur l'état réel du dépôt (jamais changé
entre D20 et ce livrable — seule la décision documentée divergeait,
`roles/local_ai/defaults/main.yml`, aucune modification de code
nécessaire au-delà des commentaires).

### 11.4 — Recherche systématique des références périmées

Recherche `grep -rn "D15"` sur tout le dépôt (`roles/`, `docs/*.md`,
`CLAUDE.md`), **pas limitée au périmètre de ce livrable** (CLAUDE.md,
règle renforcée ci-dessous) :

| Fichier | Mention | Classe | Traitement |
|---|---|---|---|
| `roles/local_ai/defaults/main.yml:7` (commentaire de décision) | D15 = résidence | Commentaire de décision | Mis à jour : `[RÉALIGNÉE le 2026-08-12, BSH-2]` |
| `roles/local_ai/defaults/main.yml:50` (juste au-dessus de la valeur) | D15 = keep_alive infini | Commentaire attaché à la **valeur effective** | Valeur elle-même inchangée (déjà correcte) ; commentaire mis à jour pour ne plus référencer une décision périmée sans le signaler |
| `roles/local_ai/defaults/main.yml:229` (déclencheurs CDI, IA-4 § 3) | « actif indéfiniment, D15/D20 » | Commentaire | **Déjà exact** pour l'état réaligné (décrit un service conçu pour rester actif indéfiniment) — aucune correction nécessaire |
| `roles/local_ai/tasks/main.yml:156` | « D20 : chargement à la demande, D15 : maintenu indéfiniment » | Commentaire (garde VRAM, IA-4) | Compatible avec D15 réalignée (le fait décrit — modèle maintenu une fois chargé — reste vrai indépendamment de la politique de chargement) ; hors périmètre strict de ce livrable, non modifié, signalé |
| `roles/local_ai/tasks/main.yml:311` | même patron | Commentaire | Idem, signalé sans modification |
| `roles/local_ai/templates/local-ai-nvidia-power-management.conf.j2:15` | « neutralisée par le modèle résident (D15) » | Commentaire | Déjà exact pour l'état réaligné, aucune correction nécessaire |
| `roles/local_ai/templates/ollama.container.j2:46` | « D15 — modèle de complétion résident en permanence » | Commentaire attaché à `Environment=OLLAMA_KEEP_ALIVE=...` | **Déjà exact** — décrit la valeur réellement déployée, jamais périmé |
| `roles/local_ai/templates/local-ai-cdi-verify.service.j2:12` | « D15 : keep_alive infini » | Commentaire | Déjà exact |
| `roles/local_ai/README.md:32` | « keep_alive infini (D15) » | Commentaire | Déjà exact |
| `roles/completion/defaults/main.yml:8`, `README.md`, `meta/main.yml`, `templates/*.j2` | « D20 (requalifie D15) » | Commentaire, **hors périmètre de ce livrable** | Fait décrit par ce rôle (`roles/completion/` ne charge ni ne choisit aucun modèle) reste vrai quel que soit l'état de D15 — **signalé, pas corrigé** : ces commentaires ne mentionnent pas le réalignement du 2026-08-12, dette documentaire mineure, non bloquante (aucune valeur effective de ce rôle n'en dépend) |
| `docs/status.md:119` | « D15, requalifiée par D20 » (table des options écartées) | Valeur documentaire | Corrigé (ci-dessous, hors `roles/`) |
| `docs/review-2026-08.md:83` | « D15 requalifiée par D20 » | Récit historique daté | **Non modifié** — décrit fidèlement un événement passé (2026-08-07), pas l'état courant ; modifier une revue déjà close reviendrait à réécrire l'histoire, contraire à la règle « marquer, jamais effacer » |

**Seule une divergence de valeur effective existait** : la variable
`local_ai_ollama_keep_alive` du rôle a toujours porté `-1` — jamais
changée par la requalification du 7 août, qui n'a vécu que dans la
documentation. Aucun gabarit, aucune valeur par défaut de
`roles/local_ai/` ne portait la valeur « à la demande » ; le
réalignement de ce livrable est donc **documentaire pour le code**
(les commentaires rattrapent une valeur qui n'avait jamais bougé) et
**correctif pour `docs/machine-facts.md`/`docs/status.md`** (qui
avaient, eux, suivi la requalification).

**Recherche étendue à d'autres décisions révisées** (§ 3 de la
demande) : seule D3 (chaîne Ansible) a fait l'objet d'une révision
comparable dans ce dépôt (scission D3a/D3b, 2026-08-06) — déjà traitée
par la revue globale du 2026-08-08 (`docs/review-2026-08.md` § 2.1,
`CLAUDE.md` § Chaîne Ansible, corrigé alors). Aucune autre décision
requalifiée trouvée avec une référence de code non alignée.

### 11.5 — Branches exercées et non exercées

| Conditionnel | Branche exercée | Branche non exercée |
|---|---|---|
| Garde VRAM avant/après (`local_ai_ps_before`/`_after`, IA-4) | Modèle déjà résident avant ce rôle, présent après, égalité confirmée — exercée à chaque exécution de ce livrable | Aucun modèle résident avant, chargement par un tiers pendant l'exécution du rôle (fenêtre de course) — non exercée, déjà signalée comme non exercée en IA-4 |
| `rpm -Vf` fichier fournisseur modprobe.d (D16) | Fichier intact avant/après — exercée | Divergence détectée (`fail_msg`) — jamais provoquée, aucune modification volontaire du fichier fournisseur dans ce livrable ni les précédents |
| Redéploiement Quadlet si le contenu change | **Non exercée dans ce livrable** — le contenu du gabarit `ollama.container.j2` n'a pas changé (seul le commentaire l'entourant a été édité, jamais le corps `Environment=OLLAMA_KEEP_ALIVE={{ ... }}` ni la valeur de la variable) ; confirmé par `changed=0` aux deux exécutions (§ 11.6) | Redéploiement effectif avec redémarrage du service — dernière fois exercée en IA-1 (premier déploiement) ; ce livrable ne le réexerce pas, par construction (aucune valeur modifiée) |

### 11.6 — Idempotence, service non redémarré

```
$ ansible-playbook --syntax-check roles/local_ai/local_ai.yml   # succès
$ ansible-playbook --check roles/local_ai/local_ai.yml           # succès, changed=0
$ ansible-playbook roles/local_ai/local_ai.yml                   # succès, changed=0 (exécution réelle)
$ ansible-playbook roles/local_ai/local_ai.yml                   # succès, changed=0 (deuxième exécution)
```
`changed=0` dès la **première** exécution réelle de ce livrable —
preuve directe que la valeur déployée était déjà celle attendue
(`local_ai_ollama_keep_alive: "-1"`, jamais modifiée par le code). Le
service `ollama.service` n'a pas été redémarré par ce rôle : la date
de démarrage du conteneur (`podman inspect ollama`, champ
`State.StartedAt`) est restée inchangée avant/après
(`2026-08-12 02:38:34.726481525 +0200 CEST`), et le modèle
est resté résident sans interruption (`ollama ps`/`GET /api/ps`
confirme `expires_at` inchangé, une date arbitrairement lointaine
propre au keep_alive infini). Rapport final du rôle, ligne pertinente :
« Aucun modèle nouvellement chargé en VRAM par ce rôle (avant :
['qwen2.5-coder:7b-instruct-q4_K_M'], après : [même liste]) ».

### 11.7 — Actions privilégiées

| # | Commande | Cible | Motif | Tentative sans privilège : résultat |
|---|---|---|---|---|
| — | Aucune | — | Ce livrable ne modifie que des commentaires et de la documentation ; la seule variable dont la valeur effective était en jeu (`local_ai_ollama_keep_alive`) n'a pas changé, aucune tâche `become` supplémentaire déclenchée | Non applicable — aucune action privilégiée exécutée par ce livrable |

### 11.8 — Confirmations finales

Aucun paquet installé, aucun modèle téléchargé (les deux modèles déjà
présents depuis IA-3 ont servi aux mesures) ; `sudoers`, `terra.repo`,
`/etc/cdi/`, `gpu_mux_mode`, `kwinrulesrc`, `site.yml` intacts (aucun
touché par ce livrable — hors périmètre) ; aucun redémarrage système ;
service `ollama.service` non redémarré (§ 11.6). `ansible-lint
--profile production roles/local_ai/` : 0 défaut.

## Voir aussi

- [`docs/dgpu-power.md`](dgpu-power.md) — mécanismes RTD3, méthode
  d'isolement de mesure réutilisée ici.
- [`docs/gpu-containers.md`](gpu-containers.md) — CDI rootless, preuve
  de non-réveil au lancement de conteneur, image de test déjà en cache.
- [`roles/gpu_cdi/`](../roles/gpu_cdi/) — mécanisme de détection de
  péremption (`verify-cdi-spec`), non déclenché automatiquement (§ 2.4).
- [`docs/editor.md`](editor.md) — D11(D12) npm fermé, capacité native
  `:pipe`/`externaltoolsplugin` réutilisée § 8.6 ; même discipline de
  nomination des surfaces d'approvisionnement appliquée aux registres
  de conteneurs.
- [`docs/repositories.md`](repositories.md) — ancrages de confiance
  D7/D10, registre de conteneurs Ollama nommé § 6 (IA-2), registre de
  modèles Ollama et écart d'intégrité découvert § 8 (IA-3).
- [`docs/completion.md`](completion.md) — `lsp-ai`/Helix (CMP-1),
  cause de l'échec de complétion distinguée § 9.5 (IA-3).
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici.
