# Future Blender tools

This directory reserves space for **original** Blender Python tools. It contains no runnable scripts yet: the public documents do not supply enough portable project data to extract and verify a general tool from the original `.blend` history.

- `validation/`: future batch view-angle, exposure, and light-direction sweeps; frame sampling; render-state reporting.
- `diagnostics/`: future active Scene / View Layer / Camera / World checks; material dependency and image colorspace audits; Normal / ILM / Mask channel inspection; compositor passthrough checks.

Tools added later should work without third-party game assets, avoid hard-coded local absolute paths, and never overwrite a `.blend` automatically. Prefer an audit or dry-run mode; explicit APPLY and RESTORE behavior; deterministic names; idempotent changes; and a state report that records exactly what was inspected or modified. Test those claims on a redistributable sample scene before publishing the tool.
