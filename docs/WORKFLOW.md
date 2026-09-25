---
name: blender-anime-npr-pbr-lookdev
version: 1.2.0
description: >-
  Audit, diagnose, stress-test, and safely refine Blender 5.2/Cycles anime-character projects using a hybrid NPR+PBR workflow. Use for the 米砂 final project or similar imported/game-character assets when working on face shading, bounded identity protection, skin, hair, eyes, sheer cloth, official packed texture channels, custom normals, animation lighting, cameras, environment adaptation, volumetrics, Geometry Nodes, compositor consistency, or reversible Blender Python automation.
---

# Blender Anime NPR/PBR LookDev Skill

## Mission

Treat the project as a **technical-art pipeline**, not as a pile of sliders.

The goal is to preserve the source character's design while translating it into a stable Blender/Cycles render for both stills and animation.

Use this governing model:

```text
Stylized Character Rendering
=
Physical Spatial Coherence
+
Bounded Identity Protection
```

- **Physical Spatial Coherence**: the character must still belong to the scene, receive plausible light, preserve material differentiation, and retain credible 3D structure.
- **Bounded Identity Protection**: face design, eye readability, expression, and other identity-critical cues may be protected from physically plausible changes, but only as much as needed.
- Protection is not permission to make the face ignore the environment. If the face looks separately lit, pasted-on, or immune to exposure, protection has become excessive.

Always determine **which layer is actually wrong** before changing anything:

1. geometry / topology
2. UV / source texture
3. texture-channel decoding
4. normal / tangent / custom normal
5. BSDF material response
6. designed NPR response
7. lighting
8. camera / DOF
9. view-layer visibility
10. compositor / color management

Never fix a problem in a downstream layer until upstream causes have been ruled out.

---

## Project baseline: do not silently change

For the current 米砂 final project, treat these as the accepted baseline unless the user explicitly asks to change them.

### Final scene

- Scene: `米砂 | 晨光露台 · FINAL`
- Renderer: Cycles
- Resolution: 1800×1500
- FPS: 30
- Current render range: 480–1000
- Color management: AgX / Medium High Contrast / Exposure +0.12
- World: `米砂_LIGHT_RIG_V2_WORLD`
- Active camera at audit time: `米砂_LIGHT_RIG_V2_CAM_03` — 65mm / f5.6

### Final compositor

`MSA_Compositor_52_TRUE_PASSTHROUGH`

Required topology:

```text
Render Layers.Image ──→ Final Output
                  └──→ Viewer
```

Do **not** re-enable old Fog Glow / Exposure / Alpha Over simply because historical nodes or reports exist.

### Current chest-B accepted state

Object:

`021_星原_30_服装_胸衣_B`

Material:

`米砂 | 30_服装_胸衣_B__OFFICIAL_V3__CHEST_FINAL_FIX__CHEST_DIAG_NO_NORMAL_ILM`

Accepted diagnosis:

- Surface-response Normal input disconnected
- Micro Weave Bump strength = 0
- Official Normal Strength = 0
- ILM-B Master = 0
- Base Color / Alpha / Transmission / Sheen / Gold remain active

Do **not** “clean this up” by blindly restoring Official Normal/ILM.

### Current face accepted state

Material:

`FACE | 宣传图面部控制 v2__PBR_CENTER_FIX_FINAL`

Important principles:

- real Cycles PBR remains dominant
- NPR is bounded/local, not full replacement
- Hybrid PBR Weight ≈ 0.80
- skin roughness override ≈ 0.48
- SSS is conservative
- central seam is softened through localized normal treatment
- original V2 remains conceptually recoverable

### Current torso baseline

`米砂 | 暖肤 04_人物_躯干_B__TORSO_SKIN__BODY_SHAPING_A`

Approximate current values:

- Roughness 0.50
- IOR 1.42
- Specular IOR Level 0.28
- SSS Weight 0.12
- SSS Scale 0.0025

### Current hair baseline

`米砂 | 头发 · 香槟金各向异性`

- Roughness 0.40
- Anisotropic 0.55
- Specular IOR Level 0.30
- Sheen Weight 0.13
- micro bump is intentionally weak

---

## Prime directive: preserve a known-good baseline

Before any meaningful edit:

1. identify active Scene / View Layer / Camera / World;
2. record current render/color settings;
3. record target object + material slot;
4. duplicate the target material or scene when the change is risky;
5. store a short state report;
6. make the smallest possible change;
7. render/compare;
8. only then promote the change to final.

Never overwrite the only known-good copy.

For automation scripts, prefer:

- `MODE = "APPLY" | "RESTORE"`
- deterministic names
- ownership prefix
- idempotent lookup before creation
- explicit changelog
- refusing to overwrite the input `.blend`

---

## Diagnostic ladder for material problems

When a material “looks misaligned”, “dirty”, “painted on”, “creased”, or “wrong even though UV seems right”, do **not** begin by editing UVs.

Create diagnostic copies and test in this order.

### Stage A — Source color

`TEXTURE_ONLY`

Keep only:

- original Base Color
- controlled Alpha if needed

Disable:

- Normal
- Bump
- ILM modulation
- Transmission
- Sheen
- Coat
- extra emission

Question: **does the color texture itself line up?**

### Stage B — No Normal / ILM

`NO_NORMAL_ILM`

Keep:

- Base Color
- Alpha
- main fabric response
- Transmission / Sheen if they are known-good

Disable:

- Principled Normal input
- micro Bump
- official Normal master
- ILM roughness/specular master

If this fixes the appearance, the problem is **not a Base Color UV error**.

### Stage C — Re-enable one channel at a time

1. micro bump only
2. official normal only
3. ILM roughness only
4. ILM specular only
5. combined

Never re-enable all channels at once after diagnosis.

---

## Official game texture adaptation rules

A filename is a hint, not a semantic contract.

### Color textures

- use sRGB for albedo/color
- preserve source alpha until its meaning is verified

### Data textures

Use Non-Color for:

- Normal
- ILM
- masks
- Denier
- Roughness
- Height

### RG normal reconstruction

If the source stores only X/Y in R/G:

```text
Nx = 2R - 1
Ny = 2G - 1
Nz = sqrt(max(0, 1 - Nx² - Ny²))
```

Re-encode to 0–1 before feeding Blender's Normal Map node.

Provide a **Flip Green** switch. Do not guess DirectX/OpenGL orientation.

### ILM

Before assigning channels to material properties:

1. separate R/G/B/A;
2. expose each channel as debug output;
3. inspect spatial meaning;
4. compare against visible material categories;
5. only then map it to gold / roughness / specular / other response.

Do not let unknown ILM channels drive final shading.

### Denier

Only apply Denier when:

- target material is genuinely sheer;
- texture family matches the material family;
- response can be controlled with a master value;
- alpha/roughness/transmission changes remain bounded.

Never apply one family's Denier globally because names are similar.

---

## Face workflow: PBR first, NPR as bounded art direction

Do not build a pure toon replacement for Cycles skin.

### PBR remains responsible for

- true scene-light response
- real shadowing
- specular
- SSS
- depth/volume cues

### NPR may control

- designed cheek/nose/chin structure
- left/right key-light threshold behavior
- subtle local tint
- pose-specific hair contact guidance
- local normal flattening where facial topology produces an unwanted split

### Directional face shadow

Use a head-local key direction.

The mask should respond to light azimuth in head space rather than be a static screen-space shadow.

Do not assume UV symmetry by using `1-U` unless the UV layout has been explicitly verified as mirrored.

### Region mask

Keep region channels semantically explicit, e.g.:

- R = nose
- G = cheek
- B = chin

Prefer `max()`/bounded combination for overlapping local effects instead of stacking multiple darkening layers additively.

### Hair-contact mask

Treat a hair-shadow mask as a **pose-specific contact guide**, not as a substitute for real Cycles hair shadow.

If hairstyle, bang pose, or key-light direction changes substantially, regenerate/repaint it.

### Bounded identity protection

Do not describe the whole character with one global “PBR/NPR percentage”.

Treat stylization as **domain- and property-dependent**:

| Domain / property | Physical-light freedom |
|---|---|
| metal reflections | high |
| cloth roughness / broad body shading | high |
| hair directional highlight | high |
| face overall luminance | medium |
| skin environment tint | medium-low |
| nose-shadow shape | low |
| cheek core structure | low |
| anime-eye readability | very low |
| expression-critical marks | very low |

The practical rule is:

> protect identity-critical information, but leave enough physical response for the character to remain spatially integrated.

If a fix makes the front view cleaner but damages 3/4 structure, lowers temporal stability, or makes the face ignore scene lighting, reject it.

### Shading responsibility allocation — Jinshi follow-up

After locating the earliest wrong layer, ask **which subsystem should primarily own each visible cue**: silhouette/head direction, nose and cheek form, designed shadow boundary, environment tint, and edge separation. Record permitted secondary contributors and the identity envelope they must not exceed. See the separate [Jinshi case study](JINSHI_RESPONSIBILITY_CASE_STUDY.md); its numerical closeout is project-specific, not a preset for 米砂 or other characters.

Flag **Redundant Volume Encoding** when geometry normals, regional normals, directional face response, designed shadow, and character lights all strongly restate the same nose, cheek, or eye-socket form. Test one layer at a time. Individually reasonable components can produce an unreasonable combined result.

For a stable face that looks too modeled only in some 3/4 views, inspect hair occlusion/contact → Face Fill → Key → Rim → camera/view → face-material architecture. Preserve silhouette, nose, chin, and head orientation; do not flatten the entire face by default. This is a diagnostic order, not an assumption that lighting is always responsible.

---

## Face normal rules

If the face looks flat, pinched, bruised, or split:

1. inspect topology and vertex normals;
2. inspect custom-normal modifiers;
3. disable tangent normal maps temporarily;
4. compare with flat/neutral normal;
5. only then change lighting.

Prefer localized normal correction:

- soften cheeks/forehead more;
- preserve nose ridge/tip;
- preserve chin silhouette;
- avoid flattening the full face.

For this project, the successful pattern is:

- Data Transfer for broad normal smoothing
- Normal Edit with a limited mix and vertex group
- localized shader normal blend around the center seam

Do not solve the center seam with a painted vertical dark/light stripe.

### Shading-domain consistency

Face, hair, body, cloth, and metal may use different shading models, but they must still describe the same scene.

Check specifically for:

- face = flat/high-key while body = strongly modeled;
- cool environment visible on hair/body but absent from the face;
- rim light obeying head direction while body light obeys world direction with no visual reconciliation;
- exposure clipping the face before the rest of the character;
- the face appearing to have a separate studio light.

The target is **different techniques, one visual language**.

---

## Skin rules

Do not equate “more SSS” with “more flesh”.

Skin meatiness comes from the combination of:

- believable roughness
- restrained SSS
- correct normal gradients
- warm/cool light separation
- preserved micro-contrast
- non-flat midtones

When face and torso differ, do not force identical numeric parameters. Match the **visual response**, not the slider values.

Avoid excessive emission on skin.

---

## Hair rules

Preferred Cycles hair strategy:

- anisotropic Principled response
- UV/tangent-aligned direction
- restrained roughness
- weak micro bump
- optional sheen
- preserve albedo detail without white painted streaks dominating

Do not use large emission to fake hair highlights.

If using a back/translucency light, keep it supportive rather than making it the only source of hair definition.

---

## Eye rules

If the imported eye feels like a flat sticker:

separate the conceptual layers:

1. sclera / eye white
2. iris/pupil
3. highlight/detail maps
4. cornea/reflection cap

Use the source H1/H2/detail maps only after verifying their semantics.

Keep eye emission bounded; it should restore iris readability, not glow like an LED.

Focus-object placement should follow the eyes, not the front surface of the cornea.

---

## Sheer fabric rules

A sheer costume can contain at least two distinct material classes:

- translucent/sheer fabric
- metallic trim

Do not make gold inherit the fabric's transmission response.

Typical fabric controls:

- Base Color
- Alpha
- Transmission
- Roughness
- Sheen
- restrained SSS if appropriate
- optional micro weave

Typical gold controls:

- Metallic = 1
- separate roughness
- separate normal decision

Use a reliable mask to mix the BSDFs.

---

## Lighting workflow for animation

A lighting setup that looks good at frame 1 can fail after a 180° turn.

### Audit pose directions

Sample sparse frames and measure:

- head yaw
- torso yaw
- head/body twist
- key/rim incidence

Flag frames where:

- head/body twist is large;
- rim crosses toward the front of face/body;
- key becomes backlight;
- face fill becomes dominant.

### Dynamic light rig

Prefer:

- sparse keyframes
- drivers bound to explicit control properties
- Bezier / AUTO_CLAMPED interpolation
- circular angle math around ±180°

Avoid frame-change Python handlers for a deliverable unless absolutely necessary.

A robust rig may include:

- warm key partially following torso
- cool rim positioned from the head/body bisector
- face fill following head
- weak body fill

Reduce rim intensity before its emitter moves into a frontal incidence zone.

---

## Camera workflow

Choose cuts near:

- completed poses
- low angular velocity
- turn transitions
- stable head/body orientation

Do not cut at arbitrary equal time intervals.

For each shot:

- use an independent focus empty;
- focus near the eyes/head for portrait work;
- preserve Euler continuity between sparse keyframes;
- frame according to the animation silhouette, not just the rest pose.

If DOF is aggressive, validate hair strands, cornea and transparent cloth for bokeh artifacts.

---

## Environment and volumetric rules

Keep environment responsibilities separate from character lookdev.

### World

HDRI may light the scene while camera rays see a different controlled background via `Light Path → Is Camera Ray`.

This is preferred over darkening the HDRI itself when the HDRI is needed for material reflections.

### Volume

Use a bounded volume density and forward anisotropy.

For the current terrace baseline:

- Density ≈ 0.026
- Anisotropy ≈ 0.45

Do not increase density simply to make sun beams visible; first check beam energy, spot angle and environment contrast.

### Dust / fireflies

Use Geometry Nodes for:

- volume point distribution
- low-amplitude temporal noise drift
- randomized instance scale
- emissive particles

Keep particle motion slow enough not to read as snow/confetti.

### Character Environment Adaptation

`Character Environment Adaptation` is a **project-level abstraction**, not a claim that Blender or a specific game exposes a standard feature with this name.

Treat it as two separate axes:

1. **Chromatic adaptation**
   - allow some environment hue into skin shadows/edges;
   - protect identity-critical skin hue from complete cyan/green/red contamination;
   - keep face/body chromatic response related rather than identical.

2. **Luminance adaptation**
   - allow overall face brightness to follow scene exposure;
   - preserve enough nose/cheek/chin/eye structure that high exposure does not collapse the face into a flat patch;
   - do not compensate exposure by simply adding emission.

Optional third axis:

3. **Expression protection**
   - preserve eyelid/eyebrow/lip/blush readability;
   - expression masks may receive stronger protection than generic skin tone;
   - do not let expression protection become a static painted shadow.

When a cool environment is present, `Environment Affect = 0` and `Environment Affect = 1` are both suspicious extremes.

---

## Compositor rules

Always diagnose a Viewport/F12 mismatch before rebuilding post-processing.

Check:

1. active Scene
2. active View Layer
3. camera
4. light visibility
5. world
6. film transparency
7. compositor enabled flag
8. Render Layers node's Scene/Layer selection
9. Viewer and Composite source

For the current final project, **passthrough is the baseline**.

Passthrough is a diagnostic and delivery baseline, **not a universal rule that compositing is forbidden**. Image-space finishing such as subtle bloom or color finishing is acceptable after structural material/lighting problems are solved.

If post is added later:

- add one node/effect at a time;
- preserve a passthrough branch;
- keep strength low;
- compare Render Result and Viewer;
- never hide a material bug with Glow or Exposure.

---

## View Layer and Collection rules

Use Collections/View Layers as responsibility boundaries:

- protected character
- environment
- volume
- contact-shadow proxy
- cameras/focus
- active animation rig
- historical/disabled lights

When debugging a missing object or light, check all of:

- object hide flags
- collection exclusion
- view-layer exclusion
- camera-ray visibility
- Cycles ray visibility
- holdout/indirect-only state

Do not assume Outliner visibility alone describes F12 visibility.

When `Light Path → Is Camera Ray` is used to decouple camera background from HDRI illumination, also verify:

- reflective metal does not reveal a contradictory environment;
- cornea/hair highlights remain semantically plausible;
- glossy/transmission rays do not expose an obviously different world.

This technique is an intentional art-direction trick, not a physically neutral operation.

---

## Blender Python mutation pattern

Every repair script should follow this pattern:

```python
MODE = "APPLY"  # or RESTORE
PREFIX = "PROJECT_FEATURE_"

# 1. Locate exact target object/material/scene.
# 2. Refuse ambiguous matches.
# 3. Snapshot current state.
# 4. Reuse owned objects if they already exist.
# 5. Create a copy for risky edits.
# 6. Apply one bounded change.
# 7. Write a report / state JSON into a Text datablock.
# 8. Save to a new path if saving is requested.
# 9. Provide RESTORE behavior.
```

### Safety requirements

- Never overwrite source automatically.
- Do not delete user data to make the script “clean”.
- Do not rename unrelated data blocks.
- Do not rebuild an entire material when a single link/value is the problem.
- Do not add a second modifier that duplicates an existing correction.
- Before adding nodes, search by stable name/label.
- After creating a material, validate assignment to the **exact intended slot** and check `target_material.users > 0`; a zero-user datablock is not an applied fix.
- Inspect node `bl_idname`, socket types, directions, and links before mutation; a label does not prove that a node is a writable scalar control.
- Do not multiply an already shaded RGBA result as though it were a contribution-strength parameter. Define and test a meaningful neutral reference before mixing.
- Cache downstream node/socket references before removing a `NodeLink`; never dereference the removed link afterward.
- Change one responsibility layer at a time unless testing its interaction with another layer is the explicit purpose.

---

## Evidence discipline for reverse engineering

When discussing a proprietary game's rendering, classify claims before writing them into reports:

1. **Engineering fact** — directly observed in the `.blend`, source asset, node tree, script, or render setting.
2. **Output observation** — visible in the project's final render/video.
3. **External technical precedent** — supported by public documentation, talks, papers, or open implementations.
4. **Working hypothesis** — a visual inference about a proprietary renderer.

Never upgrade category 4 into category 1.

In particular, do **not** state that a proprietary game uses SDF face shadows, a specific edited-normal algorithm, a specific PBR/NPR weight, or a particular packed-channel meaning unless a primary source or asset-level inspection proves it.

Use phrases such as:

- “visually consistent with…”
- “a plausible explanation is…”
- “this belongs to the same technical family as…”
- “not confirmed by the developer”

---

## Stress-test and ablation protocol

A still-image sweet spot is not enough. Before promoting a face/lighting system to reusable status, stress it.

### Minimum angle matrix

Render at least:

- 0°
- 30°
- 45°
- 60°
- 90°

Use both left and right rotations when asymmetry matters.

### Minimum lighting matrix

Test:

- frontal soft key
- side key
- top light
- back/rim light
- cool environment
- warm environment

### Exposure matrix

At minimum test:

- EV -2
- EV -1
- EV 0
- EV +1
- EV +2

High exposure is especially important because stylization already removes some form cues; clipping can erase the remaining face structure.

### Animation / pose matrix

Include:

- head tilt
- head/body twist
- partial hair occlusion
- arm crossing near face
- expression change
- near-frontal and 3/4 close-ups

### Acceptance criteria

Evaluate:

- **Cross-view structure stability** — nose/cheek/chin remain spatially coherent.
- **Temporal stability** — no shadow pop, normal flip, mask flip, or sudden rim invasion.
- **Worst-case identity preservation** — the weakest tested frame still clearly reads as the intended character.
- **Shading-domain coherence** — face/hair/body do not look like they belong to different lighting systems.
- **Environment integration** — the character receives enough chromatic/luminance influence to belong to the scene.
- **Material separation** — skin, hair, cloth, metal, and sheer fabric retain distinct response.
- **Responsibility clarity** — each key form cue has a primary owner; several systems do not strongly and unintentionally re-encode the same volume.

### Recommended ablation sequence

For face-shading R&D, compare the same camera/light/pose with:

```text
A — Pure PBR / geometry normals
B — Uniform face flatten
C — Regional normal control
D — Regional normal + designed face shadow
E — D + environment adaptation
F — E + character-specific Face Fill
G — F + final Rim polish
```

Only claim that a technique improves robustness when the controlled comparison supports it.
Treat F and G as **planned extensions** for future controlled tests, not as completed tests in the original 米砂 audit.

---

## External technical anchors

Use these as **precedent/implementation references**, not as proof of any proprietary game's internal shader:

- Blender 5.2 — Normal Edit Modifier: https://docs.blender.org/manual/en/5.2/modeling/modifiers/normals/normal_edit.html
- Blender 5.2 — Data Transfer Modifier / Custom Normals: https://docs.blender.org/manual/en/5.2/modeling/modifiers/modify/data_transfer.html
- Blender 5.2 — Light Path / Is Camera Ray: https://docs.blender.org/manual/en/5.2/render/shader_nodes/input/light_path.html
- Blender 5.2 — Packed Data: https://docs.blender.org/manual/en/5.2/files/blend/packed_data.html
- Unity Toon Shader — Normal Map effectiveness: https://docs.unity3d.com/ja/Packages/com.unity.toonshader%400.8/manual/NormalMap.html
- Unity Toon Shader — Parameter / scene-light controls: https://docs.unity3d.com/ja/Packages/com.unity.toonshader%400.9/manual/Parameter-Settings.html
- Arc System Works — *Guilty Gear Xrd's Art Style: The X Factor Between 2D and 3D* (GDC 2015): https://www.arcsystemworks.com/guilty-gear-xrds-art-style-the-x-factor-between-2d-and-3d-talk-from-gdc-2015-is-now-available-online/
- Microsoft DirectXTex — `--reconstruct-z` / `--invert-y` normal-map handling: https://github.com/microsoft/DirectXTex/wiki/texconv

---

## Final validation checklist

Before declaring a project final:

### Rendering

- [ ] correct Scene
- [ ] correct View Layer
- [ ] correct Camera
- [ ] intended render range
- [ ] intended FPS/resolution
- [ ] AgX / exposure verified
- [ ] Render Result = expected output
- [ ] Viewer uses same source when passthrough is intended

### Character

- [ ] face center seam acceptable
- [ ] face/neck/torso color continuity acceptable
- [ ] no eye-white/cornea layering artifact
- [ ] hair anisotropic highlight stable through animation
- [ ] sheer cloth does not flicker or look UV-shifted
- [ ] gold remains metallic
- [ ] chest-B diagnostic state not accidentally reverted
- [ ] 0° / 30° / 45° / 60° / 90° face stress test checked
- [ ] EV -2 … +2 exposure stress test checked for face-structure collapse
- [ ] worst-case frame still preserves character identity
- [ ] face / hair / body shading domains remain visually coherent

### Lighting

- [ ] key/rim do not swap roles during turns
- [ ] cool rim does not wash the face
- [ ] volume does not fog the character unintentionally
- [ ] Viewport/F12 light visibility matches intent
- [ ] rim does not invade the frontal face during head/body turns
- [ ] cool/warm environments influence the character without destroying protected face cues
- [ ] camera-ray background separation does not create contradictory glossy reflections

### Dependencies

- [ ] all active external textures identified
- [ ] `Pack Resources` performed for archive/delivery copy
- [ ] reopen after source texture directory is unavailable
- [ ] no Missing Files

### Project hygiene

- [ ] baseline scene retained
- [ ] risky edits have rollback state
- [ ] final material names identify their role/version
- [ ] historical experiments are not mistaken for active pipeline

---

## Current project portability warning

At audit time, the active character pipeline still references these un-packed official textures:

- `tex_05_cloth_014_denier__y0a3.tga`
- `tex_05_cloth_014_ilm__y0a3.tga`
- `tex_05_cloth_014_n__r0y0a3.tga`
- `tex_05_clothtrans_014_ilm__y0a3.tga`
- `tex_05_clothtrans_014_n__r0y0a3.tga`
- `tex_05_hair_008_d__r0y0a3.tga`
- `tex_05_eye_012_h1__y0.tga`
- `tex_05_eye_012_h2__y0.tga`

Before handing the `.blend` to another machine, pack and verify them.

---

## Decision rule to remember

When a render looks wrong, ask:

> **What is the earliest layer in the pipeline that can explain this symptom?**

Then ask a second question:

> **Is the proposed fix improving the sweet spot, or improving the worst case?**

Fix the earliest responsible layer, and reject changes that only beautify one angle while making cross-angle, cross-light, or temporal stability worse.

Do not use lighting to repair UVs.  
Do not use SSS to repair color.  
Do not use compositing to repair normals.  
Do not use a normal map merely because the file is named `_n`.  
Do not keep an ILM channel active merely because it exists.  
Do not replace a stable PBR face with full NPR when only one facial region needs art direction.
