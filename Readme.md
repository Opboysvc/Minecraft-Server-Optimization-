# Minecraft server optimization guide
Note for users that are on vanilla, Fabric or Spigot (or anything below Paper) - go to your server.properties and change sync-chunk-writes to false. This option is forcibly set to false on Paper and its forks, but on other server implementations you need to switch this to false manually. This allows the server to save chunks off the main thread, lessening the load on the main tick loop.

Guide for version 1.20. Some things may still apply to 1.15 - 1.19.

Based on this guide and other sources (all of them are linked throughout the guide when relevant).

Use the table of contents located above (next to README.md) to easily navigate throughout this guide.

Intro
There will never be a guide that will give you perfect results. Each server has their own needs and limits on how much you can or are willing to sacrifice. Tinkering around with the options to fine tune them to your servers needs is what it's all about. This guide only aims to help you understand what options have impact on performance and what exactly they change. If you think you found inaccurate information within this guide, you're free to open an issue or set up a pull request to correct it.

Preparations
Server JAR
Your choice of server software can make a huge difference in performance and API possibilities. There are currently multiple viable popular server JARs, but there are also a few that you should stay away from for various reasons.

Recommended top picks:

Paper - The most popular server software that aims to improve performance while fixing gameplay and mechanics inconsistencies.
Pufferfish - Paper fork that aims to further improve server performance.
Purpur - Pufferfish fork focused on features and the freedom of customization.
You should stay away from:

Any paid server JAR that claims async anything - 99.99% chance of being a scam.
Bukkit/CraftBukkit/Spigot - Extremely outdated in terms of performance compared to other server software you have access to.
Any plugin/software that enables/disables/reloads plugins on runtime. See this section to understand why.
Many forks further downstream from Pufferfish or Purpur will encounter instability and other issues. If you're seeking more performance gains, optimize your server or invest in a personal private fork.
Map pregen
Map pregeneration, thanks to various optimizations to chunk generation added over the years is now only useful on servers with terrible, single threaded, or limited CPUs. Though, pregeneration is commonly used to generate chunks for world-map plugins such as Pl3xMap or Dynmap.

If you still want to pregen the world, you can use a plugin such as Chunky to do it. Make sure to set up a world border so your players don't generate new chunks! Note that pregenning can sometimes take hours depending on the radius you set in the pregen plugin. Keep in mind that with Paper and above your tps will not be affected by chunk loading, but the speed of loading chunks can significantly slow down when your server's cpu is overloaded.

It's key to remember that the overworld, nether and the end have separate world borders that need to be set up for each world. The nether dimension is 8x smaller than the overworld (if not modified with a datapack), so if you set the size wrong your players might end up outside of the world border!

Make sure to set up a vanilla world border (/worldborder set [diameter]), as it limits certain functionalities such as lookup range for treasure maps that can cause lag spikes.

Configurations
Networking
server.properties
network-compression-threshold
Good starting value: 256

This allows you to set the cap for the size of a packet before the server attempts to compress it. Setting it higher can save some CPU resources at the cost of bandwidth, and setting it to -1 disables it. Setting this higher may also hurt clients with slower network connections. If your server is in a network with a proxy or on the same machine (with less than 2 ms ping), disabling this (-1) will be beneficial, since internal network speeds can usually handle the additional uncompressed traffic.

purpur.yml
use-alternate-keepalive
Good starting value: true

You can enable Purpur's alternate keepalive system so players with bad connection don't get timed out as often. Has known incompatibility with TCPShield.

Enabling this sends a keepalive packet once per second to a player, and only kicks for timeout if none of them were responded to in 30 seconds. Responding to any of them in any order will keep the player connected. AKA, it won't kick your players because 1 packet gets dropped somewhere along the lines
~ https://purpurmc.org/docs/Configuration/#use-alternate-keepalive

Chunks
server.properties
simulation-distance
Good starting value: 4

Simulation distance is distance in chunks around the player that the server will tick. Essentially the distance from the player that things will happen. This includes furnaces smelting, crops and saplings growing, etc. This is an option you want to purposefully set low, somewhere around 3 or 4, because of the existence of view-distance. This allows to load more chunks without ticking them. This effectively allows players to see further without the same performance impact.

view-distance
Good starting value: 7

This is the distance in chunks that will be sent to players, similar to no-tick-view-distance from paper.

The total view distance will be equal to the greatest value between simulation-distance and view-distance. For example, if the simulation distance is set to 4, and the view distance is 12, the total distance sent to the client will be 12 chunks.

spigot.yml
view-distance
Good starting value: default

This value overwrites server.properties one if not set to default. You should keep it default to have both simulation and view distance in one place for easier management.

paper-world configuration
delay-chunk-unloads-by
Good starting value: 10s

This option allows you to configure how long chunks will stay loaded after a player leaves. This helps to not constantly load and unload the same chunks when a player moves back and forth. Too high values can result in way too many chunks being loaded at once. In areas that are frequently teleported to and loaded, consider keeping the area permanently loaded. This will be lighter for your server than constantly loading and unloading chunks.

max-auto-save-chunks-per-tick
Good starting value: 8

Lets you slow down incremental world saving by spreading the task over time even more for better average performance. You might want to set this higher than 8 with more than 20-30 players. If incremental save can't finish in time then bukkit will automatically save leftover chunks at once and begin the process again.

prevent-moving-into-unloaded-chunks
Good starting value: true

When enabled, prevents players from moving into unloaded chunks and causing sync loads that bog down the main thread causing lag. The probability of a player stumbling into an unloaded chunk is higher the lower your view-distance is.

entity-per-chunk-save-limit
Good starting values:

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
With the help of this entry you can set limits to how many entities of specified type can be saved. You should provide a limit for each projectile at least to avoid issues with massive amounts of projectiles being saved and your server crashing on loading that. You can put any entity id here, see the minecraft wiki to find IDs of entities. Please adjust the limit to your liking. Suggested value for all projectiles is around 10. You can also add other entities by their type names to that list. This config option is not designed to prevent players from making large mob farms.

pufferfish.yml
max-loads-per-projectile
Good starting value: 8

Specifies the maximum amount of chunks a projectile can load in its lifetime. Decreasing will reduce chunk loads caused by entity projectiles, but could cause issues with tridents, enderpearls, etc.

Mobs
bukkit.yml
spawn-limits
Good starting values:

    monsters: 20
    animals: 5
    water-animals: 2
    water-ambient: 2
    water-underground-creature: 3
    axolotls: 3
    ambient: 1
The math of limiting mobs is [playercount] * [limit], where "playercount" is current amount of players on the server. Logically, the smaller the numbers are, the less mobs you're gonna see. per-player-mob-spawn applies an additional limit to this, ensuring mobs are equally distributed between players. Reducing this is a double-edged sword; yes, your server has less work to do, but in some gamemodes natural-spawning mobs are a big part of a gameplay. You can go as low as 20 or less if you adjust mob-spawn-range properly. Setting mob-spawn-range lower will make it feel as if there are more mobs around each player. If you are using Paper, you can set mob limits per world in paper-world configuration.

ticks-per
Good starting values:

    monster-spawns: 10
    animal-spawns: 400
    water-spawns: 400
    water-ambient-spawns: 400
    water-underground-creature-spawns: 400
    axolotl-spawns: 400
    ambient-spawns: 400
This option sets how often (in ticks) the server attempts to spawn certain living entities. Water/ambient mobs do not need to spawn each tick as they don't usually get killed that quickly. As for monsters: Slightly increasing the time between spawns should not impact spawn rates even in mob farms. In most cases all of the values under this option should be higher than 1. Setting this higher also allows your server to better cope with areas where mob spawning is disabled.

spigot.yml
mob-spawn-range
Good starting value: 3

Allows you to reduce the range (in chunks) of where mobs will spawn around the player. Depending on your server's gamemode and its playercount you might want to reduce this value along with bukkit.yml's spawn-limits. Setting this lower will make it feel as if there are more mobs around you. This should be lower than or equal to your simulation distance, and never larger than your hard despawn range / 16.

entity-activation-range
Good starting values:

      animals: 16
      monsters: 24
      raiders: 48
      misc: 8
      water: 8
      villagers: 16
      flying-monsters: 48
You can set what distance from the player an entity should be for it to tick (do stuff). Reducing those values helps performance, but may result in irresponsive mobs until the player gets really close to them. Lowering this too far can break certain mob farms; iron farms being the most common victim.

entity-tracking-range
Good starting values:

      players: 48
      animals: 48
      monsters: 48
      misc: 32
      other: 64
This is distance in blocks from which entities will be visible. They just won't be sent to players. If set too low this can cause mobs to seem to appear out of nowhere near a player. In the majority of cases this should be higher than your entity-activation-range.

tick-inactive-villagers
Good starting value: false

This allows you to control whether villagers should be ticked outside of the activation range. This will make villagers proceed as normal and ignore the activation range. Disabling this will help performance, but might be confusing for players in certain situations. This may cause issues with iron farms and trade restocking.

nerf-spawner-mobs
Good starting value: true

You can make mobs spawned by a monster spawner have no AI. Nerfed mobs will do nothing. You can make them jump while in water by changing spawner-nerfed-mobs-should-jump to true in paper-world configuration.

paper-world configuration
despawn-ranges
Good starting values:

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
Lets you adjust entity despawn ranges (in blocks). Lower those values to clear the mobs that are far away from the player faster. You should keep soft range around 30 and adjust hard range to a bit more than your actual simulation-distance, so mobs don't immediately despawn when the player goes just beyond the point of a chunk being loaded (this works well because of delay-chunk-unloads-by in paper-world configuration). When a mob is out of the hard range, it will be instantly despawned. When between the soft and hard range, it will have a random chance of despawning. Your hard range should be larger than your soft range. You should adjust this according to your view distance using (simulation-distance * 16) + 8. This partially accounts for chunks that haven't been unloaded yet after player visited them.

per-player-mob-spawns
Good starting value: true

This option decides if mob spawns should account for how many mobs are around target player already. You can bypass a lot of issues regarding mob spawns being inconsistent due to players creating farms that take up the entire mobcap. This will enable a more singleplayer-like spawning experience, allowing you to set lower spawn-limits. Enabling this does come with a very slight performance impact, however it's impact is overshadowed by the improvements in spawn-limits it allows.

max-entity-collisions
Good starting value: 2

Overwrites option with the same name in spigot.yml. It lets you decide how many collisions one entity can process at once. Value of 0 will cause inability to push other entities, including players. Value of 2 should be enough in most cases. It's worth noting that this will render maxEntityCramming gamerule useless if its value is over the value of this config option.

update-pathfinding-on-block-update
Good starting value: false

Disabling this will result in less pathfinding being done, increasing performance. In some cases this will cause mobs to appear more laggy; They will just passively update their path every 5 ticks (0.25 sec).

fix-climbing-bypassing-cramming-rule
Good starting value: true

Enabling this will fix entities not being affected by cramming while climbing. This will prevent absurd amounts of mobs being stacked in small spaces even if they're climbing (spiders).

armor-stands.tick
Good starting value: false

In most cases you can safely set this to false. If you're using armor stands or any plugins that modify their behavior and you experience issues, re-enable it. This will prevent armor stands from being pushed by water or being affected by gravity.

armor-stands.do-collision-entity-lookups
Good starting value: false

Here you can disable armor stand collisions. This will help if you have a lot of armor stands and don't need them colliding with anything.

tick-rates
Good starting values:

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
It is not recommended to change these values from their defaults while Pufferfish's DAB is enabled!

This decides how often specified behaviors and sensors are being fired in ticks. acquirepoi for villagers seems to be the heaviest behavior, so it's been greately increased. Decrease it in case of issues with villagers finding their way around.

pufferfish.yml
dab.enabled
Good starting value: true

DAB (dynamic activation of brain) reduces the amount an entity is ticked the further away it is from players. DAB works on a gradient instead of a hard cutoff like EAR. Instead of fully ticking close entities and barely ticking far entities, DAB will reduce the amount an entity is ticked based on the result of a calculation influenced by dab.activation-dist-mod.

dab.max-tick-freq
Good starting value: 20

Defines the slowest amount entities farthest from players will be ticked. Increasing this value may improve the performance of entities far from view but may break farms or greatly nerf mob behavior. If enabling DAB breaks mob farms, try decreasing this value.

dab.activation-dist-mod
Good starting value: 7

Controls the gradient in which mobs are ticked. Decreasing this will activate DAB closer to players, improving DAB's performance gains, but will affect how entities interact with their surroundings and may break mob farms. If enabling DAB breaks mob farms, try increasing this value.

enable-async-mob-spawning
Good starting value: true

If asynchronous mob spawning should be enabled. For this to work, the Paper's per-player-mob-spawns setting must be enabled. This option does not actually spawn mobs asynchronous, but does offload much of the computational effort involved with spawning new mobs to a different thread. Enabling this option should not be noticeable on vanilla gameplay.

enable-suffocation-optimization
Good starting value: true

This option optimises a suffocation check (the check to see if a mob is inside a block and if they should take suffocation damage), by rate limiting the check to the damage timeout. This optimisation should be impossible to notice unless you're an extremely technical player who's using tick-precise timing to kill an entity at exactly the right time by suffocation.

inactive-goal-selector-throttle
Good starting value: true

Throttles the AI goal selector in entity inactive ticks, causing the inactive entities to update their goal selector every 20 ticks instead of every tick. Can improve performance by a few percent, and has minor gameplay implications.

purpur.yml
zombie.aggressive-towards-villager-when-lagging
Good starting value: false

Enabling this will cause zombies to stop targeting villagers if the server is below the tps threshold set with lagging-threshold in purpur.yml.

entities-can-use-portals
Good starting value: false

This option can disable portal usage of all entities besides the player. This prevents entities from loading chunks by changing worlds which is handled on the main thread. This has the side effect of entities not being able to go through portals.

villager.lobotomize.enabled
Good starting value: true

This should only be enabled if villagers are causing lag! Otherwise, the pathfinding checks may decrease performance.

Lobotomized villagers are stripped from their AI and only restock their offers every so often. Enabling this will lobotomize villagers that are unable to pathfind to their destination. Freeing them should unlobotomize them.

villager.search-radius
Good starting values:

          acquire-poi: 16
          nearest-bed-sensor: 16
Radius within which villagers will search for job site blocks and beds. This significantly boosts performance with large amount of villagers, but will prevent them from detecting job site blocks or beds that are further away than set value.

Misc
spigot.yml
merge-radius
Good starting values:

      item: 3.5
      exp: 4.0
This decides the distance between the items and exp orbs to be merged, reducing the amount of items ticking on the ground. Setting this too high will lead to the illusion of items or exp orbs disappearing as they merge together. Setting this too high will break some farms, as well as allow items to teleport through blocks. There are no checks done to prevent items from merging through walls (unless Paper's fix-items-merging