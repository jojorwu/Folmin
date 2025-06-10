Foldenor is a server software fork based on Folia, which itself is a fork of PaperMC. It was originally developed for the Edenor Minecraft server.

Key goals and features of Foldenor include:
-   **Secure Seed:** Implementation of a secure seed mechanism.
-   **Useful Patches:** Incorporation of various patches, blending contributions from several other server software projects.
-   **Enhanced Plugin Compatibility:** Aims for broader Bukkit/Paper plugin support compared to standard Folia. It allows plugins to be loaded by adding `folia-supported: true` to their `plugin.yml` file. However, it includes a critical warning: "Not all plugins can run on Foldenor without changing the plugin code!" This implies that while the flag enables loading, full functionality without modification is not guaranteed due to Folia's underlying multithreaded architecture.

Foldenor credits several other projects for patches it incorporates, including Purpur, Leaf, Pufferfish, Gale, ShreddedPaper, Leaves, Kaiiju, SparklyPaper, Mizi, and Canvas.
