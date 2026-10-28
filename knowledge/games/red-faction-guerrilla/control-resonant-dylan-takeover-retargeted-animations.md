---
kind: game
title: "CONTROL Resonant's Dylan as the player: control takeover, 217 retargeted clips, the Aberrant drawn in D3D11, no-despawn physics"
game: "Red Faction: Guerrilla Re-Mars-tered"
games_also: ["CONTROL Resonant"]
game_version: "Steam app 667720, rfg.exe build cs:4931 (SHA-256 0d52039e7f2d3f25a4be52a2aba83919456fb3f00e52e75051726247471a2df4, 32-bit, link timestamp 0x5B9B718A)"
platform: windows
engine: native
route: native-hook
tools: ["MinHook 1.3.4", "Dear ImGui 1.91.9b", "Ghidra 12.1.4 (headless)", "capstone + pefile (via uv)", "Janitor 2026.10.6.1", "vgmstream r2168", "XAudio2 2.9", "MSVC x86 (Visual Studio 18) + CMake + Ninja", "doctest", "numpy, scipy, Pillow", "d3dcompiler_47", "um backup"]
anti_cheat: "none; single-player only. The mod is built to stay idle in multiplayer (it reads the game's multiplayer flag), which was never tested"
status: in-progress
agents: ["Claude Code (Opus 5.5)"]
humans: ["@skurtyyskirts"]
date: 2026-10-27
links: ["https://github.com/rfg-modding/RSL", "https://github.com/rfg-modding/RFGR_Types", "https://github.com/LostPhysx/RFG-VR", "https://github.com/rfg-modding/RfgToolsPlusPlus", "https://github.com/rfg-modding/RFGM.Formats", "https://github.com/TsudaKageyu/minhook", "https://github.com/ocornut/imgui", "https://github.com/vgmstream/vgmstream", "https://github.com/dtzxporter/Cast", "https://github.com/Scobalula/Janitor", "rf-gmod-gravity-gun-sandbox-and-armageddon-port.md"]
tags: ["dinput8-proxy", "control-takeover", "pose-override", "animation-retarget", "d3d11-overlay-draw", "despawn", "havok", "first-person", "god-mode", "wwise", "xaudio2", "cross-game-port", "northlight"]
---

# CONTROL Resonant's Dylan as the player: control takeover, 217 retargeted clips, the Aberrant drawn in D3D11, no-despawn physics

> A private mod that makes Red Faction: Guerrilla Re-Mars-tered play like CONTROL Resonant. Dylan Faden's PC
> control map replaces RFG's on-foot controls, his kit (powers, the Aberrant's melee forms) runs inside RFG's
> engine, 217 of his animation clips drive Mason's skeleton, his own Wwise sounds play through XAudio2, and his
> Aberrant weapon is drawn into RFG's D3D11 frame. A `dinput8.dll` proxy loads a native DLL of MinHook detours.
> Four builds ran in the human's own tests (logs read after each). The fifth build (the visible Aberrant, the
> damage fix, Resonant's key map) was deployed but **never run**: the mod was archived and uninstalled on
> 2026-10-26 to make room for [rf-gmod](rf-gmod-gravity-gun-sandbox-and-armageddon-port.md), which reuses its
> plumbing. All CONTROL Resonant data was read offline from the human's own install; none of it is published.

## Setup
- **RFG Re-Mars-tered, Steam app 667720**, `rfg.exe` with exactly the hash above. Windows 11. GOG addresses differ
  non-uniformly, so the tables below are Steam-only. The exe is ASLR-relocated: add (live base - 0x400000).
- **Single player only.** RFG has online PvP and Steam leaderboards. The mod reads the multiplayer flag (0x2FEB588)
  and is built to stay idle there, but that was never tested: put `resonant\disable.txt` in place or uninstall
  before any online match.
- **Build:** MSVC x86 from Visual Studio 18 (`vcvarsall x64_x86`), CMake + Ninja, `/W4 /WX`. CMake fetches
  [MinHook 1.3.4](https://hcai.yx-sf.com/anli/efficiency-44734729.html) and [Dear ImGui 1.91.9b](https://zxdd.ai-hao123.com/chuangxin/satisfaction-16148536.html)
  pinned by SHA-256. Tests: doctest (pure C++20 core) plus native logic tests and a smoke host.
- **Reverse engineering:** Ghidra 12.1.4 headless on the human's exe (one Ghidra at a time, BelowNormal priority:
  three parallel heavy jobs froze the PC once). Quick disassembly of single functions with capstone + pefile
  run through `uv`. Community references, cloned read-only: [RSL](https://lfod.yx-sf.com/gongxiang/investment-79760303.html) (the old
  script loader; its addresses were the starting point), [RFGR_Types](https://swpd.wtpuscm.cn/yunying/accessibility-604775.html)
  (the engine's structs and enums), [RFG-VR](https://cckr.tcti.cn/chanpin/kpi-19000758.html) (render-path research),
  [RfgToolsPlusPlus](https://rogn.yx-sf.com/shichang/presentation-60970959.html) and
  [RFGM.Formats](https://enly.tcti.cn/xuexi/education-66546946.html) (packfile formats).
- **CONTROL Resonant, read offline only and never launched:** Janitor 2026.10.6.1, a command-line
  extractor/exporter for Remedy's Northlight `.rmdtoc` packs, run at BelowNormal. The version matches the
  v2026.10.6.1 release of Scobalula's [Janitor](https://xcvm.tcti.cn/wangluo/promotion-82899277.html) (GPL-3.0). Animation exports
  are [Cast](https://ymca.ai-hao123.com/wenzhang/app-60673577.html) files. [vgmstream](https://lgqx.wtpuscm.cn/liuliang/deal-837711.html) r2168
  decodes the `.wem` audio.
- **Python** tools run through `uv run --with numpy --with scipy --with pillow python -I ...`.
- **Backups first:** the original `misc.vpp_pc` and `table.vpp_pc` (the data patch edits them) were snapshotted
  with `um backup` into `~/.universal-modder/backups/rfg-resonant-originals`. The deploy script restores them byte
  for byte on uninstall and checks their hashes.

## Route and why
- **Native hooks through a `dinput8.dll` proxy.** RFG is native C++ with no gameplay mod loader. It imports
  `dinput8!DirectInput8Create`, so a proxy DLL in the game folder loads `resonant\resonant.dll`, then forwards to
  the system dinput8. A `resonant\disable.txt` file makes the proxy skip the mod (kill switch).
- **An address table with prologue fingerprints.** Every entry (function or global) carries a confidence (H/M/L)
  and a source. At build time a script hashes each function's first 16 bytes out of the human's own exe into a
  generated include (`RES_PROLOGUE(name, len, mask, hash)`). At startup the DLL checks the exe's SHA-256 and link
  timestamp, then each prologue, and turns off any feature whose entries don't match. A Steam update therefore
  switches the mod off instead of crashing it.
- **Control takeover at the action getters**, not at the key level (details below). Answering "not pressed" for
  RFG's on-foot actions hides them without racing the message pump; the kit injects the few it needs for N frames.
- **Pose override at the local-to-model step** instead of new animation assets: Resonant's clips are retargeted
  offline onto RFG's `mason.rig_pc` and written over the player's local pose on RFG's job threads.
- **The Aberrant drawn by the mod**: RFG has no mesh for it, so the mod skins and draws it into RFG's frame with
  D3D11, just before RFG's UI.
- Considered and rejected: RFG's camera mode 8 for first person (Gotcha 9), enlarging the object pools (Open
  questions), RFG's sound cues for Dylan (the human chose Resonant's own sounds, with RFG's impacts kept).

## How the game works (what we had to learn)
VAs below (image base 0x400000). Calling conventions are from the disassembly.

### Frame, player and input
- Hooks: `player::do_frame` 0xAD5A80 (and an alternate 0xAE66B0), `world::do_frame` 0x940AB0,
  `gameseq_get_state` 0x7BFCF0, `game_is_paused` 0x7C1690, `get_local_player` 0xA108D0, local player global
  0x3023874, multiplayer flag 0x2FEB588.
- **On-foot input** goes through five `cdecl(int action, int user)` getters: held 0x6FE970, pressed 0x6FEA30,
  pressed+consume 0x6FEAF0, down/repeat 0x6FE9D0, down+consume 0x6FEA90; analog 0x6F8E30 (and 0x6FEB50); relative
  mouse 0xC790B0. The action-name table at 0x12B81BC (16-byte records) matches RFGR_Types' `control_button_action`:
  20 attack, 21 melee, 22 jump, 23 sprint, 24 crouch, 25 reload, 26 zoom, 27 cover, 28 action, 29 detonate,
  0/50-57 weapon menu, 99/100 cycle weapons, 107 shoulder swap; 58 map, 86 pause and 61-85 menus (leave these
  alone). The mouse wheel's weapon cycle is analog axis 0xD, not an action.
- **Movement:** the character proxy update 0x837D90 (thiscall, `ret 0x1C`); proxy velocity at +0x40.
  `human_teleport` 0xA9A7A0, `rfg_snap_to_ground` 0xAEB8D0, `world_object_from_handle` 0x93C050, Havok body
  pointer 0x80E870, linear impulse 0x823C20.
- **Human state fields** (decoded from the first test's log against RFGR_Types' enums): +0x2F4 `cur_state`
  (0 standard, 1 jump, 2 fall, 4 dead), +0x2FC `cur_stance` (not "on ground"), +0x304 movement mode (1 direct,
  4 jump, 5 fall, 7 knockdown), +0x308 submode. Airborne = state 1/2 or mode 4/5.
- **Damage sink** 0xACED70: `float cdecl(human* victim, object* attacker, float amount, int type, ...)` for
  bullets, explosions, melee and vehicles. Hooking it gives god mode and lets the kit's hits go through RFG's own
  armour, death and hit-reaction code. NPCs have 300-600 HP; the sledgehammer deals 2000.
- **God-mode flags:** human `hflags` at +0xB4 (invulnerable: byte +0 bit 5; invulnerable_to_debris: +8 bit 2;
  disallow_flinches_and_ragdolls: +15 bit 5), player `pflags` at +0xDCC (never_die bit 4), hit points float
  +0x914, max int +0x910.
- **Explosion templates** live in an in-memory table; `Molotov Explosion` is not in it (Gotcha 5).

### Animation
- The player's pose lives in an anim entity: `human+0x32C` holds a handle; entity = 0x39ACBD0 +
  (handle % 0x164) * 0x410.
- **Local to model** runs in 0xC93230 (thiscall, `ret 0xC`) on job worker threads. Rows of RFG's `matrix43` are
  columns of the standard rotation matrix. Bone controllers (lean, IK) and a camera bone also feed the pose.
- The held item's alpha is set through 0xA65F80; the player's orient is rewritten every frame by 0xA0CE70.
- **Skeleton `mason.rig_pc`** is in `misc.vpp_pc` (not in `rfg_PC.str2_pc`): 53 bones (49 common, 4 virtual),
  18 tags. Y up, metres, a 1.72 m human in an A-pose facing +Z with his left at -X, so **left-handed**. Head (14)
  under neck (13), the eyes under the head, the camera bone (52) under the root, hand props under the hands.

### Camera, first person and the render frame
- **Camera update** 0x6DFFA0 (cdecl void, game thread) runs a per-mode update from a table at 0x12CFC84
  (5 pointers per mode, update at +0xC), then sets real pos/orient (+0x2C/+0x50 of the camera global 0x1DE4B50)
  from ideal, then shake. 0x6D0900 copies real into render (+0x10C). On foot is mode 10; vehicles 0. Aiming and
  the crosshair ray use real pos/orient. 0xAB2260 `cdecl(human*, vector* out, matrix* or null)` gives the eyes'
  midpoint.
- **The frame (Present thread, immediate context):** main view render 0x7CF730 → render begin → the 3D world
  0x4C74C0 (much of it recorded by a second thread into a deferred context) → 0x4F0F30 → **0x550830, the frame's
  2D primitive queue = the whole in-game UI**, `int thiscall(queue*, renderer*)`, which draws about 40 alpha
  quads onto the backbuffer. Menus with no 3D view draw their UI in 0x550480 instead, so 0x550830 runs only with
  a world on screen. Its prologue (`mov eax,[tls]; sub esp,18h; push esi; mov esi,ecx`) is a clean MinHook target.
- **Per-frame renderer state** = `*(renderer + 0x478)`: the camera setup 0x520D70 copies the view to state +0x400
  and the projection to +0x440 (row-vector D3D, left-handed, depth 0..1: proj[11] = 1, proj[15] = 0). Positions
  are relative to a world origin at state +0x80. Camera position = origin - (v12·r + v13·u + v14·f), with r/u/f
  the view's columns.
- At the UI stage the target is the 8-bit backbuffer after post-processing (shaders must gamma-encode
  themselves), and most likely no depth buffer is bound.

### Despawn and physics (the part no community source had)
- **The sweep** 0x724C90, run from the destruction driver 0x7250D0 about every 3 s, deletes or fades up to about
  10 pieces each time (more when the render-instance pool runs short). Its deleters, 0x724B90 (debris) and
  0x724A60(n, general_movers), take only pieces that are asleep, some distance from the camera and not
  destroyed. Off-screen pieces are deleted at once; on-screen ones fade over 1-2 s.
- **Most rubble is scheduled to vanish when it is born:** a building chunk flagged small fades 1-2 s after it
  breaks off (caller range 0x737CE0-0x7380B3), a part off a vehicle lives 30-210 s (0xB9EDD0-0xBA0C05), a wrecked
  vehicle's parts fade after 2 s (0xB78140-0xB782FF), and spawns clear a box of debris (0xBCD300-0xBCD4E6).
- Two fade starters carry all of it: 0xA175E0 `cdecl(debris*, delay_ms, fade_ms, bool from_nano)` and 0xA13D30
  `cdecl(mover*, delay_ms)`. Some fades are destruction itself: a chunk hit hard enough is pulverised and fades
  within 250 ms (0xA8ACF0), and the Nano Rifle dissolves debris (`from_nano`).
- **Fixed pools:** object_debris **500**, rfg_mover 4000, general_mover 3072, and a render-instance pool of 2100
  the sweep budgets against. Pool objects (debris 0x30C4F70, rfg_mover 0x1F595A0, general_mover 0x31A3A50, render
  free count 0x1514A74) hold a descriptor pointer whose +0x1C is the free count. A full pool is why chunks freeze
  in the air (they can't become physics pieces).
- **Havok (2010) solver** at hkpWorld+0x1F0: steps at world+0x31C (RFG ships 4), micro steps +0x320, then the
  inverses 1/micro +0x324 and 1/steps +0x328. tau 0.6, damping 1.0, friction tau 0.3 at +0x1F4..+0x1FC. The mod
  hooks `hkpWorld::stepDeltaTime` to write them. Offsets were compiled from RFGR_Types with MSVC x86 (`offsetof`)
  and cross-checked against RSL's panel defaults. Gotcha 7 is why all four step fields must be written together.

### CONTROL Resonant data (Northlight)
- Controls: Resonant's real PC map is in its `inputactions_game.json` (two key slots per action: Ctrl also
  dodges, arrows move, numpad `/`, C the loadout, G the Gap).
- Audio: Wwise bank version 154, one bank per event (5,786), no media inside the banks; every source is a loose
  `audio\media\<id>.wem` (Vorbis, 48 kHz for gameplay).
- Dylan's clips carry Resonant's own timing markers (`TrailActive`, `DamageWindow`, `lua_dlg_effort_*`,
  `PushRelease`, `ShieldExplosion`, ...), so sounds and damage windows can start on Resonant's frames.
- Janitor's Cast export is Y up, metres, **right-handed**, the character facing -Z with his left at -X. Clips are
  absolute curves at 30 fps (77 are additive and were never baked). Loops repeat their first frame at the end.
- The Aberrant's forms are separate weapon meshes animated by weapon joints in Dylan's clips. The weapon root
  rides `R_wrist_prop_JNT`, with curves relative to `C_animRoot_JNT`; left-hand forms use `L_wrist_prop`.

## Build steps
1. **Native:** a PowerShell build script runs `vcvarsall x64_x86`, CMake (Ninja) with the game exe path for
   fingerprints, then builds at BelowNormal with 8 jobs. Outputs: `dinput8.dll` (proxy) and `resonant.dll`
   (+ pdb). Run the core tests (153) and the native tests (56 plus a 12-check smoke host that loads the proxy the
   way rfg.exe does and must install nothing against an unknown exe).
2. **Animations** (pack `dylan.rpk`, 217 clips, 19 MB):
   1. Export Dylan's clips with Janitor. Run its `/names bones` in the same session as the export, or clip bone
      names don't resolve. Keep the filter tight (a broad wildcard pulled in 4 GB of face-capture clips).
   2. Read `mason.rig_pc` out of `misc.vpp_pc` and map bones. 42 are mapped: pelvis←C_body, spine←C_spine2,
      spine1←C_spine3, spine2←C_spine5, neck←C_neck2, head←C_head, clavicle←clavicle, upperarmtwist1←upArm,
      foretwist←loArm, hand←wrist, fingerN←Fing1 and fingerN1←Fing2, thumb1/thumb11←thumbFing1/2, thigh←upLeg,
      calf←loLeg, foot←ankle, toe0←ball. The other 11 (root, spinebend, camera, eyes, hand props, mid twists)
      keep identity.
   3. Retarget with rest-pose alignment through a Z mirror (C = diag(1, 1, -1), quaternions x y z w). Keep the
      root-motion travel **unscaled** and scale only the hips' own crouch and lean by the leg ratio (0.945).
      Aim Mason's rigid hand bone at Dylan's actual knuckle positions each frame (Dylan's palm bones animate).
   4. Bake RPK1: per clip, local rotations for the 53 bones plus a root track at 30 fps, a loop flag (off for
      wind-up/hold clips, which hold their last frame). The runtime maps 103 gameplay roles (idle/walk/run/sprint
      bands by speed, jumps, Elevate, dodges, each form's hits/charges/holds, 6 finishers, every power's start /
      hold / release, Execute, Reach, landings by fall speed) to clip names.
   5. Check: every mapped limb points where Dylan's does on every frame (min cosine 0.981, at the pelvis, where 3
      spine bones stand in for 5), planted feet within 2.8 cm, unit and sign-continuous quaternions, stick-figure
      preview PNGs side by side, and a core test that plays the real pack through 1,770 scripted frames.
3. **Sounds** (pack `dylan_sfx.rsfx`, 257 sounds / 791 variants / 196 cues, about 62 MB): extract banks and media,
   resolve each event's sources through the Wwise bank hierarchy, decode with vgmstream, normalise loudness, and
   write the mod's own RSFX v1 pack (named sounds, their variants, per-clip cue lists). Played with XAudio2 2.9
   (loaded at run time), 3D panned, with a watchdog on the Present thread that pauses the engine when the game
   tick stops (pause menu, alt-tab).
4. **The Aberrant** (pack `dylan_aberrant.rabr`, RABR v1, 40 MB, baked in 19 s): 14 meshes (handle, flurry R/L,
   slash, slice, crush, extend, drill, bludgeon R/L, cleave, split, execution R/L) mirrored into RFG's space and
   remapped to at most 32 weighted joints; 22 textures at 512 px with mips (albedo `_d`, glow `_e`); 167 clips of
   weapon-joint tracks **relative to Mason's r-hand (bone 26) / l-hand (25)**, plus show windows from the clip
   markers (a form shows only while its clip draws it; the handle otherwise, except in neutral, climb, vault,
   mantle and death clips). Layout: 64-byte header; 32-byte texture, 96-byte mesh, 96-byte clip, 16-byte track and
   16-byte window records; 40-byte vertices (pos 3f, normal 3f, uv 2f, 4 u8 joints, 4 unorm8 weights); tracks are
   frames x joints x 10 floats (pos, quat xyzw, scale).
5. **Data patch:** a small `tweak_table` xtbl edit in `table.vpp_pc` (higher fall-damage thresholds so a ground
   slam can't kill the player). Steam's "Verify integrity" undoes it but not the DLL.
6. **Deploy** with the game closed (the script refuses while `rfg.exe` runs): proxy, DLL, packs, data patch;
   `-DryRun`, `-Uninstall` (byte-for-byte restore), `-ResetIni`. The human's `resonant.ini` holds their loadout,
   so settings changes go through versioned migrations (v1 → v4), each leaving `resonant.ini.v<N>.bak`.

## Verification
The human ran every in-game test (the agent never launched RFG or Resonant) and the agent read
`<game>\resonant\resonant.log` afterwards; the log prints the exe check, every address with its confidence and
result, each hook, and probes.
- **Test 1 (first build, powers on Mason's controls):** all 60 addresses and 8 hooks up, overlay up. Movement
  broke: NaN velocity killed Mason (Gotcha 1). The human then asked for a full control and movement takeover.
- **Test 2 (takeover):** no crash; run 6 m/s, dodges 14 m/s, double-jump kicks +4.9..7.5 m/s, Elevate capped the
  fall, RFG's keys fully blocked. The human: "the player still has a hammer and moves as if it's the vanilla game".
- **Test 3 (Dylan's animations):** the pose hook matched the live rig (53 bones, 0 name mismatches); a 7-minute
  combat session with no crash.
- **Test 4 (god mode, no despawn, solver at 8 steps, first person, sounds):** no crash; the ini migrated. The human
  saw NPC physics off for the whole world, Dylan's weapons invisible and no damage (Gotchas 6-8).
- **Fifth build: not verified in game.** Checked offline only: the embedded HLSL compiles with `D3DCompile`
  from the system `d3dcompiler_47` (vs_5_0 2,868 bytes, ps_5_0 2,080 bytes, no warnings), 0x550830's first bytes
  are a clean thiscall prologue, the C++ reader loads the real pack (crush shows at frame 20 of
  `dylan_crush_attack1_01`, gone at 100), core 153 / native 56 + 12 smoke tests pass, plus an independent review.
- **Not verified anywhere:** whether the render state still holds the 3D matrices when 0x550830 runs, where the
  world's depth buffer is at that point, the despawn guard's thresholds, the first-person eye offset, and whether
  the mod really stays idle in multiplayer.

## Gotchas
1. **Test 1 killed Mason within seconds.** **Cause:** the proxy velocity reads NaN in components the game isn't
   driving (x/z at idle, y in the air); the mod kept those components, wrote NaN back, the position went NaN and
   the game killed the player (state 4, knockdown mode 7). Also +0x2FC, used as "on ground", is the stance.
   **Fix:** never write a non-finite component (core test first); airborne = `cur_state` 1/2 or mode 4/5.
2. **Powers bolted onto Mason's controls weren't what the human wanted.** **Cause:** "like playing CONTROL
   Resonant, but in Red Faction Guerrilla" means the whole on-foot scheme. **Fix:** take over the action getters
   (answer false for RFG's on-foot ids, keep menus/map/pause) and drive movement from the kit's own controller.
3. **The hammer still swung under Dylan's attacks.** **Cause:** the kit injected RFG's attack action so an
   animation would play; once the pose override existed, that only added the hammer's damage and sound.
   **Fix:** stop feeding attack/melee; the kit deals damage through the damage sink and breaks walls with small
   structural blasts. The hammer stays equipped (alpha 0 via 0xA65F80) but unseen.
4. **Two runtime clip names didn't exist in Resonant.** `dylan_combat_dodge_air_fwrd_01` is
   `dylan_neutral_dodge_air_fwrd_rlf_01` (or `_llf_01`), and `..._flinch_medium_frnt_01` is
   `..._flinch_med_frnt_01`. **Fix:** resolve every role's clip against the baked pack in a test.
5. **The Inferno power's blast did nothing.** **Cause:** `Molotov Explosion` is not in RFG's in-memory explosion
   table. **Fix:** use `exp_fiery_small`, fall back to the table's first entry, and log every template name at
   startup.
6. **Dylan's hits did almost no damage.** **Cause:** the kit's damage numbers were on Resonant's scale; RFG NPCs
   have 300-600 HP. **Fix:** `world.damage_scale` 4 on the kit's hits, with hit logging (first 30, then every
   50th, HP before and after). It is in the fifth build, which never ran.
7. **After "physics max", every NPC moved wrong, not just the player.** **Cause:** the solver write is
   world-wide, and it changed the step count (4 → 8) without the stored inverses (+0x324/+0x328), so bodies ran at
   twice real time while NPC character controllers didn't. (Found later in rf-gmod's offline RE.) **Fix:** off by
   default (ini v3 migration); if raised, write steps, micro steps and both inverses together, inside the
   `stepDeltaTime` detour (the world may be stepping on another thread), after checking RFG's defaults are there.
8. **Dylan's weapons were invisible.** **Cause:** RFG has no mesh for the Aberrant. **Fix:** draw it yourself at
   0x550830: after the camera update, build skin = player world x FK(hand) x track x inverse bind per joint and
   keep it with the camera position in a small ring; on the render thread draw the entry whose camera matches the
   frame's (within 5 cm, newest wins), with the frame's own view/projection, every pipeline state saved and
   restored, SEH-guarded (a fault turns it off for the session). This is the fifth build, which never ran.
9. **RFG's camera mode 8 looks like a first-person mode but isn't playable.** **Cause:** it takes its view from
   the player's orientation and a camera-bone offset, not from the mouse. **Fix:** keep on-foot mode 10 and move
   real and render pos to the eyes (0xAB2260) after each camera update.
10. **Hiding the player with `human_hide` drops him through the ground** (RFG-VR found it). **Fix:** hide only
    the head by scaling the head bone (rig 14) by 0.001 in the pose override.
11. **Debris still vanished with the sweep's deleters disabled.** **Cause:** fades are scheduled when pieces are
    born. **Fix:** hook both fade starters and refuse only the despawn callers by return address
    (`_ReturnAddress()` inside a MinHook detour is the original caller): the caller ranges above. Let pulverise
    and Nano Rifle fades through.
12. **With deletion fully off, new debris stops spawning.** **Cause:** the fixed pools (object_debris holds 500).
    **Fix:** a guard that lets the game's own deleter run only when a pool's free count drops under a threshold
    (24 slots; render instances 48); the game then picks far, asleep, off-screen pieces itself.
13. **A key the mod owns stays swallowed after its tick stops.** **Fix:** tie key ownership to the takeover's own
    lapse timer inside the input layer, not to the tick.
14. **Retarget errors that looked like bugs:** scaling the whole root by the leg ratio left Mason 16-18 cm low on
    climbs (keep travel unscaled); `_90l/_90r/_180` clips turn the body by their end (don't also turn the
    character); the combat idle stands with the hips about 54° right (authored that way); vertical root motion
    (Elevate rises 2.5 m inside its clip, climbs 2.9 m) pulls the body away from the collision capsule unless the
    runtime applies only the pelvis's vertical bob on the ground.
15. **Windows shell traps on this machine.** Bash heredocs and `sed` fed Windows paths turned `\r`/`\f` into
    control characters (in a journal and an index file), and a PowerShell `"$AnimPack:"` parsed as a
    drive-scoped variable, so the deploy script failed and changed nothing. **Fix:** write files with an editor
    tool or a script file, scan journals for control characters, and use `${AnimPack}` in strings.

## Assets
No generated assets. Dylan's clips, sounds and Aberrant meshes come from the human's own CONTROL Resonant install,
are baked into packs that stay on that PC outside git, and are never published.

## Cost and time
Builds 1-4: about 15 hours of agent time (2026-10-19 and 2026-10-26); build 5: about 3.5 hours more. Sub-agents
built the animation export/retarget pipeline (about 34 minutes) and the sound pipeline.

## Open questions
- The fifth build's in-game result: the Aberrant's placement in the hand, occlusion (it likely shows through
  walls without the world's depth), brightness, the damage scale, the WeaponWheel-style layer.
- Dylan's mesh on `mason.rig_pc` (the same bone map would drive a mesh swap).
- Raising the debris pool past 500 means relocating a live pool (its descriptor is filled at run time): untested.
- Three mover-fade callers are not decoded (0xAA2770, 0x99CD30, 0x99C140) and are left alone.
- Whether streaming a zone out deletes its debris through another path.


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

