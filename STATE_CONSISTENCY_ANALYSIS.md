## Folia State Management Inconsistency Analysis

This document analyzes potential areas where state management inconsistencies could arise in Folia's architecture, focusing on the interplay between global and regional states, entity transitions, and other distributed data systems.

### 1. Global vs. Regional State Synchronization

*   **Identified Global States:** Game rules, world border settings, server time (managed by `GlobalRegion` via `RegionizedServer` and propagated as `RegionizedServer.WorldLevelData`), weather state.
*   **Propagation Mechanism:** Regions receive a snapshot of `WorldLevelData` (includes time) at the start of their tick via `RegionizedWorldData.updateTickData()`. Game rules and world border are likely read when needed, potentially from a shared (but hopefully immutable within a tick) source or a per-world instance.
*   **Potential Inconsistency:**
    *   **Stale Data:** If a region's tick is significantly delayed or if it's not actively ticking (e.g., no players/activity), its view of global states like game rules or a shrinking world border could become stale.
    *   **Consequences:**
        *   **Time:** Most per-region tick logic uses `RegionizedWorldData.getRedstoneGameTime()`, which is advanced within the region. However, if global time (`WorldLevelData.dayTime`) is used for ambient phenomena or some AI triggers, discrepancies could occur in very lagged regions.
        *   **World Border/Game Rules:** Entities or game mechanics in a lagged region might react late to changes.
    *   **Folia's Mitigation:** `RegionizedWorldData.updateTickData()` aims to refresh the region's view of time and potentially other world-level data at the beginning of each regional tick. The global tick in `RegionizedServer` handles advancing primary server time and weather.

### 2. Entity State Post-Teleport/Merge/Split

*   **State Transfer in `teleportAsync`:**
    *   The process involves detaching passengers (`EntityTreeNode`), transforming the entity (new instance for cross-world), removing from the source, adding to `PendingTeleport` in the destination, loading chunks, and then a final scheduled task for placement.
    *   **Key States:** Position, motion, health, active effects, inventory, AI (`Brain`), `entityInactiveTickCounters` (if implemented).
    *   **Potential Inconsistency Windows:**
        *   **Entity Limbo:** Between removal from the source region and full materialization in the destination, the entity is in a conceptual limbo, primarily tracked by the `PendingTeleport` system. Crashes during this window are a risk for entity state loss.
        *   **AI State:** `Brain.stopAll()` is called during dimension changes. The brain's reinitialization in the new context must correctly handle prior state or cleanly restart.
        *   **Time-Sensitive Fields:** If an entity is in `PendingTeleport` for a noticeable duration, its internal timers (cooldowns, effect durations) might become desynchronized if not adjusted upon final placement relative to the destination region's current time.

*   **State Transfer in Region Merge/Split (`RegionizedWorldData.REGION_CALLBACK`):**
    *   `Entity.updateTicks(fromTickOffset, fromRedstoneTimeOffset)` is the primary mechanism for adjusting an entity's internal timers (e.g., `activatedTick`, `lastDamageStamp`, `pistonDeltasGameTime`) to the target region's timeline.
    *   **Potential Inconsistency:** The comprehensiveness and correctness of every entity and block entity's implementation of `updateTicks` (or equivalent for BEs) is critical. If any time-sensitive field is missed, or if the provided offsets don't perfectly map the effective time progression, behaviors could be incorrect post-merge. The current merge logic for `entityInactiveTickCounters` (taking `Math.max`) is a heuristic that prioritizes keeping entities inactive, which is safer than potentially making them active too soon.

### 3. Chunk State vs. Region State

*   **External Chunk Modifications:**
    *   **Issue:** Modifications to chunk data (blocks, block entities) by sources not scheduled through the owning region (e.g., a hypothetical global command directly altering blocks).
    *   **Potential Impact:** A region might tick using a stale view of its chunk data, leading to inconsistent physics, lighting, or AI.
    *   **Folia's Mitigation:** Most game actions (player interactions, entity actions, block updates) should originate within a region and thus be naturally consistent. Operations like `/fill`, `/setblock` are queued via `RegionizedTaskQueue`, ensuring they are executed on the correct region owning the target chunks. This significantly reduces the risk of direct external modification.

*   **Chunk Loading/Saving Synchronization:**
    *   **Issue:** A region attempts to tick a chunk that is not yet fully loaded to the required state, or whose data is being saved concurrently.
    *   **Potential Impact:** Errors, crashes, or operating on partially loaded/saved data.
    *   **Folia's Mitigation:** Regions obtain chunks through `ChunkHolder` instances, which manage chunk statuses (`ChunkStatus.FULL`, etc.). Regions should only operate on chunks that have reached a status indicating they are safe for ticking. The `ChunkTaskScheduler` handles the asynchronous loading and saving, and its interaction with `ChunkHolder` statuses is designed to prevent premature access.

### 4. Distributed Data (Example: `PoiManager`)

*   **POI Data Synchronization:**
    *   `PoiManager` state (POI locations, types, occupancy) is world-scoped but critical for region-specific AI (e.g., villager job seeking).
    *   Modifications (claiming/releasing POIs) are typically done by entities. These actions *must* be scheduled to the region owning the POI's `BlockPos`.
    *   `PoiManager.tick()` calls `villageDistanceTracker.propagateUpdates()` which is synchronized, serializing updates to the distance tracking part of POIs.
    *   **Potential Inconsistency:**
        *   If Villager A (Region 1) claims a POI at `pos` (owned by Region 1), and Villager B (Region 2, adjacent) queries for that same POI at `pos` for pathfinding *before* Region 1's tick (where the claim happens) is fully processed and visible via `PoiManager`, Villager B might see it as available.
        *   The patch shows AI behaviors like `PoiCompetitorScan` and `YieldJobSite` checking `TickThread.isTickThreadFor(level, globalPos.pos())` before modifying POI states via `release` or `setMemory(JOB_SITE, globalPos)`. This is a crucial safeguard.
        *   Reading POI data across regions for pathfinding is complex. It would likely involve either scheduled queries to the owning region or accessing a (potentially slightly stale) cached/replicated view of POI data. Direct cross-thread access to `PoiManager`'s internal section data from a different region would be unsafe.

### 5. Plugin-Managed State

*   **Issue:** Plugins managing their own data related to entities, blocks, or locations that might now reside in different Folia regions.
*   **Potential Impact:** If a plugin, for example, maintains a global `HashMap<UUID, PlayerStats>` and updates it from player events (which can now run on different region threads for different players), it will face race conditions without external synchronization.
*   **Folia's Mitigation/Needs:**
    *   Folia provides `EntityScheduler` and `RegionScheduler` to enable plugins to execute logic in the correct context.
    *   This is primarily a plugin developer responsibility. Folia's role is to provide the tools and clear guidelines.
    *   **Potential Vulnerability:** The ease and safety of these schedulers, and how well they integrate with event handling (especially for events that might cross region boundaries conceptually), will influence plugin stability.

**Key Areas for Ensuring Consistency:**

*   **`REGION_CALLBACK` implementations:** Must be flawless in transferring all aspects of state, especially time-sensitive data.
*   **`teleportAsync` lifecycle:** Robust handling of all stages, including error recovery and ensuring entity state is fully consistent before the entity is "live" in the new region.
*   **Scheduler behavior for re-targeting tasks:** When an entity or location changes its owning region, associated scheduled tasks need to follow or be safely discarded.
*   **POI and similar world-scoped data managers:** Access and modification patterns must be strictly region-aware, likely by scheduling all modifications to the region owning the data's physical location.

State consistency in Folia relies heavily on (a) serializing major structural changes via `regionLock`, (b) ensuring operations are executed on the thread owning the data (via schedulers), and (c) meticulous data transfer and state adjustment logic during transitions (teleports, merges).
