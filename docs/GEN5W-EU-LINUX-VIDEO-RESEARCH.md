# Gen5W Linux européen : enquête sur le chemin vidéo SVM (recherche web)

**Date :** 30 octobre 2026. **Cible :** IONIQ 5 européenne première génération, HDA2, 360° SVM, version vérifiée **`NE_E.EUR.S5W_L.001.001.260128`**, modèle HU `NEAE.S5BLC.AEU`.

**Synthèse :** la communauté a rétro-ingéniéré le système **Gen5W_L Linux**, mais aucune capture continue des quatre flux SVM natifs sur ce firmware n'a été trouvée dans les sources consultées. Ne pas confondre accès au head unit, capture de sa sortie vidéo, et capture de quatre caméras en amont du SVM.

## Sources et degré de pertinence

| Source | Ce qu'elle apporte concrètement | Limite |
|---|---|---|
| [Hyundai Europe — versions logicielles](https://update.hyundai.com/EU/TR/updateNoticeView/software-version) | Confirme textuellement `NE_E.EUR.S5W_L.001.001.260128` et `NEAE.S5BLC.AEU` (famille IONIQ 5 originale), distincte de ccNC | N'explique pas le chemin vidéo |
| [laralbir — Hyundai/Kia Gen5W EU firmware](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware) | Audit de `S5W_L` européen, structure `mango-rootfs.tar.gz`, outils d'inspection, Docker/QEMU, doc d'exploration | Véhicule différent (Kia Rio); rootfs OTA chiffré sans éléments récupérés sur HU |
| [laralbir — écosystème Gen5W](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware/blob/main/docs/gen5w_exploit_ecosystem.md) | Nomme les outils communautaires `navi_extended`, `update_decryptor`, `gen5w-docker` et leurs prérequis | L'entrée exploit initiale est signalée comme bloquée pour certains firmwares récents; n'est pas un accès vidéo |
| [valdivia287-ui — Hyundai i20 N infotainment lab](https://github.com/valdivia287-ui/hyundai-i20n-infotainment-lab) | Tests rapportés d'une application OEM sous QEMU x86-64 + Weston/Wayland IVI + Qt/QML 5.4.1 ; observation des événements d'entrée, D-Bus | Le lab n'est pas un émulateur complet ni une capture de flux caméra ; essais privés non indépendamment reproduits |
| [Chimaera — publication scientifique (2024)](https://link.springer.com/article/10.1007/s11416-024-00522-4) | Démonstration sur une unité Gen5W_L LG; applications qui communiquent par D-Bus; accès au M-Bus, APIs `com.lge.car.micom` | Recherche sur une variante plus ancienne `AE_E_PE_EUR.S5W_L001.001.211214`; pas un pipeline SVM documenté |
| [Gen5W Android reverse engineering](https://github.com/cantcs/HKG_Gen5W_ReverseEngineer) | Pointe vers la communauté « Ioniq Dev » historiquement active sur les systèmes Linux Hyundai | Le projet lui-même cible principalement Android `S5W_M`, pas le nôtre |
| [RearDashcam upstream](https://github.com/KramerTech/gen5-RearDashcam/blob/main/TECHNICAL.md) | Explique exactement une capture depuis la HAL Android et le tampon d'affichage ; sur des SVM Android de l'auteur, les images valides ne sont disponibles au HU que pendant la vue Hyundai | Pas de preuve que le SVM Linux EU a le même pilote, la même RAM ou le même chemin de commandes |
| [Discussion Reddit RearDashcam (octobre 2026)](https://www.reddit.com/r/Ioniq5/comments/1xgykqj/free_rear_dashcam_by_recording_backup_camera_feed/) | Confirme une demande explicite de port européen, mais pas de port public établi; l'auteur précise que sa solution Android ne fonctionne pas sur Linux EU | Discussions communautaires, pas preuve d'impossibilité technique |
| [nudl — outil de téléchargement firmware Hyundai/Kia](https://github.com/chenxiaolong/nudl) | Permet de lister/télécharger les OTA selon les modèles/régions, pour étude locale | Le téléchargement n'accorde pas l'accès aux binaires déchiffrés |

Attention à l'homonymie : les occurrences « camera_editor » du repo `laralbir` concernent les **radars routiers de navigation**, **pas** les caméras vidéo de la voiture.

## Ce qu'on peut raisonnablement inférer

La **pile Linux** d'autres Gen5W européens fait intervenir des applications **Qt/QML**, un **compositeur Wayland/Weston IVI** et des services **D-Bus**. Ces éléments fournissent des pistes précises de rétro-ingénierie mais ne déterminent **pas** comment le flux vidéo du SVM est acheminé sur notre HU.

Trois couches distinctes :
1. **Avant l'ECU SVM** : 4 caméras et 4 liaisons coaxiales ; protocole sérialiseur/désérialiseur **inconnu**.
2. **Après l'ECU SVM** : une ou plusieurs liaisons vidéo vers l'unité multimédia, possiblement avec sélection/composition/correction géométrique. L'hypothèse que la HU reçoive seulement une vue sélectionnée est **plausible mais non prouvée** pour notre firmware.
3. **Dans l'unité multimédia Gen5W_L** : récepteur/pilote d'entrée vidéo → buffers/traitement → composition en calque d'affichage → écran. Technologie exacte **inconnue** (V4L2, device privé, DRM/KMS, DMA-BUF, autre). Ne **pas** affirmer qu'il existe `/dev/video0`, que l'interface D-Bus offre des images, ni que le compositor peut dupliquer quatre flux.

Les deux écrans (central SVM, combiné BVM) ne suffisent pas à déduire la topologie réelle.

## Plan de recherche **hors véhicule** prioritaire

1. **Télécharger officiellement** (aucun flash) l'OTA correspondant à la HU `NEAE.S5BLC.AEU` / version `NE_E.EUR.S5W_L.001.001.260128` ; vérifier la correspondance des métadonnées. Les fichiers du rootfs peuvent être chiffrés : la possession du package ne signifie pas qu'il sera possible d'inspecter les binaires.
2. Si un **rootfs déchiffré licitement obtenu** est disponible en local, inventorier noms de processus, executables et bibliothèques contenant `avm`, `svm`, `rvc`, `rear`, `camera`, `parking`, `surround`, `v4l2`, `drm`, `weston`, `vioc`, `lvds`, `gmsl`. Utiliser `file`, `strings`, `readelf`, `objdump`, `grep` et Ghidra sur **copies locales**. Ne pas committer de binaires/clés OEM.
3. Remonter les dépendances du processus affichant l'image caméra : bibliothèques partagées, modules kernel, configs Wayland/IVI, appels D-Bus (événements d'activation), interfaces de capture du noyau si elles existent.
4. Déterminer **si la vidéo arrive au framebuffer/HU pendant les vues Hyundai seulement** ou si un buffer amont indépendant est accessible de manière passive. Distinguer **une vue composée/sélectionnée** de **quatre vues brutes indépendantes**.
5. Préparer sur PC des tests reproductibles à partir de **buffers vidéo synthétiques**, sans commandes au véhicule. En cas de besoin ultérieur d'une session en voiture, limiter initialement les commandes aux **lectures/diagnostics** après procédure de sécurité et validation du firmware.
6. **Décision d'architecture :** si la HU ne voit qu'une vue sélectionnée, ou que ses buffers sont indisponibles à la demande, un **interposeur TAP coaxial actif à quatre canaux** en amont du SVM sera probablement indispensable pour Ioniq Control. Ne pas commander de carte GMSL/FPD-Link avant d'avoir identifié le SerDes des caméras.

## Ce qu'il ne faut pas faire

- Ne **pas** installer NEMESIS sur `S5W_L` : il cible la version Android `S5W_M`.
- Ne pas flasher un firmware modifié, forcer un mode d'ingénierie ou utiliser un exploit de HU dans le véhicule de production pour commencer.
- Ne pas introduire d'émission CAN ni ouvrir une entrée de capture exclusive susceptible de désactiver le SVM/BVM.
- Ne pas considérer un screenshot du compositeur comme preuve d'accès aux quatre caméras.
- Ne pas rendre publics des firmwares OEM déchiffrés, clés, données personnelles ou logs non nettoyés.

## Question technique précise pour la communauté

> Sur le Gen5W Linux européen `S5W_L` de l'IONIQ 5 avec SVM, quel démon/pilote reçoit la **sortie vidéo** de l'ECU SVM et vers quel buffer/calque d'affichage l'envoie-t-il ? Le processus reçoit-il une seule vue sélectionnée, ou plusieurs buffers par caméra ? Existe-t-il un mode de capture **en lecture seule** sans démarrer de session exclusive ni modifier les commandes du parking ECU ?

**Bilan au 30/10/2026 :** pas de solution prête à installer ; approche la plus prudente = analyse offline des firmwares / interaction avec développeurs Linux Gen5W, et caractérisation électronique du signal des caméras OEM en parallèle.
