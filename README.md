# Nusantara Data

Game data for Nusantara game. It contains all the game's art, sounds, config and scripts. Feel free to fork and alter for modding purposes.

Balance proposals are welcome via PRs.

[Play Nusantara](https://playnusantara.com)

# Nusantara modding guide
This document shows how to modify existing and implement new features using the Nusantara engine. Almost all content in the base game is implementable using these methods.

In general, modding boils down to modifying **config files** and **scripts**. [VSCode](https://code.visualstudio.com/) works well enough for both purposes.
1. Config files are written in [YAML](https://yaml.org/) and their extension is **.yaml**. They contain "static" data that describes, e.g, all the units in a faction or the contents of a map.
  * [VSCode extension for YAML files](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)
1. Script files are written in [NusantaraScript language](https://github.com/globulus/nusantara-script) and their extension is **.ns**. They describe dynamic behaviours at game's runtime, such as effects, scenarios, AI behavior, etc.
  * [VSCode extension for NS files](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)

## The *data* folder
At startup, Nusantara loads its content from the **data** folder, which has to be located in its persistent storage location. E.g, on Windows 10, this usually resides at `C:\Users\User\AppData\LocalLow\Globulus\Nusantara\data`. Modding the game involves making changes to its subfolders and files, all of which is discussed in the rest of this document.

The structure of the **data** folder and its subfolders, as well as names of the files are sometimes important, so make sure not to leave anything out. The details are discussed in the sections below.

Loading might fail due to invalid file structure, compilation errors in scripts or internal validation errors, all of which will be apparent during the loading process. If the game data is corrupt for whatever reason, the game won't be playable, so always be sure to **back up your existing data before modding**.

## Meta.yaml
The single standalone file in the **data** folder is **Meta.yaml**. It contains metadata that describe the game data and is used by the game to automatically check for updates.

This file describes a single object whose structure is:
* `version` - the version code of the data file. If modding, bump this up to a high value so that it doesn't get overwritten by game's automatic update system.

## Bots
The **Bots** folder contains config and script files that describe the game's bots, i.e AI entities that can play the game and simulate human players.

### Config files
There can be as many config (.yaml) files in the folder, each one of them describing a list of bots - `List<{id: string, title: string}>`.
* The `id` of the bot must correspond to an `ai` class defined somewhere in scripts (hence the PascalCase naming).
* The `title` should be a translation key.

Example:
```yaml
- id: EasyAi
  title: easyAiTitle
- id: NormalAi
  title: normalAiTitle
- id: HardAi
  title: hardAiTitle
```

### Script files
There can be as many script (.ns) files in the folder. Ideally, they should contain the code that describes `ai` classes corresponding to those listed in config files.

There's no hard and fast rule how an AI should be implemented. The game expects two methods to be implemented:
* `awake` is called when the AI assumes control of a player at the start of their turn. E.g, if your AI has a hard limit of only 10 actions per turn, you can reset the counter in this method.
* `update` is called by the game server repeatedly until it returns `false` - if it returns `true`, it means that AI still has work to do in this turn, so the `update` method will be invoked again. Once this method returns `false`, the game server will end the AI player's turn automatically.

There is the optional third method, `onGameStart(player)`, which is called just once, when the game first starts. Its argument is the player object which the AI will drive. You can use this to perform any additional modifications to the player, such as cheating by giving it extra resources.

The default bots use the "value AI" approach, where a series of possible decisions are weighted, with the one with the highest core executed for the given run. Again, the framework is flexible enough to allow for implementation of different kinds of AIs for different purposes (build tree, passive, etc.).

## Campaigns

## Effects
The **Effects** folder contains only script files (.ns), which should represent generally define different `effect` classes used in the game. Effects are events that happen based on some trigger, such as start of turn, unit attacking or defending, ability being cast, or just by some unit or upgrade having an ability with an effect attached to it. They can also define temporary effects, which are attached to units and players at runtime. The versatility of effects allows you to do wonders with gameplay, experimenting with weird abilities, auras, attacks, etc.

An `effect` class can implement the following fields/methods:
* `trigger` **(required)** - describes events that can trigger this effect. Each event can have one or more triggers, all of which are fields on the `Triggers` object.
  + If you have a single trigger, return just its value: `trigger => Triggers.aura`.
  + If you have multiple triggers, return a list: `trigger => [Triggers.attack, Triggers.retaliate]`.
  + Available triggers:
    - `startOfTurn` - target is null.
    - `endOfTurn` - target is null.
    - `onDie` - before the unit actually dies. Target is null.
    - `afterDie` - after the unit dies. Target is the cell on which the unit died.
    - `attack` - after a unit attacks. Target is unit being attacked.
    - `defend` - after a unit survives damage. Target is the attacking unit.
    - `retaliate` - after a unit retaliates. Target is the attacking unit.
    - `kill` - after a kill. Target is the killed unit.
    - `cast` - target varies, can be null, a unit or a cell.
    - `passive` - target is null.
    - `aura` - target is null.
    - `interact` - when a unit is interacted with. Caster is the interactable unit, target is unit that initiated interaction.
* `targets` (optional) - if the effect requires a target (for those with `cast` trigger), defines the possible targets, all of which are fields on the `Targets` object:
  + `none` - the cast happens without targetting in the UI.
  + `unit` (default) - requires a unit target.
  + `cell` - requires a cell target.
* `cost(caster)` (optional) - defines the cost that has to be paid by the caster (or its owner player). If the cost cannot be paid, the effect won't be executed. The expected result is a cost object, which optionally has `g`, `u` and/or `ap` fields, e.g `[g: 3, u: 2, ap: 1]`.
* `range(caster)` (optional) - the range from which the effect can be cast, used in cast and attack/defense/retaliate effects. Should return an `Int` representing the distance in cells between the caster and target, at or under which the effect can be cast.
* `isEnabled(caster)` (optional) - defines a list of conditions that have to be satisfied for the effect to be castable. The list contains lists of two elements, the first of which is a `Bool` expression that checks a necessary precondition for enabling the effect, while the second is an error message that is shown in the UI if that expression is `false`. All pairs in the list must resolve to `true` for the effect to be considered enabled. Example:
```ruby
isEnabled(caster) => [
  [caster.owner.canAddCards, "Your hand is full!"],
  [hasMajorUpgrade(caster.owner), "You need a Major Upgrade!"]
]
```
* `check(caster, target)` (optional) - checks if the effect will run when applied with the given caster and target. It should return a `Bool` value. Example:
```ruby
check(caster, target) => areAllies(caster, target) && !isOrganic(target) && target.hp <= target.health - @repairAmount
```
* `run(caster, target)` **(required)** - the heart of the effect, it is the code that is executed when the effect happens. Before its execution, the cost defined in `cost(caster)`, if any, is paid by the caster or its owner. It can return a result that can be used elsewhere in the code.
* `sprite` (optional) - key of the sprite associated with the effect. Must be the `Str` `id` of an image holder.
* `title` (optional) - the title of the effect that can be shown in the UI. 
* `description(caster)` (optional) - description of the effect that can be shown in the UI.
* `flavor(caster)` (optional) - flavor text of the effect that can be shown in the UI.

Example - the **TrainHero** effect:
```ruby
effect TrainHero {
  init(@type)

  trigger => Triggers.cast

  cost(caster) => [g: 10, ap: 1]

  targets => Targets.none

  isEnabled(caster) => [
    [caster.owner.hero == null, "You already have a hero!"],
    [hasVacantInfluencableCell(caster, 1), "You need free space around!"]
  ]

  check(caster, target) => hasVacantInfluencableCell(caster, 1)

  run(caster, target) {
    player = caster.owner
    targetCell = getVacantInfluencableCell(caster, 1)
    hero = targetCell.summon(type, player)
    player.setHero(hero)
    player.addToHand("heroUpgrade", 0)
  }

  description(caster) => "Summon this Hero at the nearest cell!\n" + Game.localizedStr(type + "Description")
  flavor(caster) => Game.localizedStr(type + "Flavor")
}
```

## Factions
The **Factions** folder contains data for game's factions, describing their metadata, units, spells and upgrades. Each faction should be in its own folder and it must contain a config (.yaml) file of the same name (e.g, **Frenza** folder contains **Frenza.yaml**).

Sound and image files can be organized in any fashion, but it's common to put them in **Sounds** and **Images** folders, respectively.

A faction config file describes an object with the following schema:
* `id` - the ID of the faction.
* `name` - UI-facing display name, should be a translatable key.
* `flavor` - flavor text of the faction, should be a translatable key.
* `image` - should be of a bit higher resolution, such as 256x256px.
* `playable` - determines if this is a faction playable in custom games or only in a scenario/campaign.
* `uniqueResource` - ID of the unique resource for this faction.
* `music` - list of sound files that consistute the musical back-drop for when this faction is played.
* `units` - list of [unit definitions](#unit).
* `heroes` - list of [hero definitions](#hero).
* `spells` - list of [spell definitions](#spell).
* `upgrades` - list of [upgrade definitions](#upgrade).

All factions need to have units. Heroes, spells and upgrades are optional, but only for non-playable factions, such as neutral or campaign factions,

### Playables
Units, heroes, spells and upgrades are all *playables*, meaning they share a set of properties. Here's a list of those properties:
* `id` **(required)**
* `title` **(required)** - should be a translatable key.
* `description` **(required)** - should be a translatable key.
* `flavor` **(required)** - should be a translatable key.
* `image` **(required)** - relative path to a 64x64px PNG.
* `sounds` (optional) - list of sounds that are played based on triggers. All triggers are optional. All sounds are specified as as relative paths to an OGG file. Sounds play only for the currently active player if they can see the action associated with the sound and if the sound is produced by their playable.
  + `summon` - when a: unit/hero is summoned on the map; spell is played from hand; upgrade is slotted.
  + `select` - when a unit/hero is selected.
  + `attack` - when a unit/hero is instructed to attack another.
  + `cast` - when a unit/hero is instructed to cast an ability.
  + `die` - when a unit/hero dies.
* `playCost` (optional) - cost that the owning player has to pay when playing this playable from hand. It is a cost object with two optional `Int` fields: `g` and `u`, standing for *gold* and *unique resource*, respectively.
* `tags` (optional) - list of IDs of [tags](#tags) applicable to this playable.
* `requiredUpgrades` (optional) - list of IDs for [upgrades](#upgrade) that need to be slotted for this to be playable from hand.

### Unit
A unit is a [playable](#playables) that represents an object on the game board. This includes creatures, buildings, interactable map objects, etc. Here is the list of available properties for a unit type:
* `merit` **(required)** - describes the importance of a unit and is used by AIs to decide on valuable targets or build orders. This is also the amount of XP a Hero gains when killing a unit. There's no good rule what this number should be, as it is based on the relative strengths of other units in the game.
* `h` **(required)** - represents the maximum health of a unit. It needs to be a positive number.
* `a` (optional) - attack value for this unit. Units with no attack value (or attack value of zero) cannot attack or retaliate.
* `as` (optional) - attack spread for this unit. When non-deterministic combat is enabled in a [ruleset](#rulesets), it represents the range around the attack value from which the actual attack value for each attack will be chosen. E.g, if a unit's attack is 10, and attack spread is 2, each attack will have a value from range [8, 12].
* `ac` (optional)  - attack count for this unit, defaults to 1. This is the number of times a unit will (try to) attack when engaging in combat.
* `rc` (optional) - retaliation count for this unit, defaults to 1. This is the number of times a unit can retaliate when attacked before its next turn (when retaliations refresh).
* `d` (optional) - defense value for this unit.
* `s` (optional) - speed value for this unit. It represents the number of APs that a unit regenerates at the start of its turn, as well as its maximum number of available APs.
* `r` (optional) - range value for this unit, defaults to 1. This is the range from which the unit can engage in combat, represented as number of cells between it and the opponent. Valid for both attack and retaliation.
* `reg` (optional) - regeneration value for this unit, defaults to 0. This is the number of HP a unit regenerates at the start of each turn. Can be a negative integer to represent automatic loss of health.
* `movementType` **(required)** - the [movement type](#movementTypes) for this unit.
* `projectile` (optional) - the ID of a [projectile](#projectile) this unit uses in combat.
* `sight` (optional) - the sight value for this unit, defaults to 1. This is the number of sorrounding cells from which the unit removes the fog of war.
* `influence` (optional) - describes the influence this unit exterts, if any. This is an object with two required properties:
  + `amount` - an integer describing the amount of influence exerted. The actual effect depends on `kind`. If this is zero, the unit doesn't extert any influence.
  + `kind` - the type of influence. Available options:
    - `radial` - influences all cells in `amount` radius around itself.
    - `spiral` - influences `amount` cells around itself, starting with the lower-right one.
    - `line` - influences `amount` cells in a straight line to the right of itself.
* `loadCapacity` (optional) - the number of units this unit can *load*, defaults to 0. Loaded units are gone from the board until unloaded. When a unit does, all of its loaded units die as well (although you can write abilities and effect that mitigate this).
* `applyTeamColor` (optional) - if this is `false`, the unit won't have its team color applied to the white pixels in its image.
* `stances` **(required)** - the list of IDs for [stances](#stances). Each unit **must have at least one stance defined**.
* `skills` (optional) - the list of IDs of [skills](#skills) that this unit has.
* `canInteract` (optional) - if this is `true`, the unit can interact with Neutral units that have an `interactionEffect` defined.
* `interactionEffect` (optional) - the ID of an [effect](#effects) that is invoked when another unit tries interacting with this one.
* `extras` (optional) - a list of bundled extra data that you can ship with a unit (and, of course, change at runtime). Each *extra* object has the following structure:
  + `id`
  + `title` - should be a translatable key.
  + `image`
  + `value` - this can be any kind of value - a number, string or boolean - depending on the actual context of the extra itself.

### Hero
A Hero is a [unit](#unit) with some extra properties. The game supports controlling a single Hero at the time. Here is the list of available properties for a hero type:
* `v` (optional) - initial Vitality value for this hero, defaults to 0. Has to be a non-negative integer.
* `f` (optional) - initial Fortitude value for this hero, defaults to 0. Has to be a non-negative integer.
* `g` (optional) - initial Guard value for this hero, defaults to 0. Has to be a non-negative integer.
* `p` (optional) - initial Prowess value for this hero, defaults to 0. Has to be a non-negative integer.
* `vitalityProgression` (optional) - how much vitality does the hero gain by leveling up, defaults to 0. Has to be a non-negative integer.
* `fortitudeProgression` (optional) - how much vitality does the hero gain by leveling up, defaults to 0. Has to be a non-negative integer.
* `guardProgression` (optional) - how much vitality does the hero gain by leveling up, defaults to 0. Has to be a non-negative integer.
* `prowessProgression` (optional) - how much vitality does the hero gain by leveling up, defaults to 0. Has to be a non-negative integer.
* `upgradeTree` **(required)** - this hero's [upgrade tree](#upgrade-tree), indicating skills that become available by leveling up.

### Spell
A Spell is a [playable](#playables) that holds a single [skill](#skills) which is executed when the spell is played (cast) from the hand. Here is the list of available properties for a spell type:
* `skill` **(required)** - the ID of the [skill](#skills) invoked when this is played.

### Upgrade
An Upgrade is a [playable](#playable) that can be slotted in an [upgrade tree](#upgrade-tree), be it a Faction or a Hero one. Upgrades have skills that are executed when they're played (for the `summon` trigger), passively or at start/end of turn. Here is the list of available properties for an upgrade type:
* `kind` **(required)** - the kind of the upgrade which determines which tree can it go into and which slots can it occupy. There are three upgrade kinds:
  + `minor` - minor slots in a faction upgrade tree.
  + `major` - major slots in a faction upgrade tree.
  + `hero` - slots in a hero upgrade tree.
* `skills` (optional) - the list of [skill](#skills) IDs associated with this upgrade.

## Upgrade Tree
Upgrade trees are properties of both factions (via [rulesets](#rulesets)) and [heroes](#hero). An upgrade tree is, well, a tree structure, which consists of nodes. Those nodes can have other nodes as their subnodes, etc.

Nodes are described by their `type` property, which is the ID of the [upgrade](#upgrade) that can be slotted at that node. Each node can only have a single type.

A node can have multiple parents (as shown below with the *buildWonder* upgrade), as long as all of them are at the same depth of the tree. The engine will automatically resolve the tree and merge all of those subnodes into a single node with multiple parents.

Example upgrade tree:
```yaml
nodes:
  - type: obsidianWeapons
  - type: slaveAuctions
  - type: strengthInUnity
    subnodes:
      - type: ironForging
      - type: ransackers
      - type: karrish
        subnodes:
          - type: thunderousHooves
          - type: onslaught
          - type: buildWonder
      - type: reignOfBeastlords
        subnodes:
          - type: eyeToEyeClawToClaw
          - type: tasteOfManflesh
          - type: buildWonder
  - type: ritesOfRogash
    subnodes:
      - type: harbingersOfChange
      - type: theDreamFuel
      - type: excrescence
        subnodes: 
          - type: shapingCorral
          - type: toTheSkies
          - type: buildWonder
      - type: beastmastery
        subnodes:
          - type: thickShell
          - type: daringTamers
          - type: buildWonder
```

## Maps
The **Maps** folder contains map files that can be used and played in the game. You can have as many levels of subfolders as you'd like in this folder, but only the maps at the root level (inside the **Maps** folder itself) will be presented in the dropdown for Custom Games. Likewise, if you have scenario or campaign-specific maps that you don't want listed there, put them in appropriate subfolders. By default, **Scenarios** and **Campaigns** subfolders are already present.

A map file is a YAML document describing an object with the following properties, all of which are required:
* `id`
* `title`
* `description`
* `size` - the size of the map. For hexagonal maps, it indicated the radius length of the map. E.g, a hexagonal map of size 7 has has 127 cells.
* `kind` - 0 for hexagonal maps, 1 for rectangular. Only hexagonal maps are supported at the moment.
* `maxPlayers` - the maximum number of players that can play on this map in a custom game.
* `defaultTerrain` - the ID of the [terrain](#terrains) that is set as default for cells without the terrain (`t`) property.
* `hexes` - describes the actual layout of the map, as a list of hexes (cells), all of which have the following properties:
  + `c` **(required)** - the coordinates of this cell, in Cube Coordinates. This is a list with three integers, describing the X, Y and Z values.
  + `t` (optional) - the ID of the [terrain](#terrain) at this cell. If omitted, `defaultTerrain` is used.
  + `s` (optional) - the ID of the terrain sprite used. Set this if you want to use an alternative sprite for your terrain at this cell. Note that this usually interferes with [terrain blending](#terrain-blending).
  + `content` (optional) - the ID of a [unit type](#unit) that is present on this cell when map loads.
  + `startingLocation` (optional) - if this cell is a starting location for a player, set this value to the index of that player. E.g, a cell on which the 2nd player starts will have `startingLocation: 2`.
  + `comment` (optional) - a hidden comment associated with this cell, which can be used by scripts at runtime.

Making and editing maps manually is difficult, so it's highly recommended to use the built-in **Map Editor**, available in the **Tools & Extras** menu in the game.

## Movement Types
The **MovementTypes** folder contains config (.yaml) files that describe movement types. A movement type describes how a unit's pathing works - if it can move at all, and if yes, how much APs does it spend based on the types of terrain it moves across. It also determines if a movement can trasverse obstacles or if it can be blocked by other units at the map.

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of movement type objects, which have the following schema:
* `id` **(required)**
* `title` **(required)** - should be a translatable key.
* `stationary` (optional) - if set to `true`, it indicated that this is a stationary object which doesn't turn towards its target when attacking, retaliating or casting. It has no bearing on the unit's actual ability to move.
* `costs` **(required)** - a list of movement costs, which describe how many APs does a unit have to spend to move from its current cell to the target one, depending on the terrain of the target cell. The terrain of the origin cell doesn't matter, since the unit is already on it - only the terrain of the destination cell plays a part. The schema of a movement cost is as following:
  + `terrain` **(required)** - the ID of the [terrain](#terrains) this cost is applied to. You can use a **wildcard**, `*`, to indicate that the cost is applied to all terrains for which there's no other explicitly listed cost.
  + `mp` **(required)** - how many movement points does the unit have to expend to arrive at the cell with the given territory. If you want to mark that a unit can't move over a certain terrain, put a large number here - 1000 is used in the default data set.
  + `ignoreObstacles` (optional) - if `true`, the unit will ignore any obstacles on this terrain unless it is the destination hex. For example, flying units can fly over a cell even if there's something on it, but they can't arrive at a cell if it isn't empty.

## Projectiles
The **Projectiles** folder contains config (.yaml) files that describe projectiles. Projectiles are visual and audible representations of unit attacks: if a unit type has a `projectile` property defined, that projectile will appear whenever the unit attacks or retaliates, for each attack and retaliation.

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of projectile objects, which have the following schema:
* `id` **(required)** - this is the ID specified in the unit type's `projectile` property.
* `image` **(required)** - relative path to a 32x32px PNG that represents the projectile's sprite.
* `speed` **(required)** - speed multiplier that affects how fast does the projectile fly from caster to target.
* `sounds` (optional) - list of sounds that are played based on triggers. All triggers are optional. All sounds are specified as as relative paths to an OGG file. Sounds play only for the currently active player if they can see the action associated with the sound.
  + `cast` - when the projectile leaves the caster.
  + `attack` - when the projectile hits the target.

## Resources
The **Resources** folder contains config (.yaml) files that describe resources. The only customizable resource that players can use is the **unique resource** for their faction - all players have access to **gold** by default. Generation and spending of a resource is a separate matter that is done via [effects](#effects).

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of resource objects, which have the following schema (all properties required):
* `id`
* `title` - should be a translatable key.
* `description` - should be a translatable key.
* `image` - relative path to a 32x32px PNG.

## Rulesets
The **Rulesets** folder contains config (.yaml) files that describe rulesets, as well as script (.ns) files that describe their initial setup and win conditions.

A ruleset if, as it name says, a set of rules that governs the game. This allows for a wide array of customization of how game mechanics work, which can drastically affect individual games. Rulesets provide a fairly simple way of adding completely new game modes or and replayability to existing ones via small twists that can have a big impact.

Each ruleset should be in its own config file, named according to its ID. Win condition scripts can be spread out over as many script files as needed, since all scripts are compiled at once anyhow.

To reduce clutter, rulesets can copy each other with the `inherits` property and then only override certain other properties.

### Rules
Here's the full list of ruleset properties:
* `id` **(required)** - this is usually capitalized (PascalCase).
* `title` **(required)** - should be a translatable key.
* `description` **(required)** - should be a translatable key.
* `inherits` (optional) - the ID of the other ruleset that this ruleset inherits/copies.
* `playable` (optional) - if `true`, this ruleset will appear as selectable for custom games. Otherwise, it is meant for scenarios/campaigns.
* `maxTeams` (optional) - the maximum number of teams that players can be distributed into.
* `teamsShareVision` (optional) - if `true`, players can see everything that their teammates can as well.
* `fogOfWar` (optional) - if `true`, the map has fog of war, which prevents seeing cells which are out of your sight.
* `timeoutLength` (optional) - indicates how much time does a player have before the server automatically ends their turn, in *seconds*. If `null` or zero, indicates that players have infinite turns. The actual behavior tightly correlates with the `timerKind` property.
* `timerKind` (optional) - indicates how do timeouts work. Possible values:
  + `perTurn` (default) - `timeoutLength` is reset for each turn and the timeout ends the turn. E.g, if `timeoutLength` is 120, each player has 2 minutes to play each of their turns, after which their turn is automatically ended.
  + `chess` - `timeoutLength` spans the entire game, counting down for each player on their turn. Timeout kicks the user from the game (meaning they lose). E.g, if `timeoutLength` is 600, each player has only 10 minutes in total for everything they wish to play, regardless of how many turns they take. This is akin to how timers work during chess games, hence the game.
* `maxHandSize` (optional) - the maximum number of playables that a player can have in their hand at any given time. Effectively limits the building queue for a player and limits how fast they can grow and expand.
* `deterministicCombat` (optional) - if `true`, attack spread (`as`) isn't applied when units attack/retaliate, meaning that the combat is entirely deterministic and can be computed solely based on attack/defense/attack count/retaliation count values of units involved (not counting combat [effects](#effects), which can be coded to have random outcomes).
* `autoCameraReposition` (optional) - if `true`, the camera will jump to the current player when their turn starts. This is usually set only for scenarios and campaigns.
* `randomizedFirstPlayer` (optional) - if `true`, the first player (the one which starts on the first turn) is chosen randomly. Othwerise, players always play in the order which is set by the game rules, i.e the first player goes first, the second one second, etc.
* `winCondition` (optional) - the class name of the [scenario](#scenarios) the game runs in. Each game, even a custom one, runs in a scenario, which responds to game events in order to set the game up, check if anyone won, etc.
* `factionData` (optional) - determines the setup for each individual faction. It is a map where the key is the faction ID, mapping to the following schema:
  + `startingData` **(required)** - the initial conditions for each faction when the game starts. Has the following required properties:
    - `resources` - the resource distribution this factions starts with, as an object with `g` and `u` properties.
    - `cards` - the [playables](#playables) that start in the player's hand. It is a map, where the key is the playable's ID and the value is the quantity the player starts with.
    - `units` - a list of [unit](#unit) IDs that will be spawned at the player's starting location. Add an ID multiple times to indicate multiple units of the same type.
  + `playerEffects`  (optional) - list of class names of [effects](#effects) that are attached to the player at the start of the game.
  + `upgradeTree` **(required)** - the [faction upgrade tree](#upgrade-tree) for this faction.
* `bans` (optional) - list of banned skills and stances that can't be used in a game. It has two optional properties:
  + `skills` - a list of IDs of [skills](#skills) that can't be used in the game.
  + `stances` - a list of IDs of [stances](#stances) that can't be used in the game.
* `otokRuleset` (optional) - the set of rules for Otok minigames when played within a Nusantara game with this ruleset. It has the following required properties:
  + `rounds` - total number of rounds for a game of Otok. This should be an odd integer.
  + `initialResources` - the amount of gold each player received at the start of the game.
  + `resourcesPerDraft` - the additional gold each player receives before each draft (including the first one).
  + `draftDuration` - duration of the draft phase, in seconds.
  + `combatTurnsPerRound` - the number of combat turns per each round, after which the winner is determined.
  + `combatTurnDuration` - duration of each combat turn, in seconds.

### Win conditions
Ruleset win conditions are [scenarios](#scenarios) that govern how does the game go by responding to different in-game events, just like regular scenarios. They're usually placed in script files in the **Rulesets** folder.

The basic `DestroyAll` scenario highlights important event methods that every win condition should have:
* `onStart` to set the game up, by adding creeps and sites, adding the objective, etc.
* `onStateChange` to figure out if any player lost all of their units and therefore kick them out of the game.
* `checkVictory` to see if all remaining players are from the same team and declare the winner.

Just as rulesets can inherit other rulesets, it's common for win conditions to extend each other to reuse script code.

## Scenarios

## Skills
The **Skills** folder contains config (.yaml) files that describe skills. [Playables](#playables) can have skills, which are shown in the UI and each wraps a single [effect](#effects). Skills can also be banned by [rulesets](#rulesets).

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of skill objects, which have the following schema:
* `id` **(required)**
* `title` **(required)** - should be a translatable key.
* `image` **(required)** - relative path to a 64x64px PNG.
* `effect` **(required)** - the class name of an [effect](#effects) that this skill wraps.
* `params` (optional) - this is the list of parameters that are passed to the effect `init` method when the skill effect is instantiated. The presence and format of this property depends on the effect and its initializer. The purpose is to allow reuse of effect classes (e.g, reuse of **TrainHero** or **GenerateFavorMonument** effects).
  
## Stances
The **Stances** folder contains config (.yaml) and script (.ns) files that describe stances.

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of tag objects, which have the following schema (all properties required):
* `id` - must correspond to a `stance` class name from scripts, hence it's capitalized (PascalCase).
* `title` - should be a translatable key.
* `image` - relative path to a 64x64px PNG.

Units assume stances during gameplay and it alters their stats, which is described by the correspoding `stance` class in the script files, which can implement the following fields/methods:
* 

## Tags
The **Tags** folder contains config (.yaml) files that describe tags. A [unit](#unit) can have multiple tags, and tags can be reused between multiple unit types.

You can have as many config files in this folder (or its subfolders) as you'd like, as all are read and merged when data is loaded. Each file needs to be a list of tag objects, which have the following schema (all properties required):
* `id`
* `title` - should be a translatable key.

## Terrains

### Terrain Blending

## Translations