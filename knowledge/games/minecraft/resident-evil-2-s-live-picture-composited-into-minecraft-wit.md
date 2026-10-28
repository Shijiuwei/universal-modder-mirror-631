---
kind: game
title: "Resident Evil 2's live picture composited into Minecraft with depth"
game: "Minecraft"
games_also: ["Resident Evil 2"]
game_version: "Minecraft Java 26.3 + Fabric Loader 0.19.5 / Fabric API 0.161.0+26.3 (Loom 1.18.2, JDK 25); Resident Evil 2 (Steam app 883710, RE Engine, D3D12) + REFramework 1.5.9.1"
platform: windows
engine: java
route: passthrough
tools: ["REFramework 1.5.9.1 (native plugin + Lua probe)", "Fabric Loader 0.19.5 + Fabric API 0.161.0+26.3 (Loom 1.18.2, JDK 25)", "MSVC 14.44 (VS 2022 Build Tools)", "sheet preflight, fake_frame_producer.py, dump_live_frame.py", "um win (shot/drive)"]
anti_cheat: "none detected on re2.exe; both games single-player and offline only, never launched online"
status: working
agents: ["OpenCode (mimo-v2.6-flash)"]
humans: []
date: 2026-10-06
links: ["https://github.com/praydog/REFramework", "https://github.com/rehan-remade/universal-modder/tree/main/examples/minecraft-gta5-passthrough"]
tags: [mashup, passthrough, shared-memory, seqlock, depth-compositing, d3d12, frame-capture, reframework, fabric, mixin, reversed-z, camera-mirror]
---

# Resident Evil 2's live picture composited into Minecraft with depth

> Minecraft Java 26.3 is the visible game: Resident Evil 2's live rendered picture (colour **and** linear
> depth in metres) is drawn into Minecraft's view and depth-tested against Minecraft's own depth buffer, so
> Minecraft blocks occlude RE2's world and RE2's world shows through Minecraft's sky and gaps. RE2 keeps
> camera and input authority (Leon drives; his view is mirrored into Minecraft's camera). It runs in the
> real games: a two-game session verified the F8 fixture stone occluding RE2's picture and RE2's night sky
> replacing Minecraft's day sky through the probes and screenshots, with Minecraft's clouds and terrain
> still winning where nearer. Local prototype, not published.

## Setup
- **Host PC:** Windows 11 build 26200, NVIDIA driver 616.56 (GL 3.3.0 NVIDIA),
  `d3d12.dll` 10.0.26100.9278, `D3D12SDKLayers.dll` absent (no D3D12 debug layer - see the technique note).
- **Resident Evil 2:** Steam app 883710 at `...\steamapps\common\RESIDENT EVIL 2  BIOHAZARD RE2`, RE Engine,
  `re2.exe` x64, always DX12 (`Capability/TargetPlatform/PCDXver=DirectX12` in `re2_config.ini`), no
  anti-cheat detected. **REFramework v1.5.9.1** (official `RE2.zip`, SHA-256 verified) installed into the
  game folder with the user's approval; saves backed up first (`um backup`, 19 files / 56 MB zip).
  `re2_config.ini` backed up next to itself as `re2_config.ini.dx12.bak` and restored after the final run.
- **Minecraft:** Java Edition **26.3**, Fabric Loader **0.19.5**, Fabric API **0.161.0+26.3**, Loom
  **1.18.2**, JDK **25.0.4.1**, development client with its own run dir and an isolated single-player world
  (the player's normal profile and worlds are never touched). Java 25 FFM needs
  `--enable-native-access=ALL-UNNAMED` (from the sheet) for the Win32 shared-memory calls.
- **Bridge:** one MSVC-built native plugin (`leon_bridge_host.dll`, VS 2022 Build Tools, MSVC 14.44,
  `/std:c++20`) in `{game}/reframework/plugins`, one Lua probe in `{game}/reframework/autorun`, and a
  Fabric client mod. Design sheet (`design/sheets/vertical-slice.json`) is the source of truth:
  `tools/preflight_sheets.py --generate` emits the Java protocol constants **and** the C++ wire-struct
  header from it and validates every offset/width, so Java and C++ cannot drift.
- Credit: [REFramework](https://iuee.yx-sf.com/gongju/restaurant-20376348.html) (praydog) for the whole RE2-side hook
  surface; this repo's `examples/minecraft-gta5-passthrough` (MCPassthrough.fx depth maths, FrameExporter
  seqlock layout, GameRendererMixin injection points) was the port template - this note mirrors it with the
  games' roles reversed.

## Route and why
**Passthrough:** two unmodified games run side by side; a mod in each exchanges state over local shared
memory. Direction decided by the user: Minecraft visible, RE2 the picture source and the input/camera
authority (so the earlier camera-mirror slice was reused untouched). Alternatives considered:
- **ReShade compositor inside RE2** (archived design under `design/archive/`): wrong direction once RE2
  had to be the guest picture, and it would have shipped as an RE2-side screen overlay of Minecraft.
- **Wiring only `REFrameworkRendererData`'s D3D11 device:** our first capture path was D3D11-only, but RE2
  runs D3D12 - that path never fired once (see Gotcha 9). The SDK isn't D3D11-only (its example plugin
  branches on `renderer_type` for D3D11 and D3D12); the plugin now queries the swapchain.
- **A screen-in-world stepping stone** (RE2 on a monitor block first): skipped; the user wanted the full
  depth-aware composite as the first picture milestone.
- **Reimplementing either game:** never sensible for a picture-in-picture effect.

## How the game works (what we had to learn)
- **RE2 capture (D3D12).** Colour comes from the swapchain backbuffer (1920x1080, format 28 = `R8G8B8A8`
  sRGB); depth comes from a device-side `CreateDepthStencilView` wrap that keeps a DSV table and prefers the
  backbuffer-sized handle. RE2's depth renders at **2880x1620 = exactly 1.5x supersample** of the colour, so
  depth is accepted under an exact-match-or-uniform-supersample rule and nearest-mapped to picture
  dimensions. Three hardcoded command-list/device vtable slots do the hooking:
  **`ResourceBarrier`=26, `OMSetRenderTargets`=46, `CreateDepthStencilView`=21** (fixed by the D3D12 COM
  ABI, so RE2 updates can't move them).
- **Depth maths.** RE Engine is reversed-Z (1.0 = near, 0.0 = far); stored depth linearises to metres with
  `z = n*f / (n + d*(f - n))` using the clip snapshot the pose writer publishes (probe: near 0.01, far
  3000, fov 74.8). Published frames carry metres, flags `1|4` (bottom-up rows, reversed Z).
- **Transport.** Windows named mapping `Local\LeonRe2FrameV1` (199,069,696 bytes): 4096-byte header, three
  128-byte descriptors at `256 + 128*slot`, two RGBA8/r32f layers up to 3840x2160, seqlock (sequence odd
  while writing, reader re-checks after use), stale limit 250 ms, future allowance 50 ms. The 200-byte
  camera pose record uses the same seqlock idea on a second mapping at ~52 Hz. All constants are generated
  from the sheet - the fake producer and the dump tool read the same sheet so they cannot drift from the
  reader.
- **Minecraft internals (26.3).** The composite injects at `GameRenderer.renderLevel` just before
  `render3dHud` (world complete, hand not yet drawn). `GameRenderer.render()` works on `mainRenderTarget`'s
  **offscreen** textures; `GlSurface.blitFromTexture -> present()` rewrites default FBO 0 *after* the
  inject, so the pass draws into a private FBO built over main's colour+depth via `GlTexture.glId()`
  (identity-checked, recreated when the window resize swaps textures, 5 s retry, fail-closed).
  Minecraft 26.3's projection is ZERO_TO_ONE reversed-Z (`b = -1`, `func=GEQUAL`, ortho when `b == 0`); a
  fragment distance maps to window space with `window(dist) = (c - a*dist) / (d - b*dist)` - no `0.5(·+1)`
  GL shift. Minecraft's far clip is **1024 m** against RE2's real 3000 m (Gotcha 2). The hand is suppressed
  with a cancellable `renderItemInHand` HEAD mixin while the pass is active; RE2's frame carries Leon and
  his HUD.
- **Input.** Minecraft 26.3 key bindings use **SDL scancodes** (`KEY_F8 = 65`), not GLFW codes (Gotcha 5).
  The F8 fixture places one `minecraft:stone` three units ahead of the mirrored camera, refuses without a
  fresh RE2 pose, and never replaces a non-air block.

## Build steps
```powershell
# 1. Sheet -> generated Java + C++ protocol (validates cells, offsets, widths)
uv run --offline --with pillow tools\preflight_sheets.py --generate     # 63 cells, 0 unresolved

# 2. RE2 side: build the plugin with MSVC x64 (/std:c++20), then - WITH THE USER'S APPROVAL - install
#    hash-verified into {RE2}/reframework/plugins (+ the Lua probe in reframework/autorun).
#    Backup first: um backup (Steam remote saves), copy re2_config.ini -> re2_config.ini.dx12.bak

# 3. Minecraft side (development client, isolated run dir)
$env:JAVA_HOME = "C:\Program Files\Java\jdk-25.0.4.1"
.\gradlew.bat build --no-daemon
.\gradlew.bat runClient --no-daemon

# 4. Iteration tools (no RE2 needed for the first three)
uv run --offline tools\fake_frame_producer.py                 # synthetic colour+depth writer; stop it
                                                              # by exact PID before opening live RE2
uv run --offline --with pillow --with numpy tools\dump_live_frame.py   # live slot -> PNGs, stats, RANSAC
uv run --offline --project <universal-modder> um win shot --exe java out.png
uv run --offline --project <universal-modder> um win drive --proc java "focus" "key 0x1B"  # close MC's pause menu
```

## Verification
- **Harnesses first (no games):** Java decoder self-test 23/23 checks (torn/stale/future/malformed pose
  records); frame protocol self-test 41/41 (25 rejection paths); fake producer cross-process run accepted
  1409 frames / 0 torn / 0 rejected at ~29.6 fps, then correctly rejected as stale after the writer exited.
- **Capture, live RE2:** first-capture log line with geometry and `invalid=0`; 30 s heartbeats with
  `frame==total` (zero rejects); depth range 0.11..3000.00 m across a real session. Oracles: statue-room
  colour/depth silhouette pixel-aligned (Leon nearest ~1.34 m); eyeballed wall 2.6-2.7 m predicted vs
  ~2.9-3.1 m measured (wall angled); open-sky aim puts 49.0% of pixels at exactly 3000.000 m (an inverted
  reversed-Z would read ~0.01 m); floor plane fit rms 3.6 cm agreeing with the game's own camera height
  within 1.5 cm.
- **Composite, fake producer then live:** with the real depth test (`func=GEQUAL`, `glError=0x0`, private
  `main FBO` attached), the fake checkerboard fills sky/gaps and vanilla clouds/terrain occlude it; the
  stale path reverts to vanilla; the hand stays hidden. Live two-game run: pose connected and calibrated,
  frames ~30 fps, fov tracking the probe, F8 stones placed at `(47,78,-5)` and `(47,78,-6)`; sky probes
  `before=184,210,255` (Minecraft day blue) `after=15,19,23 / 13,17,20 / 7,8,8` (RE2 night), centre stone
  probe `68,68,69` unchanged through the draw (Minecraft wins), MC's day clouds and lit grass winning
  where nearer, HUD on top, no hand; window resize re-attached the FBO and kept working.
- **Not verified:** third-person avatar hand policy (out of the first-person slice); audio of either game
  in the mix; whether the sky-flag `ModifyArg` actually suppresses Minecraft's sky (the beyond-clip floor
  made it moot - probes show MC's sky colour present *before* our draw); long-session (>1 h) stability;
  other GPUs/drivers (one NVIDIA 616.56 machine); no showcase video; no release (no one-click install path
  for the Fabric side yet - stays local/draft-only).

## Gotchas
1. **Composite draws but the screen never changes.** Probes right after the draw show the right colours on
   `fbo=0`, the picture is still vanilla. **Cause:** `GameRenderer.render()` writes `mainRenderTarget`'s
   offscreen textures and `GlSurface.blitFromTexture -> present()` rewrites default FBO 0 *after* the
   `render3dHud` inject, erasing the draw every frame. **Fix:** build a private FBO over main's colour+depth
   textures (`GlTexture.glId()`), identity-check each frame, recreate on texture swap (resize), retry 5 s,
   fail closed.
2. **Guest sky never wins: open-sky pixels keep Minecraft's sky.** **Cause:** RE2's real depth reaches
   3000 m but Minecraft's far clip is 1024 m; `window(dist)` extrapolates *negative* past the clip (and can
   NaN), so every beyond-clip pixel lost `GEQUAL` against Minecraft's cleared/sky depth. The fake
   producer's 700 m sky sat inside the clip and hid this for days. **Fix:** floor the guest depth at
   `1e-6` (the far end of Minecraft's own clip) in the fragment shader - ordering stays correct for every
   distance Minecraft can render (view distance 16), because no host geometry exists past the clip.
3. **Depth comparison looks inverted/garbage with the textbook GL formula.** **Cause:** 26.3 uses
   ZERO_TO_ONE reversed-Z: the logged matrix decodes `a=4.883051E-5 b=-1.0 c=0.05000244 d=-0.0` (near 0.05
   -> 1.0, far 1024 -> 0), so the classic `0.5(·+1)` shift is wrong. **Fix:**
   `window(dist) = (c - a*dist) / (d - b*dist)`, `func=GEQUAL`, ortho guard `b == 0`; print `a,b,c,d` plus
   `fdNear/fdFar` from the shader once to decode any future matrix change.
4. **Camera publishes fov=2.0 and near=2.0 for 27 minutes while the probe reads 74.8.** **Cause:**
   REFramework's `InvokeRet` is one union (bytes[128] with `float f` / `double d` aliasing) and float-returning
   getters are widened to double by the invoke wrapper; reading the *float* view gets the low 4 bytes of the
   double. Values whose float bits end in `010` decode to exactly `2.0` - inside the plausible-magnitude
   window - and RE2's `74.8` (`0x4295999A`) and `0.01` (`0x3C23D70A`) both do; 7 of 8 bit patterns fall
   outside the window and happened to read correctly. **Fix:** read the double view first (matching
   REFramework's own parser), keep the float view only as fallback for genuinely raw-float returns (whose
   double view is denormal and fails the window).
5. **F8 does nothing although options.txt shows a binding.** **Cause:** Minecraft 26.3 input uses SDL
   scancodes (`InputConstants.KEY_F8 = 65`), while the saved binding and GLFW say `key.keyboard.297`;
   vanilla F-keys worked only because they use their own scancode constants (F3 = 60). **Fix:** sheet
   `triggerKeyCode` 297 -> 65 (source of truth; preflight regenerates the protocol), patch the saved
   binding, and keep a raw-poll fallback trigger while diagnosing. Disassemble the Loom-mapped jar to read
   the real constants.
6. **Minecraft boots with default options and a `NumberFormatException` on `key.keyboard.g`.** **Cause:**
   PowerShell 5.1 `Set-Content -Encoding UTF8` wrote a UTF-8 BOM; MC failed to parse `version:`, ran the
   legacy key datafixer, and misread the named format. **Fix:** never BOM-write Minecraft config files from
   PS 5.1 - no-BOM or line-scoped edits only.
7. **Writer runs for 9 minutes, guest never calibrates, no reason is logged anywhere.** **Cause:** publish
   gates early-returned silently (one-shot latch, silent world-matrix/forward early-outs) so the reader
   only ever saw "no record". **Fix:** log every blocked-publish reason *on change*, throttled to 5 s, with
   per-getter status and raw values; give the reader an `explainRejection` that re-reads seqlock misses and
   names the exact rejection (torn, timestamp, clip range, short forward).
8. **`E_INVALIDARG` creating what looks like a valid D3D12 readback texture.** **Cause:** D3D12 forbids
   textures on UPLOAD/READBACK heaps, and a `ROW_MAJOR` texture is only legal on a cross-adapter shared
   heap. **Fix:** buffer-based readback (`GetCopyableFootprints` + `CopyTextureRegion` into a READBACK
   buffer) - full rule matrix in the companion technique note. With no debug layer installed, the HRESULT
   was the only diagnostic: prove the rule matrix in a tiny harness against **WARP** first (it matched
   NVIDIA exactly, i.e. spec, not driver).
9. **Our D3D11 capture path never fires.** **Cause:** the first plugin only wired D3D11, but RE2 runs D3D12.
   The SDK isn't D3D11-only: REFramework's example plugin branches on `renderer_type` for D3D11 and D3D12,
   and `PluginLoader::init_d3d_pointers` sets it from the active hook every frame. **Fix:** branch on
   `REFrameworkRendererData::renderer_type` (as the example plugin does) or query the swapchain; on D3D12
   hook `ResourceBarrier=26`, `OMSetRenderTargets=46`, `CreateDepthStencilView=21` (fixed by the D3D12 COM
   ABI), track DSVs in a table, refuse MSAA/unsupported depth with bounded logs.
10. **First-person hand renders on top of the composite.** **Cause:** the inject point precedes
    `renderItemInHand`. **Fix:** a single cancellable `@Inject(method="renderItemInHand", at=HEAD)` mixin
    that cancels while the composite is active (screen effects keep running; third-person avatar untested).
11. **RE2 paused/minimized would freeze a ghost frame over Minecraft.** **Cause:** no new guest frames.
    **Fix (by design):** fail closed past the 250 ms stale limit - Minecraft renders completely vanilla,
    and recovers when frames resume; keep the stale reason logged so silence isn't misread as failure.
12. **Screenshots keep catching Minecraft's pause menu.** **Cause:** Minecraft opens its game menu within
    ~1 s whenever its window loses focus during these two-game runs (every switch away to RE2 or chat does
    it). **Fix:** focus the Minecraft window and send Esc in one `um win drive` invocation (only with the
    user's approval - input automation needs asking), then shoot immediately.
13. **`GL_COLOR_ATTACHMENT0` does not compile.** **Cause:** it lives in `GL30`, not `GL11`. **Fix:**
    reference `GL30`; keep a `glGetError` probe after every composite draw anyway.
14. **F8 on an occupied cell logs nothing at all.** **Cause:** the never-overwrite refusal path is silent
    (no placement happened - the safety rule holds - but the user can't tell a refusal from a dead key).
    **Fix:** add a bounded refusal log line (noted, not yet done).
15. **Depth picture is 1.5x bigger than colour.** **Cause:** RE2 renders depth at 2880x1620 while the
    swapchain is 1920x1080. **Fix:** accept exact-match or uniform-supersample ratios only, nearest-map depth
    to picture dimensions, and keep provenance flags from the source storage (values stay metres).

## Assets
No game assets are shipped or republished: RE2's picture is read at runtime and never leaves the machine,
screenshots stay outside the repo, and the only "art" is the synthetic checkerboard/depth ramp from
`tools/fake_frame_producer.py` plus F8-placed stones in the isolated dev world.

## Cost and time
Two calendar days (2026-10-05 to 2026-10-06), six logged combined game runs plus tool-only iterations, one
56 MB save backup, no API spend (everything local), no new hardware. Most time went to three root causes
(FBO present-order, InvokeRet union, far-clip extrapolation) that all hid behind "it looks fine but isn't".

## Open questions
- Third-person avatar policy (out of the first-person slice; presumably suppress like the hand).
- Whether the sky-flag `ModifyArg` is needed at all now that beyond-clip guest depth is floored.
- Audio mixing between the two games (RE2's audio is the natural choice but nothing is wired).
- One-click/launch story: there's no one-click install path for the Fabric side yet; the project stays
  local/draft-only until then.
- A showcase video (`um video` contact sheets / EDL) and a `media/` thumbnail for the note.


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

