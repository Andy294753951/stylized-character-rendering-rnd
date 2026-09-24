# Stylized character rendering validation framework

[中文](validation-framework.md) · [Home](../README_EN.md) · [Methodology](methodology_EN.md)

The objective is a character rendering system that survives its **weakest tested condition**. This is a proposed reproducible protocol; it does not claim that the original project completed every test in this matrix. The existing audit and output observations are documented in the [technical summary](../docs/TECHNICAL_SUMMARY_REVISED_v4.md).

## Baseline and records

For each comparison, hold pose, camera, lights, environment, renderer, color management, resolution, and random seed fixed except for the variable under test. Record the `.blend` version, material and modifier switches, light and exposure settings, shot and frame numbers, output images, observer notes, and rollback path. Compare matched crops of face, neck, hair, and body. Mark conditions as **untested**, **pass**, or **fail** rather than turning an impression into an engineering fact.

## Stress-test matrix

| Axis | Minimum samples | Inspect |
| --- | --- | --- |
| View angle | 0°, 30°, 45°, 60°, 90°; both sides where asymmetry matters | Continuity of eyes, cheeks, nose, chin, and silhouette |
| Lighting | Front, side, top, back, rim, mixed | Face-shadow flips, rim intrusion, conflicting material-domain responses |
| Environment | Neutral, warm, cyan/blue, green, red, dark, bright | Bounded color and luminance integration without identity loss |
| Exposure | EV −2, −1, 0, +1, +2 | Loss of nose/cheek/forehead form cues in bright output |
| Pose and animation | Head tilt, head/body twist, partial hair occlusion, arm near face, expression changes; dense samples around turns | Shadow/mask/normal pops, rim intrusion, expression readability |

First scan one axis at a time and identify risky combinations. Then cross-test high-risk views, light directions, environments, and exposures. Single-axis coverage does not imply coverage of every combination. Inspect both turn directions and temporal continuity.

## Face ablation

Use identical pose, camera, lighting, environment, and exposure for each version:

| Variant | Configuration | Question |
| --- | --- | --- |
| A | Pure PBR / geometry normals | How does the unprotected baseline fail? |
| B | Uniform Normal Flatten | Does global flattening help front view while damaging spatial cues? |
| C | Regional Normal Control | Can local normals clean cheeks and retain nose/chin volume? |
| D | C + Designed Face Shading | Does designed shading improve identity and cross-view stability? |
| E | D + Environment Adaptation | Does scene integration improve without erasing identity? |

Log active nodes, modifiers, and masks for every variant. A difference between D and C can be attributed only to the controlled change between those variants; a final image alone does not identify a single node as the unique cause.

## Evaluation

Score each criterion from **0 (fails) to 4 (stable)** and attach a crop and specific failure description. Define subjective anchors before scoring and have a second observer blindly review high-risk samples when possible. Scores are judgments, not objective pixel ground truth.

| Criterion | What to examine |
| --- | --- |
| Character Identity Preservation | Eyes, face design, expression, and core color still read as the same character |
| Spatial Coherence | Nose, chin, and head orientation agree with the scene's 3D structure |
| Face/Body/Hair Shading-domain Coherence | Domains appear lit by the same environment and lights |
| Cross-view Stability | Structure transitions smoothly across left/right and frontal/profile views |
| Temporal Stability | No shadow, mask, or normal pops; no sudden rim intrusion |
| Worst-case Identity Preservation | The lowest-scoring condition still reads as the intended character |
| Exposure Robustness | Facial structure survives EV +1/+2 |
| Environment Integration | Scene hue and luminance affect the character within identity-safe bounds |

Report the **minimum score** for each criterion, its condition and frame, and then the median. Do not select only the most flattering frame. If a change raises the average but damages worst-case identity or temporal stability, keep it experimental rather than promoting it.

## Testing the “pale face” hypothesis

A plausible chain is:

```text
Stylized face → reduced geometry-driven shading → low local contrast
→ high face/scene luminance → exposure → tone-mapping highlight compression
→ remaining nose/cheek/forehead cues collapse → pale or flat-looking face
```

`Pale Face ≠ necessarily hard RGB clipping`. In an owned, controlled scene, keep texture, normals, lights, and camera fixed while scanning exposure and tone mapping. Compare pre-display linear luminance, local contrast in the output, and actual channel saturation. Then vary face protection and environmental luminance separately. If local structure collapses without saturation, that supports the low-contrast/high-luminance/tone-mapping explanation; if hard clipping occurs, record it as another mechanism. For public 《蓝色星原》 imagery this remains a **visual working hypothesis**, never confirmation of proprietary internals.

## Reporting claims

Label each conclusion as an **engineering fact**, **output observation**, **public technical precedent**, or **working hypothesis**. Include test version, conditions, weakest frame, negative results, and untested combinations. Controlled A/B comparisons can support component-level causal claims; final footage supports the combined outcome in this particular case.
