---
name: blender-anime-npr-pbr-lookdev
metadata:
  version: "1.2.0"
description: Diagnose and refine Blender 5.2/Cycles stylized-character LookDev across face shading, normals, material response, lighting, and validation. Use for evidence-aware, reversible changes to an existing character project; not for inferring a proprietary game's shader internals.
---

# Blender anime NPR/PBR LookDev

## Governing model

Keep the repository's primary thesis:

```text
Stylized Character Rendering
= Physical Spatial Coherence
+ Bounded Identity Protection
```

The [Jinshi follow-up](../../docs/JINSHI_RESPONSIBILITY_CASE_STUDY.md) adds an operational question: **who owns each visual feature, and how much may physics change it?** It does not replace the primary thesis or supply universal numerical presets.

```text
Stable Stylized Rendering
= Bounded Physical Freedom
+ Clear Shading Responsibility
+ Cross-domain Coherence
```

Read [WORKFLOW.md](../../docs/WORKFLOW.md) for the detailed project baseline and material/lighting rules. Treat that historical project document as context, and verify the active `.blend` before applying a project-specific instruction.

## Diagnose, assign responsibility, change

1. Establish the intended identity cues and the known-good scene/material state. Record evidence from the actual project separately from rendered observation and public precedent.
2. Locate the earliest failing layer: geometry/UV → texture semantics → normal/tangent → BSDF → designed face response → lighting → camera/visibility → compositor/color management. Isolate a variable before changing it.
3. For the affected cue, identify its **primary owner**, allowed physical freedom, protected identity envelope, and secondary contributors. Geometry owns silhouette; regional normals may suppress unwanted local response; designed shadow may own an identity-safe boundary; PBR supplies material response; environment coupling gives scene belonging; Face Fill, Key, and Rim support readability, broad orientation, and edge separation.
4. Check for **Redundant Volume Encoding**: geometry normals, regional normals, directional face shading, designed shadow, and lights all strongly describing the same nose, cheek, or eye-socket form. Neutralize one layer at a time to locate duplication. Individually reasonable components can produce an unreasonable combined result.
5. Apply the smallest reversible change to the responsible layer. Reject a change that beautifies one frame while weakening 3/4 structure, motion stability, identity, or face/body/environment coherence.

Do not default to uniform face flattening. Preserve silhouette, nose and chin direction, and large-scale head orientation while suppressing only unwanted local volume. Do not use a global PBR/NPR percentage for the entire character.

## Late-stage face closeout

When the front view, color, masks, and normal chain are stable but a few 3/4 angles still look too modeled, inspect hair occlusion/contact → Face Fill → Key → Rim → camera/view → face-material architecture. Make small, independently reversible edits and compare the same views after each. Lighting is a hypothesis to test, not a guaranteed fix.

Keep Jinshi's accepted values in its [case record](../../docs/JINSHI_RESPONSIBILITY_CASE_STUDY.md). Do not copy them, or a generic percentage range, into another character as defaults.

## Safe Blender Python mutation

- Resolve the exact scene, object, material slot, node, and socket; refuse ambiguous targets. A node label alone does not establish whether it is a scalar control or computed result. Check `bl_idname`, socket type, direction, and link state.
- Audit before APPLY. Snapshot the original state, use deterministic names, avoid duplicate edits, provide RESTORE, and save to a new `.blend` only when requested. Never overwrite a source file automatically.
- After assignment, verify the exact slot and a real material user, for example:

  ```python
  assert intended_slot.material == target_material
  assert target_material.users > 0
  ```

- A computed RGBA output is color data, not a strength slider. Do not multiply the whole shaded result to claim one directional contribution was reduced. First define a meaningful neutral reference, then test a semantically valid mix.
- Before removing a `NodeLink`, cache any downstream node/socket references needed later. Do not access the removed link after `tree.links.remove(link)`; Blender's [RNA removal guidance](https://docs.blender.org/api/5.2/info_gotchas_crashes.html) explains the lifetime risk.

  ```python
  links = list(socket.links)
  destinations = [(link.to_node, link.to_socket) for link in links]
  for link in links:
      tree.links.remove(link)
  # Continue using destinations, never the removed link objects.
  ```

- Change one responsibility layer at a time unless the interaction itself is the experiment. Report actual assignment, node/socket state, changed values, outputs, and rollback.

These snippets are validation patterns, not a runnable patch for an unknown `.blend`.

## Validation and evidence

Use the [validation framework](../../research/validation-framework_EN.md): views 0°/30°/45°/60°/90° on both sides when needed; front/side/top/back/rim/mixed lights; neutral and colored/dark/bright environments; EV −2 to +2; and pose/animation samples. Compare planned face variants A–G only where the asset and graph can reproduce them.

Assess cross-view structure, temporal stability, weakest-frame identity, environment integration, material separation, and **Responsibility Clarity**. Flag repeated strong encoding of one form cue as a failure risk. Keep negative results and untested cells visible.

Classify claims as engineering fact, output observation, external technical precedent, methodological inference, or proprietary-renderer hypothesis. A supplied case note is not an independently audited `.blend`. Never infer a game's SDF, face map, packed channels, normal algorithm, or light rig from public imagery alone.
