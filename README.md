# 我的世界服务器优化指南 - Minecraft server optimization guide

对于使用原版Vanilla、Fabric或Spigot（或任何早于Paper的版本）的用户，请在server.properties文件中将`sync-chunk-writes`更改为`false`。这个选项在Paper及其分支中会被强制设置为false，但在一些服务器核心中，你需要手动将其更改为false。这允许服务器在主线程之外保存区块，从而减轻主线程的负担。

本指南针对1.20. 但一些优化项也可用于 1.15 - 1.19。

本指南基于 [此文](https://www.spigotmc.org/threads/guide-server-optimization%E2%9A%A1.283181/) 以及其他资料来源（所有相关资料在本指南中均有链接）。


# 前言
永远不会有一个指南能带来最完美的结果。每个服务器都有自己的需求和限制。根据你的服务器需求对选项进行微调，这才是关键所在。本指南的目的只是帮助你了解哪些选项会对性能产生影响，以及它们究竟会改变什么。如果您认为本指南中的信息或翻译不准确，可以提出Issues或创建Pull requests来更正。

# 准备

## 服务器核心JAR
服务器核心的选择会对性能和API接口的可用性造成很大影响。目前有很多种主流的服务器核心，但也有一些核心出于各种原因应该避免使用。

推荐:
* [Paper](https://github.com/PaperMC/Paper) - 最常用的服务器核心，旨在提高性能，同时修复游戏和机制不一致的问题。
* [Pufferfish](https://github.com/pufferfish-gg/Pufferfish) - Paper 分支，进一步提高服务器性能。
* [Purpur](https://github.com/PurpurMC/Purpur) - Pufferfish 分支， 旨在提高服务器性能和高度自定义配置。

你应该远离:
* 任何声称完全异步的付费服务器核心 - 99.99%是骗局。
* Bukkit/CraftBukkit/Spigot - 远古核心，性能过时。
* 任何插件或软件，用于热加载/卸载插件。 请阅读 [此文](#plugins-enablingdisabling-other-plugins) 来了解为何。
* 许多从Pufferfish或Purpur分支出来的核心可能会遇到不稳定性和其他问题。如果您想要更多的性能提升，可以尝试优化您的服务器或自行编写分支。

## 地图/区块预生成

由于游戏多年来对区块生成进行了各种优化，地图预生成仅对使用单线程的或低性能CPU的服务器有较大作用。预生成还通常用于为Pl3xMap或Dynmap之类的世界地图插件生成区块。

如果你仍然想预生成世界，可以使用[Chunky](https://github.com/pop4959/Chunky)之类的插件。确保设置一个世界边界，以防玩家生成新的区块！请注意，根据你在预生成插件中设置的生成半径，预生成可能需要数小时。请记住，使用Paper及其分支时，你的TPS不会受到区块加载的影响，但当服务器的CPU过载时，加载区块的速度可能会显著减慢。

请记住，主世界、下界和末地都应该有独立的世界边界。下界维度比主世界小 8 倍（如果没有用数据包修改），所以如果你设置了错误的边界大小，你的玩家可能会卡出世界边界！

**请确保你设置了原版世界边界 (`/worldborder set [diameter]`), 它会限制一些卡顿因素包括在边界之外查找藏宝图宝藏.**

# 核心配置文件

## 网络项

### [server.properties]

#### network-compression-threshold

`推荐值: 256`

当一个数据包的大小达到该值，服务器就会压缩它。设置得更高可以节省一些 CPU 资源，但会增加带宽占用，将其设置为 -1 会禁用它。设置的太高有可能会影响到网速较慢的玩家。 如果您的服务器位于具有代理的网络中或在同一台计算机上（ping 值小于 2 毫秒），则禁用（-1）将是有益的，因为内部网速通常可以处理额外的未压缩流量。

### [purpur.yml]

#### use-alternate-keepalive

`推荐值: true`

此项可以启用 Purpur 的心跳检测，这样网络情况较差的玩家就不会经常超时。已知与 TCPShield 不兼容。

> 启用此功能后，每秒向玩家发送一次 keepalive 包，玩家在 30 秒内未响应才会超时。玩家以任何顺序响应都不会超时。 
~ https://purpurmc.org/docs/Configuration/#use-alternate-keepalive

---

## 区块生成项

### [server.properties]

#### simulation-distance  模拟距离

`推荐值: 4`

此项为服务器开始计算各种游戏机制的距离（以区块为单位）。 这包括熔炉燃烧或作物生长等等。 您或许该将此值设低， 比如`3` 或`4`， 因为还存在着`view-distance`项。 这允许服务器加载不计算机制的假区块。可以让玩家在不影响性能的情况下看得更远。

#### view-distance  视距

`推荐值: 7`

此项为玩家能看到的距离（以区块为单位），类似于paper里的`no-tick-view-distance`。

实际总距离为`simulation-distance`和`view-distance`的最大值。 比如, 模拟距离是4 ，视距是12，那么玩家能看到的实际区块距离就是12。

### [spigot.yml]

#### view-distance

`推荐值: default`

如果不设置为`default`，此项将覆盖 server.properties。

### [paper-world configuration]

#### delay-chunk-unloads-by

`推荐值: 10s`

此项允许配置玩家离开后，区块保持加载的时间。这有助于避免玩家来回移动时，服务器不断加载和卸载相同的区块。过高的值可能会导致一次加载太多区块。在玩家频繁传送或加载的区域，可以考虑让该区域永久加载。这可以减轻服务器不小的负担。

#### max-auto-save-chunks-per-tick

`推荐值: 8`

通过降低世界区块保存速度来提高平均性能。当玩家超过 20-30 人时，您可能希望将其设置高于`8`。如果一个tick加载区块超过本设定值，bukkit 将立刻保存剩余的区块并重新开始下一个保存周期。

#### prevent-moving-into-unloaded-chunks

`推荐值: true`

防止玩家进入未加载的区块，以避免同步加载区块造成的主线程卡顿。view-distance视距越小，玩家进入未加载区块的可能性就越大。

#### entity-per-chunk-save-limit

```
推荐值:

    area_effect_cloud: 8
    arrow: 16
    dragon_fireball: 3
    egg: 8
    ender_pearl: 8
    experience_bottle: 3
    experience_orb: 16
    eye_of_ender: 8
    fireball: 8
    firework_rocket: 8
    llama_spit: 3
    potion: 8
    shulker_bullet: 8
    small_fireball: 8
    snowball: 8
    spectral_arrow: 16
    trident: 16
    wither_skull: 4
```

此项可以设置可保存的指定类型实体数量限制。您应该为每种弹射物规定一个限制，以避免保存大量弹射物时服务器崩溃。您可以在此处输入任何实体 ID，请参阅 Minecraft Wiki 以查找实体 ID。请根据您的喜好调整限制。所有弹射物的建议值约为`10`。您还可以根据名称将其他实体添加到该列表中。此项并不适用于阻止玩家建造大型生物农场。

### [pufferfish.yml]

#### max-loads-per-projectile

`推荐值: 8`

此项指定弹射物可以加载的最大区块数量。可减少弹射物造成的区块负载，但可能会导致末影珍珠等出现问题。

---

## Mobs

### [bukkit.yml]

#### spawn-limits

```
推荐值:

    monsters: 20
    animals: 5
    water-animals: 2
    water-ambient: 2
    water-underground-creature: 3
    axolotls: 3
    ambient: 1
```

生物生成的最大数量为 `[playercount] * [spawn limits]`，"playercount" 为玩家在线数量。 逻辑上, 该项数值越小, 玩家能遇到的生物就越少。 此项能确保每个玩家周围的生物数量是平均的。 这是一把双刃剑; 较低的值会减轻服务器负担，但在某些游戏模式中，自然生成的生物是游戏玩法的重要组成部分。 如果你适当的调整 `mob-spawn-range`，此项甚至还可以小于20。 较小的`mob-spawn-range`值会让人感觉每个玩家周围都有更多生物。 如果你使用paper或分支，你还可以在 [paper-world configuration] 中设置并覆盖此项。

#### ticks-per

```
推荐值:

    monster-spawns: 10
    animal-spawns: 400
    water-spawns: 400
    water-ambient-spawns: 400
    water-underground-creature-spawns: 400
    axolotl-spawns: 400
    ambient-spawns: 400
```

此项限制服务器每tick尝试生成特定实体的频率。 水生生物或环境生物（热带鱼或蝙蝠）不需要每tick都生成，因为它们通常不会那么快被杀死。对于怪物: 稍微增加间隔不会影响生成率，即使对于刷怪塔也是如此。 在大多数情况下，所有的值应该大于`1`。 将其设置得更高还可以让您的服务器更好地应对禁止怪物生成的区域。

### [spigot.yml]

#### mob-spawn-range

`推荐值: 3`

此项可以限制玩家周围生成怪物的范围 (区块为单位)。 考虑到服务器的游戏玩法和在线人数，你或许应该一并调整 [bukkit.yml] 里的`spawn-limits`。 将其设置得较低会让你感觉周围有更多的怪物。 这应该低于或等于你的`simulation-distance 模拟距离`，并且永远不要大于你的`despawn-ranges/16`。

#### entity-activation-range

```
推荐值:

      animals: 16
      monsters: 24
      raiders: 48
      misc: 8
      water: 8
      villagers: 16
      flying-monsters: 48
```

此项可以设置实体的激活AI距离。降低这些值有助于提高性能，但可能会导致怪物反应迟钝。将此值降低太多可能会破坏某些生物农场；比如刷铁机。

#### entity-tracking-range

```
推荐值:

      players: 48
      animals: 48
      monsters: 48
      misc: 32
      other: 64
```

This is distance in blocks from which entities will be visible. They just won't be sent to players. If set too low this can cause mobs to seem to appear out of nowhere near a player. In the majority of cases this should be higher than your `entity-activation-range`.

#### tick-inactive-villagers

`推荐值: false`

This allows you to control whether villagers should be ticked outside of the activation range. This will make villagers proceed as normal and ignore the activation range. Disabling this will help performance, but might be confusing for players in certain situations. This may cause issues with iron farms and trade restocking.

#### nerf-spawner-mobs

`推荐值: true`

You can make mobs spawned by a monster spawner have no AI. Nerfed mobs will do nothing. You can make them jump while in water by changing `spawner-nerfed-mobs-should-jump` to `true` in [paper-world configuration].

### [paper-world configuration]

#### despawn-ranges

```
推荐值:

      ambient:
        hard: 72
        soft: 30
      axolotls:
        hard: 72
        soft: 30
      creature:
        hard: 72
        soft: 30
      misc:
        hard: 72
        soft: 30
      monster:
        hard: 72
        soft: 30
      underground_water_creature:
        hard: 72
        soft: 30
      water_ambient:
        hard: 72
        soft: 30
      water_creature:
        hard: 72
        soft: 30
```

Lets you adjust entity despawn ranges (in blocks). Lower those values to clear the mobs that are far away from the player faster. You should keep soft range around `30` and adjust hard range to a bit more than your actual simulation-distance, so mobs don't immediately despawn when the player goes just beyond the point of a chunk being loaded (this works well because of `delay-chunk-unloads-by` in [paper-world configuration]). When a mob is out of the hard range, it will be instantly despawned. When between the soft and hard range, it will have a random chance of despawning. Your hard range should be larger than your soft range. You should adjust this according to your view distance using `(simulation-distance * 16) + 8`. This partially accounts for chunks that haven't been unloaded yet after player visited them.

#### per-player-mob-spawns

`推荐值: true`

This option decides if mob spawns should account for how many mobs are around target player already. You can bypass a lot of issues regarding mob spawns being inconsistent due to players creating farms that take up the entire mobcap. This will enable a more singleplayer-like spawning experience, allowing you to set lower `spawn-limits`. Enabling this does come with a very slight performance impact, however it's impact is overshadowed by the improvements in `spawn-limits` it allows.

#### max-entity-collisions

`推荐值: 2`

Overwrites option with the same name in [spigot.yml]. It lets you decide how many collisions one entity can process at once. Value of `0` will cause inability to push other entities, including players. Value of `2` should be enough in most cases. It's worth noting that this will render maxEntityCramming gamerule useless if its value is over the value of this config option.

#### update-pathfinding-on-block-update

`推荐值: false`

Disabling this will result in less pathfinding being done, increasing performance. In some cases this will cause mobs to appear more laggy; They will just passively update their path every 5 ticks (0.25 sec).

#### fix-climbing-bypassing-cramming-rule

`推荐值: true`

Enabling this will fix entities not being affected by cramming while climbing. This will prevent absurd amounts of mobs being stacked in small spaces even if they're climbing (spiders).

#### armor-stands.tick

`推荐值: false`

In most cases you can safely set this to `false`. If you're using armor stands or any plugins that modify their behavior and you experience issues, re-enable it. This will prevent armor stands from being pushed by water or being affected by gravity.

#### armor-stands.do-collision-entity-lookups

`推荐值: false`

Here you can disable armor stand collisions. This will help if you have a lot of armor stands and don't need them colliding with anything.

#### tick-rates

```
推荐值:

  behavior:
    villager:
      validatenearbypoi: 60
      acquirepoi: 120
  sensor:
    villager:
      secondarypoisensor: 80
      nearestbedsensor: 80
      villagerbabiessensor: 40
      playersensor: 40
      nearestlivingentitysensor: 40
```

> It is not recommended to change these values from their defaults while [Pufferfish's DAB](#dabenabled) is enabled!

This decides how often specified behaviors and sensors are being fired in ticks. `acquirepoi` for villagers seems to be the heaviest behavior, so it's been greately increased. Decrease it in case of issues with villagers finding their way around.

### [pufferfish.yml]

#### dab.enabled

`推荐值: true`

DAB (dynamic activation of brain) reduces the amount an entity is ticked the further away it is from players. DAB works on a gradient instead of a hard cutoff like EAR. Instead of fully ticking close entities and barely ticking far entities, DAB will reduce the amount an entity is ticked based on the result of a calculation influenced by [dab.activation-dist-mod](#dabactivation-dist-mod).

#### dab.max-tick-freq

`推荐值: 20`

Defines the slowest amount entities farthest from players will be ticked. Increasing this value may improve the performance of entities far from view but may break farms or greatly nerf mob behavior. If enabling DAB breaks mob farms, try decreasing this value.

#### dab.activation-dist-mod

`推荐值: 7`

Controls the gradient in which mobs are ticked. Decreasing this will activate DAB closer to players, improving DAB's performance gains, but will affect how entities interact with their surroundings and may break mob farms. If enabling DAB breaks mob farms, try increasing this value.

#### enable-async-mob-spawning

`推荐值: true`

If asynchronous mob spawning should be enabled. For this to work, the Paper's per-player-mob-spawns setting must be enabled. This option does not actually spawn mobs asynchronous, but does offload much of the computational effort involved with spawning new mobs to a different thread. Enabling this option should not be noticeable on vanilla gameplay.

#### enable-suffocation-optimization

`推荐值: true`

This option optimises a suffocation check (the check to see if a mob is inside a block and if they should take suffocation damage), by rate limiting the check to the damage timeout. This optimisation should be impossible to notice unless you're an extremely technical player who's using tick-precise timing to kill an entity at exactly the right time by suffocation.

#### inactive-goal-selector-throttle

`推荐值: true`

Throttles the AI goal selector in entity inactive ticks, causing the inactive entities to update their goal selector every 20 ticks instead of every tick. Can improve performance by a few percent, and has minor gameplay implications.

### [purpur.yml]

#### zombie.aggressive-towards-villager-when-lagging

`推荐值: false`

Enabling this will cause zombies to stop targeting villagers if the server is below the tps threshold set with `lagging-threshold` in [purpur.yml].

#### entities-can-use-portals

`推荐值: false`

This option can disable portal usage of all entities besides the player. This prevents entities from loading chunks by changing worlds which is handled on the main thread. This has the side effect of entities not being able to go through portals.

#### villager.lobotomize.enabled

`推荐值: true`

> This should only be enabled if villagers are causing lag! Otherwise, the pathfinding checks may decrease performance.

Lobotomized villagers are stripped from their AI and only restock their offers every so often. Enabling this will lobotomize villagers that are unable to pathfind to their destination. Freeing them should unlobotomize them.

#### villager.search-radius

```
推荐值:

          acquire-poi: 16
          nearest-bed-sensor: 16
```

Radius within which villagers will search for job site blocks and beds. This significantly boosts performance with large amount of villagers, but will prevent them from detecting job site blocks or beds that are further away than set value.

---

## Misc

### [spigot.yml]

#### merge-radius

```
推荐值:

      item: 3.5
      exp: 4.0
```

This decides the distance between the items and exp orbs to be merged, reducing the amount of items ticking on the ground. Setting this too high will lead to the illusion of items or exp orbs disappearing as they merge together. Setting this too high will break some farms, as well as allow items to teleport through blocks. There are no checks done to prevent items from merging through walls (unless Paper's `fix-items-merging-through-walls` setting is activated). Exp is only merged on creation.

#### hopper-transfer

`推荐值: 8`

Time in ticks that hoppers will wait to move an item. Increasing this will help improve performance if there are a lot of hoppers on your server, but will break hopper-based clocks and possibly item sorting systems if set too high.

#### hopper-check

`推荐值: 8`

Time in ticks between hoppers checking for an item above them or in the inventory above them. Increasing this will help performance if there are a lot of hoppers on your server, but will break hopper-based clocks and item sorting systems relying on water streams.

### [paper-world configuration]

#### alt-item-despawn-rate

```
推荐值:

      enabled: true
      items:
        cobblestone: 300
        netherrack: 300
        sand: 300
        red_sand: 300
        gravel: 300
        dirt: 300
        short_grass: 300
        pumpkin: 300
        melon_slice: 300
        kelp: 300
        bamboo: 300
        sugar_cane: 300
        twisting_vines: 300
        weeping_vines: 300
        oak_leaves: 300
        spruce_leaves: 300
        birch_leaves: 300
        jungle_leaves: 300
        acacia_leaves: 300
        dark_oak_leaves: 300
        mangrove_leaves: 300
        cactus: 300
        diorite: 300
        granite: 300
        andesite: 300
        scaffolding: 600
```

This list lets you set alternative time (in ticks) to despawn certain types of dropped items faster or slower than default. This option can be used instead of item clearing plugins along with `merge-radius` to improve performance.

#### redstone-implementation

`推荐值: ALTERNATE_CURRENT`

Replaces the redstone system with faster and alternative versions that reduce redundant block updates, lowering the amount of logic your server has to calculate. Using a non-vanilla implementation may introduce minor inconsistencies with very technical redstone, but the performance gains far outweigh the possible niche issues. A non-vanilla implementation option may additionally fix other redstone inconsistencies caused by CraftBukkit.

The `ALTERNATE_CURRENT` implementation is based off of the [Alternate Current](https://modrinth.com/mod/alternate-current) mod. More information on this algorithm can be found on their resource page.

#### hopper.disable-move-event

`推荐值: false`

`InventoryMoveItemEvent` doesn't fire unless there is a plugin actively listening to that event. This means that you only should set this to true if you have such plugin(s) and don't care about them not being able to act on this event. **Do not set to true if you want to use plugins that listen to this event, e.g. protection plugins!**

#### hopper.ignore-occluding-blocks

`推荐值: true`

Determines if hoppers will ignore containers inside full blocks, for example hopper minecart inside sand or gravel block. Keeping this enabled will break some contraptions depending on that behavior.

#### tick-rates.mob-spawner

`推荐值: 2`

This option lets you configure how often spawners should be ticked. Higher values mean less lag if you have a lot of spawners, although if set too high (relative to your spawners delay) mob spawn rates will decrease.

#### optimize-explosions

`推荐值: true`

Setting this to `true` replaces the vanilla explosion algorithm with a faster one, at a cost of slight inaccuracy when calculating explosion damage. This is usually not noticeable.

#### treasure-maps.enabled

`推荐值: false`

Generating treasure maps is extremely expensive and can hang a server if the structure it's trying to locate is in an ungenerated chunk. It's only safe to enable this if you pregenerated your world and set a vanilla world border.

#### treasure-maps.find-already-discovered

```
推荐值:
      loot-tables: true
      villager-trade: true
```

Default value of this option forces the newly generated maps to look for unexplored structure, which are usually in not yet generated chunks. Setting this to true makes it so maps can lead to the structures that were discovered earlier. If you don't change this to `true` you may experience the server hanging or crashing when generating new treasure maps. `villager-trade` is for maps traded by villagers and loot-tables refers to anything that generates loot dynamically like treasure chests, dungeon chests, etc.

#### tick-rates.grass-spread

`推荐值: 4`

Time in ticks between the server trying to spread grass or mycelium. This will make it so large areas of dirt will take a little longer to turn to grass or mycelium. Setting this to around `4` should work nicely if you want to decrease it without the decreased spread rate being noticeable.

#### tick-rates.container-update

`推荐值: 1`

Time in ticks between container updates. Increasing this might help if container updates cause issues for you (it rarely happens), but makes it easier for players to experience desync when interacting with inventories (ghost items).

#### non-player-arrow-despawn-rate

`推荐值: 20`

Time in ticks after which arrows shot by mobs should disappear after hitting something. Players can't pick these up anyway, so you may as well set this to something like `20` (1 second).

#### creative-arrow-despawn-rate

`推荐值: 20`

Time in ticks after which arrows shot by players in creative mode should disappear after hitting something. Players can't pick these up anyway, so you may as well set this to something like `20` (1 second).

### [pufferfish.yml]

#### disable-method-profiler

`推荐值: true`

This option will disable some additional profiling done by the game. This profiling is not necessary to run in production and can cause additional lag.

### [purpur.yml]

#### dolphin.disable-treasure-searching

`推荐值: true`

Prevents dolphins from performing structure search similar to treasure maps

#### teleport-if-outside-border

`推荐值: true`

Allows you to teleport the player to the world spawn if they happen to be outside of the world border. Helpful since the vanilla world border is bypassable and the damage it does to the player can be mitigated.

---

## Helpers

### [paper-world configuration]

#### anti-xray.enabled

`推荐值: true`

Enable this to hide ores from x-rayers. For detailed configuration of this feature check out [Configuring Anti-Xray](https://docs.papermc.io/paper/anti-xray). Enabling this will actually decrease performance, however it is much more efficient than any anti-xray plugin. In most cases the performance impact will be negligible.

#### nether-ceiling-void-damage-height

`推荐值: 127`

If this option is greater that `0`, players above the set y level will be damaged as if they were in the void. This will prevent players from using the nether roof. Vanilla nether is 128 blocks tall, so you should probably set it to `127`. If you modify the height of the nether in any way you should set this to `[your_nether_height] - 1`.

---

# Java startup flags
[Vanilla Minecraft and Minecraft server software in version 1.20.5+ requires Java 21 or higher](https://docs.papermc.io/java-install-update). Oracle has changed their licensing, and there is no longer a compelling reason to get your java from them. Recommended vendors are [Adoptium](https://adoptium.net/) and [Amazon Corretto](https://aws.amazon.com/corretto/). Alternative JVM implementations such as OpenJ9 or GraalVM can work, however they are not supported by Paper and have been known to cause issues, therefore they are not currently recommended.

Your garbage collector can be configured to reduce lag spikes caused by big garbage collector tasks. You can find startup flags optimized for Minecraft servers [here](https://docs.papermc.io/paper/aikars-flags) [`SOG`]. Keep in mind that this recommendation will not work on alternative JVM implementations.
It's recommended to use the [flags.sh](https://flags.sh) startup flags generator to get the correct startup flags for your server

In addition, adding the beta flag `--add-modules=jdk.incubator.vector` before `-jar` in your startup flags can improve performance. This flag enables Pufferfish to use SIMD instructions on your CPU, making some maths faster. Currently, it's only used for making rendering in game plugin maps (like imageonmaps) possibly 8 times faster.

# "Too good to be true" plugins

## Plugins removing ground items
Absolutely unnecessary since they can be replaced with [merge-radius](#merge-radius) and [alt-item-despawn-rate](#alt-item-despawn-rate) and frankly, they're less configurable than basic server configs. They tend to use more resources scanning and removing items than not removing the items at all.

## Mob stacker plugins
It's really hard to justify using one. Stacking naturally spawned entities causes more lag than not stacking them at all due to the server constantly trying to spawn more mobs. The only "acceptable" use case is for spawners on servers with a large amount of spawners.

## Plugins enabling/disabling other plugins
Anything that enables or disables plugins on runtime is extremely dangerous. Loading a plugin like that can cause fatal errors with tracking data and disabling a plugin can lead to errors due to removing dependency. The `/reload` command suffers from exact same issues and you can read more about them in [me4502's blog post](https://madelinemiller.dev/blog/problem-with-reload/)

# What's lagging? - measuring performance

## mspt
Paper offers a `/mspt` command that will tell you how much time the server took to calculate recent ticks. If the first and second value you see are lower than 50, then congratulations! Your server is not lagging! If the third value is over 50 then it means there was at least 1 tick that took longer. That's completely normal and happens from time to time, so don't panic.
  
## Spark
[Spark](https://spark.lucko.me/) is a plugin that allows you to profile your server's CPU and memory usage. You can read on how to use it [on its wiki](https://spark.lucko.me/docs/). There's also a guide on how to find the cause of lag spikes [here](https://spark.lucko.me/docs/guides/Finding-lag-spikes).

## Timings
Way to see what might be going on when your server is lagging are Timings. Timings is a tool that lets you see exactly what tasks are taking the longest. It's the most basic troubleshooting tool and if you ask for help regarding lag you will most likely be asked for your Timings. Timings is known to have a serious performance impact on servers, it's recommended to use the Spark plugin over Timings and use Purpur or Pufferfish to disable Timings all together.

To get Timings of your server, you just need to execute the `/timings paste` command and click the link you're provided with. You can share this link with other people to let them help you. It's also easy to misread if you don't know what you're doing. There is a detailed [video tutorial by Aikar](https://www.youtube.com/watch?v=T4J0A9l7bfQ) on how to read them.

---

# Minecraft exploits and how to fix them
To see how to fix exploits that can cause lag spikes or crashes on a Minecraft server, refer to [here](https://github.com/YouHaveTrouble/minecraft-exploits-and-how-to-fix-them).

[`SOG`]: https://www.spigotmc.org/threads/guide-server-optimization%E2%9A%A1.283181/
[server.properties]: https://docs.papermc.io/paper/reference/server-properties
[bukkit.yml]: https://docs.papermc.io/paper/reference/bukkit-configuration
[spigot.yml]: https://docs.papermc.io/paper/reference/spigot-configuration
[paper-global configuration]: https://docs.papermc.io/paper/reference/global-configuration
[paper-world configuration]: https://docs.papermc.io/paper/reference/world-configuration
[purpur.yml]: https://purpurmc.org/docs/Configuration/
[pufferfish.yml]: https://docs.pufferfish.host/setup/pufferfish-fork-configuration/
