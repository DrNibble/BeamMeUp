# CHANGES

## 2026.10.08 — House Tours : recherche initiale au chargement de l'addon

### Objectif
Le cache House Tours ne démarrait qu'au premier rafraîchissement de la liste (section 5b de `BMU.createTable` déclenchant `RequestHouseTourSearch`). La recherche initiale est désormais lancée dès le chargement de l'addon.

### Modifications
- `BeamMeUp/BeamMeUp.lua` (`OnAddOnLoaded`, après l'enregistrement du rappel de recherche House Tours) : si le paramètre `showHouseTours` est activé, appel direct de `BMU.RequestHouseTourSearch()`.
  - La recherche s'exécute côté serveur ; l'enrichissement des résultats reste en tâche de fond via LibAsync (`BMU.buildHouseToursCacheEntries`), donc le temps de démarrage n'est pas pénalisé.
  - `refreshListAuto` est sans effet tant que la fenêtre du téléporteur est cachée (garde existante), donc les rafraîchissements déclenchés par l'arrivée des résultats sont inoffensifs pendant la connexion.
  - Les garde-fous existants s'appliquent : `RequestHouseTourSearch` ignore les appels redondants (état « recherche en cours »), et le paramètre `showHouseTours` désactivé ne lance aucune recherche.

### Limites
- Vérifié par analyse statique et test automatisé avec API ESO simulées (lancement initial, absence de recherche dupliquée, remplissage en tâche de fond, libération de l'état) ; non testé en jeu.

## 2026.10.08 — House Tours : enrichissement du cache en tâche de fond

### Objectif
À la fin d'une recherche House Tours (BROWSE), le callback `onHouseTourSearchComplete` enrichissait chaque listing de façon synchrone (appels API ESO : zone de la maison, zone parente, noms formatés, surnom, index de carte, `applyHouseFixedMapData`, cas spéciaux 102/124...) ; avec plusieurs centaines de maisons, ce travail bloquait la frame. L'enrichissement est désormais exécuté en tâche de fond via LibAsync, sur le modèle du cache des maisons possédées.

### Modifications
- `BeamMeUp/core/TeleporterChecker.lua`
  - `onHouseTourSearchComplete` ne collecte plus que les données brutes peu coûteuses (houseId, propriétaire, nom, collectibleId), avec les mêmes exclusions qu'avant (maisons possédées, doublons par houseId, propres listings du joueur).
  - Nouvelles fonctions :
    - `BMU.enrichHouseTourListing` : calcule tous les champs statiques d'un listing brut (identiques à l'ancien chemin, y compris la variante « world map » de la maison 102 et les remplacements de zone parente) ; protégée par pcall — un listing en échec est ignoré sans interrompre la construction.
    - `BMU.buildHouseToursCacheEntries` : répartit l'enrichissement sur plusieurs frames via LibAsync (tâche `BMU_HouseToursCache`) ; une nouvelle recherche pendant l'enrichissement annule la tâche en cours (garde par génération : une tâche annulée ne peut jamais finaliser la construction de sa remplaçante, et la finalisation n'a lieu qu'une seule fois même si le rappel de fin échoue) ; sans LibAsync, repli synchrone.
    - `BMU.finishHouseTourSearch` : fusion (déduplication houseId + zone de contexte, entrées existantes en premier) puis enchaînement des lots de filtre, comme avant.
  - L'état « recherche en cours » (`houseTourSearchPending`) est maintenu pendant tout l'enrichissement : `RequestHouseTourSearch` reste bloqué, afin qu'aucune recherche concurrente ne puisse relancer la chaîne de lots (index de lot, filtres du jeu) en pleine construction ; il est libéré à la finalisation (ou sur les chemins d'erreur).
  - Pour la maison 102, la variante « world map » est désormais construite avant l'insertion : une erreur pendant sa construction ignore la maison entière au lieu de publier une entrée incomplète.
  - Le cache précédent reste utilisé tant que l'enrichissement n'est pas terminé ; la chaîne de lots se poursuit même si des listings échouent, afin que les filtres House Tours du jeu soient toujours restaurés.
- `BeamMeUp/TeleporterGlobals.lua` : état de l'enrichissement en tâche de fond (`houseTourCacheBuilding`, `houseTourCacheBuildTask`, `houseTourCacheBuildGeneration`) et mise à jour du commentaire de `houseTourListings` (listings enrichis).
- Commentaire de la section 5b de `BMU.createTable` mis à jour (enrichissement en tâche de fond).

### Limites
- Vérifié par analyse statique et tests automatisés avec API ESO simulées (enrichissement LibAsync réparti sur plusieurs frames, publication, blocage des recherches concurrentes pendant la construction, annulation d'une tâche en cours avec publication exacte des résultats de la nouvelle tâche, transition entre lots, erreur dans la finalisation sans double finalisation, repli synchrone sans LibAsync, exclusions/déduplication, cas 102/124, erreur par listing ignorée) ; non testé en jeu. Les tests avec ordonnanceur simulé valident l'enchaînement des opérations, pas une mesure de performance en frames.

## 2026.10.08 — Optimisation de l'affichage des maisons

### Objectif
L'affichage des maisons possédées (liste principale et onglet Maisons) recalculait, à chaque rafraîchissement, l'ensemble des données statiques de chaque maison via les API ESO (GetHouseZoneId, GetCollectibleIdForHouse, GetZoneNameById, GetCollectibleDefaultNickname, GetCollectibleInfo, recherches LibZone, formatage des noms...). Ces données sont statiques pour une session de jeu : elles sont désormais calculées une seule fois, au lancement, dans un cache rempli en tâche de fond.

### Modifications
- `BeamMeUp/core/TeleporterChecker.lua`
  - Nouveau cache `BMU.ownHousesCache` : entrées statiques par maison (identifiants de maison/zone/collectible, noms de zone formatés et non formatés — avec et sans articles —, surnom par défaut formaté, carte parente, catégorie, type de catégorie, icône, image d'aperçu), construit en tâche de fond au démarrage via LibAsync (`BMU.startOwnHousesCacheBuild`, tâche `BMU_OwnHousesCache` répartie sur plusieurs frames).
  - La liste principale (`BMU.createTable`, section « Own houses ») et l'onglet Maisons (`BMU.createTableHouses`) copient désormais les valeurs statiques depuis le cache ; le surnom (renommable), la résidence principale et le nombre de meubles restent lus en direct. Les filtres, tris et enrichissements par entrée (addInfo_2, applyHouseFixedMapData) restent appliqués à chaque rafraîchissement.
  - L'ordre de parcours des maisons du cache est identique au chemin direct (l'énumération d'origine est conservée, pas de tri par identifiant), afin que la maison retenue par zone (paramètre « une seule entrée par zone » / préférence de maison par zone) ne change pas.
  - Reconstruction automatique en tâche de fond lors des changements de collection, avec anti-rebond de 2 s et mise en file d'attente si une construction est déjà en cours (jamais perdue). Événements écoutés (conformes au gestionnaire de collectibles du jeu, [collectibledatamanager.lua](https://raw.githubusercontent.com/esoui/esoui/live/esoui/ingame/collections/collectibledatamanager.lua)) : EVENT_COLLECTIBLE_UPDATED (renommage), EVENT_COLLECTIBLES_UNLOCK_STATE_CHANGED (déverrouillages : achats) et EVENT_COLLECTION_UPDATED (rafraîchissement complet).
  - Repli sûr : tant que le cache n'est pas construit (ou en cas d'erreur pendant la construction — chaque maison est traitée de façon protégée et le cache existant est conservé), l'ancien chemin de code est utilisé — les listes restent correctes en toutes circonstances.
  - Correction au passage : dans la liste principale, `houseNameFormatted` était calculé avec un identifiant de collectible lu avant son affectation ; l'identifiant est désormais affecté avant l'appel, comme dans l'onglet Maisons.
- `BeamMeUp/BeamMeUp.lua` : lancement de la construction du cache à la première activation du joueur (`PlayerInitAndReady`) et enregistrement des gestionnaires d'événements de collection.
- `BeamMeUp/TeleporterGlobals.lua` : déclaration de l'état du cache.
- LibAsync 3.1.5 intégrée dans l'addon (`BeamMeUp/lib/LibAsync/LibAsync.lua`) avec une garde anti-double-chargement et initialisation directe des variables (non persistantes) : si une LibAsync autonome est installée, elle est utilisée à la place de la copie intégrée (déclarée en `OptionalDependsOn`). Sans LibAsync, repli sur une construction synchrone (peu coûteuse : seules les maisons possédées sont parcourues).
- `BeamMeUp.addon.template` / `BeamMeUp/BeamMeUp.addon` : entrée du fichier LibAsync intégré et dépendance optionnelle.

### Limites
- Vérifié par analyse statique et tests automatisés avec API ESO simulées (construction, publication, erreur, reconstruction, équivalence des chemins avec/sans cache, ordre de parcours identique) ; non testé en jeu. Les tests avec ordonnanceur réel valident l'enchaînement des opérations, pas une mesure de performance en frames.
