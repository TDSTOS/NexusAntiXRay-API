# NexusAntiXRay API

A small Java API for plugins that want to work with NexusAntiXRay: ask whether a block is hidden from a player,
switch the filter off for a player, read the mining statistics of the x-ray detection (the running numbers and the
totals stored across restarts), and listen to its events (alerts, bait veins, bypass changes, self-throttling).

- API version: **1.0.0** (first shipped with NexusAntiXRay 1.3.0)
- Java 8 or newer, Spigot, Paper, Purpur and Folia
- Package: `de.nexus.antixray.api` (events in `de.nexus.antixray.api.event`)
- Download: [Releases](https://github.com/TDSTOS/NexusAntiXRay-API/releases)
- Javadoc: [tdstos.github.io/NexusAntiXRay-API](https://tdstos.github.io/NexusAntiXRay-API/)

The API is one interface, `NexusAntiXRayApi`, two small data interfaces (`MiningStats` and `StoredMiningStats`), one
enum (`AntiXRayFeature`) and four Bukkit events. Everything else inside the NexusAntiXRay jar is obfuscated and not meant to be called.

## Contents

- [Getting the jar](#getting-the-jar)
- [Adding it to your build](#adding-it-to-your-build)
- [plugin.yml](#pluginyml)
- [Getting the API instance](#getting-the-api-instance)
- [Methods](#methods)
- [Events](#events)
- [Threads](#threads)
- [What happens when ...](#what-happens-when-)
- [Versioning](#versioning)

## Getting the jar

Download `NexusAntiXRay-API-1.0.0.jar` from the
[Releases page](https://github.com/TDSTOS/NexusAntiXRay-API/releases). Next to it you find
`NexusAntiXRay-API-1.0.0-sources.jar` and `NexusAntiXRay-API-1.0.0-javadoc.jar`; attach them in your IDE to get the
documentation while you type. The same Javadoc is online at
[tdstos.github.io/NexusAntiXRay-API](https://tdstos.github.io/NexusAntiXRay-API/), one folder per API version
(for example `/1.0.0/`).

You only compile against this jar. **Do not put it into your own plugin jar.** At runtime the classes come from the
NexusAntiXRay plugin itself. If you shade a copy into your plugin, your copy is a different set of classes on current
servers: `NexusAntiXRayApi.get()` fails with a message that says so, and your event listeners are never called.

A shaded copy does not only break your own plugin. The server hands a plugin that has no copy of its own the first one
it finds among all plugins, so correctly built plugins on the same server can end up with yours and fail the same way.
NexusAntiXRay warns at startup about every plugin that carries a copy, and the message of `get()` names the plugin the
classes were loaded from.

## Adding it to your build

Put the jar into a folder of your project, for example `libs/`.

### Gradle

```groovy
dependencies {
    compileOnly files('libs/NexusAntiXRay-API-1.0.0.jar')
}
```

Kotlin DSL:

```kotlin
dependencies {
    compileOnly(files("libs/NexusAntiXRay-API-1.0.0.jar"))
}
```

If you use the Shadow plugin: `compileOnly` dependencies are not shaded, so there is nothing else to do.

### Maven

Either point Maven at the file:

```xml
<dependency>
    <groupId>de.nexus.antixray</groupId>
    <artifactId>nexusantixray-api</artifactId>
    <version>1.0.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/libs/NexusAntiXRay-API-1.0.0.jar</systemPath>
</dependency>
```

or install it into your local repository once

```
mvn install:install-file -Dfile=libs/NexusAntiXRay-API-1.0.0.jar -DgroupId=de.nexus.antixray -DartifactId=nexusantixray-api -Dversion=1.0.0 -Dpackaging=jar
```

and depend on it with scope `provided`:

```xml
<dependency>
    <groupId>de.nexus.antixray</groupId>
    <artifactId>nexusantixray-api</artifactId>
    <version>1.0.0</version>
    <scope>provided</scope>
</dependency>
```

Both scopes keep the jar out of the maven-shade-plugin. Do not add a relocation for `de.nexus.antixray`.

## plugin.yml

Your plugin has to be enabled after NexusAntiXRay. If it cannot work without it:

```yaml
depend: [NexusAntiXRay]
```

If NexusAntiXRay is optional:

```yaml
softdepend: [NexusAntiXRay]
```

**In both cases, check that NexusAntiXRay is enabled before you touch any API class.** The API classes live in the
NexusAntiXRay jar. They cannot be loaded if the plugin is missing (`softdepend`), and also not if it disabled itself
while starting, for example because no license key is set: Bukkit closes the class loader of a disabled plugin and
enables your plugin anyway, `depend` or not. The first line that touches an API class then fails with a
`NoClassDefFoundError`, in the middle of your `onEnable`.

Keep every use of the API in a class of its own and only touch that class after the check:

```java
@Override
public void onEnable() {
    if (!getServer().getPluginManager().isPluginEnabled("NexusAntiXRay")) {
        // with depend: NexusAntiXRay is installed but shut itself down, its log lines say why
        getLogger().warning("NexusAntiXRay is not running - continuing without it.");
        return;
    }
    // AntiXRayHook is the only class of this plugin that imports de.nexus.antixray.api
    getServer().getPluginManager().registerEvents(new AntiXRayHook(this), this);
}
```

A Paper plugin (`paper-plugin.yml`) declares the dependency like this, so that it can see the classes:

```yaml
dependencies:
  server:
    NexusAntiXRay:
      load: BEFORE
      required: true
      join-classpath: true
```

## Getting the API instance

```java
NexusAntiXRayApi api = NexusAntiXRayApi.get();
```

`get()` throws an `IllegalStateException` while NexusAntiXRay is not enabled. The message says what is wrong: the plugin
is not enabled yet (missing `depend`/`softdepend`, or called before your `onEnable`), it was disabled while the server
is running, or the API classes in use are a shaded copy; in that case the message names the plugin they came from.

`get()` can only explain a problem while the API classes can be loaded. If NexusAntiXRay is not installed, or disabled
itself during startup, there is no `IllegalStateException` to catch: you get the `NoClassDefFoundError` described under
[plugin.yml](#pluginyml). The `isPluginEnabled` check shown there covers both.

The instance is also registered with Bukkit's services manager. This variant returns `null` instead of throwing:

```java
NexusAntiXRayApi api = Bukkit.getServicesManager().load(NexusAntiXRayApi.class);
if (api == null) {
    return; // NexusAntiXRay is not running
}
```

NexusAntiXRay registers the API at the end of its `onEnable` and removes it at the start of its `onDisable`. An
instance you kept stays safe to call after that, but it answers like a server without players. Call `get()` when you
need the API instead of storing it for the lifetime of your plugin.

### Checking the version

```java
if (api.isApiAtLeast(1, 1)) {
    // safe to call something that was added in API 1.1.0
}
getLogger().info("NexusAntiXRay API " + api.apiVersion());
```

`api.apiVersion()` is the version that is running. `NexusAntiXRayApi.API_VERSION` is a compile-time constant: read from
your code it is the version you compiled against.

## Methods

All methods can be called from any thread, see [Threads](#threads). All of them answer at once from memory, except
`loadStats`, which reads from disk and answers through a `CompletableFuture`.

### isBypassed

Is the anti-xray filter switched off for this player? A bypassed player gets the world unmodified and is left out of
the x-ray detection.

```java
if (api.isBypassed(player)) {
    player.sendMessage("You see the world as it is.");
}
```

The state comes from the permission `nexusantixray-autobypass` and from the session switch (`/nax bypass` or
`setBypass`). While the session switch is set, it wins over the permission in both directions. It is dropped when the
player leaves and whenever the permission changes.

### setBypass

Sets the session switch, like `/nax bypass` (which toggles it).

```java
// a build event: everybody in the arena sees real ores
boolean tracked = api.setBypass(player, true);
if (!tracked) {
    // the player is offline or has not joined yet - nothing was done
}

// later
api.setBypass(player, false);
```

`false` also switches the filter on for a player who has `nexusantixray-autobypass`. Nothing is stored: after a
rejoin the permission decides again.

The return value says whether NexusAntiXRay tracks the player, not whether anything changed: it is `true` also when
the player already was in the requested state, and `false` only for a player who is offline or has not joined yet.
To learn whether the state changed, read `isBypassed` before the call or listen to `BypassChangeEvent`.

If the effective state changes, the player's chunks are sent again and a `BypassChangeEvent` with cause `API` follows.

`setBypass` works from the `PlayerJoinEvent` on, and a switch set there is in place before the player gets the first
chunk:

```java
@EventHandler // priority NORMAL; NexusAntiXRay reads the permission at LOW
public void onJoin(PlayerJoinEvent event) {
    if (isArenaBuilder(event.getPlayer())) {
        NexusAntiXRayApi.get().setBypass(event.getPlayer(), true);
    }
}
```

Do not call it from a join handler with priority `LOWEST`. That runs before NexusAntiXRay has read the permission: the
call usually returns `false`, and a switch it does set is dropped again if the player has `nexusantixray-autobypass`.

### isFeatureActive

Is a feature switched on in a world? This is the effective switch after `worlds.disabled` and `worlds.overrides` of
`config.yml`.

```java
World world = player.getWorld();
if (!api.isFeatureActive(AntiXRayFeature.HIDE_BLOCKS, world)) {
    // ores are not hidden in this world
}
```

The answer is about the world. For one player a feature is at work if `isFeatureActive(feature, world)` is `true` and
`isBypassed(player)` is `false`.

### isHidden

Does the client of this player currently show a replacement instead of this block?

```java
Block block = event.getClickedBlock();
if (api.isHidden(player, block)) {
    // the player's client shows stone here, not what the server has
}
```

`true` for blocks of the hide list the player has no line of sight to, for hidden block entities such as chests, and
for every position inside a structure that is still filled for the player. `false` for everything the player gets
unchanged.

A **fake ore** is not a hidden block: the server has ordinary stone there and the client shows an ore that does not
exist. `isHidden` returns `false` for it.

The answer is a snapshot. A block is revealed within a few ticks after the player gets a line of sight, and it may be
hidden again later.

For a bypassed player nothing is hidden, and that counts from the moment the bypass changes: right after
`setBypass(player, true)` the method returns `false` for every block, although the client needs a few more ticks until
the chunks have arrived again.

### refresh

Hides everything again for a player and sends the chunks again, like `/nax refresh <player>`.

```java
// your plugin changed what the player should be able to see
api.refresh(player);
```

Returns `false` if NexusAntiXRay does not track the player. You do not need it after `setBypass`.

### getStats

The mining statistics the x-ray detection holds in memory for a player. The detection counts *bait veins*: fake ore
veins that only an x-ray client can see. The measure is bait veins per 100 mined blocks.

```java
MiningStats stats = api.getStats(player.getUniqueId());
if (stats == null) {
    sender.sendMessage("No statistics for " + player.getName() + " yet.");
    return;
}
sender.sendMessage(stats.getPlayerName() + ": " + stats.getWindowBaitVeins() + " bait veins in "
        + stats.getWindowMinedBlocks() + " blocks (" + String.format("%.1f", stats.getWindowRate())
        + " per 100) within " + stats.getWindowMinutes() + " min, " + stats.getAlertCount() + " alerts");
```

The *window* values cover the last `xray-detection.window-minutes`; alerts are decided on them. The other values are
totals since NexusAntiXRay started counting the player in the current server run; they start at zero after a restart.
The snapshot is immutable.

The method works for offline players as long as their statistics are still in memory (until the server stops, or until
the entry makes room for more recently active players). It never reads from disk and never blocks. It returns `null`
if there is nothing in memory for the player. For the totals across restarts use [loadStats](#loadstats).

### loadStats

The totals NexusAntiXRay has stored for a player across server restarts: mined blocks, bait veins, alerts, and when
the player was first and last seen. They live in the statistics storage of the plugin (`xray-detection.storage` in
`config.yml`) and are read from disk, so the answer comes later, through a `CompletableFuture`.

```java
UUID id = target.getUniqueId(); // works for offline players too
NexusAntiXRayApi.get().loadStats(id).thenAccept(stored -> {
    // not on the server thread: switch before you touch the Bukkit API
    // Spigot and Paper; on Folia use Bukkit.getGlobalRegionScheduler().execute(plugin, () -> ...)
    Bukkit.getScheduler().runTask(plugin, () -> {
        if (stored == null) {
            sender.sendMessage("Nothing stored for this player.");
            return;
        }
        sender.sendMessage(stored.getPlayerName() + ": " + stored.getBaitVeins() + " bait veins in "
                + stored.getMinedBlocks() + " blocks (" + String.format("%.1f", stored.getRate()) + " per 100), "
                + stored.getAlertCount() + " alerts, last seen " + new Date(stored.getLastSeenTime()));
    });
});
```

The result is an immutable `StoredMiningStats`. It includes what has been counted but not written to disk yet. The
future completes with `null` if there is nothing to hand out: nothing is stored for the
player, the storage is switched off or could not be opened, reading failed, too many requests are waiting, or
NexusAntiXRay is disabled (also while your request is waiting). It never completes exceptionally, and `loadStats` never
returns `null` itself. Every call gets a future of its own.

Three rules:

- **Do not poll.** Every call is a read from disk on the one thread that also writes the statistics. For numbers that
  change while a player is mining (a scoreboard, a placeholder) use `getStats`. Load the stored totals when you need
  them: for a command, when a player joins, for a report.
- **Do not rely on the callback thread.** `thenAccept` and its relatives usually run on an asynchronous thread, but on
  your own thread if the answer is there at once (storage off, plugin disabled), and on the thread that disables
  NexusAntiXRay for requests that were still waiting. Switch to your scheduler before you use the Bukkit API, as in the
  example.
- **Never wait for the result on a server thread.** No `join()` or `get()` on the main thread, a Folia region thread or
  in an event handler: the server stalls for as long as the disk takes.

A record is created with the first block that counts for a player and removed again when the player has not been seen
for `xray-detection.storage.retention-days` days (90 by default); after that the totals start at zero. The alerts and
mining sessions NexusAntiXRay stores as well (`/nax history`) and the ores per type are not part of the API.

## Events

Register a listener as for any Bukkit event:

```java
getServer().getPluginManager().registerEvents(new AntiXRayHook(this), this);
```

All four events are synchronous. The three events about a player are called on the thread that ticks that player;
`ThrottleLevelChangeEvent` has no player and is called on the server thread (on Folia the global region thread).

### XRayAlertEvent

Called when the x-ray detection is about to raise an alert, before anything is sent. Cancellable.

```java
public final class AntiXRayHook implements Listener {

    private final JavaPlugin plugin;
    private final Set<UUID> miningEvent = ConcurrentHashMap.newKeySet();

    public AntiXRayHook(JavaPlugin plugin) {
        this.plugin = plugin;
    }

    @EventHandler(ignoreCancelled = true)
    public void onAlert(XRayAlertEvent event) {
        Player player = event.getPlayer();
        if (miningEvent.contains(player.getUniqueId())) {
            // players of our mining event get boosted ore rates - no alert for them
            event.setCancelled(true);
            return;
        }
        plugin.getLogger().warning(player.getName() + " hit " + event.getBaitVeins() + " bait veins in "
                + event.getMinedBlocks() + " blocks (" + String.format("%.1f", event.getRate()) + " per 100) at "
                + event.getWorld().getName() + " " + event.getX() + " " + event.getY() + " " + event.getZ());
    }
}
```

What cancelling means: **the alert did not happen.** Nothing is sent (console, staff chat, commands, Discord,
`detections.log`), the alert is not counted in `/nax stats` or `MiningStats#getAlertCount()`, and the cooldown does
not start. The player is still above the thresholds, so the event is called again for the next bait vein the player
digs into. If you cancel the event in order to send your own notification, limit how often you send it. If you only
want to react in addition, do not cancel.

For the server owner two traces remain, neither of which says anything about the alert itself: the first cancelled
alert of a server run is noted in the log, together with the names of the plugins that listen to the event, and
`/nax report` shows how many alerts were cancelled. Tell your users if your plugin cancels alerts.

The event is called from inside the `BlockBreakEvent` of the block that crossed the threshold. Do not kick or teleport
the player right in the handler. Schedule it:

```java
@EventHandler(priority = EventPriority.MONITOR, ignoreCancelled = true)
public void onConfirmedAlert(XRayAlertEvent event) {
    Player player = event.getPlayer();
    // Spigot and Paper; on Folia use player.getScheduler().run(plugin, task -> ..., null)
    Bukkit.getScheduler().runTask(plugin, () -> player.kickPlayer("X-ray is not allowed here."));
}
```

### BaitVeinHitEvent

Called each time the detection counts a bait vein for a player. Not cancellable. A vein counts once within the
detection window, however many of its blocks the player digs out.

```java
@EventHandler
public void onBaitVein(BaitVeinHitEvent event) {
    plugin.getLogger().info(event.getPlayer().getName() + " dug into a fake " + event.getOreType() + " vein ("
            + event.getVeinSize() + " blocks) at " + event.getX() + " " + event.getY() + " " + event.getZ());
}
```

One hit says little: honest miners run into a bait vein now and then. The hit is already counted when the event is
called, so `api.getStats(...)` includes it. If the hit makes an alert due, `XRayAlertEvent` follows right after.

### BypassChangeEvent

Called after the effective bypass of a player changed. Not cancellable. `getCause()` is `COMMAND` (`/nax bypass`),
`PERMISSION` (`nexusantixray-autobypass` granted or revoked, including a join with the permission) or `API`
(`setBypass`).

```java
@EventHandler
public void onBypassChange(BypassChangeEvent event) {
    if (event.isBypassed() && event.getCause() == BypassChangeEvent.Cause.COMMAND) {
        plugin.getLogger().info(event.getPlayer().getName() + " switched the anti-xray filter off.");
    }
}
```

The event reports changes only: setting the state the player already has calls nothing. Every player starts a session
without bypass, and no event is called when a player leaves.

To overrule a change, call `setBypass` from the handler:

```java
@EventHandler
public void keepArenaFiltered(BypassChangeEvent event) {
    if (event.isBypassed() && isInArena(event.getPlayer())) {
        NexusAntiXRayApi.get().setBypass(event.getPlayer(), false);
    }
}
```

The events of one player arrive one after the other, in the order of the changes, and never nested. The change made in
the handler above is in effect at once, but its own event (cause `API`) is called only after the current event has
reached all listeners. A plugin that keeps its own copy of the state from `event.isBypassed()` therefore always ends
up right, whatever its listener priority. While a handler runs, a later change may already be in effect;
`api.isBypassed(player)` has the state of this moment.

Two plugins that overrule each other this way would never stop. After a few dozen changes in a row NexusAntiXRay stops
calling the event for that player, logs a warning with the names of the listening plugins and drops the rest.

A change can be made from any thread (console command, `setBypass` from an asynchronous task). The event is always
called on the player's thread; if the change came from elsewhere, it follows a moment later. Do not expect the event to
have been called when `setBypass` returns.

`isBypassed` is not reliable any more inside a `PlayerQuitEvent`: the session ends with the connection, which is
often gone before the event. If you need the last state of a leaving player, remember it from this event.

### ThrottleLevelChangeEvent

Called when the self-throttling (`performance.auto-throttle` in the config of NexusAntiXRay, off by default) changes
its level. Not cancellable. Level 0 means no throttling; a higher level means NexusAntiXRay checks less often, so
hidden things are revealed and hidden again a little later. Hiding, fake veins and light masking stay the same in every
level. Today the highest level is 1; later versions may add levels, so test with `> 0`.

```java
@EventHandler
public void onThrottle(ThrottleLevelChangeEvent event) {
    String tick = Double.isNaN(event.getTickMillis()) ? "unknown"
            : String.format("%.1f ms (%s)", event.getTickMillis(), event.getTickSource());
    plugin.getLogger().info("NexusAntiXRay throttle " + event.getOldLevel() + " -> " + event.getNewLevel()
            + ", cause " + event.getCause() + ", tick " + tick);
}
```

- `getCause()` is `LOAD` (the tick time stayed above the threshold for 10 seconds, or at or below it for 30 seconds)
  or `CONFIG` (throttling switched off, `max-level` lowered, or the development switch
  `-Dnexusantixray.throttle.force` at the start).
- `getTickMillis()` is the average tick time of the last seconds, normalized to 20 ticks per second; `NaN` for
  `CONFIG` and when the value is not known. `getTickSource()` is `TICK_TIME` (the server reports it, Paper and forks),
  `TICK_INTERVAL` (NexusAntiXRay measures the time between ticks, Spigot; that shows nothing below about 51 ms) or
  `UNKNOWN` (whenever the value is `NaN`).
- No event during the start-up period (60 seconds after the server, the plugin or the throttling starts), with
  `max-level: 0`, when NexusAntiXRay is disabled, and never for single players.
- On Folia each player gets the level of the region they are in. The event is called on the global region thread and
  reports the highest level of all players, with the tick time of the player that has it, up to a second after that
  player's change. When the only throttled player leaves the server, the reported level drops at once with `LOAD`
  (nobody is throttled any more; the tick time is another player's, or `NaN`). A player who logs in again keeps their
  level.

## Threads

| What | Thread |
|---|---|
| `NexusAntiXRayApi.get()` and every method of `NexusAntiXRayApi`, `MiningStats` and `StoredMiningStats` | any thread: server thread, Folia region, entity and global threads, asynchronous tasks |
| callbacks attached to the future of `loadStats` | not specified: usually an asynchronous thread, sometimes the calling one; see [loadStats](#loadstats) |
| `XRayAlertEvent`, `BaitVeinHitEvent` | the thread that ticks the player (server thread; on Folia the region thread that owns the player), inside the `BlockBreakEvent` of the mined block |
| `BypassChangeEvent` | the thread that ticks the player |
| `ThrottleLevelChangeEvent` | the server thread; on Folia the global region thread (the event has no player) |

The methods only read and change NexusAntiXRay's own state and never touch the world, which is why they are safe
everywhere. `loadStats` is the only one that reads from disk, and it does that on a thread of its own, not on yours.
No event is ever called from a network thread or from one of the ray tracing threads.

In a handler of a player event you are on the player's thread and may use the Bukkit API for that player as usual. On
Folia that does not include other players or far away chunks; use their schedulers. A `ThrottleLevelChangeEvent`
handler on Folia runs on the global region thread: it may not touch players, entities or chunks directly.

## What happens when ...

| Situation | `isBypassed` | `setBypass` | `isHidden` | `refresh` | `getStats` | `loadStats` (result of the future) |
|---|---|---|---|---|---|---|
| player offline | `false` | `false`, nothing changes | `false` | `false` | snapshot while in memory, else `null` | stored totals, `null` if nothing is stored |
| player logging in (login events) | `false` | `false`, nothing changes | `false` | `false` | as above | as above |
| in a `PlayerJoinEvent` handler with priority `LOWEST` (before NexusAntiXRay read the permission) | not reliable | not reliable, do not call it here | `false` | `false` or `true` | as above | as above |
| in the `PlayerJoinEvent` (priority `LOW` or later) and until the first chunk | current state (permission, then the switch) | works, in place before the first chunk | `false` | `false` or `true`, nothing to send yet | as above | as above |
| in the `PlayerQuitEvent` | not reliable, often already `false` | not reliable, the session ends anyway | `false` or the last state | `false` or `true` | as above | as above |
| world listed under `worlds.disabled` | current state | state is stored, applies in the next world | `false` | `true`, no visible effect | nothing new is counted there | as above |
| player is bypassed | `true` | works | `false`, from the moment of the change | `true`, no visible effect | nothing new is counted | as above |
| statistics storage off (`xray-detection.storage.enabled: false`, or the file could not be opened) | as usual | as usual | as usual | as usual | as usual | `null`, at once |
| NexusAntiXRay disabled while the server runs, instance kept | `false` | `false` | `false` | `false` | `null` | `null`, at once; requests still waiting complete with `null` |
| NexusAntiXRay not installed, or disabled itself during startup | no API class can be loaded: `NoClassDefFoundError`, see [plugin.yml](#pluginyml) | | | | | |

`isFeatureActive` is `false` for every feature in a world listed under `worlds.disabled` and after NexusAntiXRay was
disabled.

`null` arguments are rejected with an `IllegalArgumentException`, by `loadStats` too (at the call, not in the future).
Apart from that the methods do not throw. If NexusAntiXRay fails internally, it logs the problem and returns the
neutral value of the row "disabled while the server runs". The future of `loadStats` never completes exceptionally.

## Versioning

The API has its own version, independent of the plugin version, and follows semantic versioning:

- **Patch** (1.0.x): documentation and fixes, no change for callers.
- **Minor** (1.x.0): additions only. New methods, new events, new enum constants. Code compiled against an older
  1.x keeps working.
- **Major** (x.0.0): something was removed or changed.

What that means for your code:

- Every class and method carries `@since` with the API version it appeared in. Before you call something newer than
  1.0.0, check `api.isApiAtLeast(major, minor)`; on an older NexusAntiXRay the method does not exist.
- `NexusAntiXRayApi`, `MiningStats` and `StoredMiningStats` are implemented by NexusAntiXRay only. New methods on them
  are additions, so do not implement these interfaces yourself.
- Enums may get new constants. Give every `switch` over `AntiXRayFeature`, `BypassChangeEvent.Cause` or the enums of
  `ThrottleLevelChangeEvent` a `default` branch.
- The event constructors are public because Bukkit events are plain classes, but only NexusAntiXRay calls them.
