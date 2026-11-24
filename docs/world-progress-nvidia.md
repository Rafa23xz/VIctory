# World rendering on an NVIDIA host (RTX 4080 SUPER): progress notes

Branch `rafa/world-progress` of this fork. Everything here was measured on one
machine (Intel i7-13700KF, RTX 4080 SUPER 16 GB, driver 621.24, Windows 11
26300), with runs of 4 to 7 minutes from boot into the open world. No title
data is in this branch: no shader code, hashes, addresses or offsets.

## Where it stands

- The title boots, plays its intro and reaches the world; runs of 240 to 420 s
  in the world now finish without a device removal (they used to end in
  `DXGI_ERROR_DEVICE_HUNG` / `INVALID_CALL` within 80 to 260 s).
- **The world's geometry rasterizes correctly.** With
  `VICTORY_DIAG_FLAT_PS=1` (every color mesh pipeline gets a pixel shader that
  writes 1.0) and `VICTORY_DIAG_NODEPTH=1`, the G-buffer shows the city with
  correct geometry and perspective. Mesh shaders, the GPU culling passes,
  indirect arguments and pipeline state are working for a large part of the
  scene.
- **The screen is still black** because the title's own pixel shaders leave
  nothing in the G-buffer (the albedo target ends every frame all zero, also
  with depth testing off). That is the open problem; see "Next".
- ~5 s per frame in the world (host copies of title memory and write-watch
  queries dominate); memory ~26 GB of process commit (the NVIDIA driver
  charges every D3D12 resource, VRAM included, to the process commit).

## What this branch contains

On top of `main` (`ffe310e`), squashed into one commit because the local
history this came from also held title data. Upstream pull requests taken in
whole or in part (thanks to their authors): #55, #89, #135, #136, #189, #193,
#205, #206, #261, #269, #305, #432, #439 (with #233 and #320), #586 (only the
mesh allocation counts), #638. #439 replaces #189's compute path on 32-lane
hosts; #189's mesh emulation stays.

Local changes (all marked `local:` in the code):

1. **Host validates translated DXIL** (`pipeline.rs`, `validate_host_dxil`).
   The host signs DXIL with its own hash, so nothing checked it before the
   driver. A translation the validator refuses for what it does ("operation
   incompatible with the shader stage", bad control flow, ...) is now refused
   like a shader that does not translate, with the validator's words.
   Refusals only about feature flags (`Flags must match usage`) are logged and
   let through. `VICTORY_VALIDATE_DXIL=0|warn`.
2. **Host copies of title memory stay coherent** (`shadow.rs`, `host.rs`):
   - copy-back asked write-watch only after its whole loop, so a title write
     made while the range was being copied back was taken as already copied;
     it now asks before the first read;
   - a page a copy could not read got zeros and was marked copied forever; a
     first copy now gets zeros there and is tried again, and a page copied
     before keeps its copy where a read fails (it may hold what shaders
     wrote).
3. **Buffer view guards**: views D3D12 removes the device for (no elements,
   past the end of the buffer, constant views over 64 KiB or misaligned) are
   cut to the host copy or made null, with a warning; this ended the
   `INVALID_CALL` removals when the world's culling passes started to run.
4. **Console pipeline syncs are UAV barriers** on the host list (they were
   no-ops), which ended the mesh-dispatch device hangs seen early on.
5. Texture-to-buffer copies, placed texture release (with an epoch in the
   view slot key), RTV guard, pipeline workers that survive a translator
   panic, and smaller fixes; see `local:` comments.
6. **Diagnostics** (`--features diagnostics` builds):
   - `VICTORY_DIAG_FLAT_PS=1`, `VICTORY_DIAG_NODEPTH=1`: A/B switches above
     (never fixes);
   - indirect executions in captured frames are read back after their list
     ran: what the GPU executed, the title memory before and after;
   - `VICTORY_PRODUCERS=1` now follows root UAV writes and checks the
     arguments and counts indirect executions read;
   - each host mesh pipeline's state (depth, stencil, raster, blend per
     target, formats) is logged.

Shader census (all ~26,000 programs of the title, translated for a 32-lane
host and checked with dxil.dll; counts of validated programs), before ->
after #439 on this base: compute 1522 -> 1736 (+219 / -5), mesh 8372 -> 8576,
vertex 2139 -> 2165, pixel unchanged.

## Findings that may help others

- A diagnostic that reads GPU buffers back must keep its read-back buffer alive
  with the list and read it only after the list ran: lists recorded for a frame
  can run after that frame's present. An early version of ours read too soon
  (it saw zeros) and freed buffers the GPU later wrote (a page fault "inside
  freed").
- About 93% of the world's indirect executions really have zero groups (the
  title memory says so too); no dropped shader writes the mesh argument
  buffers, so this is the title's culling deciding, possibly on bad inputs.
- On 32-lane hosts, #439 still refuses compute programs with console wave64
  lanes in groups of more than 64 threads; one of them feeds about twenty
  indirect dispatches of an atmosphere pass.
- 644 compute translations are refused by the validator as "incompatible with
  the shader stage" (cs_6_6, 32-thread groups, implicit-LOD samples among
  other things); a hand-written HLSL equivalent validates, so it is something
  the emitter declares or uses.

## Next

1. Why the title's G-buffer pixel shaders leave nothing: color export
   (MRT mapping, packed exports), `discard` conditions, mesh-to-pixel attribute
   linkage, and texture/descriptor reads returning zero (an alpha test then
   discards every pixel).
2. The 644 validator refusals in compute.
3. Frame time (host copies, write-watch) and memory (host copies of buffers
   and per-texture staging).
