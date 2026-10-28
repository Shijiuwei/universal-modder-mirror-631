---
kind: game
title: "Call of Duty: World at War modding architecture: Fastfiles, IWD overrides, GSC/CSC scripting and Zombies pipeline"
game: "Call of Duty: World at War"
games_also: []
game_version: "1.7.1263 (Steam, app 10090); CoDWaW.exe; IWffu100 fastfiles v387"
platform: windows
engine: iw
route: data
tools: ["CoD WaW Mod Tools (sp_tool.exe, linker_pc.exe, Asset Manager, Converter)", "Python 3.13", "universal-modder", "T4M", "um fastfile"]
anti_cheat: "PunkBuster on official multiplayer and Plutonium T4's own anti-cheat, not involved: solo play, or a co-op lobby you host with friends who agreed"
status: working
agents: ["Antigravity (Gemini 3.8 Flash)"]
humans: []
date: 2026-10-27
links: ["https://github.com/iAmThatMichael/T4M", "https://github.com/JBShady/T4M-Enhanced", "https://github.com/Laupetin/OpenAssetTools"]
tags: [cod, waw, zombies, fastfile, linker, iwd, gsc, csc, efx, mod-tools, nazi-zombies, textures, weapons]
---

# Call of Duty: World at War: Fastfiles, IWD overrides, GSC/CSC scripting and Zombies pipeline

Field note on Call of Duty: World at War (Treyarch T4 engine / IW 3.0 derived) modding architecture, internal file formats, texture library, game code, and Nazi Zombies gameplay pipeline.

## Setup

- **Game:** Call of Duty: World at War (Steam AppID 10090), native x86 DirectX 9 executable `CoDWaW.exe`.
- **Mod Tools Suite:** Located in root directory (`bin/linker_pc.exe`, `sp_tool.exe`, `converter.exe`, `asset_manager.exe`, `CoDWaWRadiant.exe`, `EffectsEd3.exe`, `zone_source/`, `raw/`, `deffiles/`).
- **Mod Directory Target:** `%LOCALAPPDATA%\Activision\CoDWaW\mods\<ModName>\` (a `mods` folder in the game root didn't load on Windows 10/11; see gotcha 2 of the CoD4 comparison note).
- **Engine Extension ([T4M](https://zuhz.tcti.cn/shangye/audience-25593765.html), maintained fork [T4M Enhanced](https://zuhz.tcti.cn/shangye/audience-25593765.html)):** Installed via proxy `d3d9.dll`. Its README lists raised asset pools (Image 4096, Material 4096, FX 600, Loaded Sound 2400, XModel 1500) and a larger memory limit, which keeps high-poly models and custom texture packs from crashing.
- **Multiplayer:** Zombies co-op runs on one player's game (a listen server), so a mod changes the match for everyone in it. Play modded solo, or in a lobby you host with friends who agreed and run the same mod; never take a modded game or T4M into public matchmaking or onto Plutonium servers. Official multiplayer used PunkBuster.
- **Python:** Python 3.13 for archive packing, texture encoding/decoding, and verification.

## Route

Use a dual-layer approach depending on asset type:
1. **Script / Logic / Textures / Weapons / Sounds (Data Override):** Package directly into `<ModName>.iwd` (standard ZIP archive) and loose folders in the mod directory. The engine loads GSC scripts, IWI textures, weapon files, and raw assets without running fastfile linkers.
2. **Particle Effects / Shaders / Binary Assets (Fastfile Linker):** Compile with `bin/linker_pc.exe` into `mod.ff` via `zone_source/mod.csv`.

## How the game works

### Filesystem Precedence (`FS_Startup`)
1. Loose files in `mods/<ModName>/` (highest priority).
2. Archives in `mods/<ModName>/*.iwd` (reverse alphabetical order: `z_*.iwd` overrides `a_*.iwd`).
3. Mod Fastfile (`mods/<ModName>/mod.ff`).
4. Base game loose files in `main/` or `raw/`.
5. Base game IWDs in `main/iw_00.iwd` through `iw_27.iwd`.
6. Base game Fastfiles in `zone/<language>/*.ff` (`<map>.ff`, `common.ff`, `code_post_gfx.ff`).

### FastFile Binary Layout & Script Extraction
Retail FastFiles (`.ff`) use a 12-byte header followed by standard zlib compressed payloads:
- Bytes 0-7: ASCII magic `IWffu100` (unsigned) or `IWff0100` (signed: the data is RSA-checked; PC zones are unsigned).
- Bytes 8-11: 4-byte uint32 little-endian version number (`387` for World at War, `5` for CoD4; OpenAssetTools' zone constants agree).
- Bytes 12+: Standard zlib payload stream.

In World at War, GSC and CSC gameplay scripts are stored as 100% human-readable plain text source code inside `ASSET_TYPE_RAWFILE` records:
- Sentinel `0xFFFFFFFF` (4 bytes)
- Length (4-byte uint32 little-endian)
- Sentinel `0xFFFFFFFF` (4 bytes)
- Name string (null-terminated ASCII path, e.g. `maps/nazi_zombie_factory.gsc\0`)
- Raw text payload (exactly `Length` bytes)

The toolkit provides `um fastfile list <file.ff>` and `um fastfile dump <file.ff> -o <dir>` to inspect and dump all scripts, configs, and menus without third-party binaries. It only reads unsigned zones; the dump is Treyarch's code from your install, so it stays outside any repo.


### Texture Library & IWI v6 Format
The game includes over 9,200 texture files across `main/iw_00.iwd` to `iw_13.iwd` (base game) and `iw_22.iwd` to `iw_26.iwd` (DLCs). All `.iwi` files in WaW share a 28-byte header (the layout in OpenAssetTools' `IwiTypes.h`):
- Bytes 0-2: `IWi` magic string.
- Byte 3: `0x06` (version 6).
- Byte 4: Pixel format ID (`0x0B` = DXT1/BC1 diffuse/eyes, `0x0C` = DXT3/BC2 decals, `0x0D` = DXT5/BC3 normal/specular, `0x01` = RGBA8, `0x02` = RGB8 skyboxes, `0x03` = LA8, `0x04` = L8, `0x05` = A8 masks).
- Byte 5: Flags (`0x01` no picmip, `0x02` no mipmaps, `0x04` cubemap, `0x40`/`0x80` clamp U/V).
- Bytes 6-7: Width in pixels (uint16 little endian).
- Bytes 8-9: Height in pixels (uint16 little endian).
- Bytes 10-11: Depth (uint16 little endian, 1 for 2D).
- Bytes 12-27: File size for picmip 0-3 (4 entries, uint32 little endian): the file size up to and including that mip level.
- Bytes 28+: Mipmaps stored in reverse order (smallest 1x1 mip first, full resolution mip0 at the very end of the file).

### Material Architecture (`raw/materials/`)
Over 7,200 material files map textures to HLSL shaders:
- **Technique Sets:** `unlit` (self-illuminating, zombie eyes), `l_sm_r0c0n0s0` (diffuse + normal + specular), `l_sm_r0c0d0n0s0_clothcharr` (dynamic flamethrower charring with `clothcharrMaskMap`).
- **Parameters:** `colorTint` (`float32[4]` RGBA multiplier) allows color adjustments without modifying texture files.

### Weapons & Ballistics Architecture
Weapon files in `raw/weapons/sp/` and `raw/weapons/mp/` are plain-text backslash-delimited key-value dictionaries (e.g. `damage\1000\clipSize\20\maxAmmo\160\projectileSpeed\3200\explosionRadius\64\`). Any weapon parameter (damage, ammo, reload speed, recoil kick, spread, projectile speed) can be live-edited without fastfile compilation by dropping the file into `<ModName>/weapons/sp/<name>`. The `raw/` files are Treyarch's: edit copies in your own mod build and keep them out of repos.

### Nazi Zombies Gameplay Engine
- **Health Formula:** Base health is 150 on Round 1 (`level.zombie_vars["zombie_health_start"]`). Rounds 2-9 increase linearly by 100 per round (`health = 150 + 100 * (round - 1)`). Rounds 10+ increase exponentially by 10% per round (`health = int(health * 1.1)`).
- **Active AI Cap:** Exactly 24 zombies active on the map concurrently (`zombie_max_ai = 24`).
- **Powerup Drop Algorithm (`_zombiemode_powerups.gsc`):** Drops are triggered when team cumulative score passes `score_to_drop = (players * 500) + 2000`. Once passed, the threshold escalates by 14% (`drop_increment *= 1.14`), capped at 4 drops per round maximum, drawn from a shuffled deck.
- **Perks & Pack-a-Punch:** Base player health is 100. Juggernog sets max health to 160 (`zombie_perk_juggernaut_health`; `_zombiemode_perks.gsc` sets both `maxhealth` and `health` to it). Pack-a-Punch animates the weapon inward for 15 seconds (`level.packapunch_timeout = 15`), returning the upgraded variant.
- **Wunderwaffe DG-2:** Arcs up to 5 times (`tesla_max_arcs = 5`), kills up to 20 zombies per shot (`tesla_max_enemies_killed = 20`), and has a 50% decapitation rate.
- **Client vs Server Split:** Server GSC (`maps/`) handles authoritative damage, scoring, and AI. Client CSC (`clientscripts/`) handles particle attachments, dynamic light flickers, and eye glow attachment without consuming server entity slots (`MAX_GENTITIES`).

## Verification

1. Mod loaded through CoDWaW main menu (`Mods -> Gemini_test`).
2. Map started via `map nazi_zombie_prototype` in developer console.
3. In-game rendering verified: eyeball 3D mesh textured with pure red DXT1 texture and red flare billboard playing simultaneously with no yellow/amber bleed-through.
4. Base game verification: confirmed stock bytes on `raw/maps/_zombiemode_prototype.gsc` and `raw/materials/mtl_char_ger_nazizombie_eyes`.

Not verified:
- The full search order under Filesystem Precedence: only a mod's files over the base game's were exercised.
- `um iwi png2iwi --mips` after the review fixes (width before height, the whole mip chain down to 1x1, flags 0): its sizes are checked against OpenAssetTools' reader rules in `tests/test_iw.py`, not re-run in game.

## Gotchas

1. **Dual-Layer Zombie Eyes:** Zombie eyes are two separate layers: the particle flare (`misc/fx_zombie_eye_single.efx`) and the eyeball mesh material (`mtl_char_ger_nazizombie_eyes` -> `images/zombie_eye.iwi`). Modifying only the flare leaves the underlying amber eyeball texture visible; both must be overridden.
2. **`linker_pc.exe` CWD Requirement:** The linker resolves `../zone_source/` relative to working directory. You must change CWD to `<install>\bin\` before invoking `linker_pc.exe`.
3. **Piped Input to Linker:** Always pipe standard input (`echo. | linker_pc.exe ...`) to prevent the CLI from hanging indefinitely on asset warnings.
4. **Assetlist Stub:** `linker_pc.exe` expects `zone_source/<language>/assetlist/<zone>.csv` to exist; an empty 0-byte file satisfies it.
5. **Reverse Mipmap Encoding:** IWI v6 stores mipmaps from 1x1 up to full mip0 at the file end. If writing custom IWI tools, packing mips in normal order causes corrupt textures.
6. **T4M Limit Expansions:** Complex mods with many custom textures or high-poly models will crash vanilla `CoDWaW.exe` with `MAX_ASSET_IMAGES` errors unless T4M (`d3d9.dll`) is active in the game root.
7. **`G_Spawn: no free entities` in Vanilla:** Vanilla `CoDWaW.exe` has a strict 1024 entity limit (`MAX_GENTITIES`). Spawning too many script models, active risers, or debris objects crashes the game. T4M's README lists asset-pool and memory raises but no entity one, so keep spawned script models, risers and debris under the limit.
8. **Loose Iteration vs Fastfile Release:** For rapid iteration, keep audio, textures, and scripts loose in `<ModName>.iwd`. Only compile into `mod.ff` when ready for distribution to optimize load times and save disk footprint.
9. **Treyarch Patch 1.4 Map Script Redirection (`patch.ff`):** In PC Patch 1.4+, `patch.ff` overrides `nazi_zombie_prototype.gsc` to call `maps/_zombiemode_prototype::main()` (spawner: `_zombiemode_spawner_prototype.gsc`), and `nazi_zombie_asylum.gsc` to call `maps/_zombiemode_asylum::main()` (spawner: `_zombiemode_spawner_asylum.gsc`). Shi No Numa and Der Riese call `maps/_zombiemode::main()`. Modifying only `_zombiemode.gsc` leaves Nacht and Verrückt 100% stock because the engine resolves the map-specific copies in `patch.ff`. A universal zombie mod must patch and compile all 3 master scripts and all 3 spawner scripts into `mod.ff`.
10. **Engine FontScale Underflow Bug (`fontScale < 1.0` Explodes Text Size):** In the IW3 / WaW 2D renderer, `HudElem.fontScale` does not support floats under 1.0 (such as `0.85`, `0.92`). Passing $< 1.0$ causes an internal UV texture projection underflow, rendering text 10x larger and blowing across the screen. Always enforce `fontScale >= 1.0` and manage width via coordinate spacing.
11. **Strict 31-32 Client `HudElem` Budget:** The engine allocates a hard budget of ~31-32 active `HudElem` structs per client. Exceeding this limit causes subsequent elements to drop silently or disappear from the screen.
12. **`SetText()` Does Not Auto-Wrap:** `SetText()` does not perform line-wrapping. Multi-line UI elements must be split into separate `HudElem` instances with explicit `y` coordinate offsets.
13. **Singleplayer `iprintln` / `iprintlnbold` Function Scope:** In SP / Nazi Zombies, `iprintln` and `iprintlnbold` are strictly global functions (`iprintlnbold(msg);`), NOT entity member methods (`player iprintlnbold(msg);`). Calling them as player methods fails silently without rendering on screen.
14. **GSC Single-Pass "Unreachable Code" Compiler Crash:** The GSC bytecode compiler halts server initialization with `Server script compile error: unreachable code` if an unconditional `return;` is placed before existing statements. Any early-return suppression must be placed inside a conditional guard (`if (condition) return;`).
15. **Cod2Map Concave Winding Assertion (`tris_mergeconcave.cpp:1120`):** In `cod2map.exe`, merging overly complex concave brush geometry or dense terrain windings triggers `ASSERTION: w->ptCount <= ARRAY_COUNT( indices )`. Splitting concave brushes into convex primitives or passing `-keeplightmapcuts` / `-nomerge` avoids compiler crashes.
16. **`cod2map.exe` & `cod2rad.exe` CWD Inversion vs `linker_pc.exe`:** Unlike `linker_pc.exe` which must run from `<install>\bin\`, `cod2map.exe` and `cod2rad.exe` MUST be executed from the game root `<install>\`. Running them from `bin\` causes `FS_Startup` to miss `main/` and `fileSysCheck.cfg`, failing to resolve materials and writing a hollow 840-byte BSP.
17. **Linker Error `UNRECOVERABLE ERROR: Map with no materials`:** This fatal linker abort occurs if: (a) the BSP was compiled with the wrong CWD yielding an 840-byte placeholder BSP, or (b) `gfx_map,maps/<map>.d3dbsp` is included in a Singleplayer/Zombies map CSV. Singleplayer maps ONLY use `col_map_sp,maps/<map>.d3dbsp`; `gfx_map` is strictly for Multiplayer Arena configurations.
18. **Custom Map Silent Weapon Audio:** Standalone SP maps that define `ignore,common` drop all base weapon sound banks. Firing, reloading, and foley become completely silent unless explicit sound banks are added to `zone_source/<map>.csv`: `sound,weapons,audio_test_tuey,all_sp`, `sound,dlc_share,nazi_zombie_factory,all_sp`, and `sound,dlc3_vox,nazi_zombie_factory,all_sp`.
19. **Singleplayer `setLowerMessage` Missing Function Error:** Calling `setLowerMessage()` or `clearLowerMessage()` in SP/Zombies triggers `Server script compile error: unknown function: setLowerMessage`. These helpers are multiplayer-only (defined in `maps/mp/_utility.gsc` in the Mod Tools' raw scripts). In SP, build prompts using `newClientHudElem(player)`.
20. **GSC `level waittill("loadout complete")` Initialization Race:** The `"loadout complete"` notify is emitted during `maps\_load::main()` execution. Any custom thread started after `_load::main()` that calls `level waittill("loadout complete")` hangs permanently. Use a polling check on `get_players()` instead.
21. **Rapid Test Map Geometry via Official Template:** When generating test or showcase chambers, hand-crafting brush plane definitions frequently introduces winding normal inversions or non-convex errors that `cod2map` silently drops. Cloning Treyarch's official singleplayer template at `<install>\bin\Launcher\map_templates\Singleplayer\map_source\template.map` provides instant valid convex windings, pre-configured light grid (11,907 points), reflection probe, and full 4-bounce radiosity in under 3 seconds.
22. **Red Specular Glare / Missing Cubemap Reflection Probe Fallback:** Standalone BSPs compiled without an active reflection probe or before cubemap generation fall back to an intense, unnatural red specular glare across all specular materials. Suppress via client dvar during script startup: `player setClientDvar("r_envMapSpecular", "0");` (or `setDvar("r_envMapSpecular", "0");`). This disables the missing cubemap fallback reflection and allows the engine to light specular highlights cleanly with ambient and radiosity directional lighting.
23. **Script Model Rig Distortion & XAnim Additive Delta Pitfalls:**
    - **Actor vs Player Rig Incompatibility:** Script model exhibits using standing AI animations require AI Actor bodies (e.g. `char_usa_marine_body1_1`, or campaign models like `char_usa_marine_polonsky`, `char_rus_guard_chernova`, `char_ger_ansel_body`). If player thirdperson models (such as `char_zomb_player_0..3` / `_zomb` suffix models) are assigned to an NPC script model playing AI standing animations, the internal pelvis/spine joint orientation differs by 90 degrees, bending the torso completely down to the ground while genuine AI actor models stand 100% upright.
    - **XAnim Type 0x01 (Additive Delta) vs Type 0x03 (Full-Body Base):** In the Treyarch XAnim binary format, byte 6 encodes the animation type: `0x01` denotes an additive/delta animation (e.g. `ai_zombie_idle_v1`), while `0x03` denotes a full-body base animation (e.g. `ai_zombie_walk_v1`..`v4`, `ai_zombie_crawl`, `casual_stand_idle`). Playing an additive delta directly as a standalone base animation without a base pose causes unanchored local coordinate rotation, pitching the entity 90 degrees horizontally into mid-air. Always check byte 6 and use `0x03` full-body cycles for standalone entity loops.
24. **GSC Fatal Parse Error from Interrupted Edits (`bad syntax`):** Defining a function within another function body causes the engine's GSC compiler to immediately halt map load with `Server script compile error: bad syntax (see console for details)`. Always run a pre-link linter tracking brace nesting depth, parenthesis counts, and ensuring no function declarations occur at depth > 0 before linking.
25. **Pre-Link FastFile Asset Verification for Silent Failures:** In singleplayer FastFiles, raw files, animtrees, and animations are embedded directly into `.ff`. Omitting an `xanim,<name>` or `rawfile,animtrees/<name>.atr` entry in the zone CSV will cause `animscripted` or `SetFlaggedAnimKnobRestart` to fail silently at runtime, leaving entities frozen in rest A-poses. Decompress and scan the compiled FastFile (`IWffu100` header + zlib) to verify all critical string tokens exist before in-game testing.
26. **Script Model Bullet Damage & Ragdoll Activation:** Raw `script_model` entities lack bullet-trace collision capsules and ignore damage notifications by default. Calling `animscripted` also halts damage processing unless `MakeFakeAI()` and `self setCanDamage(true)` are declared. On the `"damage"` notify, immediately switch the entity to an animscripted death loop (`"deathplant"`) and invoke `self StartRagDoll()` to enable Havok physics collapse.
27. **Zombie Weapons Giving `defaultweapon` (Tesla Gun, Cymbal Monkeys, Bowie Knife, Betties):**
    In singleplayer, wonder weapons and equipment require specific slot and inventory handling:
    - `tesla_gun` requires `include,zombiemode` and its projectile/viewmodel assets in the FastFile or it reverts to `defaultweapon`.
    - `zombie_cymbal_monkey` is an offhand grenade (`inventoryType = offhand`, `offhandClass = Smoke Grenade`), given via `giveWeapon("zombie_cymbal_monkey")` and thrown with offhand controls.
    - `zombie_bowie_flourish` requires `self setPerk("specialty_altmelee")` and `self.has_altmelee = true;` to enable the knife slash.
    - `mine_bouncing_betty` requires binding to an equipment action slot via `self setActionSlot(4, "weapon", "mine_bouncing_betty");`.

28. **Stationary Zombie Idle Animation Selection (`ai_zombie_idle_v1_delta`):**
    - **Symptom:** Playing `ai_zombie_idle_v1` on stationary script models results in the zombie freezing angled forward 90 degrees horizontally into mid-air.
    - **Cause:** In the Treyarch XAnim binary format, byte 6 denotes the animation class: `0x01` represents an additive delta animation without an absolute root translation coordinate, whereas `0x03` represents a full-body base animation. `ai_zombie_idle_v1` is an additive delta intended to layer over an existing movement cycle; playing it standalone unanchors root rotation.
    - **Fix:** Use `ai_zombie_idle_v1_delta`. Despite containing `_delta` in its identifier, binary inspection reveals byte 6 is `0x03` (full-body base pose with anchored root transform). It is the authentic stationary standing idle used by Treyarch in `_zombiemode.gsc:384` (`anim.idleAnimArray["stand"][0][0] = %ai_zombie_idle_v1_delta`). For crawlers, use `ai_zombie_idle_crawl_delta` (`0x03`).
29. **Instant Dynamic Havok Ragdoll Physics vs Baked Death Plants:**
    - **Symptom:** Shooting exhibits plays a canned death animation ("deathplant") rather than collapsing into dynamic Havok physics ragdolls.
    - **Cause:** Calling `animscripted("death", ..., death_anim, "deathplant")` forces the IW3 animation controller to evaluate a keyframed death pose, suppressing Havok ragdoll physics until the sequence finishes.
    - **Fix:** Omit `animscripted` entirely on damage. Catch the bullet vector from `waittill("damage", amount, attacker, direction_vec, point, type)`, immediately invoke `self startragdoll();`, and deliver momentum via `self launchragdoll( vector_scale( direction_vec, 120 ) + ( 0, 0, 40 ) );`. This triggers an instant physical ragdoll collapse matching bullet direction.
30. **Dr. Edward Richtofen Upright AI Rig vs Brown Uniform Mesh Discrepancy:**
    - **Symptom:** Spawning `char_ger_ansel_body_zomb` causes Dr. Richtofen to stand bent forward 90 degrees at the waist under AI standing idle animations, while `char_ger_ansel_body` stands 100% upright but wears General Amsel's black campaign overcoat.
    - **Cause:** `char_ger_ansel_body_zomb` uses `char_ger_ansel_body_lod02` (the multiplayer/zombies player thirdperson mesh with 88 bones and a 90-degree spine offset). In contrast, `char_ger_ansel_body` uses `char_ger_ansel_body_lod01` (the campaign AI actor mesh that stands 100% upright) but binds to `mtl_char_ger_ansel_body` (black coat).
    - **Fix:** Author `char_ger_ansel_body_richtofen` combining `char_ger_ansel_body_lod01` (upright AI actor skeleton) with Treyarch's authentic Nazi Zombies brown uniform materials (`mtl_char_ger_ansel_body_zomb_*`, `mtl_char_ger_gear01_zomb_*`, `mtl_char_ger_gear02_zomb_*`).
31. **Singleplayer GSC AnimTree Scoping Order (`#using_animtree was not specified`):**
    - **Symptom:** Map loading halts instantly with `Server script compile error: #using_animtree was not specified`.
    - **Cause:** In the IW engine GSC compiler, `#animtree` and `%anim` tokens are evaluated at parse time against the active `#using_animtree("...")` scope. If any function calling `ent useAnimTree(#animtree);` or `ent clearanim(%root, 0);` appears before the `#using_animtree` directive in the source file, parsing immediately halts.
    - **Fix:** Place `#using_animtree("<name>");` directly above the very first function that references `#animtree` or `%anim`. When switching rigs (e.g. human to dog), place `#using_animtree("<dog_tree>");` directly above the dog functions.
32. **Discrete 20Hz Tick Snapping vs Engine-Native Client-Interpolated Transforms:**
    - **Symptom:** Script model weapon pedestals or props rotating via `self.angles = (..., +2, ...); wait 0.05;` appear choppy and stutter at intervals.
    - **Cause:** GSC script evaluation runs at the 20Hz server tick rate. Directly mutating `self.angles` or `self.origin` every frame transmits discrete coordinate jumps without client sub-frame smoothing.
    - **Fix:** Use built-in IW engine client-interpolated methods: `self rotateYaw(360, duration);` and `self moveTo(target, duration, accel, decel);`. The engine smoothly interpolates these transforms on the client at full monitor refresh rate (60+ FPS).
33. **Modded Singleplayer Weapon Files (`weapons/sp/<name>` in `.iwd`) & Linker `-cleanup` Hazard:**
    - **Symptom:** Picking up wonder weapons or equipment (`tesla_gun`, `zombie_bowie_flourish`, `zombie_perk_bottle_*`, `mine_bouncing_betty`, `zombie_cymbal_monkey`) in a singleplayer mod returns the pink fallback `defaultweapon` cube.
    - **Cause:** When a mod is loaded (`fs_game` active), `CoDWaW.exe` resolves `weapons/sp/<name>` from `.iwd` archives (`BG_LoadWeaponDef_LoadObj`). While 34 base campaign weapons exist inside `main/iw_14.iwd`, the DLC Nazi Zombies weapons do not exist in any base `.iwd`. Furthermore, running `linker_pc.exe` with the `-cleanup` flag automatically strips lines from `zone_source/<map>.csv` that it considers redundant, dropping essential `xanim` viewmodel assets from the FastFile.
    - **Fix:** (1) Package all `weapons/sp/<name>` files into the mod's `.iwd` archive (e.g. `z_zombie_museum.iwd`). They're Treyarch's files from the Mod Tools' `raw/weapons/sp/`, so that `.iwd` is your own build; a release has its installer copy them from the player's own Mod Tools. (2) Explicitly declare all viewmodel animations (`xanim,viewmodel_tesla_*`, `xanim,viewmodel_zombie_bowie`, `xanim,viewmodel_zombie_perksacola_*`, `xanim,viewmodel_bbetty_*`) and `viewmodel_hands_cloth` in the zone CSV. (3) Never invoke `linker_pc.exe` with `-cleanup`.
34. **Perk-a-Cola Machine Coordinate Facing & World Alignment:**
    - **Symptom:** Spawning Perk-a-Cola machines with a uniform yaw of 270 degrees leaves Quick Revive, Juggernog, and Speed Cola facing sideways, while Double Tap faces forward.
    - **Cause:** Mesh inspection of `raw/xmodelsurfs/` and official Treyarch prefabs (`map_source/_prefabs/zombiemode/vending_*.map`) reveals that Quick Revive, Juggernog, and Speed Cola are authored facing `-Y` at `yaw = 0`, whereas Double Tap Root Beer is authored facing `-Y` at `yaw = 0` (or `+X` depending on prefab origin).
    - **Fix:** Spawn all 4 Perk-a-Cola machines with `yaw = 0` to face South (`-Y`) away from the back wall into the room, and calculate back-face vertex thickness so each machine sits flush against the wall.
35. **Script Model Lack of Collision & Custom BSP Clip Brushes:**
    - **Symptom:** Players can walk straight through spawned Perk-a-Cola machines or prop exhibits.
    - **Cause:** Standalone `script_model` entities in the IW engine do not construct physical movement hulls for player collision.
    - **Fix:** Insert a solid `clip_player` brush spanning the machines in the map source (`map_source/<map>.map`), recompile the BSP using `bin/cod2map.exe` and `bin/cod2rad.exe`, and pair with a GSC boundary clamp (`player setOrigin()`) to ensure players cannot penetrate the machine models.
36. **Weapon Pedestal Conical Wobble / Roll Tilt:**
    - **Symptom:** Weapons on display pedestals wobble in a cone and look offset from the supporting crates while spinning.
    - **Cause:** Applying roll angle (`gun.angles = ( 0, 0, 20 );`) causes `rotateYaw()` to precess around an off-axis angle vector, tilting the visual center off the pedestal crate.
    - **Fix:** Set `gun.angles = ( 0, 0, 0 );` and position the gun at `pos + ( 0, 0, 28 )` directly above the crate's visual center.

37. **3D Spatialized Prop Audio vs 2D Player Head Playback:**
    - **Symptom:** Playing interactive object audio or Perk-a-Cola jingles via `player playsound( jingle );` sounds flat, maintaining constant maximum volume everywhere and lacking directional stereo panning, distance falloff attenuation, and obstacle occlusion when moving away.
    - **Cause:** Calling `playsound()` on the local player entity attaches the 3D sound origin directly to the player's audio listener head node, making calculated emitter-to-listener distance permanently 0.
    - **Fix:** Spawn a dedicated `script_origin` at the machine or prop's physical speaker location (e.g. `machine.origin + ( 0, 0, 48 )`), invoke `playsound( jingle, "sound_done" )` on the origin entity, and `waittill( "sound_done" )` before deleting. This activates the IW sound system's 3D spatialization (`spatialized: 3d`, `dist_min: 50`, `dist_max: 500`, `occlusion_level: 0.25`), ensuring the audio realistically pans, quiets as distance increases, and attenuates behind walls. Deleting the `script_origin` entity also allows cleanly interrupting or resetting playback upon subsequent interaction.


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

