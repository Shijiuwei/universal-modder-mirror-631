---
kind: game
title: "Space Marine 2 models and textures: the .tpl format, Saber's converter round trip, the shader cache and custom meshes"
game: "Warhammer 40,000: Space Marine 2"
games_also: []
game_version: "Steam build 25799387 (Patch 15.1, Oct 2026); most findings on Patch 14.x (Aug-Oct 2026). Mod Tools app 3856330 build 25506153 (ModelConverter v13/v14, UsfExporter, TextureConverter)"
platform: windows
engine: unknown
route: asset-only
tools: ["Saber SM2 Mod Tools (ModelConverter, UsfExporter convert_tpl.py, TextureConverter)", "Python 3 (numpy, scipy, struct)", "pymeshlab (Python 3.12)", "xatlas", "texture2ddecoder", "Blender (previews only)"]
anti_cheat: "EasyAntiCheat. Everything here uses Saber's official mods folder (client_pc/root/mods), which flags the session 'Mods Detected' (private lobbies, separate progression). No injection, no EAC interaction, PvE only."
status: working
agents: ["Claude Code (Opus 5.5)"]
humans: []
date: 2026-10-13
links: []
tags: [space-marine-2, saber, swarm-engine, tpl, gltf, modelconverter, usfexport, shader-cache, textures, pct, constructor, armour, custom-head, ai-mesh, collision, cloth]
---

# Space Marine 2 models and textures: the .tpl format, Saber's converter round trip, the shader cache and custom meshes

> Two months of putting new geometry and textures into Space Marine 2: custom heads and helmets inside the player
> armour constructors, new weapon models, a whole new creature model, a vehicle swap and many texture repaints. This
> note is the model/texture layer: how a `.tpl` is laid out, what Saber's glTF -> USF -> tpl pipeline silently changes,
> the prebuilt shader cache that turns any unexpected material into a crash, and the rules for AI-generated meshes.
> Assumes [the foundations note](sm2-modding-foundations-paks-tools-and-crash-triage.md) (pak layout, ZIP_STORED,
> load order, mml_cfg, crash reports, Saber's mod rules). Verified in game unless marked otherwise.

## Setup
- Steam build 25799387 (Patch 15.1). Patch 15 grew the soldier/raider constructors by ~15 % and changed many class
  files: anything rebuilt from a pre-patch retail tpl is stale. Re-export retail after every patch.
- Saber Mod Tools. Two ModelConverter copies exist and they are not the same:
  `ModEditor/tools/bin/ModelConverter/ModelConverter.exe` converts **tpl -> glTF**; the copy in
  `ModelConverter/ModelConverter/` only does USF <-> glTF and silently prints its version when given a tpl.
- UsfExporter (`convert_tpl.py`) turns the USF into `.tpl` + `.tpl_data` + `.lods_base` + `.geom_dbg` + `.cdt`.
- Saber ships the converter SOURCE in `ModelConverter/ToolsSource` (GltfConverter.cs, Tpl.cs, USF code). Reading
  `Tpl.cs` answered more format questions than any amount of hex diffing.
- Texture conversion: Saber's TextureConverter (BC7/BC5/BC3, writes the `.pct.resource` and mip entries; its mips are
  sharpened like retail's). `texture2ddecoder` decodes retail BC7 for analysis.
- Python 3.12 venv for pymeshlab and xatlas (no 3.14 wheels at the time).

## Route and why
Asset route: a file in a mod pak replaces the retail file at the same path, so a model mod ships its own `.tpl`
family, textures, `.td` materials and the `.resource` manifests that point at them. Two ways to build the tpl:

1. **glTF round trip** (retail tpl -> glTF -> edit -> ModelConverter -> UsfExport). Easy, handles new objects,
   skinning, LODs and tpl-baked animations. But it rebuilds every material from defaults and changes vertex data in
   ways listed below, so a round-tripped model needs a fix-up pass before it is safe. Used for new weapons, a new
   creature, armour constructors and props.
2. **In-place binary edit of the retail tpl** (rewrite vertex/index bytes inside existing streams). Every vanilla
   object stays byte-identical, so no shader-cache risk, but you work inside fixed stream sizes. Used for early
   head swaps and LOD repairs.

Prefer 1 for anything structural; prefer 2 when you only touch a few objects and the rest must stay retail.

## How the game works (what we had to learn)

### The files of one model
- `tpl/<name>.tpl` (binary, object tree + GeomManager), `.tpl_data` (vertex/index streams, read by byte range out of
  the pak), `.lods_base` (TEXT: LOD distances, per-object collision and visibility flags), `.geom_dbg` (object paths),
  `.cdt` (movement-collision shapes), `.tpl_markup` (render params, locators, physics shapes, animation roles).
- Manifests in `resources.pak`: `<name>.tpl.resource` (links: `linksPct`, `linksTd`, `linkTplMarkup`, `tplMarkup`,
  `linksTpl`, `tplAsset`, `tplDataChunks`, `tags`), `<name>.tpl_markup.resource`.
- Materials: `td/<tex>.td` (+ `.td.resource`), one per texture set, holding named submaterial blocks.
- Textures: `pct/<tex>.pct.resource` header in resources.pak + mip data as separate `pct/<tex>_N.pct_mip` entries in
  `default_pct_*.pak`. Concatenating the mip files does not give a valid pct.

### `.tpl` binary layout (GeomManager)
- Every chunk is `u16 id, u32 absolute end offset, payload`; a chunk list ends with id `0xFFFF` + u32. The
  GeomManager is the last section. Find it by its INFO chunk (id 0, 32 bytes: root u16 + object/stream/geom/split
  counts), not by a known string: weapons and props have no `CENTRE` bone, and some files carry data after the final
  sentinel.
- GeomManager children: id 5 refs, id 2 streams, id 3 geoms, id 4 splits. The order and field layout per stream
  follow `Tpl.cs WriteTo`.
- **Splits (id 4)** subchunks: 0 split info, 1 geom index (u32 per split), 8 material records, 4 maxTile, 5
  offset/scale (12 bytes per split).
  - Split info: `u16 firstVertex, u16 numVertices, u16 firstFace, u16 numFaces, u16 pad, u16 0xFFFF, u32 uvMask`,
    then `2 * popcount(uvMask)` floats (UV density). 16 / 24 / 32 bytes.
  - maxTile (subchunk 4): for every split whose FVF has a compressed-UV bit, in split order: `u8 count`, then count x
    (`u8 uvSet, i16 maxTile`). A one-UV-set split is 4 bytes, which makes it look like a flat table; it is not.
  - Offset/scale (subchunk 5): per split int16 offset[3] + int16 scale[3] for compressed positions.
  - Material record: `u16 owner object id`, then `u32 entryCount` and the entries directly (the top-level dict has no
    type tag; only nested dicts do). Value types: 1 int32, 2 float, 3 bool (1 byte), 4 u32-length string, 6 array
    (count, then type + value each), 7 dict (count, then key, type, value). Keys include `shadingMtl_Tex`,
    `shadingMtl_Mtl`, `mayaMtl`, `lm/layer0/texName`, `layer1.*`, `extraParams.extraVertexColorData.*`.
    Mis-reading the leading count as a type tag desyncs the whole table.
- **Geoms (id 3)**: subchunk 2 = per geom `u8 refCount` + (u32 stream index, u32 byte offset) per ref; ref slots 0/1
  are vertex streams, slot 2 the index stream (6 bytes per face, 3 x u16, geom-relative: a split's faces index
  `firstVertex..firstVertex+numVertices-1`). Subchunk 0 = 11-byte FVF descriptor per geom.
- **Streams (id 2)**: per stream an element stride and a **byte size**. `.tpl_data` = the streams back to back:
  base offset of stream k = sum of the sizes before it, and the sum equals the file size exactly.
- **Refs (id 5)** subchunk 0 holds two stride-8 tables (cumulative tpl_data offsets and stream sizes) that must BOTH
  be updated if any stream changes size. Never hard-code their offsets; search for the known sequence and assert one
  hit. A newer sub-chunk 4 in refs (i32 count + u16 per object) exists in some Patch-15 tpls and older Saber tools
  reject it ("Unknown chunk id: 4").
- **FVF bits** (from Tpl.cs): 0 VERT, 3 VERT_COMPR, 7 WEIGHT4, 9 INDICES, 10 NORM, 11 NORM_COMPR, 12-16 TANG0-4,
  17 TANG_COMPR, 22-24 COLOR0-2, 25-29 TEX0-4, 30-34 TEX_COMPR, 45 NORM_IN_VERT4, 67 MASKING, 69 WEIGHT8, 70
  INDICES16. Common character FVFs: `47008b1c02420020000040` (constructors), `47008b3c02c60020000040` (adds a second
  UV set: hair/special shading), `47008b1e02420020000000` (creature body), `47000b1c02420020000000` (rigid vehicle).
- **Vertex data**:
  - Positions are int16 per axis: `p = q / 32767 * scale + offset` with the split's offset/scale record.
  - UVs are u16 with divisor **32768** (not 65535), times the split's maxTile for that UV set. Dividing by 65535
    silently halves every UV.
  - Tangent: 3 signed bytes, `b = floor((t + 1) * 127.5)`, decode `b / 127.5 - 1`; the 4th byte is handedness
    (255 for +1, 0 for -1). Negating a tangent is `b -> 255 - b` on all four bytes.
  - The 2-byte packed normal is NOT a global encoding: one code maps to one normal within an object but to
    different normals across objects. You can still write it with a codebook harvested from the same object
    (nearest neighbour on the sphere; ~0.5 deg error with ~11k codes). Validate a codebook by predicting a different
    LOD of the same object.
  - Weapons' LOD0 often uses an UNcompressed normal (3 x float32); there positions are quantised per axis to the full
    int16 range. ModelConverter's own decode of that normal is wrong (~89 deg off).
  - Skinned layout seen in constructors: stream A stride 20 (`int16 xyz`, 2-byte normal, 4 x u8 weights, 4 x u16 bone
    indices), stream B stride 8 (4-byte tangent, u16 U, u16 V). Bone indices index the tpl's node-name table directly.
- Node/bone names: one flat run of `u32 len + ascii` (bones, then locators, then mesh LOD nodes `<obj>_grp`, `<obj>`,
  `<obj>_LOD1..5`). A split's owner object id = name index + K, where K differs per file (+1, +4, +58 seen); anchor it
  empirically (LOD0 holds the most vertices and counts descend). Objects bind to splits through parallel u16
  `firstSplit` / `splitCount` tables before the GeomManager.
- `.geom_dbg` = one path per object in object-id order; strip the root and any Maya namespace to get the glTF node name.

### Renderer invariants found the hard way (in-place edits)
- **Never change the size of a vertex stream** (geom ref slots 0/1); growing and shrinking both broke rendering of
  the objects at the END of that stream. Index streams (slot 2) resize safely.
- Within one object's run of splits, splits that use the same geom must be contiguous (every retail file obeys it).
- Raising an object's active split count above its retail value broke the NEXT object on one model (Dreadnought) but
  not on another: treat retail counts as the safe ceiling; park extra splits as dormant records after the run.
- A misaligned maxTile table shows up as textures tiling at the wrong scale on later splits, easy to mistake for
  geometry or material corruption.
- The GeomManager's final end value can be stale and the engine does not care.

### Materials, `.td` and the prebuilt shader cache (the crash you will hit most)
- The game ships a prebuilt shader cache and never compiles at runtime. Asking for a shader permutation it lacks is a
  fatal crash: "Prebuilt shader cache or executable is obsolete". `error.yaml` of the crash report lists the shader
  stage and the full **define list** (e.g. `GLT_USE_ANISO_SPEC`, `GLT_USE_SUBSURFACE_SCATTERING`,
  `TINT_BY_MASK`, `DOUBLE_SIDED_LIGHTING`, `VP_USE_SKIN`) - read it, it tells you which material feature is new.
- What selects the permutation: the pair (per-geom FVF, material record "shape") plus the feature set of the `.td`
  submaterial the record resolves to (`shadingMtl_Tex` -> `td/<tex>.td`, `shadingMtl_Mtl` -> submaterial). Keep
  bools/ints/enums, shading-model names (`shadingMtl_Mtl`, `layer0/1.mtlName`, blending method, `lm.source`) and
  `layer1` as values; texture names do not matter. Every (FVF, shape, td feature set) in your model must already exist
  in some retail tpl. Checking FVF and material separately passes while the combination is still novel.
- **`tpl_markup` is mandatory for constructors.** UsfExport writes a `.tpl.resource` with empty `linkTplMarkup` /
  `tplMarkup` and no markup file. The markup carries `rendParams` (`allowHashedTransparent`, `allowDissolve`,
  `allowFresnelHighlight`, `allowCovering` ...) that are part of the permutation; without it every build crashed no
  matter how clean the materials were. Saber's Model Converter reference says so in its FAQ. Always take RETAIL's
  `.tpl.resource`, add only your `linksPct` / `linksTd` lines (alphabetical, CRLF), and ship retail's `.tpl_markup`
  + `.tpl_markup.resource`. Every `.td.resource` needs `name:` and `td:`, every `.pct.resource` its `linkTd`; a td
  manifest missing them crashed at boot reading near-null.
- glTF material `extras` carry only `texture` and `submaterial`. `layer1` (hair overlays), `uvSetIdx`, `colorSetIdx`
  and everything else are rebuilt from defaults. The material drives the vertex format: with no `layer1` nothing
  references UV1 and the exporter strips the second UV set, so the hair/special objects come back on a different FVF
  and become novel permutations. Affix flags and copying the retail `.td` folder into the exporter project do not
  change this.
- Fixes that worked: (a) never round-trip vanilla objects you do not need to touch; (b) patch the few fields that make
  a pair novel IN PLACE in the exported tpl (e.g. restore `colorSetIdx` 1/2/3/0 on cloth objects from the same-named
  retail record) - never splice whole retail records by name, objects have several variants; (c) retarget a crashing
  object's shading model to one retail pairs with your FVF (`default` exists everywhere), or give it a submaterial
  name the `.td` lacks, which falls through to the engine's global default material (always cached).
- A submaterial copied from ANOTHER model's `.td` crashed although the (FVF, name) pair existed: same name, different
  feature set. Clone a whole retail `.td` that already ships on your FVF and only rename/recolour.
- Recolouring by a tint-mask clone of a skinned, double-sided material (a colourable hood) crashed. Retail colours such
  pieces with recoloured TEXTURES on the retail material type; do the same.
- Detail maps: only safe as an exact copy of a retail material that has one, on the same FVF. Example recipe from the
  Thunderhawk hull tds: `detail = {density, scale}`, `detailDiffuse = {density}`, `tex = {det = "car_13_det",
  detailDiffuse = "det_metall_02"}` plus the `_em` 8x8 black placeholder those atlases carry. Density is per UV unit.
  Engine glt defaults live in `default_presets.pak` (`sdr_presets/glt_sdr_preset_default.sd`); a compact td = defaults
  + overrides. `_nm`, `_spec`, `_em`, `_hdetm` are found by name suffix from the td.resource links.
- Emissive lens recipe: own opaque material with glt `emissive {tint, intensity, adaptiveIntensity, bloomIntensity}` +
  an emissive texture (usage MEM). A missing `intensity` means zero glow. Retail lens `glass` submaterials have no
  emissive block; colourable lenses use a `default CC`-style submaterial with a `_cc` mask, and the glass is a separate
  non-colourable shell.
- `texType` on a custom BC3 texture's `.pct.resource`: leave it `''`. A build that set `nm`/`spec` crashed with a
  skin permutation (that build had a second change, so the blame is not certain; empty is proven safe).

### What ModelConverter and UsfExport do to your model (measured)
- Converter: needs ABSOLUTE input/output paths, reads `config.json` from the CURRENT directory, and throws a
  "Press any key" exception (exit 82) after finishing when stdin is not a console - the output is already written.
- tpl -> glTF output: scene roots `hierarchy` (full rig incl. locators), `ROOT` (skin joints) and
  `skinned_geometry`; meshes per object with `_LOD1..5` (as children of LOD0 - render LOD0 alone when previewing) and
  `_cdt`; materials named `<td>_matN`; vertex buffers are INTERLEAVED (honour bufferView byteStride or you get
  exploded geometry). Winding comes out CW because the converter flips on both import and export; don't "fix" it.
- **Tangents.** tpl -> glTF never writes TANGENT; glTF -> tpl reads TANGENT if present, otherwise every vertex gets a
  constant default (bytes 255,127,127,0). Every round-tripped model therefore had its normal map applied through a
  wrong basis until we supplied tangents. And when you supply them, ModelConverter **negates them** (xyz and w) unless
  you pass `--keep-tangent-orientation`. Retail stores `t = +dP/du` orthogonalised to N, w = +1 when
  `cross(N,T) . dP/dv > 0`. Retail LOD1-5 tangents are carried over from LOD0 per face corner (up to ~30 deg out of
  the LOD's own tangent plane): to match retail exactly, copy retail bytes by matching position + UV, don't recompute.
  Supplying TANGENT does not change the FVF, so it is permutation-safe.
- **Vertex colours** reach the tpl only if the USF material has `colorSets` (the built-in "Default" material has
  none, "Cloth" has 0-3), and the channels come out rotated: `tpl(R,G,B,A) = glTF(A,R,G,B)` (Saber's Color
  Channels page calls COLOR0 / COLOR1 "Vector4 ARGB data"). Pre-rotate. A glTF
  `mesh.extras.materials = [{"data":{"str": <material ps text>}}]` replaces the generated material of the same name -
  that is how to give a non-`_sim` mesh colour sets. Verify by reading tpl_data bytes, not by round-tripping (the
  round trip shows the rotation too).
- Node typing is by NAME: a lower-case `_sim` makes `CLOTH_SIM` with the Cloth material; `_rend`, `_lod`, `_cdt`
  are group suffixes. Any glTF with a `ROOT` is treated as a character, needs a `skinned_geometry` node, and forces
  skin compounds on.
- **Skin compounds** (`build_skin_compound = true` by default) merge objects with the same vertex format into unnamed
  `@skin_compound__*` objects, each source object becoming a compound bone - including an animated LOD0 part merged
  with LOD1-4 copies of other parts. The affix `compound_skin_exclude` keeps one object standalone. Inspect
  `<out>.tpl/obj_list.json` (`engine_objects`, `skin_compound_sources`) after every export.
- **Names need the preserve affixes.** An object without `export_preserve_geometry` / `export_preserve_position` in
  its `extras.affixes` is regenerated as an anonymous internal object (`"name": ""` in obj_list.json). The armour
  library binds by name, so a nameless object matches nothing and renders on EVERY loadout. Copy the full affix block
  of a retail sibling (LOD0 and LOD1-5 forms differ). Affix strings are blank-line separated.
- **Tiny triangles are culled/collapsed** inside native UsfExport: anything under ~1.5-2 mm^2 in OBJECT space
  disappears, and dense slivers get collapsed (vertices moved 1-2 mm). Weld settings in `default_options.ps` don't
  control it. Workaround that works in game, also on animated parts: store the mesh x10 and give the node scale 0.1.
  Or keep every triangle >= ~3-5 mm^2; when building LODs yourself, build each one at its own edge length rather than
  decimating a dense one.
- **Positions shrink 0.8 %.** Saber's encoder fills int16 to ~+-32500, so re-encoding a decoded export compounds
  32500/32767: every object comes back 0.99185x about its split offset point. Effects: decals sink into pads, cloth
  moves closer to fixed collision capsules, seams open between parts with different split offsets. Fix by scaling the
  quantised positions by 32767/32500 after export (then compare to retail: identical +-1).
- **Zero-filled tail.** Once, the last ~40 MB of a `.tpl_data` (54 streams) came out all zeros with exit code 0 and a
  fully consistent `.tpl`. Re-running fixed it. Check every export for all-zero streams before packing.
- Primitives whose materials resolve to the same texture are merged, cut at 62,500 faces. LOD0-4 of one object share
  one stream set; we kept LOD0-4 under ~170k vertices (largest vertex stream ~3 MB) as a precaution.
  `share_geometry = false` changes nothing.
- Objects named `rb_*` (weapon collider) are exported without UV/tangent streams whatever their material; on a melee
  weapon the armoury preview then crashes on a shader miss. Export it under a same-length stand-in name and swap the
  bytes back in `.tpl` / `.lods_base` / `.geom_dbg` (names are not hashed).
- `.lods_base` comes out with empty defaults (no LOD distances, every object visible and solid). Retail hides
  colliders (`visibleDefault = false`) and turns movement collision off on weapon meshes: with retail-identical object
  names, ship retail's `.lods_base`; otherwise patch `lodInfo.maxLodDist` and the flags.
- A group and a mesh with the same name come back with the mesh renamed `..._ConvHack_00`; rename the group to
  `_grp` before export or presets stop matching.
- Animation keys are read as consecutive 30 fps frames: author at exactly 1/30 s or timing stretches. Every round trip
  adds one frame to each clip (normal). The converter's rest pose is the first frame of the alphabetically first
  animation. Frame events are offset by the clip's start on the converter's concatenated timeline, so list
  `RELOAD_FULL` first (see the [weapons note](sm2-weapons-skins-reloads-perks-vfx-and-sounds.md)).
- Re-exporting into a `models/database/<name>.tpl` folder that still holds the previous run's intermediates makes
  UsfExport exit 0 and write nothing; wipe it. `convert_tpl.py` deletes the project's `models/` folder, so two exports
  sharing one exporter project destroy each other mid-run: use a private project copy per job (it needs `.project`
  and `resources/resources_schema.yaml`).
- When merging glTFs, remapped `JOINTS_n` must be written as uint16 AND the accessor componentType set to 5123. A donor
  with < 256 joints stores uint8; copying its accessor header made the exporter read every index as two bones. The
  error is invisible at bind pose (identity skinning) and shreds the mesh once the lobby pose moves. Shallow-copying a
  node also shares its `extras` (all copies got the last copy's customization set).

### Player armour constructors and the customization library
- All armour pieces of a class live in one constructor tpl (`cc_sm_constructor_soldier`, `_raider`, ...; soldier
  ~587 joints, raider ~598). `ssl/body/customization/character_customization_library.sso` lists body sections ->
  regions -> items whose `meshes` name tpl objects. Presets live under `customizationPresets`, keyed by mastery type;
  `PVE_PVP_*` and `STORY_*` sections exist per class, and CHAOS reuses class names (parse the LOYALIST team only).
- **Visibility is per class:** a region-flagged object that none of that class's items list is DRAWN. When several
  classes share one tpl, every class shows the other classes' pieces unless you add `isHidden = True` regions
  listing them (retail uses `isHidden` on ~32 regions; key order `... rarity, isHidden, isForAllCharacters,
  appearanceUids`). Cover every drawable object at any depth (seals, ropes, glass, ammo), skip LODs and sim meshes.
  Some retail objects have no region affix and are always drawn (e.g. a class's torso base): never duplicate them.
- **Removing a region whose object is still in the tpl makes it an orphan that is drawn at load**; if its material is
  the problem you get the crash earlier, not later. Hide, don't delete. Conversely, an item copied into a class whose
  tpl lacks the mesh shows as an empty slot (a blank head).
- Menu names: `ssl/ui/screens/codex/ui_heraldry_library.sso` (uid -> `UI_..._NAME/_DESC`) + `texts/*.str`.
- **Physics is per CLASS FILE, not per tpl**: `prop_cloth_sim_magnus` (cloth), `prop_phys_jiggle`
  (`objectsSettings`), `prop_motion_damper` (`springDampers`). Moving a piece to another class's constructor means
  merging the matching entries into that class's cls (both directions); seals and tabards are cloth, not jiggle.
- **Cloth (engine name "magnus")** is declared in the character's client `.cls` AND its `_client_base.cls`:
  `clothes = [{ simMeshName, renderMeshes, simRemapConfig, config { stiffnesses, gravity, cdtCapsules, cdtSpheres,
  solverFrequency, damping, drag } }]`. The render mesh is remapped onto the sim mesh at runtime by proximity. The sim
  mesh is a hidden object (`_SIM`, skinned 100 % to CENTRE, needs `visibility_hidden` or it renders as a grey tube)
  whose **pin weight is COLOR_0.r: 0 = pinned, 254 = free**. With the converter's channel rotation, unrotated pins
  read as "nothing pinned" and robes fly apart. (Saber's Cloth page agrees on the direction: some sim vertices must
  be completely black, "otherwise cloth falls off".)
- **Grafted bones must be animated.** Pieces skinned to bones the class's animation set does not drive (bones grafted
  from an NPC rig) look perfect in the lobby (rest pose at the origin) and stretch into huge sheets in missions. Move
  such weights to the nearest driven ancestor, or to a jiggle-driven bone the class already simulates. Test in a
  mission after moving, never only in the lobby.
- The raider skeleton already has ~1,300 `_jiggle` bones and hair joints that a retail helmet drives; re-weighting
  hair strands onto them is the cheap route to secondary motion (not tried in game). Changing weights onto bones a
  vertex already uses cannot move geometry at bind pose - it only changes deformation.
- The armoury figure is a separate dummy (`ssl/loading_scene_logic/customizable_character_dummy_factory.sso`), keyed
  by mastery type and carrying its own physics. Robe cloth that works in missions exploded on the dummy; removing the
  robe sims from the dummy only fixed it. The dummy and preset traps of a cloned class (`masteryType`, `isChampion`
  presets) are in the [class note](sm2-new-playable-class-on-character-mod-slot.md).
- After Patch 15 split sections into HALO / CLOTH / COAT, any preset missing those keys leaves the equipped uid blank
  and the WHOLE slot shows "not available for the selected Class". Presets must match retail's key set for the
  current patch. An equipped challenge-locked item gives the same message.
- Painted backpacks draw albedo from `<tex>_a` and a mask `<tex>_cc`; overriding the base diffuse does nothing under
  paint. Retail also ships unused alternate textures (e.g. a `_1` set of a helmet) worth checking before repainting.
- Colour values for "game-like" metals come from `ssl/body/customization/armor_color_library.sso` (tints, metalness
  and roughness bias per paint).

### Custom heads and helmets
- **Host choice matters.** A head object in the raider constructor (`tank_head_01`) owns two LOD0 skin splits and
  fixed stream budgets; `sniper_head_01` frees ~58k contiguous vertices in one geom once its hair/beard/lash/eye
  splits are switched off and has its own 2048 atlas used by nothing else. Both are skinned 100 % to bone `HEAD`, so a
  fitted transform transfers between them.
- Authoring geometry in place without understanding the packed encodings: decimate with half-edge collapses only
  (survivors keep their records byte-for-byte); fill a broken LOD by copying a good LOD's records and rebasing its
  indices; recover a scrambled vertex order with a Hungarian assignment on position (max error ~1-2 quantisation
  units means it is a pure permutation). An earlier third-party build "shredded" only because LOD0/4/5 index buffers
  were built against a different vertex order than the one written.
- Neck weights: a head 100 % on HEAD swings its neck as a rigid cylinder against the collar. Copy retail's head skin
  profile (only the bottom inch inside the collar blends into NECK/BACKup), not the collar's.
- Skull helmets: rigid on the `helmet` bone, a separate lens object with customization set `HELMET_LENSES` for a
  colourable lens, and an optional glass shell submaterial in front. Fit by measuring penetrations against the hood /
  collar meshes; AI models are normalised differently (by height or by bounding box), so refit scale per model.
- Head texture channels measured on retail: spec R (metal) ~0 on skin, G roughness 70-255, B AO 185-255; normal-map
  xy std ~17; diffuse alpha = subsurface mask AND transparency (`useDiffAlphaMask`): 255 on skin, ~128 on hair/beard,
  0 in sockets/mouth. Don't ship an all-zero channel or a near-flat normal map; aim for retail's statistics.

### AI-generated meshes (Tripo-style): rules that held
These apply to private builds only: Saber's mod rules don't allow AI-generated content in mods you release.
- **Measure the UV atlas before planning LODs**: chart count, median triangles per chart, share of vertices on a seam.
  Atlases with thousands of 2-triangle charts (~92 % seam vertices) cannot be decimated (triangles end up spanning two
  unrelated charts) and their mips bleed. Big-chart atlases (~3 % seam vertices) decimate fine with pymeshlab's
  texture-aware quadric collapse (feed it OBJ + MTL; a mesh built from arrays has no texture slot).
- A per-VERTEX textured preview hides all of this; render per pixel with UV interpolation.
- Re-unwrapping (xatlas into a few hundred charts with padding) and baking through the surface fixes seams, mip mud and
  texel density, but re-laying out a fixed painted atlas cannot add detail that the source export does not have.
- **Weld before averaging normals** (accumulate on unique positions); per-index accumulation on seam duplicates gave
  one-sided normals, away-facing patches (dark blotches that change with the light) and cracked meshes after inflation.
- AI normal maps can be a mosaic of square patches with different tilt - it is in the asset (check in a painting
  tool). Drop or heavily reduce it on skin; it reads as blotches through any tangent basis.
- AI "ORM" channels are R=AO, G=rough, B=metal; SM2 spec is R=metal, G=rough, B=AO. Some generators put metal ~200
  on everything (renders chrome/gold): check per model and rebuild from colour classes with retail values.
  AI normal maps are OpenGL (green up); retail is DirectX (green down).
- Hard edges come smooth-shaded from AI meshes (lines on flat plates): crease-split normals at ~40 deg. Audit for
  inconsistent winding, duplicate triangles (z-fight hairlines), T-junctions from local subdivision and open boundary
  loops before export; a software preview hides all four.
- Hi-poly and low-poly exports of the same generation are not at the same scale; align with an affine ICP before
  transferring detail.

### Textures (`.pct`)
- Formats seen: 52 = BC7 colour, 51 = BC7 (spec and others), 36 = BC5 (normal maps; weapon spec is BC5 R metal /
  G rough with td usage `MSCG+MRGH`), 17 = BC3 (accepted for custom textures). Usage tags: `MD` diffuse, `MD+MT` diffuse
  with transparency alpha, `MAO` (the converter reads AO from ALPHA: an RGB-only source gave black AO), `MEM` emissive,
  `MDT` detail. Normal maps are DirectX convention.
- Retail character sets are 2048 and listed in `mml_cfg` (see the foundations note): an unlisted 4096 is shown at
  2048 on High (measured from the exe), part of why a custom creature looked muddier than retail ones. The rest was
  the AI surface itself (lumpy mid-scale relief), which no resolution fixes. 8192 is only used for skies.
- Pillow resizes RGBA as premultiplied: with an SSS mask in alpha (0 on armour) an RGBA resize blacks out the RGB
  there. Resize RGB and alpha separately. `Image.BOX` when UPscaling is nearest-neighbour (blocky). Near-black paint
  needs variation in sRGB levels, not linear percentages, or it looks flat.
- AI atlases need chart-aware mips (each lower-mip texel averages only its dominant chart) or skin goes dirty at
  distance; the real fix is fewer, larger charts.
- Source TGAs handed to the converter should match retail's layout (uncompressed type 2, top-left origin, the
  TRUEVISION footer; copy the retail alpha plane if you only repaint RGB).
- UI textures (crosshair bitmaps): `.pct.resource` with format 0, one mip, and the mip file holding raw
  premultiplied BGRA pixels.

### Props, vehicles and collision
- Physics shapes built from mesh streams (`*_glr` convex shapes on a rigid-body object) crashed a level load (null
  stream pointer inside a convex-hull builder) for an exported prop, while retail ones work; cause unknown. Analytic
  boxes in the `tpl_markup` work: `tpl_markup_shape_box` entities (full extents W/H/D along the owner's local axes,
  centred at `localTransform.position`) on the rigid-body object, as retail turrets and the Dreadnought do. Capsules
  exist as `tpl_markup_shape_capsule`. Keep boxes axis-aligned (rotation order unverified). The server has no `.tpl`
  and builds physics from `.cdt`.
- **Player movement collision is the `.cdt` system, not Havok.** Affix groups `cdt_low_res_geometry` (invisible
  collision with player/AI/vehicles/camera) vs `cdt_high_res_geometry` (weapons/projectiles only). UsfExport bakes
  per-object flags into `.lods_base` (`collision = { plrMove = false ... __type = "tpl_markup_cdt_flags" }`).
  `geom.collisionLayerName` only picks a Havok layer and does NOT stop movement collision; a skinned prefab's movement
  collision comes from `geom.cdtSkeleton` (`geom_cdt_skel_tpl`), which can be removed from the class.
- **Level static collision** lives in the scene: `<scene>.cdt` is per-layer MOPP code over the geometry of
  `<scene>.lg` / `.lg_data` (no vertices of its own). The Map Converter guide says scenes can't be decompiled; reading
  them works anyway (the `.lg` is GeomManager-style chunks). Invisible walls are objects named
  `scripting|constrains|...` and similar. Remove one by making its faces degenerate in `.lg_data` (copy the first
  index over the other two): same size, MOPP untouched. Moving vertices does not move the collision.
- Prefab vehicles: a modded tpl BIND pose is ignored for animation; the game poses prefab bones from the retail rest
  and stacks additive TPL clips on it. Build anything riding a prefab bone relative to the retail rest pose and
  re-author the clips that move it.
- Swapping one vehicle model for another per level is a level-file edit (`class_list` `nameTpl`, `linksTpl`,
  cutscene copies: see the [missions note](sm2-missions-scenarios-faction-swaps-and-esd-spawning.md)). Model side:
  `objectsOverrides` hides old parts by RUNTIME object name, but unnamed skinned objects can't be targeted (hide
  `ROOT`), or point part classes at a locator-only tpl. A rigid hull placed directly inside an animated rig (no skin
  compounds) crashed in vertex-buffer binding; a static tpl on a root-attached part worked.
- Effect locators (jets, eye glows) come from the ANIMATION RIG at runtime; moving them in the tpl changes nothing.
  Either ship a renamed copy of a locator-only tpl and point `geom.nameTpl` at it, or offset the effect's own
  `transform` in the class, which IS applied relative to the locator.

### Replacing an NPC's mesh with a differently proportioned one
- Morpheme clips carry every bone's translation, so at runtime joints sit where the HOST's clips put them, whatever
  bind pose your tpl stores. Fitting the tpl skeleton to the new mesh stretched it in game. Either deform the mesh
  offline onto the host's bone offsets, or give the creature its own skeleton and rewrite the clips' fixed offsets
  (see the [NPC/animation note](sm2-new-enemies-npc-classes-and-morpheme-animation.md)).
- Verify by posing with the game's skinning (linear blend on the real rig), not by reading weight tables: rigid
  islands that share a welded shell (head vs hood) need a blend band between them; armour plates must be rigid on one
  bone each or they fold at the elbow; fingers need whole-finger components on the WELDED mesh.

## Build steps
New standalone model (weapon, prop, creature body):
1. Export the retail tpl you replace to glTF (ModEditor's ModelConverter, absolute paths) and copy its node tree,
   object names, affixes, LOD layout and tpl-baked animations.
2. Build your meshes into that tree (x10 + node scale 0.1 for small detail), one material/texture set per `_matN`,
   glTF TANGENT on every primitive, COLOR_0 pre-rotated where needed.
3. ModelConverter (cwd = its folder, `--keep-tangent-orientation`) -> UsfExport in a private project copy.
4. Check `obj_list.json` (names, face counts vs glTF, compounds), stream sizes, all-zero streams, tangents vs UV-derived
   frames, and every (FVF, material, td) triple against retail tpls.
5. Fix up: retail `.tpl.resource` + your links, retail `.tpl_markup(.resource)`, `.lods_base` flags/distances, collider
   name swap, unshrink positions if you compare to retail.
6. Textures via Saber's converter; td cloned from a retail td on the same FVF; `mml_cfg` entries.
7. Pak as ZIP_STORED with the manifests; test in game.

Adding pieces to an armour constructor: same, plus merge into the retail constructor glTF (uint16 joints, own `extras`
per copy, bones only from the class's driven rig), fix cloth pin colours and `colorSetIdx`, ship retail markup, add
library items/regions and `isHidden` regions for leftovers, merge physics entries into each class cls, update the
armoury dummy.

## Verification
- In game: round-tripped armour constructors with added helmets/heads/robes (rendering, cloth, class visibility), a
  new creature model with LODs and BC7 textures, new weapon models with tpl-baked reload/shot animations, a vehicle
  hull swap in levels, crash/no-crash for every shader-cache rule above, the tiny-triangle workaround, the
  grafted-bone stretch and its fix, the per-class visibility rule, the prefab bind-pose rule, the locator rule.
- Offline only: tangent byte encoding (99.95-100 % byte-exact vs UsfExport output), position shrink (5,160 of 5,172
  objects integer-identical to retail after the fix), maxTile parsing (28,084/28,084 bytes), scene `.cdt` decoding.
- Not verified in game: the latest tangent and unshrink fixes on every model, the scene-collision face removal, cloth
  hair re-weighting, the convex-shape crash's root cause.

## Gotchas
1. **Crash "Prebuilt shader cache or executable is obsolete" when an item is equipped or a level loads.** Cause: a
   novel (FVF, material shape, td submaterial) combination, very often from the glTF round trip dropping `layer1` /
   `colorSetIdx` or from a submaterial cloned from another model. Fix: read the define list in `error.yaml`, check
   every triple against retail, patch the fields in place or fall back to a cached shading model.
2. **Every rebuilt constructor crashed on the shader cache even with a clean material table.** Cause: UsfExport's
   `.tpl.resource` has empty `linkTplMarkup` / `tplMarkup`. Fix: retail resource + retail markup.
3. **Whole model renders with wrong materials after an in-place edit.** Cause: a hand-written `.tpl.resource` that
   blanked the markup link and dropped keys. Fix: rebuild it from retail, change only `linksPct` / `linksTd`.
4. **Normal map lighting "blotchy" on every round-tripped model.** Cause: no glTF TANGENT -> constant default
   tangents. Fix: supply tangents.
5. **Supplied tangents made it worse (99.6 % reversed).** Cause: ModelConverter negates TANGENT by default. Fix:
   `--keep-tangent-orientation`.
6. **Capes, tabards and robes hang rigid or fly apart after a rebuild.** Cause: COLOR_0 pin weights rotated by one
   byte, or lost entirely on cloth meshes whose names lack `_sim`. Fix: pre-rotate, give those meshes a material with
   colour sets via `mesh.extras.materials`, then restore retail `colorSetIdx`.
7. **A custom pauldron showed on every pauldron choice.** Cause: missing preserve affixes -> nameless object. Fix:
   copy the retail affix block; grep `"name": ""` in obj_list.json.
8. **Holes in fine detail (insignia, rims, lens LOD0).** Cause: UsfExport culls triangles under ~2 mm^2 in object
   space. Fix: mesh x10 + node scale 0.1.
9. **Several unrelated cosmetics vanished at once, no crash.** Cause: zero-filled tail of `.tpl_data`. Fix: re-export
   and scan for all-zero streams before packing.
10. **Decals sink into armour, seams open a few mm.** Cause: the 0.8 % re-encode shrink. Fix: rescale quantised
    positions by 32767/32500.
11. **Mesh shredded into long shards in the armoury, fine at bind pose.** Cause: merged JOINTS written as uint16 under
    a uint8 accessor header. Fix: set componentType 5123; verify by listing top-weighted bone names per object.
12. **Huge stretched sheets in missions only, lobby perfect.** Cause: weights on grafted, never-animated bones. Fix:
    re-bind to driven ancestors or jiggle-driven bones; test in a mission.
13. **Every class wears every other class's armour.** Cause: shared tpl + per-class visibility of unlisted region
    objects. Fix: `isHidden` regions per class covering every leftover object.
14. **Removing a crashing vanilla item made the crash happen earlier, at load.** Cause: orphaned tpl object drawn
    unconditionally. Fix: hide the region instead of deleting it.
15. **Whole armour slot "not available for the selected Class" (misleading text).** Cause: a preset lacks a Patch 15
    section key (blank equipped uid), or an equipped item is challenge-locked. Fix: match retail's preset key set for
    the current patch; strip or satisfy challenge locks.
16. **A helmet rendered collapsed although its own bytes were retail.** Cause: the previous object's streams changed
    size (vertex stream growth), plus misaligned maxTile entries. Fix: keep vertex stream sizes, fix maxTile per split.
17. **Textures tiling at the wrong scale on some parts.** Cause: maxTile table read/written as a flat array. Fix: the
    per-split `u8 count` + (u8 set, i16 tile) layout.
18. **UVs look like they sample a quarter of the atlas.** Cause: dividing u16 UVs by 65535. Fix: 32768 x maxTile.
19. **Writing the stream-size table 5 bytes off crashed the game, and the validator said PASS.** Cause: hard-coded
    offset, validator read the same wrong place. Fix: locate tables by searching, verify at the discovered location.
20. **Melee skin crashed in the armoury preview; then the collider drew as a white wedge.** Cause: `rb_*` objects lose
    UV/tangent streams in UsfExport, and the exported `.lods_base` makes every object visible. Fix: stand-in name +
    byte swap; ship retail `.lods_base`.
21. **Exporter exits 0 and writes nothing.** Cause: stale `models/database/<name>.tpl` intermediates. Fix: wipe it.
    Two jobs sharing one exporter project also wipe each other: use private copies.
22. **Moving tpl locators/bind pose changed nothing in game.** Cause: locators and bone positions come from the
    animation rig; prefab bind patches are ignored. Fix: offset effects in the class, re-author clips.
23. **Mesh fitted to a custom skeleton warped "to hell" when animated.** Cause: clips carry the host's bone
    translations. Fix: deform the mesh to the host offsets, or rewrite the clips' offsets for an own skeleton.
24. **Dark blobs on a custom face that change with the light.** Cause: normals averaged per vertex index on an AI atlas
    (seam duplicates) -> away-facing normals. Fix: weld before averaging.
25. **Polygonal patches on skin that survive every texture fix.** Cause: AI normal-map patch mosaic, or tangents copied
    from donor vertices with different UVs. Fix: drop the AI normal map on skin; write UV-derived tangents.
26. **Face looks clean close up, muddy at distance.** Cause: box-filter mips across tiny charts, and an unlisted 4K
    texture halved on High. Fix: chart-aware mips or a re-unwrap; `mml_cfg` entries.
27. **Every chitin/armour texel went black after resizing.** Cause: Pillow's premultiplied RGBA resize with an SSS
    alpha of 0. Fix: resize RGB and alpha separately.
28. **Level crashed loading an exported prop with physics.** Cause: convex `*_glr` shapes read stream data that was not
    resident. Fix: markup box shapes.
29. **Invisible wall the player can't pass after moving a prop.** Cause: level static collision (scene `.cdt` over
    `.lg`) or the prefab's `cdtSkeleton`, not Havok. Fix: degenerate the wall faces in `.lg_data`; remove
    `cdtSkeleton` from the class. `collisionLayerName` does not help.
30. **Lens lost its glow after export.** Cause: the lens faces were merged into the `default` submaterial. Fix: keep
    the lens as its own primitive with the retail `glass` (or an emissive) submaterial, with `intensity` set.
31. **Boot crash reading near-null after adding a texture set.** Cause: a cloned `.td.resource` lost its `name:` /
    `td:` lines. Fix: assert both on every td manifest.
32. **Robes explode on the armoury figure only.** Cause: dummy cloth config. Fix: remove the robe sims from the dummy
    entries (missions keep them), or use Saber's long-robe dummy settings.

## Assets
Custom meshes came from AI 3D generation (Tripo) and from retail parts re-used inside the game's own tpls; textures
were baked from those sources, repainted in Substance Painter by the human, or derived from retail textures
(recoloured, high-pass detail transferred triplanar). Nothing from the game is attached to this note. Saber's mod
rules forbid AI-generated content in mods (see the foundations note's Setup), so keep builds with the Tripo meshes
private and use hand-made meshes for a release.

## Open questions
- Root cause of the convex-shape physics crash on exported props.
- Whether the engine draws skin compounds that mix LOD levels per bone or as a whole.
- Exact effect of `texType` on custom textures.
- A reliable way to enumerate the cached shader permutations (the `.sdc` cache is hash-keyed, no define strings).


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

