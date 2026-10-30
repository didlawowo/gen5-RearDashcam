# Pré-étude du code `gen5-RearDashcam` et de NEMESIS — IONIQ 5 EU 2024

**Date :** 30 octobre 2026. **Statut :** revue statique, aucune exécution ou installation dans le véhicule.

## Compatibilité du véhicule : confirmée par photo du système (30/10/2026)

**Informations relevées directement sur l'unité multimédia de l'IONIQ 5 EU 2024 :**

- Plateforme : `Standard-class Gen5W Navigation`
- Modèle : `NEAE.S5BLC.AEU`
- Version du logiciel : **`NE_E.EUR.S5W_L.001.001.260128`**
- Date de mise à jour affichée : `06/05/2026`

**Conclusion définitive pour cet exemplaire : NO-GO NEMESIS / gen5-RearDashcam.** Le suffixe `S5W_L` indique le Gen5W Linux ; NEMESIS exige la famille `S5W_M` Android. La preuve vient de l'écran de la voiture (photo transmise par son propriétaire). Ne pas tenter l'installation NEMESIS, le paquet RearDashcam ni de firmware US ; une simple recompilation de l'application ne rend pas compatibles les services, la HAL et les pilotes.

**Direction d'étude :** documenter la pile vidéo du Gen5W Linux européen et les possibilités de capture non intrusive en sortie SVM ; en parallèle, continuer l'étude d'un TAP actif sur les 4 coaxiaux en amont du SVM si 4 vues brutes simultanées sont indispensables. Pas de commande CAN ou changement de firmware en voiture.

## Synthèse / go-no-go

1. L'upstream est **actif et très récent** (commit du 29/10/2026, [a17861f](https://github.com/KramerTech/gen5-RearDashcam/commit/a17861f37194f22f3ceb97b7859dd9cb77f3d3f4)).
2. `gen5-RearDashcam` exploite l'API privée **`android.hardware.RearCamera` sur un head unit Android Gen5W** et lit une zone mémoire vidéo réservée. Il **ne lit pas** le coaxial entre les quatre caméras et le calculateur SVM.
3. Le logiciel actuel fonctionne avec une **caméra arrière analogique standard, HAL type 0**, et **refuse explicitement le type 6 SVM** ; la fonction de diagnostic n'est pas une compatibilité de capture permanente.
4. **Bloquant maintenant **confirmé sur le véhicule cible** :** le projet NEMESIS ne cible que les firmwares `S5W_M` (Gen5W Android), **pas** `S5W_L` (Gen5W Linux), ni ccNC. Hyundai Europe référence le premier IONIQ 5 en `NE_E.EUR.S5W_L`. La photo des paramètres établit `NE_E.EUR.S5W_L.001.001.260128`.
5. **Décision : aucun essai d'installation** de NEMESIS, du package, ni de désactivation du `preinstall` avant vérification explicite de la version et conception d'un test non destructif.

Sources : [fork utilisateur](https://github.com/didlawowo/gen5-RearDashcam), [README upstream](https://github.com/KramerTech/gen5-RearDashcam/blob/main/README.md), [TECHNICAL upstream](https://github.com/KramerTech/gen5-RearDashcam/blob/main/TECHNICAL.md), [README NEMESIS](https://github.com/xwtk/hmg_nemesis/blob/main/README.md), [tableau Hyundai Europe](https://update.hyundai.com/EU/FR/updateNoticeView/software-version).

## NEMESIS, précisément

- **Pas de matériel à brancher au coaxial.** C'est une interface de développement / débogage en logiciel, installée dans le **système multimédia** via la **prise USB de données** et un mécanisme de mise à jour DSP/OTA sur un matériel compatible.
- Cible du projet : Gen5W sous **Android 4.4.2**, **ARM 32 bits, Telechips TCC803x**, présence de support OTA et **identifiant `S5W_M`**.
- Installation de NEMESIS : modifie le `/system` de la voiture, ajoute une CLI `/system/xbin/nemesis`, une APK Manager, un watchdog, un manager de services, un handler USB. Fournit des commandes avec privilèges `root`, des hooks et l'installation de paquets `NEMESIS.data`/zip.
- NEMESIS se connecte **logiquement** aux services Android de la HU; le transfert des paquets se fait via USB, ou ADB une fois un accès ADB disponible. Ce n'est **ni un adaptateur caméra**, **ni un décodeur GMSL/FPD-Link**, **ni un connecteur CAN**.
- Il exploite une faille de la logique OTA; installation non triviale, **non officielle** et potentiellement susceptible de rendre le système multimédia inutilisable.
- NEMESIS (AGPLv3) et RearDashcam (MIT) ont des licences **différentes**. Les fichiers de RearDashcam modifiés doivent conserver la notice MIT.

Sources : [fonctionnement NEMESIS](https://github.com/xwtk/hmg_nemesis/blob/main/02-Debugging-Execution-Interface.md), [installation](https://github.com/xwtk/hmg_nemesis/blob/main/03-Install-and-Uninstall.md), [gestion des paquets](https://github.com/xwtk/hmg_nemesis/blob/main/04-Packages.md).

## Revue par fichiers du fork

| Fichier | Comportement réellement observé | Réutilisation possible |
|---|---|---|
| `package/NEMESIS.data` | Déclare APK, service enregistreur, watchdog et helper overlay | Emballage seulement sur Gen5W Android compatible |
| `package/scripts/preinstall.sh` | Lance `CameraCheck`; **refuse type 6**. S'il ne peut pas identifier le type, le script **accepte quand même** l'installation, d'où nécessité de bloquer notre propre déploiement. | À préserver ; modifier seulement après validation sur banc |
| `src/dev/skramer/dashcam/CameraCheck.java` | Imprime `RearCam.type()`, utilisé par le préinstallateur | Bonne entrée de diagnostic conceptuelle |
| `src/dev/skramer/dashcam/RearCam.java` | Appels par réflexion à `android.hardware.RearCamera`, `native_setup`, `startPreviewDD`; accès exclusif à l'entrée caméra et parcours matériel direct-display | Très spécifique au firmware US Android |
| `src/dev/skramer/dashcam/CameraRam.java` | Lit `/proc/pmap`, trouve la région `rearcamera`, mappe **en lecture seule** via `/dev/admem`, quatre buffers BGRA | Technique liée à ce noyau/driver, non portable telle quelle en EU Linux |
| `src/dev/skramer/dashcam/Recorder.java` | Sonde changement des buffers, copie frames, convertit BGRA→I420, encode en H.264 avec `OMX.TCC.ENC.avc` et transmet à `SegmentWriter` | Algorithmes, séparation des responsabilités, mais HAL et encodeur à remplacer sur Linux |
| `src/dev/skramer/dashcam/SegmentWriter.java` | File d'écriture asynchrone, segments MPEG-TS, coupe aux keyframes, sync USB, sauvegarde événements | Réutilisable conceptuellement, plus facile à porter |
| `src/dev/skramer/dashcam/Guard.java` | Poll position boîte (50 ms en capture), écoute événement stock et gère libération caméra, statut contact, heartbeat ; refuse type 6 au runtime | Logique de sécurité importante ; couplée aux services propriétaires US |
| `src/dev/skramer/dashcam/CameraDebug.java` | Rapports HAL, caméra, `/proc/pmap`, device tree, logs et `getprop` ; le debug peut aussi **ouvrir** une session caméra — ne pas le qualifier intégralement de lecture seule | Bonne liste d'observables à adapter |
| `native/dashcam.c` | NEON ARMv7 : conversion de pixels, copie et réduction d'images | Algorithme utile mais dépend de l'architecture |
| `native/overlay.c` | Mappe des **registres VIOC du TCC803x** via `/dev/mem`, rend transparent le calque caméra, restaure à la fermeture | Très spécifique au matériel ; ne pas exécuter sur autre plateforme |
| `package/svc/watchdog.sh` | Tue l'enregistreur s'il ne produit plus de heartbeat, afin de libérer la caméra stock | Très bonne exigence fail-safe |
| `package/scripts/common.sh`, `preun.sh` | Nettoyage/restauration des lignes de guidage Hyundai via services `com.hkmc.micom.pgs` | Couplage constructeur, à documenter et tester |
| `build.sh` | Compile Java/Dalvik API 19 + NEON ARMv7 + APK, signe et prépare les ZIP | Non directement portable sur Mac M4 Pro : chemin NDK codé en dur `prebuilt/linux-x86_64`, clé privée de l'auteur absente |

Sources : fichiers identiques disponibles dans le [fork sur `main`](https://github.com/didlawowo/gen5-RearDashcam).

## Chemin exact des frames dans le logiciel US

```text
Caméra arrière analogique OEM
    -> électronique originale / driver vidéo du head unit Gen5W Android
    -> HAL propriétaire android.hardware.RearCamera
    -> buffer rearcamera (BGRA, ring 4 slots, /proc/pmap)
    -> /dev/admem mmap PROT_READ
    -> détection d'une frame complète + copie
    -> BGRA -> I420 (NEON)
    -> encodeur OMX.TCC.ENC.avc (H.264)
    -> SegmentWriter -> MPEG-TS sur USB
    -> APK DashcamPlayer sur écran de l'autoradio
```

Mode parking : l'enregistreur lâche l'entrée vidéo pour l'application Hyundai, et surveille temporairement les buffers passifs pendant l'utilisation de la vue arrière. Attention : ceci **partage une ressource caméra exclusive** et ne garantit pas le fonctionnement des fonctions OEM en cas de bug.

### Pourquoi l'IONIQ 5 360° est bloquée

Le document technique de l'auteur indique :
- `getCameraType()==6` sur ses configurations SVM.
- Un nœud `tcc_camera/max96706_lvds` existe **côté autoradio**. Ce n'est **pas** l'identification des SerDes individuels des quatre caméras vers le SVM.
- Démarrer directement une session vidéo ne produit pas un flux actif continu : quelques images gris foncé, puis arrêt.
- Des images apparaissent dans la mémoire quand l'ECU de parking ordonne au véhicule d'afficher une vue caméra; l'affichage est signalé au head unit par CAN.
- Le code n'a donc accès, à ce stade, qu'à la **vidéo affichée / commutée en sortie du système SVM**, pas aux quatre flux bruts individuels.

## Comment vérifier la compatibilité du véhicule (sans installation)

1. Véhicule immobilisé, aller dans **Paramètres → Général → Informations système / Informations logiciel**.
2. Photographier ou recopier la version (ex. `NE_E.EUR.S5W_L...` ou `NE1.EUR.ccNC...`).
3. Si `S5W_L` : NEMESIS **non supporté** (architecture Linux différente). Si `ccNC` : également non supporté. Si exceptionnellement `S5W_M` : vérifier support OTA et firmware exact **sans lancer de mise à jour expérimentale**.
4. Même avec `S5W_M`, le **SVM type 6** reste un second blocage : aucune installation du package RearDashcam tel quel.

**Attention millésime :** Hyundai Europe classe les premières IONIQ 5 dans la famille `NE_E.EUR.S5W_L` et les IONIQ 5 restylées dans ccNC. Une immatriculation en 2024 ne prouve pas à elle seule laquelle est montée.

## Contraintes de compilation et d'essais

- `build.sh` demande Java 17+, Android SDK/API 19, outils `d8` et `apksigner`, **NDK r25**, ainsi qu'une clé de signature APK.
- Chemin utilisé : `$ANDROID_HOME/ndk/25.*/toolchains/llvm/prebuilt/linux-x86_64/bin/...`; le script original **ne fonctionne donc pas tel quel sur le Mac Apple Silicon**.
- Le script exige la **SHA-256 du certificat de l'auteur** (présente en constante) ; pour un fork, utiliser sa propre clé et mettre à jour l'empreinte attendue, sans exporter de clé secrète.
- Une CI Linux ou VM Linux est adaptée à un **build hors véhicule**, mais un build réussi ne confère aucune compatibilité embarquée.

## Suites recommandées

1. **P0 (FAIT)** : version confirmée `NE_E.EUR.S5W_L.001.001.260128`, NEMESIS incompatible. Ne pas installer.
2. **P1** : ne pas porter le lanceur NEMESIS ni la HAL Android sans justification; étudier les chemins de capture Linux Hyundai à partir d'une documentation vérifiable, sans flash.
3. **P1** : identifier le sérialiseur d'une **caméra de rechange** avant d'acheter un désérialiseur/TAP, puis valider un canal **sur banc**.
4. **P2** : réutiliser les bonnes idées du fork (MPEG-TS segmenté, détection/horodatage des frames, mécanismes de restauration et watchdog) dans un **nouveau backend** Ioniq Control; ne pas présumer qu'il faut exécuter l'APK en voiture.
5. **P2** : documenter schéma SVM → unité multimédia : la présence d'un MAX96706 côté head unit révèle quelque chose sur une liaison **sortante**, pas sur quatre flux caméra **entrants** au SVM.

**Aucun matériel à commander / aucune installation à faire avant l'étape P0.**
