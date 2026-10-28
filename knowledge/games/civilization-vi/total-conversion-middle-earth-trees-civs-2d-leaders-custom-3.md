---
kind: game
title: 'Total conversion (Middle-earth): trees, civs, 2D leaders, custom 3D wonders, victory reskin'
game: Civilization VI
games_also: []
game_version: 'Civ VI 1.0.12.68 (Steam 289070, build 15296837), Gathering Storm ruleset'
platform: windows
engine: native
route: data
tools: ['Civilization VI Development Tools (SDK + SDK Assets, Steam)', 'Civ6AssetCooker_FinalRelease', 'FbxImporter_Release.dll (from the Asset Editor)', 'Blender 5.2', 'fal (nano-banana-2, trellis-2, birefnet/v2)', 'ffmpeg gfxcapture', 'sqlite3', 'custom SendInput mouse (PowerShell)']
anti_cheat: 'none (EOSSDK is Epic Online Services for 2K accounts, not anti-cheat); single player only'
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-10'
links: ['https://forums.civfanatics.com/forums/civ6-creation-customization.538/', 'https://forums.civfanatics.com/resources/civnexus6.25987']
tags: [total-conversion, data-mod, modinfo, sqlite, lua, tech-tree, custom-civ, 2d-leader, fallback-leader, wonder-3d, granny, fgx, asset-cooker, headless-sdk, victory-screen, autoplay, test-harness]
---
# Total conversion (Middle-earth): trees, civs, 2D leaders, custom 3D wonders, victory reskin

> "Ages of Arda" turns Civ VI (Gathering Storm) into Middle-earth with a plain `.modinfo` data mod:
> - nine eras, restructured tech and civic trees;
> - 8 civilizations with 14 leaders, shown as painted 2D portraits;
> - gunpowder and air units locked;
> - city-states, Great People, governors, religion, Eurekas and victories reskinned;
> - 52 of 53 world wonders with their own 3D models.
>
> The wonder models went concept image → image-to-3D → Blender, then into Firaxis' Granny `.fgx` format with no
> GUI, by calling the SDK's FBX importer DLL directly.
>
> All of it was checked in the running game, using real mouse input, screenshots, `Lua.log` and `Modding.log`, a
> 100-turn AI autoplay, and scripted test scenes that place wonders and force a victory.

## Setup
- **Game:** Windows 11, Civ VI 1.0.12.68 (Steam), Base + Rise and Fall + Gathering Storm, played on the GS ruleset.
  - Windowed at a 1920x1080 client size; every click coordinate below assumes it.
- **SDK:** Civilization VI Development Tools from Steam (Library → Tools).
  - "Sid Meier's Civilization VI SDK" holds `Civ6AssetCooker_FinalRelease.exe` and the Asset Editor.
  - "...SDK Assets" holds the pantries. The real pantry is `SDK Assets\Civ6\pantry`; DLC pantries are under
    `Civ6\DLC\*\pantry`.
- **Art and checks:**
  - Blender 5.2 (headless, `--background --python`).
  - ffmpeg for image work.
  - `sqlite3` for offline database checks.
- **Mod location:** `Documents\My Games\Sid Meier's Civilization VI\Mods\<Mod>\`. The game picks it up from there;
  enable it under Additional Content.
- **Logs:** `%LOCALAPPDATA%\Firaxis Games\Sid Meier's Civilization VI\Logs\`, mainly `Modding.log`, `Database.log`
  and `Lua.log`.
- **Debug database:** after a game starts, `...\Cache\DebugGameplay.sqlite` holds the merged gameplay database.
  Query it to confirm what really loaded.

## Route and why
- **The route:** an official data mod.
  - A `.modinfo` whose actions are UpdateDatabase (SQL/XML), UpdateText, UpdateIcons, UpdateColors and UpdateArt
    (a cooked `.dep`).
  - Plus ImportFiles and AddGameplayScripts for Lua.
- **Proxy-DLL hooks: no.** `um scan` reports "unknown native" and suggests them; that's wrong for this game,
  because data and Lua reach every system needed here.
- **ModBuddy: no.** The cooker is a command-line tool, so the whole pipeline is scripts. A generated modinfo
  (`dev/gen_modinfo.sh` from a manifest) avoids hand edits.
- **Custom 3D wonders:** the documented path is the Asset Editor importing from 3ds Max/Maya. The community's
  is Deliverator's CivNexus6 (`.fgx` import/export, Blender scripts, and FBX → `.fgx`). Both are GUI tools; the
  headless DLL route below replaces them.

## How the game works (what we had to learn)
**Databases and load order**
- There are two SQLite databases.
  - **Configuration** (FrontEnd) drives game setup. The civ and leader picker reads `Players` rows in the domains
    `Players:StandardPlayers`, `Players:Expansion1_Players` and `Players:Expansion2_Players`, with Portrait and
    PortraitBackground columns for the 2D art.
  - **Gameplay** (InGame) holds rules and text.
- Each action has a LoadOrder, and later wins. Text is one `LocalizedText` table keyed by (Language, Tag).
  `<Replace Tag Language>` rows override base text.

**Tech and civic trees**
- Layout is derived, not stored: the column is the node's cost rank within its era, and the row is `UITreeRow`.
- Prerequisites must cost less than the node that needs them.
- **Never delete base TECH_/CIVIC_ rows.** Boosts, AI and modifiers reference them. Re-link the prerequisites and
  rename the text instead.

**Units**
- Gunpowder and air units can be locked by giving them a `TraitType` that no civ has (e.g. `TRAIT_ARDA_DISABLED`).
  Then fix `ObsoleteTech` / `MandatoryObsoleteTech` on the units left playable, so nothing upgrades into a locked
  unit.
- New units: `INSERT ... SELECT` from a base unit, and clone its `Units.artdef` entry for the art.

**Civs and leaders**
- New civs and leaders can reuse existing trait kits.
- To have only your own civs in setup, delete the other `Players` rows in all three domains. The game then offers
  only yours.
- Civs need citizen names (male and female, classic and modern; base civs have 10 of each). See Gotcha 6.

**2D leaders (no 3D)**
- An entry in `ArtDefs/FallbackLeaders.artdef` per leader points at a BLP texture.
- The diplomacy screen draws it with an `Image` control (`FallbackLeaderImage`), centred at 3/4 of screen width and
  scaled to screen height.
- The 3D leader scene is absent, so anything the image doesn't cover is **black** (Gotcha 9).
- Loading screen and setup use `LoadingInfo.ForegroundImage` / `BackgroundImage` and `Players.Portrait` /
  `PortraitBackground` (UI textures by name).

**Cooking 2D art without ModBuddy**
- PNG → DDS (a small C# writer: uncompressed RGBA plus mips) → `.tex` / `.xlp` / `.artdef` / `.Art.xml`.
- Then `Civ6AssetCooker_FinalRelease.exe` in its three modes: `XLP` → BLP, `ArtDef`, `Dependency` → one generated
  `<Mod>.dep`, listed under UpdateArt.
- The cooker accepts several `--pantry`; pass the SDK Assets pantries so cloned base assets resolve.

**Wonder models**
- A wonder model is a `WonderMovie` asset: `.geo` (XML mesh list) + `.fgx` geometry, and `.anm` + `.fgx`
  animation (the build-up, 500 frames at 30 fps), plus `.ast`, `.mtl` and textures.
- The element is keyed by `BUILDING_*` in `WonderMovie.artdef`. Overriding that element in your own artdef/package
  swaps the model for the map, the build-up and the completion cinematic.
- `.fgx` is binary **Granny 3D** (GR2: Oodle-compressed sections, Firaxis string remapping). Blender can't write it
  directly. CivNexus6 converts between `.fgx` and its own `.cn6`, which its Blender scripts read and write, but it
  runs as a GUI and isn't part of this pipeline.

**The headless .fgx route**
- The Asset Editor's managed `Firaxis.CivTech.Impl.dll` has an FBX interface whose export only P/Invokes
  `FbxImporter_Release.dll!ExportFBXScene(fbx, node, out.fgx, ShowInterface, ShowProgress, hWnd)`.
- Call it directly with `ShowInterface=false`, `ShowProgress=false`, `hWnd=0` and it converts an FBX scene to
  `.fgx` with no window.
- Defaults omit tangents: you get `Position3 Normal3 TextureCoordinates0x2`. The shipped layout is
  `Position3 Normal3 Tangent3 Binormal3 TextureCoordinates0x3`. Patch the vertex data afterwards: the SDK's own
  Granny wrapper can load and save the file.
- Missing tangents cook to zeros (black/flat shading); see Gotcha 11.
- **Expansion/DLC wonders** (Etemenanki, Statue of Zeus, Angkor Wat, Torre de Belém, Bolshoi, Venetian Arsenal...)
  have **no SDK pantry asset**. Clone the `.ast`, camera, sounds and footprint from a similar base-game pantry wonder,
  and keep the artdef element from the DLC.
- **Coastal wonders** (Colossus, Great Lighthouse, Torre de Belém, Venetian Arsenal, Sydney Opera House) sit on a
  `TERRAIN_COAST` water tile next to land, not on coastal land. A self-contained diorama model makes the facing
  irrelevant.

**End-game screen**
- The base `UI/EndGame/EndGameMenu.lua` ends with `include("EndGameMenu_", true)` (a wildcard include).
- So an ImportFiles Lua named `EndGameMenu_<yours>.lua` loads inside that context and can patch the global
  `Styles` table:
  - `Background` (a UI texture name);
  - `Movie` (set it to nil to drop the base cinematic; the replay button hides itself);
  - ribbons.
- No Firaxis file is replaced. Expansion files can add styles after yours, so patch again in a wrapper around the
  global `View(data)`.
- Several other base screens end with the same kind of wildcard include. Grep for `include(".*_", true)` before
  reaching for ReplaceUIScript.

**Testing hooks (Lua)**
- In gameplay scripts (AddGameplayScripts):
  - `cities:Create(x, y)`, `UnitManager.Kill(unit)`;
  - `capital:GetBuildQueue():CreateIncompleteBuilding(buildingIndex, plotIndex, 100)` places a finished wonder
    and plays its completion view;
  - `WorldBuilder.CityManager():SetPlotOwner(plot, city)` claims tiles;
  - `PlayersVisibility[0]:ChangeVisibilityCount(plotIndex, 1)` reveals them.
- In a UI script: `UI.LookAtPlot`, `UI.SetMapZoom`, and
  `UI.RequestAction(ActionTypes.ACTION_ENDTURN, { REASON = "UserForced" })`. The last one is what shift-clicking
  End Turn does, and it ignores the "choose production / research / unit needs orders" blockers.
- `AutoplayManager` (`SetTurns`, `SetReturnAsPlayer`, `SetObserveAsPlayer(PlayerTypes.OBSERVER)`, `SetActive`)
  runs AI-only turns for soak tests.
- `UserConfiguration.TutorialLevel(-1)` plus `LuaEvents.TutorialUIRoot_AdvisorLower()` on
  `LuaEvents.TutorialUIRoot_AdvisorRaise` silences advisor popups for the session.

## Build steps
1. **Back up.** Back up `Documents\My Games\Sid Meier's Civilization VI` (`um backup create`). Make a copy of the
   stock gameplay database for offline checks: let a game start, then copy `Cache\DebugGameplay.sqlite`.
2. **Write the data.** One SQL file per era for the trees; units, civs (gameplay) and civs (configuration) SQL.
   Text as `TAG|Text` lists turned into `<Replace>` XML by a script.
3. **Generate the `.modinfo`** from a manifest:
   - FrontEnd: configuration SQL, text, icons, colours, art.
   - InGame: gameplay SQL, text, icons, colours, art, ImportFiles.
   - Text at LoadOrder 5000 (Gotcha 7).
4. **Check offline before every launch.** Apply the gameplay SQL in manifest order to a copy of the stock database
   with `PRAGMA foreign_keys=ON` and check for:
   - foreign-key violations;
   - unreachable tree nodes;
   - prerequisites that cost more than their dependants;
   - XML that doesn't parse;
   - text tags that don't exist in the game;
   - a modinfo that doesn't match the manifest.

   This catches most mistakes in seconds; a game launch takes minutes.
5. **2D art.** Leader cut-outs and civ landscapes → PNG inputs (ffmpeg) → DDS → cook (XLP, ArtDef, Dependency) →
   install BLPs under `Platforms/Windows/BLPs/` plus the `.dep` and artdefs; list them under `<Files>`.
6. **Wonders**, one at a time:
   - concept image → image-to-3D GLB;
   - Blender: clean up, decimate to about 25k tris, bake one texture set, level the albedo to the base wonders'
     brightness, cut into height bands and keyframe a rise-up build animation;
   - FBX → `ExportFBXScene` → `.fgx` → patch tangents → `.geo` → clone `.ast`/camera from the base wonder;
   - write the WonderMovie artdef → cook all wonders into one package.
7. **Victories.** Text, plus `UI/EndGameMenu_<Mod>.lua` (ImportFiles) and UI textures for the end-game backgrounds.

## Verification
- **The game is the oracle.** Each step was launched for real.
- **Tech and civic trees:** opened and screenshotted (gfxcapture).
- **New games:** started as several leaders; checked setup, the loading screen and the diplomacy view.
- **Logs:** `Modding.log` for "Error Loading" lines (anything not ours was already there before the mod).
  `Lua.log` for runtime errors and the harness `print`s.
- **Soak test:** 100 turns of AI autoplay as several civs. 0 Lua errors; the Database.log error count stayed at
  the pre-mod baseline.
- **Wonders:** a test scene places them at 100% next to the capital and frames the camera; each screenshot was
  reviewed. In game, I checked the Argonath, Orthanc, the Tower of Ecthelion and five coastal wonders. The other 44
  were only reviewed from Blender renders taken from the game's camera angle. Loads of a game with all 52 models
  were fine; low-memory machines were not tested.
- **Victory screen:** a harness removes the rival civs' units on turn 1 and force-ends the turn. Victory of Arms
  showed with the new background, no movie and the new ranking list. The other victory types were not triggered,
  but they share the patched code path.
- **Not verified:** multiplayer, Steam Workshop upload, non-English languages (English only), other rulesets
  (vanilla / Rise and Fall).

## Gotchas
1. **`um win drive click` is ignored by Civ VI.**
   - **Cause:** it sets the cursor position and sends button down/up with no move events; Civ needs real moves.
     Two bugs in my own replacement followed: an x64 `INPUT` struct padded to 48 bytes (SendInput returns 0,
     error 87; it must be 40), and a PowerShell `-File` script with `param([int[]]$a)` that bound only the first
     number.
   - **Fix:** SendInput with ABSOLUTE|VIRTUALDESK moves, gliding about 25 steps to the target, a 300 ms dwell,
     then down/up, with separate int parameters.
2. **Autoplay or tests hang on "the AI's turn".**
   - **Cause:** an advisor popup is waiting (first wonder, volcano, historic moment...).
   - **Fix:** in a dev UI script, `UserConfiguration.TutorialLevel(-1)` and lower or clear raised advisors.
     Do this in every harness, before anything else.
3. **The new civ or leader is missing from setup, or setup shows base civs only.**
   - **Cause:** the Configuration database is a separate FrontEnd action.
   - **Fix:** put `Players` rows in all three domains, in a FrontEnd UpdateDatabase.
4. **Removing a tech/civic crashes or breaks boosts and the AI.**
   - **Cause:** many tables reference the base IDs.
   - **Fix:** keep every ID, re-link prerequisites, rename by text only.
5. **Tree columns come out wrong.**
   - **Cause:** the column is the cost rank within the era.
   - **Fix:** costs must rise along each prerequisite chain. Check this offline against the stock database.
6. **The game freezes for good after a particular AI's turn** (turns 25 and 49 in two runs).
   - **Cause:** a male leader reused Catherine de Medici's ability. Its modifier `UNIQUE_LEADER_ADD_VISIBILITY`
     uses `SourceType = DIPLO_SOURCE_FEMALE_ONLY`, and the custom civ had **no female citizen names**, so the
     engine looped forever looking for one.
   - **Fix:** set that modifier argument to `DIPLO_SOURCE_ALL_NAMES` and give every civ full citizen-name lists
     (male and female, classic and modern).
   - **Method:** bisect abilities with **new** games. A loaded save keeps the modifiers attached at game start.
7. **Some of your text reverts to base names in game** (e.g. city-states, other `LOC_CITY_NAME_*`).
   - **Cause:** another enabled mod sets the same tags later. Yet (not) Another Maps Pack sets about 11k
     `LOC_CITY_NAME_*` at LoadOrder 2000.
   - **Fix:** load your text at a higher LoadOrder (5000). Check with `Locale.Lookup` in a test script.
8. **A cook step "succeeded" but the art never changed.**
   - **Cause:** the cooker's error was swallowed by `| tail` in a script.
   - **Fix:** check each mode's exit status and log. Also:
     - pass every needed `--pantry`;
     - don't put commas in PowerShell `-File` arguments;
     - PowerShell variables are case-insensitive, so a local `$extraArtDefs` overwrote the `-ExtraArtDefs`
       parameter.
9. **2D leaders stand on a black screen in diplomacy.**
   - **Cause:** there is no 3D scene; the fallback image is only as wide as the portrait.
   - **Fix:** a **2048x1024** fallback with the portrait centred on a dimmed civ landscape. The game centres it at
     3/4 screen width and scales it to screen height, so it reaches under the left panel.
   - **Cost:** 2.5x the texture memory when the DDS is uncompressed (14 leaders: 63 → 157 MB). Use BC compression
     if you can.
10. **Custom wonders render nearly black in game** while the Blender preview looks fine.
    - **Cause:** the AI textures' mean linear albedo was about 0.03–0.07. Base wonder textures are 0.15–0.33, and
      UV padding was black.
    - **Fix:** fill the padding, then level the albedo towards about 0.18 (capped gain).
    - **Check:** compare against the base wonder's own textures before cooking. Very dark sources (a black cliff)
      stay dark by design.
11. **Custom geometry cooks with flat/odd shading.**
    - **Cause:** the headless Granny export has no tangents, and the cooker copies the zeros.
    - **Fix:** add tangents and binormals (and a third UV set) to the `.fgx` vertex data after export.
12. **`ArtDef` element not found / "not in any pantry" for an expansion wonder.**
    - **Cause:** expansion/DLC wonders have no SDK asset.
    - **Fix:** stand in a base wonder's asset (Colosseum; Great Lighthouse for coastal ones) and keep the DLC
      artdef element.
13. **A coastal test wonder found "no valid plot".**
    - **Cause:** these wonders go on coast **water** tiles next to land, and a new city owns only one ring.
    - **Fix:** claim tiles in the test with `WorldBuilder.CityManager():SetPlotOwner`, and reveal them, or the
      camera looks at unexplored parchment.
14. **The end-game screen never opens during autoplay.**
    - **Cause:** the victory fires (`Events.TeamVictory`), but the menu does not show while the AI drives player 0.
    - **Fix:** in a test, force-end the local player's turn with `REASON = "UserForced"` instead.
15. **A text file stops parsing after a harmless edit.**
    - **Cause 1:** `--` inside an XML comment (generated from `# --- heading`).
    - **Cause 2:** writing `&lt;&lt;` into a source that is escaped again on generation; grammar markers like
      `{LOC_GRAMMAR_CIVNAME << {1_CivName}}` must end up as a single `&lt;&lt;`.
    - **Also:** some tags really have no `LOC_` prefix (e.g. `ADVISOR_LINE_LISTENER_*`).
16. **The paid asset budget burned on failures.**
    - **Cause:** several parallel agents shared one scratch folder and overwrote each other's batch scripts. About
      20 image-to-3D calls failed with HTTP 422 `file_download_error` on bad input URLs, and others were
      duplicated.
    - **Fix:** a single serial generator. At most one paid call per item, and only if its output is missing.
      Stop at the first API error. Price the batch and ask before spending. After that, 39 image-to-3D runs in a
      row had 0 failures.

## Assets
- **Leaders:** 14 leader portraits (painted, original designs) from `fal-ai/nano-banana-2`, cut out with
  `fal-ai/birefnet/v2`.
  - Head crops for circular icons.
  - Leader-select 748x1024 foregrounds.
  - 2048x1024 diplomacy images.
- **Civs:** 8 civ symbols, generated white-on-black, with alpha taken from luminance (gpt-image-2 refused a
  transparent background in the size needed). 8 civ landscapes (16:9) serve as loading, diplomacy and end-game
  backgrounds.
- **Wonders:**
  - 53 concept images (nano-banana-2): one isometric diorama per wonder on a round or hex plinth, neutral grey
    background, no text.
  - 52 GLBs from `fal-ai/trellis-2`, processed in Blender as in Build steps 6.
  - Weak results came from busy concepts (big trees, thin spires); one clear silhouette per concept works best.
- **Movies:** none are generated. The base victory movies are simply not played.
- **Records:** prompts and request ids are kept in each folder's `fal_manifest.jsonl`. The README discloses that
  the art is AI-generated.

## Cost and time
- About two long days of agent time across several context compactions.
- Each wonder takes about 2–4 minutes of Blender and cooker time, run one at a time because of memory use.
- **fal:**
  - Image-to-3D was the main spend: about 39 good runs plus the wasted ones in Gotcha 16.
  - 2D art was a few dozen image calls.
  - Check prices with `um fal price` before a batch.
- **Size:** the cooked mod is about 500 MB (wonder package ~225 MB, uncompressed leader images ~157 MB).

## Open questions
- **Leader portraits:** a few look too much like the film actors. Regenerate them for a public release, with
  prompts that steer away from actor likeness.
- **Texture size:** add BC1/BC3/BC7 compression to the DDS writer to shrink the BLPs.
- **Units:** still use base-game 3D models. The same headless `.fgx` route should work for units, but unit assets
  need rigs and an animation state graph; not tried.
- **Rulesets and languages:** only English and Gathering Storm were tested.


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

