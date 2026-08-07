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
série. `@VERIF : chargement simultané réel de deux modèles (un petit
tenu en permanence, un plus grand à la demande) — non mesuré, nécessite
de charger des modèles réels, hors périmètre lecture seule de ce
livrable.`

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

## 3. Ce qui existe dans les onze dépôts activés

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
`python3-torch` depuis les onze dépôts activés ne donnerait accès à la
RTX 4090 sous aucune forme** — au mieux un repli CPU (non vérifié ici,
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
  exécuté). Une reprise après coupure ou un fichier corrompu serait, en
  principe, détecté par la vérification d'empreinte du protocole —
  `@VERIF : comportement exact d'ollama pull en cas de blob corrompu ou
  coupure réseau — non testé, aucun modèle téléchargé dans ce
  livrable.`
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

## Voir aussi

- [`docs/dgpu-power.md`](dgpu-power.md) — mécanismes RTD3, méthode
  d'isolement de mesure réutilisée ici.
- [`docs/gpu-containers.md`](gpu-containers.md) — CDI rootless, preuve
  de non-réveil au lancement de conteneur, image de test déjà en cache.
- [`roles/gpu_cdi/`](../roles/gpu_cdi/) — mécanisme de détection de
  péremption (`verify-cdi-spec`), non déclenché automatiquement (§ 2.4).
- [`docs/editor.md`](editor.md) — D11(D12) npm fermé, même discipline
  de nomination des surfaces d'approvisionnement appliquée ici aux
  registres de conteneurs.
- [`docs/repositories.md`](repositories.md) — ancrages de confiance
  D7/D10, pertinents pour tout registre de conteneurs retenu.
- [`CLAUDE.md`](../CLAUDE.md) — règles de sourcing appliquées ici.
