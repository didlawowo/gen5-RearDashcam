# Portage caméra Gen5W Linux EU — mode opératoire ciblé

**Véhicule :** Hyundai IONIQ 5 2024 EU, HDA2 + SVM 4 caméras, firmware **`NE_E.EUR.S5W_L.001.001.260128`**, modèle **`NEAE.S5BLC.AEU`** (version lue directement sur le véhicule).  
**Date :** 30 octobre 2026. **État :** étude et procédure à valider ; **aucun accès shell, flash ni capture sur ce véhicule effectués**.

## Objectif unique

Reproduire la **capture vidéo** réalisée par [KramerTech/gen5-RearDashcam](https://github.com/KramerTech/gen5-RearDashcam) (version américaine Android) sur notre Gen5W **Linux européen**. Si possible, obtenir des images lorsque la vue Hyundai est affichée, puis vérifier si elles sont disponibles en continu. Récupérer quatre flux individuels reste une **question distincte** : le head unit reçoit peut-être une seule sortie sélectionnée/composée du SVM.

**Hors périmètre :** redesign de l'interface Hyundai, commandes CAN, fonctions ADAS autres que la vision, installation d'une dashcam de production.

## Ce que fait la version américaine (preuve dans le code)

1. NEMESIS lance `svc/recorder.sh` puis `dashcam.jar` sous `app_process`.
2. `RearCam.java` utilise l'API propriétaire Android `android.hardware.RearCamera`.
3. `CameraRam.java` identifie une zone `rearcamera` dans `/proc/pmap`, puis la mappe **en lecture seule** via `/dev/admem`.
4. `Recorder.java` copie les images BGRA d'un anneau de 4 buffers, convertit en I420 (NEON) et encode avec `OMX.TCC.ENC.avc`.
5. `SegmentWriter.java` écrit des segments MPEG-TS vers USB et le player les lit.
6. `Guard.java` rend la main à l'application OEM lors de l'ouverture de la caméra. **Le logiciel refuse actuellement SVM type 6**, dont les buffers ne reçoivent des images valides que pendant l'activation OEM sur les véhicules Android testés par l'auteur.

Lire [README](https://github.com/KramerTech/gen5-RearDashcam/blob/main/README.md) et [TECHNICAL.md](https://github.com/KramerTech/gen5-RearDashcam/blob/main/TECHNICAL.md). Ne pas retirer la garde type 6 du fork en l'état.

## Notre procédure européenne : 5 jalons, avec arrêt conditionnel

| Jalon | Action | Résultat attendu | État |
|---|---|---|---|
| **P0** | Identifier exactement HU et OS via paramètres | `S5W_L`, firmware `260128` | **Fait** |
| **P1** | Sur PC : récupérer une copie **officielle** du firmware pour `NEAE.S5BLC.AEU`, inventorier la structure et rechercher les composants vidéo accessibles sans modifier la voiture | Manifest/version, fichiers disponibles et chiffrement noté | À faire |
| **P2** | Déterminer une **méthode d'accès shell confirmée** pour `NE_E.EUR.S5W_L.001.001.260128` (de préférence sur HU de rechange/banc) | Console shell et mécanisme de récupération documentés | **Bloqué / pas de méthode vérifiée** |
| **P3** | Avec shell déjà disponible : dresser un inventaire **en lecture seule** des processus, pilotes, périphériques d'entrée vidéo et journaux **avant/pendant/après** la vue SVM déclenchée normalement par la voiture à l'arrêt | Nom du processus, du périphérique/buffer, événements d'activation et disponibilité des frames | Dépend de P2 |
| **P4** | Porter uniquement le **backend de capture** vers les interfaces Linux identifiées; test sur banc d'une frame, de clips courts puis de l'enregistrement sans perturbation OEM | Images valides enregistrées ; détermination flux composite vs individuel | Dépend de P3 |

### P1 — Ce qu'on peut faire tout de suite sur ordinateur, sans flasher

- Page Hyundai officielle : <https://update.hyundai.com/EU/FR/updateNoticeView/software-version> — `NEAE.S5BLC.AEU` / `NE_E.EUR.S5W_L.001.001.260128`.
- Utiliser **Navigation Updater officiel** pour télécharger la version **correspondant strictement au modèle** sur un support local. **Ne pas lancer l'installation dans le véhicule**.
- Consigner noms, tailles, hashes et versions de paquet. La communauté signale notamment `mango-rootfs.tar.gz` et des applications chiffrées : **on ne peut pas supposer lire les drivers vidéo depuis l'archive téléchargée**.
- Sur fichiers réellement lisibles, chercher `avm`, `svm`, `rvc`, `rear`, `camera`, `video`, `drm`, `v4l2`, `wayland`. **Ne pas confondre `camera_editor` (radars routiers) et caméras de la voiture**.

### P2 — Comment accéder au Linux : découvertes et limites actuelles

- **NEMESIS** utilise `S5W_M` Android ; **incompatible** avec la HU `S5W_L` de notre véhicule.
- [`navi_extended` (g4933/gen5w)](https://gitlab.com/g4933/gen5w/navi_extended) est conçu pour remplacer temporairement `AppNavi` et permettre l'exécution de code sur un HU Linux. Il n'est pas un outil vidéo.
- Le laboratoire [laralbir/hyundai-kia-gen5w-navi-firmware](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware) documente `navi_extended`, l'étude du rootfs, et des variantes d'accès **sur un autre véhicule Kia**. Son auteur indique explicitement que l'accès initial en production peut être **bloqué par `checkSOPVersion()`** dans Engineering Mode ; les tests hors véhicule ne prouvent pas le succès sur un HU d'origine.
- Les recherches [Chimaera (2024)](https://link.springer.com/article/10.1007/s11416-024-00522-4) prouvent que des Gen5W_L plus anciens peuvent exécuter du code et charger un firmware personnalisé. **Cela ne valide pas la compatibilité ni la sécurité du procédé avec notre version `260128`.**
- La présence d'une interface de mise à jour USB **ne donne pas** directement un shell, `SSH` ou un moyen de copier une application ; **SSH n'est pas confirmé actif d'origine** sur notre voiture.
- Si une procédure requiert le **flash d'un firmware modifié**, ne la considérer **qu'après** sauvegarde/version exacte, vérification sur unité de rechange, stratégie de restauration démontrée et tests de non-régression SVM/BVM. Le flash n'est pas une étape implicite du diagnostic.

**Décision P2 en l'état : aucune procédure fiable de flash ou d'accès privilégié vérifiée pour `NE_E.EUR.S5W_L.001.001.260128` n'a été trouvée.**

### P3 — Inventaire du chemin caméra, seulement après un accès shell validé

Toutes les actions ci-dessous sont de l'**observation** ; elles ne doivent pas ouvrir de périphérique caméra en exclusivité, écrire dans `/dev`, modifier des règles Wayland ou émettre des trames CAN.

Relever l'état de base sur une HU de banc ou une HU accessible de manière validée :

```sh
uname -a
ps -ef
ls -l /dev/video* /dev/dri/* /dev/media* 2>/dev/null
cat /proc/modules
ls /sys/class/video4linux/ 2>/dev/null
ls /sys/class/drm/ 2>/dev/null
```

Adapter l'inventaire à l'outil installé (`journalctl`, `dmesg`, `dbus-monitor` en **écoute passive** si présents). Dans une session **immobilisée**, relever les événements d'affichage lors des vues Hyundai arrière puis SVM, et comparer les processus/buffers. Les logs peuvent contenir des données véhicule/personnelles : **anonymiser avant partage**.

**Questions à trancher par la mesure :**
1. Existe-t-il un récepteur/pilote vidéo et quels formats de buffers fournit-il ?
2. Le flux arrive-t-il en continu lorsque la caméra Hyundai n'est pas affichée ?
3. Le buffer montre-t-il **une vue sélectionnée/composée**, ou plusieurs flux séparés ?
4. La lecture d'un buffer **déjà produit** peut-elle être faite sans prendre possession du périphérique caméra ?
5. Peut-on transférer des images vers un stockage USB sans altérer l'affichage original ?

**Ne pas lancer par réflexe** `ffmpeg -f v4l2 -i /dev/video0` ou `v4l2-ctl --stream-mmap` sur la voiture : ouvrir l'entrée peut en prendre la propriété et bloquer une fonction Hyundai.

### P4 — Portage logiciel seulement une fois le backend prouvé

Un backend `FrameSource` abstrait les frames sans imposer Android ni `/dev/admem`. Démonstrateur minimal sur banc :

```text
[Récepteur vidéo natif Linux Gen5W_L, non identifié]
            │ lecture passive prouvée
            ▼
       FrameSource
            ▼
  Conversion -> H.264 -> segments .ts sur USB
```

Conserver la segmentation et les idées de watchdog/récupération du projet américain, **pas** ses API Android ni ses registres Telechips (incompatibles ou non démontrés). Si la HU ne fournit qu'une sortie SVM commutée, le logiciel ne pourra pas, à lui seul, produire quatre flux indépendants permanents : retour à l'analyse du **TAP actif sur quatre coaxiaux** avant l'ECU.

## Sources de travail et liens directs

- [Firmware officiel Hyundai UE, correspondance version-modèle](https://update.hyundai.com/EU/FR/updateNoticeView/software-version)
- [KramerTech — RearDashcam source et détails](https://github.com/KramerTech/gen5-RearDashcam/blob/main/TECHNICAL.md)
- [laralbir — Guide complet des outils Linux Gen5W](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware/blob/main/tools/README.md)
- [laralbir — Blocage d'accès initial au HU stock](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware/blob/main/tools/phase1_usb/README.md)
- [g4933 — `navi_extended`](https://gitlab.com/g4933/gen5w/navi_extended)
- [Chimaera — recherche scientifique Gen5W_L](https://link.springer.com/article/10.1007/s11416-024-00522-4)

**Prochain livrable attendu :** P1 réel (inventaire firmware officiel IONIQ 5). Le passage à P2 ne doit se faire qu'en présence d'une preuve de compatibilité/rollback sur un HU `S5W_L 260128`, et P3 ne sera pas présenté comme accompli sans logs de capture.
