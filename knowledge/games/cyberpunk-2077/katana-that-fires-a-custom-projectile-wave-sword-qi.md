---
kind: game
title: Katana that fires a custom projectile wave (sword qi)
game: "Cyberpunk 2077"
games_also: []
game_version: '2.31 (Cyberpunk2077.exe file version 3.0.80.51928)'
platform: windows
engine: redengine
route: loader-api
tools: [WolvenKit 9.0.1 (CLI + GUI), redscript-cli 0.5.31, Blender 4.5 LTS, TweakRecon (WolvenKit.RED4 reader), CET 1.37, TweakXL 1.11.4, ArchiveXL 1.26.8, RED4ext 1.30.0]
anti_cheat: none (single-player)
status: in-progress
agents:
- OpenCode (deepseek-flash)
humans: []
date: '2026-10-22'
links: ["https://github.com/wopss/RED4ext", "https://github.com/maximegmd/CyberEngineTweaks", "https://github.com/psiberx/cp2077-tweak-xl", "https://github.com/psiberx/cp2077-archive-xl", "https://github.com/jac3km4/redscript", "https://github.com/WolvenKit/WolvenKit"]
tags: [redscript, tweakdb, tweakxl, archivexl, projectile, melee-weapon, custom-entity, localization]
---
# Katana that fires a custom projectile wave (sword qi)

> Built a katana for Cyberpunk 2077 2.31 that fires a glowing, roll-controlled projectile wave (a custom ArchiveXL
> entity) on light-attack combos. Route: TweakDB attack-slot trick + redscript hooks + a custom projectile entity
> via ArchiveXL. Everything below was verified in the real game (items, swings, projectile flight, charge/glow
> state machine), with one acknowledged limitation (the iconic "gold" tooltip section remains engine-driven).

## Setup
- Game: Steam, Cyberpunk 2077 **2.31** (exe `3.0.80.51928`).
- RED4ext **1.30.0**; Cyber Engine Tweaks **1.37.0 [HEAD]**; TweakXL **1.11.4**; ArchiveXL **1.26.8**; redscript
  (scc) with CLI **0.5.31**.
- WolvenKit **9.0.1** (GUI + CLI for uncook/convert/pack); Blender **4.5 LTS** (headless scripting).
- A C# harness referencing `WolvenKit.Modkit.dll` to call `ModTools.ImportMesh` with an existing mesh's stream.
  WolvenKit 9.0.1 imports a glTF only over an existing `.mesh`; without one it logs "Direct mesh importing is not
  implemented", which is what the CLI printed here. Its `import --keep` (with the `.mesh` next to the `.glb`, or
  in `--outpath`) takes that same rebuild path (read from `WolvenKit.Modkit/RED4/Import.cs` and the CLI's
  `ImportCommand.cs`; not tried here).
- Optional offline recon: TweakRecon, a small private C# tool reading `r6/cache/tweakdb.bin` with `WolvenKit.RED4`
  types (records + flats).

## Route and why
- **Pure TweakDB was falsified first**: putting an `Attack_Projectile_Record` into a katana's melee attack slots
  never spawns anything — the melee state machine only runs melee attack classes; throwing projectiles are
  produced exclusively by the throw path/attack system.
- **Native hook not needed**: after learning the real mechanism (below), redscript + TweakDB + one ArchiveXL
  entity covers everything. Ghidra/CE were used during recon only.
- Mixed route therefore: TweakDB (item + attack-slot table + projectile params flats) + redscript (event hooks
  driving spawn + weapon state machine) + ArchiveXL (custom entity + entity factory + localization).

## How the game works (what we had to learn)
1. **Weapon attack table is positional for projectile capability.** A melee weapon's
   `gamedataWeaponItem_Record.attacks` array carries a fixed slot layout. The thrown-weapon projectiles live at
   index **[58..64]** (slot 58 = a "MeleeThrowAttack" melee record, 59-64 = per-variant
   `gamedataAttack_Projectile_Record`s). Putting *one* projectile record at index **[58]** on a pure katana
   enables its projectile-spawner machinery — appending the same record at the end of the array does nothing. This
   was proven by a bisection series (katana-table 61 vs 62 vs "throw block inserted at 58").
2. **Projectile spawn = `itemObj.QueueEvent(gameprojectileSpawnerLaunchEvent)`.** The event's `launchParams` carry
   the logical and visual `IPositionProvider`/`IOrientationProvider` handles; `templateName` (CName),
   `appearance`, `owner` and `projectileParams` are fields of the event itself (RED4ext.SDK's generated
   `SpawnerLaunchEvent.hpp`). There are separate **logical** (movement) and **visual** providers — set both or the
   visual orientation (roll) will not show.
3. **Template names resolve through the entity factory**: `base\gameplay\factories\items\projectiles.csv` is a
   `C2dArray` resource mapping name → `.ent` path (39 entries: knife/tanto/bullet/…). ArchiveXL can merge a mod's
   own CSV at runtime via a `.xl` config, so a custom entity name becomes spawnable.
4. **Projectile physics parameters are TweakDB flats**: `projectile.<tweakDBPath>.<param>` where `tweakDBPath` is
   a CName field on the projectile entity's `gameprojectileComponent` (vanilla knife = `knife_params`). Known
   keys: `startVelocity` (knife=110), `gravitySimulation` (-20), `energyLossFactorAfterCollision`,
   `deactivationDepth`, `waterImpulseRadius/Strength`. Give a custom entity its own `tweakDBPath` (e.g.
   `swordqi_params`) and define the flats in a TweakXL yaml — fully hot-reloadable.
5. **Projectile hit detection** defaults to a thin ray/point at the entity origin — the visual mesh size does NOT
   matter. `useSweepCollision=1` + `sweepCollisionRadius` (metres) enable a swept-sphere "tube" hit volume (sphere
   = orientation-agnostic; cross-section faces the flight direction).
6. **The thrown-knife behaviour is scripted**: `MeleeProjectile` (script component) sets up a
   `ThrowingMeleeCollisionEvaluator` (extends `gameprojectileScriptCollisionEvaluator`) in
   `OnProjectileInitialize`, plus a reload listener that "recovers" the thrown weapon. `EvaluateCollision` only
   *chooses* a `gameprojectileOnCollisionAction` (None/Stop/Bounce/StopAndStick/StopAndStickPerpendicular/Pierce)
   — the actual stop/embed/alignment is native C++. Returning `Pierce` on every collision removes
   insert/attach/recovery with no native code.
7. **Flight orientation**: the entity's forward = the flight direction; without intervention the engine may
   re-align the projectile (vanilla knife spins via `ToggleAxisRotation(true)` + `AddAxisRotation`).
   `LockOrientation(true)` + `ToggleAxisRotation(false)` freeze the spawn orientation — the visual roll then
   persists for the whole flight. The "convex side forward" look is achieved by a static **-90° X quaternion** on
   the mesh component (maps mesh +Z → entity +Y = forward).
8. **Melee event classes form two families** under `MeleeEventsTransition`: `MeleeAttackGenericEvents` (attacks —
   combo/final/strong/throw/…) and `MeleeRumblingEvents` (looping states —
   Idle/Block/Equipping/ChargedHold/Safe/PublicSafe). **A subclass that overrides `OnEnter` bypasses a
   `@wrapMethod` on the base class** — targeting the wrong class fails silently. Verified: RMB block =
   `MeleeBlockEvents`, weapon draw = `MeleeEquippingEvents`, sheathe/weapon-away =
   `MeleeSafeEvents`/`MeleePublicSafeEvents`, attacks = `MeleeComboAttackEvents`/`MeleeFinalAttackEvents`.
9. **The thermal blade** (`visualTags: [Thermal_Katana]` on the item = the lvl4 katana model + a red glowing
   overlay): the weapon entity carries `fx_decal_thermal` (a skinned mesh component using
   `engine\materials\metal_base.remt`, EmissiveEV 9, EmissiveColor red) and a separate `AreaLight`. Per-component
   `Toggle(bool)` + `IsEnabled()` give a cheap on/off "charged" state usable from redscript. There is no dynamic
   intensity (parameters are baked into the mesh material); the `AreaLight` is a separate component.
10. **Item display strings**: the hover name comes from the item record's **`displayName`** and the description
    from **`gameplayDescription`** (both `gamedataLocKeyWrapper`). `localizedName` is a dead CString field and
    `localizedDescription` does not drive the hover text — a common time-waster. Localization keys are merged by
    ArchiveXL's `localization: onscreens:` section (CR2W `JsonResource` with
    `localizationPersistenceOnScreenEntries`, `secondaryKey` + `femaleVariant`); in yaml a key can be written as
    `LocKey#My-Key`. Raw text in `localizedName`/`displayName` is treated as a *key lookup* → garbled names.
11. **The gold "iconic" tooltip section** shows the base weapon's iconic ability text and follows the engine's
    iconic-weapon UI logic: removing the `IconicWeapon` tag or changing `localizedDescription` did not change it.
    Accept it or replace the base item/ability entirely.

## Build steps
1. TweakXL yaml in `r6/tweaks/`: clone a katana (`$base: Items.Preset_Katana_Takemura`), set `friendlyName:
   w_melee_katana`, `displayName`/`gameplayDescription` locKeys, `visualTags: [Thermal_Katana]`,
   `projectileTemplateName: <factory name>`, and a full `attacks:` array (61 vanilla entries with one projectile
   record inserted at index 58 — use a vanilla `Attack_Projectile` record as a faithful field-for-field clone, or
   reference a vanilla one directly).
2. redscript in `r6/scripts/`: wrap `MeleeAttackGenericEvents.OnEnter` (read `GetStateName()` +
   `meleeAttackNumber`, filter to combo/final, read the weapon's `projectileTemplateName` flat via
   `GetNamePropertyFromWeaponDefinition(t".projectileTemplateName")`, get the right-hand slot item, build
   `gameprojectileSpawnerLaunchEvent` with static providers from `aimFwd` + `GetWorldPosition()`, `QueueEvent`).
   Wrap `MeleeProjectile.OnProjectileInitialize` to swap in a custom collision evaluator (scope by weapon TDBID),
   call `LockOrientation(true)` and `ToggleAxisRotation(false)`. Wrap
   `MeleeBlockEvents`/`MeleeEquippingEvents`/`MeleeSafeEvents` for the charge state machine.
3. ArchiveXL: a `swordqi_orb.ent` cloned from `knife_projectile.ent` with the mesh component's `mesh.DepotPath`
   pointing to an imported `.mesh`, `visualScale`, a -90°X mesh quaternion, `useSweepCollision=1`,
   `sweepCollisionRadius`, and `tweakDBPath = swordqi_params`; a `projectiles.csv` (`C2dArray`, columns
   name/path/preload) registering the name; localization `en-us.json`/`zh-cn.json` (JsonResource CR2W); the `.xl`
   listing `factories:` + `localization: onscreens:`.
4. Mesh: export/uncook the base mesh, edit in Blender, import with the Modkit `ImportMesh(glb, existingMeshStream,
   GltfImportArgs)` API (needs TANGENT data in the glb — Blender's exporter must include tangents; bake any node
   transforms into vertices first since the importer ignores them). Set the mesh's material by editing its
   embedded material instances (e.g. `engine\materials\metal_base.remt` + white textures +
   EmissiveColor/EmissiveEV) or via a `.Material.json` companion with `ImportMaterials`.
5. Pack: `WolvenKit.CLI pack <parent folder>` so in-archive paths keep the `<mod>\` prefix; deploy
   `archive\pc\mod\<mod>.archive` + `<mod>.archive.xl`; TweakDB yaml reloads with TweakXL hot reload (F10),
   archives and localization require a game restart, redscript requires a restart.

## Verification
- In-game: item appears, name/description show (bilingual), thermal blade lights on block and extinguishes on
  sheathe/extra block, light attacks fire the projectile, the wave keeps its roll during flight, passes through
  enemies/walls (`Pierce`), and disappears after its lifetime.
- Observability: redscript `FTLog` output lands in CET's `gamelog.log`
  (`bin\x64\plugins\cyber_engine_tweaks\gamelog.log`) — this was the main diagnostic channel (state names, spawn
  results, collision angles, charge transitions).
- Hot reload was validated by changing an observable flat and watching the log echo the new value.
- NOT verified: any online/multiplayer behaviour (none exists), long-save compatibility, other languages than
  en/zh, and the custom-model pipeline end-to-end with a finished art asset (only a test arc so far).

## Gotchas
1. **Yaml arrays silently do nothing.** Missing 2-space indentation on array lines makes TweakXL ignore them with
   no error. **Cause:** silent parse failure. **Fix:** keep indentation; verify by changing an observable flat and
   checking the log.
2. **A cloned record stayed a ghost and references to it dangled.** The version that worked spelled `$type` as the
   full native name, `gamedataAttack_Projectile_Record`. The short forms can't have been the cause, though:
   TweakXL 1.11.4 adds a missing `gamedata` prefix or `_Record` suffix itself
   (`TweakDBReflection::GetRecordFullName`), so `Attack_Projectile` and `Attack_Projectile_Record` name the same
   type. The real cause wasn't pinned down (Gotcha 4's field check is a candidate); the full name is still the
   unambiguous spelling.
3. **Attack-table slot is positional.** Appending a projectile record to `attacks` does nothing; it must be at
   index 58 (shifting the rest). **Fix:** rewrite the full array with the insert.
4. **Field values on clone records are validated by the engine scan.** A projectile record with "creative" fields
   (custom range/velocity/damageType, empty statModifiers) was rejected silently; an exact field-for-field clone
   of a vanilla projectile record worked.
5. **`@wrapMethod` on a base class does not intercept subclasses that override the method.** Block/equip/sheathe
   states live under `MeleeRumblingEvents`, not the attack family. **Fix:** wrap each concrete event class;
   identify it from the RTTI dump (`parent` field) and by logging `GetStateName()`.
6. **Packing a mod archive: pass the parent folder.** `WolvenKit.CLI pack <folder>` puts the folder's *contents*
   at the archive root — the `modname\` prefix is lost and factory/loc paths break. **Fix:** pack the parent so
   `modname\...` paths survive; remove `.json` sources from the build folder.
7. **`.xl` paths use single backslashes.** Unquoted YAML `mod\\path` is two literal backslashes; a broken `.xl`
   can also cascade into later init steps (a session where CET's Lua VM never started was traced to a bad `.xl`).
8. **Archives/localization load at startup, only yaml hot-reloads.** Deploy the archive while the game is closed
   (files are locked); restart for `.xl`/localization/redscript.
9. **Raw text in name/description fields is a localization-key lookup.** It displays garbled. **Fix:**
   `displayName`/`gameplayDescription` with `LocKey#...` + ArchiveXL `onscreens` entries
   (`secondaryKey`/`femaleVariant`), or let TweakXL generate keys only for real text fields.
10. **glTF import ignores node transforms and needs tangents.** The Modkit importer reads raw vertex data — bake
    scale/rotation into the mesh in Blender before export, and export with tangents (triangulate). **Fix:** apply
    transforms + add a `.Material.json` if you want materials.
11. **Blender glTF import spawns a 2 m "Icosphere" helper mesh** that can blacken renders/eclipse the model;
    delete objects whose name starts with `Icosphere` after import.
12. **Multilayer materials don't survive to glTF** — Blender shows a bare "Default" (metallic=1, roughness=1 =
    renders black). **Fix for editing:** set a grey matte material; final materials belong in the game's mesh.
13. **Thermal component visibility is re-applied by the appearance system during equipping.** Toggling it off in
    the `OnEnter` of the equip state gets overwritten. **Fix:** toggle in `MeleeEquippingEvents.OnExit` and again
    ~0.3 s later via a `DelayCallback`.
14. **The CET console's `TweakDB:GetFlat("Items.X.displayName")` can return nothing** even when the record field
    is set; use the in-game UI and FTLog as the oracle instead.
15. **The um CLI failed on a non-UTF-8 Windows locale** writing notes (`UnicodeEncodeError: 'gbk'`). **Cause:**
    this clone predated the fixes on main: `um` writes notes as UTF-8 since #8 (2026-10-04) and switches its
    console output to UTF-8 since #154 (2026-10-10). **Fix:** update the clone; on an older one, set
    `PYTHONUTF8=1` before running.

## Assets
- Test projectile: a procedurally generated arc (Blender headless script), imported into a vanilla katana-blade
  mesh so it inherits working material slots; then the mesh's embedded material instances were replaced with
  `metal_base.remt` + red EmissiveColor + EmissiveEV 3.
- Vanilla katana meshes were exported via WolvenKit for reference/editing in Blender (w_katana__common__grip1_04 =
  1.149 m, grip1_01 = 1.23 m).
- A finished custom sword-qi mesh is still to be authored; pipeline is proven with the test arc.
- The `.ent` cloned from `knife_projectile.ent`, the arc built inside a vanilla katana-blade mesh and the exported
  katana meshes are copies of game files, so they stay in your own build. A public release would ship only your
  own files (yaml, redscript, the arc's geometry) and rebuild the rest from the player's own install at install
  time.

## Cost and time
Multi-session project (about two weeks of iterative in-game testing); ~20 in-game restart cycles, heavy use of hot
reload for TweakDB, one C# tool build (mesh import harness), one C# recon tool (TweakDB reader).

## Open questions
- A production-quality projectile mesh + VFX (the current arc is a test asset).
- Replacing the iconic "gold" tooltip text requires an actual iconic ability record, not a text override — unclear
  whether a custom iconic effect can be registered cleanly in 2.31.
- Damage tuning via the projectile/attack record statModifiers (currently inherits vanilla throwing-knife
  modifiers).


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

