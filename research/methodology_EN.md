# Methodology: bounded choices in character rendering

[中文](methodology.md) · [Home](../README_EN.md) · [Validation framework](validation-framework_EN.md)

This method abstracts decisions from the [technical summary](../docs/TECHNICAL_SUMMARY_REVISED_v4.md) and [LookDev workflow](../docs/SKILL_optimized_v1.1.md). It is a decision process for Blender/Cycles, not a universal node graph or parameter preset.

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

## Evidence discipline

Classify claims as **engineering facts** (datablocks, nodes, scripts, settings), **output observations** (visible in the final render or animation), **public technical precedent** (general methods documented elsewhere), or **working hypotheses** (inferences about proprietary rendering). Final footage supports the combined result in this particular case; it does not identify one algorithm as its unique cause. Public frames cannot establish a game's internal shader architecture.

## Promotion criterion

Run the controlled tests in the [validation framework](validation-framework_EN.md) before making a correction reusable. Compare identity preservation, spatial and shading-domain coherence, cross-view and temporal stability, exposure robustness, and environmental integration. Record the weakest sample and its failure mode. Promote a correction only when it improves that envelope and remains reversible.
