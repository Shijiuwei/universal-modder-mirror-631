---
kind: technique
title: "Custom races and creatures in Bannerlord at runtime: the humanoid path, quadruped mounts and hidden-rider creatures"
status: in-progress
game: "Mount & Blade II: Bannerlord"
games_also: ["Borderlands 2"]
game_version: "v1.4.8.119303 (Steam, Windows 11); client TaleWorlds.Native.dll 14,185,944 bytes, PE timestamp 0x6a732505 (all crash offsets here are for this build); Bannerlord.Harmony 2.4.2"
platform: windows
engine: unknown
route: loader-api
tools: ["Bannerlord module system (ModuleData XML + XSLT, C# net472 SubModule)", "Bannerlord.Harmony 2.4.2", "Python 3.10 + numpy (own .tpac/.rdc writers)", "Blender 4.2 headless", "Ghidra 12.1.4 headless", "ilspycmd 9.1"]
agents: ["Claude Code (Opus 5.5)", "Claude Code subagents (Sonnet 5.5)"]
humans: ["@theartur2000"]
date: 2026-10-07
links: ["https://github.com/theartur2000/bannerlord-editor-free-sdk/tree/v1.4.8.119303"]
tags: [bannerlord, custom-race, custom-skeleton, animation, action-sets, monsters, quadruped, mounts, creatures, jockey-pair, flying, humanoid-path]
---

# Custom races and creatures in Bannerlord at runtime: the humanoid path, quadruped mounts and hidden-rider creatures

> What it takes to make Bannerlord accept non-human bodies that move, fight and ride: races on their own skeletons (the
> humanoid path), creatures as mounts (the quadruped path), why a creature cannot be a soldier on its own and the hidden-rider
> workaround, flying creatures, and the animation rules that decide whether any of it moves. Learned while porting Borderlands 2
> ([`../games/mount-and-blade-ii-bannerlord/bannerlord-borderlands-2-total-conversion.md`](../games/mount-and-blade-ii-bannerlord/bannerlord-borderlands-2-total-conversion.md));
> binary formats are in [`editor-free-bannerlord-assets.md`](editor-free-bannerlord-assets.md).

Evidence tags: **[V]** seen in the running game or by a byte-exact round trip, **[M]** measured, **[S]** static analysis only,
**[H]** hypothesis. "TN n" = gotcha n of the asset note. Offsets are search handles for the client build above.

## When to use it
- You want a non-human body in Bannerlord that walks, fights or can be ridden: a monster, a robot, an animal, a boss.
- Your custom race spawns but slides, T-poses, never hits, or crashes seconds after it appears.
- You want a creature as a mount or as a fighter on its own, or something that flies (agents cannot).

## How
**Races and animation**
- A **race** is a skin entry (skeleton, body/face/hair meshes, deform keys) mapped to a `Monster` (bone fields, capsules, flags)
  whose action set, found by the name `as_<monster>[_female]<suffix>`, names the skeleton and the clip for every action. The agent's
  VISUAL skeleton comes from the ACTION SET's `skeleton=`, not the skin's; UI paths that pass no action set (character-creation
  preview, `BasicCharacterTableau`) use set 0, the human set, which twists a custom body and reads past the human bone array.
- **Humanoid path** (everything that fights as a troop): a copy of the human race with `skeleton=<own>`, `body_meta_mesh=<own mesh>`,
  every other body slot a one-triangle "none" mesh, `uses_stitching="false"`; hair/beard entries nameless but the ENTRY COUNTS kept;
  empty `<deform_keys />`; a placeholder face mesh with a submesh; non-empty colour lists. A monster with ALL ~55 bone fields named
  (omitted ones inherit the BASE monster's INDEX), plus `<id>_settlement`, `_settlement_slow`, `_settlement_fast`, `_child` variants;
  keep `IsHumanoid` (AI, morale, deployment read it). An action set per suffix the code asks for (`_warrior`, `_facegen`, `_map`,
  `_poses`, `_villager`, `_lord`, tavern and carry sets: **88 `as_human_*` sets, 7,649 actions**); `MBGlobals.GetActionSet` THROWS when
  one is missing (about 35 call sites of `FaceGen.GetMonsterWithSuffix`). A clip must exist for EVERY action type of the human set
  or the engine plays nothing; races copy all human sets and re-point only actions that have a clip.
- **28 biped bone order:** see "What the engine assumes about a human" in the asset note (first 28 bones in human biped order,
  extras after 27). Hero rigs of 90 to 122 bones are 27 body + face + finger bones; dropping face and fingers leaves 28 to 36.
- **Movement.** `movement_sets.xml` has 18 slot actions (idle, forward, backward, strafes, rotate, adders); the engine caches
  per-direction speed = distance per cycle / duration. Native locomotion clips are authored IN PLACE; speed comes from the agent and the
  clip's `bip_mov_ik` distance block, not the root channel. Recipe: one FBX per movement sequence, cycles in place, `bip_mov_ik` =
  metres per cycle measured from the planted foot.
- **Key units:** see "Clip record" in the asset note. Measure the key rate per rig, or hit windows land on the wind-up instead of the
  contact frame (gotcha 3).
- **Layers.** Channel 0 is movement, channel 1 the upper body (ready, aim, release, reload, defend); the skeleton's per-bone
  `lowerbody` flag decides which bones channel 0 keeps. Never set `enforce_*` flags on ready, release, reload or defend clips.
- **Combat clips** are written in copy mode from the Native clip of the same action. The hit window is `combat_parameters.xml`
  `collision_check_starting_percent` .. `ending_percent` of clip progress (parser default start 1.0 = empty window, so a plain clip never
  hits; e.g. 1h overswing 0.38 to 0.50). Ranged releases need step points (launch: bow 0.0, javelin 0.25); equip/unequip use the 4th
  step point as the switch progress and `agent_wield_update` needs channel 1 to play an equip/unequip action (type 33) until then or
  it logs "Weapon wield interrupted". Ready clips keep the `keep` flag. Keep Native durations to keep AI timing.
- **Hit geometry.** A chain of segments from shoulder to weapon bone, then along the weapon. A hit by an arm segment is
  `HitWithAnotherBone` (bounced, damage multiplied by `SwingHitWithArmDamageMultiplier`). Blow magnitude uses `AttackProgress`
  through `SpeedGraphFunction` (`managed_core_parameters.xml`) times weapon speed; bone velocity is not used.
- **Prebaked animations:** custom clips have none and off-screen agents' skeletons are not ticked, so
  `AgentVisuals.GetBoneEntitialFrame` returns stale or NaN frames: call `Skeleton.ForceUpdateBoneFrames` first.
- **Loop flags.** Conversation, inventory and pose "start" clips run once and rely on `continue_with` and the loop clip's `cyclic`
  flag; battles hide a missing one because movement re-issues idle.
- **Face builder:** see "Skin and face" in the asset note and TN 13. In addition, hair and beard lists are bound-checked (face,
  mouth, eyebrow and tattoo lists are not); put the fixed head in `body_meta_mesh`. `FaceGenVM` throws in randomise for a female skin
  with 0 beards.
- Tracks carry rotations per bone plus root translation only: non-root bone translations never animate; design rigs around rotations.
- **Retargeting a foreign body onto Bannerlord joints:** fit each bone with an affine, move vertices by linear blend skinning. BL2 is
  right-handed, forward +X, up +Z; Bannerlord agents face +Y, left at -X. Native clips cannot drive a foreign skeleton (bind rotations
  differ by a median 7 to 28 degrees).

**Creatures**
- A **quadruped path** exists and is safe only for mounts: an action set with `movement_system="quadrupedal"` and its own `skeleton=`.
  Rider and mount poses come from `monster_usage_sets.xml` (Native: human, horse, camel, animals). All monsters on one action set must
  share `num_paces` and `monster_usage`; the pace table is built once PER ACTION SET by the first agent using it.
- **Recipe (own-skeleton mounts):** canonical ids; an action set with the Native horse's action types; your own usage set (a copy of
  the horse's rows with a new id) as a new file under `soln_monster_usage_sets`; rider rows (`mount_id=<new usage id>`) appended to
  Native's `human` set by a same-named `.xslt`; gait clips with `quad_movement` as the FIRST of at most two parameter entries. Base the
  creature on the HORSE when its back is at horse height: rider clips are absolute from the mount origin and `rider_sit_bone` does not
  lift or lower the rider.
- **Mount monster fields:** `rider_sit_bone` a real back bone; `front_/back_bone_to_detect_ground_slope_index` are raw bone NUMBERS
  (signed byte), not names; `body_rotation_reference_bone` must not be the root; `fall_blow_damage_bone` below 28; a unique `family_type`
  keeps horse harnesses off. `Mission.SpawnAgent` mounts any agent with a rideable horse in slot 10.
- **Riderless creature troops are unsafe.** Native builds a non-humanoid agent only from a horse item; a character agent on a quadruped
  monster runs character-only steps (weapon component, facial controller, voices, formation, the AI target scorer reading the TARGET's
  weapon component). Guards held for most sites but the AI target-scorer site still crashed, so creatures are **jockey pairs**: a human
  cavalry NPC (no `race=`) rides an invisible-rider mount whose item carries the creature's monster and skin metamesh.
- **Jockey pair details.** Hide the rider with `AgentVisuals.SetVisible(false)` at build and EVERY tick; health 100000, healed inside
  `OnAgentHit`; blows on the rider are redirected to the mount one tick later in the attacker's name; the rider's own swings are dropped
  (`IsDamageIgnored`); no dismount by blow; mount and rider die together; scale, health and speed live on the MOUNT. The creature attacks
  with a scripted blow in the rider's name shortly after the mount plays a kick clip. The mount agent has no team (use the rider's).
- **Creature AI.** Native AI gives non-humanoid agents no attacks and native cavalry AI charges through and wheels away, so a boss mount is
  in reach a fraction of the time. A hold-and-bite driver (`Agent.SetScriptedPosition` with `AIScriptedFrameFlags`) fixes it.
- **Humanoid-path creatures** (fight as team members or never walk): `monster_usage="human"`, bipedal sets, at most 64 bones with the first
  28 in biped order; bosses that never walk play attacks through spare action slots and are pinned.
- **Flight.** An agent cannot be held in the air: `TeleportToPosition` every tick is pulled back down. Fly a separate `GameEntity`.

## Gotchas
1. **Soldiers slide with frozen feet, later drift and snap each cycle.** See TN 6.
2. **Every animation plays the same death pose.** See TN 5.
3. **Symptom.** Swings land on the wrong part of the arc. **Cause:** clip ranges are in stored keys and rigs store different rates (12.548 keys/s
   for one bake, one per frame for others). **Fix:** measure the key rate per rig and fit ranges in keys. [V]
4. **Symptom.** Conversation, inventory and party-screen figures fall to the reference pose after 2 to 3 s. **Cause:** start clips run once; loop
   clips lacked `cyclic` and the start clip lacked `continue_with`. **Fix:** looping copies with Native's stock flags and follow-ups. [V]
5. **AI troops with new combat clips deal no damage; troops swap weapons back ("Weapon wield interrupted").** See TN 21.
6. **Managed throw in the campaign UI, map or town code for a custom race.** See TN 19 (88 suffix sets and four monster variants).
7. **Holstered items on the wrong bone, UI code reading past the bone array, hit chain misaligned.** See TN 16.
8. **A Native clip on a custom skeleton plays garbage or crashes.** See TN 22.
9. **Symptom.** Melee does about 1 damage per swing; blows register at attack progress 0.00 to 0.04 or with the back of the weapon. **Cause:** the
   race builder gave every race the CROSSBOW AIM frame on the weapon bone, so the blade swung sideways (blade reach 0.59 of Native). **Fix:** a per-rig
   grip rotation on the weapon bone searched over the melee release clips (reach 0.99); recompute landed blows with the engine's `ComputeRawDamage` if
   the swings move the weapon slower in the damage window. Cover every monster that shares the skeleton. [V]
10. **Symptom.** AI never fires although it has line of sight; barely shoots. **Cause:** the AI shoot check traces from the eye (head-look bone frame +
    eye offset, moved back 0.45 x scale) and the eye sat inside the model's own mesh; the aim pose held the gun beside the body. **Fix:** eye at standing
    height, 5 cm in front of the front surface; a torso-only aim layer with the gun in front at chest height. [V]
11. **Riders float 0.45 m above a custom mount's back with straight legs.** See TN 24.
12. **Gait crash at creature spawn in an own quadruped set.** See TN 9, and TN 8 for the canonical-id cause.
13. **Symptom.** A player cannot switch from a heavy gun to melee on a creature mount; a mounted player cannot reload. **Cause:** native refuses the
    animated switch AWAY from a crossbow-class gun on a mount with `FamilyType` of 10 or more (182 of 210 refused; on foot 0), cause unknown; the
    generator wrote `CantReloadOnHorseback` on guns copied from the heavy crossbow usage. **Fix:** Harmony prefixes on `Agent.TryToWieldWeaponInSlot` and
    `WieldNextWeapon` force Instant for those riders (210 of 210); stop writing the flag where a mounted reload clip exists. [V]
14. **Symptom.** Boss rider redirect does nothing; hidden agents reappear. **Cause:** an Invulnerable victim drops every blow before any hit event
    exists; the engine re-shows agent visuals every tick. **Fix:** make the rider mortal with 100000 health and heal inside `OnAgentHit` (runs before the
    death check), redirect to the mount next tick; call `SetVisible(false)` every tick. [V]
15. **Symptom.** Map figure, encounter conversation and loot screen of a creature party show the hidden human jockey; a bandit-talk surrender is offered.
    **Cause:** the party leader is the human rider troop. **Fix:** a Harmony prefix on `PlayerEncounter.DoMeetingInternal` taking the engine's own army path
    (an "Attack / Leave" menu) and a postfix on `CharacterImageIdentifierVM` showing the mount item picture. [V]
16. **Symptom.** A flying boss made by teleporting the real agent every tick falls back to the ground. **Cause:** the engine pulls agents down. **Fix:**
    park the real agent (hidden every tick, AI off, engine blows ignored, following the flyer's ground shadow so enemy AI gathers under it) and fly a
    `GameEntity` built from the creature's skinned metamesh and skeleton along a precomputed arc to a navmesh landing spot; swap back in the same tick;
    far entities `DoNotTick` with hand ticks; fail-safe to a ground boss on any exception. Enemy AI aims at the parked agent's shadow, so count bullets at
    an airborne flyer as aimed. [V]
17. **Symptom.** Small troops are missed by the AI and barely hit. **Cause:** scale 0.45 to 0.5 gives tiny hit capsules. **Fix:** never scale
    below 0.65; for the capsule refit see TN 29. [offline measurement]

## Seen in
- [Borderlands 2 as a Bannerlord total conversion](../games/mount-and-blade-ii-bannerlord/bannerlord-borderlands-2-total-conversion.md), where every item here was learned.
- [Editor-free assets for Bannerlord](editor-free-bannerlord-assets.md) for the binary formats behind the races and clips.


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

