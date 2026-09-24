# Planned experiments

These are **future reproducible experiments**. The original project has **not** completed this entire matrix, and no result is asserted here. Use the [validation framework](../research/validation-framework_EN.md) for evaluation criteria and report format.

Hold the asset, pose, camera, render settings, and unrelated material parameters fixed for each single-variable comparison. Record the Blender version, source revision, scene, view layer, camera, frame, changed parameter, lighting/world, color management, render settings, output paths, and a way to restore the baseline. Report failures as well as successful views. Use only assets and images that may be redistributed.

| Directory | Planned sweep |
| --- | --- |
| `face-ablation/` | A: Pure PBR / geometry normals; B: Uniform Normal Flatten; C: Regional Normal Control; D: C + Designed Face Shading; E: D + Environment Adaptation. State which variants can actually be reproduced before comparing them. |
| `view-angle/` | 0°, 30°, 45°, 60°, 90°; test both left and right when asymmetry matters. |
| `lighting/` | Front, side, top, back, rim, and mixed lighting. |
| `environment/` | Neutral, warm, cyan/blue, green, red, dark, and bright environments. |
| `exposure/` | EV −2, −1, 0, +1, +2 with the view transform recorded. |

Store future run notes beside the corresponding experiment, with links to owned inputs and outputs. A proposed matrix is not evidence; promote a conclusion only after controlled comparison and cross-view checks.
