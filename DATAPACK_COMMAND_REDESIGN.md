# Folia-Compatible DataPackCommand Redesign Proposal

## I. Goal

To redesign the `DataPackCommand` to be compatible with Folia's regionised multithreading architecture, ensuring that datapack reloads are coordinated across all server regions and the GlobalRegion, maintaining data consistency and server stability.

## II. Core Design: Coordinated Server-Wide Reload via GlobalRegion

The core idea is to make the GlobalRegion responsible for coordinating the datapack reload across all ticking regions. This involves a multi-phase approach:

### Phase 1: Preparation & Quiescence Command (GlobalRegion -> All Regions)

1.  **Initiation:** The `/datapack reload` command is initially handled by the region where the command executor (e.g., a player) resides. This region's `DataPackCommand` forwards the request to the `GlobalRegionScheduler` (or a similar new global mechanism).
2.  **Global Coordination Start:** The GlobalRegion receives the reload request. It then broadcasts a "prepare for datapack reload" command/event to all active `TickRegionScheduler.RegionScheduleHandle` instances.
3.  **Region Quiescence:**
    *   Upon receiving this command, each `RegionScheduleHandle` will:
        *   Complete the current tick if it's in progress.
        *   **Block new player actions that might be affected by datapack changes** (e.g., new crafting, command execution related to recipes/advancements, loot table interactions). This could be achieved by setting a flag in the `RegionizedWorldData` or a similar per-region state.
        *   **Pause non-essential ticking activities** that rely heavily on datapack content (e.g., custom recipe checks, advancement triggers, loot table lookups in mob deaths if feasible without major gameplay disruption). Essential ticking like entity movement and basic block ticks should continue if possible.
        *   Drain its current task queue of any tasks that might conflict with a datapack reload.
        *   Signal back to the GlobalRegion that it is "ready for reload."

### Phase 2: Datapack Reload (GlobalRegion)

1.  **Global Quiescence Check:** The GlobalRegion waits until all active regions have signaled "ready for reload."
2.  **Global Datapack Reload:**
    *   The GlobalRegion then performs the actual datapack reload logic, similar to what `MinecraftServer#reloadResources` does. This includes:
        *   Loading new/changed datapacks.
        *   Updating registries, recipes, loot tables, advancements, functions, tags, etc., that are stored globally or managed by the `MinecraftServer` instance.
    *   This step must ensure that any data structures accessed by regions are updated atomically or in a way that regions will pick up the new data upon resuming.
3.  **Error Handling:** If the reload fails at the global level, the GlobalRegion informs all regions to abort the reload and resume normal operations. The failure is reported to the command issuer.

### Phase 3: Resumption & Notification (GlobalRegion -> All Regions)

1.  **Reload Completion Signal:** Once the GlobalRegion successfully completes the reload, it broadcasts a "datapack reload complete" command/event to all `RegionScheduleHandle` instances.
2.  **Region Resumption:**
    *   Each `RegionScheduleHandle` receives this signal and:
        *   Instructs its region(s) to pick up the new datapack configurations. This might involve:
            *   Reloading caches or pointers to datapack-dependent resources.
            *   Notifying relevant regional systems (e.g., per-region recipe managers if such exist, though unlikely for vanilla resources).
        *   Clears the "block player actions" and "pause ticking" flags.
        *   Resumes normal task processing.
3.  **Command Feedback:** The GlobalRegion, once all regions acknowledge resumption (or after a timeout), informs the original command issuer (via the initial region) that the reload was successful.

## III. Conceptual Code Modifications

### 1. `DataPackCommand.java` (in the region context)

*   Modify `reload` (and potentially other subcommands like `enable`, `disable`):
    *   Instead of directly calling `MinecraftServer#reloadResources` or similar, it will now package the request and send it to a new method in `MinecraftServer` or a dedicated global service (e.g., `GlobalDatapackReloader`).
    *   It might need to register a callback or use a `CompletableFuture` to receive the final success/failure status from the GlobalRegion to send back to the command sender.

### 2. `MinecraftServer.java` (or a new `GlobalDatapackReloader` service)

*   **New Method for Initiating Reload (e.g., `coordinateDatapackReload`)**:
    *   Called by `DataPackCommand` from any region.
    *   Sets a server-wide "datapack reload pending" state.
    *   Sends a message/task to `RegionizedServer` (or `TickRegions.getScheduler()`) to propagate the "prepare for reload" signal to all `RegionScheduleHandle` instances.
    *   Manages the list of regions that have acknowledged readiness.
*   **Method to Perform Actual Reload**:
    *   This will contain the core logic from the current `MinecraftServer#reloadResources` (or `MinecraftServer#loadWorldDataPacks`, `ReloadableServerResources#loadResources`, etc.).
    *   Must be callable only when all regions are quiescent.
*   **Method to Signal Reload Completion/Abortion**:
    *   Sends a message/task to `RegionizedServer` to propagate the "reload complete" or "reload aborted" signal.

### 3. `TickRegionScheduler.RegionScheduleHandle` (or `TickRegions.TickRegionData`)

*   **New State/Flags**:
    *   `boolean preparingForDatapackReload`
    *   `boolean datapackReloadAborted`
*   **Modified `tickRegion` / `runRegionTasks`**:
    *   Check the `preparingForDatapackReload` flag. If true:
        *   Prioritize essential ticks.
        *   Delay or skip tasks/ticks that are sensitive to datapack changes.
        *   Once the region is "stable" (e.g., critical tasks done, queue drained of conflicting tasks), it calls back to the `MinecraftServer`/`GlobalDatapackReloader` to acknowledge readiness.
    *   After global reload, if not aborted, it needs to refresh its view of datapack-dependent resources. This might be passive (just using new global instances) or active (clearing specific caches).
*   **New Method to Handle "Prepare for Reload" Signal**:
    *   Sets `preparingForDatapackReload = true`.
    *   Initiates quiescence logic.
*   **New Method to Handle "Reload Complete/Aborted" Signal**:
    *   If complete: `preparingForDatapackReload = false`, triggers any necessary regional refresh, resumes normal operations.
    *   If aborted: `preparingForDatapackReload = false`, `datapackReloadAborted = true` (to handle any cleanup/logging), resumes normal operations.

## IV. Why this Approach?

*   **Centralized Control:** The GlobalRegion is the natural place to manage server-wide state and resources like datapacks.
*   **Ordered Operation:** Ensures that datapacks are not reloaded while regions are actively using them in potentially inconsistent ways.
*   **Minimizes Disruption (Relatively):** While there will be a brief period of reduced activity in regions, it avoids a full server stop or the chaos of regions reloading datapacks independently and at different times.
*   **Leverages Existing Global Tick:** The GlobalRegion already has a ticking mechanism that can be used for coordination.

## V. Trade-offs

*   **Increased Complexity:** This is significantly more complex than the single-threaded model.
*   **Temporary Performance Impact:** Regions will experience a brief slowdown or partial pause during the "quiescence" phase. The duration depends on how quickly regions can become "ready."
*   **Potential for Deadlocks/Hangs:** Careful design is needed to avoid situations where a region never signals readiness or the GlobalRegion gets stuck waiting. Timeouts and clear state management are crucial.
*   **Defining "Quiescence":** Determining exactly which tasks/ticks to pause in regions without breaking gameplay is challenging. It might require iterative refinement. Some plugins might behave unexpectedly if their expected ticks are heavily delayed.

## VI. Key Testing Points

*   **Reload Success and Data Consistency:** Verify that all datapack resources (recipes, loot tables, functions, advancements, tags) are correctly updated and accessible in all regions post-reload.
*   **Concurrent Player Actions:** Test what happens if players attempt actions (crafting, commands, triggering advancements) during the quiescence and reload phases. Expected: actions are gracefully delayed or temporarily denied.
*   **Region Stability:** Ensure regions remain stable and do not crash or deadlock during the process.
*   **Performance During Quiescence:** Measure the impact on region tick times and overall server responsiveness.
*   **Error Handling:** Test scenarios where the global datapack reload fails (e.g., due to a malformed datapack). Regions should correctly abort and resume.
*   **Plugin Compatibility (Basic):** While full plugin compatibility is a larger Folia concern, basic tests with plugins that use common datapack features should be done.
*   **`/datapack enable/disable` commands:** Ensure these also follow a similar coordinated approach if they modify the live datapack set.

This design prioritizes safety and consistency for a critical server-wide operation in a heavily parallelized environment.
