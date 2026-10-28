---
kind: game
title: 'Modding Lobotomy Corporation: LMM/BaseMod Harmony mods, and why Mirror''s MelonLoader setup can''t run with them'
game: Lobotomy Corporation
games_also: []
game_version: '1.0.2.13f, Steam app 568220, buildid 7641517 (Unity 2017.4.7f1, x64)'
platform: windows
engine: unity-mono
route: loader-api
tools:
- Lobotomy Mod Manager (LMM) 1.3.8/1.3.9 (abcdcode, kygi123, tinue)
- Harmony 1.0.9.1 (0Harmony.dll shipped by LMM)
- LobotomyBaseModLib.dll (shipped by LMM)
- MelonLoader 0.7.1 Open-Beta (Mirror multiplayer route only)
- dnSpyEx / ILSpy
- AssetStudio
- Nexus Mods GraphQL v2 API (research)
anti_cheat: none (single-player; Steamworks only, no EAC/BattlEye/Denuvo; saves are unencrypted)
status: in-progress
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-24'
links:
- https://lobotomycorporationmodded.wiki.gg/wiki/Lobotomy_Mod_Manager
- https://lobotomycorporationmodded.wiki.gg/wiki/Troubleshooting
- https://lobotomycorporationmodded.wiki.gg/wiki/Code
- https://lobotomycorporationmodded.wiki.gg/wiki/GettingStarted
- https://www.nexusmods.com/lobotomycorporation/mods/1182
- https://github.com/CTristan/lobotomy-corporation-mods
- https://github.com/Rowrow620/FastForwardSpeed
- https://docs.google.com/document/d/1xQHffu-QQCqWJ26WadJGj4qEa_10c4-q7xnoNMavXIs
tags:
- unity-2017
- mono
- net35
- harmony-1
- basemod
- lmm
- melonloader
- multiplayer
- binaryformatter-saves
- steam-cloud
---
# Modding Lobotomy Corporation: LMM/BaseMod Harmony mods, and why Mirror's MelonLoader setup can't run with them

> This note maps how Lobotomy Corporation (Project Moon, 2018) is modded in 2026. It comes from a read-only
> inspection of a real Steam install, the open-source mods and the community wiki. Nothing was built or tested
> in game yet, so `status: in-progress`. The main route is the community's **Lobotomy Mod Manager (LMM)**. LMM
> patches `Assembly-CSharp.dll` once and then loads Harmony 1 DLLs from `LobotomyCorp_Data/BaseMods/`. A second,
> incompatible ecosystem runs the **Mirror** co-op multiplayer mod on MelonLoader 0.7.1.

## Setup
- **Game:** Steam app 568220, patch **1.0.2.13f** (last Steam patch note 2021-11-02), local buildid 7641517.
  The game has not changed since 2021, so game updates do not break mods. Changes between mod-loader
  generations do.
- **Engine/runtime** (from the MelonLoader log on that install): `Unity Version: 2017.4.7f1`,
  `Game Type: MonoBleedingEdge`, `Runtime Type: net35`, x64. `boot.config` has no `scripting-runtime-version`,
  so the game uses the legacy 3.5 scripting profile. There is no `GameAssembly.dll`, so it is not IL2CPP.
- **LMM:** 21 GitHub releases, from `Update` (2024-03-16) to `Update1.3.9` (2025-09-27), each with
  `Lobotomy.Mod.Manager.<ver>.zip` and `Updater.<ver>.zip`. The inspected copy contains only the string `1.3.8`.
  LMM is a separate Unity app (`LobotomyModManager.exe`) that runs from its own folder, outside the game, and it
  needs Steam running to find the game path. The download carries a modified copy of the game's
  `Assembly-CSharp.dll` and some of its story and stat files (`LobotomyModManager_Data/PatchFiles/`), so this
  note links the community wiki's LMM page instead of the download, and a mod you publish never includes those.
- **What LMM puts in `LobotomyCorp_Data/Managed/`:** a patched `Assembly-CSharp.dll` (4,318,208 bytes, versus
  4,164,096 bytes vanilla), `0Harmony.dll` (FileVersion **1.0.9.1**), `LobotomyBaseModLib.dll`, `NAudio.dll`,
  `Facepunch.Steamworks.Win64.dll`, `steam_api64.dll`, and a `BaseMod/` folder. That folder holds `BackUp.dll`
  (the vanilla assembly), `BaseCreatureGen.xml` and `BaseEquipment.txt`. LMM also creates
  `LobotomyCorp_Data/BaseMods/BaseModList_v2.xml`.
- **Build toolchain used by working open-source mods:** an SDK-style class library. CTristan's mods target
  **net35** with the NuGet package `Microsoft.NETFramework.ReferenceAssemblies.net35`. Rowrow620's mods target
  net472 and also load. References are `Assembly-CSharp`, `0Harmony`, `UnityEngine.CoreModule`/`UnityEngine.UI`
  and optionally `LobotomyBaseModLib`, all with `Private=false`.

## Route and why
- **loader-api through LMM.** Nexus lists about 980 mods for the game, the LMM tool is maintained (by abcdcode,
  kygi123 and tinue, per the wiki), and the wiki documents it. LMM continues abcdcode's BaseMod (5.x). It
  supersedes the Nexus "Basemod" v4.13 page (mod 2), which another user uploaded in 2019 (last updated 2020) and
  which is very outdated.
- **Not plain BepInEx/MelonLoader** for normal mods: none of the content mods would load. MelonLoader 0.7.1 is
  only worth it for the **Mirror** multiplayer mod (Nexus 1182, v0.06ex2, 2026-10-16). Mirror rewrites large
  parts of the game for netcode. Its FAQ says BaseMod mods do not work with it, and it asks you to remove
  BaseMod's `0Harmony.dll` and verify the game files first. Mirror-compatible ports are shared on its Discord.
  In a Mirror lobby a gameplay mod changes the game for everyone in it, so use one only in a lobby you host,
  with friends who agreed and run the same mods.
- **Not assembly replacement** (shipping a whole edited `Assembly-CSharp.dll`). LMM rewrites the assembly on
  every launch, and the wiki lists replacements (Kaizo Corp, an older Memory Leak Fix) as incompatible.

## How the game works (what we had to learn)
- **The vanilla game has no mod loader.** The vanilla assembly contains no `Harmony`, `BaseMods` or `ModInfo`
  names. It does contain a debug console (`ConsoleScript`, `ConsoleCommand`). Abnormality mods publish
  "console spawn IDs" for it. The patched assembly adds `ConsoleCommand_Mod`.
- **Mod entry point:** the patched assembly contains the string `Harmony_Patch`, and every studied mod exposes a
  `public class Harmony_Patch`. The loader appears to construct that class, and the constructor creates a
  `HarmonyInstance` and applies its patches. Harmony is the 1.x API (namespace `Harmony`, `HarmonyInstance`,
  `HarmonyMethod`), not Harmony 2. Classes and members must be `public` (BaseMod readme).
- **Mod discovery:** every folder under `BaseMods/` is a mod. Its DLL must be at the folder's top level.
  `Info/GlobalInfo.xml` holds `<ID>`, and `Info/<lang>/Info.xml` holds the name and descriptions. Mods without
  info show as "Unknown", which does no harm. Load order is the order in `BaseModList_v2.xml`, set by dragging
  mods in LMM.
- **Content folders** a mod can carry (Ziv's Modding Documentation): `Creature/` (CreatureGen,
  CreatureInfo/<lang>, CreatureList, Creatures stats, Portrait 256×256), `CreatureAnimation/` (Spine exports
  renamed `atlas.txt` + `json.txt`, loaded as `Custom/<name>`), `AgentAnimation/`, `Equipment/` (Sprite, txts,
  xmls/<lang>), `BaseModAudioClip/`, `BaseModAssetBundle/`, `BaseModImage/`, `Localize/<lang>/`. An
  Abnormality's behaviour is a class deriving from `CreatureBase`, and its name must match the `<script>` value
  in the stat file. Equipment IDs share the Abnormality ID's digits, with a type prefix: 2 for weapon, 3 for
  armor, 4 for gift.
- **Vanilla data:** gameplay data is TextAsset XML inside `resources.assets`: about 303 `<?xml` documents, with
  301 `<creature` and 448 `<equipment` tags. The code loads them by names such as `xml/CreatureList`,
  `xml/creatureStats/…`, `xml/CreatureGenInfo`, `xml/Equipment/Equipment`, `xml/Research` and
  `xml/Story/StoryList`. All localized text is **loose** XML in
  `LobotomyCorp_Data/ExternalData/xml/Language/<lang>/` (13 languages, 2,121 files), so text and translation
  mods need no tools. `LobotomyCorp_Data/CustomData/{Face,Hair}` is a vanilla folder for agent cosmetics as PNG.
- **Saves:** `%USERPROFILE%\AppData\LocalLow\Project_Moon\Lobotomy\saveData170808.dat`, `saveGlobal170808.dat`
  and `etc170808.dat` (each with a `.backup`), plus `agentData/` and `creatureData/`. The header is
  `00 01 00 00 00 FF FF FF FF 01…`, which is .NET **BinaryFormatter**, a serialized
  `System.Collections.Generic.Dictionary`, not encrypted. Steam Cloud syncs this folder (`steam_autocloud.vdf`).
- **Useful game symbols:** `GameManager` (`gameSpeedLevel`, `state`, `UpdateGameSpeed`), `WorkerModel` →
  `AgentModel`, `CreatureBase`, `CreatureModel.metaInfo` → `CreatureTypeInfo`, `CreatureTypeList.instance`,
  `CustomizingWindow.Confirm`.
- **Logs:** gameplay errors go to `LocalLow\Project_Moon\Lobotomy\LobotomyBaseMod\Log.txt`. Load errors go to
  `LobotomyCorp_Data\BaseMods\*error.txt` (`LMMerror`, `Herror` for Harmony, `DllError`, `LTDerror` for
  localization, `Glerror` for save read). LMM's own errors go to `<LMM>\LobotomyModManager_Data\Log.txt`, which
  is in Korean (`패치 성공!` means "patch success"). Hard crashes go to
  `LocalLow\DefaultCompany\LobotomyModManager\Player.log`. Every launch overwrites these logs.

## Build steps
1. Turn off Steam Cloud for the game, and copy `LocalLow\Project_Moon\` somewhere outside the game folder.
2. Extract the LMM zip to a folder **outside** the game. Start Steam, run `LobotomyModManager.exe`, check
   *Set Game Path*, then **Start Game**. Success looks like this: the title screen shows the BaseMod/LMM version
   in the top-left corner, and `Managed/` has the files listed under Setup.
3. Make the project: `dotnet new classlib -f net35`, add the reference-assemblies NuGet package, and reference the
   game's `Managed` DLLs with `Private=false`. The minimal entry class looks like this (written for this note,
   not taken from game code):
   ```csharp
   public class Harmony_Patch {
       public Harmony_Patch() {
           var h = Harmony.HarmonyInstance.Create("MyFirstMod");
           var target = typeof(GameManager).GetMethod("UpdateGameSpeed",
               BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
           h.Patch(target, null, new Harmony.HarmonyMethod(typeof(Harmony_Patch).GetMethod("Post")));
       }
       public static void Post(GameManager __instance) {
           if (__instance.gameSpeedLevel == 3) UnityEngine.Time.timeScale = 6f;
       }
   }
   ```
4. Deploy to `LobotomyCorp_Data/BaseMods/MyFirstMod/MyFirstMod.dll` with `Info/GlobalInfo.xml` and
   `Info/en/Info.xml`. Always launch through LMM. Quit with **Options → Quit Game**, because only that exit
   makes LMM's save backup (`Managed\BaseMod\BackUp`).
5. Compare with the readable sources: `Rowrow620/FastForwardSpeed` (under 50 lines) and
   `CTristan/lobotomy-corporation-mods` (MIT, net35, unit-tested patches, shared common package).

## Verification
- **Verified, read-only on a real install:** the Unity version and runtime (MelonLoader log), the vanilla vs
  patched assembly sizes and which names each contains, the files LMM adds (LMM's own patch log lists each copy),
  the Harmony FileVersion, the save header bytes and location, the counts of loose language XML and embedded XML.
- **Verified from sources:** the mod layout and entry class (three public repos), the LMM release dates
  (GitHub API), and mod counts and activity (Nexus GraphQL v2: 982 mods, more than 20 updated 2026-10-17…24).
- **NOT verified:** that the build steps above load in game, the exact rule the loader uses to find
  `Harmony_Patch`, the console hotkey and commands, a stand-alone BinaryFormatter round-trip of the saves, and
  LMM 1.3.9 behaviour (the inspected copy says 1.3.8).

## Gotchas
1. **Title screen shows "BaseMod 5.2" and later mods don't run.** **Cause:** one mod threw during load (an
   "AddOn Init error"), and loading stops there. A common trigger is calling an API that is not in Unity
   2017's .NET 3.5. **Fix:** check `BaseMods\*error.txt` and remove mods one at a time. Target net35, or avoid
   newer APIs even if you build for net472.
2. **The mod is in `BaseMods` but never loads.** **Cause:** a double folder
   (`BaseMods\Mod-608-1-0-…\Mod\Mod.dll`), files extracted loose into `BaseMods` (you see `Info/`,
   `Equipment/` at the top), or a GitHub source zip (`.sln`/`.cs`). **Fix:** keep exactly one folder level, with
   the DLL at the top. LMM 1.3.6+ extracts only `.zip`; use Manual install for `.rar`/`.7z`.
3. **Edits to `Managed/Assembly-CSharp.dll` disappear.** **Cause:** LMM patches the assembly again on every
   launch through `LobotomyModManager.exe`. **Fix:** never edit the assembly. Use Harmony from a BaseMods DLL.
   LMM ships a patch assembly in `LobotomyModManager_Data\PatchFiles` for reference.
4. **Mirror multiplayer and LMM mods break each other.** **Cause:** Mirror needs MelonLoader 0.7.1 and a clean
   `Managed/` without BaseMod's `0Harmony.dll`, while LMM needs a patched assembly and that same Harmony.
   **Fix:** choose one per install. Switching means LMM install/uninstall (Steam *Verify integrity* restores
   vanilla), or keeping a second game copy for multiplayer.
5. **Saves "come back wrong" after removing mods, or a new game stalls.** **Cause:** Steam Cloud restores
   saves that reference content you removed. **Fix:** turn Cloud off for the game while modding, and restore
   from your own copy of `LocalLow\Project_Moon\`.
6. **"Unexpected binary element" popup on load.** **Cause:** a corrupt `ModOptionData.dat`. **Fix:** delete it,
   and LMM regenerates it (wiki Troubleshooting).
7. **A mod works for some players only.** **Cause:** it needs the old unofficial `12Harmony.dll` (from BaseMod
   5.2.1), which some installs still have and others don't. **Fix:** for private members, depend on
   `LobotomyBaseModLib.dll` (always installed by LMM) instead: its `ExtenionUtil_Lib` extension methods
   (`GetFieldValue<T>`, `SetFieldValue`, `Invoke`) reach them by reflection. The wiki's other fix is to ship
   `12Harmony.dll` in your mod's folder.
8. **A research snapshot of the install went stale.** **Cause:** the human was switching loaders (MelonLoader
   removed, LMM patched in) while the agent was hashing files, and `Assembly-CSharp.dll` changed size between
   two reads a few minutes apart. **Fix:** list the mtimes of `Managed/` and the game root before drawing
   conclusions, and re-check after any surprise. LMM's `Managed/BaseMod/BackUp.dll` is a reliable vanilla copy
   for decompiling.
9. **The Nexus web page returns 403 to fetch tools.** **Fix:** the keyless GraphQL endpoint
   (`POST https://api.nexusmods.com/v2/graphql`, `mods(filter:{gameDomainName:{value:"lobotomycorporation"}})`,
   `legacyModsByDomain` for descriptions) returned everything needed.

## Assets
None made. For reference: art is extracted with AssetStudio from `resources.assets`/`sharedassets*.assets`.
Extracted art stays on your machine; a published mod carries its own art.
Abnormality animations are Spine 2D (the trial version cannot export). Audio is `.mp3` in `BaseModAudioClip/`.
AssetBundles need the Unity **2017.4.7f1** editor (Ziv's doc says it needs modification to build them).

## Credits
- Lobotomy Mod Manager and BaseMod: abcdcode, with kygi123 and tinue maintaining LMM (per the community wiki).
- Mirror multiplayer mod: Clair (Clairidze on Nexus).
- Ziv's Modding Documentation (the Google Doc in `links`) and the Lobotomy Corporation Modded wiki on wiki.gg.
- CTristan's `lobotomy-corporation-mods` and Rowrow620's `FastForwardSpeed` (both MIT), the readable sources in
  Build step 5.

## Cost and time
About one session of research: web, GitHub API, Nexus API and read-only local inspection. No API spend.

## Open questions
- Build and load the minimal `Harmony_Patch` mod in game to confirm the net35 build steps and the entry rule
  (decompile the patched assembly's loader; the wiki's Code page names `Add_On` as the class that loads mods).
- The console hotkey and command list (`ConsoleScript` / `ConsoleCommand_Mod`).
- A save editor: deserialize the BinaryFormatter `.dat` files outside the game, with `Assembly-CSharp.dll`
  referenced for the game types.
- A clean way to switch between the Mirror (MelonLoader) and LMM setups, or to keep them side by side.
- Mirror's internals (it bundles its own Steam/LAN transport) if a BaseMod port ever ships.


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

