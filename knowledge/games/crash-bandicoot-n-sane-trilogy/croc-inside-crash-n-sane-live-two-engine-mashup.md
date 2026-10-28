---
kind: game
title: 'Croc inside Crash N. Sane: live two-engine mashup and native port'
game: Crash Bandicoot N. Sane Trilogy
games_also: ["Croc: Legend of the Gobbos (2025 remaster)"]
game_version: 'Steam 731490, exe PE timestamp 2018-07-19 (sha256 170c61d3...6af3); Croc remaster GOG 1152820153 build 58936832420733913'
platform: windows
engine: native
route: passthrough
tools: ["ReShade 6.8.0 add-on build", "Ultimate ASI Loader v9.7.4 (version.dll, Croc)", "MSVC 14.44", "capstone", "pefile"]
anti_cheat: none (Crash exe is SteamStub-wrapped; only the running process was hooked)
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-10'
links: []
tags: [alchemy-engine, mashup, passthrough, frame-compositing, reshade-addon, d3d11, d3d12, depth-compositing, render-hijack, input-forwarding, character-controller, hardware-breakpoint]
---
# Croc inside Crash N. Sane: live two-engine mashup and native port

> The real Croc (Croc remaster, D3D12) runs as a second process. His engine draws only Croc, through Crash's
> camera, and the picture is depth-composited into Crash's frame (D3D11) with Crash hidden.
> - **Stage A:** Crash's physics move the player and Croc mirrors his run, jump and spin.
> - **Stage B:** Croc's own physics (run, acceleration, tank turning, jump arc) drive Crash's character
>   controller. Crash keeps collision response, crates, enemies, lives and camera.
> - **Evidence:** verified in game with screenshots and recordings (run, jump onto the Aku Aku crate, spin
>   breaking crates, pit death and respawn, game over and retry, pause).

## Setup
- **Games:** both are the user's own copies. Crash Bandicoot N. Sane Trilogy from Steam (its exe is
  SteamStub-wrapped and is never modified on disk). Croc is the 2025 remaster, *Croc: Legend of the Gobbos*
  from GOG (DRM-free, released 2 April 2025; also on consoles). The 1997 PC original isn't sold any more and
  isn't needed: everything Croc-side comes from the remaster's own install (`remaster/` and `gdata/`).
- **Install:**
  - Crash: ReShade64.dll as `d3d11.dll` (Crash imports d3d11 directly).
  - Croc: ReShade as `ReShade64.asi`, loaded by the Ultimate ASI Loader installed as `version.dll`.
  - A `dxgi.dll` proxy is ignored by Croc: it loads dxgi/d3d12 with a System32-only search, but its static
    imports aren't restricted.
- **Add-ons load from a folder outside the game:** each `ReShade.ini` sets `[ADDON] AddonPath`, so rebuilds
  never touch game folders (a running game locks its add-on: LNK1104).
- **Croc settings:**
  - Windowed 1280×720 so a frame fits the shared slot.
  - MSAA Off (in-game Display Setup): a multisampled depth can't be copied.
- **Crash settings:** Keyboard Controls → Attack second key = F (see Gotchas). Borderless windowed is forced
  by the host add-on.

## Route and why
- **Taken:** passthrough with a render hijack, i.e. frame compositing plus state exchange in both directions.
- **Considered and deferred:** a native port (Croc model, animations and moveset rebuilt in Crash's .igz
  data). It's the better shippable product, and the live bridge is now its measuring tool. The user chose
  "both games running live".
- **Not used:** a whole-frame overlay of Croc's own level. It looks pasted on; skipping every non-Croc draw
  gives a clean character layer with true depth.
- **Related:** Stage A is the frame-compositing route (`techniques/frame-compositing-depth-and-pose-sync.md`;
  the other ReShade add-on compositor is `games/gta-v/minecraft-passthrough.md`). Stage B's split (Croc owns
  movement, Crash owns collision, lives and camera, and takes the player back on death) is an ownership choice
  of the kind `techniques/bridge-contracts-ownership-units-and-lifecycle.md` asks you to write down. The native
  port is Pattern 1 in `skills/mashup-mods/SKILL.md`: the guest's assets converted from the user's install.

## How the game works (what we had to learn)
**Crash (Alchemy, D3D11)**
- **Per-view constant buffer:**
  - World vertex shaders read `SceneGeometryConstants` (400 B, VS slot 0): `ig_matrix_model` @0,
    `ig_matrix_viewproj` @64 (row-vector), `ig_matrix_viewproj_previous` @128.
  - The game re-maps it per draw, so the camera VP is the perspective matrix written about 165 times per
    frame at +64 of 400-byte buffers.
  - The names come from the main scene VS's reflection data.
- **Units:** render units are inches; render = collision units × 39.3701 (Crash's blob shadow sits exactly
  there).
- **Depth:**
  - World geometry is drawn with viewport depth range [0, 0.8]; far layers use 0.8–1.
  - The scene depth is R24G8 at back-buffer size. Pick it by aspect ratio, not draw count: the 4096×16384 R16
    shadow atlas has more draws.
- **Player position:**
  - The collision-capsule array `[exe+0x1A5C160]` region and `[exe+0x1A5C158]+0x38` (0x90 stride) is
    rebuilt every frame from entity transforms. The player's slot moves and can go stale after respawn; the
    rendered body (4016-byte skinned buffer, World @64 ÷ 39.37) is reliable.
  - The authoritative, writable position is the character-controller body at `[exe+0x1A5C160]+0xE0 → +0x80`
    (float3, collision units). It was found by a displacement diff scan plus live write tests.
- **Pause:** detected as "bone data in the 4016-byte buffer unchanged for 6 frames".
- **Death state:**
  - The int32 at `exe+0x1a59210` reads 2 while Crash is alive and under control, and 0 from about 0.4 s into a
    death until the respawn teleport (also on the game-over screen). Jump, spin and pause don't change it.
  - It's static, so it survives level reloads.
  - Found by sampling the module's writable data through pit deaths and filtering against no-death runs.
  - The lives counter is a heap int that changes too late (about 1.5 s after the death starts) and moves on
    every retry.
  - Use it to hand back: show Crash, don't composite the guest, and stop writing the body.

**Croc remaster (Argonaut/Titanium engine, D3D12)**
- **Draws:** about 280 indexed draws per frame, each with one 512-byte root CBV in persistently mapped upload
  memory. @0 is world·view·proj (row-vector), @64 is world, @480 is the projection.
  - Croc's body parts are the draws whose world translation sits at the player.
  - Rewriting @0 at draw time (world × Croc→Crash mapping × Crash VP) and skipping other perspective draws
    renders only Croc, seen through Crash's camera.
- **Depth:** Croc's depth target is reached through D3D12 render passes (ReShade's `begin_render_pass` event,
  not `bind_render_targets_and_depth_stencil`). Format R32_TYPELESS (39 → decode as float).
- **Player struct:** `[Croc64.exe+0xB26800]`. Fixed point 1/4096: x @+0x7B0, y @+0x7B4 (Y down), z @+0x7B8,
  previous position @+0x7C8.
  - The float at exe+0xA79AF8 only mirrors the camera target: a setter at exe+0x1CC740 is called from
    camera code.
  - Found with an in-process hardware write-watch and disassembly of the exe on disk.

## Build steps
1. Install as in Setup. Build both add-ons with MSVC (`/std:c++17 /we4129`, ReShade 6.8.0 headers).
2. Start Croc into a level, then Crash into a level (scripted with WinDrive; both games' menus need scan
   codes, and Crash's menus need *held* keys). The quick boot in the
   [uncapped FPS note](uncapped-fps-and-live-boss-powers.md) skips Crash's intros without any input; pressing
   Enter through them once loaded a save (its gotcha 9).
3. **Stage A:** Croc's frame plus depth go through 3 seqlocked shared-memory slots at about 30 Hz (D3D12
   readback with a fence, never waiting). The Crash pixel shader shows Croc where his depth is less than 1 and
   0.8·z ≤ Crash depth. Crash's own model draws are skipped while a fresh Croc frame exists.
4. **Stage B:**
   - Crash's input is blocked (ReShade `block_input_next_frame`) and the keys are read with
     `runtime->is_key_down`, then posted to Croc's window.
   - Croc is pinned each tick and his own displacement is published. The host rotates and scales it (one
     fixed rotation phi, × 14.4) into Crash's controller body; airborne height × 14.4 goes into z.

## Verification
- **Oracles:** screenshots read back after every change (ReShade's own screenshots while exclusive
  fullscreen), recordings tiled into frame grids, per-second logs on both sides, and the depth error of the
  camera candidate against the real depth buffer (≈ 0.001).
- **Seen working:** occlusion behind crates, walking, jumps, a spin breaking crates, Aku Aku pickup, pit death
  and respawn, game over and retry, pause.
- **Not verified (live mashup; the native port's checks are at the end of its section):**
  - vehicle and riding levels and bosses;
  - other levels' camera behaviour, beyond Jungle Rollers and the Crash 2 warp room;
  - long play sessions;
  - Proton.

## Gotchas
1. **Croc's `dxgi.dll` proxy never loads.** **Cause:** Croc loads dxgi/d3d12 from System32 only.
   **Fix:** load ReShade through an ASI loader installed as `version.dll`, a static import.
2. **The first camera lock was wrong but tracked the player on screen.** **Cause:** an object/model matrix
   with a unit column. **Fix:** validate candidates against the real depth buffer, then use the structural
   source (most-written VP at +64 of 400-byte buffers).
3. **The depth test failed everywhere.** **Cause:** Crash draws the world with viewport depth [0, 0.8].
   **Fix:** stored depth = 0.8 · ndc_z, i.e. read the viewport most scene draws use.
4. **Crash exits on Alt+Enter after adding an overlay.** **Cause:** our back-buffer RTV and a context-state
   binding kept a reference, so ResizeBuffers failed with INVALID_CALL. **Fix:** release both every present.
5. **Crash froze on a black screen at startup with borderless.** **Cause:** SetWindowPos called from the
   present path, where the render thread waited on the window thread. **Fix:** do window changes on a worker
   thread.
6. **Croc pauses whenever Crash has focus.** **Fix:** subclass Croc's window and swallow
   WM_ACTIVATE(inactive), WM_ACTIVATEAPP(0), WM_KILLFOCUS and WM_NCACTIVATE(0).
7. **Croc's whole frame painted over Crash.** **Cause:** R32 depth decoded as D24. **Fix:** treat format 39 as
   float.
8. **Croc rendered dark at times.** **Cause:** his vertex lighting comes from where he stands in his own
   level. **Fix:** pin him on his lit spawn spot.
9. **Writing the obvious position copies does nothing.** **Cause:** capsule records and skeleton roots are
   rebuilt from the controller each frame. **Fix:** a diff scan by the exact displacement (m and in), then a
   live write test against the rendered position (`findmover`).
10. **Stage B keys read as 0.** **Cause:** ReShade's input block also hides GetAsyncKeyState inside the game
    process. **Fix:** `effect_runtime::is_key_down`.
11. **Menus stuck in Stage B.** **Cause:** the input block also blocked menus and game over. **Fix:** block
    only in a live level that isn't frozen; unblock-and-resend Esc.
12. **Attack never reached Crash while blocked.** **Cause:** buffered DirectInput drops presses made while
    blocked, and borderless Crash ignored synthetic clicks. **Fix:** bind F as Crash's second Attack key,
    unblock for 14 frames on the attack edge and SendInput F.
13. **Write AV crash during a level load.** **Cause:** Stage B applying idle drift through a stale pointer
    chain. **Fix:** write only with a live camera, a body near the player and real movement.
14. **Ini paths silently broken.** **Cause:** heredocs/scripted edits collapsed `\\` in `L"..."` paths into
    `\m`, `\c`. **Fix:** `/we4129` makes unknown escapes errors.
15. **Hardware write-watch crashed the game after disarm.** **Cause:** a late DR0 hit was passed on as
    unhandled. **Fix:** the VEH always absorbs its own hits and clears DR7.
16. **The screenshot oracle lied twice.** **Cause:** window capture freezes in exclusive fullscreen; a 1 s
    capture start misses a 0.4 s spin; an injected PrintScreen with Ctrl held is not ReShade's key.
    **Fix:** ReShade screenshots or short recordings tiled into frames.
17. **Croc slid and drifted in Stage B.** **Cause:** his level (2-4) is ice, so his real physics slide.
    **Fix:** run him in a grass level (1-1); also apply his motion only while a move key is held (+0.6 s coast)
    or while airborne.
18. **Remastered Croc rendered pale blue.** **Cause:** with the Croc → Crash matrix, clip w was in Crash inches,
    so his shaders' distance fog saw him as far away. **Fix:** scale the whole matrix by 1/k; x/w, y/w and z/w
    are unchanged.
19. **Croc stayed bright over Crash's black death fade.** **Fix:** average the finished back buffer (mipmapped
    copy, 1×1 mip) and scale Croc by luminance / 0.2.
20. **"Croc ignores arrows after respawn".** **Cause:** tank controls; Left/Right turn on the spot. Not a bug.
21. **Crash's death played with nobody on screen** (Crash hidden, Croc standing idle). **Fix:** hand back while
    `exe+0x1a59210 == 0` (see Death state). Latch on the alive → 0 edge; end on the flag going alive or on a
    horizontal respawn teleport, not on losing the player capsule (it vanishes during enemy deaths). Verified on
    pit, TNT and enemy (crab) deaths: Crash's own death animations play.
22. **Croc drawn over the HUD.** **Fix:** composite into Crash's intermediate final texture at the first draw
    that has no depth target bound (the HUD pass), not at present.
23. **The pause menu's level preview showed only Crash's shadow.** **Cause:** the preview is a still: a separate
    3840×2160 render target, captured when the pause begins and sampled by the pause frame's first no-depth
    quad into the back buffer. **Fix:** while paused, composite the guest once into that texture, reusing the
    last live frame's composite settings. Find it by logging the PS t0 of each back-buffer draw.
24. **Crash's silhouette shadow stayed under Croc.** **Cause:** the shadow passes (R16 depth targets) bind no
    object buffer near the player, so the main-pass "is this the player" test never matches. **Fix:** remember
    the geometry of the draws hidden in the main pass (index buffer, index count, first index, base vertex)
    and skip identical draws in the shadow passes. Give the guest a contact shadow of its own: a screen-space
    ellipse from the projected ground ring, depth-gated against the scene, kept on the ground during jumps.
25. **The island map ignored the arrow keys.** **Cause:** a "player + camera present" test also passes on the
    map, so the host kept blocking Crash's input. **Fix:** block and forward input only after a few consecutive
    frames in which the guest's motion actually drove a sane level body. Level changes then work with no
    restart (verified Jungle Rollers → map → N. Sanity Beach).
26. **After some respawns no capsule has the player's shape.** **Fix:** fall back to the rendered model when the
    character-controller body agrees with it.

## Native port: Croc as an extra character, no Croc process (2026-10-10)
After the live mashup, the user wanted Croc native: an extra character with his real model, animations,
feel and attacks.
- **No `.igz` authoring:** there's no public tool to author a new skinned `.igz` model, and character
  behaviour is code in the exe.
- **Instead**, the same ReShade add-on draws a converted Croc rig at Crash's player draw, and Crash's own
  controller moves him.

**Croc remaster formats** (from Croc64.exe's loader and its shipped shaders):
- **ZMOD** (`remaster/models/hd/*.zmod`): objects; each has sub-meshes {name = atlas id, index count, first
  index, AABB}, a vertex block, u32 indices, then a morph binding or 0 (static).
  - Morph objects: UNORM16 UVs, per-vertex (first, count) influence ranges, then per influence a cage index
    and three float weights (position, tangent U, tangent V), stored as separate arrays.
  - Static objects: 24-byte vertices {float3, packed normal, UNORM16 uv, rgba}.
- **Deformation** (decompiled from `gdata/shaders/DIRECT3D12/morph_gpu_hd_obj.vert`, a DXIL container at
  +0x34; `dxc -dumpbin`): `pos = Σ w·P`, `normal = normalize(cross(Σ w_u·P, Σ w_v·P))`, with P lerped between
  two cage frames.
- **ZMOR** (`remaster/anims/*.zmor`): frames × cage verts × float4. Croc's cage is the original 311-vertex
  model.
- **Rigid parts** (eyes, eyelids, arms, backpack): their transforms are in the original `gdata/anims.wad`
  `.ANI`.
  - WAD index lines are `NAME,off,size,unpacked,type`. Type `w` is a word RLE (signed control byte: ≥0
    repeat next u16 c+2 times, <0 copy −c words); `b` is the byte RLE.
  - The last frames × 240 + 4 bytes of an ANI are 10 × int16 {R 3×3 4.12, t/4096} per frame; part k drives
    object k + 1.
- **Textures:** sub-mesh `016` means `remaster/atlases/hd/atlas016.DDS` (BC7); sample at (u, 1 − v).

**Inside Crash:**
- **Deferred G-buffer:** the scene pass is 4 × RGBA8 + D24S8. Layout from Crash's own player pixel shader:
  o0 albedo | pack.r, o1 normal × 0.5 + 0.5 | 0.49, o2 (pack.b, pack.g, emissive, 0.49), o3 (0, 0.5, 0.5, 1).
- **Stencil:** write the G-buffer with Crash's own depth-stencil state and stencil ref, or the lighting pass
  won't shade the pixels.
- **Feel:** Croc's measured run (1.10 u/s), jump (0.319 u, 0.70 s) and acceleration match Crash's
  controller (11.6 u/s, 3.36 u, 0.66 s) within 1–6% at Croc height 1.95 Crash units. So Crash's controller
  moves him: real collision and camera-relative "modern" controls.
- **Proxy velocity is a dead end:** writing it fails; it comes out of Havok's simplex solver.
- **Player draw detection:** recognise the player's draw by geometry signature too, or it fails
  mid-jump/spin.
- **Deaths:** play `crocdie1` and draw Croc at any G-buffer draw while dying.
- **Choice:** Croc is a third hero on the game's own swap (see "Hero swap" below); the choice is saved in the
  add-on's ini.
- **Cache location:** the converter writes the rig into a local cache next to the mod, not
  `%LOCALAPPDATA%`; sandboxed writes there may be virtualised. The cache is converted Croc data: keep it out
  of any repository and don't share it.
- **Camera when the scene pass gives none** (Crash 1's Cortex boss, sparse map views): derive it from Crash's
  own skinned buffer. VP = World⁻¹ · WVP of the 4016-byte player buffer. In normal levels it equals the scene
  camera exactly, so use it only as a fallback. Also accept a lone camera candidate written ≥ 3 times/frame,
  and views up to 1000 units away (the airship node on the island map).
- **Mounts** (Hog Wild's hog): other models drawn centred under the player, about 100/s while riding and 0
  otherwise. While mounted, loop the guest's balancing clip instead of letting the bobbing trigger jump and
  land clips.
- **Croc's moveset only:**
  - Crash reads keyboard and mouse through user32, not DirectInput: its DirectInput object only serves pads,
    and no raw input is registered. So hook `GetKeyboardState`, `GetKeyState` and `XInputGetState` in the
    running process's import table (the exe on disk stays untouched), and subclass the game window.
  - **Hide Crash-only moves:** crouch/slide/body slam (right mouse; pad B and RT), Speed Shoes (Shift),
    bazooka (Q), the air spin, and the Crash 3 double jump.
  - **Croc's stomp:** a second jump in the air presses Crash's slide button for 150 ms. The engine's body
    slam then breaks what's below and shows the impact dust.
  - **Edges:** take key edges from `GetAsyncKeyState`. The game's own sources disagree for a few ms and
    create fake re-presses.
- **Vehicle levels** (Crash 3's biplane and motorbike):
  - **Detection:** no on-foot capsule or controller body exists, and the level-state flag reads 0 for the whole
    level. Key vehicle mode on the area archive's name (below): the pilot waits still behind the start line,
    so "the model keeps moving" never fires there, and his still bones read as a pause.
  - **What sits at the player:** three groups of draw: Crash's body (4016-byte buffer), his pilot outfit
    (other buffers, another vertex shader) and the vehicle (one vertex shader with the most draws there).
  - **What to hide:** everything at the player except the vehicle's shader. Croc then sits in the cockpit, and
    the pause preview shows it too.
  - **The vehicle's shader also draws the pilot's outfit** (the motorbike's ears, fur collar and gloves; the
    biplane's scarf, goggles and cap). Hide it by the index counts of `CrashBiker.igz` / `CrashAviator.igz`;
    Crash's on-foot body meshes use the same counts in every vehicle.
  - **Plane deaths:** the parachute pack mesh on the pilot's back is also the canopy once the parachute opens:
    hide it only while flying. The game keeps the body's World where the plane was hit and moves the body with
    its skinning matrices, so Croc isn't placed under the canopy yet; the parachute comes down empty.
- **Which area is loaded:** patch the running exe's `CreateFileW`/`CreateFileA` import entries (the file on
  disk stays untouched) and note the area archive it opens (`l100_hub.pak`, `l200_hub.pak`, `l300_hub.pak`,
  `l322_orangeasphalt.pak` ..., in lower case). The `*_hub` archives are the level-select screens. The archives
  also tell which levels are vehicle levels (`CrashBiker`/`CrashAviator` actors) and which are Coco-only (Crash 3
  tiger and jet-ski levels). The same hook drives
  [uncapped FPS and live boss powers](uncapped-fps-and-live-boss-powers.md).
- **Hero swap (Crash → Coco → Croc):**
  - The game swaps Crash and Coco with R2 (keyboard Ctrl) on Crash 1's island map and the Crash 2/3 warp rooms,
    once Coco is unlocked. The hero crouches and spins up in a swirl, and the other one drops in (about 1 s).
  - **Telling Crash from Coco:** a memory scan across swaps found only per-game heap flags, with no static path
    shared by the three games. The meshes drawn at the player work instead: their index counts are the same in
    all three games (Crash 22461, 9528, 5694, 4092; Coco 18288, 10614, 10404, 8256).
  - **Coco → Croc:** let the game swap back to Crash, hide the Crash who drops in (his meshes by index count,
    wherever drawn: he falls in from above, away from the player position) and draw Croc in his place.
  - **Croc → Crash:** Croc stays where he stood while the game swaps to Coco, who stays hidden. The add-on
    presses the swap again until the game takes it (about 1 s later, once Coco has landed), and Croc spins out
    upwards as Crash drops in. Press with the device the player used (Ctrl, or R2 in `XInputGetState`).
  - **No Coco yet** (no hero badge on screen): the press swaps Crash and Croc directly, with the spin effect.
  - Croc only stands in for Crash: while the game's hero is Coco, she is shown.
  - The island map draws the hero 1.4× larger (read the scale from the World matrix), and the level node under
    him uses a 1488-byte buffer: don't hide it.
- **Croc's face on the HUD:**
  - The hero icons are standalone BC3 textures:
    - the swap badge: `ui/Crash/HUD/Crash_Head1`, `Coco_head1` and their `_Glow` (stored upside down);
    - the lives counter: `ui/Crash/crashtoken` plus its line layer `crashtoken2`, used by
      `vfx/UI/Crash_Life_GUI`;
    - Crash 3's level card: a cell of `Boss_Portraits/character_sheet_filmstrip`, drawn in the scene pass;
    - the biplane health bar: `PilotHUD/Pilot-Crash-Portrait-2`;
    - Crash 3's pause screen: a Crash & Coco sticker painted on the left panel (`Pause/Crash3/BG_Left`). Its
      stand-in is that panel with the sticker painted out and a Croc sticker on it.
  - A script hashes their 16×16 centre blocks from the user's archives. It builds Croc's stand-ins from his own
    remaster lives icon (`remaster/ui/panel/croclife.tga`), outlined in black and framed like each original,
    into the local cache.
  - At runtime each texture bound to pixel-shader slot 0 is recognised once (by size, then a GPU read-back of
    the same blocks), and the stand-in is bound in its place for that draw. The badge's small face is the next
    hero: when Croc plays, Coco's head shows Crash.
- **Crash's props and effects around Croc:**
  - Crash's idle "timeout" animations (yo-yo, fruit juggling, a spin), his spin (hands swirl, Death Tornado ring)
    and the super body slam bring their own models and particles. They aren't part of the player draw, so hide
    them separately.
  - **Models by index count:** an IGZ model's `igIndexBuffer` objects hold the index count at +0x10, so the
    counts of every Crash prop can be read from the archives. Hide each group only in its moment and near the
    player, so scenery sharing a count stays.
  - **Particles by texture:** their vertex counts change every frame, and they're drawn in later passes. Follow
    Crash's effect files (`vfx/Characters/Playable/Crash/Crash_Attack_*`, `Crash_Timeout_*`) to their textures,
    hash those, and hide draws that sample them right after an attack or stomp, or after a long idle.
  - One idle draws Crash's whole body again in a later pass (a fruit-goo overlay): hide his body meshes near the
    player in every pass, not only the G-buffer pass.
  - **Finding them:** learn every draw signature (count, vertex shader, texture) in normal play and log the ones
    that appear only after an attack, a stomp or a long idle. Read texture info at draw time: a view saved for
    later may already be freed (that crashed the game once).
  - **Mount detection** must count only mount meshes (hog, jet-board, polar bear): crates and platforms under
    the player otherwise switch Croc into his riding pose.
  - **Death Tornado:** hold back a second attack press within 0.9 s; Crash chains spins into the tornado.
- **Test tooling:** `um win drive "idle N"` doesn't wait; it reports seconds since the last input. For timed
  combos, put the whole combo in one drive call: separate calls are too slow, because each one starts
  PowerShell. Pauses inside a call need the `sleep <ms>` verb from #206; without it, use `hold <vk> <ms>` or
  drive `um.win.Drive` from Python with `time.sleep` between commands.
- **Verified with the user's completed saves:**
  - Crash 1: the island map, the Cortex boss, Upstream (water), Hog Wild and The Great Hall.
  - Crash 2: the warp room and Hang Eight on foot.
  - Crash 3: the warp room, Flaming Passion, Orange Asphalt (motorbike) and Mad Bombers (biplane).
  - The swap: the full cycle in Crash 1's map and Crash 3's warp room; Crash ↔ Croc in Crash 2's warp room
    (Coco not unlocked in that save).

## Assets
None generated. Croc is drawn by his own engine from the user's install; nothing from either game is
redistributed.

## Cost and time
About a day and a half of agent time over four sessions, with the human mostly away.

## Open questions
- **Native port follow-ups:** Crash 2's jet-board and polar rides, Coco-only levels, bonus rounds, plane-level
  deaths, gamepad verification with a real controller (the R2 swap included), user rebinding support.
- **Remaining issues (live mashup):**
  - pose lag of 1–3 frames during fast camera moves (reproject).


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

