# Nusantara Data

Game data for Nusantara game. It contains all the game's art, sounds, config and scripts. Feel free to fork and alter for modding purposes.

Balance proposals are welcome via PRs.

[Play Nusantara](https://playnusantara.com)

## Nusantara modding guide
This document shows how to modify existing and implement new features using the Nusantara engine. Almost all content in the base game is implementable using these methods.

In general, modding boils down to modifying **config files** and **scripts**. [VSCode](https://code.visualstudio.com/) works well enough for both purposes.
1. Config files are written in [YAML](https://yaml.org/) and their extension is **.yaml**. They contain "static" data that describes, e.g, all the units in a faction or the contents of a map.
  * [VSCode extension for YAML files](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)
1. Script files are written in [NusantaraScript language](https://github.com/globulus/nusantara-script) and their extension is **.ns**. They describe dynamic behaviours at game's runtime, such as effects, scenarios, AI behavior, etc.
  * [VSCode extension for NS files](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)

### The *data* folder
At startup, Nusantara loads its content from the **data** folder, which has to be located in its persistent storage location. E.g, on Windows 10, this usually resides at `C:\Users\User\AppData\LocalLow\Globulus\Nusantara\data`. Modding the game involves making changes to its subfolders and files, all of which is discussed in the rest of this document.

The structure of the **data** folder and its subfolders, as well as names of the files are sometimes important, so make sure not to leave anything out. The details are discussed in the sections below.

Loading might fail due to invalid file structure, compilation errors in scripts or internal validation errors, all of which will be apparent during the loading process. If the game data is corrupt for whatever reason, the game won't be playable, so always be sure to **back up your existing data before modding**.

### Meta.yaml
The single standalone file in the **data** folder is **Meta.yaml**. It contains metadata that describe the game data and is used by the game to automatically check for updates.

This file describes a single object whose structure is:
* `version` - the version code of the data file. If modding, bump this up to a high value so that it doesn't get overwritten by game's automatic update system.

### Bots
The **Bots** folder contains config and script files that describe the game's bots, i.e AI entities that can play the game and simulate human players.

#### Config files
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

#### Script files
There can be as many script (.ns) files in the folder. Ideally, they should contain the code that describes `ai` classes corresponding to those listed in config files.

There's no hard and fast rule how an AI should be implemented. The game expects two methods to be implemented:
* `awake` is called when the AI assumes control of a player at the start of their turn. E.g, if your AI has a hard limit of only 10 actions per turn, you can reset the counter in this method.
* `update` is called by the game server repeatedly until it returns `false` - if it returns `true`, it means that AI still has work to do in this turn, so the `update` method will be invoked again. Once this method returns `false`, the game server will end the AI player's turn automatically.

There is the optional third method, `onGameStart(player)`, which is called just once, when the game first starts. Its argument is the player object which the AI will drive. You can use this to perform any additional modifications to the player, such as cheating by giving it extra resources.

The default bots use the "value AI" approach, where a series of possible decisions are weighted, with the one with the highest core executed for the given run. Again, the framework is flexible enough to allow for implementation of different kinds of AIs for different purposes (build tree, passive, etc.).

### Campaigns

### Effects
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