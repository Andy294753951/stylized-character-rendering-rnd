# References

These are public technical sources for the concepts discussed in this repository. They establish available methods and production precedents; they do **not** identify the internal rendering architecture of 《蓝色星原》 or prove which component caused the observed result in this project. The project's own audit and rendered observations are recorded in [Technical summary](docs/TECHNICAL_SUMMARY.md); proposed tests are in the [Validation framework](research/validation-framework_EN.md).

## Blender / Cycles

- **Normal Map Node** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/render/shader_nodes/displacement/normal_map.html). Reference for tangent-space normal input and its UV/color-space requirements; a source texture's channel meaning still requires inspection.
- **Tangent Node** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/render/shader_nodes/input/tangent.html). Reference for tangent directions used in anisotropic shading; it is not evidence for a particular game's hair shader.
- **Light Path Node** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/render/shader_nodes/input/light_path.html). Documents ray-type outputs such as *Is Camera Ray*, relevant to the audited scene's camera/background separation.
- **Blender Python API: Node and NodeSocket** — Blender Foundation, Blender 5.2 API. [Node](https://docs.blender.org/api/5.2/bpy.types.Node.html) · [NodeSocket](https://docs.blender.org/api/5.2/bpy.types.NodeSocket.html). Primary reference for checking node type, socket type, and link state before automated graph edits.
- **Blender Python API: removing data** — Blender Foundation, Blender 5.2 API. [Troubleshooting guidance](https://docs.blender.org/api/5.2/info_gotchas_crashes.html). Explains why scripts should not access removed RNA data; relevant to safe `NodeLink` mutation in the Jinshi follow-up.

## Stylized rendering and face-shading control

- **Locally Controllable Stylized Shading** — Hideki Todo, Ken-ichi Anjyo, William Baxter, and Takeo Igarashi, SIGGRAPH 2007. [Author-linked publication page](https://research.google/pubs/locally-controllable-stylized-shading/). A primary research precedent for localized, artist-directed light and shade integrated with conventional lighting; it is not this project's implementation recipe.
- **Unity Toon Shader: shader settings** — Unity Technologies, 2023, version 0.9 manual. [Official documentation](https://docs.unity3d.com/ja/Packages/com.unity.toonshader%400.9/manual/Parameter-Settings.html). A public example of separate controls for shading steps, normal maps, rim lighting, and scene-light influence. It supports comparing responsibilities, not inferring proprietary face maps.

**Project-specific formulation:** Head-local light direction and region masks are described as choices in the audited project. The sources above provide related principles, but none is cited as the origin of this exact node graph. Face SDF is a **future candidate**, not a verified component of the current project or of 《蓝色星原》.

## Normal editing and texture interpretation

- **Normal Edit Modifier** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/modeling/modifiers/normals/normal_edit.html). Documents custom-normal editing and partial mixing, including stylized shading uses; relevant to regional control.
- **Data Transfer Modifier** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/modeling/modifiers/modify/data_transfer.html). Documents transferring mesh data, including custom normals, relevant to the audited normal workflow.
- **DirectXTex `texconv`** — Microsoft, maintained project documentation. [Official tool reference](https://github.com/microsoft/DirectXTex/wiki/texconv). Documents normal-map operations including Z reconstruction and Y inversion. These options do not establish the semantics of any particular imported texture.

## Color management and tone mapping

- **Displays and Views** — Blender Foundation, Blender 5.2 Manual. [Official documentation](https://docs.blender.org/manual/en/5.2/render/color_management/displays_views.html). Reference for view transforms and exposure controls used when testing whether facial contrast survives bright conditions.
- **Blender 4.0 Color Management release notes** — Blender Foundation, 2023. [Official release notes](https://developer.blender.org/docs/release_notes/4.0/color_management/). Documents the introduction of AgX and its treatment of over-exposed color. The repository's “pale face” explanation remains a hypothesis to test with linear and displayed values.

## Related production talk

- **Guilty Gear Xrd’s Art Style: The X Factor Between 2D and 3D** — Junya C. Motomura, Arc System Works, GDC 2015. [Studio announcement with original talk and handout](https://www.arcsystemworks.com/guilty-gear-xrds-art-style-the-x-factor-between-2d-and-3d-talk-from-gdc-2015-is-now-available-online/). A primary production example of deliberate 2D-style character design in a 3D pipeline; it is not a source on 《蓝色星原》.
