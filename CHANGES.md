# CHANGES

## 2026.10.08 — House Tours: initial search at addon load

### Goal
The House Tours cache only started on the first list refresh (section 5b of `BMU.createTable` triggering `RequestHouseTourSearch`). The initial search is now started as soon as the addon loads.

### Changes
- `BeamMeUp/BeamMeUp.lua` (`OnAddOnLoaded`, right after the House Tours search callback registration): if the `showHouseTours` setting is enabled, `BMU.RequestHouseTourSearch()` is called directly.
  - The search runs on the server; the enrichment of the results stays in the background via LibAsync (`BMU.buildHouseToursCacheEntries`), so the startup time is not penalized.
  - `refreshListAuto` is a no-op while the teleporter window is hidden (existing guard), so the refreshes triggered when results arrive are harmless during login.
  - The existing safeguards apply: `RequestHouseTourSearch` ignores redundant calls ("search in progress" state), and a disabled `showHouseTours` setting starts no search at all.

### Limits
- Verified through static analysis and an automated test with simulated ESO APIs (initial launch, no duplicate search, background fill, state release); not tested in game.

## 2026.10.08 — House Tours: background cache enrichment

### Goal
At the end of a House Tours BROWSE search, the `onHouseTourSearchComplete` callback used to enrich every listing synchronously (ESO API calls: house zone, parent zone, formatted names, nickname, map index, `applyHouseFixedMapData`, special cases 102/124...); with several hundred houses this work blocked the frame. The enrichment now runs in the background via LibAsync, following the model of the owned houses cache.

### Changes
- `BeamMeUp/core/TeleporterChecker.lua`
  - `onHouseTourSearchComplete` now only collects the cheap raw data (houseId, owner, name, collectibleId), with the same exclusions as before (owned houses, duplicates by houseId, the player's own listings).
  - New functions:
    - `BMU.enrichHouseTourListing`: computes all static fields of one raw listing (identical to the old path, including the house 102 "world map" variant and the parent zone overrides); protected by pcall — a failing listing is skipped without interrupting the build.
    - `BMU.buildHouseToursCacheEntries`: spreads the enrichment over several frames via LibAsync (task `BMU_HouseToursCache`); a new search during the enrichment cancels the running task (generation guard: a cancelled task can never finalize its replacement's build, and the finalization runs exactly once even if the completion callback fails); without LibAsync, synchronous fallback.
    - `BMU.finishHouseTourSearch`: merge (deduplication by houseId + context zone, existing entries first) then the filter batch chain, as before.
  - The "search in progress" state (`houseTourSearchPending`) is held for the whole enrichment: `RequestHouseTourSearch` stays blocked, so no concurrent search can restart the batch chain (batch index, game filters) in the middle of a build; it is released at finalization (or on the error paths).
  - For house 102, the "world map" variant is now built before insertion: an error while building it skips the whole house instead of publishing an incomplete entry.
  - The previous cache stays in use until the enrichment finishes; the batch chain continues even when listings fail, so the game's House Tours filters are always restored.
- `BeamMeUp/TeleporterGlobals.lua`: background enrichment state (`houseTourCacheBuilding`, `houseTourCacheBuildTask`, `houseTourCacheBuildGeneration`) and updated comment for `houseTourListings` (enriched listings).
- Updated the comment of section 5b of `BMU.createTable` (background enrichment).

### Limits
- Verified through static analysis and automated tests with simulated ESO APIs (LibAsync enrichment spread over several frames, publication, blocking of concurrent searches during the build, cancellation of a running task with the new task's results published exactly, batch transition, error in the finalization without double finalization, synchronous fallback without LibAsync, exclusions/deduplication, cases 102/124, per-listing error skipped); not tested in game. The tests with a simulated scheduler validate the sequence of operations, not a frame-level performance measurement.

## 2026.10.08 — Optimized house display

### Goal
The display of owned houses (main list and Houses tab) used to recompute, on every refresh, all the static data of each house through the ESO APIs (GetHouseZoneId, GetCollectibleIdForHouse, GetZoneNameById, GetCollectibleDefaultNickname, GetCollectibleInfo, LibZone lookups, name formatting...). This data is static for a game session: it is now computed once, at startup, in a cache filled in the background.

### Changes
- `BeamMeUp/core/TeleporterChecker.lua`
  - New `BMU.ownHousesCache`: static entries per house (house/zone/collectible identifiers, formatted and unformatted zone names — with and without articles —, formatted default nickname, parent map, category, category type, icon, preview background image), built in the background at startup via LibAsync (`BMU.startOwnHousesCacheBuild`, task `BMU_OwnHousesCache` spread over several frames).
  - The main list (`BMU.createTable`, "Own houses" section) and the Houses tab (`BMU.createTableHouses`) now copy the static values from the cache; the (renameable) nickname, the primary residence and the furniture count are still read live. Filters, sorting and per-entry enrichment (addInfo_2, applyHouseFixedMapData) are still applied on every refresh.
  - The traversal order of the cached houses is identical to the direct path (the original enumeration is kept, no sorting by identifier), so the house kept per zone ("one entry per zone only" setting / per-zone house preference) does not change.
  - Automatic background rebuild on collection changes, with a 2 s debounce and queueing if a build is already running (never lost). Events listened to (matching the game's collectible manager, [collectibledatamanager.lua](https://raw.githubusercontent.com/esoui/esoui/live/esoui/ingame/collections/collectibledatamanager.lua)): EVENT_COLLECTIBLE_UPDATED (renames), EVENT_COLLECTIBLES_UNLOCK_STATE_CHANGED (unlocks: purchases) and EVENT_COLLECTION_UPDATED (full refresh).
  - Safe fallback: as long as the cache is not built (or on an error during the build — each house is processed in a protected way and the existing cache is kept), the old code path is used — the lists stay correct in all circumstances.
  - Fixed along the way: in the main list, `houseNameFormatted` was computed with a collectible identifier read before its assignment; the identifier is now assigned before the call, as in the Houses tab.
- `BeamMeUp/BeamMeUp.lua`: cache build started at the player's first activation (`PlayerInitAndReady`) and registration of the collection event handlers.
- `BeamMeUp/TeleporterGlobals.lua`: declaration of the cache state.
- LibAsync 3.1.5 embedded in the addon (`BeamMeUp/lib/LibAsync/LibAsync.lua`) with a double-load guard and direct (non-persistent) variable initialization: if a standalone LibAsync is installed, it is used instead of the embedded copy (declared as `OptionalDependsOn`). Without LibAsync, fallback to a synchronous build (cheap: only the owned houses are iterated).
- `BeamMeUp.addon.template` / `BeamMeUp/BeamMeUp.addon`: entry for the embedded LibAsync file and optional dependency.

### Limits
- Verified through static analysis and automated tests with simulated ESO APIs (build, publication, error, rebuild, path equivalence with/without cache, identical traversal order); not tested in game. The tests with a real scheduler validate the sequence of operations, not a frame-level performance measurement.
