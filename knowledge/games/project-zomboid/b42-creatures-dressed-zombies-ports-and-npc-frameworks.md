---
kind: game
title: 'Creatures in Build 42: monsters as dressed zombies, porting creatures from other games, other chassis, and how the NPC frameworks work'
game: Project Zomboid
games_also: ["Cry of Fear", "Half-Life", "Cataclysm: Dark Days Ahead"]
game_version: 'Build 42.21.0 (Steam, Windows 10). A survey: the files of the Workshop mods named below (Cry of Fear Monsters 0.68.25 with versionMin=42.20, Bandits, CompanionDogs, ProjectALifeNPCs), a local-only Half-Life port, javap listings and the install''s Lua; nothing was built or run in game for this note'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", javap, "Blender + DirectX X importer", "Paddlefruit rig / PY-BobBatchExportGLB.py", "Crowbar (Half-Life .mdl)"]
anti_cheat: 'none (single-player creature mods; the multiplayer note covers servers you host)'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3268487204"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3740052292"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3803984183"
- "https://pzwiki.net/wiki/Animation_guide"
tags: [b42, creatures, custom-monster, zombies, clothing, bodylocation, persistent-outfit, animsets, animnode, animation-variables, spawning, outfits, cross-game-port, npc, survivors, companion, bandits, mannequin, thumpable, case-study]
---
# Creatures in Build 42: dressed zombies, ports from other games, other chassis and NPC frameworks

> How custom creatures are built on Project Zomboid Build 42, read from mods that ship them: Svide's *Cry of Fear
> Monsters* (23 humanoid monsters, each a **zombie wearing an invisible whole-body clothing item** whose model is
> the monster), a local Half-Life Houndeye port on the turkey's animal rig, and three NPC frameworks (Bandits,
> CompanionDogs, ProjectALife). The animal route is in main's
> [CDDA ants note](cdda-enemy-pack-ants-custom-hostile-animals-a-breeding-colon.md) and isn't repeated here. This
> is a survey of other people's mod files plus javap listings; nothing was built and run in game.

## Setup
- Project Zomboid **42.21.0**, Steam, Windows 10. General mod layout and the Lua runtime: the
  [orientation note](b42-modding-orientation-layout-lua-runtime-tooling-interop-ui.md); Java names: the
  [Java-surface note](b42-java-surface-lua-callable-api-and-engine-internals.md).
- Mods read (structure only, described in our own words):
  - *Cry of Fear Monsters* by Svide (Steam Workshop), v0.68.25, `versionMin=42.20`: about 13k lines of Lua in
    `42/media/lua/{shared,client,server}`, data in `common/media/{models_X,anims_X,clothing,scripts,sound,
    textures,fileGuidTable.xml,registries.lua}` and `42/media/{AnimSets,sandbox-options.txt}`. Its files carry
    the authors' own bug history in comments.
  - *HLMonsters*: a Half-Life Houndeye port that only exists in the contributor's local `Zomboid/mods` folder; it
    was never published.
  - Bandits (Workshop 3268487204), CompanionDogs (3740052292) and ProjectALifeNPCs (3803984183).

## Route and why
Loader API: everything below is the game's own Lua plus its asset pipeline. The first decision is the chassis:

| | Dressed zombie (Cry of Fear) | B42 animal ([CDDA note](cdda-enemy-pack-ants-custom-hostile-animals-a-breeding-colon.md)) |
|---|---|---|
| Base object | `IsoZombie` wearing a custom whole-body clothing item | `IsoAnimal` (a subclass of `IsoPlayer`) |
| Free | hunting and attacking players, hit reactions, deaths, crawling, fake-dead, vehicle hits, netcode | pathing, eating, fleeing, growth, butchering, saving |
| You build | the look (body item, model, AnimSet nodes), identity, spawning | the whole attack brain in Lua |
| Skin to | the `Bip01` human rig (Bob/Kate) | the closest animal rig (pig, turkey, raccoon ...) |
| Best for | humanoids that hunt players | creatures with their own behaviour, packs, colonies |

Rule of thumb from the ports: if the creature should hunt the player like a zombie, use a zombie and get the AI
free; if it needs its own behaviours, use an animal and write the brain.

## How the game works (what we had to learn)

### The dressed-zombie chassis (Cry of Fear Monsters)
- **One new body location.** `common/media/registries.lua` is one call,
  `ItemBodyLocation.register("COF:MonsterBody")`. At `OnGameBoot` a shared Lua file takes the "Human"
  `BodyLocationGroup`, calls `getOrCreateLocation` for the new location and `moveLocationToIndex` to put it at the
  `UNDERWEAR_BOTTOM` index, the lowest body layer. The monster's model then replaces the zombie's visible body and
  ordinary clothing still renders over it.
- **The body is an invisible clothing item:** a script item with `ItemType = base:Clothing`,
  `ClothingItem = COF_<Monster>Body`, `BodyLocation = COF:MonsterBody`, `WorldRender = false`, `hidden = true`,
  `Tags = base:noragdoll`.
- **The clothingItem XML** points `m_MaleModel`/`m_FemaleModel` at the mesh in `models_X`, has one fixed GUID,
  sets all 16 `m_Masks` (hiding the base body), `m_Static` false and one `textureChoices` atlas. A matching
  corpse XML per monster keeps the look on dead bodies. Every clothing XML's GUID goes into `fileGuidTable.xml`.
- **Identity is the outfit, not mod data.** Each monster is a persistent outfit containing only its body item.
  `dressInPersistentOutfit(name)` applies it, `getOutfitName()` reads it, `isPersistentOutfitInit()` says whether
  the game has dressed the zombie yet.

### Why identity is the hard part (from the mod's own bug notes)
- **The engine recycles zombie objects.** A recycled zombie keeps its old `getOutfitName()` until it's re-dressed,
  which happens when it comes into view. Controllers that claim zombies by outfit name then grab the wrong one,
  and two controllers strip each other's body item (a naked zombie playing the wrong sounds).
- **Mod-data marks get lost** when a monster leaves and re-enters the loaded area, so unmarked types spawn again
  (their log: 13 of one monster and endless flowers in an hour).
- **Their fix:** in `OnZombieCreate`, when `isPersistentOutfitInit()` is false, clear the stale outfit with
  `getHumanVisual():setOutfit(nil)` and immediately `dressInPersistentOutfitID(getPersistentOutfitID())`, so the
  controller owns the zombie before it's ever seen. The high half of `getPersistentOutfitID()` is the outfit
  index, and a table learned from dressed monsters maps index to monster.
- **One dispatch point:** every controller registers with the mod's dispatcher on `OnZombieUpdate`; a weak-keyed
  "previous identity" table spots a recycled zombie and gives the old controller one cleanup pass (stop loops,
  drop tables).

### Animation plumbing
- **The stack:** an AnimSet (per entity, `media/AnimSets/<entity>/<state>/*.xml`), an ActionGroup state machine
  over its states (`media/actiongroups/`), and `animNode` XML: `m_AnimName`, `m_Looped`, `m_BlendTime` and
  `m_Conditions` on animation variables that Lua sets with `character:setVariable("Name", value)`.
- **Merge, don't replace.** Mod XML under `media/AnimSets/zombie/<state>/` merges into the zombie's graph. Cry of Fear
  adds nodes to the vanilla states (`attack`, `walktoward`, `idle`, `hitreaction`, `lunge`, `getup`, `eatbody`,
  `fakedead`, `bumped`, the vehicle-collision and corpse-drag states ...) gated on `COF_Is<Monster> == true`, and
  picks attack clips through a STRING variable the conditions read. The vanilla `ZombieBiteDone` variable gates
  bite damage, and controllers set it themselves. Their `COF_AnimTemplates/` folders are reference copies that
  the game doesn't load (only `media/AnimSets/` is read).
- **Clip folders:** only `anims_X` subfolders named by the model's `animationsMesh` script entry are scanned, with
  the prefix that entry sets; human rigs scan `Bob/` and `Kate/` with `Bob_`/`Kate_` prefixes, which come from
  each ActionGroup root's `animNodeFilter` (`media/actiongroups/zombie.xml`, `player.xml` ...). The install's
  `Master_Variables.xml` lists the variables conditions can read.
- **Formats:** `.glb` (recommended, hot-reloadable; the Paddlefruit community rig and its
  `PY-BobBatchExportGLB.py` exporter, 30 fps), `.fbx`, or vanilla text `.x`. Import vanilla `.x` into Blender with
  the "DirectX X Importer" extension or ZomboidAssetConverter. Models load through the game's assimp
  (`jassimp64`), so no conversion to `.x` is needed. AnimZed, TIS's own tool, was announced but not released at
  research time.
- `media/animscript/combat.xml` is a second, older AnimStates language (`<PlayAnim>`/`<Out>` states); don't mix
  it up with AnimSet `animNode` files.

### Spawning
- **Outfits** are `media/clothing/clothing.xml` entries per sex: a name, a GUID and items by clothingItem GUID with
  optional probabilities and alternates. Every item GUID must be in your `fileGuidTable.xml` or the outfit
  silently fails.
- **Zone spawn tables:** `media/lua/shared/NPCs/ZombiesZoneDefinition.lua` maps a map zone name to
  `chanceToSpawn`/`toSpawn` and outfit entries (`name`, `toSpawn`, `gender`, `mandatory`, `room`, `chance`). A mod
  adds keys from a shared Lua file: that's how "special zombies in special places" works (vanilla: a wedding party
  in a church).
- **By hand:** `addZombiesInOutfit(x, y, z, count, outfit, femaleChance, ...)` needs the zombie population on.
  With population None, Cry of Fear calls `VirtualZombieManager.instance:createRealZombieNow(x, y, z)` and sets
  everything itself (`setFemaleEtc`, `dressInPersistentOutfit`, `setCrawler`, `setFakeDead`, `setHealth` ...).
  It works only on the server (an MP client gets an empty result), and an old version read a Java field there and
  spawned nothing.
- On top sits the mod's own spawn system: weighted rarity per monster independent of the zombie population,
  per-monster sandbox toggles and rarity enums, a night event, spawn diagnostics in the log, and debug
  right-click spawns.
- Multiplayer: creation is server-only, controllers early-return on `isServer()` and the factory on `isClient()`;
  the server tells clients "this zombie is monster X" and each client sets the variables and visuals locally. See
  the [multiplayer note](b42-multiplayer-internals-packets-gameserver-sync.md).

### Porting a creature from another game: three asset routes
- **A. The other game's own assets** (Cry of Fear Monsters ships Cry of Fear's models, animations and sounds;
  its Workshop page credits them to Team Psykskallar and states no licence or permission). If the source game
  exports skinned meshes and clips as `.fbx`/`.glb`/`.x`, they load directly.
- **B. Converted assets** (HLMonsters): Half-Life `.mdl` -> Crowbar (or similar) to `.smd`/`.fbx` -> re-skin to a PZ
  rig -> text `.x` (`media/models_X/Skinned/Houndeye.x`, clips in `media/anims_X/houndeye/he_*.x`, an
  `animationsMesh` with `animationDirectory = houndeye`, `animationPrefix = he_`). Its AnimSet is a copy of the
  turkey's state folders with `he_` entries.
- **C. Generated assets** (the CDDA ants): no source assets at all; the pipeline is in the CDDA note's Assets.
- **Rights:** A and B are another game's copyrighted content, taken from your own install. Keep such a port
  private; a public release needs the rights holder's permission, or your own assets (route C). Converting a
  format doesn't change who owns it. HLMonsters stays local for this reason.
- **Skeleton:** reuse a PZ rig (`Bip01` or an animal's) so the vanilla AnimSet works; a custom skeleton means
  authoring every state. Rename source clips into PZ's state folders on export, and make sure the clip name
  inside each file matches `m_AnimName`.

### Other chassis (catalogue; none of these was tried)
| Body | Engine object | Notes |
|---|---|---|
| Friendly or neutral humanoid | `IsoZombie` that never gets a target | `clearAggroList`, no `spotted(player)`, steer with `pathToLocationF`/`pathToCharacter`; other zombies, melee auto-aim and player panic still treat it as a zombie |
| Pet or pack animal | `IsoAnimal` on a lead | `IsoPlayer:addAttachedAnimal`, `checkAnimalAttachedToRope`; troughs, milk, eggs |
| Static figure | `IsoMannequin` | `mannequin` script blocks, `registerMannequinZone` (on `IsoWorld` and `IsoMetaGrid`); poses, no skeletal animation |
| Turret, mimic, plant | `IsoThumpable` | `Thump`, `WeaponHit`, `animalHit`, `changeSprite`; no locomotion; tick it with `IsoCell:addToProcessIsoObject` |
| Visual only | `IsoObject` + sprites | `addAttachedAnimSprite` overlays; `WorldSoundManager.instance:addSound` for noise |
| Boss or swarm | several of the above | one controller, shared ModData pool; reassemble after chunk unload |

Whether Lua can drive the `Bullet` physics wrapper or `ModelManager.addStatic` directly is unknown; plan around the
engine's own hit and vehicle paths. `playerVoiceSound` is player-only: NPC barks are positioned `sound` entries,
and overhead speech text is `IsoGameCharacter:addLineChatElement(text, r, g, b ...)` (BeyondOneWeek uses it with
`getSpeakColour()` for debug lines).

### NPC frameworks in the wild
- **Bandits** (survivor NPCs on zombies):
  - identity is a boolean animation variable, read with `zombie:getVariableBoolean("Bandit")`, plus a brain table
    per bandit (`BanditBrain.Get(zombie)`) keyed by a stable id, and position-only "light" caches for the per-tick
    path;
  - verbs are 49 action modules in `shared/ZombieActions/` with `onStart`/`onWorking`/`onComplete(zombie, task)`
    (walking, climbing, aiming, shooting, reloading, looting, bandaging, fishing, generator work, burying ...);
  - programs in `shared/ZombiePrograms/` (`ZPBandit`, `ZPCompanion`, `ZPLooter`, `ZPThief` ...) are state machines
    that return `{status = ..., next = "State", tasks = {...}}`;
  - locomotion uses the player-style animation channel on the zombie: a `BanditWalkType` variable,
    `setBumpType("IdleToRun"|"IdleToWalk")`, `pathToLocationF` and `faceLocationF`, with the mod's own AnimSets
    and actiongroups;
  - in MP only the owning client steers (`BanditUtils.IsController(zombie)` gates the path calls), and
    `OnZombieUpdate` returns early on the server, so the brains run on clients and sync through commands.
- **CompanionDogs** builds a dog on the **raccoon**: `cd`-prefixed animNodes merged into
  `media/AnimSets/raccoon/`, a radial command menu (`ISRadialMenu`), status, stray-scan and bag windows.
- **ProjectALife** splits an NPC suite into subsystem folders, a shared `alife_runtime` catalog that several
  scenario mods build on, and a `Compat/` folder with one adapter per other mod.

## Build steps
1. Chassis (table above); write down which free systems you rely on.
2. Dressed zombie: register a body location in `common/media/registries.lua` and move it to the
   `UNDERWEAR_BOTTOM` index at `OnGameBoot`; skinned mesh in `models_X/<Monster>/`; clothingItem XML with all 16
   masks, a fixed GUID and a texture atlas, plus a corpse variant; GUIDs in `fileGuidTable.xml`; an invisible
   clothing item on your location; an outfit with only that item.
3. AnimSet nodes under `media/AnimSets/zombie/<state>/` keyed on `MyMod_Is<Monster>`; clips in
   `anims_X/<Monster>/` matching the `animationsMesh` entry.
4. A controller per monster behind one `OnZombieUpdate` dispatcher keyed on the persistent outfit; set the
   variables at claim time, drive attacks with `ZombieBiteDone` and a clip-select variable, clean up on identity
   change.
5. Spawn on the server only: `createRealZombieNow` + `dressInPersistentOutfit`, or `addZombiesInOutfit` when the
   population is on; sandbox toggles and rarities; debug right-click spawns for testing.
6. On `OnZombieCreate`, clear the stale outfit and re-dress before the zombie is seen.

## Verification
- Survey only: the mods' files were read end to end (structure, names, comments); none of them was run for this
  note, and no creature was built. The Java names were checked against the unofficial 42.21.0 javadocs
  (albion.codeberg.page/PZ-JavaDocs) at review time, and the Bandits identity check against a public copy of its
  42.20 Lua.
- For your own creature: `console.txt` after every session, the `-debug` animator window to watch which
  animNode plays, and debug spawn entries.
- Not verified: Cry of Fear's MP layer on a dedicated server; whether the persistent-outfit timing changed after
  42.21; any of the "other chassis"; Bandits in MP.

## Gotchas
1. **A monster's controller grabs the wrong zombie.** Cause: recycled zombies keep a stale outfit name.
   Fix: on `OnZombieCreate`, `getHumanVisual():setOutfit(nil)` when not `isPersistentOutfitInit()`, then
   re-dress from `getPersistentOutfitID()`.
2. **Duplicate spawns over time.** Cause: mod-data marks don't reliably survive chunk unload and reload. Fix:
   identify and count monsters by persistent outfit, not marks.
3. **A naked zombie playing the wrong sounds.** Cause: two controllers on one zombie. Fix: one dispatch point,
   previous-identity tracking, a cleanup pass on change.
4. **Looping monster sounds outlive the monster.** Fix: stop them in the identity-change cleanup, not only on
   death.
5. **`addZombiesInOutfit` does nothing.** Cause: zombie population set to None. Fix: server-side
   `createRealZombieNow` and set the properties by hand.
6. **Custom attack clips never play.** Cause: vanilla bites need the target within reach (about 0.72 tiles, CDDA
   note), a check Lua can't call. Fix: your own attack variables, a clip-select string variable, and set
   `ZombieBiteDone` yourself.
7. **The clothing body vanishes across saves or in MP.** Cause: a clothing XML without its `fileGuidTable.xml`
   entry. Fix: a unique, stable GUID per clothing XML.
8. **The monster turns back into a zombie after a reload.** Cause: the body item was lost while the zombie
   persisted. Fix: a corpse body variant, and the controller re-adds the body item while the identity holds.
9. **Vanilla hands or head poke through the monster.** Fix: all 16 `m_Masks`, `m_Static` false, no attach bone.
10. **The new AnimSet replaces movement instead of adding to it.** Cause: replacing the base set (the same
    failure as the animal-animset copy in CDDA gotcha 1). Fix: add nodes into the vanilla state folders, gated
    on variables.
11. **An animation file never loads, with no error.** Cause: the `anims_X` folder or file prefix doesn't match the
    `animationsMesh` entry (or the `Bob_`/`Kate_` prefixes for humans), or the clip name inside the file differs
    from `m_AnimName`. Fix: match all three.
12. **A recycled Bandits-style NPC keeps its flag.** Cause: animation variables stay on the zombie object. Fix:
    clear them in the same recycle cleanup as outfits.
13. **A friendly zombie still behaves like a zombie.** Cause: other zombies, melee auto-aim and player panic key
    on the zombie type. Fix: budget overrides, or use a mannequin or an animal.
14. **Publishing another game's assets gets the mod pulled.** Fix: keep such ports private, get permission, or
    generate your own assets.

## Assets
- Meshes in `models_X/<Monster>/`, texture atlases in `textures/` (`textureChoices`), clips in
  `anims_X/<Monster>/`, sounds in `sound/<mod>/` bound by `sound` blocks (category `Zombie`, 3D falloff, several
  `clip {}` variants per sound).
- Credit: Svide's *Cry of Fear Monsters* for the dressed-zombie technique (Cry of Fear's assets belong to Team
  Psykskallar); the Bandits, CompanionDogs and ProjectALife authors for the NPC architectures.

## Cost and time
Part of one research session on 2026-10-12.

## Open questions
- How much of Cry of Fear's net layer a listen-server co-op needs versus a dedicated server.
- A hybrid: an animal body with a zombie-hunter brain (the CDDA note shows hostile animals; zombie-vs-animal
  targeting crashes, see its gotcha 5).
- Whether a non-network `IsoPlayer` can be spawned as a puppet; the smallest workable AnimSet for a custom
  skeleton; `zombie.popman` hooks for horde-scale monster populations.


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

