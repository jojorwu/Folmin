Folia is a fork of PaperMC, a popular high-performance Minecraft server software. Folia's primary innovation is the introduction of **regionised multithreading**.

This core feature fundamentally changes how the server handles game logic. Instead of a single main thread processing all game events, Folia divides the game world into "independent regions." Each region groups nearby loaded chunks and has its own tick loop, running in parallel with other regions on a configurable thread pool. This effectively gives each region its own "main thread."

This architectural shift allows Folia to achieve significantly improved scalability, especially for servers with many spread-out players. Server types like Skyblock or Survival Multiplayer (SMP) that naturally distribute players across the world can benefit greatly. Folia is best suited for deployment on hardware with a high number of CPU cores (ideally 16 or more).

It's important to note that due to this fundamental change in threading, Folia is not a drop-in replacement for PaperMC, and existing plugins will require modifications to be compatible. Folia is its own distinct project and is not planned to be merged into PaperMC.

## Key Architectural Changes in Folia

Folia's regionised multithreading introduces several fundamental architectural changes compared to traditional Minecraft server software:

### Independent Regions and Parallel Ticking
The core concept in Folia is the **Independent Region**. The server dynamically groups nearby loaded chunks into these regions. Each region possesses its own independent game tick loop, which processes game logic (entity updates, block ticks, etc.) for the chunks it contains. These regional tick loops run in parallel on a configurable thread pool, allowing different parts of the game world to be updated simultaneously. This is a stark contrast to the traditional single-threaded approach where all game logic is processed sequentially. Regions are designed to be isolated; they do not directly share data during ticking and have rules to prevent simultaneous ticking of adjacent regions to maintain independence.

### Removal of the Main Thread
A direct consequence of regionised multithreading is the **elimination of the server's main thread**. In traditional Minecraft servers (including PaperMC), a single main thread is responsible for the vast majority of game logic execution. In Folia, this responsibility is distributed among the threads ticking the independent regions. Each region effectively has its own "main thread" for its own set of chunks and entities. This has profound implications for plugin compatibility, as any plugin code assuming a single, global main thread will need significant rewriting. Operations that were previously synchronized by the main thread now require new mechanisms for thread safety.

### New Scheduling APIs
To manage tasks in this new multi-threaded environment, Folia deprecates the standard BukkitScheduler and introduces a suite of new scheduling APIs:
-   **`RegionScheduler`**: Allows tasks to be scheduled for execution on the next tick of the region that owns a specific location. This is crucial for operations that need to interact with world data within a particular region.
-   **`EntityScheduler`**: Enables tasks to be scheduled to run on the region that currently owns a specific entity. This is useful for operations that need to be synchronized with an entity's updates.
-   **`AsyncScheduler`**: Provides a mechanism for scheduling asynchronous tasks, similar to the old BukkitScheduler's async tasks, but adapted for Folia's architecture. This is suitable for tasks that do not need to run on a specific region's tick.
-   **`GlobalRegionScheduler`**: Schedules tasks to be executed on the `GlobalRegion`.

These new schedulers are essential for plugin developers to ensure their code executes in the correct context and maintains thread safety.

### The GlobalRegion
While most game logic is regionalised, some data and tasks are inherently global. For these, Folia introduces the **`GlobalRegion`**. This is a special, unique region that is always scheduled to run at 20 TPS. It is responsible for managing data and tasks not tied to any specific in-game region, such as:
-   Global game rules
-   Global game time and daylight time (which are then copied to regions at the start of their ticks)
-   Console command handling
-   World border management
-   Server-wide weather

The `GlobalRegion` ensures that operations requiring a single, unified context can still function correctly within the regionised multithreading model. It does not own any region-specific data like chunks or entities.

## Plugin Compatibility Considerations

The shift to regionised multithreading in Folia has significant implications for plugin compatibility:

-   **Widespread Incompatibility:** Due to the removal of the main server thread and the introduction of parallel region ticking, **most existing Bukkit, Spigot, or PaperMC plugins are expected to be incompatible with Folia without substantial modifications.** Plugin authors should assume their plugins will not work out-of-the-box. Issues can range from benign errors to critical data corruption due to thread-safety problems.

-   **Explicit Support Required (`folia-supported: true`):** To prevent server administrators from unknowingly running incompatible plugins, Folia will only load plugins that explicitly declare their compatibility. Plugin developers must add the line `folia-supported: true` to their `plugin.yml` file. This acts as an acknowledgment from the developer that the plugin has been updated and tested for Folia's multi-threaded environment.

-   **Stricter Threading Model & Data Handling:** Folia enforces a much stricter threading model. Regions tick in parallel, and their data is generally not shared. Plugin developers must be acutely aware of thread safety:
    -   Accessing or modifying data in one region from code running in another region is unsafe and can lead to data corruption.
    -   Plugins must utilize the new scheduling APIs (e.g., `RegionScheduler`, `EntityScheduler`) to ensure code executes on the correct thread context (i.e., the thread that "owns" the relevant data like a location or entity).
    -   Careful consideration is needed for any plugin-managed data. Concurrent collections might hide some issues but are not a universal solution and can lead to hard-to-debug race conditions if used improperly. Folia plans to introduce more aggressive thread checks to help identify incorrect data access.

Developers need to diligently review and update their plugins, paying close attention to event handling, task scheduling, and any shared data access to ensure stability and correctness on Folia.

## API Changes and Additions

Folia introduces several breaking changes to the existing Bukkit/Paper API and adds new APIs to support its regionised architecture.

### Broken API (Examples)
The fundamental shift in threading means many existing APIs are no longer safe or valid in Folia. Plugin developers should be aware that:
-   **`Entity#teleport` is permanently broken and removed.** Plugins MUST use the asynchronous alternatives like `Entity#teleportAsync` and then use the appropriate scheduler to handle the result.
-   **All Scoreboard API is considered broken.** Scoreboards often represent global state, and a new approach for handling them in a multi-threaded environment is still being determined.
-   APIs related to **portals, respawning players, and some player login mechanisms** are largely broken or behave differently.
-   **World loading/unloading APIs** are not currently functional.

This list is not exhaustive, and developers should assume that any API that implies global state or relies on a single main thread might be affected.

### New API Additions (Examples)
Folia introduces new APIs to help plugins work correctly within its threading model:
-   **`Bukkit#isOwnedByCurrentRegion(Location/Entity)`**: This crucial method allows plugins to check if a given location or entity is owned by the region whose tick is currently executing the code. This helps in making decisions about accessing world data safely.
-   **New Schedulers**: As mentioned previously, `RegionScheduler`, `EntityScheduler`, `AsyncScheduler`, and `GlobalRegionScheduler` are key additions for managing task execution in the correct thread context.

Folia also plans to add more thread-check APIs to help developers ensure their code is running in the appropriate context and to make incorrect accesses fail hard, aiding in debugging. Proper asynchronous event handling is also a planned API addition.

## Performance and Scalability

Folia's core architectural change, regionised multithreading, is specifically designed to enhance server performance and scalability under certain conditions. The primary goal is to leverage multi-core processors more effectively.

-   **Target Conditions:** Folia is expected to perform best on servers with a high number of CPU cores (the documentation recommends at least 16 cores, not just threads) and a player base that is naturally spread out across the game world (e.g., Skyblock, Survival Multiplayer). This allows Folia to create multiple independent regions that can genuinely run their tick loops in parallel.

-   **Regional TPS:** Each independent region in Folia runs its own tick loop and aims to maintain the standard Minecraft tick rate of 20 TPS. As long as the server's thread pool is not maximally utilized, the scheduling algorithm is designed to ensure that regions taking 50ms or less per tick will achieve this target.

-   **Scalability Approach:** By distributing the game logic processing across multiple threads, Folia theoretically allows the server to support more players or more demanding game worlds without the single-thread bottleneck that can limit traditional Minecraft server software. The more independent regions that can be processed in parallel, the better the server should scale.

-   **Benchmark Data & Anecdotal Evidence:** The official documentation does not provide specific quantitative benchmarks comparing Folia to PaperMC or vanilla Minecraft. However, it does mention that configuration advice was derived from experiences on a test server that peaked at around 330 players. While not a formal benchmark, this indicates the scale of player counts Folia is intended to handle. The focus is on the architectural capacity for scaling rather than on specific performance figures at this stage.

It is crucial to configure Folia appropriately, considering thread allocation for regions, I/O, and other tasks, to achieve optimal performance on a given hardware setup.

## Conclusion

Folia represents a significant step towards highly scalable Minecraft servers. By fundamentally redesigning the server's core processing around regionised multithreading, it aims to deliver substantial performance improvements for specific server types, particularly those with large, spread-out player bases and powerful multi-core hardware. While this necessitates considerable adaptation from plugin developers, Folia's innovative approach holds the promise of pushing the boundaries of Minecraft server performance.
