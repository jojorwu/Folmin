## Folia Error Handling and Edge Case Analysis

This document analyzes potential error handling scenarios and edge cases within Folia's architecture, focusing on previously identified high-risk areas.

### 1. Error Handling in Regional Ticks & Scheduled Tasks

*   **Entity/BlockEntity Tick Exceptions:**
    *   **Issue:** An exception occurs during an entity or block entity's tick method within a specific region.
    *   **Potential Impact:**
        *   *Same Region:* If not handled per-entity/BE, it could halt further processing in that region for the current tick. Standard server behavior (remove offending element) is expected.
        *   *Other Regions:* Should be isolated. However, if an unhandled exception crashes a `TickThreadRunner` thread, it could impact the `TickRegionScheduler`'s ability to manage other regions or tasks assigned to that thread.
    *   **Folia's Approach/Needs:**
        *   The game loop for entities/BEs within `ServerLevel.tick()` must have try-catch blocks for individual ticks to prevent one failure from stopping others in the same region.
        *   `TickThreadRunner` has an `uncaughtExceptionHandler` that calls `scheduler.regionFailed()`, leading to a server shutdown. This is a last resort.
        *   Robust logging with region context is essential for diagnosing such errors.

*   **Scheduled Task Exceptions:**
    *   **Issue:** An exception occurs within a task executed by `EntityScheduler`, `RegionScheduler`, `GlobalRegionScheduler`, or `AsyncScheduler`.
    *   **Potential Impact:** Could propagate to the scheduler thread, potentially crashing it (and thus the region or server via `uncaughtExceptionHandler`).
    *   **Folia's Approach/Needs:**
        *   All schedulers must wrap task execution in try-catch blocks. The `FoliaRegionScheduler` (via `wrap` method) and presumably others should log exceptions to the relevant plugin's logger and prevent the exception from killing the worker thread.
        *   A single failing task should not disrupt the scheduler or other tasks in the same/other regions.

### 2. Edge Cases for Region Management (`ThreadedRegionizer`)

*   **Rapid Fluctuations in Chunk Loading/Player Movement:**
    *   **Issue:** Frequent and rapid addition/removal of chunks or player movement across boundaries, leading to continuous attempts to create, merge, or split regions.
    *   **Potential Impact:** "Thrashing" on the `regionLock`, high CPU usage for region management, potentially delaying actual game ticks. While `regionLock` serializes changes, the overhead of frequent, complex operations could be substantial.
    *   **Folia's Approach/Needs:**
        *   The `regionLock` prevents data races during modifications.
        *   Configurable parameters like `minSectionRecalcCount` and `maxDeadRegionPercent` offer some level of hysteresis against over-aggressive splitting/merging.
        *   Further debouncing or cooldown mechanisms for region operations might be needed if thrashing is observed in practice.

*   **Very Large or Very Small Regions:**
    *   **Issue:** A single region covering (almost) the entire world, or many tiny, isolated regions.
    *   **Potential Impact:**
        *   *Large Region:* Performance degrades towards a single-threaded model for that world.
        *   *Many Small Regions:* Increased overhead in region management, task scheduling, and potentially inter-region communication if it becomes more common.
    *   **Folia's Approach/Needs:**
        *   The system is designed to accommodate these scenarios. The `regionSectionMergeRadius` helps consolidate overly fragmented regions. Performance scaling of `ThreadedRegionizer` data structures (`sections`, `regionsById`) is important.

*   **Server Start/Shutdown:**
    *   **Issue:** Correct initialization and teardown of regions.
    *   **Potential Impact:**
        *   *Startup:* Regions around spawn must be correctly formed and active before players join.
        *   *Shutdown:* All regional data must be saved, tasks completed or cancelled, and cross-region operations finalized.
    *   **Folia's Approach/Needs:**
        *   `regioniser.addChunk` is called during level creation to form initial regions.
        *   `RegionShutdownThread` is designed for graceful shutdown: it halts the scheduler, then iterates regions for saving chunks, finishing pending teleports, and closing player inventories. This serialized, controlled shutdown is crucial.

### 3. Edge Cases for Cross-Region Operations (`teleportAsync`)

*   **Teleport Target Becomes Invalid:**
    *   **Issue:** Destination of a teleport (chunk, region) becomes invalid (e.g., unloaded, region destroyed) after initiation but before completion.
    *   **Potential Impact:** Entity loss, teleport failure, or entity stuck in `PendingTeleport`.
    *   **Folia's Approach/Needs:**
        *   `TELEPORT_FLAG_LOAD_CHUNK` helps ensure destination chunks are loaded.
        *   The `PendingTeleport` mechanism and `RegionShutdownThread.finishTeleportations` provide some robustness for clean shutdowns.
        *   The final task placing the entity (scheduled by `placeInAsync`) needs robust error handling: if the target is definitively invalid, the entity should be moved to a safe fallback (e.g., world spawn) rather than being lost.

*   **Simultaneous Conflicting Teleports:**
    *   **Issue:** Multiple entities teleporting to the exact same coordinates, or one entity being the target of several concurrent teleports.
    *   **Potential Impact:** Temporary co-occupation (resolved by physics), potential race conditions on the target entity's state if it's the subject of multiple teleports.
    *   **Folia's Approach/Needs:**
        *   Standard entity collision handles multiple entities in one spot post-teleport.
        *   An entity's own task scheduler should serialize operations *on that entity*, including being the target of a teleport.

*   **Plugin Interference:**
    *   **Issue:** Plugins cancelling teleport events or modifying entity state mid-teleport.
    *   **Potential Impact:** Broken teleports, inconsistent entity state.
    *   **Folia's Approach/Needs:**
        *   Events like `PlayerTeleportEvent` must be fired at correct junctures. `teleportAsync` should respect cancellation.
        *   Ideally, entity state relevant to the teleport is captured before plugin event calls that could modify it, or modifications are part of the event's outcome.

### 4. Edge Cases for Data Consistency (`REGION_CALLBACK`)

*   **Entity Removal During Merge/Split:**
    *   **Issue:** An entity is removed (e.g., by command, despawn) from a region while that region's data is being merged or split.
    *   **Potential Impact:** `ConcurrentModificationException` if list iteration is not safe, or processing of an already-removed entity.
    *   **Folia's Approach/Needs:**
        *   The `regionLock` (write lock) held by `ThreadedRegionizer` during merge/split operations should prevent the involved regions from ticking concurrently. This means the entity lists within `RegionizedWorldData` should not be modified by normal ticking operations while the `REGION_CALLBACK` is processing them.

*   **Prolonged Region Inactivity Before Merge:**
    *   **Issue:** A region remains unticked for an extended period then gets merged.
    *   **Potential Impact:** Time-sensitive data (cooldowns, AI timers, crop growth stages) will be severely desynchronized.
    *   **Folia's Approach/Needs:**
        *   The `fromTickOffset` and `fromRedstoneTimeOffset` in `REGION_CALLBACK.merge` are critical. `Entity.updateTicks`, `BlockEntity.updateTicks`, and `LevelChunkTicks.offsetTicks` must be used comprehensively to adjust all such values to the target region's timeline. For very long periods of inactivity, simply adding an offset might not perfectly simulate continuous ticking (e.g., accumulated "inactive ticks" for EAR might need special handling or reset).

### 5. Plugin API Misuse Scenarios

*   **Incorrect Caching & Cross-Thread Access:**
    *   **Issue:** Plugin caches an entity from Region A, then attempts to use it directly from Region B's thread.
    *   **Potential Impact:** Data corruption, CME, server crash.
    *   **Folia's Approach/Needs:**
        *   Strict enforcement via `TickThread.ensureTickThread` in NMS methods.
        *   Clear plugin developer documentation and guidance.

*   **Task Running in Wrong Context After Entity Teleport:**
    *   **Issue:** A task is scheduled for an entity; the entity then teleports to a new region before the task executes.
    *   **Potential Impact:** Task executes on the old region's thread, accessing a (potentially removed) entity or incorrect world state.
    *   **Folia's Approach/Needs:**
        *   `FoliaEntityScheduler` (and similar) must re-validate entity existence and *current* region ownership before task execution. Tasks for entities that have moved regions should ideally be migrated to the new region's scheduler or safely discarded. The `Entity.getBukkitEntity().taskScheduler.schedule()` already handles retired (removed) entities.

This analysis highlights that while Folia's design incorporates many safeguards (locks, schedulers, specific callback mechanisms), the complexity of regionalized concurrency means that robust error handling within individual operations and meticulous attention to edge cases (especially around state transitions like teleports and region merges) are critical for server stability.
