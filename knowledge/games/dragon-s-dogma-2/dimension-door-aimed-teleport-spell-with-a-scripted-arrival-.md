---
kind: game
title: 'Dimension Door: aimed teleport spell with a scripted arrival camera (REFramework Lua)'
game: Dragon's Dogma 2
games_also: []
game_version: dd2.exe 3.2.0.0, Steam 2054970 build 24831693; REFramework rev 0a07faae (v1.5.9.1+505)
platform: windows
engine: re-engine
route: loader-api
tools:
- REFramework (Lua)
- _ScriptCore (SilverEzredes and alphaZomega, raycast helper)
- il2cpp_dump.json (REFramework SDK dump) for symbol lookup
anti_cheat: none found by um scan; single-player only
status: released
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-05'
links:
- https://github.com/ntgoten/dimension-door
- knowledge/games/dragon-s-dogma-2/blink-strike-teleports-behind-the-target-reframework-lua-ski.md
tags:
- teleport
- camera
- cutscene
- animation
- input-hooks
- gamepad
- effects
- raycast
- fall-damage
---
# Dimension Door: aimed teleport spell with a scripted arrival camera (REFramework Lua)

> One REFramework Lua script (`reframework/autorun/DimensionDoor.lua`) adds a D&D-style Dimension Door to
> Dragon's Dogma 2. A key (B) or gamepad combo (hold the Vocation Action + give Go!) starts the Mage's casting animation
> (borrowed from the staff motlist) and casts a beam from the character's eyes to the camera aim, up to 500 ft.
> A second press opens a door of the Frost Boon shimmer next to the player and a visual one at the target.
> Walking through it plays a scripted arrival: the camera flies over and the character walks out of the far door
> toward the camera. Then the camera swings back behind the player. Tested in the real game by the human over
> ~65 iterations (v0.1 to 0.8.1), with a mod-written JSON debug log plus screenshots as the oracle.

## Setup
- Dragon's Dogma 2, Steam, Windows 11, dd2.exe 3.2.0.0 (Steam build 24831693).
- REFramework installed (`dinput8.dll`, revision `0a07faae`, menu shows v1.5.9.1+505).
- The ray/sphere casts `require("_SharedCore/Functions")`. That is **_ScriptCore** by SilverEzredes and
  alphaZomega; NickCore and UDD2P users already have it. Its `cast_ray(from, to, layer, mask, radius, options)` is
  used throughout.
- Symbols were looked up in REFramework's `il2cpp_dump.json`, using a small script that prints one type's members.
- Builds on the earlier note [Blink Behind](blink-strike-teleports-behind-the-target-reframework-lua-ski.md)
  (effects by ID, sounds, `play_efx` / `play_sound` helpers).

## Route and why
`loader-api`: a REFramework Lua script, with no game files changed. Everything visible is the game's own:
- Fangtooth effects and sounds (by EffectID and trigger ID);
- the Frost Boon shimmer from the held weapon's own effect list;
- the Mage casting animations, loaded as a dynamic motion bank.

Considered and dropped:
- `TimeSkipManager.requestPlayerWarp`: always fades to black.
- A custom glowing-panel mesh: not needed.
- A drawn laser overlay: the user found it jarring.

## How the game works (what we had to learn)
- **Teleporting a character: `app.Character.warp(via.vec3, via.Quaternion, app.CharacterWarpOption)`** with the
  static `app.CharacterWarpOption.ResetPosRotContext`.
  - Setting the transform directly (plus `CharaController.warp()`) gets undone. The character keeps a position
    history (`CharacterPosRotContext`) that restores the old spot: the body mesh arrives and the character and
    camera stay behind.
  - We call warp from a post-hook of `app.UserInputManager.updateInput` (the game's input processing step), then
    hold the position for 8 frames.
- **`TimeSkipManager.requestPlayerWarp(h, m, day, via.Position, quat, Action, bool, bool)`** is a full fast-travel
  warp: fade to black and a reload around the target. Its position is the **universal** `via.Position`
  (`transform:get_UniversalPosition()`), not `get_Position()`. Passing a local position sent the player
  somewhere else, into a low-LOD empty world.
- **Cameras after a long jump:** they keep looking at the old spot unless told.
  - Call `CameraManager.setCurrentCameraReset(ResetOption)`.
  - On every `app.CameraControllerBase` in the scene, set `IsFirstProcAfterWarp = true` and call
    `onWarpTarget()`.
  - `ResetOption` bits (low 32 bits of the dump defaults): DisableManualControl 1, ImmediateTerrainCorrection 2,
    ForceApply 4.
- **Overriding the camera for a fly-by:** `app.MainCameraController` (`_Role` 0 = main) positions the view through
  its **camera joint** (`_CameraJoint`) from its `_Position` / `_Rotation`.
  - Write your pose with `_CameraJoint:set_Position / set_Rotation` in a post-hook of
    `MainCameraController.lateUpdate`. The game writes the joint every frame, so control comes back by itself
    when you stop.
  - Writing the camera GameObject's transform instead froze the rendered view for good. The minimap and pawn
    name tags (which use the controller's pose) kept moving with the mouse, but the picture didn't.
  - Writing the controller's `_Position` before `lateUpdate` does nothing: the game recomputes it.
  - In the post-hook, `mainCam._Position / _Rotation` is the game's own pose for that frame. It's useful for
    blending the camera back.
- **Locking the player's input without the game's cutscene lock:** in the same `updateInput` post-hook, compare
  the `app.UserInput` argument (`args[3]`) with `player:get_Input()` **by `get_address()`**. Two Lua handles to the
  same object are not `==`. Then zero `ButtonOnFlags / ButtonTriggerFlags / ButtonReleaseFlags /
  ButtonRepeatFlags` and call `setAxisL / setAxisR`.
- **Scripted walk:** inject the left stick there too. It's relative to the game's own camera: x = right,
  y = forward, from `mainCam._Rotation`. A self-check flips the stick if the character moves away from the
  target.
- **Gamepad:** raw pad from `via.hid.GamePad.get_MergedDevice():get_Button()`. `via.hid.GamePadButton` values:
  - d-pad: LUp 1, LDown 2, LLeft 4, LRight 8;
  - shoulders: LTrigTop (L1) 256, RTrigTop (R1) 1024, LTrigBottom 512, RTrigBottom 2048.
- **The player's action flags:** `ButtonOnFlags / ButtonTriggerFlags / ButtonReleaseFlags / ButtonRepeatFlags`
  (UInt64) on the player's `app.UserInput` have **one bit per `app.CharacterInput.Action` value** (bit = 1 << index;
  take the low 32 bits of the dump's enum defaults).
  - Examples: AttackS 1, AttackL 2, Jump 3, Dash 4, Grab 5, Interact 6, Skill1-4 7-10, Shift1 14, Shift2 15.
  - The pawn commands are Come 17, Go 18, Help 19 and Wait 20.
  - JobSpecialAction 29 = the "Vocation Action". Other values: Sheathe 36, Draw 37, Lantern 38, RecoverHpItem 39,
    RecoverStaminaItem 40, Walk 60.
  - So a combo can follow the player's own button mapping: "Vocation Action held + Go! triggered" is R1 + d-pad up
    on default controls, wherever the player rebound it. Clearing bits 17-20 while the Vocation Action is held
    turns the d-pad into a free shortcut layer.
  - Raw pad as an alternative: `via.hid.GamePad.get_MergedDevice():get_Button()` (`via.hid.GamePadButton`:
    LUp 1, LDown 2, LLeft 4, LRight 8, LTrigTop/L1 256, RTrigTop/R1 1024). But physical buttons ignore the
    player's remapping.
- **Animations from another vocation:** the player's motlists are
  `animation/ch/ch00/motlist/ch00_00X_com|atk.motlist`.
  - 006 is the staff (Mage); 005 has a charge set too.
  - Load one as a dynamic bank: `sdk.create_resource("via.motion.MotionListResource", path)` →
    `create_holder` → `via.motion.DynamicMotionBank` with `set_OverwriteBankID(true)` and `set_BankID(id)` →
    `Motion:setDynamicMotionBank`. This is the same method as _ScriptCore and NickCore's `add_dynamic_motionbank`.
  - Then play a motion on layer 0 with `changeMotion(bank, motion, frame, interp, 1, 1)`.
  - Motion names can be listed at runtime: `Motion:getMotionInfo(bank, id, via.motion.MotionInfo)` →
    `MotionInfo.get_MotionName / get_MotionEndFrame`, scanning ids 0-3000.
  - The Mage's shared cast set in `ch00_006_atk`:
    - 700 `attackspelling_start` (60 f);
    - 701 / 702 `attackspelling_chant1/2` (230 / 180 f, loopable);
    - 750-753 `spellstock_release_*` (45 f);
    - per-spell `*_ready`, `*_ready_loop` and `*_activate` sets, e.g. firestorm 1021/1022, frostspike 1102-1104.
- **Keeping your motion on screen:** in some idle states the player's `via.motion.MotionFsm2` rewrites layer 0
  every frame. Pause it while the scripted animation plays and always unpause on end, cancel, error and script
  reset. It's a `via.behaviortree.BehaviorTree`, so `set_Paused(true / false)`.
- **Fall damage:** `app.Character.<FallInfo>k__BackingField` is an `app.FallInfoHolder` (`HighestPositionOnAir`,
  `BaseFallHeight`). Right after the teleport, call `resetBaseHeight(via.Position)` and
  `set_HighestPositionOnAir(via.Position)` with the arrival position, plus `resetFallHeight()`, for ~12 frames.
  The height you jumped from then doesn't count; later falls do.
- **Collision layers for casts** (via `_ScriptCore`'s `cast_ray`):
  - Layer 2 is open-world terrain, and it answers **sphere** casts but not thin rays. Town ground
    (`EnvRoot_TWN01_01`, layers 2/23/24) answers thin rays, but some spots don't answer spheres.
  - Layer 0 holds big invisible quest volumes (`Resource_qu...`) around the player.
  - Use `options = 1` (all contacts) and skip by GameObject name: `^Resource_`, `^ch%d` (characters), Area,
    Trigger, Sensor, Volume, Sound, Wwise, Event.
  - `_ScriptCore`'s plain-ray path *does* use the layer you pass. It shares the filter object with the sphere path.
- **Effects:**
  - The Frost Boon shimmer is `EffectID{-1, 0, 25}` on the **held weapon's** `ObjectEffectManager2`. Get it from
    `player._WeaponElementController:getWeapon(false):get_GameObject()`.
  - It loops until finished. It can be attached to a bare GameObject you create, so you can place it anywhere:
    `requestEffect(EffectID, GameObject, Int32, WwiseTriggerInfo)`.
  - Scaling that anchor object makes the effect invisible.
  - Fangtooth's spark `{14, 7, 11}` has its colour baked in. The only colour control (`EffectPlayer.Color`)
    drives a screen-wide layer, so recolouring it pulses the whole screen.
  - Requesting the same borrowed effect many times along a line showed only one instance (pooling, apparently).

## Build steps
1. Install REFramework and _ScriptCore.
2. Drop `DimensionDoor.lua` into `reframework/autorun/`. The source and release zip are at
   https://github.com/ntgoten/dimension-door.
3. In game: Insert → Script Generated UI → Dimension Door. Settings are saved to
   `reframework/data/DimensionDoor.json`, and a rolling log goes to `DimensionDoor_debug.json`.

## Verification
- **Oracle:** the human played every build. The script writes a 60-entry JSON event log (beam lock distance,
  hit / open air, landing corrections, cast start / skip, learned pad bits, errors). We read the log after each
  test, alongside the human's screenshots and descriptions.
- **Verified in game:**
  - aiming and locking the target;
  - casting animation, after the FSM pause;
  - door crossing and teleport (`Character.warp`);
  - camera fly-out and hand-back (joint write);
  - walk-out direction;
  - Vocation Action + Go! with pawn commands blocked (on a remapped pad);
  - doors floating at roof edges;
  - no fall damage after a downward jump.
- **Not verified:** carrying another character through the door; online / pawn-share side effects; every terrain
  type. The last tested build still got one user report of "probably good, not sure" on rocky ground.

## Gotchas
1. **Body arrived, but the camera and control stayed behind.** **Cause:** a direct transform set is undone by the
   character's position history (PosRotContext); only the mesh moved. **Fix:**
   `Character.warp(vec3, quat, CharacterWarpOption.ResetPosRotContext)` from the input-processing hook.
2. **The game warp blacked out and dropped us in an empty, low-LOD world.** **Cause:**
   `requestPlayerWarp` wants the universal position. **Fix:** add `UniversalPosition - Position` of the player,
   or don't use it for short hops (it always fades).
3. **After the teleport, the picture froze while the minimap and name tags still turned.** **Cause:** our fly-by
   wrote the camera GameObject's transform, but the view is driven by `MainCameraController._CameraJoint`.
   **Fix:** write the joint after `lateUpdate`. Also check your test switches: ours didn't really disable the
   flight, which wasted two rounds.
4. **The camera didn't follow a 20 m+ jump.** **Fix:** `setCurrentCameraReset(6)` plus `onWarpTarget()` and
   `IsFirstProcAfterWarp` on every `app.CameraControllerBase`.
5. **The game's CutScene input lock (`UserInputManager.requestDisable(1, 7)`) left the character broken.**
   **Fix:** do your own lock by zeroing the player's `UserInput` in the `updateInput` post-hook.
6. **The injected walk did nothing.** **Cause:** `curInput == player:get_Input()` is always false for two handles
   to the same object. **Fix:** compare `get_address()`.
7. **The walk-out went backwards, then forwards again later.** **Cause:** the stick's y axis is relative to
   whichever camera you take the rotation from. We inverted it when that was our overridden shot, then forgot
   once we switched to the game's camera. **Fix:** derive it from the game's own pose, plus a runtime flip check.
8. **The casting animation "started" but never showed: re-applied 66 times in 1 s.** **Cause:** `MotionFsm2`
   rewrites layer 0 every frame in fresh idle. It worked after walking a bit. **Fix:** `set_Paused(true)` on the
   player's `MotionFsm2` while casting, always unpaused afterwards.
9. **The marker or beam stopped at 0 ft / found "no ground", in town and outdoors.** **Causes:**
   - an overhead check that started 0.3 m above a slope hit the slope itself;
   - only one cast flavour was used (rays miss terrain, spheres miss some town ground);
   - `options = 0` stopped at the first contact, which was the quest volume we stood in.

   **Fix:** try both flavours on layers {2, 23, 24}, use `options = 1`, and skip volumes by name.
10. **Aiming outdoors always ended "500 ft, open air".** **Cause:** a rays-only optimisation; terrain doesn't
    answer rays. **Fix:** always run one sphere cast on layer 2 too.
11. **Under the map, twice.**
    - The thin ray went through a hill (terrain ignores rays) and hit something behind it. The sphere pass only
      ran when rays found nothing. **Fix:** always run both and take the nearer hit.
    - Safety net: a mid-air landing with any surface within 80 m above it is moved up onto that surface.
12. **Landed inside a rock.** **Cause:** the beam hit the rock's side, and the ground search found the terrain
    continuing *inside* the rock: casts that start inside a mesh don't see it. **Fix:** a landing must be reachable.
    There must be a clear line from the beam's open-air path to chest height, plus head room. Otherwise step
    back along the beam (0.6 / 1.5 / 3 / 5 / 8 m), or fizzle.
13. **The head was above ground, then the character sank through it after a long jump.** **Cause (likely):** the
    ground collision at the destination isn't loaded yet. **Fix:** for 2 s, if the character drops 0.6 m below
    the ground height measured at lock time, put them back on top.
14. **Fall damage after jumping into the door and arriving lower.** **Fix:** reset `FallInfoHolder` to the
    arrival position for a few frames.
15. **The door sank onto a lower roof / looked lifted in the air.** **Causes:** the ground search reached 4 m
    down; the shimmer plumes rise ~1 m above their spawn point. **Fix:** keep the door at feet height (snap
    within 0.5 m only), and shift the look by -0.4 m. A shorter frame looked bad.
16. **The target light blinded up close and was heavy; scaled effects vanished.** **Fix:** sparks alone are
    visible far away even at night. No light, no scaling.
17. **The recoloured target pulsed the whole screen red.** **Cause:** the spark's colour lives in its texture;
    `EffectPlayer.Color` drives a screen-wide layer. **Fix:** don't recolour that effect.
18. **Once you zero the player's input every frame (your own lock), a held button is only reported on its first
    frame.** Example: hold the Vocation Action, then press Go! during the lock; the combo never fires. **Fix:**
    track "held" yourself, from the On/Trigger bit until the Release bit (which is still reported), with a
    timeout as a safety net. While everything is blocked anyway, accept Go! alone.
19. **The RE Asset Library pak extractor found nothing**, not even a catalog mesh path, through
    `extractFilesFromPakCache` from Blender's Python. Unresolved. Listing motion names at runtime, through a
    dynamic bank and `getMotionInfo`, was simpler anyway.

## Assets
None made. All visuals and sounds are the game's own, used by ID:
- Fangtooth effects 14/7/10, 14/7/11 and 13/0/10;
- sounds 1717783091 and 2823464530;
- the Frost Boon shimmer -1/0/25 on the held weapon;
- Mage motions 700 / 701 / 750 from `ch00_006_atk.motlist`.

## Cost and time
One long session over two days, ~60 builds. The camera (gotchas 3-5) and ground detection (9-13) took the most
rounds.

## Open questions
- A solid glowing panel inside the door (the user's idea): needs either a flat emissive mesh or a sheet-like game
  effect.
- Whether carried characters reliably come along through `Character.warp`.
- Why the RE Asset Library pak cache lookup failed in a standalone run.
- A water-surface layer, for landing on water. A runtime layer probe was written but never got a result.


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

