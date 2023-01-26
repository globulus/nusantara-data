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
A unit is a [playable](#playables) that represents an object on the game board. This includes creatures, buildings, interactable map objects, etc. Here are the list of available properties for a unit type:
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
* `skills` (optional) - the list of IDs of [skills](#skill) that this unit has.
* `canInteract` (optional) - if this is `true`, the unit can interact with Neutral units that have an `interactionEffect` defined.
* `interactionEffect` (optional) - the ID of an [effect](#effects) that is invoked when another unit tries interacting with this one.
* `extras` (optional) - 