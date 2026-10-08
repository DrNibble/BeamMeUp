## 2026.10.08
- Optimisation de l'affichage des maisons : la base des maisons possédées (zone, collectible, noms, icône, catégorie, carte parente...) est désormais construite une seule fois dans un cache, en tâche de fond au lancement de l'addon, au lieu d'être recalculée à chaque rafraîchissement de la liste principale et de l'onglet Maisons. Les filtres, tris et enrichissements par entrée (addInfo_2) restent appliqués à chaque rafraîchissement.
- Le remplissage du cache est réparti sur plusieurs frames via LibAsync (intégrée à l'addon, `lib/LibAsync/LibAsync.lua`) afin de ne pas pénaliser le temps de démarrage ; si une LibAsync autonome est déjà installée, elle est utilisée à la place de la copie intégrée.
- Le surnom de chaque maison, la résidence principale et le nombre de meubles restent lus en direct à l'affichage (renommage et changements immédiatement visibles) ; l'achat d'une maison ou un renommage déclenchent une reconstruction du cache en tâche de fond, avec anti-rebond de 2 s et mise en file d'attente jamais perdue si une construction est déjà en cours (événements EVENT_COLLECTIBLE_UPDATED, EVENT_COLLECTIBLES_UNLOCK_STATE_CHANGED et EVENT_COLLECTION_UPDATED, conformément au gestionnaire de collectibles du jeu).
- L'ordre de parcours des maisons du cache est identique au chemin direct, afin que la maison retenue par zone (paramètre « une seule entrée par zone » et préférence de maison par zone) ne change pas.
- Tant que le cache n'est pas prêt, l'ancien chemin de code est utilisé, donc les listes restent correctes dans tous les cas ; en cas d'erreur pendant la construction, le cache existant est conservé.
- Correction au passage : dans la liste principale, le nom formaté des maisons possédées (`houseNameFormatted`) était calculé avec un identifiant de collectible lu avant son affectation ; l'identifiant est désormais affecté avant l'appel, comme dans l'onglet Maisons.

## 2026.03.08
- Add gamepad support and ci/cd
