---
kind: game
title: "Helldiver survivor: custom skills, stratagems, Helldivers 2 models on the Commando rig, config, lab bridge and benchmark"
game: "Risk of Rain 2"
games_also: ["Helldivers 2"]
game_version: "1.4.1#912 (Unity 2021.3.33f1, Mono), Steam, Windows 11; separate lab copy of the install"
platform: windows
engine: unity-mono
route: loader-api
tools: ["BepInExPack 5.4.2122", "RoR2BepInExPack 1.43.0", "R2API ContentManagement 1.0.11 / Language 1.1.0 / Prefab 1.1.1", "HookGenPatcher 1.2.9 (lab only)", ".NET SDK 8 (netstandard2.1)", "ilspycmd 8.2", "filediver 0.7.55", "Blender 4.0 (headless)", "uv + numpy/pillow"]
anti_cheat: "none (co-op PvE, no anti-cheat); all testing in a separate lab copy of the game"
status: working
agents: ["Claude Code (Claude Opus 4.x, Sonnet and Opus 5.5 across sessions)"]
humans: ["Terr4"]
date: 2026-10-04
links: []
tags: [survivor, skills, entity-states, stratagems, skinned-mesh, model-swap, ik, cloth, normal-maps, decimation, texture-compression, config, benchmark, test-bridge]
---

# Helldiver survivor: custom skills, stratagems, Helldivers 2 models on the Commando rig

> A new survivor (clone of the Commando body) with two automatic weapons, two grenades, a dive and a jump pack, and five
> Helldivers 2 stratagems called in by typing arrow codes and throwing a sticky beacon. An optional Model folder replaces the Commando
> with a Helldiver converted from the player's own HD2 install (body skinned to Commando's bones, two weapons held with two-hand IK,
> a Verlet cape, a jump pack with flames). 1.0.0 adds a full BepInEx config and optimized models. Verified in the real game through
> an in-process test bridge: scripted feature runs, screenshots and an A/B benchmark. Multiplayer not tested.

## Setup
- Lab: a copy of the Steam install with BepInEx; the real install stays vanilla. RoR2BepInExPack must live in
  `plugins/RiskofThunder-RoR2BepInExPack/` (a folder named `plugins/RoR2BepInExPack` is deleted by the preloader).
- R2API 5.x split modules at their latest versions. The versions pinned by the R2API 5.0.5 meta-package are stale and fail on Newtonsoft/MMHOOK.
- Game code is in `RoR2.dll`, not Assembly-CSharp. HookGenPatcher needs `MMHOOKAssemblyNames = RoR2.dll`.
- Build: netstandard2.1, references straight from the lab `Managed/` and plugin folders. A `Lab` configuration adds the test bridge (`#if LAB`)
  and the MMHOOK/MonoMod references; the release DLL has neither.

## Route and why
BepInEx plugin + R2API (ContentAddition, PrefabAPI, LanguageAPI). This is the standard RoR2 route, and RoR2 has no anti-cheat.
- **Body:** a `PrefabAPI.InstantiateClone` of CommandoBody keeps animations, hurtboxes and networking for free.
- **Skill families:** the clone's GenericSkill slots get new SkillFamilies, set through the private `_skillFamily` field.
- **States:** every skill is its own EntityState type, registered with `ContentAddition.AddEntityState`.
- **Models:** HD2 art is converted offline and loaded at runtime from loose files. It is never compiled into the DLL, so a share-safe package is just the DLL.

## How the game works (what we had to learn)
- **Entity states are rebuilt by type on remote clients.** A state can't carry constructor arguments, so every stratagem needs its own state
  types. Keep them as one-line subclasses over a data table (`StratagemDef` indexed by an enum).
- **EntityStateConfiguration.** Static fields apply to the exact type. Instance fields go through `EntityStateCatalog.InitializeStateFields`,
  keyed by `GetType()`. A subclass of a vanilla state therefore does not get the vanilla tuning; copy it from a plain `new VanillaState()` instance.
  Vanilla states also share static effect fields (DodgeState's `jetEffect`): null them around `base.OnEnter()` instead of hooking.
- **Skill selection order.** Player profiles store the selected variant per slot by index, so only ever append variants.
  In the character select the display model has no body or skills: read the choice from
  `userProfile.loadout.bodyLoadoutManager.GetSkillVariant(bodyIndex, slot)`.
- **Model swap.** The skin system (ModelSkinController) re-applies Commando's mesh and materials whenever the loadout or skin changes, which happens
  constantly on the character select. Check the body SkinnedMeshRenderer every LateUpdate and re-apply.
  - **Bones:** a mesh fitted to the Commando rest pose can reuse Commando's bindposes, with bones resolved by name.
  - **Add-ons:** for extra renderers (weapons, backpack, cape) to fade with the camera, cloak with stealth items and show hit flashes and elite
    overlays, they must be in `CharacterModel.baseRendererInfos`. The skin system replaces that array, so re-append them after every skin apply.
- **Update order.** `ModelLocator.LateUpdate` moves the model to the body. Anything reading bones in LateUpdate (cloth, IK) needs a late
  `[DefaultExecutionOrder]`, or it lags one frame (about 35 cm while sprinting at 20 FPS).
- **Hopoo deferred standard shader.** Properties: `_MainTex`, `_NormalTex` + `_NormalStrength`, `_EnableCutout` + keyword `CUTOUT` (no `_Cutoff`),
  `_Cull`, `_EmPower`, `_Smoothness`, and `_Fade` / `_FlashColor` driven by CharacterModel. The normal map is unpacked RG-or-AG, so DXT5nm works.
- **AimThrowableBase** fires its projectile in `OnExit`, and `PickNextState()` chooses the follow-up. Return a tiny "wait for the fire button to be
  released" state, or the held click immediately fires the primary on the same Weapon state machine.
- **Automatic weapons.** A one-shot state with interrupt priority Any re-triggers every physics frame (an endless stream). Use one long-lived state
  that fires at an exact rpm while the key is held.

## Build steps
1. Code: Content/ (survivor, tokens, assets, passive), States/, Stratagems/, Visuals/, Settings.cs (all BepInEx config entries), Dev/ (lab only).
2. Models: filediver (read-only on the HD2 install) writes GLB units. Python scripts then:
   - fit the avatar onto Commando's rest skeleton (one rigid + axial-scale transform per Commando bone, linear blend skinning);
   - bake weapon and backpack rest poses;
   - resample the cape into a 9 x 13 grid.

   Then `optimize.py` runs inside Blender (`blender -b -P`):
   - weld the triangle soup;
   - collapse-decimate (vertex groups carry the skin weights);
   - Smart UV Project + pack;
   - Cycles bake, selected-to-active: emission for colour, tangent-space normal map;
   - write the mesh with Blender's MikkTSpace tangents, so the game decodes the normal map exactly as baked.
3. Runtime textures: PNG, then block-compressed at startup:
   - colour maps: copy into an RGB24 texture, then DXT1 (PNG `LoadImage` always gives ARGB32, which compresses to DXT5);
   - normal maps: swizzle to DXT5nm, then compress without dithering;
   - release the CPU copy with `Apply(false, true)`;
   - preload everything in `Awake`, inside the loading screen.
4. Package: manifest + README + CHANGELOG + icon. Ship two zips: DLL + Model folder (personal), and the DLL alone (share-safe, passes `um publish check`).

## Verification
- **Oracle: an in-process test bridge.**
  - **Input:** a MonoBehaviour polls `BepInEx/hd_cmd.txt`. Input is injected after `PlayerCharacterMasterController.FixedUpdate` (MMHOOK), so
    held skills, move/look, arrow keys and loadout changes need no OS input.
  - **Output:** screenshots via `ScreenCapture`, `status` lines in the log, and timed `seq` scripts.
  - **Quiet runs:** director enemies are removed unless asked for.
  - **Fixed location:** `stage <scene>` calls `Run.instance.AdvanceStage`, and `tpabs` teleports to a set point. Together they give repeatable
    scenes; cheats are server-only, so console `set_scene` is unavailable.
- **Scripted checks:**
  - **Full feature run:** both guns empty their magazine and reload; grenades and stratagems spend charges with the configured cooldowns; the
    jump pack nozzles exist and point down; the log has no errors.
  - **Config:** edit the .cfg, launch, and confirm health, charges, magazine, rpm, reload and cooldown changed.
  - **Share-safe build:** no Model folder, no errors.
- **Benchmark:** MonoMod detours time the mod's Update/LateUpdate/OnGUI methods and its loading. Four uncapped 8 s scenes: idle, firing,
  sprinting in circles, and 12 extra display-model copies in front of the camera. It reports FPS, 1% lows, mod ms/frame, GC and texture bytes
  computed from the formats; the profiler reports 0 for non-readable textures. Hooks are resolved by type name, so the same tool builds against
  old source for an A/B.
  - **Result** (7800X3D + 4080 Super, 1440p): idle 398 → 542 FPS, 13 Helldivers 318 → 420. Mod CPU 0.385 → 0.097 ms/frame. Our meshes
    209k → 30k vertices; GPU textures 48 → 13 MB.
- **Not verified:** multiplayer (no second client in the lab), and other users' r2modman setups.

## Gotchas
1. **Commando flashes for one frame on the character select.** Cause: the skin system re-applies Commando's mesh on every loadout/skin change. Fix: check the SMR's sharedMesh and material every LateUpdate and re-apply; a 0.5 s poll is too slow.
2. **The cape lags behind the body.** Cause: it read bones before `ModelLocator.LateUpdate` moved the model. Fix: `[DefaultExecutionOrder(32000)]` on the cloth component.
3. **The cape cost 20 FPS.** Cause: every one of ~1,300 vertices was simulated, and a low frame rate meant more sub-steps (a spiral). Fix: a 5 x 7 particle grid rendered as an interpolated 9 x 13 mesh, at most 3 sub-steps, and the body's movement partly passed straight to the particles. Fix 2: write the hot loops in floats; Mono runs Vector3 operators as real calls, and this was another 4x.
4. **Stealth items cloak the body but not the gun or backpack** (they also don't fade near the camera or take overlays). Cause: CharacterModel only manages `baseRendererInfos`. Fix: append your renderers there, and again after each skin apply.
5. **The fitted body's hands had the wrong chirality and needed 110-160° of wrist twist.** Cause: the z mirror of the coordinate conversion. Fix: reflect the glove vertices across the hand bone's local plane offline. A weighted blend twists the wrist triangles, so use a hard cut. Lesson: plot the mesh in bone space before guessing axes.
6. **A subclassed vanilla state does not get vanilla tuning** (duration, speeds). Cause: state configuration is keyed by exact type. Fix: copy the values from a plain instance of the vanilla type once.
7. **Throwing the beacon also fires the gun.** Cause: after AimThrowableBase exits, the held fire button triggers the primary on the same state machine. Fix: `PickNextState` returns a short state that ends when the button is released.
8. **The aim arc goes NaN when aiming straight down or at something in your face** (log spam). Cause: AimThrowableBase trajectory maths divides by zero. Fix: override `UpdateTrajectoryInfo` and fall back to a plain forward throw.
9. **Decimation was hampered by the converted meshes.** Cause: they were triangle soups with a per-triangle colour cell, so every edge is a UV seam (41k triangles meant 123k vertices). Fix: weld by position, drop UVs, decimate, re-unwrap, and bake the colour back from the soup.
10. **HD2 normal maps look wrong if used as RGB.** Cause: R/G hold X/Y with +Y up in the image, B is not Z, and A is a height map. Check by correlating R/G with the derivatives of A. Fix: rebuild Z from X/Y before baking. The bake then puts HD2's panel lines and screws on the low-poly model.
11. **Colour maps came out DXT5 instead of DXT1.** Cause: `Texture2D.LoadImage` gives ARGB32 for every PNG. Fix: copy into an RGB24 texture before `Compress`.
12. **Benchmark FPS differed by up to 30% between runs of the same build.** Cause: `tptele` stands next to the teleporter, which spawns somewhere different each run. Fix: a fixed stage and an absolute teleport.
13. **Lab bridge commands got lost.** Cause: command files written less than ~1.5 s apart race the 0.25 s poll. Fix: one file with a `seq "a; wait 0.5; b"` command.
14. **Config sections came out in the wrong order** ("10." before "2."). Cause: BepInEx writes sections sorted as strings. Fix: zero-pad the numbers ("01.").
15. **Old config entries linger after an update.** Cause: BepInEx keeps unknown entries as orphans. Fix: read the protected `OrphanedEntries` by reflection, migrate the values, remove the keys and Save.

## Assets
- Icons are procedural, embedded in the DLL.
- **Models:** converted from the player's own Helldivers 2 install. They ship only in a personal package, never in the share-safe one.
- **Colour:** HD2 stores no albedo for this gear (layered material IDs), so colours are baked per region by script.
- **Normal maps:** the weapons and pack carry HD2's normal-map detail into the bake.
- **Budgets:** body 12,000 tris (2048 colour, 1024 normal); weapons ~3,500 each; pack 1,800 (1024 maps).

## Cost and time
Several sessions over about four days. The 1.0 cleanup, optimization, config and benchmark took one long session.

## Open questions
- Multiplayer: networking follows the vanilla patterns (registered state types, a server-side beacon), but it has never been played with a second client.
- An in-game options menu (Risk of Options) instead of editing the .cfg.
- Under the cloak shader the cape ignores its alpha cutout, so it shows as a plain sheet.


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

