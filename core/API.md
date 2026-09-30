# Vystorm Core 0.24.0 – Plugin API Reference

A self-contained reference for building Paper/Purpur plugins on top of **Vystorm Core** – written for humans and for AI
coding assistants ("vibe coding"). With this file and the Core jar on the compile classpath you should be able to write
a working plugin without reading Core's source.

- Core version documented here: **0.24.0** (`0.24.0-Vystorm-Core-26.2`), Minecraft/Paper **26.2**, Java **25**.
- Every class, method and limit below was checked against the 0.24.0 jar and source. If something is not listed here,
  treat it as internal.
- Package root: `Vystorm.vystorm_core` (capital `V`). Some legacy names contain typos on purpose – they are public API
  and never change: `GuiInteface`, `singeltons`, `AdminComand`, `ressource()`, `luckperms()`, main class `main`.
- In signatures below, `Player`, `Plugin`, `Component`, `JsonElement` etc. are the usual Bukkit/Paper/Adventure/Gson types.

## Contents

1. [Setup](#1-setup)
2. [Build a plugin in 10 minutes](#2-build-a-plugin-in-10-minutes)
3. [Ground rules: threads, lifecycle, errors](#3-ground-rules-threads-lifecycle-errors)
4. [Localization (`api.lang()`)](#4-localization-apilang)
5. [Settings (`api.settings()`)](#5-settings-apisettings)
6. [Menus and managed inventories (`api.gui()`)](#6-menus-and-managed-inventories-apigui)
7. [Native dialogs (`api.dialogs()`)](#7-native-dialogs-apidialogs)
8. [Chat actions and menu buttons](#8-chat-actions-and-menu-buttons)
9. [Tutorials](#9-tutorials)
10. [Admin dashboard modules (`api.admin()`)](#10-admin-dashboard-modules-apiadmin)
11. [Shared database](#11-shared-database)
12. [Player data, XP, preferences, file storage](#12-player-data-xp-preferences-file-storage)
13. [Debug messages](#13-debug-messages)
14. [Web platform](#14-web-platform)
15. [Client mod: overlay and native HUD elements](#15-client-mod-overlay-and-native-hud-elements)
16. [Items, regions, curves, NBT and legacy handlers](#16-items-regions-curves-nbt-and-legacy-handlers)
17. [Cross-plugin services](#17-cross-plugin-services)
18. [Not public API](#18-not-public-api)
19. [Rules for AI assistants](#19-rules-for-ai-assistants)
20. [Versioning and compatibility](#20-versioning-and-compatibility)
21. [Appendix: Core commands and permissions](#21-appendix-core-commands-and-permissions)

---

## 1. Setup

### 1.1 Gradle (compile against the local Core jar)

Core is not published to a Maven repository. Put the jar next to your project (e.g. `libs/Vystorm_Core.jar`) and use
it **compileOnly** – never shade it; the server loads Core as its own plugin.

```kotlin
// build.gradle.kts
plugins { java }

group = "com.example"
version = "1.0.0"

// -PvystormCoreJar=/path/to/VystormCore-0.24.0.jar overrides the default location
val coreJar = file(providers.gradleProperty("vystormCoreJar").getOrElse("libs/Vystorm_Core.jar"))

repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    compileOnly("io.papermc.paper:paper-api:26.2.build.121-stable")
    compileOnly(files(coreJar))
    testImplementation(files(coreJar))            // optional, for unit tests (e.g. Lang.create)
}

java { toolchain.languageVersion = JavaLanguageVersion.of(25) }

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.release = 25
}

tasks.processResources {
    val v = project.version.toString()
    inputs.property("version", v)
    filesMatching("plugin.yml") { expand("version" to v) }
}

tasks.compileJava {
    doFirst { require(coreJar.isFile) { "Vystorm Core jar missing: -PvystormCoreJar=/path/to/core.jar" } }
}
```

Notes:
- Gson (`com.google.gson`) and Adventure/MiniMessage come with Paper; you do not need extra dependencies for the web or
  lang APIs.
- Only the legacy `api.database().getDataSource(id)` returns a `HikariDataSource`; if you call it, add
  `compileOnly("com.zaxxer:HikariCP:5.1.0")`. The shared-database API (`shared()`, `sharedReady()`) only uses
  `javax.sql.DataSource`.

### 1.2 `plugin.yml`

```yaml
name: MyPlugin
version: '${version}'
main: com.example.myplugin.MyPlugin
api-version: '26.2'
depend: [vystorm_core]        # or softdepend if Core is optional for you
```

Core's plugin name is **`vystorm_core`** and it loads with `load: STARTUP`, so with `depend` it is always enabled before
your `onEnable`. Use `softdepend` only if your plugin also works without Core – then guard every Core call (1.4).

### 1.3 Getting the API

```java
import Vystorm.vystorm_core.API;
import Vystorm.vystorm_core.CoreApi;

API core = CoreApi.api().orElseThrow(() -> new IllegalStateException("Vystorm Core is not enabled"));
```

| Access | Returns | Notes |
| --- | --- | --- |
| `CoreApi.api()` | `Optional<API>` | Empty before Core's `onEnable` and after its `onDisable`. Preferred. |
| `getServer().getServicesManager().load(API.class)` | `API` or `null` | Same instance (Core registers it as a Bukkit service). |
| `main.getApi()` | `API` | Legacy; may return a stale instance after Core was disabled. Avoid in new code. |

`API` is the entry point for the per-feature handlers (`api.lang()`, `api.settings()`, `api.gui()`, …). Many newer
services are static facades in their own packages (`WebServices`, `ClientUi`, `SharedDatabase`, `ItemRefs`, …) and do
not need the `API` instance.

### 1.4 Version and capability checks

`Vystorm.vystorm_core.CoreApi` (since 0.11.0, all methods thread-safe):

| Member | Description |
| --- | --- |
| `static final String VERSION` | Running Core version, e.g. `0.24.0-Vystorm-Core-26.2` (`0.0.0` if unknown). Not a compile-time constant, so you always see the runtime value. |
| `static String version()` | Same as `VERSION`. |
| `static boolean atLeast(String minimum)` | Compares only the numeric `x.y.z` part. |
| `static boolean has(String capability)` | Whether this Core has the capability; unknown names → `false`. Case-insensitive. |
| `static Optional<String> since(String capability)` | Version that introduced the capability. |
| `static Map<String, String> capabilities()` | All capabilities → since-version (unmodifiable). |
| `static int compare(String a, String b)` | Numeric version comparison. |
| `static Optional<API> api()` | See 1.3. |
| `static Plugin plugin()` | Core's `Plugin`, or `null` if not loaded. |

Capability names are strings in `CoreApi.Capability` (use the constants, e.g. `CoreApi.Capability.WEB`):

| Constant | Name | Since | What it guarantees |
| --- | --- | --- | --- |
| `RUNTIME_HUBS` | `runtime-hubs` | 0.2.0 | `runtime.RuntimeContextHub`, `RuntimeActionHub` |
| `COMBAT_REGISTRY` | `combat-registry` | 0.2.0 | `combat.CombatRegistry` |
| `THREAT` | `threat` | 0.2.0 | `threat.ThreatServices` |
| `WORLD_EVENTS` | `world-events` | 0.3.0 | `worldevent.WorldEventServices` |
| `VERSIONED_STORE` | `versioned-store` | 0.3.0 | `storage.VersionedStore` |
| `CONFIG_MIGRATOR` | `config-migrator` | 0.3.0 | `config.ConfigMigrator` |
| `TASK_GROUPS` | `task-groups` | 0.3.0 | `schedule.TaskGroup` |
| `MYTHIC_BRIDGE` | `mythic-bridge` | 0.3.0 | `mythic.MythicMobsBridge` |
| `REGIONS` | `regions` | 0.4.0 | `region.RegionService` |
| `CURVES` | `curves` | 0.5.0 | `curve.Curve`, `CurveRegistry`, `RelativeStat` |
| `PARTY` | `party` | 0.6.0 | `party.PartyServices` |
| `SETTINGS` | `settings` | 0.7.0 | `api.settings()` |
| `CHAT_ACTIONS` | `chat-actions` | 0.8.0 | `_uiMethods.chat.ChatActions` |
| `MOB_SERVICES` | `mob-services` | 0.9.0 | `mob.MobServices`, `RelationServices`, `MobContextServices`, `MobExtensions` |
| `ITEM_REFS` | `item-refs` | 0.9.0 | `item.ItemRefs` |
| `ITEM_SERVICES` | `item-services` | 0.10.0 | `item.ItemServices` |
| `RESOURCE_PACK` | `resource-pack` | 0.10.0 | `pack.ResourcePacks` |
| `DISCIPLINES` | `disciplines` | 0.10.0 | `progression.DisciplineServices` |
| `LIFECYCLE_CLEANUP` | `lifecycle-cleanup` | 0.10.3 | Registries are released automatically on plugin disable |
| `TRIGGER_SINK_OWNER` | `trigger-sink-owner` | 0.10.3 | `MobExtensions.sink(Plugin, TriggerSink)` |
| `CORE_API` | `core-api` | 0.11.0 | `CoreApi` itself |
| `ITEM_REF_PRIORITY` | `item-ref-priority` | 0.12.0 | Several `ItemRefs` providers per namespace with priority |
| `MOB_REFS` | `mob-refs` | 0.12.0 | `mob.MobRefs` |
| `SPAWN_POINTS` | `spawn-points` | 0.13.0 | `spawn.SpawnPointService` |
| `SHARED_DATABASE` | `shared-database` | 0.14.0 | `database.SharedDatabase`, `SchemaMigrator`, `api.database().shared()` |
| `CHARACTER` | `character` | 0.16.0 | `progression.CharacterServices` |
| `WEB` | `web` | 0.17.0 | `web.WebServices`, `WebModule`, `WebRoute` |
| `WEB_LIVE` | `web-live` | 0.18.0 | `WebServices.push`, `publishData`, `WebModule.Builder.snapshot` |
| `WEB_CLIENT` | `web-client` | 0.19.0 | `web.ClientServices` (overlay instead of menu) |
| `WEB_CLIENT_UI` | `web-client-ui` | 0.20.0 | `web.ClientUi`, `ClientElement`, client events |
| `WEB_CLIENT_TRUST` | `web-client-trust` | 0.21.0 | `ClientServices.originTrusted`, protocol 2 |
| `LOCALIZATION` | `localization` | 0.22.0 | `api.lang()`, `lang.Lang`, `lang.Localization` |
| `WEB_LOCALIZATION` | `web-localization` | 0.23.0 | `WebContext.language()`, `/core/i18n`, `{{…}}` tokens, `Lang.ref` in web titles/errors |
| `WEB_OPEN` | `web-open` | 0.24.0 | `WebServices.openOrLink/openOrElse/overlayPage/broadcast`, bundled web UI |
| `WEB_SETTINGS` | `web-settings` | 0.24.0 | Web page `/app/settings`, `SettingsInterface.changed(player)` |

**The guard pattern.** `CoreApi` itself exists since 0.11.0. On an older Core, touching it throws a `LinkageError`
(`NoClassDefFoundError`), so wrap the first check:

```java
boolean web;
try { web = CoreApi.has(CoreApi.Capability.WEB_OPEN); } catch (LinkageError tooOld) { web = false; }
if (!web) {
    getLogger().severe("Vystorm Core >= 0.24.0 required (running: " + safeCoreVersion() + ")");
    getServer().getPluginManager().disablePlugin(this);
    return;
}

static String safeCoreVersion() {
    try { return CoreApi.VERSION; } catch (LinkageError tooOld) { return "< 0.11.0"; }
}
```

Keep code that references newer Core classes in separate classes/methods that you only call after the check; a
`NoSuchMethodError`/`NoClassDefFoundError` is thrown lazily when such code first runs. For optional features, check the
capability and fall back (e.g. no web page → chat menu only).

---

## 2. Build a plugin in 10 minutes

A complete plugin "Greeter": localized `/greet` command, a server settings section bound to `config.yml`, a personal
setting, and a web page with one API route. Requires Core ≥ 0.24.0.

```
greeter/
├── build.gradle.kts                  (as in 1.1, rootProject.name = "greeter" in settings.gradle.kts)
├── libs/Vystorm_Core.jar
└── src/main/
    ├── java/com/example/greeter/GreeterPlugin.java
    └── resources/
        ├── plugin.yml
        ├── config.yml
        ├── lang/en.yml
        ├── lang/de.yml
        └── web/greeter/index.html, app.js
```

**`plugin.yml`**

```yaml
name: Greeter
version: '${version}'
main: com.example.greeter.GreeterPlugin
api-version: '26.2'
depend: [vystorm_core]
commands:
  greet:
    description: Greets you
    usage: /greet [web]
permissions:
  greeter.admin:
    description: Edit Greeter server settings
    default: op
```

**`config.yml`**

```yaml
greeting:
  enabled: true
  suffix: "!"
```

**`lang/en.yml`** (English is the required fallback; every key your code uses must exist here)

```yaml
greet:
  hello: "<green>Hello, {player}{suffix}</green>"
  disabled: "<gray>Greetings are switched off."
  quiet: "<gray>You turned greetings off in /vsettings."
  players-only: "Only players can be greeted."
settings:
  title: "Greeter"
  description: "Greeting options"
  enabled: "Greetings enabled"
  enabled-desc: "Allow /greet on this server"
  suffix: "Suffix"
  suffix-desc: "Text after the player name"
  personal-title: "Greeter"
  personal-description: "Your greeting options"
  receive: "Greet me"
  receive-desc: "Show greetings to me"
web:
  module:
    title: "Greeter"
    description: "Greeting statistics"
  ui:
    title: "Greeter"
    loading: "Loading…"
    stats: "{count} greetings so far, {player}."
```

**`lang/de.yml`** (same keys; missing keys fall back to English)

```yaml
greet:
  hello: "<green>Hallo, {player}{suffix}</green>"
  disabled: "<gray>Begrüßungen sind ausgeschaltet."
  quiet: "<gray>Du hast Begrüßungen in /vsettings ausgeschaltet."
  players-only: "Nur Spieler können begrüßt werden."
settings:
  title: "Greeter"
  description: "Begrüßungsoptionen"
  enabled: "Begrüßungen an"
  enabled-desc: "/greet auf diesem Server erlauben"
  suffix: "Endung"
  suffix-desc: "Text nach dem Spielernamen"
  personal-title: "Greeter"
  personal-description: "Deine Begrüßungsoptionen"
  receive: "Mich begrüßen"
  receive-desc: "Begrüßungen für mich anzeigen"
web:
  module:
    title: "Greeter"
    description: "Begrüßungsstatistik"
  ui:
    title: "Greeter"
    loading: "Lädt …"
    stats: "Bisher {count} Begrüßungen, {player}."
```

**`GreeterPlugin.java`**

```java
package com.example.greeter;

import Vystorm.vystorm_core.API;
import Vystorm.vystorm_core.CoreApi;
import Vystorm.vystorm_core._uiMethods.settings.ConfigSettings;
import Vystorm.vystorm_core._uiMethods.settings.PlayerPrefs;
import Vystorm.vystorm_core._uiMethods.settings.SettingsSection;
import Vystorm.vystorm_core.lang.Lang;
import Vystorm.vystorm_core.web.WebModule;
import Vystorm.vystorm_core.web.WebResponse;
import Vystorm.vystorm_core.web.WebRoute;
import Vystorm.vystorm_core.web.WebServices;
import com.google.gson.JsonObject;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.plugin.java.JavaPlugin;

public final class GreeterPlugin extends JavaPlugin {
    private final AtomicInteger greetings = new AtomicInteger();   // read from web threads -> thread-safe type
    private Lang lang;

    @Override
    public void onEnable() {
        boolean ok;
        try { ok = CoreApi.has(CoreApi.Capability.WEB_OPEN); } catch (LinkageError tooOld) { ok = false; }
        if (!ok) {
            getLogger().severe("Vystorm Core >= 0.24.0 required");
            getServer().getPluginManager().disablePlugin(this);
            return;
        }
        API core = CoreApi.api().orElseThrow();
        saveDefaultConfig();

        // 1) Language bundle: lang/*.yml from this jar, copied to plugins/Greeter/lang/ for admins
        lang = core.lang().register(this);

        // 2) Settings (server thread; shown in /vsettings and on the web page /app/settings)
        core.settings().register(this, ConfigSettings.of(this)
                .toggle("greeting.enabled", lang.ref("settings.enabled"), lang.ref("settings.enabled-desc"))
                .text("greeting.suffix", lang.ref("settings.suffix"), lang.ref("settings.suffix-desc"), 16)
                .section("server", lang.ref("settings.title"), lang.ref("settings.description"), "greeter.admin", null));
        core.settings().register(this, SettingsSection.player("personal",
                lang.ref("settings.personal-title"), lang.ref("settings.personal-description"),
                List.of(PlayerPrefs.toggle(this, "receive", lang.ref("settings.receive"), lang.ref("settings.receive-desc"), true))));

        // 3) Web module: static page from web/greeter/ in this jar + one JSON route at /api/greeter/stats
        WebServices.register(this, WebModule.builder("greeter", lang.ref("web.module.title"))
                .description(lang.ref("web.module.description"))
                .assets("web/greeter/")
                .route(WebRoute.get("/stats", request -> {
                    // Web handlers run OFF the server thread: Bukkit/config access only via request.sync(...)
                    boolean enabled = request.sync(() -> getConfig().getBoolean("greeting.enabled"));
                    JsonObject out = new JsonObject();
                    out.addProperty("count", greetings.get());
                    out.addProperty("player", request.playerName());
                    out.addProperty("enabled", enabled);
                    return WebResponse.json(out);
                }).rateLimit(60).build())
                .build());
        // Nothing to undo in onDisable: Core removes settings, lang bundle and web module of a disabled plugin.
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!(sender instanceof Player player)) { lang.send(sender, "greet.players-only"); return true; }
        if (args.length == 1 && args[0].equalsIgnoreCase("web")) {
            // Overlay (client mod) if possible, else a one-time login link in chat, else Core's hint
            WebServices.openOrLink(player, "greeter", "", null);
            return true;
        }
        if (!getConfig().getBoolean("greeting.enabled")) { lang.send(player, "greet.disabled"); return true; }
        if (!PlayerPrefs.bool(this, player, "receive", true)) { lang.send(player, "greet.quiet"); return true; }
        greetings.incrementAndGet();
        lang.send(player, "greet.hello", "player", player.getName(), "suffix", getConfig().getString("greeting.suffix", "!"));
        return true;
    }
}
```

**`web/greeter/index.html`** (static assets are public; data comes only from the API; no inline scripts/styles – CSP)

```html
<!doctype html>
<html lang="{{@lang}}" data-vw-texts="{{@texts}}" data-vw-languages="{{@languages}}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="referrer" content="no-referrer">
  <title>{{greeter:web.ui.title}}</title>
  <link rel="stylesheet" href="/core/web.css">
  <script src="/core/web.js"></script>
  <script src="app.js"></script>
</head>
<body>
  <header data-vw-header></header>
  <main class="vw-main">
    <h1>{{greeter:web.ui.title}}</h1>
    <p id="stats" class="vw-muted" data-template="{{greeter:web.ui.stats}}">{{greeter:web.ui.loading}}</p>
  </main>
</body>
</html>
```

**`web/greeter/app.js`**

```js
(function () {
  'use strict';
  var W = window.VystormWeb;
  document.addEventListener('DOMContentLoaded', async function () {
    var box = document.getElementById('stats');
    try {
      var s = await W.get('/api/greeter/stats');          // JSON from the route above
      box.textContent = box.dataset.template               // textContent, never innerHTML
        .replace('{count}', String(s.count))
        .replace('{player}', s.player);
    } catch (e) {
      if (e.status !== 401) W.showError(e);               // 401 = not logged in, Core shows a hint
    }
  });
})();
```

Try it: set `web.enabled: true` in `plugins/vystorm_core/config.yml` (embedded mode, local), restart, then in game
`/greet`, `/vsettings` (Greeter sections), `/greet web` or `/web greeter` and open the link on the same machine. The
bundled web UI sends `/app/greeter` to your static page `/m/greeter/` because it has no built-in page for your module.

---

## 3. Ground rules: threads, lifecycle, errors

### 3.1 Threads

| API | Thread |
| --- | --- |
| `api.gui()`, `api.dialogs()`, `api.settings()` (register/open), `ChatActions` callbacks, `ClientServices`, `ClientUi`, `WebServices.loginLink/issueLogin/sendLink/openOrLink/openOrElse` | **Server thread only.** Most throw `IllegalStateException` otherwise. |
| `Lang` / `Localization`, `CoreApi`, `SharedDatabase`, `WebServices.register/push/broadcast/publishData`, `api.settings().changed(...)` | Any thread (thread-safe). |
| `WebHandler`, `WebSnapshot` callbacks | Run **off** the server thread (Core's handler pool). Use `request.sync(...)` for Bukkit. |
| JDBC connections from the shared pool | **Never** on the server thread. |
| `SharedDatabase.whenReady()` / `sharedReady()` callbacks | Core's init thread or the calling thread – never assume the server thread. |
| Legacy handlers (`area`, `particle`, `interactionGenerator`, `packages`, world lifecycle, XP helpers) | Server thread. |

To get back to the server thread: `Bukkit.getScheduler().runTask(plugin, () -> …)`. To leave it:
`runTaskAsynchronously`.

### 3.2 Lifecycle

Register everything in `onEnable`. When your plugin is disabled, Core releases its registrations automatically
(`lifecycle.CoreLifecycle` and Core's disable listeners): settings sections, menus (`gui().unregisterMenus`), open
dialogs, admin modules, inventory watchers, structure generators, interaction hitboxes, messaging channels, web modules,
client HUD elements, the lang bundle, and all provider registries (items, mobs, parties, world events, regions, spawn
points, runtime hubs, …). You only clean up your own tasks, files and connections – plus these registries that have
no owner and are **not** released automatically:

| Registry | Clean up in `onDisable` / on reload |
| --- | --- |
| `CurveRegistry` curves | `CurveRegistry.unregisterPrefix("curve.myplugin.")` |
| `api.inventoryInteractor()` entries | `unregister(entry)` |
| `api.area()` areas | `api.area().unregister(area)` |
| `api.worldGenerator()` named generators | `api.worldGenerator().unregister(name)` |
| Threat modifiers | `ThreatServices.get().removeOwner(this)` (only released automatically in some cases) |

Re-registering the same key while still enabled (e.g. from your own `/myplugin reload`) usually throws
`IllegalArgumentException` ("already registered") – unregister first (`api.settings().unregister(this)`,
`api.gui().unregisterMenus(this)`), or better: register once in `onEnable` and only reload values.

### 3.3 Errors

- Registries validate eagerly: invalid ids/sizes throw `IllegalArgumentException`/`NullPointerException` at
  registration time, not later.
- Exceptions thrown by *your* providers/callbacks are contained and logged by Core (`lifecycle.ProviderFailures`);
  they do not break other plugins. Do not rely on that – handle your own errors.
- `Lang` lookups never return `null` and never throw; a missing key renders as the key itself.

---

## 4. Localization (`api.lang()`)

Capability `localization` (0.22.0). Every plugin ships its own language files and gets one `Lang` bundle; Core picks the
language per viewer.

**Files.** `src/main/resources/lang/en.yml` (required, final fallback), `lang/de.yml`, any `lang/<code>.yml`. On
register Core copies missing files to `plugins/<Plugin>/lang/`, where admins edit them or add languages. Core never
rewrites an existing admin file; keys you add later work at once through the jar fallback.

**Lookup** for language `L`: disk `L` → jar `L` → disk `en` → jar `en` → the key itself.

**Language per viewer:** (1) the player's personal choice (`/vsettings` → Language), (2) your plugin override
`lang.setLanguage(code)` if set, else Core's `config.yml` `language.default` (`auto` = client locale, `de_at` → `de`;
or a fixed code), (3) `en`. Console/RCON get `en` unless the default is a fixed code.

**Format:** MiniMessage (`<red>`, `<bold>`, `<#ff8800>`, `<hover:…>`, `<click:…>`) and legacy `&c`/`§c`. Placeholders are
named – `{player}` – and passed as name/value pairs. Values are inserted literally (no tag injection from player
input); a `ComponentLike` value is inserted as a component.

### 4.1 `Localization` (service, `api.lang()` or `Localization.get()`)

| Method | Description |
| --- | --- |
| `Lang register(Plugin plugin)` | Registers (or returns the existing) bundle. Call in `onEnable`. |
| `Optional<Lang> bundle(Plugin plugin)` / `bundle(String namespace)` | Look up a registered bundle. |
| `Lang core()` | Core's own bundle (namespace `vystorm_core`). |
| `Set<String> languages()` | Language codes known to Core. |
| `String defaultLanguage()` | Current `language.default` (`auto` or a code). |
| `Optional<String> personalLanguage(UUID)` | A player's own choice, if any. |
| `String language(CommandSender)` | Language Core would use for the sender. |
| `static boolean isRef(String)` | Whether a string is a `Lang.ref(...)` reference. |
| `String resolve(CommandSender, String textOrRef)` | Resolves a ref (or returns the text) for a viewer. |
| constants `AUTO = "auto"`, `FALLBACK = "en"`, `KEY_PREFIX = "vlang:"` | |

`setDefaultLanguage`, `setPersonalLanguage`, `rememberClientLocale`, `configure`, `installTranslator`,
`uninstallTranslator`, `clearPlayers`, `unregister`, `reloadAll` are managed by Core (`/vadmin language reload`,
lifecycle) – don't call them from plugins.

### 4.2 `Lang` (your bundle) – thread-safe, cached

| Method | Description |
| --- | --- |
| `Component component(CommandSender viewer, String key, Object... placeholders)` | Formatted text in the viewer's language. |
| `Component component(String language, String key, Object... placeholders)` | In a given language code. |
| `String text(CommandSender viewer, String key, Object...)` / `text(String language, …)` | Legacy `§` string for String APIs (inventory titles, `setDisplayName`). |
| `String plain(CommandSender viewer, String key, Object...)` / `plain(String language, …)` | No formatting (logs, tab completion, web JSON). |
| `List<Component> lines(CommandSender viewer, String key, Object...)` / `lines(String language, …)` | One component per line (YAML list or `\n`), e.g. lore. |
| `void send(CommandSender viewer, String key, Object...)` | Chat message (console too). |
| `void actionBar(CommandSender viewer, String key, Object...)` | Action bar (console gets chat). |
| `ItemStack name(ItemStack item, CommandSender viewer, String key, Object...)` | Sets display name (non-italic). Returns the same stack. |
| `ItemStack lore(ItemStack item, CommandSender viewer, String key, Object...)` | Sets lore from `lines`. |
| `String ref(String key)` | `{lang:<namespace>:<key>}` – for text you register once and Core shows to many viewers (see below). |
| `Component translatable(String key, Object... args)` | Component rendered per viewer in Core dialogs and in chat; positional `{0}`, `{1}` in the text. |
| `void setLanguage(String codeOrAuto)` | Plugin-level override (`null`/blank/`auto` = follow Core). Personal choice still wins. |
| `String language(CommandSender)` | Code this bundle uses for the viewer (available language, else `en`). |
| `String languageFor(Locale)` / `languageFor(UUID player, List<String> browserLanguages)` | Code for a locale / an (offline) player. |
| `String resolveLanguage(String code)` | Available code for a wanted code (`de_at` → `de`, unknown → `en`). |
| `Map<String, String> texts(String language, String prefix)` | Raw texts below a prefix (prefix removed), English fills gaps. |
| `boolean has(String key)` / `String raw(String language, String key)` | Key present? / unformatted template. |
| `Set<String> languages()` / `String namespace()` | Available codes / lower-case plugin name (spaces → `_`). |
| `void reload()` | Re-reads files (call from your reload command). |
| `static Lang create(String namespace, Map<String, String> yamlByCode)` | Standalone bundle for unit tests (not registered; `ref` won't resolve). Overload with a `File folder`. |

```java
Lang lang = CoreApi.api().orElseThrow().lang().register(this);
lang.send(player, "shop.bought", "amount", 3, "item", itemName);
Component title = lang.component(player, "menu.title");
lang.name(item, player, "item.sword.name");
lang.lore(item, player, "item.sword.lore", "damage", 7);
```

**Text registered once, shown to many** – pass `lang.ref("key")` instead of a literal: menu titles/descriptions
(`gui().registerMenu`), settings section titles/labels/option labels/descriptions, admin module/action labels,
`dialogs().prompt/confirm` titles and questions, `MenuButtons.settings` labels, `ChatActions` labels/hovers, web module
titles/descriptions, `WebException` messages. Core resolves them per viewer. For components you build once (e.g. a
`DialogButton` label in a field) use `lang.translatable("key", args…)`.

**Web texts:** keys under `web.ui.` are available to web pages (`{{<namespace>:web.ui.key}}` in HTML, `GET
/core/i18n?ns=<namespace>`); they are sent as *raw* text – write them without MiniMessage tags.

---

## 5. Settings (`api.settings()`)

Capability `settings` (0.7.0), web page `web-settings` (0.24.0). Plugins register **sections**; Core renders them as
native dialogs (`/vsettings`, `/einstellungen`, main menu, pause menu) and – since 0.24.0 – on the web page
`/app/settings`. Values stay in your storage: you supply getter and setter.

### 5.1 `SettingsInterface`

| Method | Description |
| --- | --- |
| `void register(Plugin owner, SettingsSection section)` | Server thread. Key = `<lowercase plugin name>:<section id>`; duplicate → `IllegalArgumentException`. |
| `void unregister(Plugin owner)` | Removes all your sections (automatic on disable). |
| `List<SettingsSection> sections(Player, SettingScope)` | Sections the player may open. |
| `void openOverview(Player)` / `void openScope(Player, SettingScope)` | Open Core's settings dialogs. |
| `boolean openSection(Player, String key)` | Open `owner:id` (or a unique `id`); `false` if missing/not allowed. |
| `List<String> keys(Player)` | Keys the player may open. |
| `default void changed(Player player)` | (0.24.0) Tell open settings views that values changed outside Core's dialogs/web page; `null` = server-wide values. Any thread. |

### 5.2 `SettingsSection` (record)

`SettingsSection(String id, String title, String description, SettingScope scope, String permission, List<Setting> settings, Consumer<Player> afterSave)`

- `static SettingsSection player(String id, String title, String description, List<Setting> settings)` – personal.
- `static SettingsSection server(String id, String title, String description, String permission, List<Setting> settings, Consumer<Player> afterSave)` – server-wide; blank permission = `vystorm_core.admin`.
- `id` `[a-z0-9_-]{1,40}`; 1..`MAX_SETTINGS` (48) settings; unique setting ids. `afterSave` runs **once** after at
  least one value changed (e.g. save the file, reload runtime state).
- `SettingScope`: `PLAYER`, `SERVER`.

### 5.3 `Setting` (record)

Factories (getter/setter get the viewing `Player`; for server settings that is the editing admin). The setter returns
`null` on success or a short error message shown to the player (may be a `Lang.ref`).

```java
static Setting toggle(String id, String label, String description, Function<Player, Boolean> getter, BiFunction<Player, Boolean, String> setter)
static Setting number(String id, String label, String description, double min, double max, double step, Function<Player, Double> getter, BiFunction<Player, Double, String> setter)
static Setting choice(String id, String label, String description, LinkedHashMap<String, String> options, Function<Player, String> getter, BiFunction<Player, String, String> setter)
static Setting text(String id, String label, String description, int maxLength, Function<Player, String> getter, BiFunction<Player, String, String> setter)
Setting permission(String node)   // restrict just this setting
```

Rules: id `[a-z0-9_.-]{1,64}`; option id `[A-Za-z0-9_.:-]{1,64}` (options map id → label, insertion order); number
needs `min < max`, `step > 0`, finite (whole-number steps deliver whole `Double`s); text `maxLength` 1..256. Kinds:
`Setting.Kind.TOGGLE/NUMBER/CHOICE/TEXT`.

### 5.4 Helpers

`ConfigSettings` – a **server** section bound to config keys (written into the config, saved once after a save, then
your `reload` callback runs):

```java
static ConfigSettings of(Plugin plugin)                                        // getConfig()/saveConfig()
static ConfigSettings of(Supplier<FileConfiguration> config, Runnable save)    // any YAML file
ConfigSettings toggle(String path, String label, String description)
ConfigSettings integer(String path, String label, String description, int min, int max, int step)
ConfigSettings decimal(String path, String label, String description, double min, double max, double step)
ConfigSettings choice(String path, String label, String description, LinkedHashMap<String, String> options)
ConfigSettings text(String path, String label, String description, int maxLength)
ConfigSettings text(String path, String label, String description, int maxLength, Function<String, String> validator)
ConfigSettings integerText(String path, String label, String description, long min, long max)
ConfigSettings decimalText(String path, String label, String description, double min, double max)
ConfigSettings add(Setting setting)
SettingsSection section(String id, String title, String description, String permission, Consumer<Player> reload)
static String id(String path)                                                  // setting id derived from a path
```

`PlayerPrefs` – small per-player values in the player's persistent data (for plugins without own player storage):

```java
static boolean bool(Plugin owner, Player player, String id, boolean fallback)
static void setBool(Plugin owner, Player player, String id, boolean value)
static double number(Plugin owner, Player player, String id, double fallback)
static void setNumber(Plugin owner, Player player, String id, double value)
static String text(Plugin owner, Player player, String id, String fallback)
static void setText(Plugin owner, Player player, String id, String value)
static void clear(Plugin owner, Player player, String id)      // back to the fallback
static boolean has(Plugin owner, Player player, String id)
static Setting toggle(Plugin owner, String id, String label, String description, boolean fallback)
static Setting number(Plugin owner, String id, String label, String description, double min, double max, double step, double fallback)
static Setting choice(Plugin owner, String id, String label, String description, LinkedHashMap<String, String> options, String fallback)
```

### 5.5 Web settings page (0.24.0)

Every registered section automatically appears on `/app/settings` (`/web settings`): personal sections for everyone,
server sections only with their permission. Reads and saves run on the server thread through the same checks as the
dialog (section/setting permission, type, range, options, length, your setter's error, `afterSave` once). Values are
shown and changed only while the player is online. If your plugin changes a value elsewhere (own command/menu), call
`api.settings().changed(player)` (or `changed(null)` for server values) so open pages refresh.

```java
LinkedHashMap<String, String> modes = new LinkedHashMap<>();
modes.put("compact", lang.ref("settings.mode.compact"));
modes.put("full", lang.ref("settings.mode.full"));
core.settings().register(this, SettingsSection.player("display", lang.ref("settings.title"), lang.ref("settings.desc"), List.of(
        PlayerPrefs.choice(this, "mode", lang.ref("settings.mode"), lang.ref("settings.mode-desc"), modes, "full"),
        Setting.number("range", lang.ref("settings.range"), "", 4, 64, 4,
                p -> (double) store.range(p.getUniqueId()),
                (p, v) -> v > store.maxRange() ? lang.ref("settings.too-far") : store.setRange(p.getUniqueId(), v.intValue()) ? null : lang.ref("settings.failed")))));
```

---

## 6. Menus and managed inventories (`api.gui()`)

`GuiInteface` (sic). All GUI operations: **server thread**.

### 6.1 Main menu entries (`/vmenu`)

```java
void registerMenu(Plugin owner, String key, String title, Material icon, Predicate<Player> access, Consumer<Player> opener)
void registerMenu(Plugin owner, String key, String title, Material icon, Predicate<Player> access, Consumer<Player> opener, String group)
void registerMenu(Plugin owner, String key, String title, Material icon, Predicate<Player> access, Consumer<Player> opener, String group, String description)
void registerMenu(Plugin owner, String key, Function<Player, String> title, Material icon, Predicate<Player> access, Consumer<Player> opener, String group, Function<Player, String> description)   // 0.22.0, per viewer
boolean openMenu(Player player, String key)            // "plugin:key"; re-checks access
boolean hasVisibleMenus(Player player, String group)
void openDirectory(Player player, int page) / openDirectory(Player player, int page, String group)
List<String> menuKeys(Player player)
void unregisterMenus(Plugin owner)                     // automatic on disable
```

- Entry id = `NamespacedKey(owner, key)` → `<lowercase plugin name>:<key>`; `key` must be a valid `NamespacedKey` key
  (`[a-z0-9/._-]`). Duplicate → `IllegalArgumentException`.
- `group`: use `MenuCategory.X.group()` for the standard categories (`CHARACTER`, `ECONOMY`, `COMMUNITY`, `WORLD`,
  `BUILDING`, `TUTORIALS`, `ADMIN`); `""` = main directory. A category shows only while it has visible entries.
  `MenuCategory` also has `title(CommandSender)`, `langKey()`, `icon()`, `key()`, `static ordered()`, `static byGroup(String)`.
- `access` is checked when listing and again on open. `title`/`description` may be `lang.ref(...)`.

```java
core.gui().registerMenu(this, "shop", lang.ref("menu.shop"), Material.EMERALD,
        p -> p.hasPermission("myshop.use"), this::openShop,
        MenuCategory.ECONOMY.group(), lang.ref("menu.shop-desc"));
```

### 6.2 Managed inventories (chest GUIs with real items)

```java
Inventory createInventory(Plugin owner, InventoryHolder holder, UUID viewer, int size, String title, GuiCallbacks callbacks)
boolean isManaged(Inventory inventory)
void releaseInventory(Inventory inventory)   // for shared inventories (viewer == null)
```

`GuiCallbacks` (all `default`): `click(InventoryClickEvent)`, `drag(InventoryDragEvent)`, `close(InventoryCloseEvent)`,
`discarded(Inventory)`. Defaults **cancel** clicks and drags. Core dispatches events by inventory identity; a personal
inventory (`viewer` UUID) ignores clicks from others. Menu items do not trigger Core's global NBT interaction. Schedule
inventory switches after the current event (`runTask`). `viewer = null` binds a shared real inventory until
`releaseInventory`. The owner must be enabled (`IllegalStateException` otherwise).

```java
Inventory inv = core.gui().createInventory(this, null, player.getUniqueId(), 27, lang.text(player, "shop.title"),
        new GuiCallbacks() {
            @Override public void click(InventoryClickEvent e) {
                e.setCancelled(true);
                if (e.getRawSlot() == 13) Bukkit.getScheduler().runTask(plugin, () -> buy((Player) e.getWhoClicked()));
            }
        });
inv.setItem(13, lang.name(new ItemStack(Material.DIAMOND), player, "shop.diamond"));
player.openInventory(inv);
```

The older `createGui/addItem/addMenu/removeMenu/removeItem/setSpecificItem/getGui/getInventory/getMenuNames` methods and
`handleClick/handleDrag/handleOpen/handleOpenResult/handleClose` belong to the legacy auto-GUI and Core's own listeners –
don't use them in new plugins.

---

## 7. Native dialogs (`api.dialogs()`)

`DialogInterface`, Paper native dialogs. **Server thread.** Each dialog is bound to plugin, player and a one-time
token; opening and answers are deferred to the next tick; `valid` is re-checked; duplicate answers, expired dialogs
(60 s), a disabled owner or a replacing dialog never run old actions.

```java
void open(Plugin owner, Player player, DialogRequest request)
void prompt(Plugin owner, Player player, String title, String question, String initial, Consumer<String> submit, Runnable cancel, Predicate<Player> valid)
void confirm(Plugin owner, Player player, String title, String message, Runnable submit, Runnable cancel, Predicate<Player> valid)
void close(Player player)
void removeAll(Plugin owner)    // automatic on disable
```

`prompt`/`confirm` titles and texts may be `lang.ref(...)`; `prompt` input max length is 256.

`DialogRequest(Component title, List<DialogBody> body, List<DialogInput> inputs, List<DialogButton> buttons, Predicate<Player> valid, Runnable cancel, String cancelLabel, int columns)`
– short form without `cancelLabel`/`columns` (defaults: `DEFAULT_CANCEL`, 3 columns). `columns` 1..4; all lists and
`valid`/`cancel` non-null. `DialogBody`/`DialogInput` are Paper types (`io.papermc.paper.registry.data.dialog.*`).
The always-present exit button runs `cancel`; `cancel` also runs on ESC and on the 60 s timeout, **not** when replaced,
closed via `close(...)` or when the owner is disabled.

`DialogButton(Component label, Component tooltip, BiConsumer<Player, DialogResponseView> action)` / without tooltip.
Read inputs with `response.getText("id")`, `getFloat`, `getBoolean` (Paper `DialogResponseView`).

```java
core.dialogs().open(this, player, new DialogRequest(
        lang.component(player, "rename.title"), List.of(),
        List.of(DialogInput.text("name", lang.component(player, "rename.label")).maxLength(32).build()),
        List.of(new DialogButton(lang.translatable("common.save"), (p, r) -> rename(p, r.getText("name")))),
        p -> p.isOnline(), () -> {}));

core.dialogs().confirm(this, player, lang.ref("sell.title"), lang.ref("sell.question"),
        () -> sell(player), () -> {}, p -> p.getInventory().contains(Material.DIAMOND));
```

Validate every input server-side (price, ownership, stock); the dialog only checks shape. After a dialog, create
inventory menus fresh.

---

## 8. Chat actions and menu buttons

### 8.1 `_uiMethods.chat.ChatActions` (0.8.0)

Clickable chat components. Callbacks run on the server thread, check permissions at click time and live for
`LIFETIME` (30 min). Labels/hovers may be `lang.ref(...)`.

| Method | Description |
| --- | --- |
| `static TextComponent text(String legacy)` | Legacy `§` text as component, to append actions. |
| `static Component location(Location)` / `location(World, double, double, double)` / `location(Location, String permission)` | `[world x y z]`; click teleports with `TELEPORT_PERMISSION` (`vystorm_core.chat.teleport`, default op) or the given permission (empty = everyone). |
| `static Component button(String label, String hover, Consumer<Player> action)` | Runs every click, no permission check. |
| `static Component button(String label, String hover, String permission, Consumer<Player> action)` | Click-time permission check. |
| `static Component buttonOnce(String label, String hover, Consumer<Player> action)` / `+ String permission` | At most once in total (rewards, confirmations). |
| `static Component menu(String label, String menuKey)` | Opens a `/vmenu` entry (`plugin:key`). |
| `static Component settings(String label, String sectionKey)` | Opens a settings section (`plugin:section`). |
| `static Component command(String label, String hover, String command)` | Runs a command (with leading `/`) as the player. |
| `static Component suggest(String label, String hover, String command)` | Puts text into the chat input. |
| `static Component copy(String label, String value)` | Copies to clipboard. |

```java
player.sendMessage(ChatActions.text(lang.text(player, "quest.reward-ready")).append(Component.space())
        .append(ChatActions.buttonOnce(lang.ref("quest.claim"), lang.ref("quest.claim-hover"), p -> claim(p))));
```

### 8.2 `_uiMethods.menu.MenuButtons`

Shared `DialogButton`s for your own dialogs: `home()` (Core main menu), `category(MenuCategory)`,
`back(Consumer<Player> target)`, `settings(String sectionKey)`, `settings(String sectionKey, String label, Component tooltip)`
(label may be a `Lang.ref`), `settings(String sectionKey, Component label, Component tooltip)`. Labels render in each
viewer's language.

---

## 9. Tutorials

`api.registerTutorial(Plugin owner)` (or `TutorialGuide.register(API api, Plugin owner)`) registers a read-only guide
from `tutorial.yml` in your jar as menu entry `<plugin>:tutorial` in the Tutorials category. Optional translations
`tutorial_<code>.yml` (e.g. `tutorial_en.yml`); players get their language, unknown languages get `tutorial_en.yml` if
present, else `tutorial.yml`. Call once in `onEnable` (server thread).

```yaml
# src/main/resources/tutorial.yml
title: Greeter
permission: ''            # optional: only players with this permission see the guide
pages:
  - title: Getting started
    text: |
      1. Type /greet.
      2. Open /vsettings to turn greetings off.
  - title: Admins
    permission: greeter.admin     # optional per page
    text: |
      Server options: /vsettings server.
```

---

## 10. Admin dashboard modules (`api.admin()`)

`/vadmin` (`/admin`) shows a permission-checked dashboard. A module is a curated list of your **own command's**
sub-commands; Core runs them as the clicking admin (normal permission checks) and asks for confirmation before
`mutating` actions, showing the exact command.

```java
AdminModule(String id, String title, String plugin, String command, String permission, String nativeEntry, List<AdminAction> actions)
AdminAction(String id, String label, String permission, String fixedArguments, String argumentHint, String example, boolean mutating)
```

- `id`, `command`, action `id`: `[a-z0-9-]{1,40}`; ≤ 64 actions; `plugin` must equal your plugin's `getName()`.
  At most 128 modules in total; a module id owned by another plugin → `IllegalArgumentException`.
- `command` is your command name (no slash); `fixedArguments` the fixed sub-command; `argumentHint` (non-empty) makes
  Core ask for trailing arguments (max 192 chars, unsafe input rejected); `nativeEntry` (optional) is a command line
  opened by a "native menu" button. Your `SERVER` settings sections are linked automatically.
- `title`/`label` may be `lang.ref(...)`. `AdminInterface`: `register`, `unregister(Plugin)` (automatic on disable),
  `modules()`, `owner(String id)`.

```java
core.admin().register(this, new AdminModule("greeter", lang.ref("admin.title"), getName(), "greet", "greeter.admin", "",
        List.of(new AdminAction("reset", lang.ref("admin.reset"), "greeter.admin", "reset", "", "", true),
                new AdminAction("give", lang.ref("admin.give"), "greeter.admin", "give", "<player> <amount>", "Steve 5", true))));
```

---

## 11. Shared database

Capability `shared-database` (0.14.0). One MySQL/MariaDB pool for all Vystorm plugins, configured only in Core's
`config.yml` (`database.shared`, default `enabled: false`). Plugins never see credentials; each plugin owns tables with
its own prefix.

`api.database()` (`DataBaseInterface`, default methods) and the static `database.SharedDatabase`:

| Method | Description |
| --- | --- |
| `Optional<DataSource> shared()` / `SharedDatabase.dataSource()` | The pool if connected and tested; empty if disabled, still connecting, failed or closed. |
| `boolean isSharedAvailable()` / `SharedDatabase.available()` | Whether the pool is present right now. |
| `CompletableFuture<Optional<DataSource>> sharedReady()` / `SharedDatabase.whenReady()` | Completes once Core's asynchronous connection test is done (empty = not available). **Use this at startup.** |
| `String sharedTableName(String prefix, String table)` / `SharedDatabase.tableName(...)` | `<prefix>_<table>`: prefix 2–16 chars `[a-z][a-z0-9]*` (unique per plugin), table `[a-z0-9][a-z0-9_]*`, total ≤ 64. Invalid → `IllegalArgumentException` (never rewritten). |
| `SharedDatabase.status()` | `SharedDatabase.Status`: `DISABLED`, `CONNECTING`, `AVAILABLE`, `FAILED`, `CLOSED`. |
| `SharedDatabase.failure()` | `Optional<String>` reason of a failure (no credentials). |
| `SharedDatabase.ID` | `"vystorm"` – id of the pool in the legacy handler. Never call `unregisterDataBase("vystorm")`. |

`database.SchemaMigrator` – versioned schema steps, one row per owner in `vystorm_schema_versions`
(`SchemaMigrator.VERSION_TABLE`):

```java
new SchemaMigrator(String owner)                       // owner = your table prefix
SchemaMigrator step(int version, String... sql)        // version >= 1, unique
SchemaMigrator step(int version, SchemaMigrator.Step step)   // Step: void apply(Connection) throws SQLException
int migrate(DataSource ds) throws SQLException         // applies missing steps in order, saves version after each
int currentVersion(DataSource ds) throws SQLException
int latest() / String owner()
```

A database newer than your highest step → `IllegalStateException`. MySQL commits DDL implicitly: make steps idempotent
(`IF NOT EXISTS`) or one DDL statement per step.

```java
API core = CoreApi.api().orElseThrow();
core.database().sharedReady().thenAcceptAsync(db -> {
    if (db.isEmpty()) { useFlatFiles(); return; }
    DataSource ds = db.get();
    String stash = core.database().sharedTableName("greet", "stats");   // "greet_stats"
    try {
        new SchemaMigrator("greet")
                .step(1, "CREATE TABLE IF NOT EXISTS " + stash + " (player BINARY(16) PRIMARY KEY, count INT NOT NULL, updated BIGINT NOT NULL)")
                .migrate(ds);
        try (Connection c = ds.getConnection();
             PreparedStatement ps = c.prepareStatement("SELECT count FROM " + stash + " WHERE player = ?")) {
            // ... always PreparedStatement parameters for values
        }
        Bukkit.getScheduler().runTask(this, this::databaseReady);        // back to the server thread for Bukkit
    } catch (SQLException e) {
        getLogger().severe("Database: " + e.getMessage());
        useFlatFiles();
    }
}, task -> Bukkit.getScheduler().runTaskAsynchronously(this, task));
```

Rules: never use connections on the server thread; return them immediately (try-with-resources); never close the
`DataSource`; the pool is small and shared; store times as epoch millis (`BIGINT`); UUIDs as `BINARY(16)`; always
`PreparedStatement` parameters. There is no automatic reconnect after a failed start (needs a restart); short outages
while running are handled by the pool.

---

## 12. Player data, XP, preferences, file storage

### 12.1 XP (`api.player()`, `PlayerHandlerInterface`) – server thread

```java
int experience(Player player)                          // spendable XP incl. level progress
void giveExperience(Player player, int amount)         // like Bukkit giveExp; amount >= 0
void setExperience(Player player, int amount)          // amount >= 0
boolean withdrawExperience(Player player, int amount)  // false = not enough, XP unchanged
void depositExperience(Player player, int amount)
```

Negative amounts throw `IllegalArgumentException`.

### 12.2 Per-player values

- **Preferences:** `PlayerPrefs` (5.4) – booleans/numbers/strings in the player's persistent data, saved with the
  player file. Best choice for small per-player options.
- **Legacy string storage:** `api.player().getSetting(UUID, String)` / `setSetting(UUID, String, String)` and
  `getPlayerData(UUID, String)` / `setPlayerData(UUID, String, String[])` (temporary data). Core loads/saves player
  data on join/quit. Prefer `PlayerPrefs`, your own storage or the shared database for new code.

### 12.3 Item/inventory serialization (`api.data().dataEncoder()`)

`DataEncoder`: `String encodeItem(ItemStack)`, `ItemStack decodeItem(String)`, `String encodeInventory(ItemStack[])`,
`ItemStack[] decodeInventory(String)` – Paper's native serialization (meta, PDC and empty slots round-trip), plus
`encodeInt/decodeInt`, `encodeString/decodeString`, `encodeBoolean/decodeBoolean`.

### 12.4 Files

`_coreMethods.dataStorage.AtomicFiles`: `static void write(Path, byte[])` and `static void write(Path,
FileConfiguration)` – write-to-temp + atomic replace (throws `IOException`). Use it for your own YAML/JSON files.

The legacy `api.data()` loader/creator/saver/`dataEntry(Plugin, String folder, Path, String name)` storage (YAML
`DataEntry`s kept in memory and saved by Core) still works but has an awkward API; new plugins should use their own
files (`AtomicFiles`), `storage.VersionedStore` (section 17) or the shared database. Deprecated:
`DataStorageInterface.convertDataEntry`/`exportDataEntry`, `DataBaseEntry`, `Saver.saveDataEntryExtern`.

---

## 13. Debug messages

`_parsersDebuggerEtc.debugger.DebugMessages` – static, any thread, never throws:

```java
DebugMessages.low(String message)    / low(String message, Throwable failure)
DebugMessages.mid(String message)    / mid(String message, Throwable failure)
DebugMessages.high(String message)   / high(String message, Throwable failure)   // appends class, message, 8 stack lines
```

Start every message with a marker `[Name]` or `[Name 42]` (e.g. `"[Greeter/Store] save failed"`): the text in the
brackets without a trailing line number is the switch key in `plugins/vystorm_core/debug/states.yml`
(`Greeter/Store.LOW: false`). New switches default to on (except a few noisy Core ones); console output defaults to
HIGH/SHUTDOWN only (`_console.LOW` etc.). HIGH also goes to players with `vystormCore.debug`. Files:
`debug/low-mid/` and `debug/high-shutdown/`, each capped at 512 MiB, 8 MiB per file, messages ≤ 8192 chars.

`api.debug()` (`DebugInterface`) additionally offers `low/mid/high/shutdown(String, LocalDateTime)`,
`boolean highConfirmed(String, LocalDateTime, Duration)` (blocks until written and synced; `false` = switched off;
throws `IOException`, `InterruptedException`, `TimeoutException`), `flush(Duration)` and `reloadStates(Duration)`.
Don't log secrets, tokens or full payloads. `api.debug().shutdown(...)` requests a server shutdown – don't use it in
normal plugins.

---

## 14. Web platform

Capabilities `web` (0.17.0), `web-live` (0.18.0), `web-localization` (0.23.0), `web-open` + `web-settings` (0.24.0).
Package `Vystorm.vystorm_core.web`.

Core serves browser pages for all plugins – in a normal browser (`/web`) and, for players with the Vystorm client mod,
as an in-game overlay. Core handles login (one-time links), sessions, CSRF/Origin checks, permissions, rate limits,
body limits, handler timeouts, security headers and an audit log. **Browser and client are never authoritative:** every
change goes through your `WebHandler` on the server, which validates it.

The platform is off unless the admin sets `web.enabled: true` in Core's `config.yml`. Registering modules always works;
while the platform is off they are simply not served (`WebServices.available()` is `false`).

### 14.1 URL layout

| Path | Content |
| --- | --- |
| `/`, `/login`, `/app/<id>[/sub…]` | Web UI (bundled vystorm-web-ui, or a newer build configured by the admin). |
| `/m/<id>/…` | Your module's **static assets** (from your jar). Public – no login needed, cacheable. |
| `/api/<id>/…` | Your module's **routes** (JSON). Session required. |
| `/data/<id>/<name>.json`, `/data/<id>/<name>.<hash>.json` | Data published with `publishData`. |
| `/core/web.js`, `/core/web.css`, `/core/session`, `/core/live`, `/core/i18n` | Core helpers and endpoints. |

### 14.2 `WebServices` (static facade)

| Method | Thread | Description |
| --- | --- | --- |
| `static void register(Plugin owner, WebModule module)` | any | Registers/replaces your module. Throws `IllegalStateException` if another active plugin (or Core) owns the id, `IllegalArgumentException` if assets can't be loaded. |
| `static void unregister(Plugin owner)` / `static boolean unregister(Plugin owner, String moduleId)` | any | Remove your modules (automatic on disable). |
| `static boolean available()` | any | Platform configured and running. |
| `static boolean registered(String moduleId)` / `static Set<String> modules()` | any | Active modules. |
| `static Optional<String> loginLink(Player, String moduleId)` | server | One-time link `<public-url>/login#<token>` (~60 s, single use). Empty if platform off, module missing or no permission. |
| `static boolean sendLink(Player, String moduleId, Component label)` | server | Clickable link in chat (`label` null = default text). `false` + hint to the player if impossible. |
| `static Optional<WebLogin> issueLogin(Player, String moduleId, String issuer)` | server | Token without chat (for own channels). The token is secret – only for that player, never log it. |
| `static WebOpenResult openOrLink(Player, String moduleId, String subPath, Runnable fallback)` | server | (0.24.0) Overlay if client mod present and trusted, else chat link, else `fallback` (or Core's hint if `fallback` is null). |
| `static WebOpenResult openOrElse(Player, String moduleId, String subPath, Runnable fallback)` | server | (0.24.0) Overlay if possible, else `fallback` – no chat link. Use it to replace a chest/dialog menu. |
| `static String overlayPage(String moduleId, String subPath)` | any | (0.24.0) Page string for `ClientServices.show`. |
| `static boolean push(UUID player, String moduleId, String type, JsonElement data)` / `push(Player, …)` | any | (0.18.0) Live event to all open tabs/overlays of the player; returns whether a live connection exists. |
| `static int broadcast(String moduleId, String type, JsonElement data)` | any | (0.24.0) Live event to every connected player allowed to open the module; returns the count. |
| `static String publishData(Plugin owner, String moduleId, String name, JsonElement data)` | any | (0.18.0) Versioned static JSON; returns `/data/<id>/<name>.<hash>.json`. |
| `static Optional<String> dataUrl(String moduleId, String name)` | any | Current versioned URL. |
| `static Optional<String> publicUrl()` | any | Public base URL (no trailing slash) while running. |
| `static int revokeSessions(UUID player)` | any | Ends all web sessions of a player. |
| `static String coreModuleId()` / `USE_PERMISSION` | – | `"core"` / `"vystorm.web.use"` (base permission, default: everyone). |

`WebOpenResult`: `OVERLAY`, `LINK`, `FALLBACK`, `NONE`. `subPath`: `""` (module start page), `"tab/x"` (→
`/app/<id>/tab/x`) or `"#frag"`; letters, digits and `/_=&.,:~+-`, at most one `#fragment`, no `..` or `//`, about
150 chars max; invalid → `IllegalArgumentException`. The chat link always opens the module page (sub-path only in the
overlay).

Players can also open any module themselves with `/web <id>`.

### 14.3 `WebModule`

```java
WebModule.builder(String id, String title)      // id [a-z0-9-]{1,32} (WebModule.ID); title 1..64 chars (a lang.ref string counts)
    .description(String text)                   // ≤ 200 chars (may be lang.ref)
    .permission(String node)                    // for page list AND all routes; empty = only vystorm.web.use
    .assets(String classpathPrefix)             // e.g. "web/greeter/" in YOUR jar → /m/<id>/…
    .asset(String path, byte[] content)         // inline file (overrides a jar file of the same path)
    .route(WebRoute route) / .routes(Iterable<WebRoute>)
    .entry(String path)                         // start page, default "index.html"
    .snapshot(WebSnapshot source)               // live state for newly connected tabs (0.18.0)
    .listed(boolean)                            // show on the overview page, default true
    .build()
```

Reserved ids: `core`, `settings` (Core's modules), `logout`, `status`, `link` (`/web` sub-commands). Duplicate routes
(same method and pattern) → `IllegalArgumentException`.

**Static assets are public** (served without login and cached by gateways): never put secrets or player data in
them; data always comes from routes. Allowed extensions: html, js, mjs, css, json, svg, png, jpg, jpeg, gif, webp, ico,
woff, woff2, txt. Limits: 512 files, 4 MiB per file, 32 MiB per module. The Content-Security-Policy blocks inline
scripts and styles – put JS/CSS into files.

### 14.4 `WebRoute`

```java
WebRoute.get(String path, WebHandler handler)      // also post, put, patch, delete
WebRoute.builder(WebMethod method, String path, WebHandler handler)
    .permission(String node)        // extra permission, re-checked on every request
    .requiresOnline()               // player must be online in game, else HTTP 409; also requiresOnline(boolean)
    .rateLimit(int perMinute)       // per session and route, 1..6000, default 120 (DEFAULT_RATE_PER_MINUTE)
    .maxBody(int bytes)             // default 16 KiB (DEFAULT_MAX_BODY), max 1 MiB (MAX_BODY_LIMIT); GET always 0
    .mutating(boolean)              // audit-logged; default true for non-GET; GET may only be false
    .build()
```

Paths are relative to `/api/<id>`: fixed segments `[A-Za-z0-9._~-]{1,64}` and placeholders `{name}`
(`[a-zA-Z][a-zA-Z0-9]{0,31}`), at most 16 segments; `"/"` is the module root. `WebMethod`: `GET`, `POST`, `PUT`,
`PATCH`, `DELETE`. Every non-GET request needs the CSRF header and a matching `Origin` – `/core/web.js` and the web UI
handle that.

### 14.5 Handlers, requests, responses

`WebHandler`: `WebResponse handle(WebRequest request) throws Exception`.

**Handlers do not run on the server thread** (bounded pool, default timeout 10 s → HTTP 504). For Bukkit access use
`request.sync(...)` (runs on the server thread, default timeout 5 s → `WebException` 503).

`WebContext` (base of `WebRequest` and snapshot context):

| Method | Description |
| --- | --- |
| `UUID playerId()` / `String playerName()` | The logged-in player (name at login time). |
| `String moduleId()` | Your module id. |
| `String clientIp()` / `String sessionId()` | Trusted client IP; stable, non-secret session id. |
| `boolean online()` | Whether the player was online when the request arrived. |
| `<T> T sync(Callable<T> task)` / `sync(Callable<T> task, Duration timeout)` | Run on the server thread and wait. |
| `default String language()` | (0.23.0) Language of this request – use with your bundle: `lang.plain(request.language(), "web.error.x")`. |

`WebRequest` adds: `WebMethod method()`, `WebRoute route()`, `String path()`, `Map<String, String> pathParams()`,
`default String pathParam(String name)` (missing → `WebException` 400; values ≤ 128 chars, no control chars or `/`),
`Map<String, List<String>> query()`, `default Optional<String> query(String name)`, `JsonElement body()` (strict JSON;
`JsonNull` if empty), `<T> T body(Class<T> type)` (Gson, e.g. a record; mismatch → 400).

Core has already checked session, permissions, CSRF, rate limit and body size – **the values still come from the
browser: validate everything** (ranges, ownership, ids, state).

`WebResponse` (always JSON, `Cache-Control: no-store`): `static json(Object value)` (Gson; a `JsonElement` is used as
is), `static json(int status, Object value)` (200–299 or 400–599), `static noContent()` (204),
`static error(int status, String message)` → `{"error": "<code>", "message": "…"}`.

`WebException(int status, String message)` (400–599) + `badRequest`, `forbidden`, `notFound`, `conflict`. The message
is shown to the user unchanged (may be a `lang.ref`) – no internal details. Any other exception is logged by Core and
answered with HTTP 500 without details.

```java
record Allocate(String node, int points) {}

WebRoute.post("/nodes/{node}/allocate", request -> {
    String node = request.pathParam("node");
    Allocate body = request.body(Allocate.class);
    if (body == null || body.points() < 1 || body.points() > 5) throw WebException.badRequest(lang.ref("web.error.points"));
    boolean ok = request.sync(() -> tree.allocate(request.playerId(), node, body.points()));   // server thread
    if (!ok) throw WebException.conflict(lang.ref("web.error.no-points"));
    return WebResponse.json(tree.view(request.playerId()));
}).requiresOnline().rateLimit(30).maxBody(1024).build();
```

### 14.6 Live data and published data (0.18.0)

- `WebModule.Builder.snapshot(WebSnapshot)` – `Map<String, JsonElement> snapshot(WebContext ctx) throws Exception`:
  current state when a tab/overlay connects (keys = event types). Runs off the server thread (same timeout as handlers).
- `WebServices.push(player, moduleId, type, json)` – after every change (cheap no-op without a connection). Events of
  the same module+type are coalesced per player (100 ms); max 16 KiB per event; `type` `[a-z0-9][a-z0-9._:-]{0,63}`
  (e.g. `character`, `node:12`). Invalid type or too large → `IllegalArgumentException`.
- `WebServices.publishData(owner, moduleId, name, json)` – large, rarely changing JSON (layouts, catalogs); `name`
  `[a-z0-9][a-z0-9-]{0,47}`, ≤ 8 MiB; module must be yours (`IllegalStateException`). Private (session needed) if the
  module has a permission, else public. Call after registering and on reload.
- The browser receives `{"kind":"event","module","type","data","snapshot"}`, `{"kind":"ready"}` (snapshot done),
  `{"kind":"data","module","name","url"}` and `{"kind":"bye","reason"}` over Server-Sent Events (`GET
  /core/live?modules=<id>`). No polling.

```java
WebServices.register(this, WebModule.builder("stats", lang.ref("web.title"))
        .snapshot(ctx -> Map.of("summary", summaryJson(ctx.playerId())))
        .route(WebRoute.get("/summary", r -> WebResponse.json(summaryJson(r.playerId()))).build())
        .build());
// whenever the value changes (any thread):
WebServices.push(playerId, "stats", "summary", summaryJson(playerId));
```

### 14.7 Writing the page

**Option A – static page in your jar (works everywhere, no Core release needed).** Register `.assets("web/<id>/")`.
The bundled UI sends `/app/<id>` to `/m/<id>/` when it has no built-in page for your module. Use Core's helpers:

```html
<html lang="{{@lang}}" data-vw-texts="{{@texts}}" data-vw-languages="{{@languages}}">
<link rel="stylesheet" href="/core/web.css">
<script src="/core/web.js"></script>
<script src="app.js"></script>
<header data-vw-header></header>   <!-- shared header: brand, theme, language, logout -->
```

- HTML tokens (rendered per request language, HTML-escaped): `{{<namespace>:web.ui.key}}` – raw text from **your**
  bundle (namespace = lower-case plugin name); `{{@lang}}`; `{{@texts}}` (Core's `web.ui.*` texts as JSON for
  `VystormWeb.t`); `{{@languages}}`. Pages without `{{` are served unchanged.
- `GET /core/i18n?ns=<namespace>` → `{language, languages:[{code,name}], texts:{…}}` with your keys below `web.ui.`
  (prefix removed).
- `window.VystormWeb`: `session()`, `get(path)`, `post(path, body)`, `put`, `patch`, `del` (CSRF header automatic;
  errors throw `VystormWeb.WebError {status, code, message}`; 401 shows a re-login hint), `live(modules, onMessage)` →
  `{close()}`, `el(tag, attrs, ...children)` (safe DOM, no `innerHTML`), `showError(error, target)`, `t(key, vars)`,
  `setLanguage(code)`, `logout()`, `toggleTheme()`.
- Styling: classes `vw-main`, `vw-card`, `vw-button`, `vw-primary`, `vw-input`, `vw-table`, `vw-row`, `vw-muted`,
  `vw-notice`, `vw-mono`; theme tokens `--vw-bg`, `--vw-surface`, `--vw-text`, `--vw-accent` (light/dark).
- Never use `innerHTML`/inline handlers with server or user data; use `textContent`/`VystormWeb.el`.

**Option B – page inside the web UI app (vystorm-web-ui, Preact).** For first-party Vystorm plugins: add
`src/modules/<id>/` plus one line in `src/modules/index.js`, use `useApp`, `useLive(module, type)`,
`usePluginTexts(namespace, fallbacks)`; the page ships with the next Core release (Core bundles the UI build). Keep a
static fallback page (option A) so older UIs still work.

### 14.8 Settings page for free (0.24.0)

All sections registered with `api.settings()` appear on `/app/settings` automatically (section 5.5). Endpoints (used by
the UI): `GET /api/settings/sections`, `POST /api/settings/save {section, values:{id:value}}`. Live event
`settings/changed` refreshes open pages; call `api.settings().changed(player)` after changing values yourself.

---

## 15. Client mod: overlay and native HUD elements

The optional **Vystorm client mod** (Fabric/NeoForge) shows web pages as an in-game overlay and renders native HUD
elements. Everything here is **additive**: players without the mod must get the same function through vanilla means
(chest/dialog menus, chat links, boss bars, action bars). All calls: **server thread**.

### 15.1 `ClientServices` (0.19.0)

| Member | Description |
| --- | --- |
| `static boolean hasClient(Player)` | Handshake done. |
| `static int capabilities(Player)` | `CAP_*` bits (0 without mod). |
| `static int clientState(Player)` | `STATE_*` bits (0 without mod). |
| `static boolean overlayAvailable(Player)` | Overlay possible **and** the client would follow a show request (player trusted the web origin). |
| `static boolean show(Player, String moduleId, String page)` | Open a module in the overlay; `page` e.g. `"/app/skills#node/12"` or `""`. `false` → open your normal menu. |
| `static boolean close(Player, String moduleId)` | Close the overlay (null/empty = any module). |
| `static boolean originTrusted(Player)` | (0.21.0) Hint only, not a permission. |
| `static boolean uiHidden(Player)` / `static boolean uiAvailable(Player, int capability)` | (0.20.0) Native elements hidden (key H) / accepted. |
| `static boolean enabled()` | Client channel on (`web.client.enabled`). |

Capabilities: `CAP_BROWSER_OVERLAY`, `CAP_EXTERNAL_BROWSER`, `CAP_HUD`, `CAP_TOASTS`, `CAP_MARKERS`, `CAP_TOOLTIPS`,
`CAP_JS_BRIDGE`. States: `STATE_UI_HIDDEN`, `STATE_OVERLAY_OPEN`, `STATE_ORIGIN_TRUSTED`.

Prefer the 0.24.0 shortcuts: `WebServices.openOrElse(player, id, sub, () -> openMenu(player))`.

### 15.2 `ClientUi` (0.20.0)

```java
ClientUi ui = ClientUi.of(this);            // per plugin; ids get the namespace "<plugin>:" (xp → greeter:xp)
boolean available(Player, int capability)  // CAP_HUD, CAP_TOASTS, CAP_MARKERS, CAP_TOOLTIPS
boolean has(Player, String localId)
boolean put(Player, ClientElement.Element<?> element)          // create or replace; false without mod/capability or if invalid
boolean patch(Player, ClientElement.Element<?> fields)          // change set fields only; false if the element is gone → put
boolean remove(Player, String... localIds)
boolean clear(Player) / void clearAll()
boolean toast(Player, ClientElement.Toast toast)
// raw JSON variants: put(Player, JsonObject), patch(Player, String, JsonObject), toast(Player, JsonObject)
String fullId(String localId) / String namespace() / Plugin owner()
```

Core validates every element with the client's schema (invalid → `false` + one log warning, limits are rejected, not
truncated), batches patches (default 4 per second per element, unchanged fields not sent), rate-limits sending, and
removes your elements when your plugin is disabled. Client limits: 32 HUD elements, 64 markers, 128 tooltip rules per
player.

### 15.3 `ClientElement` builders

Local ids `[a-z0-9][a-z0-9_.:/-]*`. A builder contains only the fields you set, so the same builder works as a patch.
Builders are not thread-safe.

| Factory | Type | Main setters |
| --- | --- | --- |
| `ClientElement.panel(String id)` | HUD box | `width(int)` (0 or 16..320), `padding`, `background`, `border`, `textColor`, `title(String/Text)`, `icon(Icon)`, `line(String/Text)` (≤ 12), `lines(List<Text>)` |
| `ClientElement.bar(String id)` | progress bar | `width`, `height`, `value(double)`, `max(double)`, `color`, `background`, `textColor`, `label(String/Text)`, `segments(int)` |
| `ClientElement.slots(String id)` | skill bar | `size(int)` (16..32), `gap`, `slot(Slot)` (≤ 12), `slots(List<Slot>)` |
| `ClientElement.slot()` | slot | `icon`, `key(String)` (≤ 4 chars), `count`, `cooldown(long remainingMs, long totalMs)`, `active(boolean)`, `label` |
| `ClientElement.marker(String id)` | world marker | `at(x, y, z)` (required), `world(String/World)`, `label`, `color`, `icon`, `maxDistance`, `showDistance`, `clampToScreen` |
| `ClientElement.tooltip(String id)` | tooltip rule | `matchItem(String/Material)`, `matchCustomData(String, String)`, `line(String/Text)` (1..8) |
| `ClientElement.toast(String/Text title)` | one-off message | `text`, `icon`, `duration(int)`, `style(Toast.Style.INFO/SUCCESS/WARNING/ERROR)` |

HUD elements (`panel`, `bar`, `slots`) also have `anchor(ClientElement.Anchor)` (`TOP_LEFT`, `TOP`, `TOP_RIGHT`, `LEFT`,
`CENTER`, `RIGHT`, `BOTTOM_LEFT`, `BOTTOM`, `BOTTOM_RIGHT`), `offset(int, int)` (±4000 GUI px, inward from the anchor),
`order(int)` (±1000). Every element: `ttl(long millis)` (0 = until removed).
`ClientElement.Text`: `plain(String)`, `of(String text, String color)`, `then(...)`, `bold()`, `italic()`,
`underlined()`, `strikethrough()`; colors `#RRGGBB`, `#AARRGGBB` or a Minecraft color name; ≤ 16 parts, 256 chars.
`ClientElement.Icon`: `item(String/Material)`, `item(String item, String model)`, `sprite(String)`,
`image(String webPath)` (PNG on the web platform, ≤ 64 KiB, ≤ 128 px).

### 15.4 Events (`org.bukkit.event.Listener`)

- `VystormClientReadyEvent` – handshake accepted: `capabilities()`, `has(int)`, `modVersion()`, `loader()`. Good place
  for the first `put`.
- `VystormClientStateEvent` – state changed: `state()`, `previous()`, `uiHidden()`, `uiHiddenChanged()`,
  `overlayOpen()`, `originTrusted()`, `originTrustedChanged()`. Show your vanilla display again when `uiHidden()`.
- `VystormClientUiRejectEvent` – the client rejected one of your elements: `owner()`, `elementId()`, `localId()`,
  `code()` (`INVALID`, `LIMIT`, `UNKNOWN_TYPE`, `UNKNOWN_ID`, `RATE`), `detail()`.

```java
@EventHandler public void onReady(VystormClientReadyEvent e) {
    if (!e.has(ClientServices.CAP_HUD)) return;
    ui.put(e.getPlayer(), ClientElement.bar("xp").anchor(ClientElement.Anchor.BOTTOM).offset(0, 52)
            .value(xp(e.getPlayer())).max(need(e.getPlayer())).label("Lv " + level(e.getPlayer())));
}
void onXpChange(Player p) {
    if (!ui.patch(p, ClientElement.bar("xp").value(xp(p)))) { /* not there (yet) → put, or vanilla boss bar */ }
}
```

---

## 16. Items, regions, curves, NBT and legacy handlers

### Items

Core does not implement items. It is the neutral layer between the item engine (Vystorm-Items) and everyone else
(mobs, loot, markets, storage, skills). This layer has two parts:

- `item.ItemRefs`: resolves text references such as `vystorm:ORC_WAR_AXE` to an `ItemStack`, for any plugin that
  registers a namespace.
- `item.ItemServices`: a facade over **one** `ItemProvider` (the item engine) for reading, creating, validating,
  storing, repairing and rolling loot for managed items.

Never import classes from Vystorm-Items. Go through these two classes.

| Capability (`CoreApi.Capability`) | Since | Covers |
| --- | --- | --- |
| `ITEM_REFS` | 0.9.0 | `ItemRefs`, `ItemRefResolver` |
| `ITEM_SERVICES` | 0.10.0 | `ItemServices`, `ItemProvider`, the records, `item.event.*` |
| `CORE_API` | 0.11.0 | `ItemServices.repair/damage/encode/decode/templateExists/provenance` |
| `ITEM_REF_PRIORITY` | 0.12.0 | Priorities, several providers per namespace, `providers()`, built-in `mythic:`/`mmoitems:` fallbacks |
| `LIFECYCLE_CLEANUP` | 0.10.3 | Automatic unregistration on plugin disable |

#### ItemRefs – item references

**Reference syntax** (`namespace:id`; the namespace is case-insensitive and everything after the **first** colon is the id):

| Reference | Resolved by |
| --- | --- |
| `DIAMOND`, `minecraft:diamond` | Built in: `Material.matchMaterial(id)`. Air and non-item materials give empty. |
| `vystorm:<TEMPLATE>` | Vystorm-Items (registered at `PRIORITY_VYSTORM`) |
| `mythic:<Item>` | Registered providers first (Vystorm-Items imports, then bridges), then the built-in MythicMobs fallback |
| `mmoitems:TYPE:ID` | Registered providers first, then the built-in MMOItems fallback (the id `TYPE:ID` is upper-cased) |
| `<yourns>:<id>` | Your `ItemRefResolver` |

`vanilla:` is **not** a built-in namespace. Use `minecraft:` or a bare material name.

| Method | Notes |
| --- | --- |
| `static void register(Plugin owner, String namespace, ItemRefResolver resolver)` | Same as priority `PRIORITY_DEFAULT` |
| `static void register(Plugin owner, String namespace, int priority, ItemRefResolver resolver)` | 0.12.0. The namespace is trimmed and lower-cased. If the same owner registers again, only its own entry is replaced. |
| `static void unregisterAll(Plugin owner)` | Runs automatically on disable |
| `static Set<String> namespaces()` | Namespaces that have at least one registration |
| `static boolean available(String namespace)` | `true` for `minecraft`, for a namespace with an enabled provider, or when a built-in fallback's plugin is enabled |
| `static List<String> providers(String namespace)` | 0.12.0. Names of the provider plugins in resolution order (built-in fallbacks not included) |
| `static Optional<ItemStack> resolve(String ref, int amount)` | Returns a clone. `amount < 1` is treated as 1. A blank or null ref gives empty. |

Constants: `PRIORITY_VYSTORM = 100` (native Vystorm plugins), `PRIORITY_DEFAULT = 0`, `PRIORITY_BRIDGE = -100`
(adapters to external plugins, used only as a fallback).

`ItemRefResolver` is a functional interface: `Optional<ItemStack> resolve(String id, int amount)`. Return
`Optional.empty()` for ids you do not know. Do not throw and do not return null.

**Resolution order** for a namespace:

1. Enabled providers, highest priority first.
2. At equal priority, the plugin that owns the current `ItemServices` provider.
3. Then the newest registration.

The first non-empty result wins. If a provider throws, Core logs it (throttled) and asks the next one. After all
providers, the built-in `mythic`/`mmoitems` fallbacks run.

- **Threading:** registration is thread-safe. Call `resolve` on the **server thread**, because resolvers are server-thread code.
- **Lifecycle:** register in `onEnable`. Core removes your entries when your plugin is disabled.

```java
// onEnable
if (CoreApi.has(CoreApi.Capability.ITEM_REF_PRIORITY)) {
    ItemRefs.register(this, "myplugin", ItemRefs.PRIORITY_DEFAULT,
            (id, amount) -> Optional.ofNullable(templates.get(id)).map(t -> t.asQuantity(amount)));
}
// Later, on the server thread: a reward reference taken from config
ItemStack reward = ItemRefs.resolve(getConfig().getString("reward", "minecraft:diamond"), 3)
        .orElseGet(() -> new ItemStack(Material.DIAMOND, 3));
```

#### ItemServices – the item-engine facade

Every facade method is **guarded**. With no provider registered, or when the provider throws, the call returns the
fallback shown below: empty, `NOT_MANAGED`, an empty list or map, or `false`. Provider errors are logged (throttled).
Without a provider, items behave like vanilla items.

| Method | Fallback |
| --- | --- |
| `static boolean available()` | – |
| `static Optional<ItemView> describe(ItemStack stack)` | empty (also for null or air) |
| `static boolean managed(ItemStack stack)` | `false` (`describe(stack).isPresent()`) |
| `static Optional<ItemStack> create(ItemCreateSpec spec)` | empty |
| `static ItemValidation validate(ItemStack stack)` | `ItemValidation.notManaged()` |
| `static Optional<int[]> durability(ItemStack stack)` | empty. The array is `[current, max]`; empty means the item has no own durability. |
| `static List<ItemStack> generateLoot(String profileId, LootContext context)` | `List.of()`. Profile ids are defined in the provider's config. |
| `static Map<String, Double> equipmentStats(Player player)` | `Map.of()` (a cached snapshot of the equipment the player is wearing) |
| `static Optional<ItemStack> repair(ItemStack stack, int amount, Player player)` | empty. Returns a new stack; the original is unchanged. (0.11.0) |
| `static Optional<ItemStack> damage(ItemStack stack, int amount, Player player)` | empty. Returns a new stack, or AIR if the item breaks. (0.11.0) |
| `static Optional<StoredItem> encode(ItemStack stack)` | empty (also for unmanaged items). A lossless storage form. (0.11.0) |
| `static Optional<ItemStack> decode(StoredItem stored)` | empty. **If empty, do not delete the stored copy.** (0.11.0) |
| `static boolean templateExists(String templateId)` | `false` (0.11.0) |
| `static Optional<String> provenance(UUID instanceId)` | empty. Looks up the player or place of an instance in the provenance database; never scans worlds. (0.11.0) |
| `static Optional<ItemProvider> provider()` | Raw provider. Calls through it are **not** guarded, so prefer the facade. |
| `static void register(Plugin owner, ItemProvider provider)` / `static void unregister(Plugin owner)` | **Engine only.** There is exactly one provider. Replacing another plugin's provider logs a warning. `unregister` removes the provider only when `owner` matches. |
| `static void fire(Event event)` | **Engine only.** Fires an `item.event.*` event. Off the server thread, the event is deferred to the next tick, so fire cancellable events only from the server thread. |

- **Threading:** call every method on the **server thread**. `ItemProvider` is a server-thread contract.
- **Lifecycle:** the provider is removed automatically when its plugin is disabled.

**Records**

- `ItemCreateSpec(String templateId, int amount, int level, String rarity, String sourceType, String sourceId, UUID actor, long seed, Map<String, String> attributes)`
  - A blank `templateId` throws `IllegalArgumentException`. `amount` is at least 1.
  - `level <= 0` or `rarity == null` means the template decides. A null `sourceType` becomes `SYSTEM_REWARD`.
  - Factory: `static ItemCreateSpec of(String templateId, int amount, String sourceType, String sourceId)`.
  - `sourceType` is the provenance: `CRAFTING`, `LOOT_MOB`, `LOOT_BOSS`, `DUNGEON`, `WORLD_EVENT`, `QUEST`,
    `NPC_VENDOR`, `MARKET`, `ADMIN`, `MIGRATION` or `SYSTEM_REWARD`.
- `ItemView(UUID instanceId, String templateId, String type, Set<String> groups, String rarity, String tier, int level, double quality, Optional<UUID> boundTo, Set<String> tags, Map<String, Double> stats, int schemaVersion)`
  - A read-only view.
  - `boolean is(String typeOrGroup)`: case-insensitive match on `type` (for example `DAGGER`) or on a group
    (`WEAPON`, `MELEE`, `ARMOR`, `TOOL`, …).
  - `double stat(String id)`: returns 0 when the stat is missing. Stat ids are defined by the provider.
- `ItemValidation(Status status, List<String> messages)`
  - `Status`: `VALID`, `NOT_MANAGED`, `UNKNOWN_TEMPLATE`, `OUTDATED`, `TAMPERED`, `DUPLICATE`, `QUARANTINED`.
  - `boolean ok()` is `true` for `VALID`, `NOT_MANAGED` and `OUTDATED`.
  - Factories: `static ItemValidation valid()`, `static ItemValidation notManaged()`.
- `LootContext(String sourceType, String sourceId, UUID player, Set<UUID> party, Location location, String region, String dungeon, String faction, double threat, int mobLevel, boolean elite, boolean boss, double difficulty, double luck, Map<String, Double> eventModifiers)`
  - Factory: `static LootContext of(String sourceType, String sourceId, UUID player, Location location)`. Its
    defaults are mobLevel 1, difficulty 1, luck 0, threat 0 and no party.
  - `LootContext withMob(int level, double threat, boolean elite, boolean boss)`.
  - Callers only supply these values. The provider does all the rolling.
- `StoredItem(int schemaVersion, String templateId, UUID instanceId, String fingerprint, int amount, String data)`
  - `data` is the Base64-serialized full stack. The other fields carry identity for stack and duplicate rules.

`ItemProvider` (for implementing an engine, rarely needed) declares the same operations with these signatures:
`Optional<ItemView> describe(ItemStack)`, `Optional<ItemStack> create(ItemCreateSpec)`, `ItemValidation validate(ItemStack)`,
`Optional<int[]> durability(ItemStack)`, `ItemStack repair(ItemStack, int, Player)`, `ItemStack damage(ItemStack, int, Player)`,
`Optional<StoredItem> encode(ItemStack)`, `ItemStack decode(StoredItem)`, `List<ItemStack> generateLoot(String, LootContext)`,
`Map<String, Double> equipmentStats(Player)`, `boolean templateExists(String)`, `Optional<String> provenance(UUID)`.

```java
// Server thread: roll loot for a mob kill and inspect the weapon in hand
if (CoreApi.has(CoreApi.Capability.ITEM_SERVICES) && ItemServices.available()) {
    LootContext ctx = LootContext.of("LOOT_MOB", "orc_grunt", killer.getUniqueId(), mob.getLocation())
            .withMob(12, 1.5, false, false);
    ItemServices.generateLoot("orc_camp", ctx)
            .forEach(drop -> mob.getWorld().dropItemNaturally(mob.getLocation(), drop));
    ItemServices.describe(killer.getInventory().getItemInMainHand())
            .filter(view -> view.is("WEAPON"))
            .ifPresent(view -> getLogger().info(view.templateId() + " lvl " + view.level()));
}
```

#### Item events (`item.event`)

The engine fires these events through `ItemServices.fire`. Listeners run on the server thread. Every event has
`getHandlerList()`.

| Event | Cancellable | Accessors |
| --- | --- | --- |
| `ItemCreateEvent` | no | `ItemView item()`, `ItemCreateSpec spec()`, `Player player()` |
| `ItemLootEvent` | no | `String profileId()`, `LootContext context()`, `List<ItemStack> items()`. The list is mutable until delivery. |
| `ItemEquipEvent` | no | `Player player()`, `ItemView item()`, `String slot()`, `boolean equipped()`. The stat snapshot is already updated when this fires. |
| `ItemModifyEvent` | yes | `Player player()`, `ItemView item()`, `String kind()` (one of `UPGRADE`, `REFORGE`, `REPAIR`, `SALVAGE`, `SOCKET_INSERT`, `SOCKET_REMOVE`, `IDENTIFY`, `BIND`; Vystorm-Items also sends `SKIN`, so treat the set as open), `String detail()` |
| `ItemCraftEvent` | yes, at stages `START`/`ACTION` | `Player player()`, `String recipeId()`, `String station()`, `String stage()` (`START`, `ACTION`, `COMPLETE`, `FAILED`), `String action()`, `double quality()`, `ItemStack result()` |

```java
@EventHandler
public void onModify(ItemModifyEvent e) {
    if (e.kind().equals("SALVAGE") && e.item().tags().contains("soulbound")) e.setCancelled(true);
}
```

### Mythic bridges

The package `mythic` gives optional, reflection-only access to MythicMobs and MMOItems. Neither plugin becomes a hard
or soft dependency. When one is missing or disabled, every method reports "not available" and callers fall back to
vanilla behaviour. Since 0.12.0 these bridges are **only a fallback**:

- For mobs, use `mob.MobRefs` (Vystorm-Mobs).
- For items, use `ItemRefs`, which already consults both bridges as its last step.

Call a bridge directly only when you specifically mean MythicMobs or MMOItems. Capability: `MYTHIC_BRIDGE` (0.3.0);
the `item`, `active` and `MmoItemsBridge` members need `ITEM_REF_PRIORITY` (0.12.0).

| Method | When absent | Notes |
| --- | --- | --- |
| `MythicMobsBridge.available()` | `false` | The MythicMobs plugin is present and enabled |
| `MythicMobsBridge.active()` | `false` | Like `available()`, but returns `false` instead of throwing when no server is running (tests). (0.12.0) |
| `static Optional<ItemStack> MythicMobsBridge.item(String id, int amount)` | empty | Returns a copy of the MythicMobs item. Empty for unknown ids. (0.12.0) |
| `static boolean MythicMobsBridge.exists(String mobId)` | `false` | |
| `static Entity MythicMobsBridge.spawn(String mobId, Location location, int level)` | `null` | See the spawn contract below |
| `static boolean MythicMobsBridge.installPack(Plugin owner, String resource, String fileName) throws IOException` | `false` | Installs a bundled mob file into MythicMobs' `Mobs/` folder, then runs `mythicmobs reload` once. See the install rules below. |
| `static boolean MmoItemsBridge.available()` | `false` | (0.12.0) |
| `static Optional<ItemStack> MmoItemsBridge.item(String typeAndId, int amount)` | empty | Takes `TYPE:ID`. Empty for a bad format or an unknown id. (0.12.0) |

**`spawn` contract:**

- Returns `null` when MythicMobs is missing or the id is unknown. In that case it is safe to spawn a vanilla mob instead.
- Throws `IllegalStateException` when the MythicMobs API could not be reached before any spawn was attempted.
- Throws `MythicMobsBridge.UnknownOutcomeException` (a `RuntimeException`) when the spawn call itself failed. An
  entity may exist, so **do not retry**.

**`installPack` rules:** an existing file is replaced only if its first line still carries the same
`# managed-by:` marker. A file an admin has edited is left untouched.

Call everything on the **server thread**. There is nothing to register or clean up.

```java
Entity boss = null;
try { boss = MythicMobsBridge.spawn("OrcWarlord", loc, 20); }
catch (MythicMobsBridge.UnknownOutcomeException unknown) { return; } // never retry
if (boss == null) boss = loc.getWorld().spawnEntity(loc, EntityType.VINDICATOR);
```

### Resource pack

Core builds **one** server resource pack from the asset modules that plugins register, then publishes and sends it:

- It builds and hashes the pack (SHA-1) on a virtual thread.
- It publishes the pack through a built-in HTTP server or an external URL, both configured in Core's `resource-pack.yml`.
- It sends the pack on join and tracks a `PackState` per player.

Plugins never assign CustomModelData numbers by hand. Capability: `RESOURCE_PACK` (0.10.0).

#### ResourcePacks (plugin-facing)

| Method | Notes |
| --- | --- |
| `static void register(Plugin owner, String module, List<String> namespaces, Map<String, byte[]> files, int priority)` | Registers or replaces a module and schedules a rebuild on the next tick |
| `static PackState state(Player player)` | `NOT_SENT` if the player is unknown |
| `static boolean hasRequiredAssets(Player player, AssetRequirement requirement)` | `true` only if the player has `LOADED` the **current** hash and every required module is registered |
| `static CompletionStage<PackState> ensurePack(Player player)` | Sends the pack if needed. Completes with `LOADED`, `DECLINED` or `FAILED` (also `FAILED` on quit). Completes immediately if the current pack is already loaded. |
| `static AssetRegistry assets()` | The shared registry |
| `static Optional<String> hash()`, `static Optional<String> url()`, `static List<String> conflicts()` | Diagnostics: the published SHA-1, the download URL, and file paths overridden by several modules |
| `static void rebuildSoon()` | Coalesced rebuild. Only needed if you changed `assets()` directly. |

Core-internal members (not public API): `initialize()`, `shutdown()`, `ResourcePacks.Settings`, `PackBuilder`.

#### AssetRegistry and AssetRequirement

`AssetRegistry.register(Plugin owner, String moduleId, List<String> namespaces, Map<String, byte[]> files, int priority)` validates as follows:

- The module id is lower-cased.
- Namespaces must match `[a-z0-9_.-]{1,64}` and must not be `minecraft`. A namespace owned by another module throws
  `IllegalStateException`.
- File paths must match `[a-z0-9_./-]+`, must not contain `..`, and must start with `assets/<own namespace>/` or
  `assets/minecraft/` (overrides only). Anything else throws `IllegalArgumentException`.
- When several modules write the same path, the **higher `priority`** wins, and the path is listed in `conflicts()`.

Other members:

- `synchronized void unregisterAll(Plugin owner)`
- `synchronized List<AssetRegistry.Module> modules()`
- `synchronized boolean hasModule(String id)`
- `synchronized long revision()`
- `static NamespacedKey itemModel(String assetKey)`: parses and lower-cases an asset key for the `item_model` component,
  e.g. `myplugin:rune` (the item model definition `assets/myplugin/items/rune.json`). An invalid key throws
  `IllegalArgumentException`.
- `synchronized int customModelData(String assetKey)`: for legacy systems only. Returns a deterministic number in
  100000–9999999, with collisions resolved by probing.
- `synchronized Optional<String> assetForModelData(int value)`

`AssetRegistry.Module(String id, Plugin owner, List<String> namespaces, Map<String, byte[]> files, int priority)` is a record.

`AssetRequirement(Set<String> modules)` is a record. Factory: `static AssetRequirement of(String... modules)`.

`PackState` values:

- `NOT_SENT`, `DOWNLOADING`, `LOADED`, `FAILED`, `DECLINED`, `OUTDATED`
- `boolean usable()` returns `true` only for `LOADED`.

`PackStateChangeEvent` (Bukkit event, not cancellable, server thread) has `Player player()`, `PackState previous()`
and `PackState state()`.

**Threading and lifecycle**

- Call `register`, `state` and `ensurePack` on the **server thread**.
- Register modules in `onEnable`. When your plugin is disabled, Core removes its modules and rebuilds the pack.
- In `public-url` mode an admin must re-upload the zip after every asset change. Otherwise clients reject the hash and
  end in `FAILED`.

```java
// onEnable: ship a model and a texture under your own namespace
ResourcePacks.register(this, "myplugin", List.of("myplugin"), Map.of(
        "assets/myplugin/items/rune.json", runeItemJson,
        "assets/myplugin/textures/item/rune.png", runePng), 50);
meta.setItemModel(AssetRegistry.itemModel("myplugin:rune"));

// Before opening a GUI that needs the glyphs
if (!ResourcePacks.hasRequiredAssets(player, AssetRequirement.of("myplugin"))) {
    ResourcePacks.ensurePack(player).thenAccept(state -> Bukkit.getScheduler().runTask(this, () -> {
        if (state.usable() && player.isOnline()) openGui(player);
    }));
    return;
}
openGui(player);
```

### Curves & relative stats

This is shared balancing math. Curves are immutable, side-effect-free functions `x → y`. They are thread-safe to
evaluate. You store them under a global id, for example `curve.agility.slow.default`, and any plugin can read them by
that id. `RelativeStat` runs a ratio `source / target` through a curve and clamps the result. Capability: `CURVES` (0.5.0).

#### Curve (sealed interface)

`double apply(double x)`, `String describe()`, `static Curve fromMap(Map<?, ?> raw)`.

| Record | Formula | YAML `type` / keys (defaults) |
| --- | --- | --- |
| `Curve.Linear(double slope, double offset)` | `offset + slope·x` | `linear`: `slope` (1), `offset` (0). This is the type used when `type` is missing. |
| `Curve.Power(double scale, double exponent, double offset)` | `scale·max(0,x)^exponent + offset` | `power`: `scale` (1), `exponent` (1), `offset` (0) |
| `Curve.SoftCap(double max, double half)` | `max·x/(x+half)`, for x ≥ 0 | `softcap`: `max` (1), `half` (1, must be > 0) |
| `Curve.Logistic(double min, double max, double mid, double steepness)` | `min + (max-min)/(1+e^(-steepness·(x-mid)))` | `logistic`: `min` (0), `max` (1), `mid` (1), `steepness` (1) |
| `Curve.Logarithmic(double scale, double divisor)` | `scale·ln(1 + max(0,x)/divisor)` | `log` or `logarithmic`: `scale` (1), `divisor` (1, must be > 0) |
| `Curve.Points(double[] xs, double[] ys)` | Piecewise linear, constant outside the given points | `points`: `points: [[x,y], …]`. Needs strictly increasing x values and at least one point. |

`fromMap` throws `IllegalArgumentException` for an unknown type or bad numbers.

#### CurveRegistry (static, thread-safe)

| Method | Notes |
| --- | --- |
| `static void register(String id, Curve curve)` | Replaces any curve with the same id. A blank id or null curve throws `IllegalArgumentException`. |
| `static void unregister(String id)` / `static void unregisterPrefix(String prefix)` | |
| `static Optional<Curve> get(String id)` | |
| `static double apply(String id, double x, double fallback)` | Returns `fallback` when the curve is missing |
| `static Set<String> ids()` | Sorted copy |
| `static List<String> loadSection(String prefix, ConfigurationSection section)` | Registers each sub-section as the curve `prefix + key`. Bad entries are skipped and returned as messages. |

#### RelativeStat

- `static final double EPSILON = 1.0e-6`
- `static RelativeStat.Result resolve(RelativeStat.Input input)`
- `RelativeStat.Input(double source, double target, String curveId, double min, double max, double contextFactor)`,
  plus a 5-argument constructor with `contextFactor = 1.0`.
- `RelativeStat.Result(double ratio, double curveOutput, double finalValue, boolean capped, String breakdown)`

How `resolve` computes the result:

1. `ratio = max(0, source) / max(EPSILON, target)`.
2. The ratio runs through the curve. A missing curve counts as the identity.
3. The output is multiplied by `contextFactor` and clamped to `[min, max]`.
4. `breakdown` is a human-readable explanation of these steps, for debug commands.

**Lifecycle:** Core does **not** remove curves automatically, because curves have no owner. Use a plugin-specific
prefix, and call `CurveRegistry.unregisterPrefix(prefix)` both on reload and in `onDisable`.

```java
// config.yml:  curves: { crit: { type: softcap, max: 0.6, half: 2.0 } }
static final String PREFIX = "curve.myplugin.";
CurveRegistry.unregisterPrefix(PREFIX);
CurveRegistry.loadSection(PREFIX, getConfig().getConfigurationSection("curves"))
        .forEach(problem -> getLogger().warning("Curve " + problem));
double critChance = CurveRegistry.apply(PREFIX + "crit", agility / 10.0, 0.05);
var hit = RelativeStat.resolve(new RelativeStat.Input(attackerStr, targetStability, PREFIX + "stagger", 0, 1));
```

### Regions

`region.RegionService` is the shared region register of all Vystorm plugins. Each plugin publishes its regions (city
claims, arenas, markets, dungeons, faction territories, …). Any plugin can query them without depending on the plugin
that owns them. Gameplay rules such as rights and members stay in the owning plugin. Capability: `REGIONS` (0.4.0).

#### Region (record)

`Region(String kind, String id, String owner, int priority, RegionShape shape, Set<String> tags, Map<String, String> attributes, Map<String, Boolean> flags)`

- The key is `kind:id`. `kind` must match `[a-z0-9_-]{1,40}` and `id` must be non-blank and at most 128 characters.
  A violation throws `IllegalArgumentException`.
- Tags are lower-cased. The collections must not be null, so prefer the factory:
  `static Region of(String kind, String id, RegionShape shape)`. It sets the default priority for the kind and leaves
  the collections empty.
- Copy methods: `withPriority(int)`, `withOwner(String)`, `withTags(Set<String>)`, `withAttributes(Map<String, String>)`,
  `withFlags(Map<String, Boolean>)`.
- Read methods: `String key()`, `Optional<String> attribute(String)`, `Optional<Boolean> flag(String)`,
  `boolean hasTag(String)`.
- `owner` is filled with the registering plugin's name if you leave it empty.

#### RegionShape (sealed interface)

All bounds are inclusive. `world == null` means the shape applies in every world.

| Record | Notes |
| --- | --- |
| `RegionShape.Box(UUID world, int minX, int minY, int minZ, int maxX, int maxY, int maxZ)` | Corners are normalised |
| `RegionShape.Cylinder(UUID world, int centerX, int centerZ, int radius, int minY, int maxY)` | `radius >= 0` |
| `RegionShape.Chunks(UUID world, Set<Long> chunks, int minY, int maxY)` | `world` is required. Helpers: `static long pack(int chunkX, int chunkZ)`, `static int chunkX(long)`, `static int chunkZ(long)` |
| `RegionShape.Polygon(UUID world, List<double[]> points, int minY, int maxY)` | Points are `{x, z}`, at least 3 |

The interface methods are `world()`, `minY()`, `maxY()`, `minX()`, `minZ()`, `maxX()`, `maxZ()`,
`boolean containsColumn(int x, int z)` and `default boolean contains(UUID worldId, int x, int y, int z)`.

#### RegionPriority

On overlap, the highest priority wins. `static int forKind(String kind)` maps a kind to its default:

| Constant | Value | Kind |
| --- | --- | --- |
| `SPAWN` | 1000 | `spawn` |
| `SAFE` | 950 | `safe` |
| `ARENA_ZONE` | 820 | `arena-zone` |
| `ARENA` | 800 | `arena` |
| `MARKET` | 600 | `market` |
| `DUNGEON` | 500 | `dungeon` |
| `CITY` | 400 | `city` |
| `STRUCTURE` | 300 | `structure` |
| `TERRITORY` | 100 | `faction`, `territory` |
| `DEFAULT` | 0 | anything else |

#### RegionService

`static RegionService get()` returns the singleton.

| Method | Notes |
| --- | --- |
| `synchronized void register(Plugin owner, Region region)` | Registers or replaces the region with the same key |
| `synchronized boolean unregister(String kind, String id)` | |
| `synchronized int unregisterOwner(Plugin owner)` | Runs automatically on disable |
| `synchronized void sync(Plugin owner, String kind, Collection<Region> regions)` | Reconciles all of this owner's regions of one kind: removes missing ones, replaces changed ones, leaves the rest. A region of another kind throws `IllegalArgumentException`. |
| `List<Region> at(UUID world, int x, int y, int z)` / `List<Region> at(Location location)` | Highest priority first |
| `Optional<Region> top(Location location)` / `Optional<Region> top(Location location, String kind)` | |
| `boolean flag(Location location, String flag, boolean fallback)` | The highest-priority region that **sets** the flag decides |
| `List<String> keys(Location location)` | The `kind:id` keys at that location |
| `Optional<Region> get(String kind, String id)`, `List<Region> all()`, `List<Region> all(String kind)` | |
| `List<Region> intersecting(UUID world, int minX, int minZ, int maxX, int maxZ)` | Approximate column test, meant for placement and planning |
| `long revision()` | Increases on every change. Use it as a cache key. |

- **Threading:** register, unregister and sync on the **server thread**. Queries are lock-free and allowed from any thread.
- **Lifecycle:** Core removes a plugin's regions on disable.
- Regions covering more than 4096 chunks are not indexed. They are checked by bounding box instead, so they are
  slower but still correct.
- Legacy `api.area()` entries appear here automatically, with the kind taken from the area's NBT prefix.

#### Events and command

`RegionEnterEvent` and `RegionLeaveEvent` extend `PlayerEvent`. Each has `Region region()` and `getHandlerList()`.

- They fire on the server thread after a block-changing move, a teleport, or a join.
- They do **not** fire when regions are (un)registered around a player who is standing still, and there is no leave
  event on quit.

`/vregion at [player]` and `/vregion list [kind]` are admin views (permission `vystorm_core.admin`).

```java
// onEnable and after every data change: publish your territories
List<Region> regions = territories.stream().map(t -> Region.of("faction", t.id(),
        new RegionShape.Polygon(t.worldId(), t.points(), -64, 320))
        .withAttributes(Map.of("owner", t.ownerName()))
        .withFlags(Map.of("pvp", true))).toList();
RegionService.get().sync(this, "faction", regions);

@EventHandler public void onEnter(RegionEnterEvent e) {
    if (e.region().kind().equals("market")) e.getPlayer().sendActionBar(Component.text("Market"));
}
boolean pvp = RegionService.get().flag(victim.getLocation(), "pvp", true);
```

### NBT and inventory event helpers

These helpers come from the legacy `API` object (`CoreApi.api()`, capability `CORE_API` 0.11.0; the helpers
themselves are older and have no capability constant). Like all legacy handlers they use unguarded maps, so call them
**on the server thread only**.

#### api.nbt() – `NBTInterface`

String tags on items, entities, players and blocks:

- **Items and entities:** the tags live in the persistent data container, under Core's plugin namespace plus a
  configurable prefix. Keys are lower-cased.
- **Blocks:** the tags live in a Core-side YAML store keyed by position. Each write saves that file synchronously, so
  avoid bulk writes.
- Admins can switch items, entities or blocks off in Core's config. Calls for a disabled target become no-ops or
  return null.

| Method | Notes |
| --- | --- |
| `ItemStack setNBT(ItemStack itemStack, String key, String value)` | Mutates and returns the same stack. A `null` value removes the key. An item without meta (air) throws `IllegalArgumentException`. |
| `Entity setNBT(Entity entity, String key, String value)`, `void setNBT(Player player, String key, String value)`, `void setNBT(Block block, String key, String value)` | A `null` value removes the key |
| `String getNBT(ItemStack \| Block \| Entity \| Player, String key)` | `null` if the tag is absent |
| `String removeNBT(ItemStack \| Block \| Entity \| Player, String key)` | Returns the key, or `null` if it was absent |
| `HashMap<String, String> getAllNBT(ItemStack \| Block \| Entity \| Player)` | Only Core-owned tags, with the prefix removed from the keys |
| `ItemStack clearNBT(ItemStack)`, `Block clearNBT(Block)`, `Entity clearNBT(Entity)` | Removes only Core-owned tags (there is no player variant) |
| `default <P, C> C getTag(ItemStack item, NamespacedKey key, PersistentDataType<P, C> type)` | **Preferred for new code:** typed data under your own key |
| `default <P, C> ItemStack setTag(ItemStack item, NamespacedKey key, PersistentDataType<P, C> type, C value)` | A `null` value removes the key |

The same two typed methods also exist as statics on `PersistentDataHelper`:
`static <P, C> C get(ItemStack, NamespacedKey, PersistentDataType<P, C>)` and
`static <P, C> ItemStack set(ItemStack, NamespacedKey, PersistentDataType<P, C>, C)`.

**Legacy tag actions (not public API):**

- `NBTEntity getEntityNBT(String key)`, `NBTEntity setEntityNBT(NBTEntity entity, String key)` and
  `HashMap<String, NBTEntity> getEntityNBTHashMap()` bind an `NBTEntity` action to a tag key.
- `api.priority()` (`PriorityInterface`) is Core's dispatcher. Its listeners run those actions, in descending
  `priority()`, for block break and place, entity damage and death, interact, held-item change, item damage and harvest
  whenever the involved player, block or entity carries the key.
- There is no per-plugin cleanup and no unregister method. Use your own Bukkit listeners instead.

```java
NamespacedKey charges = new NamespacedKey(this, "charges");
api.nbt().setTag(wand, charges, PersistentDataType.INTEGER, 5);
Integer left = api.nbt().getTag(wand, charges, PersistentDataType.INTEGER);
api.nbt().setNBT(npc, "myplugin_role", "vendor");         // simple string tag on an entity
```

#### api.inventoryEvents() – `InventoryEventInterface`

Coalesced inventory change notifications, delivered after the fact:

- A subscription gets at most one callback per player or inventory per tick.
- The callback runs on the **next tick**, after the vanilla operation has completed.
- Every call must be made on the server thread. Otherwise it throws `IllegalStateException`. A disabled owner also
  throws `IllegalStateException`.

| Method | Purpose |
| --- | --- |
| `void watchPlayers(Plugin owner, Consumer<Player> callback)` | The player's own inventory changed |
| `void watchInventories(Plugin owner, Predicate<Inventory> accepts, Consumer<Inventory> callback)` | A container you accept changed |
| `void watchUses(Plugin owner, Consumer<InventoryUse> callback)` | Confirmed block placement or item consumption from the hotbar or offhand |
| `void changed(Player player)`, `void changed(Inventory inventory)` | Report a change that your plugin made itself |
| `void changed(Player player, BooleanSupplier valid)`, `void changed(Inventory inventory, BooleanSupplier valid)` | Same, but dropped if `valid` is false when the queue drains |
| `void observeUse(Player player, int slot, ItemStack original, BooleanSupplier valid)` | Emits one use observation for a custom use (for example a custom sapling) |
| `void removeAll(Plugin owner)` | Runs automatically on disable |

`InventoryUse` has four accessors:

- `Player player()`
- `int slot()`: 0–8 or 40 (offhand)
- `ItemStack item()`: a copy
- `boolean valid()`: re-checks the guards

An observation does **not** prove the item was consumed.

Not public API (vanilla-hook plumbing): `invalidateUses`, `forgetUsePlayer`, `discardUseIntent`, `captureUseIntent`,
`confirmBoneMealUse` and `clear()`. Probe for this API with `API.class.getMethod("inventoryEvents")` when you must
support very old Cores.

```java
api.inventoryEvents().watchPlayers(this, player -> hotbarCache.refresh(player));
```

#### api.inventoryInteractor() – `InteractorInterface` / `InteractorEntry`

Click, drag and drop handling for GUI items, routed by tag. Here is how the routing works:

1. You tag an item with a string key using `api.nbt().setNBT(item, key, value)` or
   `setNBTforItem(ItemStack, String name, String value)`.
2. You register an `InteractorEntry` whose `getNBT()` returns that key. Keys are lower-cased.
3. Core's listeners dispatch clicks, drags and drops on tagged items to the matching entries, highest `priority()` first.
4. An entry with `blocksOthers()` stops lower-priority entries. `canProcess(Player)` can skip an entry.

**Entries must cancel the event themselves.**

- `InteractorInterface` (plugin-facing): `void register(InteractorEntry)`, `void unregister(InteractorEntry)`,
  `InteractorEntry getByName(String)`, `InteractorEntry getNBTForMatch(ItemStack)`,
  `String getValueForItem(ItemStack, String key)`, `ItemStack setNBTforItem(ItemStack, String, String)`, and the typed
  `getTag`/`setTag` defaults.
  - Registering a key that is already taken is ignored and logged.
- `InteractorEntry` callbacks:
  - `String getNBT()`
  - `leftClick`, `rightClick`, `shiftLeftClick`, `shiftRightClick`, `middleClick`, each taking
    `(Player player, Event event, String value)`
  - `void dropClick(Player, Event)`
  - `void dragEventOld(InventoryDragEvent, InteractorEntry)`
  - `void dragEventNew(InventoryDragEvent)`
  - Defaults: `int priority()` returns 10, `boolean blocksOthers()` returns false, `boolean canProcess(Player)` returns true.
- **Lifecycle:** entries have no owner and are **not** removed automatically. Call `unregister(entry)` in `onDisable`.
- Core-dispatch methods (not for plugins): `handle(...)`, `handleEvent(...)`, `getNBTforItem`, `getAllNamespacedKeys`.

### Legacy handlers on API

`API` (`CoreApi.api()` returns `Optional<API>`) also exposes older subsystems. Unless noted otherwise, call them on the **server thread**.
Several of them check this and throw `IllegalStateException` off-thread: particle, packages, interaction, and world
create/unload.

| Accessor → type | Key methods | Cleanup |
| --- | --- | --- |
| `vault()` → `VaultInterface` | `boolean isEconomyAvailable()`, `String economyName()`, `double balance(OfflinePlayer)`, `boolean has(OfflinePlayer, double)`, `String format(double)`, `Transaction withdraw(OfflinePlayer, double)`, `Transaction deposit(OfflinePlayer, double)`, `int fractionalDigits()` (-1 = no fixed precision), `VaultInterface economySnapshot()`, `String economyIdentity()` | – |
| `luckperms()` → `LuckpermsInterface` | `boolean permissionChecker(String permission, Player player)`, `boolean permissionChecker(String permission, UUID playerUUID)` | – |
| `area()` → `AreaInterface` | `Area getArea()` (new, empty), `void register(Area)`, `boolean update(Area)`, `boolean unregister(Area)`, `List<Area> find(Location)`. `Area` has setters `setAreaType(AreaType)` (`fullCircle`, `Cylinder`, `Rectangle`), `setCenterX/Z`, `setRadius`, `setCorner1X/Z`, `setCorner2X/Z`, `setY1/Y2`, `setWorld(UUID)`, `setNBT(String)` and `boolean contains(Location)` | Manual `unregister` |
| `particle()` → `ParticleInterface` | `<T> void spawn(Particle, Location, int count, T data)`, `<T> void spawn(Particle, Location, int count, double offsetX, double offsetY, double offsetZ, double extra, T data)`, `<T> void spawn(Player viewer, Particle, Location, int count, T data)`, `<T> void line(Particle, Location from, Location to, int points, T data)`, `<T> void circle(Particle, Location center, double radius, int points, T data)`, `BukkitTask repeat(Plugin owner, Runnable effect, long periodTicks, int iterations)` | Tasks belong to `owner` |
| `structureGenerator()` → `StructureGeneratorInterface` | `TreeGenerator tree(StructureAccess, Random, TreeSpecies)`, `void register(Plugin, NamespacedKey, StructureGenerator)`, `boolean generate(NamespacedKey, StructureAccess, StructurePos, Random)`, `boolean unregister(Plugin, NamespacedKey)`, `Set<NamespacedKey> registeredNames()` | Automatic |
| `worldGenerator()` → `WorldGeneratorInterface` | `void register(String name, ChunkGenerator)`, `ChunkGenerator get(String)`, `boolean unregister(String)`, `ChunkGenerator voidGenerator()`, `ChunkGenerator flatGenerator(int bottomY, List<WorldGeneratorHandler.Layer> layers)`, `World create(WorldCreator, String generatorName)`, `boolean unload(World, boolean save)`, plus world-profile defaults (`detect`, `registerDetector`, …; see section 17, World environment) | Manual `unregister(name)`; detectors are cleaned up automatically |
| `interactionGenerator()` → `InteractionGeneratorInterface` | `Interaction spawn(Plugin owner, Location, float width, float height, Consumer<Player> rightClick, Consumer<Player> attack)`, `Interaction get(UUID)`, `boolean remove(UUID)` | Automatic |
| `packages()` → `PackageInterface` | `void register(Plugin owner, String channel, PluginMessageListener listener)` (a `null` listener registers an outgoing-only channel), `void send(Plugin owner, Player, String channel, byte[] payload)`, `boolean unregister(Plugin owner, String channel)` | Automatic |

**Notes on individual handlers**

- **Vault:** `withdraw` and `deposit` handle an unavailable economy differently from bad input and provider errors:

  | Situation | Result |
  | --- | --- |
  | No economy | `Transaction(false, 0, 0, "Vault economy is unavailable")` |
  | Negative or non-finite amount | `IllegalArgumentException` |
  | The provider throws | The exception is rethrown, and the outcome is **unknown** |

  - `economyName`, `balance`, `format`, `fractionalDigits`, `economySnapshot` and `economyIdentity` throw
    `IllegalStateException` when no economy is available. Check `isEconomyAvailable()` first.
  - `Transaction(boolean successful, double amount, double balance, String error)` is a record.
  - `economySnapshot()` binds a short synchronous sequence, such as a charge plus its refund, to the current provider.
    It is **not** atomic and does not survive a crash.
  - `economyIdentity()` is process-local. Use it to invalidate quotes, never as a persistent key.
- **LuckPerms:** despite the name, this uses Bukkit permissions, not the LuckPerms API.
  - The check is true for ops, for `*`, and for any `prefix.*` wildcard of the node.
  - The `UUID` variant returns `false` for offline players.
  - `permissionleve()` always returns 0 (not implemented) and `InventoryNametoPermission(String)` is internal. Neither
    is public API.
- **Area:** superseded by `RegionService`. Each area is mirrored there as a region, with its kind taken from the text
  before `:` in the area's NBT string. After you mutate a registered area, call `update(area)`.
  `compare(Player)` is `@Deprecated`.
- **Particle:** the default handler shares a budget of 4,096 particles per tick. Under load it reduces the count or
  resolution; nothing is queued for later ticks. `data` must match `Particle.getDataType()`, otherwise the call throws
  `IllegalArgumentException`.
- **Structure generator:** the one exception to the server-thread rule. The caller picks a thread that suits its
  `StructureAccess` implementation, for example a Paper `LimitedRegion` inside a generation callback.
- **World generator:** register a generator before the world that uses it is created or loaded.
- **Interaction generator:** `boolean handle(UUID, Player, boolean)` is Core's own dispatch hook and not public API.
- **Packages:** the payloads are untrusted client input. The channel is validated. Registering the same owner and
  channel twice throws `IllegalArgumentException`.
- **Cleanup:** "Automatic" in the table means Core's `releaseAdvancedResources(Plugin)`, which runs on plugin disable.
  `removeAll(Plugin)` and `clear()` on these handlers are Core-internal.

```java
// Charge a player and refund if the follow-up step fails (server thread)
VaultInterface vault = api.vault();
if (!vault.isEconomyAvailable()) return;
VaultInterface eco = vault.economySnapshot();          // bound to one provider for this sequence
VaultInterface.Transaction paid = eco.withdraw(player, 250.0);
if (!paid.successful()) { player.sendMessage("Payment failed: " + paid.error()); return; }
if (!deliver(player)) eco.deposit(player, 250.0);

if (api.luckperms().permissionChecker("myplugin.shop.admin", player)) openAdminShop(player);
api.particle().circle(Particle.FLAME, altar.getLocation().add(0.5, 1, 0.5), 1.5, 24, null);
```

**Advanced: replacing handlers.** Every legacy handler can be swapped out at runtime through `API.override*Handler(...)`
(for example `overrideVaultHandler(VaultInterface)`, `overrideParticleHandler(ParticleInterface)`,
`overrideAreaHandler(AreaInterface)`, `overrideNBT(NBTInterface)`, `overrideInventoryEventHandler(InventoryEventInterface)`,
`overrideInventoryInteractorHandler(InteractorInterface)`). The replacement is server-wide and affects every plugin.

- A custom `VaultInterface` whose provider can change must return a bound view from `economySnapshot()` and change
  `economyIdentity()` whenever the provider changes.
- A custom `ParticleInterface` must override the offset overload to support spread and speed.

Use these seams only for integration or tests, not for normal features.

---

## 17. Cross-plugin services

Provider/consumer registries that let Vystorm plugins cooperate without compile-time dependencies on each other.

### Registry conventions (read first)

All registries in this chapter follow the same provider/consumer pattern: one plugin **provides** an implementation and registers it with its own `Plugin` as owner; any number of plugins **consume** it through a static facade, without a compile or load dependency on the provider. Consumers only depend on Core.

| Rule | Detail |
|---|---|
| When to register | In `onEnable` (server thread). Every `register` takes the owning `Plugin`. |
| Cleanup | You do not have to unregister. `CoreLifecycle` listens for `PluginDisableEvent` (priority `MONITOR`) and removes the disabled plugin's registrations from every registry listed in [CoreLifecycle](#corelifecycle-and-providerfailures). Exception: threat modifiers (see [ThreatServices](#threatservices)). |
| Disabled owners | Most read paths skip providers whose owner `isEnabled()` is false, even before cleanup has run. |
| Single-source registries | `MobServices`, `PartyServices`, `WorldEventServices`, `CharacterServices`, `DisciplineServices`: exactly one provider. A second `register` from another plugin **replaces** the first (last writer wins, a warning is logged; load order decides). |
| Multi-provider registries, keyed by id | `RuntimeContextHub`, `RuntimeActionHub`: an id that already exists is replaced (warning logged). `RelationServices`, `MobContextServices`, `MobExtensions`, `CombatRegistry`: an id that another plugin already owns throws `IllegalStateException`; the same owner may re-register (replace). |
| Provider failures | Exceptions thrown by providers are caught. Core falls back to the documented neutral value and logs through `ProviderFailures`, at most once per minute per provider and operation. A broken provider never breaks the consumer. |
| Missing provider | Facades return neutral values (`Optional.empty()`, empty list, `1.0`, level 1, …) and never throw because a provider is missing. |

Feature detection (see section 1.4):

```java
boolean ok;
try { ok = CoreApi.has(CoreApi.Capability.MOB_SERVICES); } catch (LinkageError tooOld) { ok = false; }
```

| Area | Capability constant | Since |
|---|---|---|
| `runtime.RuntimeContextHub`, `runtime.RuntimeActionHub` | `RUNTIME_HUBS` | 0.2.0 |
| `mob.*` (services, relations, context, events, extensions) | `MOB_SERVICES` | 0.9.0 |
| `mob.MobRefs` | `MOB_REFS` | 0.12.0 |
| `MobExtensions.sink(Plugin, TriggerSink)`, `hasSink()` | `TRIGGER_SINK_OWNER` | 0.10.3 |
| `MobService.spawnRules()`, `MobSpawnRule` | none: use `CoreApi.atLeast("0.15.0")` | 0.15.0 |
| `threat.*` | `THREAT` | 0.2.0 |
| `combat.CombatRegistry` | `COMBAT_REGISTRY` | 0.2.0 |
| `party.*` | `PARTY` | 0.6.0 |
| `worldevent.*` | `WORLD_EVENTS` | 0.3.0 |
| `spawn.*` | `SPAWN_POINTS` | 0.13.0 |
| `progression.CharacterServices` | `CHARACTER` | 0.16.0 |
| `progression.DisciplineServices` | `DISCIPLINES` | 0.10.0 |
| `world.WorldEnvironmentRegistry`, player activity | none (older than `CoreApi`; present whenever `CORE_API` is) | <0.11.0 |
| `storage.VersionedStore` | `VERSIONED_STORE` (the `CorruptStoreException`/quarantine constructor needs `atLeast("0.11.1")`) | 0.3.0 |
| `config.ConfigMigrator` | `CONFIG_MIGRATOR` | 0.3.0 |
| `schedule.TaskGroup` | `TASK_GROUPS` | 0.3.0 |
| `lifecycle.CoreLifecycle` (automatic cleanup) | `LIFECYCLE_CLEANUP` | 0.10.3 |
| `lifecycle.ProviderFailures` | `CORE_API` | 0.11.0 |

---

### Runtime hubs (`runtime`)

Two singleton buses for loosely coupled data exchange. Payloads use only neutral types (`String`, `double`, `UUID`, sets and maps), so plugins can exchange data without depending on each other's classes. Access them with `RuntimeContextHub.get()` and `RuntimeActionHub.get()`.

#### RuntimeContextHub

**Purpose.** Providers publish read-only *snapshots* of their state (spawn pressure, city/faction state, world events, player activity, world generation). Consumers query one `Channel` at one position and either get every snapshot or a merged view.

`enum Channel { WORLD_EVENT, SPAWN, STRUCTURE, FACTION, CITY, PLAYER, WORLD_GENERATION, INFRASTRUCTURE, DUNGEON, CUSTOM }`

| Type / method | Notes |
|---|---|
| `record Query(UUID worldId, int blockX, int blockY, int blockZ, Optional<UUID> playerId, Set<String> tags, Map<String,String> attributes)` | `static Query world(UUID worldId, int x, int y, int z)`, `Query withPlayer(UUID player)`. |
| `record Snapshot(String providerId, Channel channel, int priority, long revision, Set<String> tags, Map<String,String> values, Map<String,Double> scores)` | `static Snapshot empty(String providerId, Channel channel, int priority, long revision)`. A blank `providerId` or a non-finite score throws `IllegalArgumentException`. |
| `record MergedSnapshot(Channel channel, long revision, Set<String> tags, Map<String,String> values, Map<String,Double> scores, List<String> providers)` | Merge result. |
| `interface Provider` | `String id()`, `Set<Channel> channels()`, `default int priority()` (0), `default long revision()` (0), `default boolean supports(Channel channel, Query query)`, `Snapshot snapshot(Channel channel, Query query)`. |
| `Registration register(Plugin owner, Provider provider)` | Blank id throws `IllegalArgumentException`. The same id (case-insensitive) replaces the earlier provider. |
| `boolean unregister(UUID registrationId)`, `boolean unregisterProvider(String providerId)`, `int unregisterOwner(Plugin owner)` | Manual removal (normally not needed). |
| `List<Snapshot> query(Channel channel, Query query)` | All matching snapshots, ordered by priority (descending) and then by registration order. |
| `MergedSnapshot merged(Channel channel, Query query)` | Tags are unioned. For `values`, the higher-priority provider wins per key. `scores` are **added up**. `revision` is a hash of the providers' revisions. |
| `Collection<Registration> registrations()` | Diagnostics. |

- **Provider contract:** fast, free of side effects, and **must never load chunks**. Returning `null` means "no contribution". If a provider throws, or returns a snapshot for a different channel, it is skipped and a warning goes to its owner's logger.
- **Threading:** `query` and `merged` run the providers synchronously on the caller's thread. Core does not switch threads, so a provider that touches Bukkit state must only be queried from the server thread, or must read only thread-safe caches.
- Merged data is meant as input for directors and planners. Do not use it as an authoritative source for persistence.
- Core registers two providers itself: `core-player-activity` (channel `PLAYER`) and `core-world-environment` (channel `WORLD_GENERATION`). Both are described below.

```java
// Provider (onEnable)
RuntimeContextHub.get().register(this, new RuntimeContextHub.Provider() {
    public String id() { return "myplugin-arenas"; }
    public Set<RuntimeContextHub.Channel> channels() { return Set.of(RuntimeContextHub.Channel.CUSTOM); }
    public RuntimeContextHub.Snapshot snapshot(RuntimeContextHub.Channel ch, RuntimeContextHub.Query q) {
        String arena = arenaIndex.at(q.worldId(), q.blockX(), q.blockZ()); // own in-memory index, no chunk access
        if (arena == null) return null;
        return new RuntimeContextHub.Snapshot(id(), ch, priority(), 1L, Set.of("ARENA"),
                Map.of("arenaId", arena), Map.of("pressure", 0.4));
    }
});
// Consumer
RuntimeContextHub.MergedSnapshot m = RuntimeContextHub.get().merged(RuntimeContextHub.Channel.CUSTOM,
        RuntimeContextHub.Query.world(loc.getWorld().getUID(), loc.getBlockX(), loc.getBlockY(), loc.getBlockZ()));
double pressure = m.scores().getOrDefault("pressure", 0.0);
```

#### RuntimeActionHub

**Purpose.** A command router: a plugin *asks* another plugin to do something by a semantic action id (for example `player.activity.signal`), without depending on its API. Handlers answer asynchronously with a `Response`.

| Type / method | Notes |
|---|---|
| `record Request(UUID requestId, String action, UUID worldId, int blockX, int blockY, int blockZ, Optional<UUID> playerId, Set<String> tags, Map<String,String> values, Map<String,Double> scores)` | `action` is normalized: trimmed, lower-cased, spaces become `.`. A blank action or a non-finite score throws `IllegalArgumentException`. Shortcut: `static Request world(String action, UUID worldId, int x, int y, int z)`. |
| `record Response(String handlerId, boolean accepted, String status, Set<String> tags, Map<String,String> values, Map<String,Double> scores)` | `static Response accepted(String handlerId, Map<String,String> values)`, `static Response declined(String handlerId, String status)`. |
| `interface Handler` | `String id()`, `Set<String> actions()` (list **lower-case** ids), `default int priority()` (0), `default boolean supports(Request request)`, `CompletionStage<Response> handle(Request request)`. |
| `Registration register(Plugin owner, Handler handler)` | The same handler id replaces the earlier handler (warning logged). |
| `boolean unregister(UUID registrationId)`, `boolean unregisterHandler(String handlerId)`, `int unregisterOwner(Plugin owner)` | Manual removal. |
| `CompletionStage<Response> dispatch(Request request)` | Sends the request to the **first** enabled handler (highest priority, then earliest registration) whose `actions()` contains the id and whose `supports()` returns true. If that handler declines, the next handler is **not** tried. |
| `CompletionStage<List<Response>> broadcast(Request request)` | Sends the request to every matching handler and completes once all of them have answered. |
| `Collection<Registration> registrations()` | Diagnostics. |

- **Threading:** `dispatch` and `broadcast` may be called from any thread. Off the server thread, Core moves handler selection and `handle(...)` to the next server tick, so **handlers are always invoked on the server thread**. Long-running work should return a stage that completes later. Handlers must not load chunks implicitly. Never block the server thread on the returned stage (`join()`/`get()`). If the continuation touches Bukkit, schedule it back onto the server thread.
- **Errors are never exceptional:** the stage completes with `accepted=false` and one of these statuses: `no-handler` (handlerId `core`), `null-stage`, or `failed:<ExceptionSimpleName>`. Off-thread calls while Core is disabled complete exceptionally.
- Action ids used by other Vystorm plugins (for example `structure.*`, `spawn.context`, `city.*`) are contracts owned by those plugins, not by Core. Core itself only handles the two `player.activity.*` actions below.

```java
RuntimeActionHub.get().register(this, new RuntimeActionHub.Handler() {
    public String id() { return "myplugin-rewards"; }
    public Set<String> actions() { return Set.of("myplugin.reward.grant"); }
    public CompletionStage<RuntimeActionHub.Response> handle(RuntimeActionHub.Request r) {
        if (r.playerId().isEmpty()) return CompletableFuture.completedFuture(RuntimeActionHub.Response.declined(id(), "missing:playerId"));
        rewards.grant(r.playerId().get(), r.values().getOrDefault("tier", "1"));
        return CompletableFuture.completedFuture(RuntimeActionHub.Response.accepted(id(), Map.of("granted", "true")));
    }
});
```

#### Player activity (Core provider)

Core tracks a lightweight, event-driven activity state per online player (mining, combat, trading, …). `player.PlayerActivityService` is public, but Core owns the only live instance and **no static accessor exists**, so do not construct your own instance. Treat the class as "not public API" and use it only through the hubs:

- **Read:** `RuntimeContextHub.get().query(Channel.PLAYER, Query.world(...).withPlayer(uuid))`. The query must carry a `playerId`. Provider id `core-player-activity`. Tags: `ACTIVITY:<ACTIVITY>` plus active flags (upper-case). Values: `activity`, `activitySince` (epoch ms).
- **Write/read via actions** (handler `core-player-activity-actions`, priority 1000; the request needs `playerId`):
  - `player.activity.signal` takes values `activity` (a `PlayerActivityService.Activity` name, default `IDLE`), `holdMillis` (default `10000`; 0 means no expiry) and `flags` (`|`-separated).
  - `player.activity.context` reads the current state.
  - Both respond with values `activity`, `since`, `expiresAt` and tags `ACTIVITY:<X>` plus flags. Invalid input gives `declined` with status `invalid:<message>` or `missing:playerId`.
- Activities: `IDLE, TRAVELLING, EXPLORING, MINING, FARMING, BUILDING, CRAFTING, TRADING, FISHING, CITY, MARKET, HOME, DUNGEON_EXPLORING, DUNGEON_COMBAT, DUNGEON_BOSS, ARENA, PVP, COMBAT, LOOTING, ESCAPING, RESTING`. An expired state reads as `IDLE`. The state is dropped on quit.

```java
var req = new RuntimeActionHub.Request(UUID.randomUUID(), "player.activity.signal", p.getWorld().getUID(),
        p.getLocation().getBlockX(), p.getLocation().getBlockY(), p.getLocation().getBlockZ(),
        Optional.of(p.getUniqueId()), Set.of(), Map.of("activity", "DUNGEON_BOSS", "holdMillis", "30000", "flags", "BOSS_ROOM"), Map.of());
RuntimeActionHub.get().dispatch(req);
```

#### World environment (`world.WorldEnvironmentRegistry`)

**Purpose.** A single semantic profile per world (dimension, terrain kinds, tags), so that every plugin classifies custom worlds the same way. Custom world generators provide a `Detector`. Consumers call `detect(world)`.

| Type / method | Notes |
|---|---|
| `enum DimensionKind { OVERWORLD, NETHER, END, AETHER, CUSTOM }`, `enum TerrainKind { GROUND, UNDERGROUND, FLOATING, CEILING, VOID, MIXED }` | |
| `record WorldProfile(String id, DimensionKind dimension, Set<TerrainKind> terrain, Set<String> tags, String generatorId, String generatorClass, boolean customGeneration, String providerId, int priority, Map<String,String> attributes)` | `boolean hasTerrain(TerrainKind kind)`, `boolean hasTag(String tag)` (case-insensitive). A blank id throws `IllegalArgumentException`. |
| `interface Detector` | `String id()`, `default int priority()`, `boolean supports(World world)`, `WorldProfile detect(World world)`. |
| `static WorldEnvironmentRegistry get()` | Singleton. |
| `Registration register(Plugin owner, Detector detector)` | The same owner with the same id replaces the earlier detector. |
| `boolean unregister(UUID registrationId)`, `int unregisterOwner(Plugin owner)` | |
| `WorldProfile detect(World world)` | Resolution order: explicit profile, then the first detector (by priority descending) that supports the world and returns non-null, then the built-in fallback. Never null. |
| `void setExplicitProfile(Plugin owner, World world, WorldProfile profile)`, `void clearExplicitProfile(World world)` | Forces the profile for one world. |
| `Optional<String> registeredGeneratorId(World world)` | The id of the generator, if it was registered through Core's generator registry. |

- **Fallback profile:** id `vanilla:<dimension>` or `custom:<generatorId or generator class>`. Tags: the dimension name, plus `CUSTOM_GENERATION` and `CORE_GENERATOR` where they apply. Terrain: `GROUND`. Provider `core-fallback`.
- **Threading:** callable from any thread. Detectors must only read world metadata (name, environment, generator). They must be free of side effects and must **never load chunks or touch entities**. A detector that throws is skipped with a warning.
- The profile is also published on `RuntimeContextHub` channel `WORLD_GENERATION` (provider `core-world-environment`). Tags: `DIMENSION:<X>`, `TERRAIN:<X>` plus the profile tags. Values: `profile`, `dimension`, `generatorId`, `generatorClass`, `customGeneration`.
- `generatorRegistered(...)` and `generatorUnregistered(...)` are called by Core's generator registry. They are **not public API**.

```java
WorldEnvironmentRegistry.WorldProfile wp = WorldEnvironmentRegistry.get().detect(player.getWorld());
if (wp.dimension() == WorldEnvironmentRegistry.DimensionKind.NETHER || wp.hasTerrain(WorldEnvironmentRegistry.TerrainKind.CEILING)) { /* no sky spawns */ }
```

---

### Mob platform (`mob`, `mob.event`, `mob.extension`)

**Purpose.** A neutral mob API: spawn, find and control managed mobs, decide who is hostile to whom, and add context such as arenas or sieges. The only provider is **Vystorm-Mobs**. SpawnDirector, Factions, Arena, Cities, dungeons and other plugins are consumers. Without the provider, `MobServices.service()` is empty and consumers fall back to vanilla or legacy behaviour.

#### MobServices and MobService

| `MobServices` (facade) | |
|---|---|
| `static Optional<MobService> service()` | Empty when there is no provider. |
| `static boolean available()` | |
| `static void register(Plugin owner, MobService service)` | **Provider only** (Vystorm-Mobs). |
| `static void unregister(Plugin owner)` | |
| `static void fire(Event event)` | **Provider only.** Fires the event synchronously on the server thread and defers it to the next tick from other threads. Cancellable events must be fired on the server thread. |

| `MobService` (implemented by the provider) | |
|---|---|
| `Optional<MobHandle> spawn(MobSpawnRequest request)` | Throws `IllegalArgumentException` for an unknown definition. Empty if the spawn was cancelled (for example through `MobSpawnRequestEvent`). |
| `Optional<MobGroupHandle> spawnGroup(MobGroupRequest request)` | |
| `Optional<MobHandle> getMob(UUID mobId)` / `Optional<MobHandle> byEntity(Entity entity)` | |
| `Collection<MobHandle> getMobs(MobQuery query)` | Live mobs. |
| `Optional<MobDefinitionView> getDefinition(String id)` / `Collection<MobDefinitionView> getDefinitions(MobQuery query)` | Definitions. |
| `ThreatQuote quoteThreat(MobSpawnRequest request)` / `Optional<MobPreview> preview(MobSpawnRequest request)` | Plan without spawning. |
| `void despawn(UUID mobId, DespawnReason reason)` | |
| `Optional<EntityRef> describe(Entity entity)` | Empty for entities the provider does not manage. |
| `Optional<MobGroupHandle> group(UUID groupId)` | |
| `default Collection<MobSpawnRule> spawnRules()` / `default long spawnRulesVersion()` | Natural spawn rules (since 0.15.0). Rebuild your catalog when the version changes. |

**Threading:** spawn methods and all mutating `MobHandle` methods run **on the server thread only**.

#### Requests, handles and value types

- `MobSpawnRequest` (record). Build it with `static Builder of(String definitionId)` and the chainable methods `at(Location)`, `level(int)` (`<= 0` means the definition decides), `tier(MobTier)` (`null` means the definition's tier), `affix(String)`, `role(String)`, `squad(UUID)`, `source(String)`, `team(String)`, `persistent(boolean)`, `objective(Objective)` and `attribute(String, String)`, then call `build()`. Also available: `Builder toBuilder()`. A blank `definitionId` throws `IllegalArgumentException`.
- `MobHandle` (interface). Read methods: `UUID id()` (the mob id, which is **not necessarily** the entity UUID and survives persistence promotion), `definitionId()`, `Optional<LivingEntity> entity()`, `valid()`, `level()`, `tier()`, `affixes()`, `role()`, `squadId()`, `team()`, `currentTarget()`, `threat()`, `ref()`, `aiTier()` (0 dormant to 3 combat). Mutating methods: `setTarget(UUID)`, `addObjective(Objective)`, `clearObjective(String)`, `pushContext(ContextLayer)`, `removeContext(String)`, `persistent()`, `UUID promoteToPersistent()`, `despawn(DespawnReason)`.
- `MobGroupRequest(List<MobSpawnRequest> members, String formation, int leaderIndex, List<Objective> objectives)` throws `IllegalArgumentException` for an empty member list. `formation` defaults to `LOOSE`. `leaderIndex < 0` means the provider picks the leader.
- `MobGroupHandle`: `groupId()`, `members()`, `leader()`, `void order(String order, Objective objective)` (orders such as `FOCUS_TARGET`, `SPREAD_OUT`, `RETREAT`, `FLANK`, `DEFEND_POSITION`, `PUSH`, `HOLD`, `BREACH`), `double morale()`.
- `MobQuery`: `MobQuery.builder().faction("ORC").role("RANGED").environment("FOREST").threat(20, 35).tiers(MobTier.NORMAL, MobTier.ELITE).near(worldId, x, y, z, r).limit(n).build()`. Also `id(String)`, `tag(String)` (all of) and `anyTag(String)`. `MobQuery.ALL` matches everything. Empty fields do not restrict the query.
- Enums: `MobTier { NORMAL, VETERAN, ELITE, CHAMPION, RARE, BOSS, WORLD_BOSS }` with `static MobTier parse(String value, MobTier fallback)`. `DespawnReason { DIRECTOR_RECYCLE, CONTEXT_ENDED, LEASH, DEATH, ADMIN, PLUGIN_DISABLE, CHUNK_UNLOAD, RELOAD, OTHER }`. `ObjectiveType { ATTACK, DEFEND, CAPTURE, ESCORT, HOLD, DESTROY, PATROL, SURVIVE, RETREAT }`.
- `Objective` (record). Factories: `static Objective at(String id, ObjectiveType type, UUID worldId, Vector point, double radius, int priority)` and `static Objective entity(String id, ObjectiveType type, UUID target, int priority)`. Also `withExpiry(long expiresAtMillis)` (`<= 0` means unlimited) and `expired(long nowMillis)`.
- Read-only records: `MobDefinitionView` (id, displayName, entityType, faction, archetype, tier, baseThreat, tags, roles, environments, abilities, traits, boss, …), `MobPreview`, `ThreatQuote(String definitionId, double total, Map<String,Double> breakdown)` (with `static ThreatQuote unknown(String)`), and `MobSpawnRule` (biomes as namespaced keys, dimensions `NORMAL|NETHER|THE_END`, `weight` with rarity already applied).

```java
Optional<MobService> mobs = MobServices.service();
if (mobs.isPresent() && mobs.get().getDefinition("ORC_GRUNT").isPresent()) {
    MobSpawnRequest req = MobSpawnRequest.of("ORC_GRUNT").at(loc).level(12).tier(MobTier.ELITE).source(getName())
            .objective(Objective.at("hold-gate", ObjectiveType.DEFEND, loc.getWorld().getUID(), loc.toVector(), 8, 10))
            .build();
    mobs.get().spawn(req).ifPresent(handle -> tracked.add(handle.id()));
} else {
    loc.getWorld().spawnEntity(loc, EntityType.ZOMBIE); // fallback without Vystorm-Mobs
}
```

#### MobRefs (Vystorm first, MythicMobs as fallback)

Resolves config references such as `ID`, `vystorm:ID` (Vystorm-Mobs only) and `mythic:ID`/`mythicmobs:ID`. The order is always Vystorm-Mobs first, then MythicMobs. **Server thread only.**

| Method | |
|---|---|
| `static String id(String ref)` | Strips a known prefix. |
| `static boolean vystormKnows(String ref)` | |
| `static Optional<MobRefs.Source> source(String ref)` | `Source { VYSTORM, MYTHIC_MOBS }`. Empty if no source knows the reference. |
| `static Optional<Entity> spawn(String ref, Location location, int level, String sourcePlugin)` | Empty means that no source knows the reference or the spawn was declined before any mob existed, so a vanilla replacement is safe. Throws `MobRefs.UnknownOutcomeException` (a `RuntimeException`) when a spawn was triggered but its outcome is unclear. In that case **do not retry and do not replace it with vanilla**. |

#### Relations

**Purpose.** One place that decides who may attack or assist whom (arena teams, factions, cities, pets, parties). Plugins provide `RelationProvider`s, and mob AI and other consumers ask `RelationServices.get()`.

| Type / method | |
|---|---|
| `enum Relation { ALLY, FRIENDLY, NEUTRAL, UNWELCOME, HOSTILE, INVADER }` | `boolean attackable()` (`HOSTILE`/`INVADER`), `boolean assistable()` (`ALLY`/`FRIENDLY`). |
| `record EntityRef(UUID id, Kind kind, String faction, String team, UUID owner, Set<String> tags)` | `Kind { PLAYER, MOB, NPC, PET, VANILLA_HOSTILE, VANILLA_PASSIVE, OTHER }`. Factories: `static EntityRef player(UUID)`, `static EntityRef mob(UUID id, String faction, String team, UUID owner, Set<String> tags)`, `static EntityRef of(Entity entity)` (server thread; resolves managed mobs through `MobService.describe`). Also `boolean hasTag(String)`. |
| `interface RelationProvider` | `String id()`, `int priority()`, `Optional<Relation> relation(EntityRef source, EntityRef target)`. Return empty for "no opinion". |
| `interface RelationService` | `Relation getRelation(EntityRef source, EntityRef target)`, `default boolean canAttack(EntityRef, EntityRef)`, `default boolean canAssist(EntityRef, EntityRef)`. |
| `RelationServices` | `static RelationService get()`, `static void register(Plugin owner, RelationProvider provider)` (throws `IllegalStateException` if another plugin owns the id), `static void unregisterAll(Plugin owner)`, `static List<String> providers()`, `static Relation fallback(EntityRef source, EntityRef target)`. |

- **Resolution:** the same id gives `ALLY`. Otherwise the providers are asked by priority, descending, and the first non-empty answer wins. If no provider answers, the fallback applies: same team gives `ALLY`; different explicit teams give `HOSTILE`; an owner/pet link or a shared owner gives `ALLY`; two players give `ALLY` if they are in the same party (`PartyServices`), else `NEUTRAL`; the same faction gives `ALLY`; a hostile mob against a player, pet or NPC gives `HOSTILE`; everything else gives `NEUTRAL`. Hostility between mob factions is never assumed; a provider must declare it.
- **Provider contract:** fast, with no world access. Thread: not constrained by Core. Callers in the mob system are on the server thread.

```java
RelationServices.register(this, new RelationProvider() {
    public String id() { return "myplugin-arena-teams"; }
    public int priority() { return 500; }
    public Optional<Relation> relation(EntityRef s, EntityRef t) {
        String a = teams.get(s.id()), b = teams.get(t.id());
        if (a == null || b == null) return Optional.empty();
        return Optional.of(a.equals(b) ? Relation.ALLY : Relation.HOSTILE);
    }
});
boolean may = RelationServices.get().canAttack(EntityRef.of(attacker), EntityRef.of(victim));
```

#### Mob context

**Purpose.** Plugins describe *where* a mob is (arena match, city, siege, event, dungeon) as `ContextLayer`s. The mob system combines them into combat and AI rules. Context providers never control the mob AI directly.

| Type / method | |
|---|---|
| `interface ContextProvider` | `String id()`, `List<ContextLayer> layers(MobContextQuery query)`. |
| `MobContextServices` | `static void register(Plugin owner, ContextProvider provider)` (throws `IllegalStateException` if another plugin owns the id), `static void unregisterAll(Plugin owner)`, `static List<String> providers()`, `static MobContext resolve(MobContextQuery query)` (server thread; returns `MobContext.EMPTY` if there are no layers). |
| `record MobContextQuery(UUID worldId, int x, int y, int z, UUID mobId, String definitionId, String faction, Set<String> tags)` | `mobId` may be null for previews. |
| `record ContextLayer(ContextKind kind, String id, String owner, int priority, Set<String> tags, Map<String,String> attributes, CombatRules combatRules, AiRules aiRules, List<Objective> objectives)` | `combatRules` and `aiRules` may be `null` ("no constraint"). Well-known attribute keys: `team`, `matchId`, `gameMode`, `cityId`, `owner`, `faction`, `siegeState`, `protectionState`, `disposition`. |
| `enum ContextKind { WORLD, REGION, ARENA, CITY, FACTION, EVENT, DUNGEON }` | |
| `record CombatRules(boolean damagePlayers, boolean damageMobs, boolean damageNpcs, boolean damagePets, boolean friendlyFire, boolean mobBlockDamage, Set<String> allowedBlockTargets, double damageMultiplier)` | Constants `OPEN_WORLD`, `ARENA`, `PROTECTED`, and `static CombatRules siege(Set<String> targets)`. Layers are combined **restrictively** (`restrict`). |
| `record AiRules(double aggressionMultiplier, boolean allowFlee, boolean allowPursuitOutsideRegion, double leashRadius, boolean passiveUnlessProvoked)` | `DEFAULT`. The highest-priority layer that sets `AiRules` wins. |
| `MobContext` | `layer(ContextKind)`, `has(ContextKind)`, `layers()`, `combatRules()`, `aiRules()`, `objectives()`, `tags()`, `arena()`, `city()`, `event()`, `team()`. |

**Provider contract:** called on the server thread. It must be fast and must not load chunks. The provider caches results and refreshes them periodically. A provider that throws is skipped.

```java
MobContextServices.register(this, new ContextProvider() {
    public String id() { return "myplugin-arena"; }
    public List<ContextLayer> layers(MobContextQuery q) {
        Match m = matches.at(q.worldId(), q.x(), q.z());
        if (m == null) return List.of();
        return List.of(new ContextLayer(ContextKind.ARENA, "arena:" + m.id(), getName(), 100, Set.of(),
                Map.of("matchId", m.id()), CombatRules.ARENA, null, List.of()));
    }
});
```

#### Mob events (`mob.event`)

These are standard Bukkit events, fired by the provider. Each has `getHandlerList()`. Unless noted otherwise they fire on the server thread.

| Event | Cancellable | Accessors |
|---|---|---|
| `MobSpawnRequestEvent` | yes | `request()`, `request(MobSpawnRequest replacement)`: replace the request before the spawn. |
| `MobSpawnedEvent` | no | `mob()`, `request()` |
| `MobDeathEvent` | no | `mob()`, `killer()` (a `UUID`, may be null) |
| `MobDespawnedEvent` | no | `mob()`, `reason()`: removed without dying. |
| `MobTargetChangedEvent` | yes | `mob()`, `previousTarget()`, `newTarget()` (null means no target) |
| `MobAbilityCastEvent` | yes | `mob()`, `abilityId()`, `targets()` |
| `MobDamageEvent` | only in `Stage.PRE` | `stage()`, `source()`, `target()`, `damageType()`, `abilityId()`, `heal()`, `amount()`, `amount(double)` (PRE only; POST throws `IllegalStateException`) |
| `MobSquadOrderEvent` | yes | `group()`, `order()`, `objective()` |
| `MobObjectiveChangedEvent` | no | `mob()`, `objective()`, `added()` |
| `MobPromotedEvent` | no | `mob()`, `persistentId()` |
| `MobBossEvent` | no | `boss()`, `stage()` (`START`, `PHASE`, `ENRAGE`, `RESET`, `KILL`, `WIPE`), `phase()` |
| `MobRegisteredEvent` | no | `definition()` |
| `MobDecisionTelemetryEvent` | no, **asynchronous** | `mobId()`, `definitionId()`, `action()`, `score()`, `scores()`. Read only; no world access. |

```java
@EventHandler public void onMobDeath(MobDeathEvent e) {
    if (e.killer() != null && e.mob().tier() == MobTier.BOSS) bounties.pay(e.killer(), e.mob().definitionId());
}
```

#### Mob extensions (`mob.extension`)

**Purpose.** Content plugins add skill building blocks to Vystorm-Mobs (mechanics, targeters, conditions, triggers, brain profiles, traits, effects) without depending on its runtime classes. Vystorm-Mobs looks them up by id. Ids are case-insensitive, and **built-in names of Vystorm-Mobs take precedence**, so give yours a plugin prefix.

| Register (content plugin) | Lookup (used by Vystorm-Mobs) |
|---|---|
| `static void registerMechanic(Plugin owner, String id, MobMechanic.Factory factory)` | `static Optional<MobMechanic.Factory> mechanic(String id)` |
| `static void registerTargeter(Plugin owner, String id, MobTargeter.Factory factory)` | `static Optional<MobTargeter.Factory> targeter(String id)` |
| `static void registerCondition(Plugin owner, String id, MobCondition.Factory factory)` | `static Optional<MobCondition.Factory> condition(String id)` |
| `static void registerTrigger(Plugin owner, String id)` | `static boolean triggerKnown(String id)` |
| `static void registerBrain(Plugin owner, MobBrainProfile profile)` | `static Optional<MobBrainProfile> brain(String id)` |
| `static void registerTrait(Plugin owner, MobTrait trait)` | `static Optional<MobTrait> trait(String id)` |
| `static void registerEffect(Plugin owner, MobEffect effect)` | `static Optional<MobEffect> effect(String id)` |
| `static Set<String> ids(MobExtensions.Kind kind)` | `Kind { MECHANIC, TARGETER, CONDITION, TRIGGER, BRAIN, TRAIT, EFFECT }` |

- `register*` throws `IllegalStateException` if another plugin owns the id, and `IllegalArgumentException` for a blank id. Lookups return empty while the owner is disabled.
- Functional contracts:
  - `MobMechanic.apply(SkillCall call, SkillTarget target)` returns `boolean` (false means "not applied").
  - `MobTargeter.select(SkillCall call)` returns `List<SkillTarget>`.
  - `MobCondition.test(SkillCall call, SkillTarget target)` returns `boolean`; `target == null` means the caster is tested.
  - Each has a nested `Factory` with `create(ExtensionArgs args)`.
- `MobTrait`: `id()`, `utilityMultiplier(String category)`, `stat(String stat, double value)`, `threat()`. `MobEffect`: `id()`, `onStart`, `onTick`, `onRefresh`, `onEnd`. `MobBrainProfile(String id, Map<String,Double> weights, double preferredRange, boolean kite, double fleeBelow, double threat)`.
- `SkillCall` (runtime view for extensions): `caster()`, `casterMob()`, `origin()`, `trigger()`, `targets()`, `args()`, `number(String key, double fallback)` (argument evaluated as a formula), `evaluate(String, double)`, `variable(...)`, `context()`, `rules()`, `boolean mayAffect(Entity target, boolean harmful)`, `castSkill(String)`, `skillId()`. `SkillTarget(Entity entity, Location location)` with `of(Entity)`, `at(Location)`, `asEntity()`, `position()`. `ExtensionArgs(Map<String,String> values)` with `string(String, String)` and `has(String)`.
- **Threading:** extensions are invoked by the mob runtime on the server thread. World access is only allowed there. Vystorm-Mobs calls the factory **on every application** (per target), so keep `create` cheap. Custom mechanics are **not** relation-checked automatically: call `call.mayAffect(entity, true)` before you apply harm.
- **Triggers.** `static void fireTrigger(UUID mobId, String trigger, Entity cause)` sends a signal to a mob. `mobId` is `MobHandle.id()`. It does nothing when no sink is set or the arguments are blank. `static boolean hasSink()`. `static void sink(Plugin owner, TriggerSink sink)` and `sink(TriggerSink)` are **provider only** (Vystorm-Mobs); `null` removes the sink. Vystorm-Mobs runs the definition's skill lines registered under that trigger name (for example `onSignal:<name>`).

```java
MobExtensions.registerMechanic(this, "myplugin_ignite", args -> (call, target) -> {
    Optional<Entity> e = target.asEntity();
    if (e.isEmpty() || !call.mayAffect(e.get(), true)) return false;
    e.get().setFireTicks((int) call.number("ticks", 60));
    return true;
});
```

---

### Threat and combat policy

#### ThreatServices

**Purpose.** A shared, thread-safe threat score per target (player, cluster, region or world). Directors (SpawnDirector, WorldDirector) read it, and any plugin can add modifiers. The effective value is `clamp(sum(additive) × product(multiplier), min, max)`.

| Type / method | |
|---|---|
| `static ThreatScoreService ThreatServices.get()` | Always non-null (implemented by Core). |
| `void putModifier(Plugin owner, ThreatTarget target, NamespacedKey source, double additive, double multiplier, int priority, Duration ttl, boolean persistent)` | Upsert keyed by `(target, source)`. |
| `default void addPoints(Plugin owner, ThreatTarget target, NamespacedKey source, double amount, Duration ttl)` | Additive only (multiplier 1). |
| `default void setMultiplier(Plugin owner, ThreatTarget target, NamespacedKey source, double multiplier, Duration ttl)` | Multiplier only. |
| `boolean removeModifier(Plugin owner, ThreatTarget target, NamespacedKey source)` | Returns false if the modifier is missing or not yours. |
| `int removeOwner(Plugin owner)` | Removes all modifiers of the plugin. |
| `ThreatSnapshot snapshot(ThreatTarget target, double minimumThreat, double maximumThreat)` / `default ThreatSnapshot snapshot(ThreatTarget target)` (bounds 0–100) | Explains the value: `additiveTotal`, `multiplierProduct`, `effectiveThreat`, `modifiers`, `calculatedAtEpochMilli`. |
| `record ThreatTarget(ThreatScope scope, String key)` | `static player(UUID)`, `cluster(String)`, `region(String)`, `world(UUID)`. `ThreatScope { PLAYER, CLUSTER, REGION, WORLD }`. |
| `record ThreatModifier(NamespacedKey source, double additive, double multiplier, int priority, long expiresAtEpochMilli, boolean persistent)` | `boolean isExpired(long now)`. |

- **Errors:** `source` must be in **your plugin's namespace** (`new NamespacedKey(plugin, "...")`), otherwise `IllegalArgumentException`. A non-finite `additive`, or a negative or non-finite `multiplier`, throws `IllegalArgumentException`. So do invalid bounds (non-finite, or min > max).
- `ttl` of `null`, zero or negative means no expiry. Expired modifiers are dropped on read and purged by Core every minute. `persistent` is stored as metadata only; modifiers are kept in memory and are lost on restart.
- **Threading:** every method may be called from any thread.
- **Lifecycle:** threat modifiers are **not** part of `CoreLifecycle`. Core removes them on disable only through a listener that registers lazily on the first `ThreatServices.get()` made from the server thread. Call `ThreatServices.get().removeOwner(this)` in `onDisable` to be safe.
- `ThreatServices.purgeExpired()` is called by Core; it is not needed by plugins.

```java
NamespacedKey key = new NamespacedKey(this, "bloodmoon");
ThreatServices.get().addPoints(this, ThreatTarget.world(world.getUID()), key, 25, Duration.ofMinutes(10));
double t = ThreatServices.get().snapshot(ThreatTarget.player(p.getUniqueId())).effectiveThreat();
```

#### CombatRegistry

**Purpose.** Pluggable **player-vs-player** damage policy (bounties, duels, protected zones). Core evaluates it on `EntityDamageByEntityEvent` at priority `HIGHEST`. Projectiles, TNT, area-effect clouds and tamed pets are resolved to the player who owns them; self-damage is ignored.

| Type / method | |
|---|---|
| `enum Decision { PASS, ALLOW, DENY }` | |
| `record Context(Player attacker, Player target)` | Both non-null. |
| `interface Policy` | `String id()`, `int priority()`, `Decision evaluate(Context context)`. |
| `static void register(Plugin owner, Policy policy)` | The id is normalized to lower case. A blank id throws `IllegalArgumentException`. An id owned by another plugin throws `IllegalStateException`. |
| `static boolean unregister(Plugin owner, String id)`, `static void unregisterAll(Plugin owner)` | |
| `static Decision evaluate(Player attacker, Player target)`, `static Decision evaluate(Context context)` | Policies are evaluated by priority, descending (then by registration order). The first non-`PASS` decision wins. A `null` result counts as `PASS`, and a policy that throws is skipped. |
| `static List<String> policies()` | Diagnostics. |
| `static void clear()` | **Not for plugins:** it removes every policy. |

- **Effect:** `DENY` cancels the event. `ALLOW` un-cancels it, but only if the event was not already cancelled before priority `NORMAL`. `PASS` leaves it unchanged.
- **Threading:** server thread. The policy must be fast.

```java
CombatRegistry.register(this, new CombatRegistry.Policy() {
    public String id() { return "myplugin-duels"; }
    public int priority() { return 100; }
    public CombatRegistry.Decision evaluate(CombatRegistry.Context c) {
        return duels.active(c.attacker(), c.target()) ? CombatRegistry.Decision.ALLOW : CombatRegistry.Decision.PASS;
    }
});
```

---

### Parties (`party`)

**Purpose.** Party membership for all plugins (XP share, friendly fire, shared loot). The only source is **VystormParties**. Without a source, every player is alone.

| `PartyServices` | Behaviour without a source or on failure |
|---|---|
| `static void register(Plugin owner, PartySource source)` / `static void unregister(Plugin owner)` | Provider only. |
| `static boolean available()` | |
| `static Optional<PartyView> partyOf(UUID player)` / `static Optional<PartyView> party(UUID partyId)` | empty |
| `static Collection<PartyView> all()` | empty list |
| `static boolean sameParty(UUID a, UUID b)` | false (also when `a` equals `b`) |
| `static List<Player> onlineMembers(Player player)` | Online members **excluding** the player; server thread. |
| `static List<Player> nearbyMembers(Player player, double radius)` | Same world and within `radius`; server thread. |
| `static boolean flag(UUID player, String key, boolean fallback)` | `fallback` |
| `static boolean setting(UUID partyId, String key, String value)` | Sets or removes (`null`) a party setting. Returns false without a source. |
| `static void fire(PartyChangeEvent event)` | **Provider only.** Fires synchronously on the server thread, otherwise on the next tick. |

- `PartySource` (provider): `partyOf(UUID)`, `party(UUID)`, `all()`, `boolean setting(UUID partyId, String key, String value)`. Reads must be **fast and thread-safe**, so the `PartyServices` read methods may be called from any thread (except `onlineMembers` and `nearbyMembers`).
- `record PartyView(UUID id, String name, UUID leader, Set<UUID> members, Map<String,String> settings, long createdAt)`: `contains(UUID)`, `setting(String, String)`, `flag(String, boolean)`. Settings are free-form keys (for example `xp-share`, `friendly-fire`). Other plugins should prefix their own keys.
- `PartyChangeEvent` (server thread): `kind()` (`CREATED, JOINED, LEFT, KICKED, LEADER, RENAMED, SETTING, DISBANDED`), `party()` (the state after the change; the last state for `DISBANDED`), `player()` (may be null for `SETTING` and `DISBANDED`), `detail()` (for example the setting key).

```java
for (Player m : PartyServices.nearbyMembers(killer, 32)) if (PartyServices.flag(killer.getUniqueId(), "xp-share", true)) giveXp(m, share);
```

---

### World events (`worldevent`)

**Purpose.** A neutral view of running world events (blood moon, invasions, …) and their numeric modifiers at a position. The only source is **WorldDirector**. Without a source there are no events and every modifier is `1.0`.

| `WorldEventServices` | |
|---|---|
| `static void register(Plugin owner, WorldEventSource source)` / `static void unregister(Plugin owner)` | Provider only. |
| `static boolean available()` | |
| `static List<WorldEventView> active()` | All running events. |
| `static List<WorldEventView> activeAt(Location location)` / `activeAt(UUID worldId, int x, int y, int z)` | Events that apply at the position (global, world, region or local scope). |
| `static boolean activeAt(Location location, String eventIdOrTag)` | Matches the event id or a tag, case-insensitively. |
| `static double modifier(String key, Location location)` / `modifier(String key, UUID worldId, int x, int y, int z)` | The combined multiplier, clamped to `MIN_MODIFIER`–`MAX_MODIFIER` (0–10). `1.0` without a source, with a null key or location, or on a non-finite value or failure. |
| `static void fireStarted(WorldEventView event)`, `static void fireEnded(WorldEventView event, WorldEventEndedEvent.Reason reason)` | **Provider only.** They fire `WorldEventStartedEvent` and `WorldEventEndedEvent` (`Reason { EXPIRED, MANUAL }`), always on the server thread. |

- `WorldEventSource` (provider): `active()`, `activeAt(UUID, int, int, int)`, `modifier(String key, UUID worldId, int x, int y, int z)`. Calls must be fast and free of side effects, and must never load chunks.
- `record WorldEventView(UUID instanceId, String eventId, String category, String type, String scope, String scopeKey, Set<String> tags, Instant startedAt, Instant endsAt, double pressure)`: `type` is one of `POSITIVE|NEGATIVE|MIXED|LEGENDARY`, `scope` one of `GLOBAL|WORLD|REGIONAL|LOCAL`. Also `boolean hasTag(String)`.
- Modifier key names are defined by the source's event catalog, not by Core.
- **Threading:** Core does not switch threads for reads. Call them from the server thread unless the source documents that it is thread-safe.

```java
double lootMult = WorldEventServices.modifier("loot", mob.getLocation());
@EventHandler public void onStart(WorldEventStartedEvent e) { if (e.event().hasTag("bloodmoon")) announce(e.event()); }
```

---

### Spawn points (`spawn`)

**Purpose.** Fixed spawn points per region. The plugin that builds a region (for example WorldForge for `dungeon:<id>`) publishes the points, and spawners (SpawnDirector) read them and fill them. Neither side depends on the other.

| Type / method | |
|---|---|
| `record SpawnPoint(String id, String region, UUID world, int x, int y, int z, String role, String theme, int level, Map<String,String> attributes)` | `static SpawnPoint of(String id, String region, UUID world, int x, int y, int z, String role, String theme)`. The constructor throws `IllegalArgumentException` if `id` is blank or longer than 200 characters, if `role` is blank, or if `level` is outside 0–100 (0 means the spawner profile decides). `region` is a region key `kind:id`. Change `id` to have a location re-populated (for example for a new wave). |
| `static SpawnPointService get()` | Singleton. |
| `void sync(Plugin owner, String region, Collection<SpawnPoint> points)` | Replaces all points of a region. An empty collection removes the region. Throws `IllegalStateException` if another plugin owns the region, and `IllegalArgumentException` if a point's `region` does not match. |
| `void retain(Plugin owner, Collection<String> regions)` | Keeps only the listed regions of this plugin (reconciliation after a restart or teardown). |
| `int unregisterOwner(Plugin owner)` | |
| `List<SpawnPoint> inRegion(String region)`, `Set<String> regions()`, `List<SpawnPoint> all()`, `Optional<String> owner(String region)` | Reads. |
| `long revision()` | Increases on every change. Use it as a cache key. |

**Threading:** writes are synchronized and belong on the server thread. Reads are lock-free and allowed from **any thread**. Ownership is tracked by plugin name.

```java
SpawnPointService.get().sync(this, "dungeon:crypt_7", List.of(
        SpawnPoint.of("crypt_7:boss:w1", "dungeon:crypt_7", world.getUID(), 120, 40, -88, "BOSS", "undead")));
```

---

### Progression (`progression`)

Two single-source bridges, both provided by **Vystorm Skills**. Mobs, quests, crafting and events use them without depending on the skills plugin. `available()` also requires the owner to be enabled.

#### CharacterServices

Character level and XP (the main class with talent points, separate from disciplines and from vanilla XP).

| `CharacterServices` | Without a source or on failure |
|---|---|
| `static void register(Plugin owner, CharacterSource source)` / `static void unregister(Plugin owner)` | Provider only. |
| `static boolean available()` | |
| `static int level(UUID player)` | `1` (the result is always at least 1) |
| `static int maxLevel()` | `1` |
| `static double awardXp(Player player, double amount, String source, int sourceLevel)` | `0`. Also `0` if `player` is null or `amount <= 0`. Returns the amount actually granted. `sourceLevel` (for example the mob level) enables level-gap damping; `0` means no damping. **Server thread.** |

`CharacterSource` (provider): `int level(UUID)`, `int maxLevel()`, `double awardXp(Player, double, String, int)`. Reads must be fast and thread-safe; `awardXp` runs on the server thread only.

#### DisciplineServices

Professions and disciplines for crafting, smithing, repair and salvage.

| `DisciplineServices` | Without a source or on failure |
|---|---|
| `static void register(Plugin owner, DisciplineSource source)` / `static void unregister(Plugin owner)` | Provider only. |
| `static boolean available()` | |
| `static Set<String> disciplines()` | empty set |
| `static int level(UUID player, String discipline)` | `0` |
| `static double modifier(UUID player, String key)` | `0` (neutral). Example keys: `smithing.quality`, `salvage.yield`. |
| `static boolean hasUnlock(UUID player, String unlock)` | `false` |
| `static double awardXp(Player player, String discipline, double amount, String source)` | `0`. Returns the amount granted. **Server thread.** |

`DisciplineSource` (provider) has the same five methods. Reads must be fast and thread-safe; `awardXp` runs on the server thread.

```java
int lvl = CharacterServices.level(p.getUniqueId());
CharacterServices.awardXp(p, 40.0, "myplugin:quest", 0);
double quality = 1.0 + DisciplineServices.modifier(p.getUniqueId(), "smithing.quality");
```

---

### Plugin utilities

#### VersionedStore

**Purpose.** A crash-safe state store with a schema version. The whole state is stored as JSON in an envelope `{"schema": n, "data": {...}}`. Older versions are migrated step by step on load; content with a newer version is never overwritten.

| Member | |
|---|---|
| `static <T> VersionedStore<T> open(JavaPlugin plugin, API core, String folder, String file, Class<T> type, int schema, Map<Integer, UnaryOperator<JsonObject>> migrations, Supplier<T> empty) throws Exception` | Store at `<dataFolder>/<folder>/<file>` through Core's data layer. Unreadable content is backed up next to it as `<file>.corrupt-<millis>`. |
| `VersionedStore(Backend backend, Class<T> type, int schema, Map<Integer, UnaryOperator<JsonObject>> migrations, Supplier<T> empty, Gson gson)` and a 7-argument overload with a trailing `Consumer<String> quarantine` (since 0.11.1) | For a custom `Backend { String read(); void write(String value); }`. A `null` gson means default Gson. |
| `T load()` | Empty storage returns `empty.get()` and saves it. Migrated state is saved again. |
| `void save(T state)` | A failed write is rolled back (default backend). |
| `int schema()` | |

- **Errors:** `schema < 1`, or a migration key outside `2..schema`, throws `IllegalArgumentException` (constructor). Content with a newer schema throws `IllegalStateException`. Unreadable JSON throws `VersionedStore.CorruptStoreException` (an `IllegalStateException`); in that case **do not continue with an empty state**. A migration returning `null` throws `IllegalStateException`. A JSON object without an envelope counts as version 1.
- `migrations` maps a *target* version to a step (key 2 migrates from version 1 to 2).
- **Threading:** synchronous I/O on the calling thread. The default backend synchronizes on its entry.

```java
store = VersionedStore.open(this, CoreApi.api().orElseThrow(), "state", "state.yml", MyState.class, 2,
        Map.of(2, json -> { json.addProperty("version2Field", 0); return json; }), MyState::new);
MyState s = store.load();
```

#### ConfigMigrator

**Purpose.** Versioned migration of `config.yml`. Each step raises the config by one version. New default keys are added without overwriting user values, and a backup `config.yml.v<old>.bak` is written before any change.

| Member | |
|---|---|
| `ConfigMigrator(String versionKey, int current)` | `current < 1` throws `IllegalArgumentException`. |
| `ConfigMigrator step(int target, Consumer<FileConfiguration> migration)` | The step migrates from `target-1` to `target`. A target outside `2..current` throws `IllegalArgumentException`. Missing steps are allowed (only defaults are added; logged). |
| `int migrate(FileConfiguration config)` | In memory only. Returns the source version; a missing key counts as 1. A newer version throws `IllegalStateException`. |
| `int migrate(JavaPlugin plugin) throws IOException` | Saves the default config, migrates, copies defaults, and backs up and saves the file if anything changed. |

```java
new ConfigMigrator("config-version", 3)
        .step(2, c -> c.set("spawn.radius", c.getInt("radius", 32)))
        .step(3, c -> c.set("radius", null))
        .migrate(this);
```

#### TaskGroup

**Purpose.** A named group of Bukkit tasks belonging to one part of a plugin. `cancel()` stops only this group's tasks. An exception in a task is caught and logged (at most every 30 s) and the task keeps running.

`static TaskGroup of(Plugin plugin, String name)`. Methods: `BukkitTask timer(long delayTicks, long periodTicks, Runnable action)`, `BukkitTask later(long delayTicks, Runnable action)`, `BukkitTask now(Runnable action)`, `BukkitTask async(Runnable action)`, `BukkitTask asyncTimer(long delayTicks, long periodTicks, Runnable action)`, `void cancel()` (the group stays usable), `int size()`, and `void close()` (`AutoCloseable`). After `close()`, scheduling cancels the new task and throws `IllegalStateException`. Thread rules are those of the Bukkit scheduler.

```java
TaskGroup ai = TaskGroup.of(this, "ai");
ai.timer(20L, 20L, this::tickDirectors);
// onDisable: ai.close();
```

#### CoreLifecycle and ProviderFailures

- `CoreLifecycle` (a Listener that Core registers itself) calls `static void releaseAll(Plugin owner)` for every disabled plugin. It removes the plugin from: `MobServices`, `RelationServices`, `MobContextServices`, `MobExtensions` (including the trigger sink), `ItemServices`, `ItemRefs`, `DisciplineServices`, `CharacterServices`, `PartyServices`, `WorldEventServices`, `CombatRegistry`, `RuntimeActionHub`, `RuntimeContextHub`, `WorldEnvironmentRegistry` (detectors and explicit profiles), `RegionService`, `SpawnPointService`, `WebServices`, `ClientUi` and `Localization`. Each step is idempotent and isolated from failures in the others. Plugins **do not need** to call it. Calling `releaseAll(this)` yourself is harmless but redundant.
- `ProviderFailures` is Core's throttled logger for registry faults (at most one message per minute per key; suppressed repeats are counted). It has `static void report(String registry, String provider, String operation, Throwable failure)` and `static void replaced(String registry, String key, Plugin previousOwner, Plugin newOwner)`. It is thread-safe. Plugins may use it for their own registries, but it is primarily Core-internal and its output format is not a contract.

---

## 18. Not public API

These are `public` in the jar for technical reasons but are **not** part of the plugin API. They may change without
notice; don't call or implement them.

| Symbol | Why |
| --- | --- |
| `Vystorm.vystorm_core.web.internal.*` (incl. `clientproto.*`), `Vystorm.vystorm_core.internal.*` | Core-internal HTTP server, gateway tunnel, sessions, client protocol, test hooks. |
| `main` (the plugin class) except the legacy `main.getApi()` | Plugin bootstrap. Use `CoreApi.api()`. |
| `bundleclasses.*`, `commands.*`, `_uiMethods.commands.*` (`api.command()`), `_uiMethods.admin.CoreAdminDashboard`, `_uiMethods.admin.AdminRegistry` | Core's own commands, listeners and dashboard implementation. |
| `API` constructor, `API.shutdownAdvanced()`, `API.shutdownDebugger()`, `API.releaseAdvancedResources(Plugin)` | Core lifecycle (runs automatically on disable). |
| `API.override*(…)` and `singeltons.*` | Replace a handler for **all** plugins on the server. Only for Core tests or a deliberate server-wide replacement – never in normal plugins. |
| `GuiInteface.handleClick/handleDrag/handleOpen/handleOpenResult/handleClose`, `DialogInterface.handle`, `DialogInterface.clear`, `GuiInteface.clearManaged`, `SettingsInterface.clear`, `AdminInterface.clear` | Called by Core's listeners / shutdown. |
| Legacy auto-GUI: `GuiInteface.createGui/addItem/addMenu/removeMenu/removeItem/setSpecificItem/getGui/getInventory/getMenuNames`; `api.message()`, `api.ressource()` (`LanguageSystem`, `language/language.yml`) | Superseded by managed inventories and `api.lang()`. |
| `SettingsRegistry` (the implementation class: `view`, `save`, `coerce`, `onChange`, `SaveResult`, `SectionView`, …) | Used by Core's dialogs and settings web page. Plugins use `SettingsInterface`. |
| `lang.CoreLanguage`, `Localization.configure/installTranslator/uninstallTranslator/clearPlayers/setDefaultLanguage/setPersonalLanguage/rememberClientLocale/unregister/reloadAll` | Core-managed language state. |
| `SharedDatabase.start(...)`, `SharedDatabase.shutdown()`, `SharedDatabaseSettings`, `SharedPool`; `DataBaseInterface.registerDataBase/unregisterDataBase/disconnectALL` | Pool lifecycle belongs to Core. Never close or unregister the shared pool. |
| `CoreLifecycle.releaseAll(Plugin)` | Runs automatically on `PluginDisableEvent`. |
| `_uiMethods.menu.CoreMenus`, `PauseMenuPack`, `_uiMethods.tutorial.TutorialGuide.Content/Page` | Core's menus and pause-menu data pack. |
| `api.debug().shutdown(...)` | Requests a server shutdown. |
| Deprecated: `DataStorageInterface.convertDataEntry/exportDataEntry`, `DataBaseEntry`, `Saver.saveDataEntryExtern`, `PlayerPickupItemListener`, `main.onReload()` | Not implemented / never called. |

Further internal members of the item, mob, region and pack APIs are marked in sections 16 and 17.

---

## 19. Rules for AI assistants

Follow these when generating code against Vystorm Core.

**Do**
- Depend on `vystorm_core` in `plugin.yml`, compile against the jar with `compileOnly`, target Java 25 and
  `api-version: '26.2'`.
- Get the API with `CoreApi.api()`; check features with `CoreApi.has(CoreApi.Capability.X)` inside
  `try { … } catch (LinkageError tooOld)` and degrade gracefully (or disable with a clear log line).
- Register everything (lang, settings, menus, web modules, providers) once in `onEnable`, on the server thread.
- Put every player-visible text into `lang/en.yml` (+ `de.yml`) and use `lang.send/component/text/...`; for text that
  Core shows to many players (menu, settings, admin, web titles, `WebException`) pass `lang.ref("key")`.
- Treat everything from the web (`WebRequest` path params, query, body), dialogs, chat and commands as untrusted:
  validate ranges, lengths, ownership and state on the server; use `PreparedStatement` parameters for SQL.
- In web handlers and snapshot sources use `request.sync(() -> …)` for any Bukkit/world/inventory/config access and
  keep shared state in thread-safe types.
- Use `WebServices.openOrElse`/`openOrLink` so players without the client mod or with the platform off still get a
  working path; every client-mod feature needs a vanilla fallback.
- Use the shared DB only via `sharedReady()`, off the server thread, with `SchemaMigrator` and your own table prefix.
- Start debug messages with a `[Plugin/Component]` marker; log failures with `DebugMessages.high(msg, throwable)`.
- Quote YAML strings in lang/config files.

**Don't**
- Don't call Bukkit API, `getConfig()` or player/world methods from web handlers, DB callbacks or other async code
  without switching to the server thread.
- Don't use `web.internal.*`, `main`, `singeltons`, `override*Handler` or anything in section 18.
- Don't hard-code player-visible strings, don't concatenate translated fragments (use placeholders), don't put
  MiniMessage tags into `web.ui.*` texts (they are sent raw).
- Don't put secrets, tokens or player data into static web assets (they are public); don't log login tokens.
- Don't use `innerHTML`, inline `<script>`/`style` or `onclick=` attributes in web pages (CSP blocks them; XSS risk).
- Don't block the server thread (JDBC, HTTP, file-heavy work, `highConfirmed`, `Future.get()`).
- Don't close or unregister Core's shared `DataSource`; don't keep connections open.
- Don't re-register the same settings section/menu id without unregistering first (throws).
- Don't assume the web platform, the shared database, Vault or the client mod is available – check
  `WebServices.available()`, `sharedReady()`, `ClientServices.hasClient()` etc.

**YAML gotchas (Bukkit uses YAML 1.1).** Unquoted `on`, `off`, `yes`, `no`, `true`, `false` (any case) become
booleans – as keys *and* as values. `toggle: {on: An}` is read as key `toggle.true`, and `answer: no` as `false`. Quote
them: `"on": "An"`, `value: "no"`. Also quote values starting with `{`, `[`, `*`, `&`, `!`, `%`, `@`, `` ` `` or
containing `: ` / ` #`; MiniMessage values starting with `<` are safer quoted too. Keys are paths separated by `.`, so a
key cannot itself contain a dot. A lang value can be a string or a list of lines. Invalid YAML makes the whole file
unusable (Core logs it and falls back to English/jar texts).

---

## 20. Versioning and compatibility

- **Additive only.** Public types and members are never removed or renamed and signatures never change. New methods on
  existing interfaces always get a `default` body. Core's build enforces this with `ApiCompatibilityTest` against a
  baseline dumped from the jar the plugins build against. Deprecated members stay.
- **Capability flags.** Every new feature gets a `CoreApi.Capability` constant with its since-version. Check the
  capability, not the version string; a plugin built against 0.24.0 can run on an older Core if it checks capabilities
  and avoids newer calls.
- **Behaviour changes** (security fixes, stricter validation) are listed in the changelog as "Geändert" (changed).
- **Minecraft version.** The version suffix (`-Vystorm-Core-26.2`) names the Minecraft/Paper line; only the numeric
  `x.y.z` part is compared by `CoreApi.atLeast`/`compare`.
- **Changelog:** `CHANGELOG.md` in the Core repository (per version: Neu/Geändert/Deprecated). The README contains the
  admin-facing documentation (configuration, web platform modes, security model in `WEB_SECURITY.md`).

---

## 21. Appendix: Core commands and permissions

| Command | Aliases | Purpose | Permission |
| --- | --- | --- | --- |
| `/menu [list\|settings\|<plugin:key>]` | `/vmenu`, `/vm`, `/m` | Main menu, menu entries | entry `access` predicates |
| `/einstellungen [server\|<plugin:section>]` | `/vsettings`, `/vystormsettings` | Settings dialogs | section permissions |
| `/web [module\|logout\|status]` | – | One-time web login link, logout, status (admin) | `vystorm.web.use` (default: everyone) |
| `/vadmin [list\|status\|<module>\|language [reload]]` | `/admin`, `/ad` | Admin dashboard | `vystorm_core.admin` (default: op) |
| `/vregion at [player] \| list [kind]` | – | Region register | `vystorm_core.admin` |
| `/nbt <key> <value>` | – | Set a Core item tag on the held item | `vystorm_core.admin` |

Other permissions: `vystormCore.debug` (HIGH debug messages in chat, default op), `vystorm_core.chat.teleport`
(clickable coordinates teleport, default op).
