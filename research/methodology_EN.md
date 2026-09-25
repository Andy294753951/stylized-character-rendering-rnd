# Methodology: bounded choices in character rendering

[中文](methodology.md) · [Home](../README_EN.md) · [Validation framework](validation-framework_EN.md)

This method abstracts decisions from the [technical summary](../docs/TECHNICAL_SUMMARY.md) and [LookDev workflow](../docs/WORKFLOW.md). It is a decision process for Blender/Cycles, not a universal node graph or parameter preset.

```text
Stylized Character Rendering
= Physical Spatial Coherence
+ Bounded Identity Protection
```

## Seven steps

1. **Identify the visual invariant.** State which eye shapes, expression marks, cheek forms, nose-shadow relationships, silhouettes, and core colors define the character. Label whether each comes from source design, the audited project, or observed output.
2. **Identify the physical degrees of freedom.** Decide by region and material which properties may vary with lighting, view, pose, and environment. Eyes and expression cues may need strong protection; hair, cloth, and metal need more physical response. Avoid a character-wide PBR/NPR ratio.
3. **Locate the earliest failing pipeline layer.** Test geometry/topology → UV/source texture → channel decoding → normal/tangent → BSDF → designed NPR response → lighting → camera/DOF → visibility → compositor/color management. Isolate one variable at a time.
4. **Apply the smallest bounded correction.** Fix only the responsible region or property and define its permitted range. Soften cheeks where useful while preserving nose and chin volume. Reject a change that improves one view at the expense of three-quarter or temporal stability.
5. **Re-couple the result to the environment.** `Decouple → Art Direction → Controlled Re-coupling`. A protected face still needs bounded responses to environment hue, luminance, exposure, and scene lighting. Inspect chromatic and luminance adaptation separately so the face does not appear lit by another studio.
6. **Stress-test the worst case.** Sample angles, light directions, environments, exposure, pose, and time. Compare controlled ablations under identical conditions. Ask whether the weakest frame still reads as the same character in the same world.
7. **Preserve rollback and reproducibility.** Record inputs, material and lighting states, reasons for edits, A/B output, and failure cases. Keep the accepted baseline, duplicate risky states, and maintain a restore path so historical experiments cannot silently replace the active setup.

## Follow-up method: Shading Responsibility Allocation

The [Jinshi follow-up](../docs/JINSHI_RESPONSIBILITY_CASE_STUDY.md) adds an **operational diagnostic axis** without replacing `Physical Spatial Coherence + Bounded Identity Protection`. After finding the earliest failing layer, ask who should primarily own each visible cue and how much physical change it may tolerate.

For nose shadow, cheek and eye-socket form, environment tint, and rim separation, record the **primary owner**, allowed physical freedom, protected identity envelope, and secondary contributors. Geometry carries silhouette; regional normals can suppress unwanted geometry response; designed face shading can shape identity-critical boundaries; PBR supplies material and spatial response; lights and environment support scene belonging. The exact allocation depends on the current asset and controlled tests.

**Redundant Volume Encoding** is the risk that several systems strongly repeat the same nose, eye-socket, or cheek form cue. Each part may be reasonable by itself while the combined result over-models the face. Isolate one responsibility layer at a time before making the smallest useful change; a problem visible on the face does not automatically call for a face-material rewrite. Jinshi's numerical settings remain case-specific notes, not reusable defaults.

## Evidence discipline

Classify claims as **engineering facts** (datablocks, nodes, scripts, settings), **output observations** (visible in a render or animation), **public technical precedent** (general methods documented elsewhere), **methodological inference** (a case-derived principle still needing wider testing), or **working hypotheses** (inferences about proprietary rendering). Jinshi's engineering events are currently supplied case notes, not independently audited repository evidence. Final footage supports the combined result in a particular case; it does not identify one algorithm as its unique cause. Public frames cannot establish a game's internal shader architecture.

## Promotion criterion

Run the controlled tests in the [validation framework](validation-framework_EN.md) before making a correction reusable. Compare identity preservation, spatial and shading-domain coherence, cross-view and temporal stability, exposure robustness, and environmental integration. Record the weakest sample and its failure mode. Promote a correction only when it improves that envelope and remains reversible.
