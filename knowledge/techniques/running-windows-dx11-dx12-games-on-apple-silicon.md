---
kind: technique
title: "Running Windows DirectX 11/12 games on Apple Silicon: choosing a route (native, CrossOver, free Wine) and what breaks"
status: in-progress
agents:
- Claude Code (Opus 5.5 and Sonnet 5.5)
humans: ["DiegzM"]
date: '2026-10-15'
links:
- https://www.codeweavers.com/crossover
- https://www.codeweavers.com/crossover/source
- https://developer.apple.com/games/
- https://github.com/KhronosGroup/MoltenVK
- https://github.com/Gcenx/macOS_Wine_builds
- https://github.com/3Shain/dxmt/issues/141
- https://github.com/notpop/steam-on-m1-wine
- https://github.com/BCD1210/soju/blob/main/scripts/play.sh
- https://github.com/codeWonderland/neutron/blob/89084b1/docs/phase0-spike.md
- https://developer.valvesoftware.com/wiki/SteamCMD
- https://github.com/BepInEx/BepInEx
- https://github.com/GDRETools/gdsdecomp
tags: [macos, apple-silicon, crossover, wine, free-wine, gcenx, d3dmetal, dxmt, moltenvk, directx-11, directx-12, vulkan, steamcmd, steam, steam-client, msync, godot, unity, unreal, bepinex, verification, oracle]
---
# Running Windows DirectX 11/12 games on Apple Silicon: choosing a route (native, CrossOver, free Wine) and what breaks

> `um win` only drives Windows games on Windows or WSL, so on a Mac a Windows-only game had no route. This note
> is which route to try for one, in what order, and what breaks on each. The routes seen here are **CrossOver's
> command-line tools** (paid; Apple's D3DMetal for DirectX 11/12, MoltenVK for Vulkan) and **free Gcenx Wine
> builds** (no D3DMetal for Wine 11; Windows Steam needs a workaround). Both let the agent observe a game like
> any other: logs, window screenshots, input, a mod loader. Everything was validated **once, on one machine**
> (Apple M5, macOS 26.5.1, CrossOver 26.3) with a Unity Direct3D 11 game, an Unreal Engine 5 DirectX 12 game and
> a Godot Vulkan game. **It does not mean any Windows game will run.** Written by an AI agent with a human
> contributor's guidance; nobody else has reproduced it.

## Pick the easiest route first
Use the least machinery that can run the game, and add a layer only when the evidence says you need it:
1. **A Mac build?** Check the store page and the engine first. A native build (or a re-export, for a project the
   user owns) beats every translation layer. Everything below is for Windows-only games.
2. **Anti-cheat files** (`EasyAntiCheat*`, `BattlEye*`, `vanguard`, `Denuvo`)? Stop and tell the user (Scope
   table).
3. **A plain exe with no launcher and no Steamworks** (`file` says `PE32+ ... x86-64`, no `steam_api64.dll`): the
   minimum is Wine and an empty prefix. Observed: a Godot 4.3 Vulkan export ran on CrossOver and on the free Gcenx
   Wine Stable 11.0_1, through MoltenVK (last paragraph of How). No bottle tooling, Steam or D3DMetal was needed.
4. **DirectX 11, no Steam:** the free builds can try it with Wine's own Direct3D over MoltenVK (not tested here
   without Steam); CrossOver gave Direct3D 11 through D3DMetal (PEAK). An outside repo reports Monster Hunter:
   World (Direct3D 11) on a Direct3D-on-Metal layer in CrossOver (third-party report; see Seen in).
5. **A Steamworks game:** Windows Steam must run in the *same* prefix (Gotcha 7). CrossOver: it signed in with no
   workaround. Free Wine: only with the web-helper wrapper and the audio workaround (How, free Wine route); one
   build confirmed, speed not measured.
6. **DirectX 12 only:** D3DMetal is the route that ran one (CrossOver: MECCHA CHAMELEON's title screen). Free: of
   the builds tried here, only the GPTK 3.0-3 build (Wine 7.7) has D3DMetal; it passed the DirectX 12 probe but
   cannot run today's Steam client. The free Wine 11 builds reach only feature level 11_0. Sikarugir's
   CrossOver-based engines can load D3DMetal too (neutron's notes created a D3D12 device on its CX 24.0.7_7 engine);
   not tried here. An outside repo reports Elden Ring (Direct3D 12) on CrossOver's D3DMetal (third-party report,
   not verified here; see Seen in).
7. **DirectX 9/10, OpenGL, 32-bit, launchers other than Steam:** untested here (Scope table; the Underrail note ran
   a 32-bit game in CrossOver).

Other routes exist (Windows ARM in a virtual machine, Linux on the Mac, cloud streaming). **None was tried**, and
their DirectX support, cost and licence change often, so check the current state before relying on one.

**Routes compared (all rows: one Mac, October 2026)**

| Route | Cost | Windows Steam client | DirectX 11/12 | Observed here |
|---|---|---|---|---|
| CrossOver 26.3 command-line tools | paid | signs in, UI draws | D3DMetal: D3D11 and D3D12 | PEAK to gameplay, MECCHA CHAMELEON title screen, Godot game to a race |
| Free: Gcenx Wine Stable 11.0_1 | free | black UI until the web-helper wrapper, then the login UI and the contributor's account loaded (Gotcha 11) | no D3DMetal; DirectX 12 probe reached only feature level 11_0 | Unity game booted; Godot game ran on Vulkan |
| Free: Gcenx Wine Devel / Staging 11.18 | free | black without the wrapper; wrapper **not tested** | same as Stable | probe at level 11_0; Unity game booted |
| Free: Game Porting Toolkit 3.0-3 build (Wine 7.7) | free; D3DMetal's licence rules out commercial ports | does not run (Gotcha 11) | D3DMetal, feature level 12_2 | probe passed; PEAK crashed at input init (Gotcha 2) |
| Free: Wine 11.18 rebuilt with a patch author's `winemac` and DXMT patches | free; about 30 minutes to build | no login window on this Mac (Gotcha 29) | DXMT (DirectX 10/11 only) | not working here; its author reports it on other chips and macOS versions |

## What the free route gives up, and why it is harder
- **No fast sync.** `strings` on the core library finds no `msync`, `esync` or `ntsync` in Gcenx Stable 11.0_1 or
  Devel/Staging 11.18, but msync in CrossOver 26.3 and in the GPTK 3.0-3 build. A likely (unproven) part of why the
  free builds feel slower (Gotcha 26).
- **No D3DMetal on a free Wine 10 or later.** D3DMetal is Apple's closed-source layer, and its licence rules out
  commercial ports (Gcenx's Sikarugir README). CrossOver ships it with its own Wine 11. Outside CrossOver it needs a
  CrossOver-based Wine 9 or older: neutron's README says it "doesn't run on Wine 10 or later", and its notes ran it
  on Sikarugir's CX 24.0.7_7 engine. Here the only such build tried was the GPTK 3.0-3 build's Wine 7.7, which
  cannot run Windows Steam.
- **Windows Steam's browser needs a workaround** and then runs on the CPU (Gotcha 11).
- **Distribution.** Homebrew disabled the WineHQ casks on 2026-09-01 and Gcenx's `wine-crossover` cask no longer
  exists (`brew info` says so); the free builds are release tarballs, run under Rosetta.
- **Patched Wine means building it.** CodeWeavers publishes CrossOver 26.3.0's open-source code
  (https://www.codeweavers.com/crossover/source); a Wine built from it should include msync but not D3DMetal. Not
  tried. The patch-author recipe (Gotcha 29) took about 25 minutes of compile and did not work here.
- **The audio crash** (Gotcha 9) happens on the free build too.

## When to use it
**Why this note exists.** An agent that wants to mod, study or test a Windows-only game on an Apple Silicon Mac
hits a wall before its usual modding loop can start: `um win` drives games on Windows and WSL only, and the Mac
runs no Windows. The real game has to run first, because the real game is the oracle. Working that out from scratch
costs hours of dead ends: which Wine, which graphics layer, why Steam shows a black window, why a launcher stub
refuses to start. This note records the order to try things in, the versions that worked, what failed and the
gotchas, so the agent can get the game running quickly and go back to its normal loop (recon, route, verify in the
real game).
- The user is on an Apple Silicon Mac, owns a Windows-only game (no Mac build) and wants it running so the agent
  can study or mod it with the real game as the oracle.
- You need to know *quickly and honestly* whether a given game can run, and what is blocking it.
- You are choosing between CrossOver, a free Wine build and other routes, and want the evidence and the costs.

**Scope: what this does and does not mean**

| Situation | Status |
|---|---|
| Unity (Mono) game, Direct3D 11, Steam, no anti-cheat | **Validated** to gameplay, with keyboard/mouse driving and a mod loader |
| Unreal Engine 5 game, DirectX 12, Steam, no anti-cheat | **Validated** to the title screen on DirectX 12 (D3DMetal loaded); gameplay not seen |
| Godot 4.3 single-exe game, Vulkan, **no Steam**, clean bottle | **Validated** to a running race (Vulkan through MoltenVK); scripts recovered; a patched exe was not built |
| Other 64-bit DirectX 11/12 games | Expected to be possible, **untested**, per game |
| DirectX 9/10, OpenGL, other Vulkan games | Other translation paths. **Untested** here (the Halo Trial note under Seen in ran a D3D9 game through mtld3d) |
| 32-bit games | D3DMetal is described as 64-bit Direct3D 11/12 only (Gcenx's Sikarugir README; no Apple or CodeWeavers statement found). **Untested by us**; the [Underrail note](../games/underrail/bg3-style-controller-support.md) played a 32-bit .NET/XNA game in a CrossOver bottle on macOS (its graphics path is not stated there) |
| Kernel anti-cheat (EasyAntiCheat, BattlEye, Vanguard, ACE, ...) | Generally will not run. **Stop and tell the user; never bypass** |
| Launchers other than Steam, installers (`setup.exe`) | **Untested** |
| Intel Macs, other Apple Silicon chips | **Untested** (D3DMetal is Apple Silicon only) |
| Online features that use Epic Online Services | Login **failed** in both games that use it (cause unknown) |

The end-to-end results of the first session came from **CrossOver's command-line tools** (a paid app; no GUI was
opened). The free Gcenx Wine builds passed the graphics probe and booted a Unity game and a Godot game; Windows
Steam first gave only a blank or black window on them. A later session found the cause and a workaround (Gotcha
11): with it, the login UI drew and the contributor's account loaded. No game was launched from that Steam and its
speed was not measured (Gotcha 26).

## Setup (what was used)
- Apple M5 (a MacBook Pro), 16 GB, macOS 26.5.1, Rosetta 2. CrossOver 26.3 (only `cxbottle`, `wine`, `wineserver`;
  bottle template `win11_64`; the bottle's graphics setting was left as created, and D3DMetal is what loaded.
  CodeWeavers documents the setting's "Auto" mode as a per-game choice from its database, falling back to wined3d,
  so another game may get DXMT, DXVK or wined3d instead: check with `lsof` (step 6)).
- Gcenx's Game Porting Toolkit 3.0-3 build (the `game-porting-toolkit` cask from `Gcenx/homebrew-wine`: a binary
  build of Apple's GPTK 1.1 Wine recipe, based on CrossOver 22.1.1 sources; its Wine reports 7.7; Apple's current
  release is GPTK 4, not tested), Gcenx Wine stable 11.0_1 and devel/staging 11.18 (sha256 checked), Homebrew
  `mingw-w64` 14.0.0_3 (to build probes as Windows exes).
- SteamCMD (Homebrew cask, build 1791918946), Windows Steam client build 1791918634, BepInEx 5.4.23.5, GDRE Tools
  2.7.4, ILSpy `ilspycmd` 11.1 on .NET SDK 10.
- Free-route additions (second session, 14-15 October): the `steamwebhelper` wrapper from notpop/steam-on-m1-wine
  (MIT, built with `mingw-w64`; its source lacked `#include <stdarg.h>`), Homebrew `bison` and `flex` plus DXMT
  v0.80 and the codeWonderland/neutron recipe at commit 89084b1 (the rebuild of Gotcha 29), and CodeWeavers'
  CrossOver 26.3.0 source page (read, not built).
- Rules that apply (repo safety rules): only games the user owns; never bypass anti-cheat, DRM or ownership checks;
  never cheat other players; **ask first** before installing anything, editing a bottle's registry, adding files to
  a game folder, driving the user's mouse and keyboard (it steals focus) or publishing; the user types their own
  Steam login; kill processes by exact PID.

## How
1. **Pre-flight, in this order.** (a) Anti-cheat files (`EasyAntiCheat*`, `BattlEye*`, `vanguard`, `Denuvo`) in the
   game folder? If yes, stop. (b) `file *.exe` should say `PE32+ ... x86-64`. (c) Engine and graphics API from the
   store requirements and folder markers (`UnityPlayer.dll`; `Engine/` + `.pak`; a PE section named `pck` is a
   Godot export). (d) Does it need Steam? Look for `steam_api64.dll`, `steam_appid.txt`, `Steamworks.NET.txt`,
   `Engine/Binaries/ThirdParty/Steamworks`. **If none, skip Steam entirely.** (e) macOS permissions the user grants
   once: Screen Recording (window screenshots) and Accessibility (input).
2. **A clean private bottle**, inside a lab folder, never the user's own:
   `CX_BOTTLE_PATH=$LAB/bottles cxbottle --bottle NAME --create --template win11_64`. Run Windows programs with
   `CX_BOTTLE_PATH=$LAB/bottles wine --bottle NAME /absolute/path/prog.exe` (the path must be absolute).
3. **Run a graphics probe first.** A ~350-line C++ program (cross-compiled with
   `x86_64-w64-mingw32-g++ ... -ld3d12 -ldxgi -ld3dcompiler -ldxguid`) creates a D3D12 device, renders a triangle
   through a flip-model swap chain, reads the back buffer with the READBACK-heap pattern in
   [D3D12 frame readback: buffers only](d3d12-frame-readback-buffers-only.md), checks a corner and the centre pixel
   and exits 0 on PASS. It needs no screenshot permission. Observed: feature level 12_2, shader model 6.6, adapter
   `AMD Compatibility Mode` (D3DMetal) under CrossOver; the same under the GPTK 3.0-3 build's Wine 7.7; only
   feature level 11_0 under the free Gcenx Wine 11.x builds (Wine's own DX12 over MoltenVK, no D3DMetal).
4. **Get the game (Steam, no Mac build).** SteamCMD can download the Windows depot of a game the account owns by
   forcing the platform:
   ```
   steamcmd +@sSteamCmdForcePlatformType windows +force_install_dir "$LAB/games/<name>" \
            +login <username> +app_update <appid> validate +quit < /dev/null > download.log 2>&1
   ```
   `+@sSteamCmdForcePlatformType windows` goes first, as in Valve's SteamCMD wiki example (other positions were not
   tried here); `+force_install_dir` before `+login`, as the wiki asks. **The user
   logs in once in their own terminal**; later runs use the cached login
   (`steamcmd +login <username> +quit < /dev/null` should say `Logging in using cached credentials`). If it reports
   no subscription, stop: no other account, no entitlement tricks. Find the app id from the store URL or the public
   `store.steampowered.com/api/storesearch/?term=<name>&cc=us&l=en`. Observed: 1.8 GB and 5.4 GB games downloaded
   in under a minute each.
5. **Windows Steam, only if the game loads Steamworks.** Valve's installer (the store's download page links
   `cdn.akamai.steamstatic.com/client/installer/SteamSetup.exe`; this run used the `cdn.fastly` mirror), run as
   `wine --bottle NAME "$PWD/SteamSetup.exe" /S` (an absolute path, Gotcha 1; `/S` is the NSIS silent switch), then
   start `steam.exe -no-cef-sandbox -cef-disable-gpu -cef-disable-gpu-compositing` from `C:\Program Files
   (x86)\Steam` with the shell's open-files limit raised (`ulimit -Sn 61440`). The first launches self-update (a
   336 MB client package, then a 64-bit one) and each ends with `Update complete, launching Steam... Shutdown`;
   launch again until nothing is pending (3 launches). The user types the login once; "Remember me" makes later
   starts sign in alone (`SetLoginState: Success`). The Steam window may sit on its spinner forever; games only
   need Steam running and logged on. Launch the game **in the same bottle**, and when starting a game exe yourself
   set `SteamAppId`/`SteamGameId` to its app id. If Steam crashes at startup, read Gotcha 9 before anything else.
   This is the CrossOver route; the free Wine route is below.
6. **Run the game windowed and read the evidence.** Pass the engine's own log option (Unity `-logFile <path>`,
   Godot `--log-file <path>`). Check which graphics modules loaded:
   `lsof -p <pid> | grep -i -E "d3d12|d3d11|D3DMetal|MoltenVK|DirectML"`. Screenshot the window by id
   (`screencapture -x -o -l <id> out.png`; ids and owner PIDs from `CGWindowListCopyWindowInfo`, which needs no
   permission). A black window can simply be loading (the Godot game stayed black for about a minute): wait and
   capture again, comparing frames numerically if an animation could fake a change.
7. **Drive it (ask first).** Details in Gotchas 19-21. What worked: focus the game's process (System Events
   `frontmost`), then post key and mouse events to the HID tap; held `W` moved the character, relative mouse deltas
   turned the camera, clicks at `window_origin + screenshot_pixel / 2` pressed menu buttons.
8. **Mod loader and a command bridge (offline sessions only).** BepInEx 5.4.23.5 loaded under CrossOver on a Unity
   6 Mono game once its `winhttp.dll` proxy was made the native DLL **through the bottle registry** (Gotcha 12). A
   plugin that polls a text file and feeds each line to the game's own command API gave a scriptable test console,
   the same idea as
   [Driving a game from a mod-side command file](driving-a-game-from-a-mod-side-command-file-lab-profile-wake.md).
   Never use it against other players.
9. **Stop by exact PID**, then record what worked, what did not, the versions and what you did not verify.

## How: the free Wine route for Windows Steam
What worked on the free **Gcenx Wine Stable 11.0_1** (x86_64 under Rosetta, Gotcha 23). Steam's browser (Chromium)
paints black on Wine 11 for Mac, and Steam's own `-cef-*` options do not change that (Gotcha 11). A tiny wrapper
that makes the real web helper start with `--disable-gpu --single-process` does.
1. **Wine and a private prefix.** Download `wine-stable-11.0_1-osx64.tar.xz` from Gcenx's releases (check the
   sha256 GitHub lists), extract it, create a prefix and install Steam as in How 5 (`wine "$PWD/SteamSetup.exe"
   /S`).
2. **Audio workaround first** (Gotcha 9): the same `mmdevapi` assertion killed Steam on this Wine, so set the empty
   `Audio` driver value in the Steam prefix. Games get sound only from another prefix without that value.
3. **Build the wrapper** from the notpop source (linked above; one include added) with `x86_64-w64-mingw32-gcc -O2
   -municode -DUNICODE -D_UNICODE -o steamwebhelper-wrapper.exe steamwebhelper-wrapper.c -static -lshell32
   -mwindows`.
4. **Install it and launch**, redoing the rename before every launch because a Steam update restores Valve's helper
   (the recipe's own comment; not observed here). The flags, overrides and the `AllowImmovableWindows` setting
   below are from the BCD1210/soju launch script (a tester's recipe, "verified 2026-08-27"); the registry setting
   was set here but not tested without:
   ```
   export WINEPREFIX="$LAB/prefixes/steam"
   export WINEDLLOVERRIDES="bcrypt=b;ncrypt=b;gameoverlayrenderer,gameoverlayrenderer64=d"
   wine reg delete 'HKCU\Software\Microsoft\Windows\CurrentVersion\Run' /v Steam /f  # Steam re-adds it
   cd "$WINEPREFIX/drive_c/Program Files (x86)/Steam/bin/cef/cef.win64"  # also cef.win7, cef.win7x64
   [ "$(stat -f%z steamwebhelper.exe)" -gt 500000 ] && mv steamwebhelper.exe steamwebhelper_real.exe \
     && cp "$LAB/steamwebhelper-wrapper.exe" steamwebhelper.exe
   ulimit -Sn 61440
   cd "$WINEPREFIX/drive_c/Program Files (x86)/Steam"
   wine steam.exe -no-cef-sandbox -cef-single-process -noverifyfiles
   ```
5. **Expect a long first start.** The client downloads about 1.4 GB and updates in several passes (about 25 minutes
   here); the first start may exit at once, in which case start it again. Steam's own log shows
   `steamwebhelper_real.exe --disable-gpu --single-process ...` once the wrapper runs (the wrapper writes
   `wrapper-debug.log` next to itself when `STEAMWEBHELPER_WRAPPER_DEBUG=1`).
6. **Check the evidence, not the clock** (Gotcha 27): a login window with content has hundreds of distinct pixel
   values, the black frames had one; the QR code loads from Steam's servers when the UI is alive.
7. **Stop** with the prefix's own `wineserver -k` (Gotcha 30), then kill leftovers by exact PID.

A non-Steam, non-DirectX example (a Godot 4.3 export, one 174 MB exe): `file` says `PE32+ ... x86-64`; the `pck`
section (90 MB) starts with `GDPC` and its header gives pack format 2 and Godot 4.3.0 with an unencrypted directory
of 446 files (scripts as compiled `.gdc`). In a clean bottle with no Steam and nothing else installed it logged
`Vulkan 1.2.290 - Forward+ - Using Device #0: Apple - Apple M5` (Vulkan through MoltenVK and Metal) and reached a
race with its HUD. GDRE Tools' headless mode (`--recover=<exe> --scripts-only`) decompiled all 45 scripts; its help
lists `--compile ... --bytecode=4.3.0` and `--pck-patch` as the route to a modded exe, which was **not built or
run**. Keep decompiles outside any repository.

## Verification
What was observed, per game (two sessions, 13-15 October 2026):

| Game | Result |
|---|---|
| **PEAK** (Steam app 3527290, Unity 6000.3.15f1 Mono, v2.6.b) | GPTK 3.0-3 build (Wine 7.7): crashed right after input init. Gcenx 11.x: booted on Vulkan (Apple M5 via MoltenVK), stalled at the Steam check. **CrossOver CLI + signed-in Windows Steam:** Direct3D 11 (`Direct3D 11.0 [level 11.1]`, renderer `AMD Compatibility Mode`; Unity's own filter denied DX12), reached gameplay (airport, island), keyboard and mouse driven, BepInEx plus a plugin loaded, a command-file bridge ran the game's console commands with verified on-screen effects (Gotcha 17). Epic login failed |
| **MECCHA CHAMELEON** (Steam app 4704690, Unreal Engine 5.6 by its config folder; an online PvP game, with no single-player mode on its store page: nothing was injected into it, and the mod-loader step 8 doesn't apply to it) | SteamCMD download 5.4 GB. The launcher stub refused to start (Gotcha 5); the real shipping exe ran: **title screen on DirectX 12** (`lsof` showed `d3d12.dll`, `dxgi.dll`, `DirectML.dll`, D3DMetal, `libmetalirconverter`). Epic login failed. Gameplay not seen; one Enter key press was inconclusive |
| **A small Godot racing game** (the contributor's own export) | no Steam, no extra install: Vulkan 1.2 Forward+ via MoltenVK, reached a race with HUD (position, lap, timer, leaderboard, boost). Scripts recovered 45 of 45. A patched exe was not built |
| **The same Godot game on free Wine Stable 11.0_1** (fresh prefix, sound on) | `Vulkan 1.2.334 - Forward+ - Using Device #0: Apple - Apple M5`; the contributor played it: the menu drew, starting a game took a long time and then lagged. The Mac was overloaded at the time (Gotcha 26), so no performance conclusion. Window screenshots by id were black while it ran (Gotcha 27) |
| **Windows Steam on free Wine Stable 11.0_1 with the wrapper** | login window with content (238 distinct pixel values; the earlier black frames had 1), QR code loaded from Steam's servers; the contributor's account loaded and the Steam window opened (reported by the contributor). **No game launched from it**; speed not measured |
| **The same Steam on Wine Devel and Staging 11.18, without the wrapper** | the login window was a uniform black frame (valid captures, Gotcha 27); the same with the `-cef-*` flags, and a Wine virtual desktop gave only a black helper window |

Checked only weakly: sound (the game process loaded `winecoreaudio.drv`; nobody listened), performance (no frame
rate or load times; Gotcha 26), input outside PEAK, any online feature.

## Gotchas
1. **Symptom.** `cannot execute L"game.exe"` from CrossOver's `wine`. **Cause:** the wrapper wants an absolute
   path. **Fix:** pass `"$PWD/game.exe"`.
2. **Symptom.** A Unity 6 game logs `Crash!!!` right after `<RI> Initializing input.` (stack in `UnityMain2`).
   **Cause:** Wine too old; the GPTK 3.0-3 build is Wine 7.7. **Fix:** use CrossOver's Wine or
   Gcenx Wine 11.x. Do not debug the game.
3. **Symptom.** `[D3D12 Device Filter] ... D3D12 API denied by user filter` (Unity). **Cause:** the game's own
   filter (feature level below 12.1 or 4 cores or fewer). **Fix:** none needed; Unity falls back to D3D11 or
   Vulkan. Do not force DX12.
4. **Symptom.** `d3d12: could not load DirectML.dll`, `failed to create D3D11On12 device`. **Cause:** D3DMetal path
   without DirectML. **Fix:** harmless; the game continues on D3D11.
5. **Symptom.** An Unreal game's launcher shows a window titled `Error`: "The following component(s) are required
   to run this program: Microsoft Visual C++ 2015-2022 Redistributable (x64)". **Cause:** the root `<Project>.exe`
   is a prerequisite stub. Its machine code and a registry trace show it reads
   `HKLM\SOFTWARE\Microsoft\VisualStudio\14.0\VC\Runtimes\x64` (`Major`, `Minor`, `Bld`; needs 14.42.34438 or
   newer), then loads `system32\msvcp140_2.dll` and checks its version. In the tested bottle the registry and every
   runtime DLL were 14.51.36247 and the DLL loaded, yet the dialog stayed; which step fails was **not found**.
   **Fix:** stop installing redistributables. Run
   `<Project>/Binaries/Win64/<Name>-Win64-Shipping.exe -windowed -ResX=1280 -ResY=720` directly, or use a 280 KB
   launcher that starts that exe with the same arguments. This is a prerequisite check, not DRM, but check for
   anti-cheat files first, and swapping a stub inside Steam's library edits a game file (Steam's "verify files"
   restores it): ask the user.
6. **Symptom.** A Godot game shows a black window for the first minute. **Cause:** loading and first-run shader
   compilation. **Fix:** wait and re-capture; the Godot log should show `Vulkan ... Forward+ ... Apple`.
7. **Symptom.** `SteamAPI_Init() failed` / endless `Steamworks is not initialized`. **Cause:** no Steam client in
   the game's bottle. **Fix:** install and sign in to Windows Steam in the **same** bottle (How 5).
8. **Symptom.** "Updating Steam..." never advances at 0% CPU; the bootstrap log says
   `Update complete, launching Steam... Shutdown`. **Cause:** the updater hands off and hangs. **Fix:** kill that
   `steam.exe` by exact PID and launch again until no update is pending.
9. **Symptom.** Windows Steam crashes about 20 s after start: the web helper logs `Quit message loop`,
   `assert_steam.exe_*.dmp` files appear (`strings -a <dump> | grep Assertion`; quote the path, it has spaces), and
   the **first** line of the launcher output is `Assertion failed: !status, file
   ../../wine/dlls/mmdevapi/mmdevapi_private.h, line 120`, followed by `Illegal termination of worker thread
   'CFileWriterThread'`, `Failed to create thread (error 0x5)` and `Waited 6000 iterations in CFileWriter::Flush`.
   **Cause (mechanism):** `wine_unix_call()` in mmdevapi calls the Mac-side CoreAudio driver and then does
   `assert(!status)`, so any CoreAudio error aborts; the same code is in upstream Wine 11.0 and 11.18 (line 120 in
   11.0, which CrossOver 26 is based on; line 118 in 11.18). It also killed Steam on the free Gcenx Wine Stable
   11.0_1 in another prefix (the same first line, line 120), so it is not CrossOver-specific. It is **intermittent
   and environment-dependent**: Steam started fine with audio on for about 45 minutes, then failed on every start
   until audio was disabled, with nothing changed in Wine or the bottle. The Mac had 9 output and 7 input devices
   including virtual ones (BlackHole, Zoom, Teams drivers) and a USB default output; none was proven to matter.
   **Not found:** the failing call. A probe that lists every device, reads each mix format and starts a real
   shared-mode stream ran 14 times without failing. **Rejected as the cause:** the open-files limit (it crashed
   with the limit raised), a cold Wine server, the Visual C++ runtime install, a bottle-level `WINEDLLOVERRIDES`
   entry. **Fix (workaround only, apply only when the first line above is present; ask first, it edits the
   registry):** back up the bottle's `user.reg`, set `HKCU\Software\Wine\Drivers` value `Audio` to empty (`wine
   --bottle NAME reg add 'HKCU\Software\Wine\Drivers' /v Audio /t REG_SZ /d '' /f`), start Steam, and when it says
   `Logged On` delete the value (`reg delete ... /v Audio /f`). Steam keeps running without sound and **games
   started afterwards get sound** (their process loads `winecoreaudio.drv`; check with `lsof`). If the mmdevapi
   line is absent, it is a different problem. On the free build the value was left empty in the Steam prefix for
   the whole session.
10. **Symptom.** The same Steam crash-loop but with file-writer assertions only. **Cause:** not the audio issue.
    **Fix:** check the open-files limit (`ulimit -Sn`, macOS Terminal defaults to 256; raise to 61440 for the
    launching shell), a hung updater, or a bottle-level `WINEDLLOVERRIDES` entry. None of these fixed the audio
    crash.
11. **Symptom.** Windows Steam shows blank white boxes (stable 11.0) or a solid black login window (devel/staging
    11.18) under the free Gcenx Wine builds. **Cause:** Steam's Chromium paints black on Wine 11 for Mac. Steam's
    own `-cef-single-process`, `-cef-disable-gpu` and `-cef-disable-gpu-compositing` options changed nothing here,
    and the codeWonderland/neutron notes say such options are not passed on to Chromium (`docs/phase0-spike.md`,
    linked above; the DXMT thread 3Shain/dxmt#141 tracks the black window). Observed A/B on one Wine, prefix and
    client: black with those options; a login window with content once a wrapper started the real helper
    (`steamwebhelper_real.exe`) with `--disable-gpu --single-process` itself. A Wine virtual desktop made it worse
    (the web helper quit). **Fix:** the wrapper route (How, free Wine route; source: notpop/steam-on-m1-wine,
    launch recipe: BCD1210/soju). **Costs:** Steam's UI is then CPU-rendered and single-process (Gotchas 26 and
    28). **Not tested:** the wrapper on Devel/Staging 11.18, signing in from a cold start, a game launched from
    that Steam. The GPTK 3.0-3 build (Wine 7.7) cannot run the current client at all: its bootstrapper never
    downloads the packages (`Failed to load steamui.dll`), and with the packages copied in from the Wine 11 prefix
    it dies with an unhandled page fault.
12. **Symptom.** BepInEx never starts (no `BepInEx/LogOutput.log`). **Cause:** the `WINEDLLOVERRIDES=winhttp=n,b`
    environment variable is ignored by CrossOver's wrapper, and so is the bottle's `[EnvironmentVariables]` entry.
    **Fix:**
    `wine --bottle NAME reg add 'HKCU\Software\Wine\DllOverrides' /v winhttp /t REG_SZ /d native,builtin /f` (back
    up `user.reg` first). Proof: the log appears (`BepInEx 5.4.23.5 - <Game> ... Running under Unity v6000.3.15`).
13. **Symptom.** `WINEDEBUG=+loaddll` prints nothing through CrossOver. **Cause:** the wrapper ignores the plain
    variable. **Fix:** `wine --bottle NAME --debugmsg +reg,+loaddll game.exe`. CodeWeavers' own support pages
    document the `CX_DEBUGMSG` and `CX_LOG` environment variables for the same job.
14. **Symptom.** Steam and the game die together. **Cause:** `wineserver -k` stops every process in the bottle.
    **Fix:** kill the game's exact PID instead; only run `wineserver -k` when nothing else shares the bottle.
15. **Symptom.** The Mac Steam app shows "Refresh Sign In" after a SteamCMD login, or a Windows Steam download is
    disturbed. **Cause:** SteamCMD logging in as the same account ends other sessions, and it shares
    `~/Library/Application Support/Steam` with the Mac app. **Fix:** do not run SteamCMD while a Steam download or
    game is running; warn the user.
16. **Symptom.** Unreal title screen: "Failed due to invalid or missing authentication token for user (e.g. not
    logged in)"; Unity: `LoginAsync result: ConnectExternalTokenValidationFailed`. **Cause:** Epic Online Services
    rejected the Steam login in both engines while Steam was signed in; **unknown**. **Fix:** none found; not
    blocking for rendering or offline tests. Untested: launching through Steam's Play button.
17. **Symptom.** A console command is "found and run" but nothing changes. **Cause:** the game's logic ignored it
    (PEAK ignores status effects in its airport scene; the same `AddPoison` put a poison segment on the stamina bar
    on the island, `ClearAll` removed it, and `SetTimeOfDay 2` turned the sky to night, mean brightness 183 to 66).
    The console command's domain can also be a renamed class (`CharacterAfflictions` is exposed as `Afflictions`;
    the wrong name returns "NOT FOUND"). **Fix:** verify every effect by screenshot or log, in the right scene. See
    [Oracles](oracles-how-agents-know-a-mod-works.md).
18. **Symptom.** `screencapture`: `could not create image from window/display/rect`. **Cause:** Screen Recording
    permission missing for the terminal's host app. **Fix:** ask the user to enable it in System Settings, Privacy
    & Security, Screen & System Audio Recording, then quit and reopen the host app.
19. **Symptom.** Synthetic keys do nothing. **Cause:** `CGEvent.postToPid` is ignored while another app is
    frontmost; Wine's macOS driver only reacts when its window is active. **Fix:** focus the game first
    (`tell application "System Events" to set frontmost of (first process whose unix id is <pid>) to true`), then
    post to the HID tap. Needs Accessibility (check with `AXIsProcessTrusted`); it steals focus, so tell the user
    first ([Driving real games safely](driving-real-games-safely.md)). The HL2-under-Wine note (gotcha 13) saw the
    same in a Sikarugir wrapper: `NSRunningApplication.activate` can't bring Wine to the front, System Events can.
20. **Symptom.** A click lands in the wrong place. **Cause:** screen points, not pixels (Retina screenshots are
    2x), and zsh does not word-split unquoted variables (`set -- $VAR`). **Fix:** compute
    `window_origin + pixel / 2` and parse window bounds in Python.
21. **Symptom.** Escape does not close a pause menu. **Cause:** not found. **Fix:** click its Back button; confirm
    with before/after screenshots because the game's log rarely records menu actions.
22. **Symptom.** `brew install --cask wine-stable` says "disabled ... Gatekeeper". **Cause:** Homebrew disabled the
    WineHQ casks on 2026-09-01. **Fix:** use the Gcenx release tarballs (check the sha256 GitHub lists) or
    CrossOver.
23. **Symptom.** `Error: spawn Unknown system error -86` or x86_64 programs will not start. **Cause:** Rosetta
    missing. **Fix:** ask, then `softwareupdate --install-rosetta --agree-to-license`.
24. **Symptom.** `zsh: no matches found: https://...?per_page=3`; or `grep` rejects `-P` / a file named
    `-Something.cs`. **Cause:** `?` is a glob in zsh; the shell may alias `grep` to `ugrep`, and leading-dash
    filenames read as options. **Fix:** quote URLs; call `/usr/bin/grep` and use `./*.cs`.
25. **Symptom.** Leftover `winewrapper.exe` processes after a run. **Cause:** wrapper processes outlive the game.
    **Fix:** kill those exact PIDs (see Gotcha 14).
26. **Symptom.** Windows Steam and its UI feel several times slower on the free Gcenx Wine 11 builds than on
    CrossOver (the contributor's report; no timing was taken). **Cause:** not isolated. Candidates: the wrapper
    makes Steam's browser CPU-rendered and single-process by design; and no msync (`strings` on
    `lib/wine/x86_64-unix/ntdll.so` finds none of `msync`, `esync`, `ntsync` in Gcenx Stable 11.0_1 or
    Devel/Staging 11.18, but msync in CrossOver 26.3 and in the GPTK 3.0-3 build), so every lock and event goes
    through the Wine server. Setting `WINEESYNC` or `WINEMSYNC` should do nothing there (the neutron launcher says
    "ignored by Wine builds without msync"; not tested separately). The Mac was also overloaded by unrelated editor
    processes (load average about 128 on 10 cores, `sysctl -n vm.loadavg`), so the comparison is not clean.
    **Fix:** none found; check the load before timing anything. A Wine built from CodeWeavers' CrossOver 26.3.0
    source should contain msync; **not tried**.
27. **Symptom.** A window screenshot is uniformly black (one distinct pixel value) though the program may be fine,
    or it shows a different run's window. **Cause (observed):** a locked or sleeping Mac makes every capture black,
    full screen included; windows of processes left over from earlier runs get captured instead of the new one; and
    the window-id capture of a Vulkan game window was black while the contributor was playing it. **Fix:** check
    that `ioreg -n Root -d1 -a | grep -A1 CGSSessionScreenIsLocked` does not say `true`; take the window id from
    the process you just started (compare its age with `ps -o etime= -p <pid>`); count distinct pixel values
    instead of trusting a glance; compare against a window whose content you know (Steam's update dialog drew fine
    in the same runs); treat a black Metal or Vulkan window as unknown, not as a failure.
28. **Symptom.** Wine shows "Program Error Details" and "Wine Debugger" windows (`winedbg --auto <pid>`), the Steam
    window reloads by itself, or the launcher output has `Unhandled exception 0x80000003`. **Cause (observed, not
    diagnosed):** with the wrapper's `--single-process`, a crash anywhere in Steam's browser takes the whole web
    helper down; Steam starts a new one while Wine's auto-debugger stays attached to the dead one. The dialog lists
    modules only. **Fix:** close the dialogs and kill that `winedbg` PID. Disabling the auto-debugger in the
    registry was not tried.
29. **Symptom.** Wine 11.18 rebuilt from the codeWonderland/neutron recipe (`winemac` patches, DXMT v0.80 DLLs
    rebuilt with its cross-process patch, fresh prefix made by that Wine, Steam's autostart value deleted,
    `Steam.exe -no-cef-sandbox`) shows **no** Steam login window at all, not even a black one. **Cause
    (observed):** Steam creates the window without the visible flag and it never becomes visible (presumably Steam
    shows it only after a first frame; not confirmed). Chromium's GPU process prints `CreateSwapChain: window
    belongs to process <pid>; presenting remotely`, then `Failed to create metal view, it seems like your Wine has
    no exported symbols needed by DXMT.`, and DXMT aborts that process (`d3d11_swapchain.cpp` in v0.80), so no
    frame can arrive. `macdrv_functions` was exported (`nm -gU`). Reading the sources, the patched Wine's
    remote-layer code is only reached from the Vulkan path while DXMT's `winemetal` asks `get_win_data()` for the
    foreign window, which returns NULL; that gap is **not proven** to be the cause. The patch author reports a
    rendering login window on an M1 under macOS 15.5 and games on macOS 27.0.1 (`docs/phase0-spike.md`); not
    reproduced on macOS 26.5.1 with an M5. **Fix:** none; use the wrapper route on Wine Stable 11.0_1.
30. **Symptom.** After several runs `ps` shows old `steam.exe`, `steamwebhelper`, `explorer.exe /desktop`,
    `winedbg` processes; or a different Wine opening an existing prefix shows "The Wine configuration ... is being
    updated" for 2 to 5 minutes. **Cause:** each prefix has its own Wine server, nothing exits when its parent
    does, and a prefix is upgraded by the first different Wine that opens it. **Fix:** stop a run with
    `WINEPREFIX=<prefix> wineserver -k` from the same Wine build, never with one that belongs to another prefix,
    then list what is left and kill it by exact PID; use one prefix per Wine build when comparing builds.

## Seen in
- PEAK, MECCHA CHAMELEON and a small Godot racing game (above); no per-game notes exist yet.
- Related notes:
  [Running xEdit and other Windows modding tools on Linux](running-xedit-and-windows-modding-tools-on-linux.md)
  (the Wine/Proton counterpart),
  [Half-Life 2 guns in Minecraft (HL2 Demo under Wine on macOS)](../games/minecraft/half-life-2-guns-in-minecraft-passthrough-hl2-demo-under-win.md)
  (Windows Steam in a Sikarugir wrapper, Wine 10 with DXVK),
  [Max Payne bullet time in Halo Trial on macOS/Wine](../games/halo-trial/max-payne-bullet-time-shootdodge-and-voice-in-halo-trial-via.md)
  (a D3D9 game under GameHub's Wine with mtld3d, D3D9 to Metal),
  [BG3-style controller support for Underrail](../games/underrail/bg3-style-controller-support.md)
  (a 32-bit .NET/XNA game played in a CrossOver bottle on macOS with a DualSense, about 15 playtest runs; its
  gotcha 11, the mouse position reading back a pixel off under Wine, is a related input quirk),
  [D3D12 frame readback: buffers only](d3d12-frame-readback-buffers-only.md),
  [Oracles](oracles-how-agents-know-a-mod-works.md), [Driving real games safely](driving-real-games-safely.md).
- Third-party reports of other games in CrossOver with D3DMetal (not verified here): Monster Hunter: World
  (Direct3D 11 on a Direct3D-on-Metal layer) and Elden Ring (Direct3D 12), each bridged to Minecraft by an outside
  repo. See [Choosing a mashup route](choosing-a-mashup-route.md),
  [Frame compositing: depth, pose sync, lighting and stale frames](frame-compositing-depth-and-pose-sync.md),
  [Bridge contracts for two-game mods](bridge-contracts-ownership-units-and-lifecycle.md) and the case list in
  [mashup-cases.md](../../skills/mashup-mods/references/mashup-cases.md).
- The [game-automation skill](../../skills/game-automation/SKILL.md) has a macOS section (`screencapture -l`,
  `osascript` or `cliclick`, the Accessibility and Screen Recording permissions): the same tools as Gotchas 18-20.
- Difference from the HL2 note: this one uses CrossOver's command-line tools end to end and adds SteamCMD
  downloads, Windows Steam in the bottle, a DirectX 12 game and a non-Steam game, and it compares the free Gcenx
  Wine builds with CrossOver (routes table, Gotchas 11 and 26-30).

## Open questions
- **Epic Online Services rejecting the Steam login** in two engines (Gotcha 16). Untested: Steam's own Play button,
  the ticket identity EOS expects, clock or network.
- **The Steam audio crash trigger** (Gotcha 9): it also happened on plain free Wine; still to do: run Steam with
  `WINEDEBUG=+mmdevapi,err`, with and without the virtual audio drivers and with Zoom/Teams/Logic running.
- **Why the Unreal stub rejects an installed C++ runtime** (Gotcha 5): both checks pass on paper; a debugger or
  deeper trace is needed.
- **Free Windows Steam:** signing in from a cold start and launching a game from it were not tested by the agent;
  the wrapper was not tried on Devel/Staging 11.18; it was not checked that the wrapper survives a Steam update;
  speed on an unloaded Mac against CrossOver (Gotcha 26) is unmeasured.
- **A GPU-drawn Steam UI on free Wine** needs the cross-process Metal path (DXMT thread 3Shain/dxmt#141) to work on
  this macOS; it did not (Gotcha 29). Why it works for its author on macOS 15.5 and 27.0.1 is open.
- **Wine built from CodeWeavers' CrossOver 26.3.0 source** (msync, Mac patches; no D3DMetal): not built or tested.
- **Untested:** other chips and macOS versions, Intel Macs, DirectX 9/10, OpenGL, other Vulkan games, 32-bit games,
  installers, launchers other than Steam, anti-cheat games, controllers, performance, actually listening to the
  sound, a Steam game started from Steam's Play button, a patched Godot export.
- **Tooling the repo lacks:** the game-automation skill covers macOS by hand, but there is no `um mac` command
  group mirroring `um win` (launch in a bottle, list windows, screenshot, focus, key/mouse, kill by PID) and `um
  scan` detecting CrossOver bottles. Suggested after maintainer discussion, with tests. Versions here were current
  in October 2026 and will age (CrossOver, Wine builds, SteamCMD, the Steam client): re-check the current release.


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

