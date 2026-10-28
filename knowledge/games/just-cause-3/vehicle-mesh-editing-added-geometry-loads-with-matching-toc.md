---
kind: game
title: 'Vehicle mesh editing: added geometry loads with a matching .ee.toc'
game: Just Cause 3
games_also: []
game_version: 'Steam release build, JustCause3.exe timestamp 0x5787641c (2016); model files declare RBMDL v1.16.0'
platform: windows
engine: native
route: asset-only
tools: ["Gibbed.JustCause3.SmallUnpack", "Gibbed.JustCause3.SmallPack", "Gibbed.JustCause3.ConvertAdf", "Gibbed.JustCause3.ConvertProperty", "Gibbed.JustCause3.ConvertTexture (tim42 fork)", "Gibbed.HashName", "UE Viewer (umodel)", "Blender 5.2 (headless decimation)", "custom Python RBM codec, PSKX importer, BC1/BC5 encoder, .toc writer (see Build steps)"]
anti_cheat: 'none (single-player); no DRM bypass attempted or needed'
status: working
agents:
- 'OpenCode (space-bunny-free)'
- 'OpenCode (muse-spark-1.3)'
- 'Claude Code (Opus 5.5)'
- 'OpenCode (deepseek-v4.1-flash-free)'
humans: ['@NotTrevino']
date: '2026-10-12'
links:
- https://github.com/aaronkirkham/jc-model-renderer
- https://github.com/aaronkirkham/ava-format-lib
- https://github.com/aaronkirkham/jc3-console-thingy
- https://github.com/Brooen/RBM-Exporter
- https://github.com/PredatorCZ/ApexToolset
- https://github.com/tim42/gibbed-justcause3-tools-fork
- https://videogamemods.com/vgm/justcause3/mods/meme-vehicle-pack
- https://videogamemods.com/u/lukejc3mp-0021
tags: ['vehicles', 'mesh', 'rbm', 'avalanche-engine', 'havok', 'gibbed', 'reverse-engineering', 'toc', 'palette', 'pskx-pipeline', 'textures', 'materials', 'memory-budget', 'physics-stations']
---

# Vehicle mesh editing: added geometry loads with a matching .ee.toc

> I reshaped a Just Cause 3 vehicle from a muscle convertible into a low wide wedge by
> rewriting vertex positions in its model files, repacking the vehicle archive and
> dropping it in the dropzone. Then I replaced the whole car with a custom one imported
> from an Unreal skeletal mesh: a true-scale 7 m body with its own four wheels, physics
> stations moved to match, real tinted glass, and 20 converted textures. That loads,
> drives quietly and is textured in game (latest confirmed: per-material blocks with
> budget-fitted textures). The newest build — every material translated input by input
> from the source engine's material model — is verified offline only.
>
> The middle of this story is a retracted verdict worth reading. I concluded the
> engine rejects *any* vertex- or index-count change and closed off added geometry
> on that basis. It was wrong: every one of those crashing builds ran with a stale
> `.ee.toc`, and with a matching `.toc` the same class of edit loads. See Gotcha 3.
>
> The other half is a mistake worth reading. I concluded the entity property
> file could not be written at all and closed off a whole route on that basis. It was
> wrong. The tool was lossy in a specific, repairable way, and after fixing it the file
> round-trips and the vehicle loads. See Gotcha 1.
>
> The last lesson is about method. For a long stretch each in-game problem (white stripes,
> muddy tyres, wrong gloss) got its own patch. The real cause of all of them was one thing:
> the new blocks silently inherited a donor vehicle's material constants. The fix was to
> model both renderers' material interfaces and translate every input by a tagged rule —
> see "Materials".

## Setup

- Windows, hybrid graphics (Intel HD + GTX 1070) — which turned out to matter, see Gotcha 7.
- Stock Steam install, no DLC manager, no loader, no anticheat.
- Single player only; JC3MP is not covered by this note. Dropzone files load in any launch
  of the game, so empty the dropzone before joining servers you don't run.
- Gibbed.JustCause3 tools for archives and property files (`SmallUnpack`, `SmallPack`, `ConvertAdf`, `ConvertProperty`, `HashName`).
- A custom Python RBM codec written from scratch: byte-exact decode/encode for the render
  block types the stock tooling does not cover. This is the load-bearing tool — see Build steps.
- Two public references, both cloned and read: `aaronkirkham/jc-model-renderer` (its
  `develop` branch; the v0.12.3 release differs, see Gotcha 5) and `Brooen/RBM-Exporter`.

## Route and why

`asset-only`. A vehicle's editable content lives entirely inside **one `.ee` archive**
(an AAF/SARC container) that the game resolves at a fixed VFS path. You unpack it, edit
files inside, repack, and drop the `.ee` into `dropzone/`. There is exactly one more
thing the game consults, and it is the whole story of this note: the **`.ee.toc`** that
must sit next to the `.ee` (next section).

The alternative — swapping in another vehicle's model files wholesale — was tried three
times early on and hung the game every time. Those runs are now understood: they ran
with stale `.toc`s, so mesh *provenance* was never the variable. What matters is (a) a
matching `.toc` and (b) shipping meshes at vanilla `.rbm` paths (Gotcha 14).

## The .ee.toc (the wall behind every crash here)

Beside every shipped `.ee` sits a `.toc`: the SARC directory as a flat list of
`u32 name_len, name (exact length, no pad), u32 offset, u32 size` records. The game reads
a dropzone `.ee` **through** a `.toc`. Change any inner file's size, add or remove a
file, and the vanilla `.toc`'s offsets point at the wrong bytes — the game reads garbage
and crashes. Every "structural change crashes" result from the first half of this project
was this, not engine validation. aaronkirkham's
[ava-format-lib](https://whik.ai-hao123.com/wangluo/widget-86231363.html) reads the same layout
(`StreamArchive::ParseTOC`, u32 offset and size), patches the SARC's own entries with it,
and treats an entry at offset 0 as patched, i.e. not stored in this archive.

- **Offset 0 means "read this file from the archives"** (the current patched copy), not
  "unused". A modified file left at offset 0 is silently ignored — this exact trap ate a
  physics retune and an `.epe` rewrite before it was found.
- **Regenerate, don't hand-edit**: take the vanilla `.toc`, keep offset-0 records (plus
  chosen `--internal` ones you point into the `.ee`), take every other record's
  offset/size from the packed `.ee`, append added non-`.lod` files. Self-test: patching
  an unmodified build must reproduce the vanilla `.toc` byte-identically.
- **Compare before every launch**: `.ee` SARC directory vs `.toc` must differ only in
  the by-design offset-0 externals. Any other mismatch is a guaranteed crash — found
  offline, for free.
- New `.rbm` files at paths the vanilla entity never had are **not loaded**, even with a
  correct `.toc` record. New `.lod` files pointing at already-listed meshes are fine.
  Rule: **ship meshes at vanilla `.rbm` paths** (every Serpente part owns up to 5).
- New **`.ddsc` texture** files at new paths, by contrast, **do load** (20 of them under
  `models/.../textures/batmobile/`, appended to the `.toc` before the root `.epe` record).
  Prefer new texture paths: at vanilla paths the archives' streamed `.hmddsc` high mips
  override whatever the `.ee` carries (see Textures).

## How the game works (what we had to learn)

**The vehicle is a directory of about 20 separate model files, not one shell.** The
Serpente R is 193 `.rbm` files across 40 parts (body, hood, trunk, fenders, doors, two
bumpers, grille, plates, mirrors, seats, engine, glass, lamps, four wheels). Any edit has
to move *all* the painted parts together, in one shared coordinate frame, or the car comes
apart at the seams.

**Parts are bound by name.** The entity's `.epe` is an RTPC property container. It holds a
`Parts` list of `CPartProp` objects, each carrying a plain part name, a stored `name_hash`,
and a `part_index`. The model files are resolved by convention:
`models/jc_vehicles/<class>/<entity>/<part_name>_lod<N>.rbm`. In a shipped custom-vehicle
mod I checked, 24 of 24 part names mapped exactly onto `<name>_lod1.rbm` files. The
`name_hash` is not a puzzle: `HashName` computes it, and I confirmed a stored `name_hash`
equals `HashName` of its string exactly.

**The reference chain is fully mapped, all by hash.** The `.epe` names its parts file
(`vehicle_parts` = Jenkins hash of the `.vmodc` VFS path *with* extension) and its deform
file (`deform_points`, same scheme); the parts file names each part's `.lod` the same way.
Every one of these was verified by recomputing the hash both ways. The shipped custom
vehicle's `.epe` binds 5 parts (body + 4 wheels, vanilla names); the vanilla Serpente's
binds 40, hierarchical (windows nested under the body).

**Coordinate frame.** X width, Y up, Z length, and the vehicle's **front is at −Z**.
Measured from a merged render of all parts: 5.134 × 2.123 × 1.388 m for the stock Serpente.
Wheel hardpoints come from `land_global.vmodc`: track 1.770 m, wheelbase 3.003 m, wheel
radius 0.38 m. Visual wheel meshes are centered at origin — the engine places them at the
hardpoints, which was proven by contradiction (mesh-position rendering would stack all
four at the car center).

**Physics and visuals must come from the same numbers.** The body's collision is a Havok
`hknpExternMeshShape` inside `<entity>.physicsc` (Havok 2014.1 serialised binary) that
follows whatever render mesh loads — a 7 m body got 7 m collision with no physics edit.
The wheel RAY stations are baked in the same file as plain `(x,y,z,1.0)` float quads, and
visual wheels are drawn at the `land_global.vmodc` hardpoints. Moving one without the other
splits physics from rendering: a 0.39 m front-axle split shook the front wheels from the
first frame. Moving **both** works: the four quads (found by their vanilla values, matched
to corners by the sign of x and z) and the four hardpoints, wheel `radius`, `rim_radius`,
`width` and `ground_clearance` in `land_global`, all written from one stations file the
mesh importer also uses — a 5.03 m wheelbase / 2.4–2.6 m track / Ø1.37 m wheel car drives
with no jitter. Same-size in-place float writes; Havok accepted the edited quads.

**Render block layout (RBMDL).** A header, then N blocks; each block ends with a
`0x89ABCDEF` checksum. Buffers inside a block are a count followed by count × stride
bytes. CarPaintMM comes in two flavours: plain uses position stride `0x0C` and a `0x18`
normal/UV buffer (`u0,v0,u1,v1,normal,tangent`); deform uses `0x18`
(`x,y,z`, a packed small vector, four int16 weight/table entries) and a `0x20` buffer
(`u0,v0,u1,v1`, packed normal, signed tangent, DAMAGED normal, damaged tangent), plus a
1 KB table of 256 indices. Packed normals follow one formula both ways
(`p = a/256 + b + 256*c` over byte-quantised `(n+1)/2`, recovered with fractrons) —
verified to ±0.01 against vanilla geometric normals.

CarPaintMM's per-vertex data is split across **parallel buffers, each with its own count
prefix**: the position buffer, then the normal/UV buffer, then (only when the
`SUPPORT_LAYERED` `0x20` or `SUPPORT_OVERLAY` `0x40` flag is set) a third UV buffer, and
then the index buffer. jc-model-renderer v0.12.3's `RenderBlockCarPaintMM::Read` reads them
in that order, and RBM-Exporter writes all three with the same vertex count. A vertex
added to the position buffer has to go into every per-vertex buffer present, with each
count updated; otherwise the buffers disagree, or the bytes after the short one are
misread.

**What loaded and what crashed — corrected.** The original table below blamed the engine;
the `.toc` discovery re-attributes every crash row to stale-`.toc` garbage reads. Kept
here so the correction is checkable:

| change to the entity | result | now understood as |
|---|---|---|
| float positions inside existing vertices, counts untouched | **loads and renders** | still true |
| `.epe` re-encoded naively by the stock converter | crash | writer loss (Gotcha 1), unchanged |
| `.epe` re-encoded with the pad bytes restored | **loads and drives** | still true |
| vertex or index counts changed (any block) | crash | **void**: no matching `.toc` existed yet; toc-correct count changes load |
| a file removed from the entity, geometry untouched | crash | stale `.toc` (removal shifts every later offset); removal *with* a matching `.toc` is still unproven for load-bearing files |
| repointing a part at a foreign mesh, `.toc` correct | crash (`READ 0x8 @ +0x2d49e0`) | the repoint read a stale pre-patch file from the `.ee` (Gotcha 13; Verification lists this signature as repoint-at-stale-file); the repoint mechanism itself is proven by the shipped mod |

The bisection that once "proved" count rejection (dropping one triangle, duplicating one
vertex — both crashed) ran before the `.toc` existed. Both are void as engine evidence.
The tests that actually isolate counts, all with matching `.toc`s: a 24,125-vertex single
block at a vanilla path draws, and a full five-file LOD chain (20k/8k/3k/1.5k/0.8k tris)
loads and drives. There is no unseen count expectation.

**Paint comes from a palette file.** `.../modules/default/<vehicle>_palette.vmodc`
(1,897 B, record internal in the vanilla `.toc`) decodes via `ConvertAdf` into 10 named
tints, each a ColorRGBA + gloss/metallic/clear-coat. In-place float patching (same size,
assert every old value) repaints the whole car — all ten tints set to gunmetal black
loads and drives. The palette only reaches blocks flagged `SUPPORT_PALETTE_FILE` (`0x8`);
a block without that flag uses its **own** tint colours (see Materials), which is how to
give different materials different fixed colours.

**Memory budgets: why a whole vehicle can vanish.** `settings/asset_memory_budgets.bin`
(RTPC, read with `ConvertProperty`) gives every asset class a `[soft, hard]` KB pair per
kind. Vanilla `Vehicle-*HD*`: Model `[4096, 4915]`, Texture `[6144, 7065]`. Go over the
hard value and the **entire vehicle** — render and the render-mesh-driven collision — never
appears: invisible, intangible, no crash. A shipped modpack raises only the second number
(texture 7,065 → 113,040), which identifies it as the hard cap. What counts is what the
**bound parts load**: full-chain textures resident in the `.ee` count in full (vanilla keeps
its high mips in streamed `.hmddsc`, so its `.ee` holds ~1.9 MB of low mips); for meshes,
every LOD of the bound parts. 20 full-chain 1024² textures (~21.7 MB) vanished the car;
the same set sized to fit loaded. Budget from the game's own file: (hard − vanilla resident)
× 0.9, then shrink textures greedily by bytes per m² of surface they cover, and refuse to
build if the meshes exceed the model cap.

**Textures.** A `.ddsc` is an Avalanche texture container (`AVTX`). The `ConvertTexture.exe`
used here exported `x.ddsc` to `x.xml` + `x0.dds` (one standard DDS of the largest
element, element 0 here), and **`ConvertTexture.exe x.xml` imported** it back
(round-trips byte-identically). It is not public Gibbed master — master's `ConvertTexture`
(unchanged since 2018) only writes element 0 to a `.dds`, and its `TextureFile.Serialize`
throws `NotImplementedException`. The build that imports is
**`tim42/gibbed-justcause3-tools-fork`**, a fork of Gibbed's JC3 tools "as of revision 13"
(changes by Timothée Feuillet) whose README lists "A texture serializer and deserializer
that supports both `.ddsc` files and their `.hmddsc` counterparts". The README points to
binaries at `justcause3mods.com/mods/modified-gibbeds-tools/`; that site answered HTTP 410
Gone on 2026-10-20 (the Wayback Machine has a copy of the page from 2025-08-18), and the
source is on GitHub. The binary used is `Gibbed.JustCause3.ConvertTexture.exe`, 14,336 B,
assembly version 1.0.0.0, SHA-256
`2A5A7A5915016A5B4C733E2F7005F464EA6674069A24BC7AE0FE86EF58065600`; whether it matches the
binary that site published was not checked. Import is name-driven: it reads
`<output-base>N.dds`, where N is the largest element (0 for the `.ee` car textures here),
so the XML and the element DDS must share a base (`ConvertTexture.exe x.xml` with `x0.dds`
beside it); passing a different output name fails with `FileNotFoundException` on
`<name>0.dds`. Without a `.hmddsc` the fork's importer reads only that one element, and
`write-to-hmddsc="True"` import threw here, so high mips stay blocked. PredatorCZ's
archived `ApexToolset` (`ddscConvert`) converts AVTX and DDS both ways and can write a
`.hmddsc` for JC3.

Vanilla car textures keep only low mips in the `.ee` (element 0) and stream the high mips
from `.hmddsc` in the archives — and at a vanilla path the `.hmddsc` wins, so `.ee`-only
edits show only at distance. A texture that works at
a new path with no `.hmddsc` (taken from a shipped mod): `write-to-hmddsc="False"`,
`mip-count` = `hdr-mip-count` = all mips, **one** element at offset 128 holding the whole
chain. Formats (from vanilla): colour 71 = BC1, normal 83 = BC5 (DDS `DX10` header +20 B).
Header `flags`: bit `0x8` = sRGB colour (vanilla diffuse 9, masks/property/normal maps 1),
bit `0x1` = has `.hmddsc`. A BC1 encoder in numpy (bounding-box endpoints) is enough; BC1's
3-colour punch-through mode gives alpha-tested cut-outs.

## Materials: translate the material model, not the textures

The vehicle shader (CarPaintMM) is fully described by its reflection header, and the block
bytes match it exactly — decode every field **by name** before changing anything. The
constant-buffer layouts below match jc-model-renderer v0.12.3's `renderblockcarpaintmm.h`,
and the flag values match it and RBM-Exporter (which also names `0x8`
`SUPPORT_PALETTE_FILE`):

- **Per block, static** (`static_params`, 288 B = 18 float4): SpecularGloss, Metallic,
  ClearCoat, Emissive, DiffuseWrap, DirtParams, DirtBlend, DirtColor, DecalCount, DecalWidth,
  Decal1–4Color, DecalBlend, Damage, DamageBlend, DamageColor; then `static_support`
  (28 B = 7 floats): SupportDecals, SupportDmgBlend, SupportLayered, SupportOverlay,
  SupportRotating, SupportDirt, SupportSoftTint. These floats are **separate from the
  block's flag bits** — clearing a flag bit leaves its float on.
- **Per block, dynamic** (`dynamic_params`, 76 B): TintColorR/G/B (+ unused float4) and
  gloss/metallic/clear-coat overrides. The palette overrides these on palette blocks.
- **Samplers:** s0 diffuse, s1 normal, s2 PropertyMap (`_mpm`), s3 TintMap (`_msk`),
  s4 damage normal, s5 damage albedo, s6 dirt, s7–s9 decal albedo/normal/property.
- **Flags:** decals `0x1`, damage blend `0x2`, dirt `0x4`, palette `0x8`, soft tint `0x10`,
  layered `0x20`, overlay `0x40`, no backface culling `0x80`, alpha blend `0x100`, alpha test
  `0x200`, deform `0x1000`, skinned `0x2000`. Only `0x20/0x40/0x1000/0x2000` change the
  vertex buffers.
- **Channels (inferred from vanilla data, not shader code):** gloss/metallic/clear-coat are
  4-vectors, one component per material channel. The PropertyMap's RGB selects channels 1–3
  (black = channel 0); the TintMap's RGB is the amount of tint colour R/G/B. The stock
  Serpente: TintMap R only (one-colour paint), PropertyMap R = metallic clear-coated paint,
  G = plastic, B = chrome.

A source engine with a "separation mask selects material 0–3, each with colour/roughness/
reflectivity" model (Batman: Arkham Knight, Unreal Engine 3, here) maps onto this almost one to one. Do it
as a table, not by eye: resolve each source material's named parameters along its parent
chain, give **every** source factor a disposition (mapped / approximated / no equivalent /
engine-global / missing on disk) and set **every** CarPaintMM input by a rule with a source
tag; then decode the written blocks and assert each input equals the spec. Template values
must not survive unannotated — inherited donor constants were behind every visual bug here.

## Importing a skeletal mesh (Unreal `.pskx`)

- The chunked `ACTRHEAD` file holds far more than points: `PNTS0000`, `VTXW0000` (u32 point
  index when >65,535 points), `FACE3200` (18-byte faces), `MATT0000`, `REFSKELT` (120-byte
  bones), `RAWWEIGHTS` (weight, point, bone), `VERTEXCOLOR`, `EXTRAUVS0`. Walk chunk headers
  by size; a scan for "4-letter ids" misses most of them.
- **Units:** this asset is real centimetres — its tyres measure Ø1.37 m against a 1.40 m
  reference. Import at ×0.01 and treat the physics as the thing that moves (see above).
- **Handedness:** the PSK→JC3 map must be a **proper rotation**,
  `(x, y, z) → (−y, z, −x)` for X-forward/Y-side/Z-up sources. A determinant −1 map looks
  fine on a symmetric car and silently mirrors it. Check against the rig's own Left/Right
  bone labels (56 of 56 agreed after the fix, 0 before), and set triangle winding so the
  signed volume is **positive**, as on vanilla meshes.
- **Parts from the skeleton, not name lists:** give each face its dominant bone from the
  weights, classify bones by shape (round in the axle plane, full angular coverage =
  spinning), take the `number_of_wheels` largest round parts as wheels, attach smaller
  round parts only inside a wheel's radius, and move faces outside the tyre's circle back
  to the body. Wheel meshes are built local to their hardpoint.
- **Materials:** `umodel`'s `.mat` role map is a guess and was wrong (it labelled a normal
  map "Diffuse"); read the `.props.txt` dumps. Vertex colours here were greyscale (not a
  mask) and the second UV set was garbage on the main body.

**Movement units, from the game's own numbers.** `vehicle_misc.official_top_speed` is km/h
(equals the stat screen); `transmission.top_speed` is the km/h physics cap (just under the
gearing's top-gear speed at the upshift rpm, given wheel radius and final drive);
`land_engine.torque_factor_at_optimal_rpm` is peak torque in N·m (490 N·m → ~422 hp with
the curve's falloff); `custom_land_global.extra_top_speed` tiers are km/h of boost;
`land_aerodynamics.frontal_area` is m². Bigger wheels change gearing: re-derive the final
drive or the drivetrain limit moves.

**Registering a genuinely new vehicle** needs no patch to any DLL. The public console mod
(`jc3-console-thingy`) hooks the game's property-file reader, lifts one property out of
every property file the game loads, and feeds those strings to the spawn system. Those
properties live in `settings/spawn_*.bin` — RTPC containers in the VFS `settings/` folder,
so they are dropzone-overridable like any `.ee`. Vanilla ships `spawn_vehicle_defs.bin`.
**Replacing an existing vehicle needs no registration at all**, because the vanilla spawn
table already lists it.

## Build steps

1. Unpack the target `.ee` from `archives_win64` (Gibbed `SmallUnpack`) or copy a pristine
   unpacked tree aside. **Never edit the only copy.**
2. Write a codec that decodes and re-encodes the block types byte-exactly, and prove it:
   a round-trip over the whole donor tree must be byte-identical. Mine reached 190 of 193
   files; the 3 exceptions were a different block type in an upgrade part.
3. Solve any unknown block layout from the files themselves. Two oracles: the block must
   land exactly on its `0x89ABCDEF` checksum, and the decoded float3 positions must
   reproduce the bounding box the engine itself stored in the file header. Validate the
   solver against a *known* layout first and require zero bbox error.
4. Author meshes in one pass with one transform: read the PSKX (geometry, skeleton,
   weights), derive parts from the skeleton, compute the wheel stations from the asset,
   translate every material into a block spec, convert the textures within the game's
   budget, decimate each material separately in Blender (one block per material: CarPaintMM,
   or Window for glass), and write the physics stations from the same numbers. Verify per
   block (byte re-encode, counts, indices, positions, normal round trip) and against the
   spec. Zero-area triangles left by decimation give zero normals that can't be packed —
   give those vertices any unit normal.
5. Render orthographic previews offline (side/top/front, z-buffered) and iterate there.
   This is the single highest-leverage thing in the whole workflow: it removes every
   game launch from the loop except the final confirmation.
6. Repack with `SmallPack`, unpack the result again and diff against the pristine tree.
   Expect *only* the files you meant to change. Then regenerate the `.toc` from the
   packed `.ee` (never reuse the vanilla one after a size change) and diff `.ee` vs
   `.toc`: only the by-design offset-0 externals may differ.
7. Deploy with a script that asserts the artifact's size **and** sha256 before copying
   (both `.ee` and `.toc`), and that offers a one-command rollback to a known-good build.

## Editing the entity property file

Now a working route, thanks to Gotcha 1, and it costs one round trip:

1. Unpack the vanilla `.epe`, convert it to XML with the stock property converter.
2. Edit the XML — the parts list, the entity's root `name`, anything with a name.
3. Convert back to binary.
4. **Repair against the original**: diff the result byte-by-byte against the untouched
   vanilla file, and wherever the original held `0x50` and the writer emitted `0x00`, put
   `0x50` back.
5. Repack, unpack again, and confirm the entity is the *only* changed file.

On a stock vehicle `.epe` (109,526 B) step 4 took the difference from 2,293 bytes to 104,
and the vehicle loaded and drove. The residual is float32 last-digit rounding and ~45
bytes of substituted defaults, both tolerated.

Structural authoring works too: carving the 40-part tree down to body + 4 wheels (front
wheels reparented from under the engine to mirror the shipped mod's flat tree) loads with
the 35 other parts gone. Two caveats: repair pads by *string context* once the structure
differs (no same-shape baseline), and the importer regenerates object IDs, so exact-XML
equality cannot hold for structural edits — semantic equality (same parts, names,
hashes, indices, re-exported) is the check.

Two lessons from this specific sequence:

- **Diff against the original, never against the first generation of your own lossy
  representation.** Comparing conversion output to conversion input proves nothing.
- **Check what the bytes sit next to before naming them.** I called the `0x50` runs
  `"PPPP"` sentinels from a hexdump glance. Context said otherwise — they follow
  null-terminated strings, which is what told me they were padding, and padding is
  *repairable* in a way a sentinel is not. The misnaming is what made the whole route
  look closed.

## Verification

- **Offline:** byte-exact round-trip over the donor tree; `.ee`-vs-`.toc` compare with
  only offset-0 externals differing; archive diff confirming exactly the intended files;
  importer spec matrix reproducing shipped meshes byte-identically.
- **In game:** the car spawns, drives, steers and brakes; a full custom body (five-file
  LOD chain, tens of thousands of new triangles) loads and drives in repainted black on
  custom wheels. True scale with moved physics: wheels sit in their arches, no jitter,
  steering correct. Glass in a Window block inside the body file renders tinted.
  Per-material blocks with 20 converted textures at new paths render (once under budget).
- **Offline, against the spec:** a checker decodes every block of every output mesh and
  compares each static/support/dynamic input, flag and texture slot with the translation
  spec (185 blocks, 0 mismatches, 0 donor values inherited).
- **Attribution:** minidump crashes are read by faulting module + offset (a stdlib-only
  reader is enough: exception stream + module list). Known signatures: repoint-at-stale-
  file `READ 0x8 @ JustCause3.exe+0x2d49e0`; stale-`.toc` garbage `table[garbage] @
  +0x197194a`; repeated single-offset `nvwgf2umx.dll` faults are the GPU driver, not the
  mod (hybrid-graphics laptop; the offset repeats across unrelated builds).
- **Not verified:** authoring Havok collision from scratch (no public tool writes
  `.physicsc`; not needed so far); the CarPaintMM channel semantics in game (inferred from
  vanilla data); the fully translated material build (offline only); lore-derived
  movement values (no in-game timing run).
- **Not covered by this note:** JC3MP.

## Gotchas

1. **Symptom:** re-encoding an `.epe` through the Gibbed property converter crashes the game
   instantly, with no other change. **Cause:** the writer, not the XML. Gibbed's RTPC writer
   (`PropertyContainerFile.Serialize`) aligns values by moving the stream position, so the
   **alignment gaps, such as the bytes right after null-terminated strings**, come out as
   `0x00`, while the game's own files fill them with `0x50`. In a stock vehicle `.epe` this
   is ~2,189 bytes — it is the bulk of the damage, and the first one sits right after the
   root class string, which is why the whole entity dies. **Fix:** the converter *is*
   usable; diff the output against the original and restore every byte where the original
   held `0x50` and the writer wrote `0x00`. That drops the difference from 2,293 bytes to
   104, and the result loads and drives normally. (Making the writer fill alignment gaps
   with `0x50` would remove the need for the repair; this note used the repair.) The
   residual 104 are last-digit float32 rounding plus ~45 bytes where the writer substitutes
   defaults (`0x00` → `0xFF`/`0xFE`/`0x01`/`0x02`); those are tolerated by the game. My
   first diagnosis here was wrong — I assumed the lost bytes were `"PPPP"` sentinels and
   dismissed the remainder as float noise measured at misaligned offsets. Measuring
   properly showed neither.

2. **Symptom:** deleting a file from the unpacked tree and repacking yields a 0-byte `.ee`
   and a `FileNotFoundException`. **Cause:** the packer treats a generated `@files.xml` as
   its manifest and refuses when a listed file is absent. **Fix:** remove the file *and* its
   `<file name=...>` line. Same trap in reverse: new files must be registered or they are
   silently dropped.

3. **Symptom:** every edit that changed vertex or index counts crashed. **Cause (corrected
   2026-10-11):** none of those builds had a matching `.toc` — the concept did not exist
   yet — so the game read shifted offsets as garbage. It was never engine validation:
   with regenerated `.toc`s, a 24,125-vertex single block and a full five-file LOD chain
   both load and drive. **Fix:** regenerate the `.toc` for every size-changing build and
   `compare` it before launching. The original misdiagnosis (a bisection "proving" count
   rejection, plus four unconfirmed hypotheses) is preserved in the project journal as a
   warning: a clean deterministic table and an uncontrolled variable are easy to conflate.

4. **Symptom:** removing `sportsmuscle.physicsc` from an otherwise unmodified vehicle crashes
   it. **Cause:** partly load-bearing physics, partly stale `.toc` — that removal test also
   ran without a matching `.toc`, so the two causes were never separated. What is solid:
   the body's collision is a Havok `hknpExternMeshShape` pointing at the render mesh, and
   the file is a Havok 2014.1 binary (read class names from `__classnames__`, don't guess
   formats). **Fix:** keep it until a removal *with* a matching `.toc` says otherwise.
   Deform-data removal with a correct `.toc` still crashed, but that build carried a second
   live variable, so deform's necessity is still open.

5. **Symptom:** planning a pipeline around JC Model Renderer. **Cause:** the two branches
   differ. The last release, v0.12.3 (tagged on `master`, Nov 2019), reads, renders and
   re-writes vehicle blocks (CarPaintMM plain and deform, CarLight, Window, GeneralJC3 and
   more) and exports OBJ, but its import into those blocks is an empty stub (`SetData` does
   nothing), so it can't put new geometry into a vehicle. The `develop` rewrite only has
   three JC3 block types (Character, CharacterSkin, GeneralMkIII). **Fix:** use v0.12.3 to
   inspect and export vehicle RBMs. For vehicle authoring look at `Brooen/RBM-Exporter`,
   which targets JC3 Carpaint/Window/Carlight. Note its own caveat: no deform support — it
   writes plain blocks. (A "custom" moped `.ee` found in a modpack looked like evidence that
   plain blocks work in a shipped mod; diffed against the vanilla moped from `game35.tab` it
   is 413/415 files byte-identical, RBMs and `.physicsc` included, and only two textures
   differ. Diff a mod against its vanilla base before treating it as a geometry precedent.)

6. **Symptom:** you cannot find the property the console mod spawner reads, no matter how
   many names you hash. **Cause:** that property has **no name** — it is an id-only field,
   visible in converted output as a value tagged with a raw id rather than a name. The id is
   `0xF71C2A21`, the hash jc3-console-thingy's `src/commands/spawn.h` looks up. **Fix:**
   scan binaries for the little-endian id bytes instead of hashing candidate names. I found
   56 hits in a shipped mod's `settings/spawn_encounter_defs.bin` in one pass.

7. **Symptom:** a crash that looks like your mod's fault but may not be. **Cause:** on hybrid
   graphics the fault can be in the GPU driver. **Fix:** read the faulting *module and
   offset* from the minidump (exception stream + module list — a ~60-line stdlib script
   does it), not just the application log, and compare against known signatures before
   blaming a build. Repeated single-offset `nvwgf2umx.dll` faults across unrelated builds
   are environmental. An empty dropzone launching fine while modded builds crash points at
   the mod, but only the faulting module tells a mod crash from a driver crash, and only
   the offset tells one mod crash from another.

8. **Symptom:** a round-trip test passes but the file is still lossy. **Cause:** I compared
   the second generation against the first generation of the *same* lossy representation,
   which can only prove the loss is idempotent. **Fix:** compare against the **original
   bytes**. A byte-diff I had already run and explained away as "reordering" was the whole
   answer — a run of identical dwords was the tell.

9. **Symptom:** displacing a car body tears the arches away from the wheels. **Cause:** the
   wheels are parented to the chassis, not the body mesh, so the arches must not move while
   the body around them does. **Fix:** protect a disc around each measured hardpoint and
   assert those vertices are byte-identical afterwards. That oracle caught a real 46 mm leak
   into the front arch before it ever shipped.

10. **Symptom:** reshaping one part looks fine but the car comes apart. **Cause:** the parts
    are separate files sharing one coordinate frame; scaling or normalising each part against
    its own bounding box warps them by different amounts. **Fix:** compute one shared
    reference frame across every part you touch and apply the same function to all of them.
    (The importer now enforces this structurally: one fit frame per spec.)

11. **Symptom:** `ConvertAdf` throws on `.epe`. **Cause:** it is a property container, not ADF.
    **Fix:** use the property converter for reading. (For writing, see Gotcha 1.)

12. **Symptom:** a validation script reports thousands of violations in files you know are
    fine — including in blocks you never touched. **Cause:** the checker, not the data. I
    read four int32s at offset +12 of a stride-24 deform vertex as bone indices, and got
    4,249 "out-of-palette" hits on a block I had edited *and* 4,223 on an untouched one.
    Run the checker against **pristine** files: it flagged 4,057 on an unedited
    `body_lod1.rbm`, which is what proved the probe wrong. **Fix:** a checker that fires on
    known-good input is telling you about itself. Unedited blocks sitting in the same file
    are a free control — use them before believing a violation report. (The deform vertex
    layout is now measured — see How the game works — but treat any new probe the same way.)

13. **Symptom:** an internalized file changes nothing (retune ignored) or a working repoint
    crashes. **Cause:** the `.ee` carries a **stale, pre-patch** copy while `.toc` offset 0
    means "read the current patched file from the archives". The Serpente's parts file is
    29,886 B in the `.ee` vs 30,540 B patched; deform 67,214 vs 67,278; `land_global`
    1,494 vs 1,600. Internalizing the stale copy reads old data out of step with the
    patched vehicle. **Fix:** always source the newest copy: scan the patch `.tab`s for the
    file's name-hash (the `.tab` is a flat hash/offset/size index; the hashed string is the
    file's path *inside* the `.ee` — `models/...` for model files, `editor/...` for entity
    files — verified both ways) and extract it with the offset/size before editing.

14. **Symptom:** a brand-new mesh file never renders, with a correct `.toc` record, while
    the same bytes at a vanilla path draw. **Cause:** the game does not load an `.rbm` at a
    path the vanilla entity never had. New `.lod` files pointing at already-listed meshes
    are fine. **Fix:** ship meshes at vanilla `.rbm` paths (every part owns up to 5 LOD
    slots); use new `.lod`s only to re-aim at listed meshes.

15. **Symptom:** wheels shake (and the nearby flank with them) from the first frame after
    moving hardpoints. **Cause:** the Havok wheel RAY stations are baked into `.physicsc`
    as plain `(x,y,z,1.0)` quads, while visuals follow `land_global`; splitting them
    (0.39 m at the front axle, 0.09 m at the rear) shakes exactly the split axle. An
    earlier "phasing" result after patching only the quads had the same root: the body
    had been fitted separately from the wheels. **Fix:** derive body, wheels, hardpoints,
    RAY quads, radius and width from **one** transform and one stations file; then the
    quad patch works (Havok accepts it). Never fit parts separately.

16. **Symptom:** white racing stripes appear on a new body. **Cause:** the template block's
    decal layer (`0x1` and `SupportDecals`) samples the second UV set, and the importer
    had copied UV2 from the nearest donor vertex — projecting the donor's livery onto the
    new shape. **Fix:** set the decal switches from the source material (off here).

17. **Symptom:** tan mud on tyres whose textures are clean (mean RGB 31). **Cause:** the
    dirt layer (`0x4`, `SupportDirt`, `DirtColor`) inherited from the template. **Fix:** as
    above — every switch and constant from the source material, never from the template.

18. **Symptom:** the whole vehicle is invisible and you drive through where it should be;
    no crash. **Cause:** the texture (or model) memory budget's hard cap. **Fix:** size
    resident textures to the budget (see "Memory budgets") or ship a raised budget file,
    which changes every vehicle.

19. **Symptom:** speed edits do nothing sensible (top speed stuck far below the target).
    **Cause:** the edited engine module was the `.ee`'s stale pre-patch copy (it also lacked
    a member the patch added, `overdrive_extra_rpm`), internalized over the live one.
    **Fix:** Gotcha 13 applies to every module — list each module's `.ee` copy against the
    archive copies by name hash before editing.

20. **Symptom:** a converted mesh looks right but some details are on the wrong side.
    **Cause:** an axis mapping with determinant −1 mirrors the model. **Fix:** see
    "Importing a skeletal mesh" — proper rotation, checked against bone labels, winding by
    signed volume.

## Assets

The source vehicle model was a third-party `.pskx` skeletal mesh (chunked `ACTRHEAD`
container, 201k points, 300k faces, 19 materials, 180 bones) with UE material dumps and 56
`.tga` textures. An earlier version of this note called its unit scale "unknowable" and
fitted it to the donor's wheelbase; that was wrong — it is real centimetres (tyre Ø1.37 m
vs a 1.40 m reference), and fitting it to the donor is what forced the physics/visual split
in Gotcha 15. Import at true scale and move the physics with it. It was exported with
`umodel` from the author's own installed copy of Batman: Arkham Knight
(`BmGame/CookedPCConsole/Playable_Batmobile_SF.upk`, Ver 863/227, Game 800016). It is
another game's extracted asset, so builds that contain it stay private: what can be shared
is the importer, its specs and the method, not the converted `.ee` or textures.

All geometry, material and texture work is one pass driven by the asset's own data
(skeleton, weights, material parameters, texture files), plus an offline orthographic
previewer, so no external modeller is needed.

## Cost and time

Roughly five dozen in-game launches across fifty-plus builds, most of them controlled
experiments that each closed off one hypothesis. The offline preview, the byte-exact
round-trip harness, and the `.ee`-vs-`.toc` compare were worth far more than their
cost — most failures were caught without launching the game at all. The most expensive
mistakes were the two premature verdicts (Gotchas 1 and 3): each cost days, and each
dissolved under one re-measurement.

## Open questions

- **Authoring Havok collision.** I know of no public tool that writes `.physicsc`.
  ApexMax is a weak lead at best: its README calls it an "Apex Engine importer for 3ds
  Max" (archived 2019). HavokMax is archived too, and HavokLib's supported classes are
  skeletons and animations, not hknp shapes, so neither looks able to produce one.
  (`ApexLib` and `ApexToolset` are both archived and contain no `.epe` or physics
  tooling: archives and textures only, and material/property serialisation.)
- **Undecoded fields.** CarPaintMM's `unknown` float (1.0) and 16-byte `material_params`
  (0.235, 65536, 65536, 0), and the Window block's 41 + 16 static bytes (so glass opacity /
  reflectivity / roughness can't be set from the source yet).
- **No CarPaintMM equivalents** for tiling micro-detail normals (could be baked into the
  main normal map), anisotropic ramps and colour ramps; emissive needs CarLight blocks.
- **Channel semantics** of PropertyMap/TintMap are inferred from vanilla data; one in-game
  test with a known two-channel mask would confirm them.
- **High-mip import at vanilla paths.** The tim42 fork's `write-to-hmddsc="True"` import
  threw (see Textures), so no `.hmddsc` was written; new paths with one full-chain element
  work without one. ApexToolset's `ddscConvert`, which writes `.hmddsc`, is the untested
  alternative.
- **Reference for the custom-vehicle pipeline:** *Meme Vehicle Pack*
  (https://videogamemods.com/vgm/justcause3/mods/meme-vehicle-pack) by Luke JC
  (https://videogamemods.com/u/lukejc3mp-0021), which ships new models such as a Shrek
  plane — diff every such mod against its vanilla base (file by file, with hashes)
  before treating anything in it as precedent.

## Follow-ups: the old bisection questions, answered from the record

Three questions outlived the original verdict; all three resolve without new launches:

1. **Which per-vertex buffers did test-019's duplicate vertex go into?** Both. The
   bisect script extends `vb0` *and* `vb1` (one stride appended to each, both counts
   +1), and block 0's flags (`0x1004`) set no third-buffer bits — so the file carried
   no disagreeing buffer. A position-only edit would have crashed on its own terms;
   this one didn't, and its crash is now attributed like the rest (stale `.toc`).
2. **Which module faulted in test-018/019?** Inside `JustCause3.exe` itself
   (`+0x1ec3e0` / `+0x1ebb1ec`, access violations read from the minidumps at the
   time) — distinct from each other, from the repoint fault (`+0x2d49e0`), and from
   the driver-fault cluster. Not the NVIDIA driver, so no discrete-GPU re-run would
   have changed anything. Caveat: Windows has since rotated those `.dmp` files out,
   so this rests on the contemporary reading, and both runs lacked matching `.toc`s —
   neither can implicate counts either way.
3. **Diff of a working custom vehicle vs its vanilla base.** The Meme Pack's DaBaby
   jeep vs the stock oldjeep, file by file:

   | | vanilla base | DaBaby |
   |---|---|---|
   | body mesh | 3 CarPaintMM blocks, flags `0x100E` (DEFORM) | 1 block, flags `0x80`, stride 12, 224 verts / 444 tris, car-sized bbox |
   | `.epe` | full part tree | 122,790 B, 5 parts (body idx 0 + 4 wheels, vanilla names) |
   | parts file | civilian table | NEW `dababy_vehicle_parts.vmodc` (body + 4×(wheel, modelswap) hash swaps, same size) + repointed civilian copy |
   | `.physicsc` | 678,576 B extern-mesh structure | untouched (same file in `game36`) |
   | `.toc` | — | 704 records; every changed/added file at a real offset, shared files at 0 |

   A repeat of test-018/019 on an exporter-style plain block *with a correct `.toc`*
   has not been run. It matters less now: a 24,125-vertex single block and a full
   five-file LOD chain already load toc-correct. The isolated single-count datapoint
   would cost two builds and two launches.


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

