# 我的世界服务器优化指南中译版 [Minecraft server optimization guide](https://github.com/YouHaveTrouble/minecraft-optimization)

作者：YouHaveTrouble 译者：igoby

对于使用原版Vanilla、Fabric或Spigot（或任何早于Paper的版本）核心的用户，请在server.properties文件中将`sync-chunk-writes`更改为`false`。 这个选项在Paper及其分支中会被强制设置为false，但在一些服务器核心中，您需要手动将其更改为false。 这允许服务器在主线程之外保存区块，从而减轻主线程的负担。

本指南针对1.20. 但一些优化项也可用于 1.15 - 1.19。

本指南基于 [此文](https://www.spigotmc.org/threads/guide-server-optimization%E2%9A%A1.283181/) 以及其他资料来源（所有相关资料在本指南中均有链接）。

# 前言 本文仅供参考
永远不会有一个指南能迎合所有需求。每个服务器都有自己的需求和限制。根据你的服务器需求对选项进行微调，这才是关键所在。本指南的目的只是帮助您了解哪些选项会对性能产生影响，以及它们究竟会改变什么。 如果您认为本指南中的信息或翻译不准确，可以提出Issues或创建Pull requests来更正。

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
* 任何插件或软件，用于热加载/卸载插件。 请阅读 [此部分](#任何用于热加载/卸载的插件) 来了解为何。

## 地图/区块预生成

由于游戏多年来对区块生成进行了各种优化，地图预生成仅对使用单线程的或低性能CPU的服务器有较大作用。预生成还通常用于为Pl3xMap或Dynmap之类的世界地图插件生成区块。

如果您仍然想预生成世界，可以使用[Chunky](https://github.com/pop4959/Chunky)之类的插件。确保设置一个世界边界，以防玩家生成新的区块！请注意，根据您在预生成插件中设置的生成半径，预生成可能需要数小时。请记住，使用Paper及分支时，您的TPS不会受到区块加载的影响，但当服务器的CPU过载时，加载区块的速度可能会显著减慢。

请记住，主世界、下界和末地都应该有独立的世界边界。下界维度比主世界小 8 倍（如果没有用数据包修改），所以如果您设置了错误的边界大小，玩家可能会卡出世界边界！

**请确保您设置了原版世界边界 (`/worldborder set [diameter]`), 它会限制一些卡顿因素包括在边界之外查找藏宝图宝藏.**

# 核心配置文件 推荐值仅供参考 推荐值仅供参考 推荐值仅供参考

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

此项可以设置可保存的指定类型实体数量限制。您应该为每种弹射物规定一个限制，以避免服务器尝试保存大量弹射物时崩溃。您可以在此处输入任何实体 ID，请参阅 Minecraft Wiki 以查找实体 ID。请根据您的喜好调整限制。所有弹射物的建议值约为`10`。您还可以根据名称将其他实体添加到该列表中。此项并不适用于阻止玩家建造大型生物农场。

### [pufferfish.yml]

#### max-loads-per-projectile

`推荐值: 8`

此项指定弹射物可以加载的最大区块数量。可减少弹射物造成的区块负载，但可能会导致末影珍珠等出现问题。

---

## 生物生成项

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

生物生成的最大数量为 `[playercount] * [spawn limits]`，"playercount" 为玩家在线数量。 逻辑上, 该项数值越小, 玩家能遇到的生物就越少。 此项能确保每个玩家周围的生物数量是平均的。 这是一把双刃剑; 较低的值会减轻服务器负担，但在某些游戏模式中，自然生成的生物是游戏玩法的重要组成部分。 如果您适当的调整 `mob-spawn-range`，此项甚至还可以小于`20`。 较小的`mob-spawn-range`值会让人感觉每个玩家周围都有更多生物。 如果您使用paper或分支，你还可以在 [paper-world configuration] 中设置并覆盖此项。

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

此项限制服务器每 tick 尝试生成特定实体的频率。 水生生物或环境生物（热带鱼或蝙蝠）不需要每 tick 都生成，因为它们通常不会那么快被杀死。对于怪物: 稍微增加间隔不会影响生成率，即使对于刷怪塔也是如此。 在大多数情况下，所有的值应该大于`1`。 将其设置得更高还可以让您的服务器更好地应对禁止怪物生成的区域。

### [spigot.yml]

#### mob-spawn-range

`推荐值: 3`

此项可以限制玩家周围生成怪物的范围 (区块为单位)。 考虑到服务器的游戏玩法和在线人数，您或许应该一并调整 [bukkit.yml] 里的`spawn-limits`。 将其设置得较低会让玩家感觉周围有更多的怪物。 这应该低于或等于您的`simulation-distance 模拟距离`，并且永远不要大于您的`despawn-ranges / 16`。

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

此项可以设置实体的激活AI距离（方块为单位）。降低这些值有助于提高性能，但可能会导致怪物反应迟钝。将此值降低太多可能会破坏某些生物农场；比如刷铁机。

#### entity-tracking-range

```
推荐值:

      players: 48
      animals: 48
      monsters: 48
      misc: 32
      other: 64
```

此项可以设置实体的可见范围（方块为单位）。 距离外的实体将不会发送给客户端。 如果设置得太低，这可能会导致怪物突然出现在玩家附近。 此项应该大于`entity-activation-range`的值。

#### tick-inactive-villagers

`推荐值: false`

是否应该在实体激活范围之外正常激活村民。 这将使村民正常工作并忽略激活范围。 禁用此功能将有助于提高性能，但在某些情况下会让玩家困惑。 此项还对刷铁机等造成较大影响。

#### nerf-spawner-mobs

`推荐值: true`

此项可以卸载刷怪笼生成的生物的AI。 被卸载AI的生物将不会做任何事情。 如果想让它们在水中跳跃（用于刷怪塔），只需将 [paper-world configuration] 中`spawner-nerfed-mobs-should-jump`设为`true`。

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

此项可以调整各种生物的消失范围（方块为单位）。降低这些值可以更快地清除远离玩家的生物。 您应该将 soft 软距离设置为约`30`，然后将 hard 硬性距离设置的稍微大于 simulation-distance，这样当玩家刚刚跑出区块时，生物不会立即消失（您可以一并调整 [paper-world configuration] 中的`delay-chunk-unloads-by`）。 当一个生物离开了 hard 距离，该生物会立刻消失。 当一个生物处于 soft 和 hard 距离之间，该生物将有概率消失。 您的 hard 距离应该大于 soft 距离。您应该根据模拟距离调整此项：`(simulation-distance * 16) + 8`。 此项还可能造成玩家经过后，区块不卸载的情况（因为生物还没消失）。

#### per-player-mob-spawns

`推荐值: true`

该项决定生物生成是否应该考虑玩家周围已有的生物数量。此项可以解决一些生物生成不一致问题，如玩家建造的农场占满生物生成上限。这更像单人游戏的生物生成，允许您设置较低的`spawn-limits`。启用此项会对性能产生轻微影响，但它带来的好处大于其影响。

#### max-entity-collisions

`推荐值: 2`

覆盖 [spigot.yml] 中的同名项。它让您决定一个实体可以同时处理多少次碰撞。`0`将导致无法推动其他实体，包括玩家。`2`应该可以处理大部分情况。 值得注意的是，这将会破坏 maxEntityCramming gamerule 也就是生物堆叠窒息。

#### update-pathfinding-on-block-update

`推荐值: false`

禁用此项将减少寻路次数，从而提高性能。在某些情况下，这会导致生物看起来更加迟钝；它们只会每 5 个 tick（0.25 秒）被动更新一次路径。

#### fix-climbing-bypassing-cramming-rule

`推荐值: true`

是否修复实体在攀爬时不受实体挤压影响的问题。这将防止大量生物在攀爬时堆叠在狭小空间内（例如蜘蛛）。

#### armor-stands.tick

`推荐值: false`

在大部分情况下，将该项设置为`false`是安全的。如果您使用盔甲架或任何相关的插件时遇到了问题，请重新启用它。这将防止盔甲架被水推动或受到重力的影响。

#### armor-stands.do-collision-entity-lookups

`推荐值: false`

是否启用盔甲架碰撞。如果您有很多盔甲架，并且不想它们与任何东西发生碰撞，这将有所帮助。

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

> 当 [Pufferfish's DAB](#dabenabled) 启用时，不建议修改该项任何默认值。

这决定了触发AI行为和传感器的间隔。 `acquirepoi`是村民最频繁的行为, 因此它的间隔已经大大增加了。 如果村民有寻路问题，请减少此项。

### [pufferfish.yml]

#### dab.enabled

`推荐值: true`

DAB（大脑动态激活）会随着实体与玩家距离的增大而减少实体运算次数。 DAB 采用梯度工作，而不是像`entity-activation-range`那样采用定值。 DAB 基于此结果 [dab.activation-dist-mod](#dabactivation-dist-mod)。

#### dab.max-tick-freq

`推荐值: 20`

无论 DAB 的结果如何，实体的 tick 都不会低于此值。如果将其设置为 20，则无论实体有多远，它每秒都会被计算至少一次。增加此值可能会提高远距离实体的性能，但可能会破坏农场或大大削弱生物行为。如果启用 DAB 会破坏刷怪塔，请尝试降低此值。

#### dab.activation-dist-mod

`推荐值: 8 或 7`

控制 DAB 的梯度。增加此值可使较远的实体更频繁地运算。减少此值可使较远的实体运算的更慢，从而提高 DAB 的性能，但会影响实体与周围环境的互动，并可能破坏刷怪塔。如果启用 DAB 会破坏刷怪塔，请尝试降低此值。

#### enable-async-mob-spawning

`推荐值: true`

是否应启用异步怪物生成。要使此功能正常工作，必须启用 Paper 中的`per-player-mob-spawns`。此项实际上不会异步生成生物，但会将生成新生物所涉及的大量计算工作转移到不同的线程。对原版生存体验的影响很小。

#### enable-suffocation-optimization

`推荐值: true`

此项将检查速率限制为伤害超时来优化窒息检查（检查生物是否在方块内以及它们是否应该受到窒息伤害）。除非您是生电玩家，能够使用精确计时窒息杀死实体的时间，否则这种优化应该是不可能注意到的。

#### inactive-goal-selector-throttle

`推荐值: true`

在实体非活动时限制其目标选择器，让非活动实体每`20 tick`更新一次其目标选择器，而不是每 tick 更新一次。可以将性能提高几个百分点，而且对游戏体验的影响很小。

### [purpur.yml]

#### zombie.aggressive-towards-villager-when-lagging

`推荐值: false`

当 TPS 低于`lagging-threshold`值 [purpur.yml] 时，启用此项会阻止僵尸追逐村民。

#### entities-can-use-portals

`推荐值: false`

此项决定是否让除玩家之外的所有实体使用传送门。这可以防止实体在主线程上加载不必要的区块，但会破坏一些生电装置或地狱交通。

#### villager.lobotomize.enabled

`推荐值: true`

> 仅当村民造成服务器卡顿时才应启用此项！否则，村民寻路会出现问题。

村民被卸载了AI后只会按时补货。启用此项会禁用村民自动寻路。

#### villager.search-radius

```
推荐值:

          acquire-poi: 16
          nearest-bed-sensor: 16
```

该项可以调整村民尝试搜索工作方块和床的半径。这大大提高了村民的性能，但会阻止他们探测到比设定值更远的工作方块或床。

---

## 杂项

### [spigot.yml]

#### merge-radius

```
推荐值:

      item: 3.5
      exp: 4.0
```

此项设置同类物品和经验球合并堆叠的距离，可减少地面未拾取物数量。 设置得太高会导致物品合并时像瞬间传送。也会使得物品穿过方块，可能破坏一些刷怪塔。 此项不会判断物品是否穿过墙壁 （除非开启 Paper 中的`fix-items-merging-through-walls）。经验球仅会在生成时合并。建议使用`alt-item-despawn-rate`来优化掉落物数量。

#### hopper-transfer

`推荐值: 8`

漏斗移动一个物品的频率（以 tick 为单位）。如果服务器上有大量漏斗，增加此值将有助于提高性能，但如果设置得太高，会破坏基于漏斗的红石计时器，甚至可能破坏物品分类装置。

#### hopper-check

`推荐值: 8`

漏斗检查上方的物品或容器的频率（以 tick 为单位）。如果服务器上有大量漏斗，增加此值将有助于提高性能，但如果设置得太高，会破坏基于漏斗的红石计时器，甚至可能破坏物品分类装置。

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

此项可以设置指定物品消失的时间（tick 为单位）。 建议用此项替代扫地姬或`merge-radius`来提高性能。

#### redstone-implementation

`推荐值: ALTERNATE_CURRENT`

将红石系统替换为优化版本，减少冗余更新，降低服务器必须计算的逻辑量。可能会对个别的红石机器产生影响，但其提升利大于弊。甚至还可以修复 Bukkit 造成的红石同步问题。

`ALTERNATE_CURRENT`是基于 [Alternate Current](https://modrinth.com/mod/alternate-current)。 更多信息请阅读该页面。

#### hopper.disable-move-event

`推荐值: false`

仅当有插件监听`InventoryMoveItemEvent`时才会触发该事件。 **如果您想使用侦听此事件的插件,请不要设置为 true，比如保护插件！**

#### hopper.ignore-occluding-blocks

`推荐值: true`

确定漏斗是否会忽略完整方块内的容器，例如沙子或沙砾中的漏斗矿车。启用该项可能会破坏一些红石装置。

#### tick-rates.mob-spawner

`推荐值: 2`

此项调整刷怪笼的刷新频率。如果服务器有大量刷怪笼，则值越高意味着卡顿越少，但如果设置得太高，怪物刷怪率就会降低。

#### optimize-explosions

`推荐值: true`

将此项设为`true`可以将原版爆炸算法替换成优化版本，但计算爆炸伤害时会略有不准确。 这通常不影响游戏体验。

#### treasure-maps.enabled

`推荐值: false`

生成藏宝图的性能占用极高，如果要定位的结构位于未生成的区块中，服务器甚至可能会未响应。只有在您预生成世界并设置原版世界边界的情况下，启用此功能才是安全的。

#### treasure-maps.find-already-discovered

```
推荐值:
      loot-tables: true
      villager-trade: true
```

此项的默认值强制新藏宝图寻找未探索过的结构，这些结构通常位于尚未生成的区块中。将其设置为 true 可使地图指向之前发现过的结构。如果不将其更改为 true，在生成新的藏宝图时可能会遇到服务器未响应或崩溃的情况。 `villager-trade`影响村民交易的地图，而`loot-tables`影响任何生成战利品的容器，如宝箱等。

#### tick-rates.grass-spread

`推荐值: 4`

服务器尝试扩散草方块或菌丝的频率（以 tick 为单位）。这会使大面积泥土需要更长的时间才能变成草或菌丝。如果您想在不明显降低扩散率的情况下优化性能，则将其设置为`4`左右应该不错。

#### tick-rates.container-update

`推荐值: 1`

容器更新频率（以 tick 为单位）。如果容器更新带来一些问题（这种情况很少发生），增加此间隔时间可能会有所帮助，但会让玩家在与库存（幽灵物品）交互时更容易遇到不同步的情况。

#### non-player-arrow-despawn-rate

`推荐值: 20`

怪物射出的箭消失的时间（以 tick 为单位）。因为玩家无法捡起这些箭，所以您不妨将其设置为`20`（1 秒）之类的值。

#### creative-arrow-despawn-rate

`推荐值: 20`

创造模式玩家射出的箭消失的时间（以 tick 为单位）。因为玩家无法捡起这些箭，所以您不妨将其设置为`20`（1 秒）之类的值。

### [pufferfish.yml]

#### disable-method-profiler

`推荐值: true`

此选项将禁用游戏进行的一些其他性能分析。这些分析是非必需的，并且可能会导致额外的延迟。

### [purpur.yml]

#### dolphin.disable-treasure-searching

`推荐值: true`

禁止海豚寻宝。

#### teleport-if-outside-border

`推荐值: true`

如果玩家恰好在世界边界之外，该项会将其传送到世界出生点。这很有用，因为原版世界边界是可绕过的。

---

## 辅助项

### [paper-world configuration]

#### anti-xray.enabled

`推荐值: true`

是否开启 Paper 自带的反矿透。更多信息请查看 [Configuring Anti-Xray](https://docs.papermc.io/paper/anti-xray)。 启用该功能会降低性能，但它比所有的反 xray 插件都更有效。在大多数情况下，对性能的影响可以忽略不计。

#### nether-ceiling-void-damage-height

`推荐值: 127`

如果此项大于`0`，地狱此高度之上的玩家就会像坠入虚空一样不断受到伤害。 防止玩家在地狱天花板之上建造建筑，原版地狱为128格高，所以您应该考虑将数值设定为`127`。 如果您以任何方式修改了下界的高度，您应该将其设置为`[地狱高度] - 1`。

---

# Java启动参数
[我的世界1.20.5+ 服务端或客户端需要使用 Java 21 及以上](https://docs.papermc.io/java-install-update)。 Oracle 更改了他们的许可证，并且不再有更好的理由从他们那里获取 Java。 推荐从 [Adoptium](https://adoptium.net/) 或 [Amazon Corretto](https://aws.amazon.com/corretto/) 获取Java。 其他 JVM（例如 OpenJ9 或 GraalVM）也可以运行，但是 Paper 不支持它们并且已知会导致问题，因此目前不推荐使用它们。

您可以配置垃圾回收（GC）以减少由大型垃圾回收任务引起的卡顿。 您可以找到针对我的世界服务器优化的启动项 [点我](https://docs.papermc.io/paper/aikars-flags) [`SOG`]。
推荐使用 [flags.sh](https://flags.sh) 来一键生成服务器启动脚本。

此外，在启动项中添加`--add-modules=jdk.incubator.vector`可提升性能。 此项使 Pufferfish 能够在您的 CPU 上使用 SIMD 指令，从而加快某些数学运算速度。 目前，它仅用于提高地图画的渲染速度至8倍。

# 不推荐使用的插件

## 扫地姬
请调整 [merge-radius](#merge-radius) 和 [alt-item-despawn-rate](#alt-item-despawn-rate)，而不是使用扫地姬，它们会占用更多的资源来扫描和删除物品。

## 生物堆叠插件
除非服务器上有大量的刷怪笼，否则对生物进行堆叠仍然会使服务器浪费性能在尝试生成更多的生物上。别用。

## 任何用于热加载/卸载的插件
在运行时启用或禁用是极其危险的。加载这样的插件可能会导致跟踪数据出现错误，而禁用插件则可能因删除依赖项而导致错误。 bukkit 自带的`/reload`命令也存在同样的问题，详细请阅读 [me4502's blog post](https://madelinemiller.dev/blog/problem-with-reload/)。 请使用每个插件自带的`reload`配置文件指令，重启服务器来增加或删除插件。

# 什么是卡顿? - 如何衡量

## mspt
Paper 提供了`/mspt`命令来告诉您服务器计算最近的 tick 所花费的时间。如果您看到的第一个和第二个值低于 50，那么恭喜您！您的服务器没有卡顿！如果第三个值超过 50，则意味着至少有1个 tick 超时了。这是完全正常的，有时会发生，所以不要惊慌。
  
## Spark
[Spark](https://spark.lucko.me/) 是一个允许您分析服务器的 CPU 和内存使用情况的插件。 你可以阅读此文 [点我](https://spark.lucko.me/docs/)。 这里还有关于如何查找卡顿原因的指南 [这里](https://spark.lucko.me/docs/guides/Finding-lag-spikes)。

## Timings
Timings 是一款工具，可让您准确了解哪些任务耗时最长。它是最基本的故障排除工具。众所周知，Timings 会对服务器的性能产生严重影响，建议使用 Spark 插件而不是 Timings，并使用 Purpur 或 Pufferfish 禁用 Timings。

想获取服务器的 Timings, 您只需要输入`/timings paste`指令并复制生成的链接。您可以与其他人分享此链接，以便他们帮助您。 Timings 很容易误读。这里有视频介绍如何阅读它们 [视频 by Aikar](https://www.youtube.com/watch?v=T4J0A9l7bfQ)。

---

# Minecraft 漏洞及其修复方法
要了解如何修复可能导致服务器卡顿或崩溃的漏洞，请参阅 [这里](https://github.com/YouHaveTrouble/minecraft-exploits-and-how-to-fix-them)。

[`SOG`]: https://www.spigotmc.org/threads/guide-server-optimization%E2%9A%A1.283181/
[server.properties]: https://docs.papermc.io/paper/reference/server-properties
[bukkit.yml]: https://docs.papermc.io/paper/reference/bukkit-configuration
[spigot.yml]: https://docs.papermc.io/paper/reference/spigot-configuration
[paper-global configuration]: https://docs.papermc.io/paper/reference/global-configuration
[paper-world configuration]: https://docs.papermc.io/paper/reference/world-configuration
[purpur.yml]: https://purpurmc.org/docs/Configuration/
[pufferfish.yml]: https://docs.pufferfish.host/setup/pufferfish-fork-configuration/
