# Vystorm Client – Server Integration API

Reference for building **server-side integrations** for the Vystorm Client mod: one-key web page sign-in and native
HUD elements. It is written to be self-contained, so a developer or an AI coding assistant can implement an integration
without seeing the mod's source code. All names, types and limits below are taken from the mod's code.

| | |
|---|---|
| Mod | Vystorm Client **0.8.0** (Fabric, Minecraft **26.2**, Java 25), client-side only, no other mods needed besides Fabric API |
| Protocol | version **2** (the mod accepts servers speaking version 1 or 2) |
| Channels | `vystorm:web` (handshake, web pages), `vystorm:ui` (native elements) and `vystorm:panel` (native menus, mod 0.8.0+) |
| Server reference implementation | **Vystorm Core** (Paper/Purpur 26.2 plugin), 0.21.0 or newer for protocol 2, 0.25.0 or newer for panels |
| License | MIT |

---

## Contents

1. [Rules that always apply](#1-rules-that-always-apply)
2. [Which path to use](#2-which-path-to-use)
3. [Path A – Vystorm Core Java API (recommended)](#3-path-a--vystorm-core-java-api-recommended)
4. [Protocol reference](#4-protocol-reference)
5. [Native elements (`vystorm:ui`) – JSON schema](#5-native-elements-vystormui--json-schema)
6. [Web pages – what your web pages must do](#6-web-pages--what-your-web-pages-must-do)
7. [Trust and security model](#7-trust-and-security-model)
8. [Path B – raw plugin messages without Core](#8-path-b--raw-plugin-messages-without-core)
9. [Native panels (`vystorm:panel`)](#9-native-panels-vystormpanel)
10. [Errors, rejections and troubleshooting](#10-errors-rejections-and-troubleshooting)
11. [Player settings that affect your integration](#11-player-settings-that-affect-your-integration)
12. [Versions and compatibility](#12-versions-and-compatibility)
13. [Checklist](#13-checklist)

---

## 1. Rules that always apply

1. **The mod is optional.** Every feature must also work for players without it (chat links, chest/dialog menus,
   boss bars, action bar). Send nothing on the Vystorm channels to a player who has not sent a HELLO.
2. **The client is never an authority.** Plugin messages only carry the handshake, one-time login tokens, "show page X"
   and display data. Every game action goes through normal commands or your authenticated web API – never through
   these channels. Treat everything the client sends as untrusted hints.
3. **The player decides.** The mod only opens a server's web pages – in the player's own browser, on the player's key
   press – after the player allowed that exact web origin for that server (a dialog the player opens with a key press –
   the server can never pop it up). Plan for players who never allow it.
4. **Validate before sending, expect rejections.** The client validates strictly and rejects rather than "repairs"
   (out-of-range numbers, unknown element types, bad colors …). Invalid data is dropped and reported back (REJECT).
5. **Rate-limit and batch.** At most one PATCH per element per tick; progress bars and cooldowns at most ~4 updates per
   second (the client counts cooldowns down itself).
6. **Never log or display login tokens.**

## 2. Which path to use

| | Path A: Vystorm Core (recommended, supported) | Path B: raw plugin messages |
|---|---|---|
| Platform | Paper/Purpur 26.2 + Vystorm Core | any server that can send custom payloads |
| Handshake, versioning, rate limits, batching | done by Core | you implement them |
| Web pages (login tokens, sessions, CSRF) | Core's web platform (`web.enabled: true`) | you need your own web backend (see §6, §8.4) |
| Native HUD elements | `ClientUi` + typed `ClientElement` builders | JSON over `vystorm:ui` |
| Native menus (screens) | `panel.Panels` + `PanelSpec` + `Ui` (Core 0.25.0+) | JSON over `vystorm:panel` (§9) |
| Validation | Core validates with the same schema as the client | you validate (or read REJECTs) |

**Core is the supported path.** The raw protocol is documented here so that other server software can talk to the
mod; it is stable within a protocol version (§4.2), but only Core is maintained alongside the mod.

---

## 3. Path A – Vystorm Core Java API (recommended)

All classes are in package `Vystorm.vystorm_core.web`. Everything is **server-thread only** (`ClientUi` throws
`IllegalStateException` when called from another thread).

### 3.1 Setup

- `plugin.yml`: `depend: [vystorm_core]` (or `softdepend` if your plugin works without Core).
- Compile against the Vystorm Core jar as `compileOnly` (do not shade it).
- Check capabilities before use; older Core versions do not have these classes:

```java
boolean client, clientUi;
try {
    client   = CoreApi.has(CoreApi.Capability.WEB_CLIENT);        // "web-client",       Core 0.19.0+
    clientUi = CoreApi.has(CoreApi.Capability.WEB_CLIENT_UI);     // "web-client-ui",    Core 0.20.0+
    // CoreApi.Capability.WEB_CLIENT_TRUST = "web-client-trust"   // protocol 2 trust bit, Core 0.21.0+
} catch (LinkageError tooOld) {
    client = clientUi = false;
}
```

### 3.2 Core configuration (`plugins/vystorm_core/config.yml`)

Opening web pages needs Core's web platform: `web.enabled: true` and a `web.public-url` that players can reach (this
URL is what the mod pins and shows to the player; use `https`). Native elements work even when the web platform is
off.

| Key | Default | Meaning |
|---|---|---|
| `web.client.enabled` | `true` | Register `vystorm:web` / `vystorm:ui`. `false` = the mod sees a vanilla server. |
| `web.client.overlay` | `true` | WELCOME flag `SERVER_OVERLAY` (in-game display allowed). Mod 0.7.0 has no in-game display and always uses the system browser, so this has no visible effect for it. |
| `web.client.preload` | `true` | Hidden sign-in right after joining. Only used by mods up to 0.6.0; 0.7.0 never asks. |
| `web.client.hotkey-module` | `skills` | Module the web key (Shift+menu key, or the menu key without panels) opens; `""` = overview. Ignored if the module is not registered. |
| `web.client.open-per-minute` | `10` | Token requests per minute and player (burst 3). |
| `web.client.end-sessions-on-quit` | `client` | `client` / `all` / `none`: which web sessions end when the player quits. |
| `web.client.ui.enabled` | `true` | Native elements channel (`vystorm:ui`). |
| `web.client.ui.patch-interval-ticks` | `5` | At most one PATCH per element every N ticks (5 = 4×/s); later changes overwrite earlier ones. |
| `web.client.ui.messages-per-second` / `burst` | `30` / `100` | Send rate per player (below the client's 40/s, burst 120). |
| `web.client.ui.per-tick` | `32` | Messages per player per tick. |
| `web.client.ui.queue-limit` | `384` | Queued messages per player; more → `QUEUE_FULL`, dropped. |

### 3.3 `ClientServices` – handshake state and SHOW

| Member | Description |
|---|---|
| `CAP_BROWSER_OVERLAY = 1` | In-game page display. **Mod 0.7.0 never sets it** (pages open in the system browser); 0.6.0 and older set it when their optional embedded browser was usable. Reserved. |
| `CAP_EXTERNAL_BROWSER = 2` | Can open the login link in the system browser (always set). |
| `CAP_HUD = 4` | `panel`, `bar`, `slots`. |
| `CAP_TOASTS = 8` | Toasts. |
| `CAP_MARKERS = 16` | World markers. |
| `CAP_TOOLTIPS = 32` | Tooltip rules. |
| `CAP_JS_BRIDGE = 64` | `window.vystormQuery` in the overlay (0.6.0 and older; 0.7.0 never sets it). |
| `STATE_UI_HIDDEN = 1` | Player hid the native elements (key H). |
| `STATE_OVERLAY_OPEN = 2` | In-game display is open (never set by 0.7.0). |
| `STATE_ORIGIN_TRUSTED = 4` | Player allowed the web origin and the client would follow a `show` (protocol 2). A hint, not a permission. Never set by 0.7.0, which never opens pages on server request. |
| `boolean hasClient(Player)` | Handshake completed (the player has the mod). |
| `int capabilities(Player)` | `CAP_*` bits from the handshake, `0` without the mod. |
| `int clientState(Player)` | Last reported `STATE_*` bits, `0` without the mod. |
| `boolean originTrusted(Player)` | `STATE_ORIGIN_TRUSTED` is set. |
| `boolean uiHidden(Player)` | `STATE_UI_HIDDEN` is set. |
| `boolean uiAvailable(Player, int capability)` | The player accepts native elements with this capability (and `web.client.ui` is on). |
| `boolean overlayAvailable(Player)` | Mod with in-game display (`CAP_BROWSER_OVERLAY`), overlay allowed by config, web platform running, **and** the player trusts the origin (protocol 2: trust bit; protocol 1: after the first token in this connection). Always `false` for mod 0.7.0. |
| `boolean show(Player, String moduleId, String page)` | Sends SHOW. `moduleId` = registered web module or `""` (overview); `page` = relative page (§4.6) or `""` (where the login redirect lands). Returns `false` when not possible (no mod, overlay not available, invalid/unregistered module, invalid page, rate limit 0.5/s burst 3) – then open your normal menu. |
| `boolean close(Player, String moduleId)` | Sends CLOSE (`""`/`null` = whatever module is open). |
| `boolean enabled()` | The client channel runs (`web.client.enabled`). |

Pattern – in-game page instead of a chest menu (with mod 0.7.0 `show` returns `false`, so the menu opens):

```java
// e.g. /shop or right-click on an NPC
if (ClientServices.show(player, "shop", "/m/shop/#npc/12")) return;   // player has the mod and allowed the origin
openChestMenu(player);                                                // everyone else: the vanilla way
```

`show` does not grant anything: the mod then signs in with a one-time token that is checked exactly like `/web`, and
every API call of the page is checked like in a normal browser. Players with mod 0.7.0 open the web interface
themselves with the web key (Shift+menu key, in their browser, already signed in); `WebServices.openOrLink` gives them a chat link instead. Register the web module itself with
`WebServices.register(plugin, WebModule.builder(id, title)…build())` (see Core's README, "Web platform").

### 3.4 `ClientUi` – native elements

```java
ClientUi ui = ClientUi.of(this);   // one instance per plugin (cached)
```

- **Namespace:** every element ID is prefixed with your plugin's namespace: plugin name in lower case, characters other
  than `[a-z0-9_.-]` → `_`, prefixed with `p` if it does not start with a letter/digit, cut to 32 characters
  (`VystormSkills` → `vystormskills`). Local ID `xp` becomes `vystormskills:xp` on the client. The **full ID** must be
  ≤ 64 characters and match `[a-z0-9][a-z0-9_.:/-]{0,63}`.
- Core validates every element with the client's schema before sending, removes your elements when your plugin is
  disabled and logs problems once per message (`[ClientUi] …` in your plugin's log).

| Method | Returns / notes |
|---|---|
| `boolean available(Player, int capability)` | Player accepts this capability (`ClientServices.CAP_HUD`, `CAP_TOASTS`, `CAP_MARKERS`, `CAP_TOOLTIPS`). |
| `boolean has(Player, String localId)` | Element exists on the client (not removed, expired or rejected). |
| `boolean put(Player, ClientElement.Element<?>)` / `put(Player, JsonObject)` | Create or fully replace. `false` without mod/capability or when invalid (warning in log). An identical PUT without `ttl` is not re-sent. |
| `boolean patch(Player, ClientElement.Element<?> fields)` / `patch(Player, String localId, JsonObject)` | Change only the given top-level fields. Coalesced per element (`patch-interval-ticks`), unchanged fields are not sent. **`false` if the element does not exist (any more) → call `put`.** |
| `boolean remove(Player, String... localIds)` | Remove elements. |
| `boolean clear(Player)` | Remove all of your plugin's elements for this player. |
| `void clearAll()` | Remove all of your plugin's elements for all players. |
| `boolean toast(Player, ClientElement.Toast)` / `toast(Player, JsonObject)` | One-off notification (`CAP_TOASTS`). |
| `String fullId(String localId)` | The ID as the client sees it. |
| `static String namespace(String pluginName)` | Namespace rule above. |

### 3.5 `ClientElement` – typed builders

A builder contains only the fields you set (the client uses defaults for the rest), so **the same builder type works
for PUT and PATCH**. Field meanings, defaults and limits: §5.

| Builder | Methods |
|---|---|
| `ClientElement.panel(id)` | `anchor(Anchor)`, `offset(x, y)`, `order(int)`, `ttl(ms)`, `width`, `padding`, `background(color)`, `border(color)`, `textColor(color)`, `title(String\|Text)`, `icon(Icon)`, `line(String\|Text)` (append), `lines(List<Text>)` (replace) |
| `ClientElement.bar(id)` | common HUD methods + `width`, `height`, `value(double)`, `max(double)`, `color`, `background`, `textColor`, `label(String\|Text)`, `segments(int)` |
| `ClientElement.slots(id)` | common HUD methods + `size`, `gap`, `slot(Slot)` (append), `slots(List<Slot>)` (replace) |
| `ClientElement.slot()` | `icon(Icon)`, `key(String)`, `count(int)`, `cooldown(remainingMs, totalMs)`, `active(boolean)`, `label(String\|Text)` |
| `ClientElement.marker(id)` | `ttl(ms)`, `at(x, y, z)` (required), `world(String\|World)`, `label`, `color`, `icon`, `maxDistance(double)`, `showDistance(boolean)`, `clampToScreen(boolean)` |
| `ClientElement.tooltip(id)` | `ttl(ms)`, `matchItem(String\|Material)`, `matchCustomData(path, equals)`, `line(String\|Text)` |
| `ClientElement.toast(String\|Text title)` | `text(String\|Text)`, `icon(Icon)`, `duration(ms)`, `style(Toast.Style.INFO\|SUCCESS\|WARNING\|ERROR)` |
| `ClientElement.Text` | `plain(s)`, `of(s, color)`, `then(s)`, `then(s, color)`, `bold()`, `italic()`, `underlined()`, `strikethrough()` (style applies to the last segment) |
| `ClientElement.Icon` | `item(String\|Material)`, `item(itemId, model)`, `sprite(String)`, `image(String path)` |
| `ClientElement.Anchor` | `TOP_LEFT, TOP, TOP_RIGHT, LEFT, CENTER, RIGHT, BOTTOM_LEFT, BOTTOM, BOTTOM_RIGHT` |

### 3.6 Events (synchronous, server thread)

| Event | When | Useful members |
|---|---|---|
| `VystormClientReadyEvent` | Handshake accepted (at most once per connection, shortly after join). Good place for the first `put` and to switch off vanilla fallbacks for this player. | `capabilities()`, `has(int)`, `modVersion()` (display/statistics only), `loader()` (`fabric`/`neoforge`) |
| `VystormClientStateEvent` | Client state changed (≤ 2/s, burst 6). | `state()`, `previous()`, `uiHidden()`, `uiHiddenChanged()`, `overlayOpen()`, `originTrusted()`, `originTrustedChanged()` |
| `VystormClientUiRejectEvent` | The client rejected one of your elements (≤ 5/s per player). After codes 2, 3, 4 the element counts as not present. | `owner()`, `elementId()`, `localId()`, `code()` (`INVALID=1`, `LIMIT=2`, `UNKNOWN_TYPE=3`, `UNKNOWN_ID=4`, `RATE=5`), `detail()` |

### 3.7 Complete example

```java
public final class LevelHud implements Listener {
    private final Plugin plugin;
    private final ClientUi ui;

    public LevelHud(Plugin plugin) {
        this.plugin = plugin;
        this.ui = ClientUi.of(plugin);
    }

    @EventHandler
    public void onReady(VystormClientReadyEvent e) {
        Player p = e.getPlayer();
        if (e.has(ClientServices.CAP_HUD)) {
            ui.put(p, xpBar(p));
            hideBossBar(p);                                   // vanilla fallback off for mod users
        }
        if (e.has(ClientServices.CAP_TOOLTIPS)) {
            ui.put(p, ClientElement.tooltip("flame-blade")
                    .matchCustomData("PublicBukkitValues.myitems:id", "flame_blade")
                    .line(ClientElement.Text.of("Burns targets for 3 s", "gold")));
        }
    }

    @EventHandler
    public void onState(VystormClientStateEvent e) {
        if (e.uiHiddenChanged()) {
            if (e.uiHidden()) showBossBar(e.getPlayer()); else hideBossBar(e.getPlayer());
        }
    }

    /** Call when XP changes; Core coalesces patches per tick. */
    public void onXp(Player p, int level, long xp, long need) {
        String label = "Lv " + level + " · " + xp + "/" + need + " XP";
        if (!ui.patch(p, ClientElement.bar("xp").value(xp).max(need).label(label))
                && ui.available(p, ClientServices.CAP_HUD)) {
            ui.put(p, xpBar(p));                              // not there (expired/rejected) → create
        }
    }

    public void onLevelUp(Player p, int level) {
        ui.toast(p, ClientElement.toast("Level " + level).text("+1 talent point")
                .style(ClientElement.Toast.Style.SUCCESS));   // no-op without the mod
        p.sendActionBar(Component.text("Level " + level)); // vanilla way for everyone
    }

    private ClientElement.Bar xpBar(Player p) {
        return ClientElement.bar("xp").anchor(ClientElement.Anchor.BOTTOM).offset(0, 52)
                .width(182).height(5).value(currentXp(p)).max(neededXp(p))
                .color("#FF7CFC00").label("Lv " + level(p)).segments(10);
    }

    public void markQuestTarget(Player p, Location l) {
        ui.put(p, ClientElement.marker("quest-target").at(l.getX(), l.getY() + 1, l.getZ())
                .world(l.getWorld()).label("Quest").color("gold").maxDistance(512).ttl(600_000));
    }
    // hideBossBar/showBossBar/currentXp/neededXp/level: your plugin
}
```

---

## 4. Protocol reference

### 4.1 Channels and framing

| Channel | Purpose |
|---|---|
| `vystorm:web` | Handshake, web pages (tokens, open/close, client state) |
| `vystorm:ui` | Native elements: HUD, toasts, world markers, tooltip rules |

Both are ordinary custom-payload plugin channels. The payload is the raw message bytes (no extra length prefix).
**Message = `VarInt messageId` followed by its fields.** The server must announce the channels (on Paper:
`registerIncomingPluginChannel` + `registerOutgoingPluginChannel` for both; Bukkit then sends `minecraft:register`).
The client sends HELLO only after the server has registered `vystorm:web`, and waits at most **60 s** after joining.

**Primitive types** (compatible with Minecraft's `FriendlyByteBuf`):

| Type | Encoding |
|---|---|
| `VarInt` | LEB128 like Minecraft: 7 bits per byte, low bits first, high bit = "more follows"; at most 5 bytes, 32 bit |
| `Bool` | 1 byte, only `0` or `1` |
| `String(n)` | `VarInt` byte length + UTF-8 bytes; at most `n` **bytes**; invalid UTF-8 is an error |

Decoding is strict: too long fields, truncated data, bad booleans and **trailing bytes** are errors. Unknown message IDs
are ignored (forward compatibility). Messages sent in the wrong direction are errors.

**Size limits per message** (bigger messages are dropped unread):

| Direction | Max size |
|---|---|
| Client → server (both channels) | 1 KiB (1024 bytes) |
| Server → client `vystorm:web` | 4 KiB (4096 bytes) |
| Server → client `vystorm:ui` | 32 KiB (32768 bytes) |

After **20 malformed messages** in one connection the client switches the protocol off for that connection
(fail closed; one warning in the client log).

### 4.2 Versioning

- HELLO announces `minVersion..maxVersion` (mod 0.4.0+: `1..2`). The server picks the **highest common version** and
  names it in WELCOME. If there is none, the server sends nothing and the client stays "vanilla".
- The client accepts WELCOME with version 1 or 2. Any other version disables the protocol for the connection.
- **Version 2** only adds CLIENT_STATE bit `ORIGIN_TRUSTED` and sends CLIENT_STATE right after WELCOME. In version 1 the
  client never sets that bit.
- Within a version only **new message IDs** and **new optional JSON fields** may be added. Changes to existing binary
  fields require a new version. Capabilities are bits in HELLO; use only what the client announces.

### 4.3 Bit fields

**HELLO `capabilities`** (mod 0.7.0 always sends exactly bits 1–5 = `62`; 0.6.0 and older also set bits 0 and 6 when
their optional embedded browser was usable):

| Bit | Value | Name | Meaning |
|---|---|---|---|
| 0 | 1 | `BROWSER_OVERLAY` | In-game page display (0.6.0 and older: embedded browser); reserved |
| 1 | 2 | `EXTERNAL_BROWSER` | Can open the login link in the system browser |
| 2 | 4 | `HUD` | `panel`, `bar`, `slots` |
| 3 | 8 | `TOASTS` | Toasts |
| 4 | 16 | `MARKERS` | World markers |
| 5 | 32 | `TOOLTIPS` | Tooltip rules |
| 6 | 64 | `JS_BRIDGE` | `window.vystormQuery` in the overlay (0.6.0 and older) |

**WELCOME `serverFlags`:** bit 0 (`1`) `SERVER_OVERLAY` = in-game display allowed (otherwise only the system browser);
bit 1 (`2`) `SERVER_UI` = server sends native elements (without it the client ignores all `vystorm:ui` messages).

**CLIENT_STATE `flags`:** bit 0 (`1`) `UI_HIDDEN`; bit 1 (`2`) `OVERLAY_OPEN`; bit 2 (`4`) `ORIGIN_TRUSTED`
(protocol 2 only: the player allowed the pinned origin for this server, the server allows the in-game display, the
client has one and the player has not disabled server-opened pages – i.e. a SHOW would be followed now). Mod 0.7.0
never sets bits 1 and 2.

### 4.4 `vystorm:web` – client → server

| ID | Name | Fields |
|---|---|---|
| `0x01` | HELLO | `VarInt minVersion` (1..255), `VarInt maxVersion` (≥ min, ≤ 255), `String(32) modVersion`, `String(16) loader` (`fabric`/`neoforge`), `VarInt capabilities` |
| `0x03` | OPEN_REQUEST | `VarInt requestId` (≥ 1, unique per connection), `String(32) module` (`[a-z0-9-]{1,32}` or `""` = overview), `VarInt target` (0 = in-game display, 1 = external browser, 2 = preload/hidden sign-in; mod 0.7.0 sends only 1) |
| `0x07` | CLOSED | `String(32) module`, `VarInt reason` (0..15; 0 = player, 1 = server; 2 = error and 3 = blocked navigation are reserved, 0.6.0 sent only 0 and 1, 0.7.0 sends none) |
| `0x08` | CLIENT_STATE | `VarInt flags` (0..0xFFFF, bits §4.3) |

- HELLO is sent **once per connection**.
- CLIENT_STATE is sent right after WELCOME (also with `flags = 0`) and on every change (checked at most once per second
  when nothing visible changes).
- Mod 0.7.0 sends OPEN_REQUEST only when the player presses the web key (Shift+menu key; the menu key alone on servers without panels): for a waiting SHOW/OPEN module (§4.7) or
  for `hotkeyModule`, always with `target = 1`. (0.6.0 and older also sent it for preloading right after WELCOME and
  for the JS bridge op `reauth`.)

### 4.5 `vystorm:web` – server → client

| ID | Name | Fields |
|---|---|---|
| `0x02` | WELCOME | `VarInt version` (1..255), `String(256) webBase` (`""` = no web), `String(32) hotkeyModule` (module ID or `""`), `VarInt serverFlags` |
| `0x04` | OPEN | `VarInt requestId` (`0` = server-initiated), `String(32) module`, `String(43) token`, `VarInt ttlSeconds` (1..600), `String(128) title` |
| `0x05` | DENIED | `VarInt requestId` (`0` = general), `VarInt reason` (1..255: 1 web off, 2 unknown module, 3 no permission, 4 rate limited, 5 other), `String(512) message` (plain text shown to the player; `""` = generic text) |
| `0x06` | CLOSE | `String(32) module` (`""` = any) |
| `0x09` | SHOW | `String(32) module`, `String(288) page` |

- **WELCOME** is accepted **once per connection**; later WELCOMEs are ignored. `webBase` pins the web origin for the
  connection. It must be `http(s)://host[:port]` – no path, query, fragment or user info, ASCII only, ≤ 256 bytes
  (a trailing `/` is tolerated). Host: lower-case `[a-z0-9.-]` labels or an IPv6 literal `[…]`. An unusable
  `webBase` disables the web part (native elements still work).
- **token**: one-time login token, exactly **43 characters** `[A-Za-z0-9_-]` (256-bit random, base64url without
  padding). The client uses it once, only in memory, and only as the URL fragment of `<webBase>/login#<token>`.
  Client-side validity: `min(ttlSeconds, 600)` seconds.
- **OPEN** with `requestId > 0` is only accepted if it matches an own pending request **with the same module**
  (pending ≤ 30 s, each ID once). `requestId = 0` (server-initiated) is only accepted with `SERVER_OVERLAY`, is
  rate-limited together with SHOW (0.5/s, burst 3) and is ignored if the player disabled server-opened pages. Mod 0.7.0
  always discards its token unused and only shows the "press <menu key>" hint; older mods used it only for an allowed origin.
- **title** is not shown by mod 0.7.0 (older mods showed it in the overlay's top bar).
- **DENIED** is shown to the player as a toast (`message`, or a generic text). DENIED for a hidden preload request
  (mods up to 0.6.0) is silent.

### 4.6 Pages (SHOW `page`)

`page` is a path on the pinned origin plus an optional fragment, e.g. `/app/skills#node/12` or `/m/shop/#npc/12`.
Allowed: `/[A-Za-z0-9/_=&.,:~+-]{0,159}(#[A-Za-z0-9/_=&.,:~+-]{0,127})?` and no `..`, no `//`, no query (`?`), no
`%`. `""` = the page the login redirect lands on. The server knows its page paths; the client never guesses. Only an
in-game display uses `page`; the system browser (mod 0.7.0) lands where the login redirect goes.

### 4.7 Flows

**Join**
```
Client                                   Server
  ── HELLO(1..2, caps = 62) ────────────▶  remember caps; pick version
  ◀──────────── WELCOME(2, webBase, hotkeyModule, OVERLAY|UI)
  ── CLIENT_STATE(flags) ────────────────▶  0.7.0: never ORIGIN_TRUSTED → use your normal menus
  ◀──────────── vystorm:ui PUT … (HUD, markers, tooltip rules)
```

**Web key (Shift+menu key; the menu key alone without panels)** – if a server SHOW or OPEN is waiting, the key opens that module; otherwise `hotkeyModule`. If the
origin is not trusted yet, the trust dialog opens (only on this key press). Then:
```
  ── OPEN_REQUEST(id, module, 1) ────────▶  same checks as a web login link
  ◀──────────── OPEN(id, module, token, ttl, title)
  system browser opens <webBase>/login#token → your /login page → your app page
```
Every key press fetches a fresh token; the mod keeps no session of its own.

**Server SHOW / OPEN with `requestId 0`** – mod 0.7.0 never opens the browser on server initiative. It remembers the
module, shows a toast "The server wants to show a page – press <menu key>" (at most 0.5/s, burst 3, only with `SERVER_OVERLAY`,
not if the player disabled server-opened pages) and discards any token from such an OPEN. Core only sends SHOW to
clients that announce `BROWSER_OVERLAY` and report `ORIGIN_TRUSTED`, so it never sends one to 0.7.0.

**CLOSE** – nothing to close in game; ignored by 0.7.0. **Disconnect** – the client drops all elements, downloaded
images and web state. The browser session stays in the player's browser; end sessions from mod tokens on quit (Core:
`web.client.end-sessions-on-quit`).

**0.6.0 and older** (optional embedded browser): preloaded a hidden browser with `target = 2` after WELCOME, showed
SHOW/K as an in-game overlay (`target = 0`) and sent `CLOSED` on Esc or CLOSE.

### 4.8 Client-side limits (`vystorm:web`)

- At most 4 pending requests; OPEN_REQUEST at most 0.5/s (burst 3); answers must arrive within 30 s.
- Server-initiated opens (SHOW and OPEN with `requestId 0`) at most 0.5/s (burst 3).
- At most 64 `vystorm:web` messages wait for the next client tick; more are dropped.

### 4.9 `vystorm:ui` messages

| ID | Direction | Name | Fields |
|---|---|---|---|
| `0x10` | S → C | PUT | `String(16384) json` – create or fully replace an element |
| `0x11` | S → C | PATCH | `String(4096) json` – `id` + top-level fields to replace (flat merge), then full validation; the merged element must stay ≤ 16 KiB |
| `0x12` | S → C | REMOVE | `VarInt n` (1..64), `n × String(64) id` (unknown IDs are ignored) |
| `0x13` | S → C | CLEAR | `String(64) prefix` – remove all elements whose ID starts with `prefix`; `""` = everything **including queued toasts** |
| `0x14` | S → C | TOAST | `String(4096) json` – one-off notification |
| `0x1F` | C → S | REJECT | `String(64) id` (`""` if unknown), `VarInt code` (1..255), `String(128) detail` |

- UI messages are ignored before WELCOME and when WELCOME lacked `SERVER_UI`.
- **Client rate:** 40 messages/s (burst 120), at most 64 processed per tick, queue ≤ 512. Excess is dropped and
  reported at most once per second as REJECT code 5 (`id ""`, detail `dropped N ui messages (rate limit)`).
- The client sends at most **50 REJECTs per connection**.
- **Server duty:** per player and element at most one PATCH per tick (merge changes within the tick); update bars and
  cooldowns at most ~4×/s.

---

## 5. Native elements (`vystorm:ui`) – JSON schema

### 5.1 General rules

- **Strict JSON**: no lenient syntax, no data after the object, nesting depth ≤ 8, the top level must be an object.
- **Unknown fields are ignored**, **unknown element types are rejected** (code 3). Numbers outside the allowed range
  are **rejected**, not clamped. Whole-number fields reject fractions (`5.5`). `null` counts as "not set".
- **`id`**: `[a-z0-9][a-z0-9_.:/-]{0,63}`, convention `<plugin>:<name>` (e.g. `myplugin:xp`) so that
  `CLEAR "myplugin:"` only hits your elements. One ID = one element; a PUT with an existing ID replaces it, even if the
  type changes.
- **Text** (`title`, `label`, `lines[i]`, `text`): a string, an object
  `{"text": "...", "color": <color>, "bold": bool, "italic": bool, "underlined": bool, "strikethrough": bool}`, or a
  list of those (≤ 16 segments, ≤ 256 characters in total). No click/hover actions. Control characters, `§`
  formatting codes, all Unicode format characters (bidi controls, zero-width, BOM, soft hyphen …) and line/paragraph
  separators are **removed**. Text is drawn with the element's text color unless a segment has a color; text is always
  drawn opaque.
- **Color**: `#RRGGBB`, `#AARRGGBB` (alpha first) or one of the 16 Minecraft names: `black`, `dark_blue`,
  `dark_green`, `dark_aqua`, `dark_red`, `dark_purple`, `gold`, `gray`, `dark_gray`, `blue`, `green`, `aqua`, `red`,
  `light_purple`, `yellow`, `white` (case-insensitive). `#RRGGBB` means fully opaque.
- **Icon** – an object with **exactly one** of:
  - `{"item": "minecraft:diamond_sword"}`, optional `"model": "ns:path"` (item model from the server resource pack –
    the way to show custom items). Unknown items show no icon.
  - `{"sprite": "minecraft:hud/heart/full"}` – a GUI sprite (vanilla or resource pack).
  - `{"image": "/path/icon.png"}` – a PNG from the **pinned web origin**. Path `/[A-Za-z0-9_./-]{1,160}\.png`, no `..`,
    no `//`. Loaded without cookies, without redirects, with normal TLS checks, response must be `200` with
    `Content-Type: image/png`, ≤ 64 KiB, ≤ 128×128 px, ≤ 64 images per connection, 4 downloads at a time. Loaded only
    if the player allowed the origin, or if the origin is `https` on the same host as the Minecraft server. Until then
    no icon is drawn.
  - Resource IDs (`item`, `model`, `sprite`, `world`, `match.item`): `[a-z0-9_.-]{1,64}:[a-z0-9_./-]{1,128}` without
    `..` (the namespace is required: `minecraft:stone`, not `stone`).
- **TTL**: `ttl` in ms, `0..3 600 000`; `0` = until REMOVE/CLEAR/disconnect. Counted from the moment the client
  **received** the element; every PUT and PATCH restarts it.

### 5.2 Common fields of HUD elements (`panel`, `bar`, `slots`)

| Field | Type | Default | Range |
|---|---|---|---|
| `anchor` | string | `top_left` | `top_left`, `top`, `top_right`, `left`, `center`, `right`, `bottom_left`, `bottom`, `bottom_right` |
| `x`, `y` | integer (GUI px) | `0` | −4000..4000 |
| `order` | integer | `0` | −1000..1000 (drawn in ascending order; higher = on top; ties: order of first PUT) |
| `ttl` | integer ms | `0` | 0..3 600 000 |

**Layout math** (GUI pixels; they scale with the player's GUI scale; `W`,`H` = screen, `w`,`h` = element size):

| Anchor column | left edge | Anchor row | top edge |
|---|---|---|---|
| `*_left`, `left` | `x` | `top*` | `y` |
| `top`, `center`, `bottom` | `(W − w) / 2 + x` | `left`, `center`, `right` | `(H − h) / 2 + y` |
| `*_right`, `right` | `W − w − x` | `bottom*` | `H − h − y` |

So `x`/`y` point **inwards** from edges (positive `x` on a right anchor moves left; positive `y` on a bottom anchor
moves up); for the centered axis, positive moves right/down. Example: `{"anchor":"bottom","y":52}` centers an
element horizontally with its bottom edge 52 px above the screen bottom, just above the vanilla health/hunger rows.

HUD elements are drawn before the chat layer. The player can hide all native elements (and tooltip lines) with **H**.

### 5.3 `panel`

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `width` | integer | `0` | `0` = automatic (widest line + padding, max 320), otherwise 16..320; with a fixed width lines are wrapped |
| `padding` | integer | `4` | 0..16 |
| `background` | color | `#90000000` | fully transparent = no background |
| `border` | color | none | 1 px outline |
| `textColor` | color | `#FFFFFFFF` | |
| `title` | text | – | drawn first; with `icon` a 10 px icon is drawn left of it |
| `lines` | list of text | – | ≤ 12 entries; after wrapping at most 24 rows are drawn; `""` = empty row |
| `icon` | icon | – | only drawn next to the `title` |

A panel needs a non-empty `title` or at least one line. Height = `2·padding + (title ? 10, or 12 with icon : 0) +
rows·10 − 1`.

### 5.4 `bar`

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `width` | integer | `182` | 8..400 |
| `height` | integer | `5` | 2..16 |
| `value` | number | `0` | 0..1e12 |
| `max` | number | `1` | > 0 .. 1e12; fill = `clamp(value / max, 0, 1)` |
| `color` | color | `#FF80FF20` | fill color |
| `background` | color | `#C0000000` | |
| `textColor` | color | `#FFFFFFFF` | label color |
| `label` | text | – | centered **above** the bar (element grows by 10 px), or **inside** when `height ≥ 9` |
| `segments` | integer | `0` | 0..20; > 1 draws `segments − 1` separator lines like the vanilla XP bar |

### 5.5 `slots`

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `size` | integer | `20` | 16..32 (square slot size; icons are 16 px from size 20, else `size − 2`) |
| `gap` | integer | `2` | 0..16 |
| `slots` | list | **required** | 1..12 slot objects, drawn left to right |

Slot object:

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `icon` | icon | – | |
| `key` | string | `""` | ≤ 4 characters (after sanitizing), drawn top-left, e.g. `"R"` |
| `count` | integer | `0` | 0..999, drawn bottom-right when > 0 |
| `cooldownMs` | integer | `0` | 0..3 600 000, **remaining** cooldown at the moment of receipt |
| `cooldownTotalMs` | integer | = `cooldownMs` | 0..3 600 000, must be ≥ `cooldownMs`; used for the dark sweep fraction |
| `active` | bool | `false` | gold outline (e.g. skill currently running) |
| `label` | text | – | validated but **not drawn** in 0.6.0 (reserved) |

The client counts cooldowns down itself (seconds shown, rounded up). **Any PUT or PATCH of the element restarts the
countdown from the `cooldownMs` in the (merged) element** – when you patch a `slots` element, always send the current
remaining cooldowns.

### 5.6 `marker` (world marker)

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `x`, `y`, `z` | number | **required** | −3e7..3e7 (block coordinates) |
| `world` | resource ID | any | dimension, e.g. `minecraft:overworld`, `minecraft:the_nether`; not set = shown in every dimension |
| `label` | text | – | drawn above the point on a dark background |
| `color` | color | `#FFFFD24A` | point and label color |
| `icon` | icon | – | 16 px above the point |
| `maxDistance` | number | `256` | 1..1024 blocks; farther away = hidden |
| `showDistance` | bool | `true` | `"<n> m"` below the point |
| `clampToScreen` | bool | `true` | off-screen/behind markers stick to the screen edge |
| `ttl` | integer ms | `0` | 0..3 600 000 |

Markers have no `anchor`/`order`: `x`/`y`/`z` are world coordinates, projected from the camera every frame.

### 5.7 `tooltip` (tooltip rule)

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `match` | object | **required** | needs `item` and/or `customData` |
| `match.item` | resource ID | – | item type, e.g. `minecraft:diamond_sword` |
| `match.customData.path` | string | – | path into the item's `minecraft:custom_data`, dot-separated, ≤ 4 segments of `[A-Za-z0-9_:-]{1,64}`, ≤ 128 chars, e.g. `PublicBukkitValues.myplugin:id` (Bukkit PDC keys live under `PublicBukkitValues`) |
| `match.customData.equals` | string | – | ≤ 128 chars; matches a **string** tag with exactly this value |
| `lines` | list of text | **required** | 1..8 lines, **appended** to the tooltip (never replace item data) |
| `ttl` | integer ms | `0` | 0..3 600 000 |

If both `item` and `customData` are given, both must match. All matching rules add their lines (in the order the
rules were first created).

### 5.8 TOAST

| Field | Type | Default | Range / notes |
|---|---|---|---|
| `title` | text | **required** | non-empty |
| `text` | text | – | second line |
| `icon` | icon | – | |
| `durationMs` | integer | `5000` | 1000..10000, multiplied by the player's "Notification Time" setting |
| `style` | string | `info` | `info`, `success`, `warning`, `error` (accent color) |

Toasts use Minecraft's toast system (top right, stacked with vanilla toasts, never overlapping). At most 5 wait in the
element store, 3 are shown at once and up to 8 more wait in the toast queue. While the trust dialog is open, Vystorm
toasts are held back (so servers cannot cover its security warnings).

### 5.9 Limits per connection

| What | Limit | Over the limit |
|---|---|---|
| HUD elements (`panel` + `bar` + `slots`) | 32 | REJECT code 2 |
| Markers | 64 | REJECT code 2 |
| Tooltip rules | 128 | REJECT code 2 |
| PUT JSON / merged element | 16 KiB | rejected |
| PATCH JSON | 4 KiB | message dropped |
| TOAST JSON | 4 KiB | message dropped |

### 5.10 PATCH semantics

1. The JSON must contain `id` of an existing element (else REJECT 4 `no element with this id`).
2. Every other top-level field replaces the stored field (flat merge – a patched `lines` or `slots` list replaces the
   whole list; nested objects are not merged).
3. `type` may be repeated but not changed (REJECT 1 `type cannot change`).
4. The merged element is validated like a PUT and must stay ≤ 16 KiB.
5. The element's receive time is reset (TTL and cooldowns restart).

### 5.11 Examples

```json
{"id":"myplugin:xp","type":"bar","anchor":"bottom","y":52,"width":182,"height":5,
 "value":340,"max":1000,"color":"#FF7CFC00","label":"Lv 12 · 340/1000 XP","segments":10}
```
PATCH: `{"id":"myplugin:xp","value":347,"label":"Lv 12 · 347/1000 XP"}`

```json
{"id":"myplugin:skills","type":"slots","anchor":"bottom_right","x":6,"y":6,"slots":[
 {"icon":{"item":"minecraft:blaze_powder"},"key":"R","cooldownMs":8000,"cooldownTotalMs":10000},
 {"icon":{"image":"/m/items/icons/flame.png"},"key":"G","count":3,"active":true}]}
```
```json
{"id":"myplugin:quest","type":"panel","anchor":"right","x":4,"width":140,"border":"#80FFD24A",
 "title":[{"text":"Quest: ","color":"gold","bold":true},"The Old Mine"],
 "lines":["Collect iron ore: 7/20",{"text":"Return to the smith","color":"gray","italic":true}],
 "icon":{"item":"minecraft:iron_pickaxe"}}
```
```json
{"id":"myplugin:home","type":"marker","world":"minecraft:overworld","x":120.5,"y":72,"z":-44.5,
 "label":"Home","color":"aqua","maxDistance":1024,"icon":{"sprite":"minecraft:hud/heart/full"}}
```
```json
{"id":"myplugin:tip-flame","type":"tooltip",
 "match":{"item":"minecraft:blaze_rod","customData":{"path":"PublicBukkitValues.myplugin:id","equals":"flame_wand"}},
 "lines":[{"text":"Right-click: fireball","color":"gold"}]}
```
TOAST: `{"title":"Level 13","text":"+1 talent point","style":"success","icon":{"item":"minecraft:experience_bottle"}}`

---

## 6. Web pages – what your web pages must do

Applies to Core modules and to your own backend on path B. Since mod 0.7.0 pages open in the **player's system
browser** as a normal tab – there is no embedded browser any more (0.6.0 and older had an optional Chromium overlay).

### 6.1 Sign-in contract

1. The client opens `<webBase>/login#<token>` in the system browser (the token is only in the **fragment** – it never
   goes over the network, into server logs or the Referer).
2. Your `/login` page reads `location.hash`, removes it immediately (`history.replaceState`), exchanges the token for a
   session (e.g. `POST` → `HttpOnly; SameSite=Strict; Secure` cookie) and redirects to an app page **on the same
   origin**.
3. Treat the token like Core does: 256-bit random, single use, short TTL (Core: 60 s; client caps at 600 s), same
   permission checks as any other login link. Sessions created from mod tokens should end when the player quits (the
   mod cannot delete cookies in the player's browser).

URL map with Core: login `<webBase>/login#<token>`; module start page = where Core's login redirect lands
(`/m/<id>/` for modules with own files, `/app/<id>` for the bundled web UI); image icons = `<webBase><path>` (public
static files).

### 6.2 Page design

The page is an ordinary browser tab: normal browser security (same-origin policy, certificate checks, popup blocker)
applies, and so does your own CSP (Core: `default-src 'self'` …). Use a valid certificate; with `http` the player gets
a warning in the trust dialog. There are no in-game restrictions on size, input or zoom.

### 6.3 JS bridge (removed in 0.7.0)

Mods up to 0.6.0 offered `window.vystormQuery` (`hello`, `close`, `sound`, `reauth`) inside their overlay. It does not
exist in a normal browser, so pages always had to work without it. Calls like `window.vystormQuery?.(…)` simply do
nothing now.

---

## 7. Trust and security model

- **Origin pinning:** the web origin comes only from the first WELCOME of the connection – never from chat, commands,
  pages or configuration.
- **Trust on first use, per Minecraft server address:** before anything is loaded the player sees
  "Open this server's web pages?" with the server address and the web origin, and must click **Allow**
  (buttons react after 1 s). The dialog only opens when the **player** presses the web key. The decision is stored
  in `config/vystorm_client/trust.json` (key = server address, port 25565 added). If a known server later sends a
  **different origin**, the player gets a warning and is asked again.
- **Extra warnings in the dialog:** plain `http` to anything other than loopback/private IP literals (separate consent
  is stored); origins in the player's own computer/local network (loopback, private, CGNAT, link-local, `localhost`,
  `*.localhost`, `*.local`) when the Minecraft server itself is not there. `http` needs no extra consent only for
  `localhost`, `127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `100.64/10`, `[::1]` and `[fd…]` literals.
- **Without consent** nothing is opened and no token is used. Server-initiated SHOW/OPEN never open the browser (mod
  0.7.0 discards such tokens). The trust bit in CLIENT_STATE is only a hint for the server (0.7.0 never sets it); a
  faked bit only makes the server send a SHOW that the real client does not follow.
- **The browser's address bar shows the origin** the player allowed.
- **Tokens:** exactly one use, memory only, never in logs, chat, crash reports, config or screenshots.
- **What the mod tells the server:** mod version, loader and capabilities (HELLO), the state bits (hidden HUD; up to
  0.6.0 also overlay open and origin trusted), OPEN_REQUESTs on key press, REJECTs for invalid elements. Nothing
  else – no other mods, no files, no system data. Image icons are fetched from the pinned origin with the user agent
  `VystormClient/<version>` and without cookies.
- **Server obligations:** never derive authority from these messages; same checks for OPEN_REQUEST as for a login
  link (web on, permission, module exists, module permission); rate-limit per player (HELLO once per connection,
  OPEN_REQUEST ≤ 10/min burst 3, CLOSED/CLIENT_STATE ≤ 2/s, REJECT ≤ 5/s); strict decoding (≤ 1 KiB, right direction,
  no trailing bytes; ignore a player after 20 malformed messages); only send to players who sent HELLO; `webBase` = the
  public URL of your web platform and nothing else; never log tokens.

---

## 8. Path B – raw plugin messages without Core

Use this only if you cannot use Vystorm Core. You implement the handshake, validation, rate limits and (for web
pages) a web backend yourself.

### 8.1 Minimal codec (Java)

```java
final class VWire {
    static final class Out {
        final java.io.ByteArrayOutputStream b = new java.io.ByteArrayOutputStream();
        Out varInt(int v) { while ((v & ~0x7F) != 0) { b.write((v & 0x7F) | 0x80); v >>>= 7; } b.write(v); return this; }
        Out str(String s, int maxBytes) {
            byte[] u = s.getBytes(java.nio.charset.StandardCharsets.UTF_8);
            if (u.length > maxBytes) throw new IllegalArgumentException("too long");
            varInt(u.length); b.write(u, 0, u.length); return this;
        }
        byte[] bytes() { return b.toByteArray(); }
    }
    static final class In {
        final byte[] d; int p;
        In(byte[] d) { this.d = d; }
        int varInt() {
            int v = 0;
            for (int i = 0; i < 5; i++) { int x = d[p++] & 0xFF; v |= (x & 0x7F) << (7 * i); if ((x & 0x80) == 0) return v; }
            throw new IllegalArgumentException("varint too long");
        }
        String str(int maxBytes) {
            int n = varInt();
            if (n < 0 || n > maxBytes || n > d.length - p) throw new IllegalArgumentException("bad string");
            String s = new String(d, p, n, java.nio.charset.StandardCharsets.UTF_8); p += n; return s;
        }
        void end() { if (p != d.length) throw new IllegalArgumentException("trailing bytes"); }
    }
}
```

(Reading past the end throws `ArrayIndexOutOfBoundsException` – catch it together with `IllegalArgumentException`.
For production, also reject invalid UTF-8 strictly.)

### 8.2 Minimal Paper plugin: handshake + native elements only (no web)

```java
public final class VystormUiBridge extends JavaPlugin implements PluginMessageListener, Listener {
    static final String WEB = "vystorm:web", UI = "vystorm:ui";
    static final int SERVER_UI = 2, CAP_HUD = 4, CAP_TOASTS = 8;
    private final Map<UUID, Integer> caps = new HashMap<>();          // players who sent HELLO

    @Override public void onEnable() {
        for (String c : List.of(WEB, UI)) {
            getServer().getMessenger().registerOutgoingPluginChannel(this, c);
            getServer().getMessenger().registerIncomingPluginChannel(this, c, this);
        }
        getServer().getPluginManager().registerEvents(this, this);
    }

    @Override public void onPluginMessageReceived(String channel, Player player, byte[] msg) {
        if (msg.length > 1024) return;
        try {
            VWire.In in = new VWire.In(msg);
            int id = in.varInt();
            if (channel.equals(WEB) && id == 0x01) {                   // HELLO
                int min = in.varInt(), max = in.varInt();
                in.str(32); in.str(16);                                // modVersion, loader (display only)
                int capabilities = in.varInt(); in.end();
                if (caps.containsKey(player.getUniqueId())) return;    // once per connection
                int version = Math.min(max, 2);
                if (version < Math.max(min, 1)) return;                // no common version: stay silent
                caps.put(player.getUniqueId(), capabilities);
                // WELCOME: no web (empty base), no hotkey module, native elements on
                player.sendPluginMessage(this, WEB, new VWire.Out().varInt(0x02).varInt(version)
                        .str("", 256).str("", 32).varInt(SERVER_UI).bytes());
                if ((capabilities & CAP_HUD) != 0) put(player, """
                        {"id":"mybridge:hello","type":"panel","anchor":"top_right","x":4,"y":4,
                         "title":{"text":"Welcome!","color":"gold"},"lines":["Your HUD works."],"ttl":15000}""");
            } else if (channel.equals(UI) && id == 0x1F) {             // REJECT (rate-limit this log!)
                String elementId = in.str(64); int code = in.varInt(); String detail = in.str(128); in.end();
                getLogger().fine("client rejected " + elementId + " code " + code + ": " + detail);
            }
            // CLIENT_STATE (0x08) etc.: read if you need them; ignore unknown IDs
        } catch (RuntimeException malformed) {
            // count per player; ignore the player after 20
        }
    }

    void put(Player p, String json) {                                  // PUT 0x10
        p.sendPluginMessage(this, UI, new VWire.Out().varInt(0x10).str(json, 16384).bytes());
    }
    void patch(Player p, String json) {                                // PATCH 0x11 – max one per element per tick
        p.sendPluginMessage(this, UI, new VWire.Out().varInt(0x11).str(json, 4096).bytes());
    }
    void toast(Player p, String json) {                                // TOAST 0x14 – needs CAP_TOASTS
        if ((caps.getOrDefault(p.getUniqueId(), 0) & CAP_TOASTS) != 0)
            p.sendPluginMessage(this, UI, new VWire.Out().varInt(0x14).str(json, 4096).bytes());
    }
    void remove(Player p, String... ids) {                             // REMOVE 0x12 (1..64 IDs)
        VWire.Out o = new VWire.Out().varInt(0x12).varInt(ids.length);
        for (String id : ids) o.str(id, 64);
        p.sendPluginMessage(this, UI, o.bytes());
    }
    void clear(Player p, String prefix) {                              // CLEAR 0x13
        p.sendPluginMessage(this, UI, new VWire.Out().varInt(0x13).str(prefix, 64).bytes());
    }

    @EventHandler public void onQuit(PlayerQuitEvent e) { caps.remove(e.getPlayer().getUniqueId()); }
}
```

Build JSON with a real JSON library (Gson ships with Paper) instead of string concatenation for user-provided text.

### 8.3 Other server platforms

Any platform works that (1) announces `vystorm:web` and `vystorm:ui` to the client via `minecraft:register`,
(2) can send and receive raw custom payload bytes on those channels during the play phase, and (3) follows the rules
above. On Fabric/NeoForge servers register both channels as raw byte payloads (the whole payload is the message).

### 8.4 Adding web pages on path B

1. WELCOME with `webBase = "https://your.web.host"` and `serverFlags = SERVER_UI` (2; add `SERVER_OVERLAY` for mods
   with an in-game display), and a `hotkeyModule` your backend understands (`[a-z0-9-]{1,32}` or `""`).
2. On OPEN_REQUEST: check rate limits and permissions exactly like your normal web login, create a one-time token
   (43 chars base64url, 256-bit random, TTL ≤ 600 s), answer `OPEN(requestId, module, token, ttl, title)` – `module`
   must equal the request's module – or `DENIED(requestId, reason, message)`.
3. Implement the sign-in contract (§6.1).
4. SHOW only when `BROWSER_OVERLAY` was announced and `ORIGIN_TRUSTED` is set (protocol 2), rate-limit it (0.5/s), and
   fall back to your normal menu – mod 0.7.0 never qualifies.
5. End sessions created from mod tokens when the player quits.

---

## 9. Native panels (`vystorm:panel`)

Since mod **0.8.0** a server can describe a whole menu as JSON (a "panel") and the mod shows it as a real Minecraft
screen: scrolling, hover tooltips, keyboard focus, real item icons, Vystorm or vanilla look. No browser, no download, no
web login. Actions go back as plugin messages; **the server checks every single one** against what it sent. Panels work
even when the web platform is off.

### 9.1 Path A – Core API (Vystorm Core 0.25.1+, capability `panels`)

```java
Panels panels = Panels.of(plugin);                         // namespace "<plugin>:"
panels.register("display", PanelSpec.builder("Display")
        .access(p -> true)                                 // checked on open AND on every action
        .dialogFallback(true)                              // players without the mod get an automatic Paper dialog
        .render(ctx -> Panel.page("Display").size(Panel.Size.MEDIUM).children(
            Ui.section("HUD").children(
                Ui.toggle("hud", "Show HUD").value(prefs.hud(ctx.player()))
                    .onChange((a, on) -> { prefs.setHud(a.player(), on); return Result.ok(); }),
                Ui.slider("range", "Range").range(4, 64, 4).value(store.range(ctx.player())).format("{v} blocks")
                    .onChange((a, v) -> v > store.maxRange() ? Result.error("Too far") : ok(store.setRange(a.player(), v.intValue()))))))
        .build());

panels.open(player, "display");                            // NATIVE / DIALOG / WEB / FALLBACK / NONE
```

A list with actions – the object behind a button lives in the handler's closure, the client only sends element IDs:

```java
panels.register("mine", PanelSpec.builder("My portals")
        .access(p -> p.hasPermission("betterportals.use"))
        .menu(MenuCategory.WORLD, Material.OBSIDIAN, "Your portals")   // entry in /vmenu
        .web("portals", "")                                             // "Open in browser" + link fallback
        .fallback(this::openChestMenu)                                  // your own chest menu has priority
        .render(ctx -> Panel.page("My portals").size(Panel.Size.LARGE).children(
            Ui.table("list")
                .column("name", "Name").sortable()
                .column("act", "").columnWidth(48)                          // width of the last column
                .rows(store.byOwner(ctx.player().getUniqueId()), g -> Ui.row(g.id())
                    .cell(g.name())
                    .cell(Ui.button("rm").icon(Ui.item(Material.BARRIER)).danger()
                        .confirm("Remove portal?", g.name() + " will be deleted.")
                        .onClick(a -> remove(a, g.id()))))        // the ID is in the closure
                .empty("You have no portals yet.")))
        .build());
```

Handlers run on the server thread, return `Result.ok()`, `Result.ok(msg)`, `Result.error(msg)` or
`Result.fieldErrors(map)`, and call `a.refresh()` to re-render: Core diffs the new tree against the old one by ID and
sends a PATCH. Full reference: Core `docs/API.md` §18.

### 9.2 Path B – raw messages

Handshake (after the normal `vystorm:web` handshake):

1. The client's HELLO has capability bit 7 `PANELS` (128); your WELCOME sets `serverFlags` bit 2 `SERVER_PANELS` (4)
   and you register `vystorm:panel`.
2. The client sends **P_HELLO** `0x20` (`schemaMin`, `schemaMax`, `features`, `cacheKiB`); you answer **P_WELCOME**
   `0x30` (`schema = 1`, `features` = intersection, `hotkeyPanel` = panel ID the menu key opens, or empty to keep the menu key on the
   web page).
3. The client asks with **P_REQUEST** `0x21` (`requestId`, `panelId`, `arg`; empty `panelId` = your hotkey panel); you
   answer **P_OPEN** `0x31` or **P_DENIED** `0x37`. You may also open on your own initiative (`requestId = 0`): while
   one of your panels is open the client always follows; otherwise at most 0.5×/s, only with the player's
   `allowServerOpen` and only when no other screen is open. **An OPEN the client does not show is answered with
   P_CLOSED** (reason 0 refused, 1 another screen, 2 broken) – forget that session, keep the one it would have replaced.
4. Clicks and changes arrive as **P_ACTION** `0x22` (`session`, `seq`, `elementId`, `kind`, `value` JSON); answer
   **P_RESULT** `0x33` (`status` 0 ok, 1 error, 2 denied, 3 stale, 4 rate-limited; JSON `{"message":…,
   "fields":{id:text}}`) – Core answers every action except an exact duplicate `seq` – and optionally **P_PATCH**
   `0x32`. Patches of a session must have `rev` = previous + 1 (the first after an OPEN sets the count); on a gap the
   client sends **P_ERROR** with `detail` starting `resync:` and ignores patches until you send a full OPEN (flag bit 2).
   Bump `rev` on every full OPEN as Core ≥ 0.25.2 does, so a lost OPEN shows up as a gap; P_REQUEST is limited to 2/s
   (burst 6) on both sides.
5. **P_CLOSED** `0x23` / **P_CLOSE** `0x34` end sessions; **P_ERROR** `0x2F` reports documents or elements the client
   refused (codes 1 invalid, 2 limit, 3 unsupported, 4 unknown id, 5 timeout, 6 transfer).

Transfers: documents/patches with raw UTF-8 ≤ 8 KiB go inline; bigger ones as **P_BEGIN** `0x35` + **P_CHUNK** `0x36`
(Deflate, chunks ≤ 30 KiB, index 0, 1, … without gaps, SHA-256 over the **raw** bytes, inflated size ≤ `rawBytes` and
ratio ≤ 1:64, at most 2 at a time, 10 s timeout). The easiest way to get the binary format right is to use the classes
in `protocol/src/main/java/at/esoren/vystorm/client/protocol/panel` (`PanelMessage`, `Transfer`, `PanelSchema`,
`PanelPatch`, `ActionCheck`); they are plain Java (Gson only). Byte-exact field table: `PROTOCOL.md` §8.

### 9.3 JSON schema v1 (short reference)

Document: `{"schema":1, "panel":"<ns>:<name>", "root":{"type":"page", …}}`. Strict JSON, limits are rejected (never
truncated), unknown fields are ignored. IDs `[a-z0-9][a-z0-9_.:/-]{0,63}`, required for everything interactive and
everything you want to patch. Common fields: `id`, `tooltip` (≤ 8 texts), `visible`, `disabled`, `width` (`auto`,
pixels, `"50%"`, `fill`), `minWidth`, `maxWidth`, `grow` (0..10), `align`. Texts are the `vystorm:ui` text format
(§5.1) with ≤ 64 segments and ≤ 2048 characters.

| Type | Purpose | Main fields | Action |
|---|---|---|---|
| `page` | root | `title`, `subtitle`, `icon`, `size` (`small`/`medium`/`large`/`full`), `theme`, `accent`, `web {module,page}`, `children`, `footer` | – |
| `section` | card/group | `title`, `variant` (`card`,`plain`,`notice`,`warn`,`error`), `collapsible`, `collapsed`, `children` | – |
| `row` / `col` | flex container | `gap`, `pad`, `justify`, `wrap` (row), `scroll` (`none`/`y`/`x`), `maxHeight`, `children` | – |
| `grid` | grid | `columns` (1..12) or `cell` (min width 16..400), `gap`, `children` | – |
| `tabs` | tabs | `tabs:[{id,label,icon,badge,children,lazy}]` (≤ 16), `selected` | TAB (lazy only) |
| `heading`, `text` | text | `text`/`lines`, `level` 1..3, `style`, `maxLines`, `wrap` | – |
| `kv` | key/value list | `rows:[{key,value,icon,tooltip}]` (≤ 64), `keyWidth` | – |
| `table` | table | `columns` (≤ 16) `[{id,label,width,align,sortable}]`, `rows` (≤ 2000) `[{key,cells,tooltip}]`, `sort`, `rowLines`, `paging {page,pages}`, `select`, `empty` | SORT/PAGE (paging), SELECT |
| `list` | list | `items:[{key,icon,title,subtitle,right,tooltip,badge}]` (≤ 2000), `select`, `empty` | SELECT |
| `button` | button | `label`, `icon`, `style`, `confirm {title,text,yes,no}`, exactly one of `action:true`, `navigate {panel,arg}`, `web {module,page}` | CLICK |
| `toggle` | switch | `label`, `description`, `value` | CHANGE |
| `slider` / `number` | number | `label`, `min`, `max`, `step`, `value`, `format` (`{v}`), `decimals`, `live` | CHANGE |
| `input` | text field | `label`, `value`, `placeholder`, `maxLength` ≤ 256, `chars` (`any`,`name`,`integer`,`decimal`), `submit`, `search` | CHANGE / SEARCH |
| `select` | choice | `label`, `options` (≤ 256) `[{id,label,icon,tooltip}]`, `value`, `style` (`dropdown`/`segmented`), `search` | CHANGE |
| `form` | groups inputs | `children`, `submit` (button text) | SUBMIT |
| `progress`, `timer` | bar, countdown | `value`/`max`/`label`/`color`/`segments`; `remainingMs`/`totalMs`/`format`/`bar` | – |
| `item`, `image` | item icon, image | `icon`, `count`, `size`, `slot`, `rarity`; `src {sprite\|image}`, `width`, `height` | CLICK (item, optional) |
| `badge`, `swatch`, `divider`, `spacer` | small parts | `text`, `color`, `size` | – |
| `chart`, `graph` | not in 0.8.0 | – | shown as "not supported" + P_ERROR 3 |

Table cells: plain text, `{text,sort}`, or a `button`/`badge`/`item`/`row` element. Core prefixes cell elements with
`<tableId>/<rowKey>/<id>`. Tables without `paging` are sorted on the client (no network).

**Patches** `{"rev":n,"ops":[…]}`: `set` (merge top-level fields), `replace`, `insert` (`parent`,`index`,`element`),
`remove` (`id`), `rows` (tables/lists: `upsert`, `remove`, `order` by key). The root can be addressed as `root`. The
patched document is validated in full again. Patches, re-layouts and results never overwrite what the player has typed
or set but not sent yet (text being edited, unsent number/slider changes, form values until SUBMIT).

**Action values** (`P_ACTION.kind`): 1 CLICK `""`, 2 CHANGE `true`/number/`"text"`/`"optionId"`, 3 SUBMIT
`{"<id>":value,…}` (disabled/hidden fields are not sent; Core ignores them), 4 SORT `{"column":"<id>","dir":"asc"|"desc"}`, 5 PAGE number, 6 TAB `"tabId"`, 7 SELECT
`"rowKey"`, 9 SEARCH `"text"` (debounced 300 ms).

### 9.4 Trust rules

- The client never runs code from a panel: no scripts, no expressions, no commands, no click events in texts. The only
  effects are P_ACTION, P_REQUEST (`navigate`), P_CLOSED and "Open in browser" (existing trust dialog, pinned origin).
- **Check every action on the server** against the document you sent (Core does this with the same `ActionCheck`
  class): element exists, is interactive, visible, not disabled, `kind` fits, value within the limits you sent, only
  known form fields, `seq` not seen before; rate limits (Core: 20/s, burst 40 per player – checked first, before any
  "stale" answer – and 10/s per element, created only for elements that exist in the document).
- Actions carry element IDs only. Never put object IDs in element IDs and trust them – keep the object in the handler.
- `navigate` arguments come back from the client as P_REQUEST `arg`: treat them as user input (check ownership).
- Every panel shows a header with the server address; inputs are marked "→ sent to the server"; Esc always closes.

## 10. Errors, rejections and troubleshooting

**REJECT codes** (`vystorm:ui` 0x1F; Core: `VystormClientUiRejectEvent`)

| Code | Name | Typical `detail` (field names only, never content) |
|---|---|---|
| 1 | INVALID | `json syntax`, `json nested too deeply`, `id invalid`, `<field> out of range`, `<field> must be a whole number`, `<field>.color invalid`, `<field> needs exactly one of item/sprite/image`, `panel needs title or lines`, `match needs item or customData`, `type cannot change`, `element too large after patch` |
| 2 | LIMIT | `too many <type> elements` (32 HUD / 64 markers / 128 tooltips) |
| 3 | UNKNOWN_TYPE | `unknown type` – typo, or the element type is newer than the player's mod |
| 4 | UNKNOWN_ID | `no element with this id` – PATCH after REMOVE/CLEAR/TTL expiry/reconnect → send PUT |
| 5 | RATE | `dropped N ui messages (rate limit)` – slow down, batch PATCHes |

**Symptoms**

| Symptom | Likely cause |
|---|---|
| No HELLO arrives | Player has no mod; channels not registered/announced; server is not in play phase yet (client waits up to 60 s after join) |
| HELLO arrives, nothing shows | WELCOME not sent or wrong version; `SERVER_UI` missing; message > 32 KiB; player pressed H (`UI_HIDDEN`); element anchored off-screen; REJECTs |
| The menu key says "This server does not support Vystorm Client" | No WELCOME in this connection |
| The menu key says "This server has no web interface enabled" | WELCOME with empty or invalid `webBase` (the client log names the reason for an invalid one) |
| "The server wants to show a page – press <menu key>" | SHOW/OPEN from the server (mod 0.7.0 never opens pages on server request) |
| Browser opens, but sign-in fails | Token rejected or expired (TTL), `/login` does not handle the fragment, page not reachable or certificate invalid |
| "Opening web pages is turned off" | The player set `externalBrowserFallback: false` |
| Image icon missing | Origin not allowed yet; not `200` + `image/png`; > 64 KiB or > 128 px; redirect; > 64 images |
| `show` always `false`, menu instead of page | Expected with mod 0.7.0 (no `BROWSER_OVERLAY`); players press the menu key for the web interface |

Protocol violations on `vystorm:web` are dropped silently (after 20 the protocol is off for the connection). DENIED
messages are shown as a toast.

## 11. Player settings that affect your integration

`config/vystorm_client/client.json` (invalid values fall back to defaults):

| Key | Default | Effect for servers |
|---|---|---|
| `hudVisible` | `true` | `false` = native elements and tooltip lines hidden (key H; reported as `UI_HIDDEN`) |
| `allowServerOpen` | `true` | `false` = SHOW and server-initiated OPEN are ignored (otherwise they only show a "press <menu key>" hint) |
| `externalBrowserFallback` | `true` | open pages in the system browser (historic name); `false` = the web key opens nothing |
| `perfLog` | `false` | timing log every 5 s |
| `nativePanels` | `true` | `false` = the mod does not announce `PANELS`, ignores `vystorm:panel`, and the menu key opens the web page as before |
| `panelTheme` | `"server"` | panel look: `server` (what the panel asks for), `vystorm` or `vanilla`; display only |

Keys of 0.6.0 and older (`preloadBrowser`, `browserFps`, `keepWarmMinutes`, `pageZoom`, `hardenRinku`) are ignored.

`trust.json` holds the per-server consents (delete to reset); `binds.json` holds the player's own command keys (the
server can neither read nor trigger them).

Keys (changeable under Controls → Vystorm): **menu key** – default **`** (the key left of 1; **^** on German keyboards;
**K** until 0.8.2, which Iris uses for "Toggle shaders") – opens the server's menu panel (with Core 0.25.0+; otherwise the
web interface in your browser), **Shift+menu key** always the web interface in your browser, **H** show/hide server
HUD elements, **J** command keys menu. Players who already played with an older version keep the key saved in their
`options.txt`. Hints like "press …" always name the key the player actually bound.

## 12. Versions and compatibility

| Mod | Protocol (HELLO) | Notes |
|---|---|---|
| 0.8.3 | 1..2 | default menu key `` ` `` instead of K (Iris), trust dialog text fits at every GUI scale, toast margin; protocol unchanged |
| 0.8.2 | 1..2 | panels: P_REQUEST 2/s like Core, `resync:` with its own budget (also for a broken full OPEN), form fields changed after SUBMIT are kept, timers anchored at receive time |
| 0.8.1 | 1..2 | panels: crash-safe tables and screen, patches in order with `rev` check (`resync:`), refused OPENs answered with P_CLOSED, form input survives patches/resize |
| 0.8.0 | 1..2 | native panels: capability bit 7 `PANELS` (190 in total), `vystorm:panel`, K opens the hotkey panel, Shift+K the web page |
| 0.7.0 | 1..2 | no embedded browser: pages open in the system browser on key press; never sets `BROWSER_OVERLAY`, `JS_BRIDGE`, `OVERLAY_OPEN`, `ORIGIN_TRUSTED`; OPEN_REQUEST only with `target = 1` |
| 0.6.0 | 1..2 | public release; DENIED for hidden preload is silent |
| 0.5.0 | 1..2 | command keys (client-only feature) |
| 0.4.0 | 1..2 | protocol 2: `ORIGIN_TRUSTED`; toasts via Minecraft's toast manager |
| 0.3.x | 1..1 | |

| Vystorm Core | Client support |
|---|---|
| 0.19.0 | `vystorm:web`, `ClientServices` (capability `web-client`) |
| 0.20.0 | `vystorm:ui`, `ClientUi`, `ClientElement`, events (capability `web-client-ui`) |
| 0.21.0 | protocol 2, `STATE_ORIGIN_TRUSTED`, `originTrusted` (capability `web-client-trust`) |
| 0.23.0 | first public release of Core (recommended minimum) |
| 0.25.0 | `vystorm:panel`, `panel.Panels`/`PanelSpec`/`Ui`, settings and `/vmenu` as panels |
| 0.25.2 | refresh results only for the visible session, big opens never overtaken, `resync:` for any session (≤ 1/s), `rev` bumped on full OPENs, patches > 128 KiB as full OPEN |
| 0.25.1 | panel API with typed builders (capability `panels` since 0.25.1), `resync:` handling, replaced panels kept until P_CLOSED |

Runtime: Minecraft 26.2, Fabric Loader ≥ 0.19.5, Fabric API ≥ 0.161.0+26.2, Java 25. Nothing else (0.6.0 and older
could use the optional Rinku mod for an embedded browser).

## 13. Checklist

- [ ] Vanilla fallback exists for every feature (menu, chat link, boss bar, action bar).
- [ ] Nothing is sent before HELLO; capabilities are checked before sending element types.
- [ ] Element IDs are namespaced (`<plugin>:<name>`, ≤ 64 chars) and validated; you clean up with `CLEAR "<plugin>:"`.
- [ ] PATCHes coalesced per tick; bars/cooldowns ≤ 4×/s; a failed PATCH (unknown ID) falls back to PUT.
- [ ] `show(...)` only as an alternative to a menu, and the menu opens when it returns `false`.
- [ ] Web pages work in a normal browser tab and implement the sign-in contract (§6.1).
- [ ] `webBase` is `https`, has a valid certificate and no path.
- [ ] Tokens are single-use, short-lived, permission-checked and never logged.
- [ ] REJECTs are logged (rate-limited) during development.
- [ ] Panels: every action is checked on the server against the document you sent; handlers re-check business rules
      (money, ownership, permission at the time of the click); IDs in closures, not in element IDs.
- [ ] Panels: a fallback exists for players without the mod (your chest menu, `dialogFallback(true)` or a web page).
- [ ] Panels: every P_ACTION gets a P_RESULT; refreshes are coalesced (Core: ≤ 10/s per session).
