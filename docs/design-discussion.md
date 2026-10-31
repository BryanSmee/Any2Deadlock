# Any2Deadlock — Design Discussion

Notes from the initial brainstorming session (2026-10-31) that led to this project.
This is a design record, not documentation of working code.

## Goal

Convert a humanoid character from another game (starting with **Maelle** from
*Clair Obscur: Expedition 33*, Unreal Engine 5) into a **Deadlock** hero skin
(starting with **Violet**).

Game assets can't be redistributed, so the project ships a **recipe, not a
mod**: scripts, mappings and templates that users run against games they own.
The tool rebuilds the mod locally from their own files.

## The manual pipeline (baseline)

There is no one-click converter. Deadlock runs on Source 2, so the character
has to be rebuilt for that engine:

1. **Extract from Unreal** — FModel / CUE4Parse (UE4/UE5) or UE Viewer (older
   UE): mesh, skeleton, textures. UE5 games usually need a mappings file
   (`.usmap`) and an AES key if paks are encrypted (community-shared; never
   bundled).
2. **Extract the target hero** — Source 2 Viewer (VRF) to export the hero's
   `.vmdl` from the game VPKs.
3. **Re-rig** — fit the source mesh onto the Deadlock hero's skeleton and
   transfer weights. Keeping the original hero skeleton means Deadlock's
   animations just work. *This is the hard part.*
4. **Convert materials** — split/remap Unreal's packed PBR textures for
   Source 2 shaders.
5. **Compile** — Source 2 workshop tools (`resourcecompiler`, ModelDoc):
   content files (`.fbx`, `.dmx`, `.psd`) → game files (`.vmdl_c`, `.vtex_c`).
   Shader/format versions must match Deadlock's.
6. **Pack & install** — build a VPK, drop it in `citadel/addons` (or use
   Deadlock Mod Manager).

Known gotchas:
- T-pose in game ⇒ the animation graph isn't found, usually a skeleton mismatch.
- Skins are client-side; other players see the default model.
- Ripped models belong to their publishers: personal use is generally
  tolerated, redistribution is not.

## Key idea: re-skin onto the target skeleton

Don't map the source skeleton *to* Violet's. **Discard the source skeleton and
re-skin the mesh onto Violet's skeleton.** Violet's skeleton stays untouched,
so her animations, attachment points and animation graph keep working.

Generalized steps:

1. **Detect ~20 key humanoid bones** — name heuristics for common conventions
   (UE mannequin `pelvis`/`spine_01`/`upperarm_l`, Mixamo, `Bip01`, side
   suffixes `_l`/`.L`/`Left`); fall back to skeleton topology (root near hips,
   longest symmetric chains = limbs, top chain = head).
2. **Pose-match** — rotate source limbs so the rest pose matches Violet's
   (A-pose vs T-pose) and bake it into vertices via linear-blend skinning.
3. **Fit proportions** — uniform scale to Violet's height; optionally
   per-segment stretching (slider: keep source proportions ↔ match target).
4. **Remap weights** — key bones transfer directly; unmapped bones (fingers,
   twists) merge into nearest mapped ancestor. Dangling chains (hair, skirts,
   tails) can become auto-generated **jiggle bones**.
5. **Guess materials** — classify textures by suffix (`_D`/`_BaseColor`, `_N`,
   `_ORM`/`_MRA`, `_M`), split packed channels, fill a `.vmat` template.

**Blender is optional.** Everything above is numpy + a mesh library. Output can
be FBX or Valve's DMX (writable directly from Python). The only Valve
dependency left is the resource compiler.

## Phase 1 — Proof of concept (no human intervention after setup)

**User provides:** game path, mappings file / AES key if needed, character
asset path.

**Automated:**
- Extraction.
- Bone mapping for conventionally named skeletons (regex heuristics).
- Weight remap onto Violet's skeleton; unmatched bones collapse into nearest
  matched ancestor (fingers/hair/skirts move rigidly — ugly but working).
- Uniform scale + axis/unit conversion (UE cm → Source units).
- Basic materials: base color + normal per slot, one generic `.vmat` template.
- Write FBX/DMX, generate `.vmdl` from template, compile, pack VPK, install.

**Out of scope:** faces (static), jiggle physics, proportion fitting, A/T-pose
matching (test only with close rest poses), packed-texture splitting.

**Success criteria:** loads in place of Violet, animates without exploding
vertices, is recognizable.

**First thing to validate (before writing the generic tool):** one manual
Maelle → Violet run, to confirm Deadlock accepts a model compiled with the
available tools (shaders, model format, compiled version). If that fails,
nothing else matters.

## Later phases

### LLM in the loop (bring your own key)

- **Good for semantics:** bone names (abbreviations, other languages), texture
  and material-parameter classification, picking the character mesh among
  thousands of assets, reading UE material instances, explaining compile errors.
- **Bad for geometry:** weights, pose alignment, proportions stay algorithmic.
- **Guess → validate:** LLM returns structured JSON with confidence;
  deterministic checks reject impossible answers (hierarchy order must match the
  target, left/right symmetry, rough positional agreement). Failures fall back
  to heuristics or get flagged.
- Cheap: one or two calls per conversion. Results get cached as profiles.

### Phase 2 — Look right
- Rest-pose matching (biggest visual win).
- LLM-assisted bone/texture/asset mapping with validation.
- Packed-texture splitting, proper multi-slot materials.
- Preview render in a few poses before compiling; optional vision-model QA
  ("are the arms twisted?") as a warning, not a judge.

### Phase 3 — Motion and fit
- Auto jiggle bones from leftover hair/skirt/coat chains.
- Proportion slider.
- Outfit/accessory selection; hide source weapon, keep the hero's weapon
  (e.g. Violet's brush).

### Phase 4 — Generalize
- **Any target hero** via per-hero profiles (skeleton, bones to leave alone,
  `.vmdl` template).
- **More source engines** (see below).
- **Community profiles per game** — data-only JSON, contributed as PRs.

### Phase 5 — Hard ceiling
- Faces, partially: jaw for talking, maybe eye look-at; ARKit-style blendshape
  names can be LLM-mapped. Full facial fidelity is unrealistic.
- Stylized shaders (e.g. Expedition 33's painterly look) can't be reproduced
  generically.

**Realistic end state:** a good-enough skin for normal humanoids in about five
minutes from a game the user owns. Hand-made mods remain better.

## Multi-engine support

Architecture: **engine-specific extractor → common format (glTF) → shared
pipeline → Deadlock.** A new engine is only a new extractor, ideally wrapping
an existing community tool.

| Source | Difficulty | Notes |
|---|---|---|
| Unreal (UE4/UE5) | Medium | FModel/CUE4Parse; mappings + AES keys. Phase 1 target |
| Generic FBX/glTF | Very easy | Almost free; covers user-made models (no IP issue) |
| Unity | Easy–medium | AssetRipper/AssetStudio. Humanoid Avatar assets give bone mapping for free; standard shader property names (`_MainTex`, `_BumpMap`). Encrypted bundles (often gacha games) unsupported |
| Source / Source 2 | Trivial | Source 2 Viewer |
| Godot | Easy | GDRE Tools; native glTF |
| Proprietary (RE Engine, Frostbite, Decima, id Tech…) | Hard | Only where community extractors exist; per-game |

Suggested order: Unreal → generic glTF/FBX → Unity → others on demand.

Constraints: offline extraction from installed files only, never read a running
game (anti-cheat, ToS).

## Confidence assessment

| Piece | Confidence |
|---|---|
| Replacing a Deadlock hero model at all | Very high (existing skin mods do it) |
| Extracting from Unreal and Unity | High |
| Automated compile/pack/install | High — verify toolchain compatibility first |
| Auto bone mapping, conventional humanoids | High |
| PoC quality (loads, animates, recognizable) | Fairly high |
| "Good" quality without manual fixes | Medium |
| Faces and stylized shaders | Low |

**External risks:** Valve adding file-integrity checks or changing the format
(especially around 1.0); hero rig changes after patches (Violet released
2026-10-16 and may still change).

## Open-source & profile contributions

- Profiles are **data only**: bone maps, asset paths, material hints, settings.
  Never mesh data or weights derived from game assets.
- JSON schema + CI on every profile PR (schema, hierarchy rules, symmetry), so
  bad profiles can't merge even when reviewers don't own the game.
- Each profile records the game version (asset paths move between patches).
- README must state: unofficial, not affiliated with Valve; use only with games
  you own; don't redistribute outputs.
