<!--
SPDX-FileCopyrightText: 2026 shadPS4 Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# GVS graphics investigation TODO

This note preserves the current evidence and next steps for the intermittent stretched or black
geometry, missing effects, and low guest-frame progression observed in CUSA08379. It is a research
checkpoint, not a compatibility claim or a title-specific workaround.

## Reproduction baseline

- Branch: `fix/asc-pm4-fragments`.
- Build: Windows `x64-Clang-Debug`.
- Host GPU: NVIDIA GeForce RTX 4070.
- Capture tool: RenderDoc 1.45, injected before Vulkan instance creation.
- The committed ASC PM4 fragment fix prevents the earlier compute-queue hang. The graphics defects
  described here remain a separate investigation.

Do not use Debug or RenderDoc capture timing as a 60 FPS benchmark. The RenderDoc overlay measures
host presents; it does not prove that the guest produced a new game frame. Compare ordinary runtime
performance with `x64-Clang-RelWithDebInfo`, shader dumping disabled, and unrelated Vulkan overlays
disabled.

## Confirmed evidence

### Capture boundaries

An external target-control capture covered one host present and contained 77 compute dispatches,
153-154 copies, and only the two ImGui draws. It therefore did not contain a game graphics
submission even though the process and Vulkan target were valid.

The useful capture was triggered through shadPS4's F12 path. shadPS4 brackets the next Liverpool GPU
submission with RenderDoc's in-application `StartFrameCapture` and `EndFrameCapture` calls. That
capture contained:

- 236 draws, 178 dispatches, 318 copies, and 52 clears;
- 140 textures and 325 buffers;
- no RenderDoc debug messages or pipeline inspection errors.

The captured frame was the main menu. Exported G-buffer, HDR, game, scaled-frame, and present images
were visually coherent and did not contain the reported black or stretched geometry. This capture
validates the tooling, but it does not reproduce the defect.

### Geometry

Automated post-VS inspection reported several draws with extreme NDC coordinates. The largest two
groups contained 360-380 huge-triangle candidates. Exporting the affected render targets showed a
correct main-menu robot and sky: the extreme vertices were outside the visible region and were not
proof of corruption. Future automated checks must combine mesh statistics with the affected draw's
visible render target or pixel history.

The suspect main-menu vertex and index buffers were static cached resources. RenderDoc reported
only vertex/index usages for them in the captured frame, with no preceding compute or transfer
write. They do not validate a compute-to-vertex synchronization theory.

### Textures and effects

The capture contained successfully created and bound BC3, BC4, BC5, BC6H, and BC7 resources,
including sRGB and unorm variants. Shader translation diagnostics reported no failed translations.
This rules out the simple explanation that all GVS effects are absent because shadPS4 does not
recognize the compressed texture formats. It does not yet prove that every guest tiling, mip,
swizzle, alpha, or storage-image transition is correct.

### Mixed buffer-access experiment

Temporary logs exposed a plausible general synchronization hazard: one compute shader can bind
disjoint guest ranges that resolve to the same merged host `VkBuffer`. Processing descriptors one
at a time lets a later shader read replace an earlier shader-write access mask for that host buffer.
Observed examples included skinning shader `0xb55630b5` and effect shader `0x52c24837`.

An experiment aggregated descriptor access masks per host buffer and forced a dependency after a
previous shader write. It produced no confirmed visual improvement. Because the current cache tracks
one state for an entire merged buffer, the forced barrier was also potentially over-broad and could
reduce performance for disjoint ranges. The experiment and all `[GFX-DIAG]` probes were removed from
the saved branch. Revisit this only if a defect capture links a visible draw to such a write, and
prefer a range-aware dependency model.

## Ruled-out experiments

- Passing guest buffer sizes through dynamic vertex binding did not improve the defect and was
  reverted.
- Changing sampler LOD limits and bias behavior did not improve missing effects and was reverted.
- SharePlay and unsupported clock log messages are guest-thread liveness markers, not demonstrated
  graphics causes.
- Disabling video playback changes required behavior and does not address the graphics issue.
- A loaded RenderDoc control connection alone is insufficient; capture initialization and the
  intended GPU submission boundary must both be verified.

## Next capture checklist

1. Start the same content and configuration with RenderDoc injected at launch.
2. Enter a 3D battle and wait until stretched/black geometry or a known missing effect is visible.
3. Press F12 once, or send the configured F12 hotkey to the focused shadPS4 window. Do not use an
   arbitrary present-to-present trigger for this comparison.
4. Wait for the `.rdc` file size to stabilize, then export the game framebuffer and confirm that the
   defect is present in the capture.
5. Use pixel history on the black or missing-effect region to find the last contributing draw.
6. Inspect that draw's VS/GS/DS output, index buffer, vertex buffers, descriptors, blending, depth,
   and preceding dispatch/copy usages.
7. For missing effects, determine whether the draw is absent, has a zero/incorrect indirect count,
   samples the wrong mip/view, produces zero alpha, fails depth, or never receives storage-image
   output.
8. Change one demonstrated invariant at a time. Re-capture the same battle state and retain a fix
   only when the before/after frame evidence and runtime behavior agree.

## Verification before proposing a graphics fix

- Run Clang Format 19 and `git diff --check`.
- Build `shadps4` successfully with `x64-Clang-Debug`.
- Run focused tests when a stable seam exists; otherwise document why runtime capture is required.
- Compare `x64-Clang-RelWithDebInfo` without RenderDoc or shader dumping for performance.
- Reproduce beyond the old failure interval with advancing guest frames and responsive input.
- Test another GPU vendor when available, or state that vendor coverage is missing.
- Keep captures, exported images, logs, shader dumps, game files, and local paths out of commits.
