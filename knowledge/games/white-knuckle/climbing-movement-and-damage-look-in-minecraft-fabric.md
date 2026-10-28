---
kind: game
title: "White Knuckle's climbing, movement and damage look as a Minecraft Fabric mod (1.21.9, ported to 26.3), from your own install"
game: "White Knuckle"
games_also: ["Minecraft Java Edition"]
game_version: "White Knuckle Steam build 25612096 (Unity 6000.3.9f1, Mono, D3D11/D3D12), run on Linux through Proton Experimental; Minecraft Java 1.21.9 with Fabric Loader 0.19.5, Fabric API 0.134.1+1.21.9; ported to Minecraft Java 26.3 with Fabric Loader 0.19.5, Fabric API 0.162.0+26.3 (OpenGL and Vulkan)"
platform: proton
engine: unity-mono
route: reimplementation
tools: ["Fabric on 1.21.9 (Loom 1.12.7, Gradle 9.1, Mojang mappings, JDK 25)", "Fabric on 26.3 (non-remapping Loom 1.18-SNAPSHOT, Gradle 9.7.1, JDK 25)", "xdg-desktop-portal FileChooser through an own pure-Java D-Bus client", "LWJGL tinyfd (Windows folder dialog, 1.21.9)", "SDL3 SDL_ShowOpenFolderDialog (Windows folder dialog, 26.3)", "ilspycmd 8.2.0.7535 on .NET 8", "UnityPy + TypeTreeGeneratorAPI", "vkd3d-compiler.exe (Fedora mingw64-vkd3d) under Wine", "Python 3.10-3.14"]
anti_cheat: "none: White Knuckle is single-player and is never modified; only its files are read. Minecraft side tested in single-player, LAN and a Fabric dedicated server."
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-16'
links: ["https://github.com/Fizz12215/White-Knuckle-Climbing"]
tags: [white-knuckle, unity, unity-6, minecraft, fabric, climbing, physics-port, shader-disassembly, dxbc, fsb5, ima-adpcm, fmod, proton, kde-wayland, reference-capture, rewired, ledge-detection, camera, flatpak, xdg-desktop-portal, d-bus, windows-11, pip-pinning, '26.3', vulkan, sdl3, porting]
---

# White Knuckle's climbing, movement and damage look as a Minecraft Fabric mod (1.21.9, ported to 26.3), from your own install

> A Fabric mod that plays like White Knuckle's climber:
> - two-handed grip climbing on ledges, bars and ice, with per-hand grip;
> - the game's on-foot physics, camera, hands, sounds and health rules;
> - its damage look and hand-fatigue shader.
>
> The gameplay is reimplemented in Java from the decompiled C# (kept in a private folder, never
> shipped) and the serialized prefab values. The art and audio never ship: each player runs a Python
> tool that builds a private resource pack from their own install. The real game was the oracle
> throughout. Shaders were disassembled, and reference screenshots were captured by driving the
> game's dev console. The Minecraft side is checked by an automated client game test, including a
> dedicated server.
>
> Since 0.4.0 the mod builds that pack itself, from a setup screen in Minecraft that explains each
> step (folders, size, source, how to undo) and waits for the player's OK. It was verified on Linux,
> in a Flatpak launcher, and on Windows 11 by a second Claude Code session on the owner's dual boot.
>
> 0.5.0 (not released yet) ports it to Minecraft 26.3, unchanged in play. The same game test passes
> under both of 26.3's graphics engines, OpenGL and Vulkan. What moved in 26.3, and the traps that only
> showed at runtime, are in
> [Fabric mods for 26.x](../../techniques/fabric-mod-for-26-x-without-mappings.md)
> (*Porting a 1.21.x mod to 26.3*, Gotchas 8-12).

## Setup
- White Knuckle from Steam (app 3195790), on Fedora 44 (KDE Plasma, Wayland) through Proton
  Experimental.
- Minecraft side: the versions above, a Prism Launcher instance per Minecraft version. 1.21.9: JDK 25
  compiling with `--release 21`. 26.3: Java 25.
- Decompile: ilspycmd **8.2.0.7535** on .NET 8 (`DOTNET_ROLL_FORWARD=Major`). Newer ilspycmd builds
  want a newer runtime.
- Asset tool: UnityPy and TypeTreeGeneratorAPI (MonoBehaviour type trees are stripped, so they're
  regenerated from `Managed/*.dll`), Pillow, soundfile. On Windows, use only Python 3.10-3.14 with
  `--only-binary=:all:`; 3.15 had no wheels yet.

## Route and why
- **Reimplement, don't embed.** White Knuckle's movement is a small set of C# methods on the
  player (Movement, Hang, Jump, HandStamina, InteractCheck, FallingDamage) driven by values stored
  in the scene. Porting them to plain Java, at the game's own 50 Hz fixed step and in game units,
  gave a unit-testable core.
- **Minecraft's role:** the adapter runs 2-3 sim steps per 20 Hz tick and uses Minecraft only for
  collision (`Entity.move`), ray casts and input.
- **Assets:** built per player from their install (the "own copy" route): the mod jar has no game
  files. The pack is the game's own art and audio, so it stays with that player and isn't shared.
- **Related ports into Minecraft:**
  - [FPV LOGIC's drones](../fpv-logic/fpv-logic-drones-flight-model-rc-radio-and-3d-models-in-minecraft.md)
    rebuild another game's physics as a Fabric sim;
  - [Halo 3 from your own MCC install](../halo-3-mcc/halo-3-weapons-covenant-vehicles-and-maps-ported-into-minecr.md)
    builds its resource pack from the user's install the same way.

## How the game works (what we had to learn)
- **Data beats code.**
  - Many serialized prefab values differ from the C# field defaults: gravity -0.2 (not -9.81),
    drag 8 (not 0.1), interact distance 2.5 units.
  - The player is `ENT_Player` in `level3`. Handhold presets and icy handholds are in
    `sharedassets1.assets`.
  - Read values from the data (UnityPy + generated type trees), not just the code.
- **Units:**
  - The capsule is 2.25 units tall, so 1 unit = 0.8 Minecraft blocks.
  - `vel` is not m/s: displacement per step is `vel * dt * 175`.
  - Health is 5, so it was scaled x4 for Minecraft.
- **Grip:**
  - each hand drains on its own, at half rate while both hands hang (one flag set each frame in
    `Actions()`);
  - icy handholds slide down at 0.3 units/s, shake after 1 s and slip at 2 s.
- **Climbing details that are easy to port wrong** (found in a line-by-line review, 0.3.1):
  - `Hang()` picks how W/S pushes you from a wall probe of `cCon.radius * 2.5`. That reads the
    controller's own unscaled radius (0.3), so the probe is 0.75 units long; using the 1.5-scaled
    capsule radius gives 1.125.
  - Wall friction while hanging tests the sprint *state* (sprint key, input above 0.8, not
    crouched), not the raw key. Shift alone doesn't count.
  - Walking off an edge zeroes the downward speed (grounded at the last move, not now, falling), so
    the fall starts from rest. The post-move speed clamp uses the grounded flag from before the move.
  - The hang pull is measured from `transform.position`, the standing capsule's centre. Crouching
    shrinks the capsule but doesn't move it.
- **Input (Rewired):** the move vector is two independent axes and is *not* normalised, so W+D is
  (1, 1). `_digitalAxisSimulation` is 0, so keys are instant with no ramp. A host that normalises
  diagonals (Minecraft gives 0.71, 0.71) feeds about 30% less input into the climb push and
  one-hand climbing.
- **Camera and body:**
  - "Main Cam Target" sits 0.45 (local) above the capsule centre at the 1.5 player scale. The eye is
    1.8 units up, which is 1.44 blocks (Minecraft 1.62). Crouching (`ChangeHeight`, 0.25 s) lowers it
    by crouchHeight - standingHeight (1.0 - 1.5), to 1.05 units, which is 0.84 blocks (Minecraft
    1.27).
  - You can look 15 degrees past straight up (`camMinY` -105).
  - The CharacterController steps only 0.15 x 1.5 = 0.225 units (0.18 blocks; Minecraft 0.6).
- **Screen blood:**
  - FX_Player_BloodSplatter_Screen is a child of "Overlay Camera". That camera renders only layer 29,
    at depth 5 (last), with a fixed FOV of 90: `SetCameraFov` changes only the main and hand cameras.
    So the blood never follows the sprint FOV (+15) or the FOV setting, and it isn't touched by the
    post-processing.
  - The UI canvas holding the hurt vignette is Screen Space - Overlay, so the vignette covers the
    blood.
- **Looping sounds** (AudioSources on the player, volume set every frame):
  - windAudio "PlayerFalling" (pitch 1.2): volume max((-vy - 2) * 0.03, 0) for the fall speed in
    units/s. It restarts on each new fall.
  - damageAudio (heartbeat) and dyingAudio (mortal wound): volume `damageFXCurve` x 0.8 and
    `dyingAudioCurve` x 0.8 of health. They're two layers of one piece, playing from level load in
    step.
  - Past the 50-unit safe fall distance, the camera shakes by 0.1 per second for as long as the fall
    lasts.
  - Grabs play "handhold-<material>" when that set exists (chain, metafungus), else "handhold".
- **The damage look** is not URP volumes. The player queues an `FXData` zone weighted by
  `damageFXCurve(health/max)`, and `FXManager` lerps its fields into shader globals:
  - `damageFX.hsv.y` = 0.1;
  - `gamma` = 2;
  - a red `worldTint`, applied in the world lighting.
- **The post shader** ("Hidden/Dark Machine/Post Process WK"):
  - converts to HSV, scales saturation (keeping value, so pure green goes *white*), then multiplies
    by `0.25 + 0.75 * _GAMMA`;
  - `_GAMMA` = the player's brightness setting + damageFX.gamma, so getting hurt brightens the
    picture.
  - A UI "hurt" vignette sits on top. Its colour comes from `Color.Lerp(clear, colour, amount)`, so
    colour *and* alpha scale with amount (it grows like amount squared). It is blended in linear
    light.
  - Another post class, "Gamma Post Process", is referenced in code but its shader isn't shipped.
- **Hand fatigue red** (`SHDR_Base_Inventory`, `Item_Hand_Material`):
  - `_StaminaAmount` = 1 - grip/max.
  - Stepped mode is on (`_StaminaMaskRepeat` 8, `_StaminaLevels` 8, repeat intensity 0.5). With no
    mask texture assigned, Unity's default *black* mask applies, so the red strength works out to
    round(8a)/8 + ceil(8a)/16: fully red (5/8 + 6/16 = 1) once 8a passes 5, i.e. just over 62% used.
  - Pixels are multiplied toward red, in linear light.
  - The default hand *cosmetic* sets an outline mask colour (0.49 grey), which turns on a branch
    where outline pixels fade grey to red linearly.
  - Because even the default hands are a cosmetic, each hand gets its own material copy, so each
    hand shows its own grip. A shared material would show the same amount on both.
- **Audio:**
  - The clips are FSB5 in `.resS`/`.resource`, mostly IMA ADPCM (codec 7) plus some Vorbis.
  - IMA frames are 0x24 bytes per channel: a header predictor that is itself the first sample,
    then 63 samples from the first 63 of the 64 nibbles (4-byte interleave, low nibble first).
  - Decoding that way matches FMOD bit for bit.
- **The dev console** opens with Shift + backtick.
  - `cheats` enables the other commands.
  - `hurtplayer dontkill <n>` deals exact damage without killing (clamps to 0.1 health).
  - `setstamina <0-10>` sets both hands' grip.
  - The playground has pure white, green and black green-screen rooms, which make ideal
    colour-measurement backdrops.

## Build steps
1. Decompile `Assembly-CSharp.dll` (ilspycmd, above) to a private folder. Dump the serialized
   values you port with UnityPy and generated type trees.
2. Port the movement methods into a dependency-free sim class. Unit-test it against hand-worked
   numbers (grip timings, jump strain, ice).
3. Wire it into Fabric:
   - a `LivingEntity.travel` mixin for the local player; collision through `Entity.move`;
   - C2S state (hanging, wkMode) so the server stops fall distance and the fly kick;
   - health rules run server-side.
4. Damage look:
   - one post pass with a writable uniform buffer, rewritten every frame;
   - the screen blood goes through its own GUI pipeline, which applies only the hurt vignette (the
     game draws it after post-processing, under the UI);
   - the hands go through a custom GUI pipeline whose fragment shader applies the stamina red and
     the damage look per pixel (parameters ride in the vertex colour);
   - shared GLSL include via `#moj_import` (26.3: `#include`, under ShaderC's rules; see
     [Fabric mods for 26.x](../../techniques/fabric-mod-for-26-x-without-mappings.md)).
5. Asset tool (Python):
   - finds the game through Steam's `libraryfolders.vdf` (and the Windows registry `SteamPath`);
   - extracts the hand sprites, crosshair, blood sheet, sound sets and the player's looping
     AudioSources into a pack folder and zip;
   - one FMOD batch subprocess per game file for audio.
6. Ship a zip with the jar, the tool and per-OS scripts (`.bat` with CRLF; `.sh`), plus a
   step-by-step guide.
7. In-game build (0.4.0): the jar carries the extractor and a pinned `requirements.txt`. A screen at
   the title finds the game and a Python 3.10-3.14, then runs each step after its own confirmation
   page:
   - make a private venv (the same folder the scripts use, so either's setup is reused);
   - pip-install the pinned wheels (`--only-binary=:all:`);
   - run the extractor straight into `resourcepacks/`;
   - turn the pack on and reload.

   A `wk_pack_info.json` version inside the pack lets the mod offer a rebuild when it needs newer
   files. "Choose its folder" uses the desktop portal on Linux (Gotchas 22-23). On Windows it used LWJGL's
   tinyfd on 1.21.9, and SDL3's folder dialog on 26.3, which ships no tinyfd.
8. Port to 26.3 (0.5.0), with the 0.4.0 release tag as the source of truth and every 26.3 name checked
   with `javap`:
   - the build switch, then one commit per area until the client and the gametest compile;
   - every mixin target compared with the 26.3 class before the first launch (3 of 21 had changed);
   - the damage look joins 26.3's list of requested post effects every frame, keeping its writable
     uniform buffer;
   - pack format 97 and a new pack content version, so 0.4.0 packs get the rebuild offer.

## Verification
- **Oracle 1, the real game:**
  - Damage look: reference screenshots in the playground's white, green and black rooms at health
    5/4/3/2/1/0.1, taken with `hurtplayer dontkill 1`. A per-pixel model (HSV saturation keeping
    value, the brightness multiply, the vignette in linear light) fits all 15 frames at rms
    2.8/255.
  - The fitted vignette amounts matched 1 - health/max, and the fit recovered the player's
    brightness setting (1.97, actual 2.0) before it was looked up.
  - The disassembled shader then confirmed every term.
- **Oracle 2, FMOD:** every clip decoded by FMOD in a fresh process, compared with the IMA decoder:
  bit-identical.
- **Oracle 3, Minecraft client game test** (Fabric client gametest, about 3.5 min):
  - grabs, ledges, fatigue, ice, bars, health rules;
  - white/lime/black concrete walls at matching health levels, each frame within 0.25/255 of the
    model applied to the full-health frame;
  - left-hand stamina at 8 levels, all sprite values within 2/255 of the formula, with the right
    hand unchanged;
  - an in-process dedicated server (join, climb, hang 5 s, no kick);
  - run with and without the asset pack;
  - since 0.3.1:
    - eye height 1.44 / 0.84, and 1.62 with climbing off;
    - crouch-walking off a ledge drops you (the server doesn't pull you back);
    - the near-death loops at 2% health: 0.422 / 0.794 (the game's curves give 0.423 / 0.797);
    - the falling wind peaks at 0.77 on a 60-block fall;
    - ledges under one snow layer grab, while inner corners and four-block floor corners don't.
  - Each of the player-reported bugs (snow, inner corners, floor seams) was reproduced in a test
    scene before the fix and passes after it.
- **The in-game build (0.4.0):**
  - A gametest with an empty `XDG_DATA_HOME` clicks Continue through all four steps on the real
    install. It checks the pack is on and the hands load, that Stop changes nothing, and that an
    out-of-date pack brings back the offer with two steps.
  - Every page is screenshotted; long Windows paths wrap clear of the buttons.
- **Windows 11:**
  - A second Claude Code session on the owner's Windows dual boot ran the same tests and the real
    game (clicking automated with universal-modder's Windows tools), then pushed results to a private
    repo.
  - Checked: Python 3.13 installed by winget, tinyfd's dialog in front of Minecraft, fullscreen
    switching, the `.bat`.
  - It found the stale-cache bug (Gotcha 28) and sent the fix.
- **Flatpak:** the owner picked White Knuckle through the portal in the real Prism Flatpak, and the
  build ran from the document-portal path.
- **The 26.3 port (0.5.0), on Linux with an NVIDIA GPU:**
  - the full game test on OpenGL without the pack, and on OpenGL and Vulkan with a pack rebuilt for 26.3;
  - the damage look within 0.24-0.26/255 of the model at every health level, as on 1.21.9;
  - the in-game build on both engines (libraries, build, switch on, the out-of-date offer, Stop);
  - screenshots of the hands, blood, damage look and every setup page on both engines. They caught a
    screen with no text that every check had passed (Gotcha 30).
- **Also:** a standalone `runServer` boots clean.
- **Not verified:**
  - the "server without the mod" auto-off path against a real vanilla server;
  - on Windows, the `.bat`'s own venv setup and its install-Python branch (it ran with a venv already
    there);
  - `flatpak document-unexport` as the undo for the portal grant;
  - the 26.3 port on Windows (SDL3's folder dialog above all) and in a long manual play;
  - the red `worldTint` on lit surfaces, left out because the green-screen rooms are unlit.

## Gotchas
1. **Extraction took 285 s.** **Cause:** UnityPy's FMOD fails in the main process (first on IMA
   ADPCM clips, then on everything), and the workaround spawned one subprocess per clip, each
   re-reading a ~1 GB asset file. **Fix:** one `--decode-clips` subprocess per game file decodes
   every clip in a fresh process: 13 s total.
2. **Grab/let-go sounds had faint block-edge glitches.** **Cause:** an IMA decoder that skipped
   the header predictor and used all 64 nibbles was off by one sample per block (correlation
   0.997, so it sounded "almost right"). **Fix:** emit the predictor as sample 0, then decode only
   63 nibbles; verify against FMOD.
3. **The extractor died silently on Windows, leaving only `*.partial`.** **Cause:** libsndfile's
   Vorbis encoder puts buffers on the stack in proportion to frames per write, overflowing
   Windows' 1 MB main-thread stack. **Fix:** write 4096-frame chunks on a thread with a 64 MB
   stack. Reproduce on Linux with `ulimit -s 1024`.
4. **`.bat` crash checks missed failures.** **Cause:** a native crash gives a negative exit code,
   which `if errorlevel 1` doesn't catch. **Fix:** `if not "%errorlevel%"=="0"`, and check that the
   output file exists. `.bat` files also need CRLF line endings.
5. **The damage effect "didn't look like the game".** **Cause:** desaturating toward luma turns
   bright green grey, but the game keeps HSV value (green goes white) *and* brightens the picture.
   Its vignette also scales colour with amount, and Unity blends in linear light. **Fix:** do the
   grade and the vignette in one post pass in linear space. Minecraft GUI blending is sRGB, so a
   GUI-drawn vignette can't match (white would come out far too red).
6. **Shader uniform names looked stripped.** **Cause:** the DXBC only has `cb0[n]`. **Fix:**
   UnityPy's `m_ParsedForm.m_SubShaders[i].m_Passes[j].m_NameIndices` names them, and
   `progFragment.m_CommonParameters` gives each name's constant-buffer offset and texture slot.
   Decompress `compressedBlob` (LZ4) by `offsets`/`compressedLengths`, cut out the `DXBC`
   containers (size at offset 24), and disassemble with
   `vkd3d-compiler.exe -x dxbc-tpf -b d3d-asm` under Wine.
7. **TypeTreeGenerator reads fail with EOF on some classes.** **Cause:** a stray `references` node
   appears before subclass fields for classes with `[SerializeReference]` lists, and the hand
   cosmetic class doesn't read at all. **Fix:** drop the first `references` node. For the cosmetic,
   read the raw bytes by field layout: bools are 4-byte aligned, and colours are four floats.
8. **Driving the game on KDE Wayland (XWayland window via python-xlib XTEST) typed into the game
   instead of its console.** **Cause:** every *new* XTEST client triggers KDE's "Remote Control: X
   is asking to control input devices" popup, which steals focus. If the user clicks "Allow"
   instead of "Always allow", the next script asks again. The commands then landed as WASD and
   Space: the player jumped around, and that run's data was useless. **Fix:** send a harmless key
   first, screenshot, and check for the popup before typing anything. Ask the human to tick
   "Always allow".
9. **The dev console closes on Enter in some cases.** **Fix:** assume nothing; screenshot after
   each command, or reopen the console before each one.
10. **Reference frames come out brighter than "stock".** **Cause:** the player's brightness
    setting feeds the post shader's multiply (`settings.json` in the Proton prefix under
    `AppData/LocalLow/Dark Machine/White Knuckle`). **Fix:** read it and model it, or capture at
    the default (1.0).
11. **Minecraft 1.21.9 post-effect uniforms are fixed per JSON.** **Fix:** a mixin adds
    `COPY_DST` to the mod's own pass uniform buffers and rewrites them every frame. Keep uniform
    blocks to plain floats, because `Std140Builder` pads a `vec3` differently from GLSL's packing.
12. **A climbing client on a vanilla server gets kicked for flying.** **Fix:** on JOIN, if not the
    local server and `ClientPlayNetworking.canSend(<mod payload>)` is false, switch climbing off
    with a message.
13. **Client gametest: keybinds do nothing on the title screen.** Toggle state directly between
    worlds. Also, the framework resets options at test start, so re-apply the custom keybinds in
    the test.
14. **Unity 6 AudioSources came out silent.** **Cause:** the clip is in `m_Resource` (an
    AudioResource). The old `m_audioClip` field is still serialized, but as an empty reference
    (m_PathID 0), so "use `m_audioClip`, else `m_Resource`" picks the empty one. **Fix:** take
    whichever reference is non-zero.
15. **Snow-covered ledges couldn't be grabbed** (player report). **Cause:** the ledge probes tested
    blocks' *visual* shapes, and one snow layer has a 2-pixel visual shape but no collision box. It
    "covered" every ledge, and wasn't a handhold itself, so snowy biomes had almost nothing to grab.
    **Fix:** blocks without a collision box don't cover a ledge. The same applies to leaf litter,
    petals, rails and plants.
16. **Concave inner corners could be grabbed** (player report). **Cause:** a ledge was "open above,
    drop below". That passed a block's back edge under a block one up and one back, and a ledge's
    end pressed into a taller neighbour. **Fix:** also require open space outward *above* the edge
    (where the hand comes over), and room for a hand (0.1 blocks) either way along it.
17. **A flat floor could be grabbed where four blocks meet** (player report). **Cause:** the
    closest point on an edge is often its exact end, which lies on block boundaries. There, a
    strict "inside the shape" test calls the diagonal neighbour's face "outside". **Fix:** judge, and
    grab, a point 0.001 blocks inside the edge.
18. **Crouching held the player at ledges.** **Cause:** Minecraft's sneak guard
    (`Player.maybeBackOffFromEdge`) runs for `MoverType.SELF`. It also runs on the server's replay
    of client moves, so a client-only fix gets the player pulled back to the edge. **Fix:** a common
    mixin keyed on the mod's per-player mode, on both sides.
19. **The eye height change didn't take at first.** In 1.21.9 player sizes come from
    `Avatar.getDefaultDimensions(Pose)`. The cached `eyeHeight` is only refreshed on a pose change,
    so a player who joins or respawns standing keeps 1.62. **Fix:** refresh when it's stale.
    Fixed-angle test aims also had to look about 3 degrees higher from the lower eye.
20. **Screen blood slid and shrank while sprinting** (player report). **Cause:** the port projected
    the splats with the current FOV (they shrank about 23% at +15 degrees), but the game's Overlay
    Camera stays at 90. **Fix:** project at a fixed 90.
21. **A test passed for the wrong reason.** The game's 0.2 s grab memory let a hold from the view
    *before* aiming at a corner satisfy the check. **Fix:** wait out the memory after aiming.
    Separately, the client gametest's NetworkSynchronizer once reported "interfacing with packets at
    a lower level" and crashed the server. That's harness accounting; a rerun passed.

22. **A Flatpak launcher can't see the Steam folder.** Prism from Flathub (and most Steam Deck
    setups) runs Minecraft in its sandbox, where `~/.local/share/Steam` doesn't exist.
    - `flatpak override --filesystem=~/.local/share/Steam:ro` works, but it also exposes Steam's
      `config/` (its login and settings files; `config.vdf` was readable from inside) to every mod in
      the launcher.
    - **Fix:** the FileChooser portal with `directory: true`. The folder picked comes back as a
      document-portal path (`/run/user/<uid>/doc/<id>/White Knuckle`) the sandbox can read. It's
      remembered across restarts, and `flatpak document-unexport --doc-id=<id>` takes it away.
    - A fallback command should grant only the game folder.
23. **Nothing in Prism's sandbox could wait for the portal's answer.**
    - The sandbox's zenity is 3.44 (GTK3), whose dialog shows only the sandbox.
    - Its Python has no `gi`.
    - `gdbus call` gets the request handle and exits, which closes the request.

    **Fix:** about 300 lines of Java D-Bus client over the session socket: `UnixDomainSocketAddress`,
    SASL EXTERNAL with the uid from `/proc/self`, `Hello`, `AddMatch` on the request path,
    `OpenFile`, then wait for `Response`. It works through the sandbox's bus proxy
    (`unix:path=/run/flatpak/bus`).
24. **Switching away from fullscreen Minecraft froze it** (NVIDIA 615 driver, KDE Wayland, Flatpak
    Prism). About 8 s after the player opened a terminal, an Xid 31 MMU fault hit Minecraft's render
    thread, which then waited in `glClientWaitSync` forever. Minimizing and restoring in a gametest
    didn't reproduce it.
    - **Fix in the mod:** leave fullscreen before opening any other window (picker, installer), and
      go back after.
    - Never ask players to run commands while the game is open.
25. **Windows: never call a bare `python`.** It can be the Microsoft Store alias, or Store Python,
    whose AppData writes are redirected (a venv in `%LOCALAPPDATA%` lands elsewhere).
    - **Fix:** use `py -3.X` plus the known install paths (`%LOCALAPPDATA%\Programs\Python\...`,
      `%ProgramFiles%\Python3XX`), which also find a fresh winget install without a restart.
    - On Windows 11, `winget install Python.Python.3.13` installed per user with no UAC prompt.
26. **Steam's registry `SteamPath` is lower case with `/`** (`c:/program files (x86)/steam`), and
    Windows paths compare case-insensitively, so that spelling wins a dedupe and shows in the UI.
    `toRealPath()` gives the real one.
27. **pip evaluates `python_version` markers against the Python running it, not
    `--python-version`.** To check a pinned file for Python 3.10-3.14 on Linux and Windows from one
    machine:
    - resolve the markers per target with pip's own `packaging` first;
    - then `pip install --dry-run --python-version X --platform Y --report`.

    numpy needed three pins (2.2.6, 2.4.6, 2.5.4).
28. **A cached "is the pack there?" answer went stale**, so a pack switched on showed up late.
    - Cached resource checks must be dropped on every reload (Fabric API v1
      `ResourceLoader.get(CLIENT_RESOURCES).registerReloader`).
    - `Options.updateResourcePacks` only reloads when the selected list changed: call
      `reloadResourcePacks()` for a rebuilt pack that's already on.
29. **Java's `Properties` mangles Windows paths.** `store` writes `C\:\\...`, and `load` eats
    single backslashes typed by hand. Read that key's raw line, and write the file by hand with `/`.
30. **After the 26.3 port the setup screen showed only its title and buttons**, though every gametest
    check passed. Two 26.3 changes, both silent: a `ScrollableLayout` given its maximum height before its
    content was laid out stays 0 px tall, and a `Screen` now has `minecraft` from its constructor, so an
    `if (minecraft != null) rebuildWidgets()` built a second set of widgets. Causes and fixes:
    [Fabric mods for 26.x](../../techniques/fabric-mod-for-26-x-without-mappings.md), Gotchas 9-10. State checks
    can't see layout: look at the screenshots.

## Assets
None generated. All art and audio come from the player's own install through the asset tool. The
hurt vignette is a fitted analytic curve, `0.086 + 0.914 * (1 - exp(-(r/0.287)^2.75))`, within
0.02 opacity of the game's image, so the mod needs no game image for it.

## Cost and time
Several long sessions over about two weeks, mostly verification. The damage-look and hand-shader
investigation (capture, fit, disassembly, port, in-game check) took about one session.

## Open questions
- The red `worldTint` on lit surfaces, which needs a lit reference scene.
- Minecraft samples aim and buttons at 20 Hz (the game at 50 Hz or more), so a fast mouse flick can
  skip a ledge the game would catch. Sampling the aim every frame into the 0.2 s grab memory would
  close most of the gap.
- Looking 15 degrees past straight up, which Minecraft clamps at 90.
- Items, the inventory and perks are not ported (the public beta is climbing only). They come next,
  on 26.3 only (0.4.0 is the last 1.21.9 version).
- The server trusts the client's "hanging" flag (no fall damage or fly kick while set). It needs a
  server-side plausibility check before use on public servers.


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

