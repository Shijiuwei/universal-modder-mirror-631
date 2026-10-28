---
kind: game
title: "Minecraft inside GTA V (passthrough mod)"
game: "Grand Theft Auto V"
games_also: ["Minecraft Java Edition"]
game_version: "GTA V Legacy (Steam, game build 3889, story mode) + ScriptHookV 3889.0 + ReShade 6.8.0; Minecraft Java 26.3 + Fabric"
platform: windows
engine: rage
route: passthrough
tools: ["ScriptHookV + ASI loader", "ReShade 6.8 add-on API", "Fabric Loader + Fabric API (Loom, JDK 25)", "MSVC", "Python (director, recorder)", "um win / um video"]
anti_cheat: "BattlEye guards GTA Online only: story mode, launched with BattlEye off (-nobattleye), never online"
status: working
agents: ["Claude Code (Opus 5.5)"]
humans: ["@rehan_shei"]
date: 2026-09-30
links: ["https://github.com/rehan-remade/universal-modder/tree/main/examples/minecraft-gta5-passthrough"]
tags: [mashup, passthrough, shared-memory, websocket, depth-compositing, reshade-addon, scripthookv, fabric, mixin, reprojection, camera-sync, director, prism-launcher, amd]
---

# Minecraft inside GTA V (passthrough mod)

> Real Minecraft Java 26.3 runs next to GTA V story mode and is drawn into GTA's frame: Steve walks, builds,
> flies an elytra over Los Santos, and his TNT, arrows and fireworks become GTA explosions and bullets, while
> Minecraft mobs fight the LAPD. GTA's camera drives Minecraft's. GTA's ground becomes invisible collision in
> Minecraft, and Minecraft's colour + depth are depth-tested against GTA's depth buffer. It runs in the real
> game (recorded for a demo video). Code: `examples/minecraft-gta5-passthrough`.

## Setup
- GTA V **Legacy** from Steam (appid 271590, game build 3889), story mode. ScriptHookV 3889.0 + its ASI
  loader. ReShade **6.8.0 with add-on support**, loaded through the ASI loader as `ReShade64.asi` (see
  Gotchas).
- Minecraft Java **26.3** with Fabric Loader + Fabric API, built with Loom on **JDK 25**. It runs from a
  separate launcher profile with its own game dir, so the player's own worlds and options are never touched.
- MSVC (Visual Studio Build Tools) for the ASI; Windows Python for the host test tools and the video
  director.
- One Windows PC with an NVIDIA GPU (agent working from WSL2). Running both games at once was fine.

## Route and why
**Passthrough:** two unmodified games run side by side, and a mod in each exchanges state. Alternatives
considered:
- **Reimplementing Minecraft inside GTA:** far too much work, and it would never feel like real Minecraft.
- **FiveM (the GTA multiplayer platform):** it needs a Cfx.re key and a server, and it may block ReShade
  add-ons.
- **Rendering Minecraft into a flat overlay:** it can't be occluded by GTA's buildings, and it doesn't
  sell the effect.

The design:
```
GTA V story mode                                   Minecraft 26.3 + Fabric
  MCPassthrough.asi (ScriptHookV script) -- WebSocket 127.0.0.1:25599 -->  camera / ground / keys / slot / commands
                                         <--------------------------------  explosions, arrow/firework hits, mob events
  ReShade add-on compositor  <-- named shared memory "Local\MCPassthroughFrame" --  world RGBA + depth, hand/HUD overlay
  MCPassthrough.fx: depth test vs GTA's (reversed-Z) depth, overlay on top
```

## How the game works (what we had to learn)
- **Coordinates:** 1 GTA metre = 1 block.
  - Position: GTA (x, y, z) → Minecraft (x, z + yOffset, −y).
  - Angles: Minecraft yaw = 180 − GTA heading; pitch = −pitch.
  - `yOffset` is picked so the ground under the player lands exactly on a block boundary. There's a re-level
    key, and an automatic re-level when nothing has been built nearby.
- **World sync:**
  - GTA's ground near the player is probed each tick (a floor-finding probe also works indoors) and sent as
    **barrier blocks**, so Steve collides with GTA's world.
  - Minecraft block changes go back as **invisible GTA collision props**, so peds and cars stop at your
    builds.
- **Frames:** each Minecraft frame exports colour + depth for the world and a separate overlay (hand + HUD)
  into shared memory. The compositor converts Minecraft's depth onto GTA's view axis and depth-tests it
  against GTA's buffer. So a lamppost correctly hides part of a Minecraft house, and the house hides the car
  behind it.
- **Latency:** Minecraft always renders slightly behind GTA's camera. The compositor re-projects:
  1. rotation first: each GTA pixel's view ray is rotated into the pose Minecraft rendered with;
  2. then full 6-DoF: ray-marched depth re-projection, so strafing and third-person swings don't make blocks
     swim or Steve drop out.
- **Lighting:** Minecraft is relit from a blurred copy of GTA's light, so a night street tints the blocks.
- **Interaction instead of guns:**
  - Steve holding GTA guns looked wrong (GTA's aim cam, hands and animations don't fit a blocky model).
  - What works is Minecraft weapons with GTA effects. The plugin traces Minecraft arrows and crossbow
    fireworks through GTA's world:
    - a firework hit → a GTA rocket explosion;
    - an arrow hit on a ped or car → a GTA bullet;
    - a sword swing ragdolls peds and launches cars.
- **Mobs vs police:**
  - Minecraft mobs and GTA peds exchange positions and hits (`peds` / `mobs` / `mobhit` / `mobdmg`
    messages).
  - GTA cops appear in Minecraft as invisible proxy entities that mobs can target, and mob hits become GTA
    damage.
  - A director op builds a police barricade that keeps the fight in frame.
- **The Nether:** walking through a lit portal turns the surrounding ground into the Nether, nether mobs pour
  out, GTA's sky goes red while its clock races to midnight, and GTA people and cars on lava or fire burn.
- **Director:** repeatable video takes are JSON shot lists run by the plugin. Ops include teleport, walk,
  face, view, look, time, weather, spawn peds/cars, explosions, `police {stars}`, pose-lag tuning and more.
  A recorder takes the GTA window plus game-only audio, and a cut script titles it in the Minecraft font.

## Build steps
1. **Minecraft side:** build the Fabric mod (`./gradlew build`, JDK 25) and put the jar next to Fabric API
   in the passthrough profile's `mods/`.
2. **GTA side:**
   1. Run `fetch_deps.sh` to get the ScriptHookV SDK, the ReShade add-on headers and `ReShade.fxh`, then
      `build.bat` (MSVC) builds the `.asi` + add-on.
   2. `install.sh` copies it into the GTA folder (and `--remove` undoes it).
   3. Install ReShade with add-on support **via the ASI loader**.
3. **Run:**
   1. Launch GTA with BattlEye off (`-nobattleye` in `args.txt`, or the launcher toggle) and enter
      **Story Mode**.
   2. Start Minecraft from the passthrough profile; it opens a void creative world by itself.
   3. Press F7 in GTA to toggle the passthrough.
4. **Without GTA:** `host/fakehost.py` flies Minecraft's camera and composites over a synthetic scene, and
   `gta/tests/fakegta.cpp` is a D3D11 app with a reversed-Z depth buffer, ReShade and the compositor, i.e.
   the whole GTA half except the natives.

## Verification
- **Alignment and occlusion, before GTA was even installed:**
  - `fakehost.py` renders known geometry and checks Minecraft's blocks land on it.
  - `fakegta.cpp` runs the real compositor and shader against a synthetic reversed-Z depth buffer, with a
    fast-shake mode and outlines of Minecraft's blocks drawn host-side.
- **Camera sync in the real game:** a measurement scene with a gold wall (Minecraft-only) against the
  skyline (GTA-only). The offset between them, frame by frame, found the one-frame pose lead and verified
  the fix.
- **Real game:** every feature was checked in story mode with the director's scripted shots, recorded, and
  reviewed frame by frame.
- **Not verified:** other GTA builds (Enhanced), ultrawide aspect ratios other than the ones used, Intel GPUs,
  and on AMD the in-game features one by one (see the second-PC run below).
- **Reproduced on a second PC (2026-10-09, Claude Code (Opus 5.5) with a different human):**
  - Setup: Windows 11 Pro 26200, **AMD Radeon RX 9060 XT**, GTA V Legacy `GTA5.exe` 1.0.3889.0 (Steam buildid
    24129523), ScriptHookV 3889.0.1158.13, ReShade 6.8.0.2155 add-on build, Minecraft 26.3 in **Prism
    Launcher** (Fabric Loader 0.19.5, Fabric API 0.161.0+26.3), portable Temurin JDK 25.0.4.1, VS 2022 Build
    Tools, Gradle 9.7.1 / Loom 1.18.3. The example code built unchanged on the first try.
  - `fakehost.py` (20 s orbit): 2354 frames over shared memory, one host frame behind; blocks land on the
    synthetic ground and occlude / are occluded by the host-only pillar correctly. So Minecraft's GL readback
    path works on AMD.
  - Real game: logs show the ASI loader loading both ASIs, ScriptHookV starting the script, ReShade compiling
    `MCPassthrough.fx` and the add-on reporting "connected to Minecraft's frame export", and Minecraft logging
    "host connected". The human played in story mode and reported it working; a window capture
    (`um win shot`) showed Minecraft's hand and hotbar composited over GTA. Not checked feature by feature
    (TNT, projectiles, mobs vs police, elytra, Nether), and no recording was made.

## Gotchas
1. **Colour readback returns garbage after a depth readback (Minecraft 26.3).**
   - **Cause:** the GL backend's depth `copyTextureToBuffer` sets the read FBO's read buffer to `GL_NONE`
     and never restores it.
   - **Fix:** a mixin on the GL command encoder restores the read buffer after the copy.
2. **Setup commands on join match nobody (`@a` is empty).**
   - **Cause:** Fabric's `ServerPlayConnectionEvents.JOIN` fires before the player is in the player list.
   - **Fix:** run them 10 ticks later.
3. **`fabric-installer -launcher microsoft_store` fails.**
   - **Cause:** the file `launcher_profiles_microsoft_store.json` doesn't exist.
   - **Fix:** add the profile to `launcher_profiles.json` by hand (back it up first). Give it its own
     `gameDir` so the player's worlds and options are untouched.
4. **ScriptHookV's download fails from scripts.**
   - **Cause:** dev-c.com rejects clients without browser headers.
   - **Fix:** send a browser User-Agent/Accept (the example's `fetch_deps.sh` does).
5. **The ReShade proxy DLL never loads in GTA.**
   - **Cause:** GTA loads the system `dxgi.dll` ahead of a proxy in its folder.
   - **Fix:** load ReShade through the ASI loader instead.
6. **Flicker, then a crash, when toggling the effect.**
   - **Cause:** the effect was toggled from inside the begin-effects callback.
   - **Fix:** gate it with a uniform (`McActive`) and never toggle techniques from that callback.
7. **Squashed HUD and skewed angles.**
   - **Cause:** Minecraft's window followed the monitor size (32:9), so the frame was squeezed into GTA's
     16:9.
   - **Fix:** size Minecraft's window to GTA's real backbuffer, and restore (un-maximize) it before
     resizing.
8. **Blocks lead the GTA picture by one frame, and explosion shake can't be stopped.**
   - **Cause:** the script reads GTA's camera for the frame being prepared, not the one on screen. GTA's
     explosion shake isn't reported by `IS_GAMEPLAY_CAM_SHAKING`, so `STOP_GAMEPLAY_CAM_SHAKING` can't
     cancel it.
   - **Fix:** re-project to the previous pose (a pose-lag of 1 frame in the compositor). Measure it with
     the gold-wall scene.
9. **The chase cam judders.**
   - **Cause:** timing used `GetTickCount` (16 ms steps) or Python `sleep` on Windows.
   - **Fix:** time everything with Minecraft's `System.nanoTime`, which is the same clock as
     `QueryPerformanceCounter` on Windows.
10. **Elytra glides are cancelled; a "dive" is really a free fall.**
    - **Cause:** with `noPhysics`, Minecraft's `onGround` never updates and can stick at true, so the server
      cancels every glide.
    - **Fix:** force `setOnGround(false)` in flight mode.
11. **A full-screen view of Steve's face.**
    - **Cause:** GTA pulls its camera in against walls, which puts Minecraft's camera inside Steve's head.
    - **Fix:** hide Steve when the camera is that close.
12. **After a restart, Steve glides into the void.**
    - **Cause:** Minecraft was closed mid-flight.
    - **Fix:** strip the elytra on join; auto-respawn on death with `keep_inventory`.
13. **Proxy entities become visible.**
    - **Cause:** Minecraft 26.3 resets an entity's Invisible flag on its first network sync unless it has an
      effect.
    - **Fix:** give proxies a hidden, infinite INVISIBILITY effect.
14. **Mobs ignore the proxies.**
    - **Cause:** invulnerable entities can never be targeted, and target goals test invisibility and line of
      sight.
    - **Fix:** leave proxies vulnerable but cancel their damage in a `hurtServer` mixin. Give target goals no
      invisibility or line-of-sight test.
15. **GTA's cinematic idle camera cuts in mid-take.**
    - **Cause:** it starts after about 30 s without input.
    - **Fix:** call `INVALIDATE_IDLE_CAM` every frame.
16. **The last Minecraft frame stays on screen over GTA's map.**
    - **Cause:** GTA's pause menu stops ScriptHookV scripts, so nothing clears the overlay.
    - **Fix used:** cut around it in the edit. Detecting the pause from outside the script and clearing the
      overlay would be the real fix; not built yet.
    - 2026-10-09, second PC: reproduced; a capture of GTA's pause-menu map still showed Minecraft's hand and
      hotbar on top.
17. **Hidden frozen "double" peds still take cop bullets.** `SetEntityVisible(false)` doesn't make them
    immune. Keep invisible stand-ins out of the line of fire.
18. **Walking off the Maze Bank Tower slides instead of dropping.** The tower's sides are sloped glass.
    **Fix:** take off with Space.
19. **Story Mode needs two clicks.** On the landing page the first click only highlights it.
20. **Don't auto-click GTA's landing page while the human is at the keyboard.**
    - **What happened:** the agent's click script focused GTA while the user was typing in another app.
      Their keystrokes landed on GTA's landing page, and it showed "ALERT: attempting to access GTA Online
      servers with an altered version".
    - ScriptHookV blocked it, and no session started. It was still a close call for the account.
    - **Fix:** auto-click only when the human is away (check `um win drive --proc GTA5 idle`). Otherwise ask
      them to click Story Mode. Keep BattlEye off so Online can't start at all.
21. **Guns held by Steve look bad.**
    - **Cause:** GTA's aim camera, hands and animations don't fit a blocky model.
    - **Fix:** use Minecraft weapons with GTA-side effects (see above).
22. **Prism Launcher: a hand-made instance folder is never listed (2026-10-09).**
    - **Cause:** Prism rescans `instances/` when the folder changes. If the instance folder (or its `mods/`)
      appears before `instance.cfg` is written, the rescan finds no instance and doesn't look again.
      `--launch "<name>"` then fails with "resolves to nothing" in Prism's log.
    - **Fix:** write `instance.cfg` and `mmc-pack.json` first, or rename the folder afterwards to trigger a
      rescan. Use a folder name without spaces as the instance ID (`--launch gta-passthrough`). Fabric adds two
      components to `mmc-pack.json`, after `net.minecraft` 26.3: `net.fabricmc.intermediary` 26.3 and
      `net.fabricmc.fabric-loader` 0.19.5.
      Prism's per-instance `JavaPath` can point at a portable JDK 25.
23. **PowerShell 5.1 writes a BOM.** `Set-Content -Encoding utf8` / `ConvertTo-Json | Set-Content` puts a BOM
    in `mmc-pack.json`. Write it with `[IO.File]::WriteAllText(..., UTF8Encoding($false))`.
24. **Prism can hang on "logging in with Microsoft account" during a scripted launch.** The token refresh
    needs the human (Accounts > Refresh, or sign in again). The agent must not handle the sign-in.
25. **The example's shell scripts fail in WSL with CRLF line endings.** A Windows checkout (or the plugin
    cache) can give `gradle.sh`, `gta/*.sh` and `mc/gradlew` CRLF endings, which WSL bash can't run. The
    repo's root `.gitattributes` now pins `*.sh`, `gradlew` and `bin/um` to LF (`mc/gradlew` has no `.sh`
    extension, so it needs its own line). A copy made before that still needs converting
    (`sed -i 's/\r$//'`).
26. **No WSL.** `fetch_deps.sh` runs as-is in Git Bash (curl + unzip); `build.bat` and `mc\gradlew.bat` run
    natively. `install.sh` needs `wslpath`: under Git Bash use `cygpath -u` and `/c/...` instead of
    `/mnt/c/...`, and point `BUILD` at `gta/build`. With WSL, a `PASSTHROUGH_WIN_DIR` inside the project
    (for example its `tools\`) keeps the build mirrors, JDK and venv in one place.
27. **Piping a script into `wsl.exe bash -s` stops after the first `cmd.exe` call.** `cmd.exe` (gradlew) reads
    the rest of the script from the shared stdin. Run the script from a file with `< /dev/null`.
28. **winget installs the C++ Build Tools unattended:**
    `winget install Microsoft.VisualStudio.2022.BuildTools --override "--quiet --wait --norestart --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"`
    (one UAC prompt). `build.bat` then finds vcvars through vswhere.

## Assets
None generated. Steve uses the classic skin while a host is attached. Titles in the demo use the Minecraft
font, with the video cut by the repo's director/recorder pipeline and `um video`.

## Cost and time
About two days of agent sessions (Claude Code, Opus 5.5) with one human directing and doing the clicks
the rules reserve for humans. The fake host and fake GTA let most of the work happen before the 125 GB
game finished installing.

## Open questions
- **Enhanced edition:** GTA V Enhanced (DX12) needs a different ReShade/compositor path.
- **Multiplayer Minecraft:** it works in principle, since it's just another client. It's untested.
- **Latency:** Minecraft could render at the predicted pose, so re-projection is only a fallback.
- **Check these with a real install before relying on the example.** See also
  `knowledge/techniques/frame-compositing-depth-and-pose-sync.md`.
  - **Pose lag:** the gotcha above says a lag of 1 frame, but the compositor defaults to 0 (a code comment
    says 0 measured best) and only the director's `poselag` op changes it. Record the value used per capture.
  - **`install.sh --remove`** deletes `ReShade.ini` only when install created it: install then writes a
    `.mcpassthrough_reshade_ini` marker. Tested on a folder with a dummy `GTA5.exe` (with and without a
    `ReShade.ini` already there); on a real install the marker was written, but `--remove` hasn't been run
    there yet. An install from before the marker keeps its `ReShade.ini` on removal. `ReShadePreset.ini` is
    still overwritten by install and deleted by `--remove`, even when the user had their own.
  - **Save helper:** it copies its two saves into every discovered profile.
  - **Cleanup scope:** detach forgets barrier tracking without removing the barriers; fighter cleanup removes
    every matching hostile or proxy in the Overworld, and hot-block cleanup can remove player-placed fire or
    lava; any invisible villager counts as a proxy.
  - **Frame reader:** it validates the magic but not every header size/capacity or the capture timestamps,
    and checks the slot sequence after the GPU upload, which can't undo a partly overwritten image.
  - **`ws_test.cpp`** passes when any received message contains `explosion`; it doesn't assert camera, depth,
    ground columns or reconnects.


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

