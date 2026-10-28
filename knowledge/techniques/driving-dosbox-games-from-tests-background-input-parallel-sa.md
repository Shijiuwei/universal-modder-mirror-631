---
kind: technique
title: 'Driving DOSBox games from tests: background input, parallel sandboxed instances and screen-state checks'
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-25'
links: ["https://www.dosbox.com/"]
tags: [dosbox, dos, automation, testing, background-input, postmessage, parallel, sandbox, screenshots, midi, overlays, memory-pressure, windows, guest-memory, readprocessmemory, conventional-memory, focus-stealing, wmi]
---
# Driving DOSBox games from tests: background input, parallel sandboxed instances and screen-state checks

> How to run a DOS game in DOSBox as a test oracle on Windows: input posted to a background window (no focus,
> no real mouse), exact-frame screenshots, several sandboxed instances in parallel under a machine-wide cap,
> every step checked against the screen, and silence. Built while binary-patching a 1994 DOS strategy game
> (Sid Meier's Colonization 3.0, GOG release) with an 85-test suite that runs in about 10 minutes at 4-10
> instances on a laptop.

## When to use it
- You patch or mod a DOS game and want in-game regression tests that run while the human keeps using the PC.
- You need many game runs (flaky bugs, random events, original vs patched comparisons) and one DOSBox at a
  time is too slow.
- Setup used: DOSBox 0.74-2 (the build GOG bundles), SDL 1.2, Windows 11, Python 3 with `ctypes`, Pillow,
  numpy, psutil.
- Ask the human before the first long run: a suite puts up to 10 DOSBox windows on their desktop and takes most
  of the CPU. Posted messages never move the real mouse or keyboard, but a DOSBox started as an ordinary child
  process can still take the foreground while they type (gotcha 21). Launch it through WMI (below); if you have
  to fall back, check `um win drive --proc dosbox idle` first and don't launch while they're typing. The general
  rules are in [Driving real games safely](driving-real-games-safely.md).

## How

### Input without focus
- Launch DOSBox with `SDL_VIDEODRIVER=windib` in its environment. When the variable is unset, DOSBox 0.74 on
  Windows switches SDL 1.2 to its DirectX driver (SDL's own default has been `windib` since 1.2.10), and that
  driver reads the keyboard and mouse through DirectInput instead of window messages (the keyboard only while the
  window is in the foreground). `windib` reads ordinary window messages, so posted ones reach the game.
- Post `WM_KEYDOWN`/`WM_KEYUP` (and `WM_SYSKEYDOWN` with Alt) to DOSBox's window with `PostMessage`, about 80 ms
  between down and up (shorter than a frame and the game misses it), and set the extended-key bit for arrows,
  Ins/Del/Home/End/PgUp/PgDn. Before input, post `WM_ACTIVATE` (wParam 1) to DOSBox only: SDL drops input while
  it thinks it is inactive, and the real focus never moves.
- Mouse: `autolock=false` in `[sdl]` (otherwise DOSBox drops mouse motion until the mouse is locked). Post
  `WM_MOUSEMOVE` in a few small steps, then button down/up at the target, in client coordinates scaled from
  320x200. Unlocked, DOSBox 0.74 sets the DOS cursor from the window position in graphics modes, but the mouse
  driver's motion counters, which some games read instead, only get the moves between events.
- **Start DOSBox outside the terminal's process tree**, or each new DOSBox window can take the foreground from
  the human (gotcha 21). On Windows, create it through WMI (`Win32_Process.Create`): the new process's parent is
  `WmiPrvSE.exe`, not your test runner. A plain child process (`subprocess.Popen`, `Start-Process`,
  `CreateProcess`) of a test started from the active console is not enough, even with a "don't activate" show
  flag. From Python, through PowerShell's CIM cmdlets:

  ```python
  q = lambda t: "'" + str(t).replace("'", "''") + "'"
  env = dict(os.environ, SDL_VIDEODRIVER="windib")      # WMI does NOT inherit your environment: pass it
  envlist = ",".join(q(k + "=" + v) for k, v in env.items() if "\n" not in v and "\0" not in v)
  ps = ("$si = New-CimInstance -ClassName Win32_ProcessStartup -ClientOnly -Property "
        "@{ShowWindow=[uint16]7; EnvironmentVariables=[string[]]@(" + envlist + ")}; "
        "$r = Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments "
        "@{CommandLine=" + q(subprocess.list2cmdline([dosbox, "-conf", conf, "-noconsole"])) + "; "
        "CurrentDirectory=" + q(dosbox_dir) + "; ProcessStartupInformation=$si}; "
        "Write-Output $r.ReturnValue $r.ProcessId")
  out = subprocess.run(["powershell", "-NoProfile", "-NonInteractive", "-Command", ps],
                       capture_output=True, text=True, timeout=60).stdout.split()
  pid = int(out[1]) if len(out) >= 2 and out[0] == "0" else None   # ReturnValue 0 = started
  ```

  `ShowWindow=7` is `SW_SHOWMINNOACTIVE`. Set `CurrentDirectory` explicitly (otherwise it is the WMI host's, not
  yours) and pass every environment variable DOSBox needs explicitly. The process isn't your child, so track it
  by the returned PID: find its window by PID, and kill it by PID. A job object or "kill my children" won't reach
  it. Keep a plain child-process launch only as a fallback for when WMI fails, and log when you use it.

  The snippet puts your whole environment on PowerShell's command line, where other programs (process lists,
  logging and security tools) can read it. If it holds secrets (API keys, tokens), pass a short list instead:
  `SDL_VIDEODRIVER` plus what DOSBox needs, such as `SystemRoot`, `PATH`, `TEMP` and `TMP` (not tested here).
- Launch minimised without activation (`SW_SHOWMINNOACTIVE`, above), then
  `ShowWindow(hwnd, SW_SHOWNOACTIVATE)` once the window exists: screenshots need it restored, not minimised.
- `EnableWindow(hwnd, FALSE)` once it is shown: the human's stray clicks and keys no longer reach the game,
  while posted messages still do. Re-enable it (or skip this) when you want to play along.

### Screenshots
- **Exact frame, slow:** DOSBox's own capture (post Ctrl+F5); `captures=` in `[dosbox]` sets the folder. It
  writes a PNG of the emulated 320x200 frame, about 0.5 s per capture.
- **Fast:** copy the window's client area (`PrintWindow`/`BitBlt`) and scale it to 320x200: about 5 ms. Check it
  pixel for pixel against a Ctrl+F5 capture once per window size and fall back if they ever differ. The process
  doing the copy must be DPI-aware, like DOSBox's window (gotcha 6).

### One sandbox per test process
Give every test process its own folder (e.g. `work/run/<run id>/`) holding:
- a copy of the game folder it launches (saves, autosaves, high-score files and the EXE never collide);
- its own DOSBox conf, written from a template by replacing the `mount c:` and `captures=` **lines** (gotcha 9);
- its own capture folder.

Remember the PID of the DOSBox you started and make every helper (find window, input, capture, kill) act on
that PID only; never "the first DOSBox window" and never kill by process name. A runner script then starts N
tests as separate processes. Remove the sandbox at exit, but first move any screenshot a test saved under its
own name to a shared results folder (gotcha 10).

### A machine-wide cap
Parallel test runners, agents and git worktrees all launch DOSBox, so cap the total, not per runner:
- N lock files in a fixed folder (e.g. `~/.coltest-dosbox-slots/slot0..N.lock`); `launch()` takes one with an
  OS byte lock (`msvcrt.locking(..., LK_NBLCK, 1)` on Windows), which is released automatically if the process
  dies; it waits when all are taken.
- Plus a load gate: beyond the first two games, wait while total CPU is over ~80 % or free RAM under ~1 GB
  (psutil).
- Measured on an Intel Core Ultra 7 155U laptop (14 threads, 15.5 GB): each DOSBox at `cycles=max` costs about
  9 % total CPU and 80 MB; 10 at once was the useful ceiling. CPU, not RAM, is the limit.

### Check the screen before and after every step
Fixed sleeps followed by blind clicks are the main source of flaky and slow tests.
- Name what is on screen from small reference crops: title, load list, map, message box, menus, the game's
  own screens. Match text by its exact palette colour (DOS games draw text in a few fixed colours) with a
  tolerance for partly covered glyphs (gotcha 8).
- `wait_until(state, timeout)` polls the fast capture instead of sleeping; every action confirms the expected
  state first and the change afterwards; any timeout raises with a named screenshot.
- Stall detector: the same frame for 90 s with nothing to answer → fail at once with a screenshot, instead of
  waiting for a long overall timeout.
- Use the game's own files as the oracle where possible: the start-of-turn autosave tells you the turn
  advanced and holds the game state to assert on.
- Original vs patched (or several cases) run as parallel processes, not one after another.

### Read the game's state from guest RAM
When a run hangs, the screen alone rarely says why. DOSBox keeps the guest's conventional memory in one private
block of its own process, so a host-side script can read it without touching the game:
- `OpenProcess(PROCESS_VM_READ | PROCESS_QUERY_INFORMATION)`, walk the regions with `VirtualQueryEx`, and find
  the guest RAM block by a signature string from the game's data segment (or by its size); read the first
  1 MB + 64 KB with `ReadProcessMemory`. Read-only: no input, no focus, the game keeps running.
- Decode the DOS memory picture from there: the MCB chain (DOSBox's List of Lists sits at 0080:0026, the first
  MCB segment two bytes before it), the game's blocks and the largest free block.
- Then read the game's own variables at their DGROUP addresses (turn phase, active unit, mode flags). In this
  project a "hang" turned out to be the game's view mode, waiting correctly for a key (gotcha 16).

### Measure free conventional memory before adding code
If you patch code or tables into the executable, every paragraph you add is 16 bytes less free DOS memory at
run time. Measure the worst case instead of guessing:
- read the game's allocator state from guest RAM (many DOS games record the largest free block after each
  allocation, or keep a low-water mark), across the heaviest scenarios (big late games, every report screen,
  save/load) and with EMS/XMS on and off;
- build test EXEs with a deliberately bigger block to see what fails first (here: large optional buffers were
  refused and the game fell back to smaller ones; nothing crashed until far past the budget);
- remember the store launcher: GOG starts this game through a front-end EXE that loads it into its own process
  and costs about 5 KB more than starting the game directly;
- turn the result into a budget with a margin and an automated test that fails when a build exceeds it.

### Silence
`[mixer] nosound=true` silences DOSBox's mixer (Sound Blaster, OPL, PC speaker, CD audio). MIDI bypasses it
(gotcha 4): also set `[midi] mididevice=none`, and keep `mpu401=intelligent` if the game is configured for
MIDI.

### Make rare overlay and memory bugs deterministic
Many DOS games use an overlay manager that swaps code in and out of conventional memory. A stress conf with
`[dos] xms=false ems=false umb=false` makes it swap constantly; a bug that showed up once in 10-20 normal runs
then fails every time (gotcha 3). Run the tests that touch your patched code under both confs.

## Not verified
Only DOSBox 0.74-2 (SDL 1.2) on Windows 11, driving one game. DOSBox Staging and DOSBox-X's SDL2 builds have no
`windib` or `directx` video driver (SDL2's is `windows`), so the input section won't carry over to them as written.

## Gotchas
1. **Posted keys did nothing; clicks did nothing.** (2026-10-22.) **Cause:** DOSBox 0.74 picks SDL 1.2's DirectX
   driver, which reads DirectInput instead of window messages, and SDL ignores input while it believes it is
   inactive. **Fix:** `SDL_VIDEODRIVER=windib` at launch and a posted `WM_ACTIVATE` before input;
   `autolock=false` for the mouse.
2. **Enter on the map "ended the turn" but sometimes skipped or moved units, and hid a real bug.** (2026-10-24.)
   **Cause:** pressing a default key blindly also answers popups you never looked at; a debug assertion box of
   the game's ("DANGER, WILL ROBINSON") was being dismissed by the test for days. **Fix:** recognise the screen
   and press Enter only on a recognised message box; click the game's own "end turn" control where it is.
3. **A test failed about 1 run in 10 with garbage prompts, random exits to DOS or a frozen screen.** (2026-10-25.)
   **Cause:** the patch entered new code through a stub the game's overlay (VM) manager didn't recognise as a
   call from an overlay, so it didn't protect the return address; when memory was tight and the overlay was
   swapped out, the hook returned into the wrong code. With EMS/XMS present that was rare. **Fix:** reproduce
   under the no-EMS/XMS/UMB conf (now every time), then fix the stub. Keep that conf in the suite.
4. **Games still played music with `nosound=true`.** (2026-10-25.) **Cause:** DOSBox sends MIDI straight to
   the Windows MIDI synthesizer, outside its mixer. **Fix:** `[midi] mididevice=none`.
5. **With `mpu401=none` the game hung on a black screen at start.** (2026-10-25.) **Cause:** the game was set up
   for General MIDI and waits for the MPU-401. **Fix:** keep `mpu401=intelligent`, silence it with
   `mididevice=none` (gotcha 4).
6. **The fast window copy returned only the top-left part of the picture.** (2026-10-25, 125 % display
   scaling.) **Cause:** DOSBox's window is DPI-aware and the capturing process wasn't, so it got DPI-virtualised
   coordinates. **Fix:** make the capturing process DPI-aware before calling `PrintWindow`/`BitBlt`.
7. **The test clicked where "End of Turn" usually is and kept toggling tile info instead.** (2026-10-24.)
   **Cause:** the side-panel text moves down when the panel lists a colony or units, and when a unit waits for
   orders the spot holds unit details. **Fix:** find the text by its glyphs and palette colour anywhere in the
   panel, click it there, and handle "a unit is waiting" separately (skip it with the game's key).
8. **Turn 1 worked, then the test waited until its timeout.** (2026-10-24.) **Cause:** after clicking, the
   (emulated) mouse pointer stayed on the text, so the next turn's exact glyph match failed. **Fix:** park the
   pointer in a neutral spot after every click, and accept about 85 % of a template's pixels.
9. **Tests in a git worktree silently played the main checkout's build.** (2026-10-25.) **Cause:** the conf
   template names the main checkout's game folder and the per-run conf replaced that exact path text; in a
   worktree the path differs, nothing was replaced, and DOSBox mounted the main folder. **Fix:** rewrite the
   `mount c:` and `captures=` lines by pattern and fail loudly if either is missing.
10. **Failure screenshots vanished.** (2026-10-24.) **Cause:** tests renamed captures inside the sandbox's
    capture folder, which is deleted at exit. **Fix:** on cleanup, move every capture not named like DOSBox's
    own (`<program>_NNN.png`) to the shared screenshots folder.
11. **A menu ignored Enter for minutes, about 1 run in 15.** (2026-10-25.) **Cause:** a click was still in
    progress when the menu opened; this game's menu loop then switched to mouse mode, where the highlight
    follows the pointer (outside the box) and Enter selects nothing. The unpatched game did the same. **Fix:**
    when a box ignores Enter and shows no highlighted row, press Down, then Enter; check for a highlight first so
    menus with a valid default aren't changed.
12. **Overlay programs need their own EXE folder.** (2026-10-22.) **Cause:** the overlay manager reopens the
    EXE by name from the current drive to load overlays, so a different EXE (original vs patched) in the same
    mount reads the wrong overlays. **Fix:** one game-folder copy per EXE variant, mounted as `C:`.
13. **A probe killed the running suite.** (2026-10-23.) **Cause:** the launcher killed every DOSBox before
    starting its own. **Fix:** PID-scoped kill (see "One sandbox per test process"); keep a separate "kill
    all" for manual clean-up only.
14. **Fast retries kept replaying the same battle.** (2026-10-25.) **Cause:** the game seeds its RNG from the
    BIOS clock; restarts within the same tick replay the same rolls. **Fix:** a short random pause before
    retrying a random event.
15. **Saves rebuilt by parallel tests could be read half-written.** (2026-10-24.) **Cause:** several tests
    rebuild the same scenario save before running. **Fix:** skip the write when the bytes are identical, else
    write a temp file and `os.replace` it, retrying while another process has it open.
16. **A game run "hung" with a unit waiting but no prompt, rarely and only under load.** (2026-10-25.) **Cause:**
    the test clicked "End of Turn" a second time; the click was made while the game was busy ending the turn,
    so the mouse driver's latched button state was read at the next turn's first input poll, where it put the
    game into view mode, waiting for a key. The unpatched game did the same. **Fix:** only act on *live* UI:
    click the end-turn text only while it blinks (stale text left on screen doesn't), and after a key wait for
    the screen to change before acting again. Clicks are latched, never just lost.
17. **Keys and clicks were lost under load (a menu didn't open).** (2026-10-25, 6-10 parallel instances.)
    **Cause:** the emulated game missed a posted key or click while busy. **Fix:** retry an action only if
    nothing at all happened (the screen is still the same, or still the plain map), and never retry a move,
    whose popup may just be slow; restart a menu sequence from the map if the menu closed without the item.
18. **Alt+G was ignored for a minute at the start of a turn.** (2026-10-25.) **Cause:** this game ignores
    keys while units make their automatic (goto/route) moves. **Fix:** keep retrying menu keys until the moves
    are over (bounded), instead of failing after a few seconds.
19. **A message box popped up just as the test opened a menu.** (2026-10-25.) **Cause:** the result of the
    last action arrived late. **Fix:** if a menu step finds a message box instead, answer it and redo the step.
20. **A full-screen "is the screen still?" check never finished.** (2026-10-25.) **Cause:** the selected unit
    and the end-turn text blink, so the frame never stays the same. **Fix:** compare screen *states* (what the
    screen shows) or masked regions, not whole frames.
21. **New DOSBox windows stole the focus from the human's terminal.** (2026-10-25.) A foreground watcher logged
    17 takeovers in about 3 minutes of a parallel suite run, each when a new DOSBox window appeared. **Cause:**
    the tests ran as descendants of the console program the human was typing into, and Windows lets processes
    started from the foreground process bring their first window to the front. The DOSBoxes were started with
    `SW_SHOWMINNOACTIVE`, which didn't stop it: SDL creates and shows its own window. It only happened while the
    human was using that console, so a test made while they weren't typing showed no steal. **Fix:** create
    DOSBox through WMI so it has no link to the console (see "Input without focus"). Afterwards a 15-minute
    watch across a full parallel suite logged none. **Check it:** poll `GetForegroundWindow()` every 100 ms
    during a suite run and log when the foreground window's process is a DOSBox. Two more traps: a test process
    that was already running (for example, one waiting for a free slot) keeps the launch code it imported, so
    restart runs before you measure. And when you swap the launch code, test the whole launch function, not
    just the new helper: a leftover `return p.pid` after the change failed every in-game test.

## Seen in
- Sid Meier's Colonization 3.0 (DOS, GOG release; VICEROY.EXE, an RTLink-overlaid executable) binary-patching
  project (no public game note yet).
- Related: [Driving real games safely](driving-real-games-safely.md) (input and focus rules for any game) and
  [Running several isolated tModLoader test instances in parallel](running-several-isolated-tmodloader-test-instances-in-parall.md)
  (the same sandbox-per-instance idea for Terraria).


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

