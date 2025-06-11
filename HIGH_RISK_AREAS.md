## High-Risk Areas in Folia's Architecture

This document outlines key components and operations within Folia's regionalized multithreading architecture that are considered high-risk due to their complexity, the potential impact of errors, and the challenges of ensuring correctness in a concurrent environment.

1.  **`ThreadedRegionizer`**:
    *   **Concern**: Complex state management for region lifecycle (creation, merging, splitting) and boundary enforcement.
    *   **Risk Explanation**: The `ThreadedRegionizer` is responsible for dynamically creating, merging, and splitting regions based on player distribution and chunk activity. Errors in this state transition logic (e.g., during region merge when a region is released by `TickRegionScheduler` and `onRegionRelease` is called) or in maintaining correct region boundaries and adjacency rules could lead to inconsistent world states, lost game data (entities, block updates), deadlocks, or game areas not being ticked.

2.  **`RegionizedWorldData` and its `REGION_CALLBACK`**:
    *   **Concern**: Ensuring correct and consistent data transfer and updates during region merge/split operations.
    *   **Risk Explanation**: `RegionizedWorldData` stores all per-region game state (entity lists, block entity tickers, scheduled ticks, chunk lists, etc.). The `REGION_CALLBACK` (`merge` and `split` methods) dictates how this data is transferred or combined when regions are modified. Off-by-one errors, missed data items, incorrect offsetting of time-sensitive data (like scheduled ticks or entity activation timers), or race conditions within the callback logic could lead to significant issues like entity duplication/disappearance, corrupted block states, or broken game mechanics.

3.  **`TickRegionScheduler` and Schedulers (`EntityScheduler`, `RegionScheduler`, `GlobalRegionScheduler`, `AsyncScheduler`)**:
    *   **Concern**: Correct task execution context, lifecycle management, and potential for deadlocks or resource starvation.
    *   **Risk Explanation**: Folia's schedulers are crucial for ensuring tasks run on the appropriate region's thread. Errors in scheduling logic (e.g., a task for an entity in region A being mistakenly run on region B's thread) can lead to severe data corruption. Proper handling of task scheduling, execution, and cancellation is vital. Complex inter-task dependencies, especially across different regions or between a region and the `GlobalRegion`, could introduce deadlocks. An imbalance in task load or scheduling priority might also lead to certain regions becoming starved of processing time.

4.  **Cross-Region Operations (`Entity.teleportAsync`, portal logic)**:
    *   **Concern**: Atomicity, data consistency, and robust error handling during the movement of entities (including players) between regions.
    *   **Risk Explanation**: Transferring an entity from one region's control to another involves multiple steps: detaching from the source region, serializing/preparing state, adding to the destination region, initializing state, and updating player trackers. This process must be effectively atomic. Failures at any stage (e.g., due to errors, chunk loading issues at the destination, or concurrent region modifications) could result in duplicated entities, lost entities, or entities in an inconsistent state. Portal logic adds further complexity by potentially involving finding/creating portal structures in the target region.

5.  **`GlobalRegion` Interaction**:
    *   **Concern**: Synchronization of server-wide data with regional data and potential for bottlenecks.
    *   **Risk Explanation**: The `GlobalRegion` (handled by `RegionizedServer` and its `GlobalTickTickHandle`) manages global concerns like parts of the player list, some commands, and global game state (e.g., server-wide time, weather announcement coordination). Data that needs to be passed to or synchronized with individual ticking regions (e.g., player connection state during login/logout, some game rules) requires careful, thread-safe mechanisms. If too many regional tasks defer work to the `GlobalRegion`, it could become a performance bottleneck.

6.  **Plugin API Interactions**:
    *   **Concern**: Potential for plugins to misuse new threading-aware APIs, leading to instability or data corruption.
    *   **Risk Explanation**: Folia introduces a new threading model. While APIs like `Bukkit#isOwnedByCurrentRegion` exist to help plugins, there's a risk that plugins (even those marked `folia-supported: true`) might incorrectly access or modify game data across regions without using the appropriate schedulers. This can bypass Folia's safety mechanisms and lead to hard-to-diagnose concurrency bugs. The API surface itself must be designed to minimize such risks.

7.  **Chunk Loading/Unloading in a Regionalized Context**:
    *   **Concern**: Ensuring chunks are loaded before a region ticks them and are correctly unloaded when no longer needed, especially with concurrent demands from multiple regions.
    *   **Risk Explanation**: Each region independently determines its chunk requirements. The chunk loading system (involving `ChunkHolderManager` and `ChunkTaskScheduler`) must handle requests from multiple regions, potentially for overlapping chunk sets. Race conditions or errors in logic could lead to a region attempting to tick an unloaded chunk (causing errors or crashes) or to chunks remaining loaded unnecessarily (chunk leaks). The management of chunk tickets (e.g., player tickets, portal tickets, force-loaded chunk tickets) in coordination with region-based demands is complex.
