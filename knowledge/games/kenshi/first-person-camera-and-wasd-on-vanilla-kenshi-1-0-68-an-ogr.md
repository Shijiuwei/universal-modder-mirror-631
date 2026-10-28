---
kind: game
title: 'First-person camera and WASD on vanilla Kenshi 1.0.68: an OGRE plugin plus signature-scanned game internals'
game: Kenshi
games_also: []
game_version: '1.0.68 x64 (Newland), Steam build 13871665, kenshi_x64.exe md5 8a03c256f0da1555d9cceb939b41530a'
platform: proton
engine: native
route: native-hook
tools:
- clang + lld-link + llvm-dlltool (freestanding x86_64-pc-windows-msvc DLL, no Windows SDK)
- Wine 11.18 (offline harness), Proton 11.0
- objdump, winedump
- KenshiLib headers (reference only)
- python-evdev over /dev/uinput (test input)
anti_cheat: none
status: working
agents:
- Claude Code (Opus 5.5)
- Codex CLI 0.155
humans: []
date: '2026-10-10'
links:
- https://github.com/BFrizzleFoShizzle/KenshiLib
- https://github.com/BFrizzleFoShizzle/RE_Kenshi
tags:
- ogre
- first-person
- camera
- signature-scan
- rtti
- proton
- two-agents
---
# First-person camera and WASD on vanilla Kenshi 1.0.68: an OGRE plugin plus signature-scanned game internals

> A plugin DLL that puts Kenshi's camera at the eyes of the selected character and walks that character with
> W A S D. It is loaded by OGRE itself from `Plugins_x64.cfg`, drives the camera through `OgreMain_x64.dll`
> exports only, and reaches the selected character and the game's own movement code by scanning the 1.0.68
> exe for signatures. It runs in the real game under Proton: toggling, mouse look, walking, running, wall
> collisions and right-click orders from the first-person view were all exercised in a loaded save, with
> positions read back from the plugin's log. Two agents built it: Claude Code did the plugin and the in-game
> tests, Codex did the static reverse-engineering of the exe and a safety review.

## Setup
- Kenshi 1.0.68 x64 from Steam, Linux (CachyOS, Hyprland/Wayland), Proton 11.0-100. No mods, no RE_Kenshi.
- The game (`kenshi_x64.exe`) is native C++ built with MSVC 2010 (`MSVCR100`, `MSVCP100`, `mfc100u`); only
  the editor, `forgotten construction set.exe`, is .NET (older `um scan` builds reported the whole game as
  .NET because of it). For the data/FCS routes, load order and the RE_Kenshi route on 1.0.65, see the
  companion note [`kenshi-1-0-65-native-ogre-recon.md`](kenshi-1-0-65-native-ogre-recon.md), and for
  `.mod` files written from code, the [generated data mods note](kenshi-generated-mods.md); this note
  covers a native plugin on vanilla 1.0.68 without RE_Kenshi.
- Engine DLLs next to the exe: `OgreMain_x64.dll` (OGRE 2.0.0 "Tindalos", 9333 exports, patched by Lo-Fi),
  `RenderSystem_Direct3D11_x64.dll`, `MyGUIEngine_x64.dll`, `OIS64.dll`, PhysX, Havok.
- Saves live in the Proton prefix: `compatdata/233860/pfx/drive_c/users/steamuser/AppData/Local/kenshi/save`
  (the `save` folder in the install only holds templates).
- Build host needs only `clang`, `lld-link`, `llvm-dlltool`. The DLL is freestanding C: import libraries
  for `kernel32`, `user32` and `msvcrt` are generated from `.def` lists, so no Windows SDK or mingw.

## Route and why
`native-hook`, in two layers.

1. **Camera through OGRE only.** OGRE loads every `Plugin=<name>` line of `Plugins_x64.cfg` and calls its
   exported `dllStartPlugin`. That is how RE_Kenshi gets in too. No proxy DLL and no `WINEDLLOVERRIDES`.
   Everything the camera needs is an export of `OgreMain_x64.dll`, resolved with `GetProcAddress` on the
   MSVC-mangled name, so this layer does not depend on the game build.
2. **Character and movement through the exe.** There is no way to know which character is selected, or to walk
   it, from OGRE. The community's map of Kenshi's classes (KenshiLib) has addresses for **1.0.65**; RE_Kenshi
   is based on 1.0.65, and for 1.0.68 its installer creates a compatible executable in the `RE_Kenshi` folder
   (RE_Kenshi README). Instead, the dozen addresses this mod needs were found in the 1.0.68 exe by static
   analysis and are located at load time by byte signature. If any signature fails to match exactly once the
   layer switches itself off and the camera still works.

Not taken: an RE_Kenshi plugin (needs the installer's compatible exe, a Windows GUI installer run through
Proton, and the VS2010 compiler for the STL ABI), and inline hooks (import-table and vtable slots were
enough).

## How the game works (what we had to learn)
**Frame loop.** The exe imports `Root::startRendering` and `add/removeFrameListener`, not `renderOneFrame`:
the game runs inside OGRE frame listeners. Lo-Fi added a virtual to `Ogre::FrameListener`:
`[0] frameStarted [1] frameRenderingQueued [2] framePostAnimationUpdates() [3] frameEnded [4] dtor`.

**Where a camera override has to go.** Inside `SceneManager::_cullPhase01` (vtable slot 86) the order is:
frustum culling, then `updateAllOldAnimations` (poses the *visible* v1 skeletons on worker threads and fires
`framePostAnimationUpdates`), then the render queue. That animation step is Lo-Fi's too: stock ogre-next
`v2-0` only culls in `_cullPhase01`. Bones are only fresh after culling. So the camera is placed twice per
pass: in a vtable hook on `_cullPhase01` before culling (last frame's pose, so the engine culls for the eyes)
and again in `framePostAnimationUpdates` (this frame's pose, so the view does not trail the head).

**Camera rig.** Camera `mainCamera` (local position 0, local orientation 180° about Y) hangs under node
`camera_node`, under node `camera_center` (the orbit pivot), under `Ogre/SceneRoot0`. The scene root is at
minus the world position of the play area (floating origin): render position = world position + root
position. Near clip 5, vertical FOV 45°. The game rewrites the node transforms every frame but never the
camera's own local transform, so overriding the camera's local position and orientation holds, and writing
the saved values back restores the normal camera exactly. Kenshi keeps the pivot position only in the
`camera_center` node: writing that node each frame keeps the pivot (and with it streaming and sound) on
the character after W A S D has broken the game's follow mode.

**Picking reads the OGRE camera.** A right-click while the camera is overridden orders the character to
what the overridden view shows. Point-and-click therefore keeps working in first person for free.

**Characters in the scene.** One character is many `Ogre::Entity` objects sharing a skeleton (body
`human_male.mesh_morph_1`, hair, clothes). Bones are 3ds Max Biped names (`Bip01 Head`, `Bip01 HeadNub`,
`Bip01 Neck`...). Scale is about 10.8 units per metre; Head→HeadNub is 2.165 units. All entities can be
walked without touching STL containers through `SceneManager::_getEntityMemoryManager(SCENE_DYNAMIC)`,
`ObjectMemoryManager::getNumRenderQueues` and `getFirstObjectData`, which fills a struct of parallel
arrays (`mOwner` is the array of `MovableObject*`); filter by the exported `Entity` vtable.

**Hiding the head.** Lo-Fi's `OldBone::setBoneSize` (the body-proportion hook) at 0.01 on `Bip01 Head`
removes head, hair and beard. The game resets bone sizes itself, so set it every frame and restore nothing.

**Mouse.** While Kenshi rotates its camera it holds the cursor still, so cursor deltas are zero. The game's
`OIS::Mouse` can be captured with an import hook on `OIS::InputManager::createInputObject` (type 2). With
MSVC 2010 layouts the mouse state is at object `+0x48`: `X.rel` `+0x58`, `Y.rel` `+0x68`, buttons `+0x80`.

**Game objects in 1.0.68** (RVAs for the md5 above; names from KenshiLib's headers, offsets re-checked in
this binary's code):
- `GameWorld` is a static object at `0x2134110`; `+0x580` is `PlayerInterface*`.
- `PlayerInterface`: selected-set element count `+0x228`, focus `hand` `+0xF0`. `hand::getCharacter`
  (`0x79D410`) turns it into a `Character*`.
- `Character`: `+0x448` AnimationClass (`+0xA8` is the body `Ogre::Entity*`), `+0x640` CharMovement.
  `getPosition` at `0x5CE680`. Live objects are `CharacterHuman` / `CharacterAnimal`.
- `CharMovement::setDirectMovement(const Vector3&, float)` at `0x333300`: sets mode 2 (`+0x378`), stores
  the direction at `+0x38C` and `sqrt(limit)` at `+0x398`. `setMovementMode` at `0x65F110`. Feet position
  `+0xC4`.
- `Character::playerMoveOrderDefault(Building*, RootObject*, const Vector3&)`: vtable slot `0x318`, body
  `0x5D22B0` for all three character classes. It is what a right-click on the ground ends in.
- `CameraClass` has no virtuals and no RTTI name in this exe; `PlayerInterface +0x30` points at it.

**What direct movement does** (measured in the game): the length of the direction vector is a fraction of
the character's own top speed (0.1 → 7 u/s, 0.3 → 22 u/s, 1.0 → 70 u/s for a starting character). The
game turns and animates the character, follows the terrain and stops at walls.

## Build steps
1. Write the DLL as freestanding C for `x86_64-pc-windows-msvc`: `-ffreestanding -fms-extensions
   -mno-stack-arg-probe`, link with `lld-link /dll /noentry /nodefaultlib` against import libraries made by
   `llvm-dlltool` from `.def` files. Define `_fltused`. Export `dllStartPlugin` and `dllStopPlugin`.
2. In `dllStartPlugin`: resolve the OGRE exports by mangled name, `Root::getSingletonPtr`,
   `Root::addFrameListener` with a hand-built vtable in the order above.
3. Patch the exe's import-table slots for `Root::createSceneManager` (capture scene managers and install
   the `_cullPhase01` vtable hook; find the slot by scanning the exported `??_7SceneManager@Ogre@@6B@` for
   the exported function), `Root::destroySceneManager`, and OIS `createInputObject`/`destroyInputObject`.
4. Find `mainCamera` by name in `SceneManager::getCameras()` (a `std::vector`: begin and end pointers at
   `+0`/`+8`). Each pass: bone position in entity space (`OldNode::_getDerivedPosition`) times
   `MovableObject::_getParentNodeFullTransform`, add the eye offset, convert to the camera's parent space,
   `Camera::setPosition` / `setOrientation`.
5. Scan `.text` of the main module for the signatures, derive the addresses, check each lands in the right
   section, run an RTTI self-check, and only then enable W A S D.
6. Copy the DLL next to `kenshi_x64.exe` and append `Plugin=KenshiFP` (CRLF) to `Plugins_x64.cfg`.

## Verification
- **Offline harness, no game process.** A small Wine host loads the game's own `OgreMain_x64.dll`,
  constructs `Ogre::Root` with a `plugins.cfg` naming the plugin (so OGRE loads it the way the game does),
  creates a scene manager, a `mainCamera` and an entity from the game's `human_male.mesh`, and drives the
  plugin through a command file. It proved the string and vector layouts, the listener vtable, hidden
  return pointers, the entity walk, the bone math and unload with hooks removed before the game ever
  loaded the DLL. A second host maps the real exe as an image and runs the same scanner code: twelve
  signatures match once at the expected RVAs.
- **In the game.** Positions and state were read from the plugin's log through a command file the plugin
  polls (`on`, `off`, `yaw`, `walk`, `dump`), with screenshots at each step. Checked: F6 on/off restores the
  camera to the same position; 200 px of mouse travel turns 34.4° at 0.003 rad/count; W, D and Shift+W
  through real key events move the character along the view direction; it holds position on release; it
  stops at a wall; a right-click on the ground walks it along the first-person view (direction error under
  1°); 217 FPS in a town with 645 entities; no fault over about 15 minutes in first person.
- **Final build**, in the game: W and Shift+W through real key events, hold on release, a quickload (F9)
  with the plugin loaded (it re-attaches to the reloaded character), and two scripted recorded takes with
  no fault logged.
- **Not verified:** the plugin's unload path in the real game. Closing the game window ends the process
  without an OGRE shutdown (`dllStopPlugin` is never called), and the quit confirmation in the Esc menu
  ignored synthetic clicks, so unload is covered by the offline harness only. Also not exercised: loading
  a save while first person is on, switching between several squad members, combat, animals, other game
  builds, and running next to RE_Kenshi (or on the compatible exe its installer creates).

## Gotchas
1. **Symptom.** A frame listener written from the OGRE headers that ship with KenshiLib has `frameEnded`
   in the wrong slot. **Cause:** Lo-Fi inserted `framePostAnimationUpdates` at slot 2; the bundled headers
   are stock OGRE 2.0. **Fix:** read the order from the exported vtable `??_7FrameListener@Ogre@@6B@` and
   match slots to exported function addresses.
2. **Symptom.** With the camera set only in `framePostAnimationUpdates`, the view would be right but
   objects go missing at the edges. **Cause:** that callback fires after the frustum culling of the same
   pass. **Fix:** also set it before culling, from a vtable hook on `SceneManager::_cullPhase01`.
3. **Symptom.** The signature scan finds the `Character` vtable, yet every live character is refused.
   **Cause:** live objects are `CharacterHuman` and their animation is `AnimationClassHuman`; an exact
   vtable compare never matches. **Fix:** walk MSVC RTTI (vtable[-1] → complete object locator → class
   hierarchy → base class names) and accept anything derived from the class.
4. **Symptom.** The same RTTI walk fails in the offline scan test. **Cause:** an exe mapped with
   `LoadLibraryEx(..., DONT_RESOLVE_DLL_REFERENCES)` under Wine is not relocated, so `vtable[-1]` holds an
   address for the preferred base. **Fix:** if the pointer is outside the image, translate it with the
   header's `ImageBase`.
5. **Symptom.** Release W and the character sprints back to where it started. **Cause:** leaving
   `MOVE_DIRECTION` with `setMovementMode(0)` hands it back to its previous order. **Fix:** follow the stop
   with `playerMoveOrderDefault` to the position it reached, and only when the stop itself ended direct
   movement, so a newer order is not overwritten.
6. **Symptom.** Mouse look does nothing while the rotate button is held. **Cause:** Kenshi freezes the
   cursor while rotating. **Fix:** read relative motion from the game's OIS mouse object. Related: in two
   recorded takes the view snapped to the ground the instant the button went down (one large relative
   move, cause not pinned down, did not recur); ignoring motion for 100 ms after the press and dropping
   any single frame above 250 counts guards against it.
7. **Symptom.** `setDirectMovement` flips the mode before it writes direction and limit, with no lock, and
   whether `CharMovement::update` can run on another thread then was not established. **Fix:** write
   direction and `sqrt(limit)` into the fields first, then call it; any interleaving reads a consistent
   order. No fault was seen either way.
8. **Symptom.** Booting OGRE headless crashes in `Frustum`, in the mesh loader and in `updateSceneGraph`.
   **Cause:** no `HardwareBufferManager`, no active render system, no `ControllerManager`. **Fix (test
   host only):** construct `DefaultHardwareBufferManager` and `ControllerManager` (both exported), and
   point `Root+0x28` at a stand-in object whose vtable answers `_convertProjectionMatrix` (slot `0x2B8/8`)
   and `getColourVertexElementType` (slot `0x2B0/8`, return 10) and whose `+0x3E0` capabilities report 8
   texture units.
9. **Symptom.** OGRE unloads plugins while the game is still shutting down, leaving patched pointers that
   aim at unmapped code. **Fix:** in `dllStopPlugin` remove the listener and restore every patched slot,
   and pin the DLL (`GetModuleHandleEx` with the pin flag) so a late call still lands on code.
10. **Symptom.** The Kenshi launcher's OK button ignores synthetic clicks under Proton. **Fix:** press
    Enter. More generally on Hyprland: compositor-level synthetic clicks reach Steam's window but not the
    Wine game window; a `/dev/uinput` device does. Steam's "Processing Vulkan shaders" dialog comes back on
    every launch and has to be skipped. Synthetic input lands on whatever has focus: check focus before
    each burst (a human came back mid-test and received the key presses in their browser).
11. **MSVC 2010 x64 layouts used by hand:** `std::string` is 0x28 bytes (16-byte inline buffer or pointer,
    size, capacity, allocator), and a `const std::string&` parameter can be a struct on the stack; member
    functions returning `Vector3` or `Quaternion` by value take a hidden result pointer in RDX after
    `this`; OGRE's `IdString` is 4 bytes in release and travels in a register.

## Assets
None.

## Cost and time
One evening session. Roughly: recon and route 30 min, camera slice with the offline harness 1 h, three Codex
rounds (static RE about 10 min, module about 12 min, review about 10 min) run in the background while the
plugin was built and tested, in-game verification over five launches.

## Open questions
- Attack, block and interact from first person without the cursor (entry points not mapped).
- Always-on mouse look needs the game's cursor hidden or captured while it is active.
- Thread on which `CharMovement::update` runs relative to `frameStarted`.
- Signatures for other builds (GOG 1.0.68, 1.0.65). The camera layer should work there as is.


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

