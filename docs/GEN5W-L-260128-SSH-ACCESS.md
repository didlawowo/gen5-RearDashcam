# Gen5W Linux EU `260128` — accès initial, SSH et diagnostic caméra

**Véhicule confirmé par photo :** IONIQ 5 2024 EU, SVM 360°, HU `NEAE.S5BLC.AEU`, firmware **`NE_E.EUR.S5W_L.001.001.260128`**. Recherches du **30 octobre 2026**.

## Verdict : pas encore de procédure SSH validée sur ce firmware

- **NEMESIS** cible les Gen5W **Android** `S5W_M`, pas notre `S5W_L` Linux : [NEMESIS README](https://github.com/xwtk/hmg_nemesis).
- [Gen5W WideOpen Wifi+SSH](https://gitlab.com/g4933/gen5w/main-loops/wifi-ssh) (2022–2023) construit une image Docker puis un `USB_PACKAGE` contenant un script/launcher Wi-Fi + SSH. **Ce n'est pas un accès initial autonome** : l'environnement `main_loop`/l'exécution arbitraire doivent déjà être présents. Son README indique « How to SSH into your car: TBA ».
- [navi_extended](https://gitlab.com/g4933/gen5w/navi_extended) remplace temporairement `AppNavi` pour exécuter des scripts USB, mais sa mise en place initiale exige `Engineering Mode → Dynamics → Navigation → Config → Update AppNavi from USB`.
- Le projet [laralbir/hyundai-kia-gen5w-navi-firmware](https://github.com/laralbir/hyundai-kia-gen5w-navi-firmware/blob/main/tools/phase1_usb/README.md), fondé sur un autre modèle S5W_L européen, **qualifie précisément cette entrée initiale de principal blocage** pour les firmwares `MASS_PRODUCT`; son analyse QML décrit un `checkSOPVersion()` empêchant Engineering Mode. Il **n'a pas** prouvé d'accès sur l'IONIQ 5 `260128`.
- Un [propriétaire d'IONIQ 5 avec **exactement le même firmware** (25/10/2026)](https://www.reddit.com/r/Ioniq5/comments/1xfc9zh/engineering_mode_on_latest_gen5w_software/) décrit l'échec des anciennes méthodes d'Engineering Mode ; il n'atteint que Dealer Mode. La discussion ne valide aucun contournement.
- **Aucun exploit, package, restauration, IP, port ou identifiant SSH fonctionnant réellement sur cette version précise n'a été vérifié**. Ne pas proposer une manipulation de firmware comme opération sûre.

## Test non destructif possible à l'arrêt (source communautaire, non confirmé sur cette IONIQ 5)

La [page Oviradio datée d'octobre 2026](https://www.oviradio.cz/hyundai-kia-radio-engineering-mode-en/) donne pour **Full Engineering Mode (2 molettes)** les étapes :

1. Voiture **immobilisée**, sans téléchargement USB ni mise à jour en cours.
2. **Ouvrir Radio** (pas `Infos logiciel`).
3. Molette gauche volume : **0**, puis **7** et appui sur molette droite ; **3** et appui droite ; **1** et appui droite, le tout rapidement.
4. Si une invite de code apparaît, **`19480717`** est le code annoncé pour les firmwares EUR `250604`, `251204` et `260128`.
5. **Se limiter à la lecture du menu**. Photographier les rubriques; chercher l'intitulé de menu indiqué par `navi_extended`, sans déclencher `Update AppNavi from USB` ni modifier une option.

**Réserve majeure :** même si une fenêtre PIN apparaît, le code peut échouer en firmware production. Des guides en ligne confondent **Dealer Mode** (`2400`) et **Engineering Mode** ; seul ce dernier offrirait éventuellement un accès initial. Un accès au menu ne prouve pas encore que l'option `Update AppNavi` soit disponible.

## Si et seulement si Engineering Mode permet `Update AppNavi from USB`

Chaîne historique documentée, **à vérifier sur banc / stock HU et avec stratégie de récupération avant exécution** :

```text
Engineering Mode + Update AppNavi from USB
     -> installer temporairement navi_extended
     -> exécution main_loop.sh depuis USB (lanceur)
     -> installer/démarrer un main_loop Wi-Fi + Dropbear SSH
     -> associer la HU au réseau Wi-Fi dédié
     -> identifier sa véritable adresse IP + mode d'authentification
     -> connexion SSH depuis le Mac
     -> inventorier les pilotes/processus vidéo, puis observer SVM en lecture seule
```

**Ne pas exécuter maintenant** : compatibilité `NE_E.EUR.S5W_L...260128` non démontrée, rollback inconnu et entrée inaccessible selon plusieurs retours. Le workflow de patch permanent du `rootfs` comporte des modifications du système de mise à jour/chiffrement, et n'est pas un prérequis acceptable pour un simple essai caméra.

## Option utile tout de suite : collecte de journaux sans SSH ni flash

Une [note de service Kia PS677_R1 (2021)](https://oemdtc.com/tsb/10196825/) décrit pour AVN 5.0/W deux façons d'**exporter des logs** sur USB :
- Après avoir ouvert et fermé la vue SVM en voiture **à l'arrêt**, brancher une clé de données puis **maintenir le bouton physique Radio > 10 s** jusqu'à `LogDump Success`, puis `Copy all to USB` si proposé. Format attendu : `.txt.gz`.
- Si indisponible, le **Dealer Mode** (`2400`) peut proposer `Copy image to USB` (ce terme désigne ici une image/archive **de logs**, pas une image caméra !) avec sortie `.tar.gz`.

Attention : cette note concerne les AVN 5.0/W **Kia d'Amérique du Nord**, ancienne génération, **non validée pour la HU EU 2024**. Il faut vérifier les libellés sans appuyer sur des fonctions de flash, diagnostic actif, recalibration ou modification de paramètres.

Les journaux pourraient donner des mots clés **`avm`, `svm`, `rvc`, `camera`, `video`, `v4l`, `wayland`, `drm`, `lvds`**, mais **aucune garantie** qu'ils exposent ces détails. Ne publier jamais les logs bruts : VIN/identifiants de périphériques, historique, localisation, clés et réseau peuvent y figurer.

## Décision

La **prochaine action en voiture** est une inspection *lecture seule* du Full Engineering Mode (ou des fonctions d'export de logs). Si le Full Engineering n'est pas accessible, **ne pas improviser de flash/downgrade** : poursuivre l'analyse hors véhicule des firmwares officiels et concentrer le prototypage caméra sur une caméra de rechange + TAP actif coaxial en banc. L'objectif Ioniq Control reste **capturer les flux vidéo**, pas « rooter » pour rooter.
