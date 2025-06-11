## Folia: Potential Bug Categories and Areas of Concern

This document categorizes potential bugs and concerns in Folia's architecture, based on prior analyses. It also suggests general testing strategies for each category.

### 1. Race Conditions/Data Corruption

*   **Concern:** Improperly synchronized access to shared data by multiple region threads or other concurrent processes, leading to inconsistent states or data corruption.
*   **Architectural Areas Most Likely Involved:**
    *   **`RegionizedWorldData.REGION_CALLBACK` (Merges/Splits):** Logic within `merge` and `split` for transferring lists (entities, block entities, tick queues) and other data structures. While the callback is under `regionLock`, internal operations on collections must be safe.
    *   **`ThreadedRegionizer` & `getRegionAtUnsynchronised`:** Systems using unsynchronized region lookups might act on stale data if a region is concurrently undergoing modification (merge/split), especially if the region object's internal state isn't fully immutable or copy-on-write during these transitions.
    *   **Plugin API Misuse:** Plugins directly modifying cached NMS data structures or shared plugin data across different region threads without proper synchronization.
    *   **Global Data Structures:** Any server-wide data that isn't inherently thread-safe and might be accessed by regions or global systems concurrently (e.g., `PoiManager` if not all modifications are region-scheduled, parts of `PlayerList`).
*   **Potential Impact:** Server crashes (e.g., `ConcurrentModificationException`), inconsistent game states (item/entity duplication or loss), corrupted world saves.
*   **Testing Strategies:**
    *   **Stress Testing:** High player counts, rapid player movement across potential region boundaries to induce frequent region merges/splits and task scheduling.
    *   **Chaos Engineering:** Injecting delays or (in a test environment) faults into critical sections like `REGION_CALLBACK` or task execution to uncover latent races.
    *   **Data Integrity Audits:** Post-operation (merge, split, teleport) programmatic validation of entity counts, item presence, block states, etc.
    *   **Fuzz Testing:** Sending malformed or unexpected sequences of operations related to region management or cross-region interactions.

### 2. Deadlocks/Livelocks

*   **Concern:** Circular dependencies between threads waiting for resources (locks) or threads getting stuck in a loop of unproductive operations.
*   **Architectural Areas Most Likely Involved:**
    *   **`ThreadedRegionizer` & Callbacks:** If callbacks made while `regionLock` is held attempt to acquire other locks that might be held by threads waiting for `regionLock` (e.g., chunk system locks, plugin locks).
    *   **Schedulers:** Tasks in one region synchronously waiting for results from tasks in another region, creating a circular dependency. Interactions between regional schedulers and the `ChunkTaskScheduler` (e.g., task needs a chunk, chunk load needs a task on that region). The `scheduleChunkTaskEventually` hints at this risk.
    *   **`GlobalRegion` & Regional Schedulers:** Synchronous waits between global tasks and regional tasks.
*   **Potential Impact:** Server freezes (total or specific regions), unresponsiveness.
*   **Testing Strategies:**
    *   **Thread Dump Analysis:** Under sustained heavy and diverse load (teleports, commands, chunk generation, entity activity), periodically capture and analyze thread dumps.
    *   **Lock Order Profiling/Analysis:** If possible, use tools or manual review to ensure a consistent global order for acquiring different system locks.
    *   **Complex Interaction Scenarios:** Design tests that involve many systems interacting simultaneously (e.g., players teleporting into areas undergoing region merges while other plugins are performing world edits via schedulers).

### 3. State Inconsistency (Transient or Persistent)

*   **Concern:** Game state becoming temporarily or permanently incorrect due to timing issues, incomplete operations, or incorrect data propagation between regions or between global and regional contexts.
*   **Architectural Areas Most Likely Involved:**
    *   **Cross-Region Operations (`teleportAsync`, Portals):** Entities might be "lost" or duplicated if a server crashes or a critical error occurs mid-teleport. The `PendingTeleport` list and `RegionShutdownThread` are mitigations for clean shutdowns. Incorrect state transfer (e.g., AI state, active effects) can lead to behavioral bugs.
    *   **Global vs. Regional State Sync:** Regions might use stale global data (game rules, world border) if their tick is delayed, or if propagation isn't immediate.
    *   **`RegionizedWorldData.REGION_CALLBACK` (Merges/Splits):** Incorrect application of `fromTickOffset` or `fromRedstoneTimeOffset` by `Entity.updateTicks` (or equivalents for BlockEntities/LevelTicks) can lead to desynchronized timers, cooldowns, or scheduled events.
    *   **`PoiManager` & Other Distributed Data:** If POI modifications aren't strictly scheduled to the POI's owning region, other regions' AI might use stale POI data.
*   **Potential Impact:** "Ghost" entities, broken AI (villagers not finding POIs), incorrect game mechanics (e.g., redstone, crop growth), visual glitches.
*   **Testing Strategies:**
    *   **Teleportation Barrages:** Numerous entities teleporting frequently between different regions, especially under load.
    *   **Forced Crashes (Test Environment):** Induce crashes at various stages of cross-region operations or region merges/splits to test recovery and data integrity.
    *   **State Verification:** After operations like teleports or region merges, use commands or custom tools to verify entity data, block states, and POI information.
    *   **Long-Term Stability Tests:** Run servers for extended periods with diverse player activity to uncover slow-developing inconsistencies.

### 4. Resource Leaks

*   **Concern:** System resources (memory, threads, file handles, tickets) being allocated but not properly released over time.
*   **Architectural Areas Most Likely Involved:**
    *   **Schedulers:** Failure to cancel all plugin-scheduled tasks (especially repeating ones) when a plugin is disabled, across all scheduler types (`EntityScheduler`, `RegionScheduler`, `GlobalRegionScheduler`, `AsyncScheduler`).
    *   **`PendingTeleport` list:** Entries not being cleared if a teleport sequence fails catastrophically before the final placement task.
    *   **`ThreadedRegionizer`:** `ThreadedRegion` or `RegionizedWorldData` objects not being fully garbage collected after becoming `STATE_DEAD` due to lingering references.
    *   **Chunk Ticketing System:** Custom tickets (e.g., `LOGIN`, `TELEPORT_HOLD_TICKET`, `END_GATEWAY_EXIT_SEARCH`) not being reliably removed after their purpose is served, leading to chunks being unnecessarily force-loaded.
*   **Potential Impact:** Gradually increasing memory usage (leading to OutOfMemoryErrors), degradation of scheduler performance, excessive chunk loading.
*   **Testing Strategies:**
    *   **Profiling:** Use memory profilers during long-duration tests with varied workloads (player activity, plugin load/unload cycles).
    *   **Plugin Enable/Disable Cycling:** Repeatedly load and disable plugins that heavily use schedulers and Folia APIs.
    *   **Heap Dump Analysis:** Examine heap dumps to identify objects that are not being garbage collected.
    *   **Simulate Failed Operations:** Create test scenarios where teleports or other complex operations might fail, then check for cleanup of associated resources/tickets.

### 5. Logic Errors in Complex Operations

*   **Concern:** Fundamental flaws in the algorithms or calculations within core Folia systems.
*   **Architectural Areas Most Likely Involved:**
    *   **`ThreadedRegionizer` Merge/Split Algorithms:** The conditions determining when and how regions are formed, merged, or split (e.g., based on `nonEmptyNeighbours`, `chunkCount`, `minSectionRecalcCount`, `maxDeadRegionPercent`).
    *   **Time-Adjustment Logic (`Entity.updateTicks`, `LevelChunkTicks.offsetTicks`, etc.):** Errors in calculating or applying time offsets during region merges.
    *   **Portal Creation/Search Logic:** The asynchronous logic for finding or creating portals (e.g., `TheEndGatewayBlockEntity.findOrCreateValidTeleportPosRegionThreading`) involves multiple steps and chunk loads.
*   **Potential Impact:** Incorrect game mechanics (e.g., mob spawning, redstone), suboptimal region configurations leading to performance issues, server instability.
*   **Testing Strategies:**
    *   **Focused Code Review:** In-depth review of the specific algorithms in question.
    *   **Unit/Integration Testing:** Create targeted tests for these algorithms with various boundary conditions and inputs.
    *   **Debug Logging & Metrics:** Add extensive logging around these operations to trace their behavior and verify calculations in a test environment.

### 6. API Misuse by Plugins

*   **Concern:** Plugins interacting with Folia's systems in ways that violate its threading model or data ownership rules.
*   **Architectural Areas Most Likely Involved:**
    *   Any Folia API, particularly schedulers, event handlers, and methods for accessing world data or entities.
    *   Incorrect use of `Bukkit.isOwnedByCurrentRegion` (e.g., check-then-act without atomicity).
*   **Potential Impact:** Can trigger any of the above bug categories (races, deadlocks, inconsistencies), often making it hard to distinguish from a Folia core bug.
*   **Testing Strategies:**
    *   **Test Plugins:** Develop a suite of "misbehaving" test plugins that intentionally try common anti-patterns.
    *   **Static Analysis (if feasible):** Explore tools that might detect obviously unsafe concurrency patterns in plugin bytecode.
    *   **Clear Developer Documentation & Best Practices:** Essential for guiding plugin developers.
    *   **Robust API-Level Thread Checks:** Implement aggressive `TickThread.ensureTickThread` checks within NMS/CraftBukkit methods that are sensitive to regional context.

### 7. Chunk System Integration Issues

*   **Concern:** Problems arising from the interaction between the regionalized ticking system and the underlying chunk loading/saving mechanisms.
*   **Architectural Areas Most Likely Involved:**
    *   **`ChunkHolderManager`, `ChunkTaskScheduler`:** Handling concurrent chunk requests from multiple regions, managing chunk statuses, and ensuring ticket levels are correctly propagated and processed.
    *   **Region Tick Loop:** Ensuring it only operates on chunks that are fully loaded and in a consistent state.
*   **Potential Impact:** Chunk data corruption, server crashes (e.g., trying to tick an unloaded chunk), load deadlocks, "ghost chunks," entities falling through the world.
*   **Testing Strategies:**
    *   **High Chunk Churn Scenarios:** Players rapidly exploring new terrain, using features that load many chunks (e.g., dynmap-like plugins if they were adapted), or large-scale world edits.
    *   **Concurrent Chunk-Reliant Operations:** Triggering teleports, portal travel, and structure generation in multiple regions simultaneously.
    *   **Forced Unload/Load Tests:** Using debug commands or specific test setups to attempt to force chunks to load or unload while regions are actively using them.

Addressing these categories requires diligent design, careful implementation with appropriate synchronization, comprehensive error handling, and a multi-faceted testing approach.
