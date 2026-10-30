# Investigation : capture vidéo SVM sur Hyundai IONIQ 5 (EU 2024)

> État : exploration technique, **aucune compatibilité SVM démontrée**. Ne pas installer ce fork sur une unité SVM et ne pas retirer les gardes de l'upstream.

## Mise à jour compatibilité (30/10/2026)

Photo des paramètres de l'unité multimédia de ce véhicule : `Standard-class Gen5W Navigation`, modèle `NEAE.S5BLC.AEU`, version `NE_E.EUR.S5W_L.001.001.260128` (mise à jour indiquée `06/05/2026`). Le `S5W_L` confirme **Gen5W Linux européen** : **NEMESIS et l'application Android RearDashcam ne peuvent pas y être installés tels quels**. Préserver le fork pour référence et réutilisation conceptuelle, et investiguer plutôt la pile vidéo Linux ou le TAP coaxial hors véhicule.

## Contexte et objectif

Véhicule cible : Hyundai IONIQ 5 européenne 2024, HDA2, vision 360° / SVM, 4 caméras (avant, arrière, gauche, droite), rétroviseurs classiques (pas de DSM).

But à terme : réutiliser le code d'enregistrement vidéo de `gen5-RearDashcam` pour Ioniq Control, en récupérant idéalement **les quatre flux indépendants**, sans perturber les fonctions Hyundai de vision 360°, stationnement ou angle mort.

Upstream : <https://github.com/KramerTech/gen5-RearDashcam> (licence MIT, copyright Sean Kramer). Documentation : [README](https://github.com/KramerTech/gen5-RearDashcam/blob/main/README.md), [TECHNICAL](https://github.com/KramerTech/gen5-RearDashcam/blob/main/TECHNICAL.md).

## Faits vérifiés dans la documentation de l'upstream

1. L'upstream fonctionne avec une caméra de recul non-SVM, en utilisant la HAL `android.hardware.RearCamera`, un tampon `rearcamera` accessible via `/dev/admem`, un encodeur H.264 matériel et un stockage USB MPEG-TS.
2. Sur les véhicules SVM testés par l'auteur, `RearCamera.getCameraType()` renvoie **6** (contre **0** pour sa caméra arrière analogique).
3. Côté **unité multimédia**, l'auteur observe un nœud device tree `tcc_camera/max96706_lvds`. Cette information concerne **la liaison recevant la vidéo dans le head unit**, et **ne démontre pas** le protocole de chacune des quatre caméras avant l'ECU SVM.
4. Les essais logiciels de l'auteur sur SVM produisent des frames gris sombre à l'ouverture directe de la caméra. Des frames valides arrivent dans le tampon lorsque **le système Hyundai affiche une vue caméra**.
5. L'auteur constate que la commande du bouton caméra arrive au contrôleur de stationnement, lequel active la vidéo et informe le head unit via CAN.
6. Pour ces raisons, le code d'installation et l'enregistreur **refusent le type 6**. Ce refus est une protection délibérée, à préserver tant qu'aucune solution sûre n'a été validée.

**Conséquence majeure :** récupérer le tampon vidéo du head unit, même de façon fiable, n'assure ni une vidéo continue ni l'accès séparé aux quatre caméras. La voie matérielle (TAP en amont de l'ECU SVM) et la voie logicielle (flux effectivement accessibles au HU) doivent être investiguées séparément.

## Hypothèses à confirmer, et non des faits acquis

- Les quatre liaisons coaxiales entre caméras et ECU SVM pourraient être GMSL, FPD-Link, GVIF ou un protocole différent ; **inconnu à ce stade**.
- Une caméra alimentée n'émet pas nécessairement un flux continu ; la possibilité d'enregistrer les quatre vues en permanence reste **non démontrée**.
- L'ECU SVM/ADAS Parking pourrait traiter, sélectionner ou composer les quatre images avant de transmettre une seule vue au head unit.
- Les références de caméra répertoriées précédemment doivent être revérifiées contre le VIN/étiquette avant achat.

## Plan de travail proposé

### Phase A : étude du code sans installation

- Identifier les implémentations et usages de `RearCamera.getCameraType()`, les protections `preinstall`, `CameraCheck`, le parcours `Camera Debug` et les buffers vidéo.
- Distinguer ce qui relève de la **capture de frames**, de l'**encodage**, de l'**archivage** et de l'**interface de lecture**.
- Préparer une abstraction `FrameSource` pour réutiliser H.264, segments, stockage USB, lecteur, en y branchant plus tard une source SVM ou externe.

### Phase B : analyse des possibilités SVM

- Construire la carte des flux : quatre coaxiaux vers ECU SVM, puis liaison vidéo vers head unit et éventuel combiné.
- Rechercher les indices sur le contrôle des flux (HAL, CAN, signaux d'alimentation), **sans envoyer de commandes en véhicule** à cette étape.
- Documenter ce qu'une capture uniquement dans le head unit peut ou ne peut pas récupérer.

### Phase C : investigation de l'interposeur matériel (sur banc)

- Identifier le sérialiseur d'une caméra Hyundai de rechange et les connecteurs automobiles exacts.
- Confirmer le protocole SerDes, les vitesses, le Power-over-Coax éventuel, la topologie de contrôle et les exigences de continuité.
- Comparer un module TAP actif compatible à un prototype de réception + retransmission. **Pas de Y coaxial passif** présumé compatible.
- Valider un canal hors véhicule avant d'envisager quatre voies.

## Prérequis de sécurité et critères d'acceptation

- [ ] Produire une cartographie des modules source du fork et un plan de refactorisation.
- [ ] Qualifier clairement la limite d'une capture en aval du SVM : vue composée/sélectionnée vs quatre flux bruts.
- [ ] Définir des diagnostics **en lecture seule** et un banc de test reproductible.
- [ ] Vérifier le protocole des caméras par documentation du fabricant ou inspection d'une pièce de rechange, avant achat d'un désérialiseur.
- [ ] Définir une politique de panne qui **n'affecte jamais les fonctions vidéo Hyundai**.
- [ ] Préserver les protections type 6, la licence MIT et l'attribution à Sean Kramer.

## Explicitement hors périmètre pour l'instant

- Installation sur la voiture ou son système multimédia.
- Contournement de `preinstall.sh` ou forçage d'une ouverture caméra.
- Modification des commandes CAN en circulation.
- Modifications en ligne des signaux coaxiaux sans validation sur banc.
