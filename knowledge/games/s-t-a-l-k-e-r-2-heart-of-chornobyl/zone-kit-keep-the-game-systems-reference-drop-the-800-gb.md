---
kind: game
title: 'Zone Kit: keep the game-systems reference, drop the 800 GB'
game: 'S.T.A.L.K.E.R. 2: Heart of Chornobyl'
games_also: []
game_version: "Steam 1643320 build 25382007 (public since 2026-09-18, the Patch 2.0.6 release; the note recorded project version 2.0.5), Unreal Engine 5.5.4; Zone Kit from the Epic Games Store with the same engine version"
platform: windows
engine: unreal
route: data
tools:
- repak 0.2.3
- retoc
- robocopy
- Python 3
- UE4SS (object dump only)
anti_cheat: none found; single-player game, nothing was injected and only files from the user's own install and kit were read
status: working
agents:
- Claude Code (Fable 5.1)
humans: []
date: '2026-10-12'
links:
- https://zonekit-support.stalker2.com/en-US/kb/article/145/config-patches
- https://zonekit-support.stalker2.com/en-US/kb/article/172/how-to-patch-contents-of-a-cfg-file
- https://zonekit-support.stalker2.com/en-US/kb/article/162/mod-game-features
- https://zonekit-support.stalker2.com/en-US/kb/article/148/creating-mods-with-mini-sdk
- https://zonekit-support.stalker2.com/en-US/kb/article/164
- https://cdn.stalker2.com/guides/Quest_Tool_complete_guide.pdf
tags:
- unreal
- zone-kit
- sdk
- gamedata
- cfg
- a-life
- emission
- quests
- artifacts
- detector
- weapons
- weather
- saves
- repak
- iostore
---
# Zone Kit: keep the game-systems reference, drop the 800 GB

> The official modding SDK for S.T.A.L.K.E.R. 2 (the "Zone Kit", Epic Games Store, about 817 GB) is
> overwhelmingly cooked editor content, but the part that explains how the game works is a 352 MB tree of
> plain-text GameData `.cfg` files plus a handful of cooked Blueprint packages. This note says how to copy just
> that part out for your own use (the kit's licence forbids sharing it) and reproduce the copy, what the systems
> look like (A-Life, emissions, saves, quests, artifacts and detectors, NPCs, weapons, weather, mod mounting,
> console commands), and which behaviour is native code that no file in the kit describes. Everything was read
> from files; the game-side claims marked "observed" were seen in the running retail game, the rest is read-only
> study.

## Setup

- Game: Steam app 1643320, build 25382007, Unreal Engine 5.5.4. That build has been the public branch since
  2026-09-18, the day GSC released Patch 2.0.6; the note recorded "project version 2.0.5". Retail content is
  split into many `pakchunk*-Windows` IoStore containers (`.utoc/.ucas`, plus a small `.pak` each).
- Kit: Zone Kit from the Epic Games Store, about 817 GB on disk as measured (Epic and GSC's guide still say
  about 700 GB). Its batch files carry the Phase 2 names (Phase 2, with quests and audio, shipped 2026-09-01).
  The engine version file says 5.5.4 on the UE5 branch. It needs a lot of space; ours sat on a slow HDD, which
  dominates every operation below.
- Tools used: `repak` 0.2.3 (list / get / unpack for classic `.pak`), `retoc` (IoStore `.utoc` listing and
  unpacking; only tried to the point of confirming the retail files are IoStore), `robocopy`, Python 3 for the
  manifest and extraction scripts, and a UE4SS object dump of the retail game for class and function names
  (the kit ships no C++ headers).
- OS: Windows 11. Nothing was installed into the game for this work.

## Route and why

Route is `data`: the useful knowledge is in configuration, not in code.

- Option A, keep the whole kit: 817 GB, no extra insight for systems work, and most of it (200 GB of world
  map, 140 GB of derived-data cache, audio) is binary and not readable without the editor.
- Option B, keep only text: covers everything that is configuration-driven, which turned out to be almost
  all gameplay tuning. The kit has the cfg files as plain text; retail ships them binarised.
- Option C, text plus a few cooked Blueprint packages: adds the visual/component side of the main mechanics
  (player character, NPC and mutant characters, weather controller, artifact, anomaly, stash container,
  A-Life placeholders). Chosen. 15 packages, 2.5 MB in total.
- Rejected: the editor and Play-In-Editor as a research tool. It would have to mount the 389 GB content pak,
  the 139 GB derived data cache and a 1.3 GB asset registry from the HDD, which is minutes of loading for a
  question that the cfg files answer in seconds. Not tried; this is an estimate from the sizes.
- Rejected: retail IoStore extraction for Blueprint logic. Cooked graphs are compiled; not attempted here.

## What is inside the kit

Sizes measured by listing the kit; the sum of the large files is about 728 GB, and the rest (engine, audio
sources) was not scanned because of the HDD.

| Part | Size | Use for understanding systems |
|---|---|---|
| `Stalker2/Content/GameLite/GameData` (plain `.cfg`) | 151,371 files, 352 MB | The main prize: every prototype table of the game |
| `Paks/FullEditor-WindowsModEditor.pak` | 389 GB | All editor content as a classic pak, 143,047 entries; a few Blueprints worth extracting |
| `Paks/WorldMap-WindowsModEditor.pak` | 189 GB | Open-world terrain and static actors, 719,343 entries; only the name list is useful |
| `Paks/Wwise-...pak` | 9.7 GB | Sound banks, not needed |
| `Paks/Stalker2-WindowsModEditor.pak` | 0.34 GB | Small base pak, name list only |
| `DerivedDataCache/Compressed.ddp` | 139 GB | Cache, regenerable |
| `EditorClientAssetRegistry.bin` | 1.3 GB | Class and dependency graph; regenerable by reinstalling |
| Editor executables (`Stalker2ModEditor`) | about 450 MB each | The only way to author Blueprint or new-asset mods |
| `Config` ini files, `Plugins` descriptors | under 1.2 MB | Project settings, mod plugin template and cook profiles |
| `Source` | one `Stalker2.Target.cs` | No C++ headers at all |

Also present: the `ModKit` plugin (editor and runtime modules, a `PlainMod` plugin template that can contain
content), two cook profiles (one for new content, one for override content), and two batch files,
`CreatePlainMod.bat` and `PakPlainMod.bat`, that wrap the engine's automation tool for config-only mods. No
sample mod project and no documentation files ship in the folder: the documentation is a set of web articles
(GSC's Zone Kit knowledge base and guide PDFs, in `links`). GSC's Quest Tool guide does point to template quest
Blueprints (`GDEQ_Template_*`) inside the editor content.

The kit's EULA (v1, June 2025) licenses one copy for personal use, to make mods for the game. It forbids
redistributing the kit's content (§2.1) and reverse engineering or decompiling the licensed materials, and it
asks you to destroy all copies if the licence ends (§12). So the trimmed copy, the pak name lists and any
extracted packages below are for your own use only: never publish or share them.

## How to keep only the useful part

Do this before deleting the kit, because three things can only be done while the paks exist: listing them,
extracting Blueprints, and (optionally) a content hash.

1. Copy the text-like parts with `robocopy`: `GameLite` (the GameData tree and its checksum file), the
   project and engine `Config` folders, the `.uproject`, `Build.version`, plugin descriptors (copy by
   extension: `uplugin`, `uplugintemplate`, `ini`, `py`, `h`, `cfg`, `json`, `txt`, `md`; no `dll`/`uasset`),
   the two batch files, and `Source`. Result: about 151.6k files, 352 MB, 104 plugin-side files. On the HDD
   this was the slow step; use `/MT:4` and expect it to be bound by file count, not bytes.
2. Save the name lists of the four kit paks with `repak list <pak>` (entry names only; 13.6 MB, gzip them).
   The list is how you later find "which asset was called what" without the 800 GB.
3. Extract a handful of cooked packages by exact path with `repak get <pak> <path>` (one entry streamed to
   stdout, so redirect to a file). The pak is a classic, unencrypted, uncompressed V11 pak with mount point
   `../../../`, so no key and no oodle are needed. Each package turned out to be a single `.uasset` with no
   `.uexp`/`.ubulk`. For a pattern instead of one path: `repak unpack -i <dir-or-file> -o <out> <pak>`
   (`-i` takes files or directories, repeatable).
4. Write a manifest: file counts and bytes per part, plus a content hash over the sorted list of
   `<relative path> <md5>` lines for the GameData tree, so a later copy can be compared.
5. Only then delete the kit. Result of this procedure: 369 MB total, of which 352 MB is GameData.

The 15 Blueprint packages kept (all from the FullEditor pak, grouped by mechanic): the player character, the
game mode, the weather controller, a Dead Valley emission restriction volume and the emission damage curve,
the artifact actor and the artifact-spawner editor placeholder, one electro anomaly, the player detector
animation layer, a backpack stash container, the human NPC character, the mutant AI controller/character,
two A-Life editor placeholders (lair and group zone), and an encounter NPC spawner. They are listed by mechanic
in a manifest next to the files with sizes and hashes.

## How the game works (what we had to learn)

Tags: **[kit]** read in a kit file; **[observed]** seen in the retail game; **[inference]** reasoned only.

### Config files and the patch language
- GameData is a tree of `<Name>Prototypes.cfg` files (one file, many nodes) or `<Name>Prototypes/` folders
  (one file per group), 167 top-level entries. Big folders: quest node graphs (about 1,300 files), quest
  definitions (about 1,300), journal quests (about 190), dialogs (about 8,200), and a spawn-placeholder tree
  with roughly 83,000 tiny files. Text localisation sits in per-language folders. [kit]
- Syntax: a node is `Name : struct.begin {refkey=Parent}` ... `struct.end`. A node can inherit a sibling by
  `refkey`, or a node of another file by `refurl` plus `refkey`; a mod can refer to the base game's files
  with an `@BaseGame/` URL prefix. `{bskipref}` means "do not merge this list with the parent's", `{bpatch}`
  means "add or replace child values instead of replacing the whole node", and `removenode` (only with
  `{bpatch}`) deletes a node. Lists use anonymous `[*]` items. Files carry a UTF-8 BOM. [kit, plus the
  official config-patch articles for `{bpatch}` and `removenode`; `{bskipref}` is from the kit files and
  community tools, not the official articles]
- A node in a patch replaces the whole base node unless `{bpatch}` is used. Patches live as `*.cfg_patch_*` /
  `.cfg.bin_patch_*` next to a single file, or as any `.cfg` inside a prototype folder; the loader scans
  everything under the GameData path. [kit docs]
- Retail ships the same tables binarised as `.cfg.bin`. The binary form accepts the same patches. [observed]

### A-Life: director, population, policy
Three layers, all configuration-driven. [kit]
1. **Director** (events around the player). One preset holds spawn delay and timeout ranges, how many squads
   an expansion sends, whether NPCs despawn, and the default scenario groups for normal, emission and empty
   situations. A table caps how many agents of each archetype (human, generic mutant, bloodsucker, blind dog,
   flesh, controller, snork, boar, chimera, and so on) may be spawned per *player rank* (Newbie, Experienced,
   Veteran, Master). There are lists of NPC archetypes the director may not use, agent types it may not use,
   and fallback types. About 70 named scenarios (humans vs humans, humans vs mutants, wounded or dead humans,
   attacks on an enemy lair, mutant packs, single predators, dead-body setups per faction...) each carry the
   required player rank, squads (archetype, hostility to player, relation group, alive/wounded/dead
   multipliers) and who they target. Scenario groups (Emission, Global, Local, Empty) weight those scenarios
   and set their own delays.
2. **Population manager** (the persistent world): lair expansion timing and radius, a simulation start
   offset in game hours, and about 30 factions (humans and every mutant kind). Each faction has a battle chance
   and three "goal" bands (Aggressive, Normal, Defensive) keyed on how many lairs it holds. [inference: that
   is the faction war: few lairs means aggressive, many means defensive.]
3. **Policy**: thresholds for extinction/refill, corpse decomposition and limits, how far a lair counts as far,
   how long "seen long ago by the player" is, and refill cooldowns and distances after partial or full wipes.
   What "extinction" numbers mean is undocumented. [inference: population caps]

Online bubble and caps are global variables, not per-scenario: the grid update delay and vision radius that
define the bubble around the player, minimum spawn and despawn distances, a maximum agent count, a corpse
online time and a hard cap on corpses. [kit]

Lairs and placeholders. A lair is a level placeholder of spawn type `LairSpawner` (hundreds in the world)
pointing to a lair prototype, with core and territory volumes, capture/attack/defend flags, an initial
inhabitant faction, rank limits and, per possible inhabitant faction, a table of spawn quantities and respawn
timers by player rank and by NPC archetype with weights. Zones that mute the director are `ALifeGroup`
placeholders carrying a scenario group id (an empty one disables scenarios) and a priority (local, global,
all). Quests can toggle a lair (a node type used hundreds of times) and the console can too. [kit]

Observed: console-spawned NPCs have no orders: no lair, squad or quest node gives them goals, so they stand
still unless you call the AI console tools. The "all AI features off" console switches do not stop the spawn
and despawn bubble around the player. In the area tested, the nearest always-active hostile lair was more
than 200 m away. [observed]

### Emissions
- `EmissionPrototypes` has a default node plus overrides for a few story emissions. A node holds the sky
  origin, reaction-time ranges for quests and A-Life, a kill delay range, a list of stages (before the storm,
  activate quest, shock wave, active, after the storm) with start times and durations, a list of timed AI
  events (start, danger, active, end), assets (sounds, damage curve, VFX, generator sequences), damage blend
  time, and applied effects (camera shake, night-vision post process). [kit] The damage curve is a cooked
  asset, one of the kept packages.
- The kit has **no explicit schedule** for the next emission. The config hooks are quest node types that pause
  or resume the scheduler and start an emission by prototype id (optionally forced), a neutrality timer and a
  start location in the global variables, a difficulty multiplier on the emission weather weight, and the
  weather selection weights below. GSC's Quest Tool guide says regular emissions start through the gradual
  weather blend, and that Force Emission bypasses the weather blend rules. So the cadence's inputs are in
  GameData, while the roll and the manager are native: a class `EmissionManager` and enums for stage, state
  and AI event show up in a reflection dump. [kit + reflection dump + Quest Tool guide]
- Emission is also a weather type, and `CalmBeforeEmission` precedes it. The weather selection tables give
  Emission a blend weight that grows with each weather change and a cooldown counted in weather changes;
  chain nodes route every other weather through `CalmBeforeEmission` into Emission and out again. [kit]
  [inference: the weight accumulates until an emission rolls.]
- Shelters are level placeholders (hundreds). The emission quest tracks the player's shelter and shows two
  journal stages (warning, stay safe). [kit]
- Observed: calling the console emission start makes `IsEmissionActive` true about 36 s later and lethal
  damage begins about 2 minutes after the call; the stop command ends it at once. Switching to weather type 7
  alone also produces a real emission. The configured stage timeline (140 s) and the measured duration
  (about 165 s) disagree and the difference is unexplained (the weather blend in and out is a candidate). In
  the prologue save, no emission could be started by any of these. [observed] [inference: the story blocks
  it; quests can pause the emission scheduler.]

### Weather and time
- `WeatherSelectionPrototypes` lists the ten weather types with blend weights, weight increase per change,
  duration ranges, repeat and cooldown limits; regional nodes refine this and world volumes (hundreds, type
  `WeatherSelection`) apply them by priority. `WeatherChainPrototypes` holds `<A>To<B>` transitions with
  intermediate steps and speed multipliers. Visual parameter sets are separate small folders. [kit]
- Enum, observed: 0 Clearly, 1 Cloudy, 2 Fogy, 3 Stormy, 4 LightRainy, 5 Rainy, 6 Thundery, 7 Emission,
  8 CalmBeforeEmission, 9 Underground. The controller's `CurrentWeatherData.TargetWeather` reads one higher
  than the value passed to the switch command (off-by-one in the readback, not in the command).
- Two clocks, observed: a game clock (quests, A-Life) and a weather/sun clock. They run at the same rate
  (real-to-game coefficient in the global variables) but have separate setters and do not move each other.
  GSC's Quest Tool guide adds that game time passes more slowly during the day than at night, so that
  coefficient isn't the whole story.
  The weather switch command does the game's own transition; a separate force command is instant.
- The controller is the Blueprint `WeatherController_BP` over a native `WeatherController`. [kit]

### Saves
- Timed autosave interval, retry interval, quick-save overwrite window, per-type save limits (manual, auto,
  quick, quest, exit, hub exit, main-quest start/finish, manual hub, point of no return), a threat radius
  that blocks saving near enemies, and save-type categories are plain variables. [kit]
- `ESaveType` has ten values: Manual, Auto, Quick, Quest, HubExit, ExitGame, StartMainQuest, FinishMainQuest,
  ManualHub, PointOfNoReturn. [reflection dump]
- **Per difficulty**, `DifficultyPrototypes` has an `AllowedSaveTypes` list (declared with `{bskipref}`),
  `TotalSaveLimits` and `AutosaveAfterQuests` (the main quests that trigger an autosave). The easier
  difficulties allow Manual, Auto, Quick, Quest and PointOfNoReturn; the top difficulty ("Stalker", also
  called Master) allows only the story-boundary and exit types plus manual-hub, with tiny total limits.
  Quest nodes `RestrictSave` block chosen types for a stretch of a quest, and `SaveGame` nodes write saves
  from quests (hundreds of them). [kit]
- Observed: a patch that makes `AllowedSaveTypes` empty blocks manual, quick and auto saves; F5 shows a
  "cannot be saved now" message and no save file appears. A workshop mod reopens manual saves on Master with
  one `{bpatch}` file replacing that list. The game also writes two Auto saves per emission (at the start and
  a little over 2 minutes later) that no config asks for; they are Auto type and obey `AllowedSaveTypes`.
  The campaign summary save is rewritten on every boot regardless. Saves are refused during cutscenes.

### Quests
- Definitions (`QuestPrototypes`: id, main-quest flag, start node, name keys), graphs (`QuestNodePrototypes`),
  the PDA journal (`JournalQuestPrototypes`), global variables (a few hundred bool/int ids), dialogs and
  dialog chains. [kit]
- Every graph node is a struct with an id, its quest, a node type, a launcher list (which node and output pin
  starts it), a start delay and type-specific fields. By count the most common types are Technical,
  Condition, SetDialog, SetAIBehavior, Spawn, SetJournal, BridgeEvent, SendSignal, Trigger, OnTickEvent,
  ChangeRelationships, If, End, Container, TeleportCharacter, Despawn, ItemAdd, SaveGame, SetGlobalVariable
  and hit/death/create events. `ConsoleCommand` nodes run console lines directly. [kit]
- **Spawn nodes never carry coordinates**; their target is the name of a placeholder file in the spawn tree
  (a 32-hex GUID file name). A fixed all-A GUID means the player. [kit]
- A new game starts the quest `rootgraph`, which holds a `Container` node per quest wired by launchers; the
  main story is a chain of those containers from the first prologue quest to the last act. Nothing in the
  graph launches the first one. Other starters: generic and encounter quests, lair quests, region quests, a
  spawn-on-start folder, and mod patches that add an `OnTickEvent` to `rootgraph`. [kit]
- The prologue quest alone is a 10,000-line graph with sub-quests, two cutscenes and an intro film; one of its
  entry nodes skips the film. Only `SetJournal` changes the journal and only `SetGlobalVariable` or a
  `Condition` of the right kind reads or writes variables. [kit]
- Observed: the console exposes start/end/skip/exclude/reset by quest id and node id; there is a setter for
  bool globals only (no int setter found); the active/finished quest lists from the mediator return arrays
  with no readable fields (counts only). [observed]

### Artifacts, anomalies, detectors
- **Anomalies** are level placeholders (many thousands in the world across gravity/electro/soap-bubble/
  pillow/psy/lightning-ball types), not A-Life agents. A prototype gives radius, blueprint, type and element,
  cooldown, duration, activation delay and speed threshold, bones traced, and effect ids for passive /
  pre-interaction / interaction / post-interaction. Anomaly spawner placeholders carry presets with a start
  preset id. Radiation fields use preset strengths named Light, Medium, Strong, Deadly, plus RadBlock and
  Topaz. Damage scales with the difficulty `Anomaly_Damage`. [kit]
- **Artifact items** (a few hundred ids): element and artifact type, rarity (Common/Uncommon/Rare/Epic), a
  "jump" behaviour (distance, amount, delays), a lifetime, a player-distance and `DetectorRequired` flag, and
  the artifact blueprint. **Artifact spawners** (about 90 prototypes) have, per player rank, a count, radius,
  cooldown range, base chance, a bonus chance added after each failed roll, rarity weights, and an optional
  fixed list. Global variables add attempt counts, placement distances and "pity" tables that raise luck of
  chosen rarities after failures. The universal spawner is what most of the world uses. [kit]
- The roll is **native and tied to the local player**: an `ArtifactSpawnerActivationComponent` sits on the
  player pawn, so spawners only act near that player. No configuration links artifacts to emissions. A
  console command adjusts the spawn luck. Anomalies can also create fake artifacts. [kit + reflection dump]
- **Detectors** are item prototypes (a template plus four concrete detectors, one with a screen/sonar and an
  anomaly icon list). Keys include display radius, minimum detect radius, work radius, sound curves, a danger-
  level sound parameter that every detector has, and an exclusion list of special artifacts. The *passive*
  detector prototypes (player, searchpoint, collar) are the anomaly danger signal and have nothing to do with
  artifacts. The kept detector Blueprint is only an animation layer. [kit]
- **Native detector logic.** The classes `PassiveDetectorComponent`, `AnomalyDetectorComponent` and
  `SearchpointDetectorComponent` expose no target list and no function you can call from script; there is
  no detector logic Blueprint to find. [reflection dump]
- Observed by ear (single session, not instrumented): the stock pickup is silent, but an artifact actor that
  has been hidden, made non-colliding and non-interactable keeps making the detector click until it is moved
  roughly 50 m or more sideways. So the detector senses *existing artifact actors*, not a spawner record: a
  place whose spawner is "used up" can keep reacting as long as an actor remains there. Actors pushed far
  below the map vanish from `FindAllOf` yet the detector still reacted, so moving sideways is what works.
  Picking an artifact up makes the actor jump to the origin first.

### NPCs and factions
- Two-level prototypes. An **object prototype** (`ObjPrototypes` and 30 sub-files: general NPCs, each mutant
  kind, bosses) is the body: faction, mass, trade/item-generator/mesh-generator ids, offline combat weight,
  per-bone damage coefficients (head high, limbs low), armour handling, abilities, hit points, and the
  character blueprint (native parent `Agent`; `BP_AI_Human` for people). A template `NPCBase` is what most
  humans inherit. An **NPC prototype** (`NPCPrototypes`, about 26,700 lines) is the person: dialogue chains,
  type, map marker, money, upgrades, skills, name key, voices. Director archetypes look like
  `GeneralNPC_<Faction>_<Role>`. [kit]
- `FactionPrototypes.cfg` is an **empty file** (0 bytes): factions are plain strings used by the other
  files. [kit]
- `RelationPrototypes` has relation level ranges (Enemy, Disaffection, Neutral, Friend, self), a minimum
  level to trade, per-event reaction matrices (damage, kill, heal...), reputation rollback radius and
  cooldown, a faction-to-player table, and faction aliases (story factions mapped onto base ones). Quest
  nodes `ChangeRelationships` and `ChangeFaction` change them at runtime. [kit]
- Placement spawn types: `Obj`, `Squad`, `DeadBody`, `Patrol`, `MoveToPath`, `DailyScheduleZone`. [kit]
- AI sensors (vision, hearing, flair, movement, threat) live in `AIPrototypes`; `AIGlobals.cfg` holds the
  caps. The AI feature flags (tick, decision, threat, combat, sensors, vision, hearing, hit, flair,
  movement detection, A-Life director, population manager, see-player, and an `All` mask) are a bit-flag
  enum with console activate/deactivate commands. [kit + reflection dump]
- Observed: the console spawn command with a prototype id (including a custom one added by a patch that
  inherits from the base templates) produces a real human NPC character at the requested point; spawns far
  outside the player's bubble silently produce nothing. In these spawns the NPC had no AI controller and zero
  walk speed, so it could only be moved externally. [observed] That doesn't square with the A-Life
  observation above (spawned NPCs stand still unless the AI console tools are called), so the result may
  depend on the spawn path. GSC's Quest Tool guide says resetting an NPC's goals hands it back to A-Life.
  Not resolved.

### Weapons and damage
- Player guns have `_Player` suffixed entries in the player weapon settings file, NPC guns a parallel file,
  and attribute folders plus general setup, ammo, projectile, damage-source, explosion and melee tables sit
  alongside. Per weapon: durability and armour damage per shot, armour piercing, cover piercing, base damage,
  bleeding amount and chance, effective distance min/max, drop-off, dispersion, comfort, loudness, UI bars.
  Weapons inherit from templates by `refurl`. [kit]
- Difficulty multipliers (`Weapon_BaseDamage` for the player, `NPC_Weapon_BaseDamage`, mutant, anomaly,
  radiation and regeneration factors) multiply on top. [kit]
- Observed: the configured values for four player weapons matched the numbers parsed from the retail
  `.cfg.bin`. In the damage structure, armour piercing acts as a doubling-per-point multiplier against the
  target's armour; a plain damage call applies the bone coefficient but ignores armour, and the structured
  call applies armour but ignores the bone. Native hit hooks on the Blueprint side registered but did not fire
  for native hits. Not measured: the other weapons, ammo and upgrade effects, walls.

### Inventory, trade, containers
Item prototypes are split into 16 files (ammo, armour, artifact, attachments, binoculars, blueprints,
consumables, detectors, key items, grenades, money, mutant loot, night vision, quest items, weapons). Stashes
(about 1,000 ids), item containers (about 45), corpses, trade tables (about 660), item generators with a
"smart loot" system per rank, and backpacks are separate tables. Trade tables hold open hours, money, refresh
condition, buy/sell modifiers and per-rank discounts. The container Blueprint's native parent is
`InteractableItemContainer`. No function that returns a container's item list turned up in the reflection
dump; the only removal path found is the `ItemRemove` quest node. [kit + reflection dump]

### Difficulty
`DifficultyPrototypes` has nodes for the base, Easy, Medium, Hard, Stalker (Master), Custom, console variants
and Default, each with economy, NPC combat, mutant combat and environment blocks, save permissions, and
HUD toggles (Master removes crosshair, compass, stash and body markers and adds hunger, radiation and
sleep mechanics). The multipliers rise or fall monotonically from Easy to Master (damage taken, NPC hit
points, radiation, anomaly damage, bleeding, emission-weather weight, rewards). [kit]

### Console command families
Console functions live on `CustomConsoleManager*` classes. The suffix letters after the class name are
obfuscated and may change after a game update [inference], so find them by function name in a fresh object
dump. Families seen: spawning (objects by prototype id, items by coordinates or near the player, containers,
A-Life director scenarios, creating items in an inventory, despawning NPCs in a radius); A-Life and AI
(disable/enable/kill all, activate/deactivate AI features, set relation between two ids or in a radius,
lair activate, follow/stop-follow, reset AI, override combat tactics); weather and time (switch/force
weather, set game time, set weather time, set time speed); emission (start, start by id, stop, get state,
pause and resume the scheduled one); saves (trigger save by type, forced autosave, load last save); quests
(start/end/skip/reset quest or node by id, developer-kit quest and stage start/finish, set a global bool);
god-mode and test helpers (set god mode, set an entity unkillable, add money, set artifact luck, switch an
anomaly spawner preset, regenerate stash items). Quest `ConsoleCommand` nodes and stock quests call the same
lines. The console surface is also reachable from a UE4SS Lua mod and from Blueprint mods. [reflection dump]

### Mod mount points and the override mechanism
- Mods are Unreal **Game Feature plugins**: the plugin descriptor has category "Game Features", explicit
  loading and a `Mod` flag; the project ini registers a `Mod` metadata key and a project-policy class
  `S2GameFeaturesProjectPolicies`. [kit]
- A packaged workshop mod unpacks to `Stalker2/Mods/<Name>/Content/Paks/Windows/<Name>Stalker2-Windows`
  as an IoStore trio (`.pak`, `.utoc`, `.ucas`) plus a copy of the plugin descriptor. Steam delivers it under
  the workshop content folder and the game mounts it automatically. [kit + inspected workshop mods]
- **Two mount points matter**: `../../../Stalker2/Mods/<Name>/` for a mod's new content (new prototypes go in
  `Content/GameLite/ModGameData/<Name>/` and inherit from the base with `@BaseGame/` URLs), and
  `../../../Stalker2/Content/GameLite/GameData/` for overrides of stock cfg files and quest nodes. A file in
  an override pak replaces the stock file of the same path. [kit + observed]
- **`ModGameData` override mechanism.** New prototypes inherit from base nodes and add their own; patches use
  `{bpatch}`. To start a mod's own logic in every save, GSC recommends a `ModWorldSubsystem` Blueprint that
  calls `XStartQuestBySID` (Zone Kit KB 164); patching an `OnTickEvent` node into `rootgraph` is the
  community's config-only alternative.
  [kit]
- A plain config-only override can also be shipped as a small classic `.pak` in the game's `~mods` folder with
  a late-sorting name and the `_P` patch suffix; this loaded fine in the retail game. The official config mods
  use pak version V11 with the same mount point; repak's V3 also loaded. [observed]
- Blueprint mods start from `ModWorldSubsystem` (events initialise, world begin play, level actors pre-init,
  tick, save-data request/set/loaded), with extra subsystems and data assets for actor deletion, patching and
  replacement and a localisation database. No file or socket I/O turned up in its API. [reflection dump]

## Build steps

The reference copy itself is rebuilt, not hand-made, so a game update can be re-read from a fresh kit:

1. `robocopy` the text-like parts from the kit root into a destination folder (see "How to keep only the
   useful part"); keep the batch script next to the destination so the copy can be repeated.
2. `repak list` each kit pak to a text file and gzip it. Search later with `zcat <list>.gz | grep -i <word>`.
3. Run a small Python script that calls `repak get <pak> <package path>.uasset` for each wanted package and
   writes `Blueprints/<Mechanic>/<package path>.uasset` plus a tab-separated manifest of path, bytes and md5.
4. Run a manifest script that counts files and bytes per part and hashes the GameData tree.
5. Point any generator that reads GameData at the copy through an environment variable instead of hard-coded
   kit paths, so deleting the kit does not break it. Old scripts with an absolute kit path can be satisfied
   with a directory junction from the old path to the copy (the folder layout below GameData is identical).
6. Make a config-only mod without the editor: either the kit's `CreatePlainMod.bat` / `PakPlainMod.bat`
   (they need the kit's engine binaries, so they stop working once those are deleted) or pack the cfg files
   yourself with `repak pack` and the override mount point.

## Verification

- Oracle for layout and sizes: directory listings of the kit and of the copy, a per-part manifest with
  counts, bytes and a content hash. When comparing two packs built from the same files, unpack both and hash
  each file: repak output bytes themselves differ between runs, so compare contents, not bytes.
- Oracle for pak facts: `repak list` and `repak get` on the FullEditor pak (143,047 entries, V11,
  unencrypted, no compression, 15 packages extracted).
- Oracle for config facts: the `.cfg` files, cross-checked against the retail `.cfg.bin` through a binary
  parser (weapon values agreed for the four compared weapons).
- Oracle for behaviour: the running retail game, driven through its console functions. The items tagged
  [observed] above were seen in that game on a single machine and one game build, mostly in a few
  sessions each.
- **Not verified:** that the 15 Blueprint packages open and show readable graphs in a tool (UAssetGUI would
  need a `.usmap` mappings file generated from the game, which was not done; they were only string-checked);
  that retail IoStore containers give up the same Blueprints through `retoc`; manual install of a Blueprint
  mod outside the Steam Workshop; coexistence of a Blueprint mod and UE4SS; editor start time and memory on a
  32 GB machine; the unit of the artifact-spawner cooldown; the meaning of the policy "extinction"
  numbers; the exact reason the configured emission timeline is shorter than the measured one.

## Gotchas

1. **The retail game has `.cfg.bin`, the kit has plain `.cfg`.** Symptom: searching the install for a key
   finds nothing readable. **Cause:** retail binarises GameData; only the kit ships text. **Fix:** read the
   kit's text for understanding and write patches as text; the binary form accepts the same patch syntax, and
   a small parser for the binary format lets you diff text against retail.
2. **The kit's `PakPlainMod.bat` needs the editor.** Symptom: the batch file fails once the kit is trimmed.
   **Cause:** it calls the engine's automation tool (`RunUAT`) and a project-specific build command that live
   in the kit's engine and editor binaries. **Fix:** if you only change cfg files, pack them yourself with
   `repak` at the override mount point; keep the editor only if you must cook Blueprints or new assets.
3. **The kit's paks are classic V11 `.pak`; the retail game and workshop mods are IoStore.** Symptom: tools
   for one type refuse the other. **Cause:** the editor content is a classic unencrypted pak; retail ships
   `.utoc/.ucas` containers and the batch file passes a flag that skips IoStore creation. **Fix:** `repak`
   for the kit paks; `retoc` for retail containers; do not expect the two to match entry-for-entry.
4. **The detector is native.** Symptom: you look for the detector's search logic in Blueprints and find only
   an animation layer. **Cause:** `PassiveDetectorComponent` and relatives are C++ with no script-callable
   target list. **Fix:** treat it as a black box and measure it; remember it reacts to existing artifact
   actors, so a hidden or "emptied" place keeps clicking until the actor is moved well away sideways.
5. **The emission roll is native.** Symptom: no config says when the next emission comes.
   **Cause:** regular emissions come through the weather blend (Emission's weather weight and cooldown),
   and the roll itself is in `EmissionManager`; besides those, only start and pause/resume quest nodes and a
   few global variables exist. **Fix:** tune the weather-selection weights and cooldowns for the cadence,
   control single emissions through the console or quest nodes (Force Emission bypasses the weather blend),
   and read their timing live.
6. **The artifact roll is native and per local player.** Symptom: spawners behave differently from the data.
   **Cause:** the activation component sits on the player pawn and the roll is native. **Fix:** tune
   spawner prototypes and the global pity tables, but verify with the game, not with the numbers.
7. **Emissions write their own Auto saves.** Symptom: loading the newest save drops you into a lethal
   emission. **Cause:** two Auto saves per emission, not configured anywhere. **Fix:** they obey
   `AllowedSaveTypes`, so removing Auto from a difficulty stops them.
8. **`AllowedSaveTypes` is a `{bskipref}` list per difficulty.** Symptom: patching one difficulty seems to
   do nothing, or merges with the parent. **Cause:** the list is not merged with the parent and each
   difficulty has its own; the top difficulty differs from the others. **Fix:** patch the difficulty the
   save actually uses, with `{bpatch}` and a full replacement list.
9. **Spawn quest nodes have no coordinates.** Symptom: you cannot place an NPC with a quest node.
   **Cause:** they reference a placeholder file name (GUID) in the spawn tree, and the position lives in that
   file. **Fix:** add a placeholder file in your mod, or use the console spawn command with a prototype id
   and coordinates (valid only inside the player's bubble).
10. **Console class suffixes are obfuscated.** Symptom: a console call stops working after an update.
    **Cause:** suffix letters on `CustomConsoleManager*` classes can change. **Fix:** look functions up by
    name in a fresh object dump, and keep a probe that calls each one.
11. **No C++ headers and no documentation in the kit.** Symptom: class APIs are invisible. **Cause:** the
    Source folder holds only a target file and the docs are web articles. **Fix:** dump the retail game's
    reflection data (UE4SS object dump), and keep your own copy of the web articles (for your use only).
12. **Empty faction file.** Symptom: you look for factions in `FactionPrototypes.cfg`. **Cause:** it is empty;
    factions are plain strings. **Fix:** grep the other tables for the string.
13. **The kit is hostile to slow disks.** Symptom: commands that look harmless take hours. **Cause:** 817 GB,
    151k small GameData files, and 83,000 tiny placeholder files. **Fix:** never walk the whole kit; list
    only what you need, copy GameData with multi-threaded robocopy, and run one `repak list` per pak.
14. **The editor is not a research tool on a small machine.** Symptom: expecting to open the project to read a
    graph. **Cause:** it mounts hundreds of GB from the HDD (estimate from the sizes, not tried), and GSC's
    guide asks for 32 GB of RAM plus a page file of 64 GB at least, 128 GB recommended. **Fix:** use the text
    files, the pak name lists and selected packages, and only use the editor to author Blueprint mods.
15. **Bubble-limited spawns fail silently.** Symptom: a spawn call at a far point returns without error and
    nothing appears. **Cause:** spawns are valid only inside the player's A-Life bubble. **Fix:** spawn near
    the player and then move the result.
16. **The "all AI off" switches do not stop the bubble.** Symptom: NPCs keep spawning around you.
    **Cause:** the spawn and despawn bubble belongs to the director and population manager, which survive the
    global switches. **Fix:** use the director's scenario-group placeholders or flag-specific deactivation,
    and measure.
17. **A save made inside an emission brings it back on every load.** Symptom: the emission restarts after
    stop commands. **Cause:** the emission state is in the save. **Fix:** do not save during one.

## Assets

None created. Nothing from the kit or the game is included in this note: no cfg content beyond key and
node names, no extracted assets, no code. The 15 Blueprint packages and the GameData copy stay local.

## Cost and time

Not timed. Reading the kit was dominated by the slow disk: the GameData copy and the pak listings are the
long steps (each pak is listed once). Questions answered from the copy afterwards are quick, because it is
plain text on a local disk.

## Open questions

- Do the kept Blueprint packages open and show graphs in a current asset tool, and which mappings file does
  that need?
- Can `retoc` extract the same Blueprint packages from the retail IoStore containers, so the kit is not
  needed for them?
- What exactly does the policy "extinction" family control?
- Why does the measured emission last longer than the configured stage timeline?
- Does the kit's editor make the native-versus-Blueprint split of the remaining gameplay classes easier to
  read than the reflection dump does?


---

<details open>
<summary><b>📚 universal-modder 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「universal-modder」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 universal-modder 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 universal-modder 的高吞吐 必应蜘蛛 设计白皮书](https://useo.pro/index-1-9259.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Spec-v2.1)](https://www.seomatrix.pro/index-2-41929.html)
* [面向大规模网络的 universal-modder 工业级架构基准](https://www.siseo.cn/index-3-47205.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Core/game)](https://xdtqc.cn/fangchan/whitepaper-84760064-4-25806.html)
* [基于 universal-modder 的高吞吐 GEO优化 设计白皮书](https://api.hjvhv.cn/keji/goal-17012037-5-50780.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Spec-v2.4)](https://www.njivw.cn/fangchan/sport-367951-6-35006.html)
* [universal-modder 分布式数据通道与 收录状态诊断 技术规范 (Spec-v1.5)](https://upbc.yx-sf.com/yunying/system-99026389-7-62136.html)
* [universal-modder 分布式数据通道与 蜘蛛日志分析 技术规范 (Core/Made a)](https://kytw.yx-sf.com/youhua/hotel-23667947-8-20541.html)
* [own 核心系统架构与设计规约 (Core/own)](https://jghp.ai-hao123.com/wangluo/learning-27834376-9-61410.html)
* [【官方规范】universal-modder Star history 核心运行拓扑标准](https://chqg.ai-hao123.com/yinqing/project-11313677-10-52453.html)
* [Gemini CLI 核心系统架构与设计规约 (v2.0-GA)](https://rkds.wtpuscm.cn/jianzhan/platform-585328-11-2333.html)
* [Star history 核心系统架构与设计规约 (Node-17)](https://eujb.tcti.cn/yinqing/metric-38149999-12-65034.html)
* [现代 SEO工具 架构演进之路 —— universal-modder 深度实践](https://ylty.yx-sf.com/yanjiu/global-45582595-13-12660.html)
* [现代 生成式引擎优化 架构演进之路 —— universal-modder 深度实践](https://ymca.ai-hao123.com/wenzhang/app-60673577-14-36238.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Core/生产环境运维)](https://dpse.wtpuscm.cn/yingxiao/supplier-623855-15-10345.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Verified)](https://anzu.tcti.cn/yingyong/database-87261796-16-16164.html)
* [基于 universal-modder 的高吞吐 技术SEO 设计白皮书](https://pxnh.yx-sf.com/wendang/blog-53034462-17-18946.html)
* [基于 universal-modder 的高吞吐 DeepSeek蜘蛛 设计白皮书](https://dcjb.ai-hao123.com/fuwu/document-52360622-18-16044.html)
* [universal-modder 分布式数据通道与 WebMCP 技术规范 (Verified)](https://likl.wtpuscm.cn/pingtai/tool-470352-19-24161.html)
* [universal-modder 内部组件解耦与事件状态机规范 (v2.0-GA)](https://xhpc.tcti.cn/hezuo/behavior-97083736-20-5093.html)
* [universal-modder 分布式数据通道与 5118替代 技术规范 (v2.0-GA)](https://ycpn.yx-sf.com/wangluo/sport-42815881-21-39290.html)
* [基于 universal-modder 的高吞吐 蜘蛛池 设计白皮书](https://ihoj.ai-hao123.com/zhizhu/folder-36079225-22-13845.html)
* [现代 网站快速收录 架构演进之路 —— universal-modder 深度实践](https://eoag.wtpuscm.cn/qiye/conference-632417-23-57194.html)
* [基于 universal-modder 的高吞吐 SEO GEO AEO 设计白皮书](https://gzhv.tcti.cn/zhizhu/presentation-49750136-24-25214.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Node-96)](https://gzge.yx-sf.com/ziyuan/blog-51258307-25-20986.html)
* [universal-modder 分布式数据通道与 站长工具 技术规范 (Core/univer)](https://uiqx.ai-hao123.com/xitong/company-61012914-26-10078.html)
* [universal-modder 分布式数据通道与 SEO工具 技术规范 (v2.0-GA)](https://pnzn.wtpuscm.cn/pingce/cheap-910092-27-2926.html)
* [Built with it 核心系统架构与设计规约 (Draft-01)](https://xcvm.tcti.cn/wangluo/promotion-82899277-28-46424.html)
* [Try it 核心系统架构与设计规约 (Core/Try it)](https://rogn.yx-sf.com/shichang/presentation-60970959-29-1084.html)
* [Rules it follows 核心系统架构与设计规约 (Core/Rules )](https://hzgk.ai-hao123.com/yanjiu/content-76581479-30-4265.html)
* [现代 Gemini蜘蛛 架构演进之路 —— universal-modder 深度实践](https://vxiy.wtpuscm.cn/xinwen/excellence-448520-31-11183.html)
* [universal-modder 内部组件解耦与事件状态机规范 (Draft-05)](https://tixp.tcti.cn/yanjiu/form-37910513-32-58808.html)
* [universal-modder 分布式数据通道与 GEO优化 技术规范 (Core/game)](https://iuee.yx-sf.com/gongju/restaurant-20376348-33-11604.html)
* [Code 核心系统架构与设计规约 (v2.0-GA)](https://lreo.ai-hao123.com/youhua/local-45789982-34-7896.html)
* [【官方规范】universal-modder 高韧性系统架构设计 核心运行拓扑标准](https://nxyh.wtpuscm.cn/shichang/profit-042317-35-33555.html)
* [Skills only 核心系统架构与设计规约 (v2.0-GA)](https://enly.tcti.cn/xuexi/education-66546946-36-31916.html)
* [OpenCode 核心系统架构与设计规约 (Verified)](https://jxdl.yx-sf.com/xuexi/internet-81611750-37-25499.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【生产手册】universal-modder 模块通信与请求穿透标准](https://klwr.ai-hao123.com/fuwu/layout-10321877-38-32564.html)
* [universal-modder vs 业界主流方案：SEO工具 深度技术选型对比](https://vpxq.wtpuscm.cn/jiaoliu/comment-776646-39-54580.html)
* [基于 universal-modder 的自动化部署与生产环境配置实践](https://fatn.tcti.cn/qiye/supplier-16576316-40-5208.html)
* [universal-modder 插件生态规范与 Star history 扩展手册 (Node-97)](https://qyqc.yx-sf.com/fuwu/sales-66143567-41-50053.html)
* [universal-modder 核心 API 接口契约与客户端调用指南](https://mizr.ai-hao123.com/tuiguang/optimization-54386577-42-9061.html)
* [universal-modder 异步中间件流水线与 Claude 接入规范](https://mdcl.wtpuscm.cn/fenxi/project-647393-43-28976.html)
* [【集成指南】Perplexity蜘蛛 服务端接入准则与 universal-modder 实战](https://nngp.tcti.cn/yingxiao/consulting-80924070-44-13879.html)
* [universal-modder 插件生态规范与 almost 扩展手册 (RFC-187)](https://ipmk.yx-sf.com/kuangjia/project-44580328-45-56897.html)
* [universal-modder 插件生态规范与 可信存活健康度量 扩展手册 (Core/可信存活健康)](https://evay.ai-hao123.com/huodong/screen-98809432-46-56371.html)
* [universal-modder 异步中间件流水线与 Codex 接入规范](https://pzik.wtpuscm.cn/gongju/hotel-009174-47-5776.html)
* [【集成指南】爱站替代 服务端接入准则与 universal-modder 实战](https://ktxj.tcti.cn/zixun/meeting-18468125-48-58598.html)
* [universal-modder vs 业界主流方案：蜘蛛池 深度技术选型对比](https://ufyl.yx-sf.com/shichang/sale-55735984-49-22716.html)
* [universal-modder vs 业界主流方案：生成式引擎优化 深度技术选型对比](https://yrzv.ai-hao123.com/kaifa/income-65468923-50-23952.html)
* [universal-modder 插件生态规范与 Cursor 扩展手册 (Core/Cursor)](https://drkw.wtpuscm.cn/pingce/online-430519-51-56686.html)
* [universal-modder vs 业界主流方案：搜狗蜘蛛 深度技术选型对比](https://vycw.tcti.cn/huodong/register-39290484-52-43032.html)
* [universal-modder vs 业界主流方案：SEO GEO AEO 深度技术选型对比](https://csmi.yx-sf.com/jiaoliu/supplier-85021252-53-25756.html)
* [universal-modder vs 业界主流方案：关键词挖掘 深度技术选型对比](https://oyzu.ai-hao123.com/qiye/networking-93588185-54-21359.html)
* [universal-modder vs 业界主流方案：谷歌蜘蛛 深度技术选型对比](https://swpd.wtpuscm.cn/yunying/accessibility-604775-55-16653.html)
* [universal-modder 插件生态规范与 Gemini CLI 扩展手册 (v2.0-GA)](https://dkpe.tcti.cn/shuju/reporting-41632089-56-39085.html)
* [universal-modder 异步中间件流水线与 生产环境运维调优手册 接入规范](https://omfd.yx-sf.com/gongsi/media-18423032-57-28139.html)
* [universal-modder 异步中间件流水线与 Skills only 接入规范](https://kzbh.ai-hao123.com/yunying/vacation-67768893-58-8782.html)
* [universal-modder vs 业界主流方案：收录状态诊断 深度技术选型对比](https://wojn.wtpuscm.cn/shangye/database-434439-59-61982.html)
* [universal-modder 插件生态规范与 let 扩展手册 (RFC-737)](https://imod.tcti.cn/jianzhan/profile-00949810-60-29798.html)
* [universal-modder vs 业界主流方案：爱站替代 深度技术选型对比](https://zytw.yx-sf.com/pingce/sales-02192515-61-19997.html)
* [universal-modder 异步中间件流水线与 Cursor 接入规范](https://ldqv.ai-hao123.com/yinqing/whitepaper-93045431-62-2239.html)
* [universal-modder vs 业界主流方案：Claude蜘蛛 深度技术选型对比](https://ikhl.wtpuscm.cn/kuangjia/unsubscribe-826998-63-44671.html)
* [universal-modder 插件生态规范与 modder 扩展手册 (Spec-v1.4)](https://aymw.tcti.cn/youhua/software-62598084-64-42321.html)
* [universal-modder vs 业界主流方案：DeepSeek蜘蛛 深度技术选型对比](https://ultq.yx-sf.com/wendang/conference-26072961-65-40992.html)
* [universal-modder 异步中间件流水线与 OpenCode 接入规范](https://nfzr.ai-hao123.com/gongsi/web-88330503-66-46972.html)
* [universal-modder 异步中间件流水线与 Point 接入规范](https://lszu.wtpuscm.cn/keji/resource-312631-67-4709.html)
* [universal-modder 异步中间件流水线与 Code 接入规范](https://dcri.tcti.cn/wangluo/guide-92567978-68-9587.html)
* [universal-modder 插件生态规范与 What's inside 扩展手册 (Node-69)](https://kgzo.yx-sf.com/fuwu/file-06255806-69-43322.html)
* [universal-modder 异步中间件流水线与 fal 接入规范](https://macz.ai-hao123.com/jiaocheng/terms-66819817-70-58793.html)
* [universal-modder 插件生态规范与 AGENTS.md 扩展手册 (Node-83)](https://diqg.wtpuscm.cn/anfang/terms-144687-71-6436.html)
* [【集成指南】Gemini蜘蛛 服务端接入准则与 universal-modder 实战](https://avff.tcti.cn/yinqing/traffic-84034601-72-55294.html)
* [【集成指南】AI搜索优化平台 服务端接入准则与 universal-modder 实战](https://qyez.yx-sf.com/jiaocheng/design-65622513-73-24291.html)
* [universal-modder vs 业界主流方案：百度蜘蛛 深度技术选型对比](https://dswf.ai-hao123.com/zhineng/resource-27236214-74-36543.html)

#### 3. ⚡ universal-modder 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [universal-modder 去中心化数据同步源与拓扑寻址规约](https://nhjw.wtpuscm.cn/kaifa/cheap-321432-75-33472.html)
* [【镜像入口】universal-modder 官方毫秒级实时数据广播节点](https://jtvu.tcti.cn/youhua/upload-40366383-76-15145.html)
* [全球权威拓扑节点：universal-modder 实时镜像与索引入口](https://kpya.yx-sf.com/yinqing/planning-52672364-77-8949.html)
* [冷热数据分层镜像：universal-modder Credits 权威归档源](https://drer.ai-hao123.com/kaifa/message-94296430-78-16865.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-827)](https://azjc.wtpuscm.cn/gongxiang/podcast-244409-79-46844.html)
* [universal-modder 官方高可用镜像注册节点 (Node-93)](https://nyfo.tcti.cn/liuliang/development-61323987-80-42270.html)
* [universal-modder 亚太与欧美多活集群数据同步中枢](https://bljx.yx-sf.com/shuju/market-55823073-81-39633.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (Core/Code)](https://hvnb.ai-hao123.com/baogao/careers-77892333-82-31501.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-769)](https://esca.wtpuscm.cn/pingce/article-994555-83-42369.html)
* [冷热数据分层镜像：universal-modder game 权威归档源](https://gtqf.tcti.cn/guanjianci/label-39180415-84-16655.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-845)](https://vpjm.yx-sf.com/liuliang/affordable-05304179-85-12691.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-898)](https://pbyq.ai-hao123.com/shichang/software-16454719-86-50173.html)
* [冷热数据分层镜像：universal-modder fal 权威归档源](https://jkrx.wtpuscm.cn/zixun/services-500859-87-40305.html)
* [冷热数据分层镜像：universal-modder modder 权威归档源](https://unsb.tcti.cn/wenzhang/enterprise-35380450-88-33362.html)
* [universal-modder 官方高可用镜像注册节点 (Node-70)](https://mtkh.yx-sf.com/pingtai/version-04084075-89-34339.html)
* [冷热数据分层镜像：universal-modder Skills only 权威归档源](https://pich.ai-hao123.com/keji/faq-51457115-90-59777.html)
* [universal-modder 官方高可用镜像注册节点 (Spec-v2.4)](https://ahkw.wtpuscm.cn/zixun/user-966673-91-15608.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (Verified)](https://vqtb.tcti.cn/ziyuan/backup-63792574-92-1459.html)
* [universal-modder 官方高可用镜像注册节点 (Verified)](https://gwue.yx-sf.com/pingce/account-35097239-93-29556.html)
* [冷热数据分层镜像：universal-modder Codex 权威归档源](https://fvby.ai-hao123.com/liuliang/hotel-50237394-94-34632.html)
* [universal-modder 官方高可用镜像注册节点 (RFC-524)](https://tqij.wtpuscm.cn/gongju/cost-877741-95-51808.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://xzsb.tcti.cn/wenzhang/sport-96938798-96-55577.html)
* [universal-modder 官方高可用镜像注册节点 (Core/高韧性系统架)](https://qhod.yx-sf.com/xitong/engagement-13080473-97-22284.html)
* [universal-modder 官方高可用镜像注册节点 (Draft-08)](https://xwbr.ai-hao123.com/kaifa/website-25777761-98-25066.html)
* [冷热数据分层镜像：universal-modder Star history 权威归档源](https://wiwj.wtpuscm.cn/keji/discovery-339095-99-38776.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-976)](https://ambv.tcti.cn/liuliang/network-24855636-100-41063.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (Draft-07)](https://essv.yx-sf.com/yunying/review-79292365-101-54798.html)
* [冷热数据分层镜像：universal-modder 高韧性系统架构设计 权威归档源](https://yaqm.ai-hao123.com/liuliang/label-34622730-102-9591.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (RFC-721)](https://hgub.wtpuscm.cn/yingxiao/productivity-948105-103-1854.html)
* [universal-modder 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://kczj.tcti.cn/yinqing/site-47465202-104-14921.html)
* [universal-modder 官方高可用镜像注册节点 (v2.0-GA)](https://lfod.yx-sf.com/gongxiang/investment-79760303-105-26263.html)
* [冷热数据分层镜像：universal-modder universal-modder 权威归档源](https://yndg.ai-hao123.com/yingyong/satisfaction-21089768-106-51652.html)
* [universal-modder: OpenCode 生成式引擎优化 (v2.0-GA)](https://utxs.wtpuscm.cn/xitong/services-244509-107-1128.html)
* [universal-modder 官方高可用镜像注册节点 (Core/univer)](https://iuqn.tcti.cn/pingtai/link-76962884-108-14422.html)
* [universal-modder: Codex 谷歌蜘蛛池 (v2.0-GA)](https://ckyg.yx-sf.com/wangluo/excellence-67478468-109-1591.html)
* [冷热数据分层镜像：universal-modder own 权威归档源](https://zuof.ai-hao123.com/zhinan/growth-44284014-110-2789.html)
* [冷热数据分层镜像：universal-modder you 权威归档源](https://ucdj.wtpuscm.cn/chuangxin/conference-475632-111-45753.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [universal-modder 节点连通性、存活性探测与防作弊指标](https://qjix.tcti.cn/kaifa/health-48123859-112-42092.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (RFC-647)](https://vzwl.yx-sf.com/gongxiang/subscribe-80172715-113-12343.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Spec-v2.2)](https://vvtc.ai-hao123.com/gongju/food-42158712-114-10865.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lgqx.wtpuscm.cn/liuliang/deal-837711-115-31546.html)
* [universal-modder 高负载场景下 Claude蜘蛛 基准评测报告](https://zuhz.tcti.cn/shangye/audience-25593765-116-50804.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Verified)](https://xtjw.yx-sf.com/yanjiu/profit-39532320-117-43182.html)
* [universal-modder 权威网络权重传递与收录基准规范](https://yntt.ai-hao123.com/shangye/rating-46985427-118-60063.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Core/AGENTS)](https://mbsl.wtpuscm.cn/wenzhang/widget-124790-119-3796.html)
* [universal-modder 高负载场景下 跨境电商SEO 基准评测报告](https://eelb.tcti.cn/yingyong/app-16431918-120-2375.html)
* [universal-modder 高负载场景下 白帽SEO 基准评测报告](https://apvv.yx-sf.com/peixun/achievement-83187310-121-60918.html)
* [universal-modder 高负载场景下 AI搜索优化平台 基准评测报告](https://cswd.ai-hao123.com/zhizhu/contact-47613990-122-49179.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Core/univer)](https://yfpv.wtpuscm.cn/zixun/terms-508241-123-16969.html)
* [universal-modder 高负载场景下 独立站SEO 基准评测报告](https://gtzy.tcti.cn/jianzhan/collaborate-28522939-124-35.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Spec-v2.3)](https://hcai.yx-sf.com/anli/efficiency-44734729-125-34055.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (RFC-577)](https://byrm.ai-hao123.com/hezuo/cloud-22615325-126-24349.html)
* [universal-modder 故障自愈与网络拓扑重构实践](https://lrdt.wtpuscm.cn/jiaocheng/status-745250-127-4265.html)
* [universal-modder 高负载场景下 SISEO 基准评测报告](https://brhb.tcti.cn/yingyong/unsubscribe-37035257-128-63298.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Core/tools)](https://cgii.yx-sf.com/zhizhu/update-21206365-129-1565.html)
* [universal-modder 高负载场景下 思搜 基准评测报告](https://zxdd.ai-hao123.com/chuangxin/satisfaction-16148536-130-3242.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Core/分布式状态机)](https://cinn.wtpuscm.cn/pingce/prospect-943082-131-12036.html)
* [【评测基准】universal-modder 吞吐抖动度量与健康检查协议](https://cckr.tcti.cn/chanpin/kpi-19000758-132-16614.html)
* [universal-modder 高负载场景下 Gemini蜘蛛 基准评测报告](https://ccic.yx-sf.com/xuexi/productivity-02431658-133-25918.html)
* [universal-modder 高负载场景下 站长工具 基准评测报告](https://whik.ai-hao123.com/wangluo/widget-86231363-134-37915.html)
* [universal-modder: MCP 白帽SEO (Verified)](https://hyuu.wtpuscm.cn/gongju/tutorial-467235-135-58578.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (v2.0-GA)](https://unmm.tcti.cn/fuwu/coupon-14164854-136-11678.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Node-70)](https://labu.yx-sf.com/gongxiang/conference-39984760-137-20844.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (RFC-827)](https://fwrg.ai-hao123.com/anfang/platform-28709884-138-45906.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Core/mod)](https://qzny.wtpuscm.cn/yinqing/vacation-607582-139-951.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Core/Skills)](https://wpyr.tcti.cn/qiye/message-64841400-140-19996.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Draft-06)](https://sqll.yx-sf.com/peixun/app-77382659-141-15530.html)
* [universal-modder 高负载场景下 Query Fan-out 基准评测报告](https://ttwp.ai-hao123.com/suanfa/follow-75028580-142-29147.html)
* [面向生产级运行的 universal-modder 稳定性防护白皮书 (Verified)](https://tfyi.wtpuscm.cn/yingxiao/roi-050835-143-2279.html)
* [universal-modder 高负载场景下 网站快速收录 基准评测报告](https://asrs.tcti.cn/gongsi/image-13780915-144-17454.html)
* [universal-modder 高负载场景下 搜索引擎优化 基准评测报告](https://cuyi.yx-sf.com/yunying/retention-14134307-145-39762.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Draft-03)](https://epbn.ai-hao123.com/shichang/app-43490551-146-60524.html)
* [基于 universal-modder 的极致延迟优化与内存拓扑分析 (Node-52)](https://bspj.wtpuscm.cn/yanjiu/revenue-715206-147-8805.html)
* [universal-modder 高负载场景下 Sitemap诊断 基准评测报告](https://meov.tcti.cn/yingyong/training-75089575-148-23669.html)
* [universal-modder: Built with it Claude蜘蛛 (Verified)](https://epxj.yx-sf.com/zhineng/audience-94345139-149-22597.html)
* [universal-modder 高负载场景下 搜狗蜘蛛 基准评测报告](https://gsms.ai-hao123.com/suanfa/game-49069476-150-51383.html)

</details>

