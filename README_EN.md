# Stylized Character Rendering in Blender/Cycles

[中文](README.md) · [Technical summary](docs/TECHNICAL_SUMMARY.md) · [LookDev workflow](docs/WORKFLOW.md) · [Methodology](research/methodology_EN.md) · [Validation framework](research/validation-framework_EN.md) · [References](REFERENCES.md) · [Planned experiments](experiments/README.md)

This is a **Blender 5.2.x / Cycles** technical-art R&D case study in anime character LookDev. Starting from diagnostic work on an imported game-character asset, it develops a stable rendering workflow for promotional stills, animation, multiple viewpoints, continuous lighting, and cinematic shots. The result is a **stylized character translation layer for Cycles**, not a recreation of any game's complete shader.

The central question is which visual properties should respond freely to a three-dimensional world, and which must remain protected by character design.

```text
Stylized Character Rendering
= Physical Spatial Coherence
+ Bounded Identity Protection
```

Physical lighting supplies volume, material distinction, and environmental response. NPR and art direction protect identity-critical cues within defined limits. Face, hair, and body can use different shading techniques as long as they still appear to inhabit the same scene.

## Pipeline Overview

Diagnose first and modify the earliest failing layer. Then protect only identity-critical cues within defined bounds and check that the result still belongs in the scene.

```text
Source Asset
    ↓
Geometry / UV Audit
    ↓
Texture Semantic Decode
    ↓
Normal / Tangent Validation
    ↓
Physical Material Base
    ↓
Bounded NPR / Identity Protection
    ↓
Environment Re-coupling
    ↓
Lighting / Camera
    ↓
Validation Matrix
    ↓
Final Stylized Character

Physical Spatial Coherence + Bounded Identity Protection
```

## Why hybrid NPR and PBR?

Unrestricted PBR can turn a shallow anime nose, eye socket, or cheek into geometry-driven shading that contradicts the design, especially as the view changes. Heavy toon treatment can make the face read as a sticker, weaken three-quarter structure, suppress environmental color, and erase remaining form cues at high exposure. The solution is a controlled allocation of responsibility across regions and properties, not a single PBR/NPR percentage for the whole character.

| Domain or property | Physical-light freedom | Design priority |
| --- | --- | --- |
| Eyes and expression cues | Very low | Preserve identity and readability |
| Core cheek structure | Low | Preserve clean designed forms |
| Nose and chin | Limited but essential | Retain spatial orientation and volume |
| Overall face luminance and environment tint | Moderate, bounded | Follow the scene without losing structure |
| Body, hair, and cloth | Higher | Maintain continuous light and material behavior |
| Metal reflection | Very high | Respond clearly to lights and surroundings |

In shorthand, `PhysicalFreedom = f(region, material, light, view)`; pose and occlusion also matter in production.

## Working method

1. **Diagnose the earliest failing layer.** Examine geometry and UVs, texture-channel semantics, normals, BSDF response, designed face shading, lights, camera, visibility, and compositing in causal order. A `_n` suffix does not verify normal-map semantics, and an ILM channel need not drive the final material. Lighting cannot repair UVs; SSS cannot repair source color; compositing cannot repair normals.
2. **Control normals by region.** Soften cheek and forehead response where needed, preserve the nose and chin's spatial role, and correct center seams locally. The audited project uses Data Transfer, Normal Edit, and local shader-normal correction; the accepted configuration is recorded in the [technical summary](docs/TECHNICAL_SUMMARY.md).
3. **Reconnect artistic shading to the scene.** A face may be temporarily decoupled from ordinary PBR response to shape design-critical cues. It must then respond in a bounded way to environment hue, luminance, exposure, and scene lighting: `Decouple → Art Direction → Controlled Re-coupling`.
4. **Test difficult conditions and preserve rollback.** A pleasing still is insufficient. Compare angles, light directions, environments, exposure, and animation while keeping reproducible A/B states and a recovery path.

See the [methodology](research/methodology_EN.md) and [validation framework](research/validation-framework_EN.md) for the full decision and test process.

## Cases from the project

**Face:** Cycles PBR remains responsible for scene lighting, shadowing, highlights, and volume. Bounded NPR controls selected design forms. Head-local light direction, region masks, and local normal correction constrain cheek, nose, and chin behavior without replacing the entire face response.

**Chest material diagnosis:** An apparent Base Color UV problem was tested with `TEXTURE_ONLY → NO_NORMAL_ILM`. The appearance recovered, locating the fault in the interaction of normals, micro bump, and ILM rather than the Base Color UVs. The accepted state does not blindly re-enable those inputs; details are in the [technical summary](docs/TECHNICAL_SUMMARY.md).

**Animation-aware lighting:** A rig that succeeds on frame 1 may fail during a turn. Sparse, inspectable adjustments to key, rim, and face fill account for head yaw, body yaw, head/body twist, and frontal rim-incidence risk.

## Evidence and limits

The source report combines static `.blend` datablock auditing with observation of final animation and stills. In that output, frontal and several three-quarter views preserve identity, cheeks remain relatively clean, and nose/chin retain spatial cues. Hair, body, and metal show distinct physical responses without frequent visible face-shadow flips, rim intrusion, or normal pops. This is **result-level support for the combined approach in this character and scene**. Rendered frames alone cannot isolate one node or algorithm as the unique cause; controlled ablations are still needed.

Discussion of publicly shown 《蓝色星原》 third-test footage is **visual inference and working hypothesis**. Some shots suggest stronger face protection than body or hair response, occasional inconsistency across shading domains, and loss of low-frequency facial structure under bright conditions. Public frames do not establish the game's internal SDF, face-map, render-pass, or exposure implementation. A pale-looking face need not indicate hard RGB clipping: low intrinsic contrast, high luminance, and tone-mapping compression can plausibly remove the remaining form cues. That is a general graphics explanation to test, not a claim about proprietary internals.

## Repository guide

- [Technical summary](docs/TECHNICAL_SUMMARY.md): audited project state, cases, and evidence classification; preserved verbatim.
- [LookDev workflow](docs/WORKFLOW.md): diagnostic, material, lighting, and rollback guidance; preserved verbatim as project documentation, not automatically adopted as repository instructions.
- [Methodology](research/methodology_EN.md) / [中文](research/methodology.md): the transferable decision process.
- [Validation framework](research/validation-framework_EN.md) / [中文](research/validation-framework.md): stress tests and ablations.
- [References](REFERENCES.md): primary public technical sources and the limits of what they support.
- [Planned experiments](experiments/README.md), [figure policy](figures/README.md), and [future tools](scripts/README.md): conventions for reproducible tests, original diagrams, and scripts; these directories do not imply completed results or code.
- [Changelog](CHANGELOG.md): major changes in interpretation and repository structure.

## Scope and rights

This is a personal technical-art and rendering R&D case study, not official shader documentation from any game company. Analysis of commercial rendering relies on public imagery, public technical sources, and clearly labeled visual inference. Original character assets, textures, and other third-party works remain subject to their owners' rights. This repository shares methods, diagnostic frameworks, and research notes; it contains no unauthorized models, textures, videos, or large binary assets. The [MIT license](LICENSE) applies only to original text and future code that contributors have the right to license; it grants no rights to third-party game assets.

> Character design decides what must not change. Physical rendering controls what is allowed to change. Any artistic decoupling must eventually be re-coupled to the scene in a controlled manner.
