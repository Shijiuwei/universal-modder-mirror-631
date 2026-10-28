---
kind: game
title: In-game ImGui trainer and save-queue item delivery for LET IT DIE Offline Edition (UE3)
game: LET IT DIE
games_also: []
game_version: 'LET IT DIE Offline Edition 5.0.1 (Steam, Offline Key DLC), Windows x64 (BrgGame-Steam.exe, PE timestamp 0x6A94FD6E), D3D9 renderer'
platform: windows
engine: unreal
route: native-hook
tools: ["MSVC 2026 Build Tools (cl, x64)", "MinHook", "Dear ImGui 1.91.5 (DX9 + Win32 backends)", "Ghidra 12.1.4 (headless, Java scripts)", "UELib 1.13.7 + lzo.net (custom C# dumper)", "Python 3.14 (sqlite3, zlib, Pillow)"]
anti_cheat: "none: the online service ended on 2026-09-01 and the Offline Edition has no servers; only in-exe '[cheat]' clamps on part durability/ammo at load (single-player, offline only)"
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-11'
links: ["https://github.com/TsudaKageyu/minhook", "https://github.com/ocornut/imgui", "https://github.com/EliotVU/Unreal-Library"]
tags: [overlay, imgui, d3d9, dinput8-proxy, ue3, processevent, processinternal, save-editing, sqlite, cheats, item-spawner, icons]
---
# In-game ImGui trainer and save-queue item delivery for LET IT DIE Offline Edition (UE3)

> A `dinput8.dll` proxy that hooks D3D9 `EndScene` and draws an ImGui mod menu inside LET IT DIE Offline Edition, the
> paid single-player release that replaced the online game (custom Unreal Engine 3, "BrgEngine"). It has infinite
> durability, ammo and stamina (done by skipping specific UnrealScript functions in `ProcessInternal`), currency
> buttons (calling script through `ProcessEvent`), an icon item picker, and permanent item and fighter-stat
> delivery. Delivery works through a save-edit queue that a helper process applies after the game exits. All of it
> is verified in the real game (play-tested 2026-10-11), except that one delivered part, Demon Gun 1, never shows
> up in the locker (Gotcha 17).

## Setup
- Game: LET IT DIE **Offline Edition 5.0.1**, Windows 11 x64.
  - **What "offline" means here.** GungHo ended LET IT DIE's online service on 2026-09-01 and released the Offline
    Edition the same day: the base game plus the $19.99 "Offline Key" DLC (Steam and PS4). It has no servers and no
    microtransactions. Death Metals (the paid currency) are gone, and Kill Coins, earned in game, replaced them.
  - That is why a trainer and save edits are fine here: they only touch your own local save and process. Nothing
    in this note touches the Offline Key ownership check; keep it that way. Don't carry this over to a game that
    still sells currency or items, or still talks to a server.
  - `BrgGame\Config\BrgEngine.ini` has `AllowD3D11=False`, so the live renderer is **D3D9**. Hook `EndScene` (vtable 42)
    and `Reset` (16).
- Toolchain:
  - **Build:** MSVC Build Tools (x64, `/MT`), MinHook and Dear ImGui 1.91.5.
  - **Static analysis:** Ghidra 12.1.4 running headless on JDK 21, with the scripts in Java. Ghidra 12 made PyGhidra
    the default Python; Jython scripts still run with a `# @runtime Jython` header.
  - **Save tools:** Python 3.14.
- **Ghidra import crash:** importing `BrgGame-Steam.exe` crashes in the exception-directory parser. Import a copy with
  PE data directory #3 zeroed instead.
- **Script dump:** UnrealScript bytecode came from `BrgGame.u` via UELib, after decompressing the package yourself (see
  Gotchas). The dump is kept outside the mod repo and is never shared.

## Route and why
- **Native hook (proxy DLL + MinHook)**, because there is no community loader.
  - UE4SS and similar tools don't apply: this is UE3, x64, with a heavily customised engine.
  - Cheat Engine tables don't cover item spawning or a menu.
- **Data-only first.** Editing `masters.db` (SQLite) and the save was tried first. It does change prices and the stats of
  *newly created* items. It cannot stop durability or ammo from draining, and owned items keep their saved values.
- **Live item spawning** through the game's debug exec `BrgGameInfo.DebugPartEquip` only works as a temporary preview
  (see Gotchas). Permanent items therefore go through **save editing with the game closed**, automated by a queue.

## How the game works (what we had to learn)
- **Offline edition = bundled backend.**
  - The save `Savedata\<steamid>.sav` is a JSON copy of what used to be server data.
  - Master data is a local SQLite file `BrgGame\Content\masters.db`. The exe opens it by bare name.
  - There is also a native in-memory save database that mirrors parts of the JSON.
- **Save format:**
  - Header: `"BRG\0"`, `u32 version` (2), `u32 total_uncompressed`, `"ZLIB"`.
  - Then chunks of `[u32 usize][u32 csize][zlib]`, which concatenate into UTF-8 JSON.
  - Then a 4-byte zero terminator.
  - **No checksum or signature.** It round-trips byte-exact apart from float formatting (`0.0000` vs `0.0`), which the
    game accepts.
- **Save JSON landmarks:**
  - `soul.free_money` / `soul.spirit`: Kill Coins in the Buffalo Bank and SPLithium in the SPLithium Tank. Their caps
    come from `master_safe_level.limit` / `master_spirit_tank_level.limit` for the current `soul.safe_level` /
    `spirit_tank_level` (500,000 at level 1). What a fighter carries is in `soul.chr.chrs[...].money/spirit`.
  - `soul.bloodnium_point` (account Bloodnium, max 999,999) and `soul.tdm_point` (TDM points).
  - `part.pts["1"]`: owned parts. Each record is
    `{uid, eid(GUID), gettime, owner USER|COIN_LOCKER, created, modified, ptid, rest, spare, grade, dur, lvl}`.
  - `item.items`: materials and consumables, `{eid, gettime, itemid, owner}`.
  - **Coin Locker = `soul.cl`:** 30 slots `{slot, type, eid}`, with type 0 = part, 3 = item, -1 = empty.
    `gameflg.cl` is unrelated.
  - Fighters: stat levels in `bodyuser["1"][i]` (`lvl, hp, str, dex, vit, stm, luk`). Level = 1 + stat points spent.
    XP pool and HP in `soul.chr.chrs["1"][i]` (`gain_exp, start_exp, total_exp, rest_exp, hp, type, grade, limit_break`).
- **masters.db:**
  - `master_part`: per-part `dur`, `dur_hit/shoot/rage` (drain), `capacity/spare` (ammo), and `lvllmttp/lvllmt`
    (equip requirement).
  - `master_item`: `itemtype ITTP_MATERIAL` covers the 106 R&D materials.
  - `master_body_detail.param_lv_max` is the per-stat cap (5 for grade-1 BAL fighters).
    `master_bodylvl_status_value` maps stat level to HP and other values.
  - `master_text (sct, id, lang, txt)` holds names. Join using the `sct.id` split of the `name` column, not the id alone.
- **Engine internals (RVAs, ASLR on):**
  - `UObject::ProcessEvent` 0x10E770, `UObject::ProcessInternal` 0x10EC70, `UObject::CallFunction` 0x103610,
    `FString::operator=` 0x19850.
  - `GNames` 0x2949C98. Entries hold the name at +0x14 and flags at +0x08 (bit 0 = wide).
  - `GObjects` is found at runtime by scanning writable sections for a `TArray` whose entries' `Index` (+0x38) equals
    their position.
- **Object layouts (x64):**
  - UObject: Index +0x38, Outer +0x40, Name +0x48, Class +0x50.
  - UField::Next +0x60, UStruct::Children +0x80.
  - UFunction: PropertiesSize +0x88, FunctionFlags +0xD0, iNative +0xD4, Func +0xF0.
  - UProperty: PropertyFlags +0x70, Offset +0x94.
  - FFrame: Node +0x14, Object +0x1C, Code +0x24, Locals +0x2C.
- **Script dispatch:** `CallFunction` calls `ProcessInternal` **directly** for script functions. Swapping
  `UFunction::Func` therefore misses script-to-script calls. Hooking `ProcessInternal` and comparing `FFrame.Node` catches
  every script function body.
- **Drain functions** (all void and decrement-only):
  - Durability: `BrgCommonPawn_CustomChara.WeaponDurabilityDown`, `WeaponShotDurabilityDown` and `ArmorDurabilityDown`.
    They run on the per-character helper object, so the owner is its `mPawn` property.
  - Ammo: `BrgWeapon_*.BulletConsumption` (9 overrides). The owner is `BrgWeapon_Base.mOwnerPawn`.
  - Stamina: `BrgPawn_CustomCharaPlayer.StaminaConsumption`.
  - Repairs use the separate `PartDurabilityRecovery*` functions.
- **Currency:** `BrgUserData` natives through ProcessEvent: `Get/AddMoney` and `Get/AddSpirit(..., cindex=-1)`
  (carried by the current fighter), `Get/SetSafeMoney` + `GetSafeLimit` (Bank), `Get/SetSafeSpirit` +
  `GetSpiritTankLimit` (Tank), `Get/SetUserBloodnium` + `GetBloodniumLimit`, `Get/SetTdmPoint`. Read, clamp to the
  game's limit (never lowering a balance already above it), then set. Bloodnium needs one more step (see Gotchas).
  The live objects are `PlayerController.WorldInfo.Game` (the GameInfo) and its `.mUserData`.
- **Icons:**
  - Every item has its own package `CookedPCConsole\UI_Icon_<id>_SF.upk`, holding one inline Texture2D (108x132, mostly
    A8R8G8B8).
  - The game has **no icon fallback**: a part without its own package has no icon in game.
- **Packages:** the cooked `.upk` files are fully LZO-compressed. Bulk-data headers use **64-bit size/offset** (24-byte
  header), which stock UE Viewer (umodel) can't read.

## Build steps
1. Build `dinput8.dll` with the exports `DirectInput8Create`, `DllCanUnloadNow`, `DllGetClassObject`, `DllRegisterServer`
   and `DllUnregisterServer`, each forwarding to `System32\dinput8.dll`. Place it next to `BrgGame-Steam.exe`
   (`Binaries\Win64`).
2. Worker thread:
   - Make a throwaway D3D9 device to read the vtable, then MinHook `EndScene`/`Reset`.
   - MinHook `ProcessEvent` and `ProcessInternal`.
   - Hook `IDirectInputDevice8::GetDeviceState`/`GetDeviceData` (vtable 9/10, both the A and W interfaces), so the game
     gets blank mouse/keyboard input while the menu is open.
3. Queue game-thread work, and run it from inside `ProcessEvent` only when the event is `PlayerTick`.
   - Resolve targets in one pass by FName index.
   - Read property offsets from the UProperty objects themselves (`Offset` at +0x94).
4. **Cheats:** in the `ProcessInternal` hook, compare `FFrame.Node` against the resolved drain functions. If the owner is
   the local `BrgPawn_CustomCharaPlayer`, return without running the body.
5. **Permanent items and stats:**
   - The overlay appends lines to `lid_mod\save_queue.txt`.
   - At launch the DLL starts a detached `pythonw` helper that waits on the game PID. After exit it takes a lock file,
     backs up the save and applies the queue (new locker records shaped exactly like the game's own, or a stat max).
   - It writes atomically (tmp → decode-verify → fsync → `os.replace`) and leaves a result file for the next launch.
6. **Item icons:** extract the `UI_Icon_*` textures with a custom UELib-based exporter (LZO + 64-bit bulk data + DXT
   decode). Then build a 64 px atlas plus a TSV index, which the overlay loads with WIC into a managed-pool D3D9 texture.

## Verification
- **Oracles used:**
  - Screenshots of the live overlay.
  - The game's own `BrgGame\Logs\Launch.log`. Engine asserts are logged there, e.g. `body_lvl distribute failed! ...`.
  - Windows Application event log entries 1000/1001 (exception code 0x1 in KERNELBASE = engine `appError`).
  - Decoding the save before and after each change.
  - A file-based RPC in the DLL (command/result text files) to drive scans and calls from the agent.
- **Verified in game:**
  - The overlay and Insert toggle.
  - Currency adds through the `BrgUserData` natives.
  - The live `DebugPartEquip` preview.
  - Locker delivery of six gear parts through the queue (shown in the locker after relaunch).
  - Fighter stat max (level 1 → 25).
  - Recovery from a launch crash caused by a stale RPC command.
  - Play-tested 2026-10-11: the `ProcessInternal` cheats in their final form (durability, ammo, stamina), input
    blocking while the menu is open, and R&D material delivery to the Coin Locker.
- **Also tested on a scratch copy of the save:** the queue helper, including the lock file, stale-lock takeover,
  structured retry, one bad entry not blocking the rest, and material (`item`) delivery.

## Gotchas
1. **Edited `masters.db` but nothing changed in game.** **Cause:** owned items keep the values stored in the save; the DB
   only seeds newly created items. Drain also has a flat `MaintenanceValue` term on top of `dur_hit`, and an exe check
   (`[cheat]part dur/rest/spare over!`) clamps saved values to the DB maximum on load. **Fix:** skip the drain script
   functions live instead; DB edits only for prices and new items.
2. **Live "add XP" crashed the stat screen** (`body_lvl distribute failed! ... exp=14, nec_exp=138`, then an assert).
   **Cause:** `AddExp` raised the script-side value only; the native stat-distribute check uses the saved XP pool. The GM
   tool natives (`DebugGmTool*_NT`) are compiled-out stubs in this build. **Fix:** edit stat levels in the save
   (`bodyuser`) with the game closed; level = 1 + points spent, capped by `param_lv_max`.
3. **Spawned item vanished on hand switch or reload.** **Cause:** `DebugPartEquip` mints a debug entity id (empty or
   leading `-`), which the save code skips. Forcing a real GUID made it persist briefly, but the game then crashed going up
   the tower: an integrity pass looks the part up in the native save DB and null-derefs because it was never registered.
   **Fix:** treat live spawn as a preview only; deliver permanent items by save injection.
4. **Item injected with `owner: COIN_LOCKER` didn't appear.** **Cause:** the locker also needs a `soul.cl` slot entry
   (type 0 part / 3 item) pointing at the `eid`. **Fix:** fill the first `type -1` slot.
5. **Hooking `UFunction::Func` didn't catch script calls.** **Cause:** `CallFunction` calls `ProcessInternal` directly
   for non-native functions. **Fix:** hook `ProcessInternal` and match `FFrame.Node`.
6. **Durability hook never fired** (caught in code review). **Cause:** the drain functions run on the
   `BrgCommonPawn_CustomChara` helper object, not the pawn, so a class check on `this` never matched the player. **Fix:**
   follow the helper's `mPawn` property; for weapons, `mOwnerPawn`.
7. **Game wouldn't launch after an update.** **Cause:** a leftover RPC command file was executed before the module base
   was known. **Fix:** ignore any command already present at startup, and initialise the base lazily.
8. **UELib saw 0 exports.** **Cause:** the cooked packages are LZO-compressed as a whole. **Fix:** decompress first,
   using the chunk table in the summary (this build reports engine version 9953, cooker version 135).
9. **umodel fails on every Texture2D ("Serializing behind stopper").** **Cause:** bulk-data headers are 24 bytes (flags,
   count, i64 size, i64 offset). **Fix:** a custom exporter.
10. **Mouse invisible in the overlay.** **Cause:** the game hides the OS cursor, and `SetCursor` from the render thread
    doesn't apply. **Fix:** `io.MouseDrawCursor = true` plus `ImGuiConfigFlags_NoMouseCursorChange`.
11. **Mouse still turned the camera with the menu open.** **Cause:** the game reads input through DirectInput, not
    window messages. Counters in the hooks showed it uses **buffered** `GetDeviceData` only (about 100k calls, zero
    `GetDeviceState`, zero `SetCursorPos`). A first filter that kept "releases" by offset (`dwOfs >= DIMOFS_BUTTON0`)
    let mouse motion through, because UE3 sets its own mouse data format whose axis offsets are not the standard
    `DIMOFS_*`. **Fix:** classify each device once with `GetDeviceInfo` (vtable 15; mouse/keyboard only, never
    joysticks, whose zeroed axes read as full deflection). While the menu is open, keep only buffered events whose
    value says "released": `dwData == 0` for the mouse, `(dwData & 0x80) == 0` for the keyboard. Nothing stays stuck
    down after the menu closes.
12. **Threading hazards** (code review). `EndScene` runs on UE3's render thread, not the window thread:
    - ImGui input must be queued from the WndProc. Capture Ctrl/Shift/Alt there, because `GetKeyState` is per-thread.
    - `SetWindowPos` for borderless must run on a separate worker thread; on the render thread it can deadlock against
      the game thread's `WM_SIZE` handler.
13. **Script-call parameter buffers.** `ProcessEvent` copies `ParmsSize` bytes in *and back out*, so use generous zeroed
    buffers. String parameters must be engine-allocated (`FString::operator=`), because script may reassign them
    (`DebugPartEquip` does) and would then free a caller-owned buffer.
14. **`DebugPartEquip` side effects.** It sets `GameInfo.mbDeadInfoOutputCancel` (a debug flag affecting death
    bookkeeping), adds to the Death Bag, and treats unknown slot names as Head ("All" wipes the inventory). **Fix:**
    whitelist the slot names and restore the flag's DWORD after the call.
15. **Adding Bloodnium, then levelling up, crashed** (`body_lvl distribute failed! ... bloodnium=0,
    nec_bloodnium=13`). **Cause:** the stat screen shows `GetUserBloodnium() + GetBloodnium()` from `BrgUserData`, but
    the native stat-distribute check (`FUN_141366480` → `FUN_14133f7c0`) takes its pool from the native save-store soul
    record (static object at `DAT_14f89ef80`, Bloodnium at +0xB4) plus the current chara record (+0x5C), and spends
    from those. `SetUserBloodnium` updates only the script-side copy, so the UI allowed an upgrade the native check
    rejected with an assert. Adding to the per-fighter pool through `AddBloodnium(cindex)` has the same problem.
    **Fix:** after the `BrgUserData` add, also write the new total into soul+0xB4. Verify first: the right exe build
    (PE timestamp), the field equal to what the game reports, and the neighbouring replica cap (+0xB8) being -1 or not
    below it. Undo the UI-side add if the native write fails. Same lesson as Gotcha 2: anything spent through a native
    check must be written to the native store.
16. **Death Metals.** The Offline Edition replaced them with Kill Coins, so `Get/SetMedal` still exists but does nothing
    useful; leave it out.
17. **Demon Gun 1 (`PT_ARM_WP017_0A1`) injected into the locker never showed** while GLIDER 2 (`PT_ARM_WP025_0K3`) did.
    Both have no icon package. Unresolved; see Open questions.

## Assets
None generated. Item icons are extracted at build time from the user's own install into a local atlas (64 px cells,
40 per row) and a TSV index. They are never committed or redistributed.

## Cost and time
One long session (about a day of wall-clock), including Ghidra/UELib tooling setup and two automated code-review passes
(30 findings, all addressed).

## Open questions
- Why some injected locker parts (`PT_ARM_WP017_0A1`) don't display. Candidates: no `UI_Icon_` package, or a native
  filter on `master_part.platform`.
- Where the native save-DB insert for parts lives. Finding it would make live spawning permanent.
- Whether `BeginGunBulletConsumptionDisable` (a native flag) is a cleaner infinite-ammo route.
- R&D blueprints (`ITTP_RMAP`, about 1,900 entries) are not in the picker yet.


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

