# CHANGES

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
