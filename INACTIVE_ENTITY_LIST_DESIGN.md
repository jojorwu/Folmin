# Inactive Entity List Management for Folia Regions

## I. Proposed System Overview

### Goal:
To improve server performance in Folia by reducing the number of entities that are actively ticked every server tick within each region. This is achieved by separating entities into "active" and "inactive" lists, where inactive entities are ticked much less frequently.

### Core Idea:
Instead of a single `entityTickList` in `RegionizedWorldData`, entities will be managed in two primary lists:
1.  `activeEntityTickList`: Entities in this list are ticked every server tick as they are now.
2.  `inactiveEntityList`: Entities in this list are ticked on a much slower, configurable interval (e.g., every 20-100 ticks).

A mechanism will be implemented to transition entities between these two states based on proximity to players and other activity heuristics (like Spigot's activation range, but on a per-region basis).

## II. Data Structure Modifications in `RegionizedWorldData`

The following changes would be made to `RegionizedWorldData.java` (or its equivalent if introduced by a patch):

1.  **`activeEntityTickList` (replaces `entityTickList`):**
    *   Type: `IteratorSafeOrderedReferenceSet<Entity>` (or similar, as currently used for `entityTickList`).
    *   Purpose: Holds entities that are currently active and require ticking every server tick.

2.  **`inactiveEntityList`:**
    *   Type: `IteratorSafeOrderedReferenceSet<Entity>` (or a more memory-efficient structure if direct iteration order isn't strictly critical for inactive entities, though `IteratorSafeOrderedReferenceSet` is robust).
    *   Purpose: Holds entities that are considered inactive and will be ticked less frequently.

3.  **`entityInactiveTickCounters`:**
    *   Type: `Map<Entity, Integer>` (e.g., `Reference2IntOpenHashMap<Entity>`).
    *   Purpose: Stores a counter for each entity in the `inactiveEntityList`. When an entity is added to `inactiveEntityList`, its counter is initialized to a random value between `0` and `inactiveTickInterval - 1`. Each tick that the `inactiveEntityList` is processed, entities whose counter reaches `0` (or a target value) are ticked, and their counter is reset to `inactiveTickInterval - 1`. This distributes the ticking of inactive entities over time.
    *   Alternatively, instead of individual counters, a single regional counter could be used, and a subset of the `inactiveEntityList` could be ticked each time the regional counter wraps around (e.g., tick 1/Nth of the list every tick, processing the whole list over N ticks). However, individual counters allow for more varied tick rates if ever needed and simpler merge/split logic.

## III. Entity State Transition Logic

A configurable `inactiveTickInterval` (e.g., `TICKS_BETWEEN_INACTIVE_ENTITY_TICKS = 40`) would be introduced.

### 1. Becoming Inactive:
*   **Periodic Scan (in the regional entity ticking loop):**
    *   After ticking entities in `activeEntityTickList`, iterate through them.
    *   If an entity no longer meets activation criteria (e.g., no players within X blocks for Y seconds, similar to Spigot's activation ranges but adapted for Folia's regional awareness), move it from `activeEntityTickList` to `inactiveEntityList`.
    *   Initialize its counter in `entityInactiveTickCounters` to `random.nextInt(inactiveTickInterval)`.
*   This check should ideally be efficient, possibly leveraging the `NearbyPlayers` structure already present in `RegionizedWorldData`.

### 2. Becoming Active:
*   **Periodic Scan (during the inactive entity tick processing):**
    *   When an entity from `inactiveEntityList` is selected to be ticked (its counter reached 0):
        *   Before its actual tick, check if it now meets activation criteria (e.g., a player has moved nearby).
        *   If yes, move it from `inactiveEntityList` to `activeEntityTickList` and remove its entry from `entityInactiveTickCounters`. Then, tick it immediately as part of the active list processing for that tick (or ensure it's ticked in the next active cycle).
*   **Optional: Event-Driven Activation:**
    *   Certain events could trigger an immediate check for nearby inactive entities to become active. For example:
        *   Player movement into a new area.
        *   Explosions or other significant game events.
    *   This would involve querying entities in `inactiveEntityList` within a certain radius of the event and checking their activation criteria. This makes the system more responsive but adds complexity.

### 3. New Entities:
*   When a new entity is added to the world (e.g., spawned, loaded from chunk):
    *   Check its activation criteria immediately.
    *   Add it to `activeEntityTickList` or `inactiveEntityList` accordingly.

### 4. Entity Removal:
*   When an entity is removed from the world:
    *   It must be removed from whichever list it currently resides in (`activeEntityTickList` or `inactiveEntityList`) and also from `entityInactiveTickCounters` if present.

### 5. Region Merges/Splits (within `RegionizedWorldData.REGION_CALLBACK`):
*   **Merging `from` region into `into` region:**
    *   `into.activeEntityTickList.addAll(from.activeEntityTickList)`
    *   `into.inactiveEntityList.addAll(from.inactiveEntityList)`
    *   For `entityInactiveTickCounters`: Iterate `from.entityInactiveTickCounters`. For each entity and its counter `c_from`:
        *   If `into.entityInactiveTickCounters` already contains the entity (should be rare if entities are unique across merging regions unless it's a complex merge scenario), decide on a strategy (e.g., take `Math.max(c_from, c_into)` or `Math.min`).
        *   Otherwise, add the entity to `into.entityInactiveTickCounters` with its counter `c_from`. The `fromTickOffset` might be used to adjust these counters if the `inactiveTickInterval` is very large and regions have been ticking independently for a long time, to maintain a semblance of the original schedule: `new_counter = (c_from - (fromTickOffset % inactiveTickInterval) + inactiveTickInterval) % inactiveTickInterval`.
*   **Splitting `from` region into `dataSet` (via `regionToData` lookup):**
    *   For each entity in `from.activeEntityTickList`: Determine its new region based on its chunk coordinates and add it to the corresponding `activeEntityTickList` in `regionToData`.
    *   For each entity in `from.inactiveEntityList`: Determine its new region and add it to the corresponding `inactiveEntityList`.
    *   For each entry in `from.entityInactiveTickCounters`: Determine the entity's new region and add its counter to the corresponding `entityInactiveTickCounters`. Tick counters do not need adjustment as new split regions typically start at the same tick number.

## IV. Impact on Regional Entity Ticking Loop (in `ServerLevel.java`'s Folia equivalent)

The main entity ticking loop for a region would be modified:

1.  **Tick Active Entities:** Iterate and tick all entities in `activeEntityTickList` as usual.
2.  **Transition Active to Inactive:** After ticking, scan `activeEntityTickList` for entities that should become inactive and move them.
3.  **Process Inactive Entities:**
    *   Decrement the region's internal counter or iterate through `entityInactiveTickCounters`.
    *   For each entity selected for an inactive tick:
        *   Check if it should become active. If so, move it to `activeEntityTickList`, remove from `entityInactiveTickCounters`, and tick it (either immediately or in the next active cycle).
        *   If it remains inactive, perform its reduced tick (e.g., simplified AI, less frequent environmental checks).
        *   Reset its counter in `entityInactiveTickCounters` to `inactiveTickInterval - 1`.

## V. Performance Considerations

### Benefits:
*   **Reduced CPU Load:** Significantly fewer entities are fully ticked every server tick, leading to lower CPU usage per region, especially in areas with many entities but few players.
*   **Improved Scalability:** Allows regions to support a higher number of total entities without proportionally increasing the per-tick processing cost.
*   **Smoother Gameplay:** Less CPU strain can lead to more consistent tick times and smoother gameplay experience, especially on servers with high entity counts.

### Overheads:
*   **List Management:** Moving entities between `activeEntityTickList` and `inactiveEntityList` introduces some overhead.
*   **Counter Management:** Updating and checking `entityInactiveTickCounters` adds minor processing.
*   **Activation Scans:** Periodically scanning entities to determine if they should change state (active <-> inactive) has a cost. The efficiency of these scans is crucial.
*   **Memory Usage:** Maintaining two lists and the `entityInactiveTickCounters` map will slightly increase memory usage per region compared to a single list.

### Net Impact:
The expectation is a **significant net performance gain**, especially on servers or in regions with a high density of entities that are not always in direct interaction with players (e.g., large farms, mob grinders far from players, entities in unexplored chunks within a region's boundary). The overheads are likely to be outweighed by the savings from not ticking most entities every tick. The `inactiveTickInterval` provides a knob to balance performance gains against the responsiveness of inactive entities.
