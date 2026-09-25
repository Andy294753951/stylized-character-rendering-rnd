# Jinshi follow-up: shading responsibility allocation

[Home](../README_EN.md) · [Methodology](../research/methodology_EN.md) · [Validation framework](../research/validation-framework_EN.md)

## Status and evidence boundary

This is a follow-up Blender 5.2/Cycles case, separate from the earlier 米砂 engineering audit in the [technical summary](TECHNICAL_SUMMARY.md). It is condensed from the Jinshi case notes supplied for this repository update. The `.blend`, scripts, state reports, and renders were **not** supplied for an independent audit here. Accordingly, the engineering events and image observations below are **author-reported case records**; the broader rules are methodological inferences. The proposed A–G matrix remains a future reproducibility protocol, not a claim that every cell was run.

The repository's primary thesis remains:

```text
Stylized Character Rendering
= Physical Spatial Coherence
+ Bounded Identity Protection
```

The Jinshi notes support that balance at the case level and suggest an operational refinement:

```text
Stable Stylized Rendering
= Bounded Physical Freedom
+ Clear Shading Responsibility
+ Cross-domain Coherence
```

The second expression is a diagnostic aid, **not** a replacement formula or a newly established graphics algorithm.

## Reported observations and interpretation

| Case record | Reported output observation | Limited inference |
| --- | --- | --- |
| Under-protected face | Nose-side modeling, eye-socket depth, and portrait-like cheek transitions became too strong. | Unrestricted geometric and directional form cues can damage this character's intended illustration language. |
| Broad or uniform flattening | A cleaner front view could lose 3/4 nose, chin, and head orientation cues; the face could separate from body shading. | “Flatter face = more anime” is not a general rule. Preserve useful spatial cues while suppressing unwanted local volume. |
| Heavy directional-output suppression | Failed tests looked darker or dirtier, including when an already shaded RGBA result was multiplied. | Lowering the entire color result is not a controlled reduction of one shading contribution. |
| Clean material baseline plus restrained light changes | The supplied notes describe the accepted closeout as retaining the known-good face material and using small Face Fill and Rim adjustments. | A face problem visible in a few 3/4 views may originate in lighting, hair occlusion, or camera rather than require another shader rewrite. |

These observations do not isolate a single node as the cause. The material, lights, view, and occlusion interact; a controlled single-variable comparison is still needed for component-level claims.

## Shading responsibility allocation

For each important visual feature, identify its **primary owner**, allowed physical freedom, protected identity envelope, and permitted secondary contributors. Check whether several systems strongly encode the same form cue.

| Subsystem | Primary responsibility | Guardrail |
| --- | --- | --- |
| Geometry | Silhouette and coarse head form | Do not expect a shader to repair a wrong silhouette. |
| Regional / edited normals | Suppress unwanted local geometric response | Keep nose, chin, and head-direction cues. |
| Designed face shadow, including an SDF **if actually used** | Identity-safe shadow boundary | Do not replace full scene illumination. |
| PBR / BSDF | Material response, restrained depth, highlight, and SSS | Do not let local form overwhelm identity-critical design. |
| Environment re-coupling | Bounded scene color and luminance response | Avoid a pasted or independently lit face. |
| Face Fill / Key / Rim | Readability / broad orientation / edge separation | Avoid independently remodeling nose and cheeks. |
| Hair occlusion/contact | Local, pose-dependent contact cue | Do not substitute a static guide for all real shadowing. |

**Redundant Volume Encoding** is the failure pattern in which geometry normals, edited normals, directional face shading, designed shadow, and character lights each strongly restate the same nose, cheek, or eye-socket form. Individually reasonable components can produce an unreasonable combined result when they repeatedly encode the same cue. One feature may have several contributors, but should have one clear primary owner.

For a stable face that reads too modeled only in some 3/4 angles, test in this order: hair occlusion/contact → Face Fill → Key → Rim → camera/view → face-material architecture. This order is a diagnostic priority, not a promise that lighting is always the cause.

## Project-specific closeout record

The supplied notes report the following **Jinshi-only accepted state**. They are not universal presets, and this update did not verify them against a `.blend` or render-state report:

```text
Face SDF Strength             ≈ 0.40
Environment Color Influence   ≈ 0.18
Environment Luminance         ≈ 0.22

Face Fill Energy              clean baseline × 1.08
Face Fill Area Size           clean baseline × 1.06
Cyan Rim Energy               original baseline × 0.84
```

The transferable lesson is the decision pattern: return to a known-good material, change one responsibility layer at a time, and test small lighting corrections before replacing a mature face architecture.

## Negative engineering lessons reported in the notes

1. **Material built but not assigned.** A generated material with zero users does not patch the intended object. After applying a change, verify the exact object and slot, for example `assert intended_slot.material == target_material` and `assert target_material.users > 0` in Blender Python. A material name or successful node creation is insufficient.
2. **Computed node mistaken for a parameter.** A label does not prove that a node exposes a writable scalar. Inspect `bl_idname`, input/output socket types, links, and whether the node is a control or a computed result before mutation.
3. **RGBA result multiplied as strength.** Multiplying a completed color result can darken the whole face and shift chroma. If attenuation is needed, first establish a meaningful neutral reference and use a semantically valid mix; do not describe full-result multiplication as reducing only directional contribution.
4. **Removed `NodeLink` reference.** Cache downstream node/socket references before calling `tree.links.remove(link)`; do not dereference the removed link. Blender's [Python API removal guidance](https://docs.blender.org/api/5.2/info_gotchas_crashes.html) explains why removed RNA data should not be accessed.
5. **Multiple variables changed together.** Changing directional face, designed shadow, environment coupling, and Face Fill in one closeout test obscures causality. Vary one layer at a time unless their interaction is the explicit test.

These are operational checks for future scripts, not evidence that any proposed script in this repository has been run. The [repository Skill](../skills/blender-anime-npr-pbr-lookdev/SKILL.md) summarizes their use.

## What remains to verify

Use the [validation framework](../research/validation-framework_EN.md) to reproduce A–G with a redistributable asset or a rights-cleared test scene. Record exact material slot, node/socket state, render settings, view/light/exposure/pose, matched outputs, failures, and rollback. Compare one responsibility layer at a time, then stress high-risk combinations. Until those records are available, the supplied numerical state and causal sequence remain case notes rather than independently confirmed repository evidence.

This case makes **no claim** about a proprietary game's SDF, face map, normal algorithm, light rig, or renderer architecture. Public technical sources in [References](../REFERENCES.md) are precedents, not proof of those internals.
