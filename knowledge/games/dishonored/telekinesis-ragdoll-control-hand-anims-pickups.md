---
kind: game
title: "Telekinesis power for Dishonored (2012): controlled ragdolls, impact damage, hand animations, object pickup"
game: "Dishonored"
games_also: []
game_version: "Definitive Edition, Steam app 205100, 32-bit UE3 licensee; same globals as the ghost co-op note (GNames 0x01435674, GObjects 0x01423630)"
platform: windows
engine: unreal
route: native-hook
tools: ["dinput8.dll proxy (own)", "llvm-mingw 20261006 (i686)", "Python 3 (debug-port scripts)", "um win shot/drive/record"]
anti_cheat: "none (single-player); offline only"
status: working
agents: ["Claude Code (Opus 5.5)"]
humans: ["depa00"]
date: 2026-10-17
links: []
tags: [telekinesis, ragdoll, physx, impact-damage, first-person-animation, anim-tree, pickups, particles, sound, wwise, bend-time, hot-reload, ue3, reflection]
---

# Telekinesis power for Dishonored (2012): controlled ragdolls, impact damage, hand animations, object pickup

> A new power in Dishonored (Windows, Steam): hold Mouse4 on a guard or a physics object to lift it, move it
> with the view and the wheel, release to throw. Guards dangle from the torso, take damage from real impacts,
> knock other guards over and get back up if they survive; whale oil tanks explode on impact; pickups
> (grenades, elixirs, ammo, rewire tools) are pulled to the hand and picked up. The left hand plays the
> Possession cast pose while holding and the Windblast push on release; the Possession smoke wraps the held
> object; the Blink charge sound plays on grab, the Windblast gust and a sword whoosh on throw. Native
> in-process DLL (dinput8 proxy) driving the engine through SDK-free UE3 reflection. Verified in the real game
> with debug-port numbers, per-frame logs, screenshots and scripted stress tests.

## Setup
- Dishonored Definitive Edition, Steam app 205100, Windows 11, launched with
  `-windowed -resx=1280 -resy=720 -nostartupmovies` (via `steam -applaunch 205100 ...` so the arguments
  actually reach the game).
- Toolchain: llvm-mingw 20261006 (`i686-w64-mingw32-clang++`), C++20, `-static`, `-Wl,--kill-at`.
- Two DLLs in `Binaries/Win32` (new files only, no game file modified):
  - `dinput8.dll`: proxy + host. Hooks `DishonoredViewportClient.PostRender`, serves a localhost debug port,
    filters the DirectInput mouse wheel, and hot-reloads the logic DLL when it changes on disk (it loads a
    private copy so the build can overwrite the original while the game runs).
  - `DisPowersLogic.dll`: all gameplay (powers, overlay, debug commands).
- Reflection layout, GNames/GObjects discovery and the PostRender `Func` swap are exactly as in
  `knowledge/games/dishonored/ghost-co-op-ue3-reflection-postrender-hook.md`; everything there held for this
  install. That note ran buildid 8335061 under Proton, this one native Windows 11; `Dishonored.exe` has no
  ASLR, so the identical GNames/GObjects addresses point to the same exe build.

## Route and why
- Native in-process DLL, same reasons as the ghost co-op note (cooked packages, no script compiler route).
- Everything runs from the PostRender hook (render time). No per-tick script hook was found (see Gotchas),
  so the design works around render-time limits instead of fighting for a tick hook.
- Hot reload of the logic DLL was decisive: dozens of rebuilds without restarting the game.
- **Related:** the ghost co-op note above (same host/logic DLL split and debug port), and its open question
  about a script event inside the world tick, which Gotcha 4 here probes.

## How the game works (what we had to learn)
**Calling functions.** ProcessEvent with a parameter block filled by property name works for script and
native functions. Subclasses often override with *renamed* parameters (e.g. `DishonoredEmitterPool.SpawnEmitter`
takes `_pEmitterTemplate, _SpawnLocation, _SpawnRotation, _pAttachToActor`; `DisAnimNodeSeqPlayer.PlayAnim`
takes `_bLoop, _fInRate, _fStartTime`; `DishonoredNPCPawn.SetPushesRigidBodies(_bNewPush)`). Always list the
functions of the actual object, not the base class.

**What fails from PostRender.** `Actor.Trace`/`FastTrace` always report nothing/blocked (even 10 uu in empty
space). Not a render-time problem: they fail the same way from inside `GameInfo.Timer` (world tick, see Open
questions), while `Actor.TraceComponent` against one given component works. `PC.Use()` does nothing.
`SetPhysics(PHYS_RigidBody)` is ignored on resting pickups. `ForceUpdateComponents(bCollisionUpdate=true)`,
`SetRBCollisionChannels(Channels)` and `SpawnEmitterMeshAttachment` on a ragdoll bone hit a fatal assert.
(The ghost co-op note calls `ForceUpdateComponents` from PostRender to move its puppet without trouble;
UE3's default is `bCollisionUpdate=false`.) These work: `InitRagdoll`/`TermRagdoll`,
`SetRBLinearVelocity`, `AddImpulse(bVelChange)` on a bone, `SetRBPosition`, `GetBoneLocation`,
`RB_BodyInstance.GetUnrealWorldVelocity`, `SetRBCollidesWithChannel`, `TakeDamage`, `SpawnEmitter`,
particle `SetTranslation`/`SetScale`/`DeactivateSystem`/`KillParticlesForced`, anim `SetAnim`/`PlayAnim`,
`PC.Dis_EquipItemByType`.

**Time (Bend Time).** `WorldInfo.TimeDilation` is ignored. `DishonoredGameInfo` recomputes
`m_fCurrentWorldTimeDilation` / `m_fCurrentPlayerTimeDilation` every frame from `m_BendTimeInfo`
(11 x `BendTimeChannelInfo`, 0x40 bytes: world current/target at +0x10/+0x14, player at +0x1C/+0x20).
Writing one channel slows the world and not the player; channel 9 is not saved (0 and 7 are). The world is
slowed per actor: the emitter pool actor itself gets `CustomTimeDilation` 0.3. A
`DisParticleSystemComponent` follows the time of its `m_pTimeBoundActor`: binding our emitters to the player
pawn keeps them at real speed (measured `AccumTickTime` rate 1.1 vs 0.3 for a level emitter).
`DishonoredEngine.bPauseOnLossOfFocus` and `m_bRequestOpenPauseMenu_FocusLost` can be cleared every frame.

**Targeting without traces.** Walk `WorldInfo.PawnList` / `Pawn.NextPawn` (NPCs) plus a once-a-second list of
`KActor`s from GObjects, and pick the one closest to the view direction inside a ~10 degree cone (no line of
sight).

**Ragdolls of living NPCs.**
- After `InitRagdoll` a living NPC's mesh stays in its pawn RB channel (54) and collides only with
  `DisPawnChannel*`: it flies through walls. A death ragdoll made by the game uses `RBChannel` 16 (DeadPawn)
  and `RBCollideWithChannels` = 0x00050221 (Default, GameplayPhysics, SeveredLimbs, DeadPawn,
  ClothingCollision: bits 0, 5, 9, 16 and 18, the stock UE3 `ERBCollisionChannel` positions, where bit 9 is
  `RBCC_Untitled3`). We set that channel and write that raw mask; `SetRBCollidesWithChannel(Default, true)`
  then pushes the whole container to PhysX (`SetRBCollidesWithChannel(20..26)` does not reach the
  DisPawnChannel bits).
- Do NOT add `BlockingVolume`: the level's invisible player-blocking walls stop thrown bodies mid-air.
- A living pawn has a kinematic push body (`PhysicsPushBody` = the `BounceBox` body,
  `bPushesRigidBodies`); death clears both. `SetPushesRigidBodies(false)` removes it.
- Ragdolls we create are never ended by the game: survivors stay on the ground forever unless the mod
  calls `TermRagdoll` itself.
- `TermRagdoll` puts the pawn back in its standing animated pose around the pelvis. Near the floor the legs
  are then inside the floor, and a ragdoll re-created from that pose (or the standing pawn) falls through
  the world (health ends near -1000). Lift first: after re-creating, move the new bodies up with
  `SetRBPosition` before the next physics step; for a get-up write `Location.z += 90` and
  `PHYS_Falling` (only the Location field: no component update calls from render time).
- City guard ragdoll bodies: Root_jnt, spine_1_jnt, spine_3_jnt, neck_jnt, arms, legs, feet, Back_pan_*;
  no Torso_jnt body. Body lookup: `PhysicsAsset.BodySetup[i].BoneName` -> `PhysicsAssetInstance.Bodies[i]`.

**The ~10 m freeze (cause NOT found).** Ragdolls we create stop dead when they get ~10-11 m from the player
sideways or ~7.7 m up, and hang there forever (re-grabbing them and letting go returns them to the same
spot). Ruled out: world collision (persists with every channel off and `BlockRigidBody` off), BlockingVolume,
channel mask, push body, Bend Time, particles, our own tracking code, `m_bFrozen`/bone springs (all off on
the hanging bodies). `TermRagdoll` + `InitRagdoll` (plus collisions) at the hang point frees it, and it then
flies on past 25 m. Re-creating it at grab or at throw time does not help. Workaround in use: detect a body
that was moving fast and is stopped in the 850-1450 uu band from the player, re-create it there and give
back its pre-stop velocity. It works most of the time but some re-created ragdolls still fly off oddly.

**Driving a held ragdoll.** Spring-damper on a velocity state (K 140, D 15, max 6000 uu/s) toward
`camera + view * distance (+40 uu up for guards)`. With bone drive, only the torso body gets
`AddImpulse(target - measured, bVelChange=true)`, so arms and legs dangle (visible in screenshots: legs trail
when swinging the view). Physics answers commands 2-3 frames late.

**Impact damage.** Per-frame velocity from Location deltas is noise (physics steps are not render frames:
1300 -> 9400 uu/s on consecutive frames of a smooth pull). Use velocity over a ~60 ms window. A wall hit
spreads over ~6 frames of that windowed speed (4155 -> 3767 -> 3235 -> 2601 -> 2020 -> 1544), so look for
speed below 60 % of the max of the last 150 ms with a drop over 900 uu/s, collect for 100 ms, apply one
`TakeDamage(DisDamageType_RagdollImpact)` with damage (drop - 700) / 30. While held, compare against the
spring's command (shifted ~30 ms) instead, and require the body to have really been moving (a guard pinned
against a wall while the spring winds up is not a hit). Full throws into a wall: 128-144 damage.

**First-person left-hand animations.** Player power anims live in `Ply_Powers_as` (object names are generic,
the real name is `AnimSequence.SequenceName`): `Powers_Cast_Blink_In/Loop/Out`,
`Powers_Cast_Possession_In/Loop/Out`, `Powers_Cast_Windblast`, ... The player tree has no AnimNodeSlot. Its
`PlayerStack` (ArkAnimNodeStack) has four overlay channels `PlayerSeq1..4` (DishonoredAnimNodeSeq) that
override the whole body. The node `LeftArmFilter` (DisAnimNodeBlendPerBone) takes the left arm from its
`Target` input. Rewiring `LeftArmFilter.Children[1].Anim` to `PlayerSeq1` playing a sequence, and putting the
original pointer back afterwards, animates only the left arm in any state. Find nodes with
`AnimTree.FindAnimNode(name)`. Set `bNoNotifies` on that channel: the Possession anims start a looping sound
from a notify that only `Possession_Out` stops.

**Freeing the left hand.** Real inventory = `pawn.m_pInventory` (the `pInventory` subobject is a different
object). Left-hand slot = `m_EquipUsageInfo[2].m_iEquippedSlot` (`EDisEquipUsage`: 0 None, 1 Primary,
2 Secondary); items in `m_Slots[i].m_pItem`. Swap with `PC.Dis_EquipItemByType(itemClass, 2)` (usage 0 gives
inconsistent results). The swap has its own arm animation: rewiring the arm before it finishes left the
crossbow attached to the hand.

**Objects.** Whale oil tank = `DisWhaleOilBattery` (KActor family, PHYS_RigidBody); our impact damage makes
it explode by itself. Placed pickups (`DisElixirHealth`, `DisElixirMana`, `DisStatPickup`,
`DisAbstractItemPickup`, `DishonoredInventoryPickup`) rest with PHYS_None and a fixed body: write
`Physics = 10` (PHYS_RigidBody), call `StaticMeshComponent.BodyInstance.SetFixed(false)` and
`WakeRigidBody`. They implement the native `IDisInteractableInterface` (no script functions): the game's own
prompt ("[F] Pick up") appears when the held object sits exactly under the crosshair, and the real use key
picks it up while held.

**Effects borrowed from powers** (already loaded in missions): `Vfx_GamePlay.Possession.PossesTarget_Human_01`
on the held object (scaled to its `Bounds.SphereRadius` / 90), `Possession_HandCast_01` at the hand (it loops:
stop it), `Windblast_lvl1` on throw, `Windblast_GroundImpact_01` on hard impacts. Take the component from
`SpawnEmitter`'s return value (with the renamed parameters).

**Sounds (Wwise).** Dishonored plays sounds through Wwise: `AkEvent` objects (e.g.
`Snd_Power_Player.Snd_Power_P_Blink_Start`) posted on an actor with the native `Actor.PostAkEvent(InAkEvent)`,
which works from PostRender. There is no generic stop call: looping events need their own paired stop event,
so for a power prefer one-shot events and check each one. We play `Snd_Power_P_Blink_Start` (the Blink
charge, one-shot ~3 s) on grab, and on throw `Snd_Pwr_P_WindBlast_lvl_1` on the player plus
`Snd_Weapon.Snd_Weapon_Sword_Whooshe_ak` on the thrown actor (both one-shot). `Snd_Power_Possession_In_Lvl1`
(one-shot ~2 s) also works but felt too repetitive on every grab. Find events with a name search over
GObjects (`AkEvent` class).

**Mouse wheel.** The game reads the mouse through DirectInput, so a low-level Windows hook does not stop the
weapon wheel. Patching the vtables of `IDirectInput8::CreateDevice` and, for `GUID_SysMouse`,
`GetDeviceState` (zero `lZ`) / `GetDeviceData` (drop `DIMOFS_Z` records) while the power is held does.

## Build steps
1. Build both DLLs with llvm-mingw (host: `-lws2_32`; logic: `-lshlwapi`), `-static -Wl,--kill-at`.
2. Copy `dinput8.dll` (game closed) and `DisPowersLogic.dll` into `Binaries/Win32`. Later logic builds can be
   copied while the game runs; the host reloads them within a second.
3. Launch with `steam -applaunch 205100 -windowed -resx=1280 -resy=720 -nostartupmovies`.
4. Press a key on the title, Continue, press a key on the loading screen.

## Verification
- Debug port on 127.0.0.1 (find/props/funcs/call/set, struct and array dumps, per-frame watch logs) to read
  engine state and call functions while the game runs; helper scripts for grab/throw trajectories.
- Screenshots of every visual change (dangling guard, hand poses, crossbow restored, FX scale, tank
  explosion) checked one by one.
- Scripted stress test: 3 rounds x 5 throws in different directions + held wheel-out + get-ups, no crash.
- Sounds: game-only audio captured with `um win record` (process loopback) while posting each event from the
  debug port, then loudness / band energy per 125-250 ms to see when a sound starts and whether it ends.
- Crashes diagnosed from the game's minidumps (module list + exception address -> symbol of the logic DLL).
- Not verified: other NPC types than city guards (bone names may differ), rats (crowd agents, not pawns),
  other gadgets than the crossbow for the hand swap, controllers.

## Gotchas
1. **Symptom:** F6 slow motion changes nothing. **Cause:** `WorldInfo.TimeDilation` is ignored.
   **Fix:** write a `m_BendTimeInfo` channel (9 is not saved).
2. **Symptom:** traces never hit anything. **Cause:** not found. `Trace`/`FastTrace` called through
   ProcessEvent fail in every context tried (PostRender and `GameInfo.Timer`), while `TraceComponent` hits
   through the same path (but see the native-index lead in Open questions). **Fix:** cone targeting over
   `PawnList` / KActors.
3. **Symptom:** native call "returns nothing". **Cause:** a subclass override with renamed parameters; unknown
   names are silently ignored. **Fix:** dump the object's own functions and fill both names.
4. **Symptom:** `GameViewportClient.Tick` hook never fires; a probe of 659 script functions on the player
   objects saw only `HUD.PostRender` and `GameInfo.Timer`. **Cause:** gameplay is native. **Fix:** stay in
   PostRender. `GameInfo.Timer` runs only once a second, and traces fail there too (Open questions).
5. **Symptom:** thrown guard flies through walls. **Cause:** living-pawn RB channel/mask. **Fix:** DeadPawn
   channel + raw mask 0x00050221.
6. **Symptom:** thrown guard stops mid-air against a building front. **Cause:** BlockingVolume channel.
   **Fix:** never enable it.
7. **Symptom:** thrown/held guards freeze ~10 m out and hang. **Cause:** unknown (see above). **Fix
   (workaround):** re-create the ragdoll at the hang point and restore its velocity.
8. **Symptom:** guards fall under the map after a re-created ragdoll or a get-up. **Cause:** standing pose
   legs inside the floor. **Fix:** lift with `SetRBPosition` / Location before the next step.
9. **Symptom:** `Fatal error!` on grab. **Cause:** `SpawnEmitterMeshAttachment` on a ragdoll bone from render
   time. **Fix:** pooled emitter moved with `SetTranslation`.
10. **Symptom:** `Fatal error!` during an unstick while held. **Cause:** cached `RB_BodyInstance` of the old
    ragdoll. **Fix:** cache keyed on the current `PhysicsAssetInstance` and non-null `BodyData`.
11. **Symptom:** access violation in our frame code. **Cause:** a destroyed actor's memory was freed.
    **Fix:** check the pointer is readable before reading `UObject::Index`.
12. **Symptom:** `Fatal error!` lifting an actor. **Cause:** `ForceUpdateComponents(bCollisionUpdate)` from
    render time. **Fix:** write Location only.
13. **Symptom:** `Fatal error!` setting collision channels. **Cause:** `SetRBCollisionChannels`. **Fix:**
    write the container and call `SetRBCollidesWithChannel(Default, true)`.
14. **Symptom:** held guard "dies" without hitting anything. **Cause:** false impacts from per-frame noise
    and command lag. **Fix:** windowed velocity, lag-shifted command, require a real measured peak.
15. **Symptom:** thrown guards take no damage. **Cause:** confirming a hit against an expectation derived from
    the already-collapsed measurement. **Fix:** compare with the max of the last 150 ms.
16. **Symptom:** wheel still switches weapons. **Cause:** DirectInput, not window messages. **Fix:** filter
    `GetDeviceState`/`GetDeviceData` in the dinput8 proxy.
17. **Symptom:** whole body animates (crossbow arm too). **Cause:** stack channel overrides all bones.
    **Fix:** rewire `LeftArmFilter`'s Target input.
18. **Symptom:** crossbow stays glued to the open hand. **Cause:** rewired before the equip swap finished.
    **Fix:** wait for the slot change + ~0.6 s.
19. **Symptom:** endless sound after throwing. **Cause:** Possession loop sound from an anim notify.
    **Fix:** `bNoNotifies` on the channel.
20. **Symptom:** dark globs stay where things were grabbed. **Cause:** `Possession_HandCast_01` loops.
    **Fix:** stop it (and kill pooled leftovers) on release.
21. **Symptom:** held pickups can't be picked up. **Cause:** held above the crosshair. **Fix:** no lift for
    objects; auto-press the use key when pulled close (key held ~80 ms; a same-instant down/up is missed).
22. **Symptom:** elixirs/ammo can't be grabbed. **Cause:** PHYS_None + fixed body. **Fix:** `Physics = 10`
    (PHYS_RigidBody) + `BodyInstance.SetFixed(false)`.
23. **Symptom:** test results made no sense. **Cause:** the human was playing at the same time.
    **Fix:** ask for hands-off windows during scripted tests.
24. **Symptom:** effects "cast from the left hand" appear on the right, by the sword. **Cause:** UE3 is
    left-handed (X forward, Y right, Z up), so `view x up` points to the LEFT; we subtracted it as if it
    pointed right. **Fix:** add it (or use `up x view` for right).
25. **Symptom:** the game-audio capture is pure silence. **Cause:** the game mutes when its window is not in
    the foreground, and on the Game Over screen (a guard alerted by the tests killed the player). **Fix:**
    focus the window right before recording (ask the human first, and not while they're typing) and keep the
    player safe; check a screenshot when a capture is flat zero.

## Assets
None created. All effects and animations are the game's own, used at runtime; nothing extracted or shipped.

## Cost and time
One long working day (about 9 hours), roughly 60 hot reloads and 8 game crashes along the way.

## Open questions
- The real cause of the ~10 m freeze of script-created ragdolls (a native system tied to living NPCs or to
  ragdolls not started by the death path?). Finding it would remove the unstick workaround.
- A natural get-up animation instead of `TermRagdoll`.
- A per-tick hook for physics work. `GameInfo.Timer` was tried (2026-10-17, same build): it fires exactly
  once a second (15 calls in 15 s), and with its `Func` swapped, `Actor.Trace` from inside it still returns
  None with HitLocation (0,0,0), the same as from PostRender. So traces do not fail because of render time.
  The same ProcessEvent path works for `Actor.TraceComponent`: against the player's CollisionCylinder it hits
  (z 4099.5, normal 0,0,1) and returns `true`. World line checks fail for every variant tried: `Trace`
  returns None with `ExtraTraceFlags` 0-15, with a box extent, with bTraceActors 0 and 1, and called on the
  pawn, the PC or WorldInfo, and `FastTrace` returns false ("blocked") even for 50 uu of empty air 5 km above
  the map. Either Dishonored routes world queries some other way, or the call itself goes wrong (next
  item). `SetLocation` from the Timer was not tested.
- A lead on the traces, not tried: every call that fails quietly here and in the ghost co-op note is a native
  with a fixed index in UDK's `Actor.uc`: `Trace` (277), `FastTrace` (548), `SetLocation` (267),
  `SetRotation` (299), `Move` (266), `Destroy` (279) and `SetPhysics` (3970). The natives that work have no
  index: `TraceComponent`, `Spawn`, `SetHidden`, `ConsoleCommand`, `InitRagdoll`, `SetRBPosition`,
  `AddImpulse`. Public UE3 SDK generators special-case such functions when calling them through
  ProcessEvent: TFL's generator sets `UFunction::iNative` to 0 for the call and restores it after, and
  BmSDK's generated wrappers also clear `FUNC_Native`. Check `iNative` on these functions in this build, and
  try the same around one `Trace` call.
- Other NPC skeletons, rats (crowd agents), other gadgets.


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

