<!--
SPDX-FileCopyrightText: 2026 shadPS4 Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# Capturing shadPS4 with RenderDoc on Windows

This guide describes a reproducible RenderDoc capture workflow for the Windows Vulkan build of
shadPS4. See RenderDoc's official [capture attachment documentation][capture-connection] and
[in-application API documentation][in-application-api] for the underlying behavior.

[capture-connection]: https://renderdoc.org/docs/window/capture_connection.html
[in-application-api]: https://renderdoc.org/docs/in_application_api.html

## Capture requirements

- Use the x64 RenderDoc build with an x64 shadPS4 executable.
- Load RenderDoc before shadPS4 creates its Vulkan instance. Late injection can expose a target
  control connection without intercepting the already-created Vulkan device.
- Keep the game, patches, input configuration, and emulator configuration equivalent to the
  reproduction run. A RenderDoc capture is evidence for one frame, not a title-specific fix.
- Expect capture-time stutter and substantial temporary memory use. Do not use capture performance
  as ordinary runtime performance data.

Set reusable paths in PowerShell, then verify the RenderDoc installation and Vulkan registration:

```powershell
$renderdocDir = "C:\path\to\RenderDoc"
$renderdocCmd = Join-Path $renderdocDir "renderdoccmd.exe"

& $renderdocCmd version
& $renderdocCmd vulkanlayer --explain
```

If RenderDoc reports a mismatched system Vulkan layer, run its registration command from an
elevated PowerShell and verify again:

```powershell
& $renderdocCmd vulkanlayer --register --system
& $renderdocCmd vulkanlayer --explain
```

The final command must report that the RenderDoc Vulkan layer is correctly registered.

## Verify the layer implementation, not only its name

Multiple manifests can advertise the same Vulkan layer name. In particular, some Mirillis
ActionRecorder installations advertise `VK_LAYER_RENDERDOC_Capture` while loading
`MirillisActionVulkanLayer.dll`. A name-only check can therefore produce a false positive.

Use Vulkan Loader diagnostics to verify the DLL selected for both the instance and device chains:

```powershell
$env:ENABLE_VULKAN_RENDERDOC_CAPTURE = "1"
$env:DISABLE_MIRILLIS_LAYER = "1"
$env:VK_LOADER_LAYERS_DISABLE = `
    "VK_LAYER_NV_*,GalaxyOverlayVkLayer*,VK_LAYER_EOS_Overlay," + `
    "VK_LAYER_OBS_HOOK,VK_LAYER_RTSS,VK_LAYER_VALVE_*"
$env:VK_LOADER_DEBUG = "error,warn,layer"

vulkaninfo --summary 2>&1 |
    Select-String "Insert instance layer|Inserted device layer|VK_LAYER_RENDERDOC_Capture"

Remove-Item Env:VK_LOADER_DEBUG
```

Both insertion messages must resolve to the intended RenderDoc directory and `renderdoc.dll`.
Reject a run that resolves the layer name to another DLL.

Do not use `VK_LOADER_LAYERS_DISABLE=~implicit~` for this workflow. RenderDoc is itself an implicit
Vulkan layer, so that blanket filter disables the component needed for capture. Use targeted layer
filters and product-specific disable environment variables instead.

## Launch shadPS4 under RenderDoc

Launching through `renderdoccmd capture` loads RenderDoc before graphics initialization and lets
shadPS4 acquire the in-application API. Replace every placeholder below with the local path for the
test run:

```powershell
$renderdocCmd = "C:\path\to\RenderDoc\renderdoccmd.exe"
$shadPS4 = "C:\path\to\shadps4.exe"
$workingDir = "C:\path\to\portable-shadps4-root"
$eboot = "C:\path\to\game\eboot.bin"
$gameRoot = "C:\path\to\game"
$captureTemplate = "C:\path\to\captures\shadps4-renderdoc"

$env:ENABLE_VULKAN_RENDERDOC_CAPTURE = "1"
$env:DISABLE_MIRILLIS_LAYER = "1"
$env:VK_LOADER_LAYERS_DISABLE = `
    "VK_LAYER_NV_*,GalaxyOverlayVkLayer*,VK_LAYER_EOS_Overlay," + `
    "VK_LAYER_OBS_HOOK,VK_LAYER_RTSS,VK_LAYER_VALVE_*"

$arguments = @(
    "capture"
    "-d", $workingDir
    "-c", $captureTemplate
    $shadPS4
    "--game", $eboot
    "--override-root", $gameRoot
)

Start-Process -FilePath $renderdocCmd -ArgumentList $arguments -WorkingDirectory $workingDir
```

When the injected API is available, shadPS4 sets its own capture template under the active user
directory's `captures` folder. The path is reported as `RenderDoc capture path` in the shadPS4 log.

## Prove that Vulkan capture is active

Do not treat a loaded `renderdoc.dll`, an open target-control port, or an attachable process as
sufficient proof. Check the target's RenderDoc log under `%TEMP%\RenderDoc`. A valid Vulkan setup
contains all of these messages:

```text
Adding Vulkan device frame capturer
Initialised capture layer in Vulkan instance
Adding Vulkan frame capturer
```

The RenderDoc status overlay should also appear in the game window. If the GUI can attach but its
capture controls are disabled and no overlay appears, the control DLL is loaded but the Vulkan
layer is not active.

## Capture with F12

shadPS4 disables RenderDoc's default capture keys and handles the configured capture hotkey itself.
By default, press `F12` once while the game window has focus. This asks shadPS4 to capture across its
next GPU submission boundary instead of using an arbitrary present-to-present range.

Wait until the overlay reports that one capture was saved. A large frame can stall for several
seconds. Do not press F12 repeatedly, close the game, or open the destination `.rdc` while it is
still being written.

Only open the capture in the RenderDoc GUI after the save has completed. Keeping an older capture
with the same name open can lock the destination. RenderDoc then logs `errno 13`, discards the newly
recorded frame, and leaves the overlay at `0 captures saved`.

### Prefer shadPS4's submission boundary

RenderDoc target control can request a present-to-present capture from an injected process, but a
host present does not necessarily contain a guest graphics submission. It may contain only async
compute, resource copies, and UI rendering while an older game image is presented again.

For GPU command-stream investigations, use the configured F12 path. shadPS4 starts the capture when
the Liverpool command processor begins draining submitted work and ends it at the corresponding
submission boundary. Use target control primarily to verify connectivity or when a host-present
capture is specifically required.

## Validate the capture

First confirm that RenderDoc logged all of the following for the target process:

```text
Starting capture
Finished capture
Captured Vulkan frame
Written to disk
```

Then validate the resulting file without relying only on the GUI:

```powershell
$capture = "C:\path\to\capture.rdc"
$thumbnail = "C:\path\to\capture-validation.png"

& $renderdocCmd thumb --out=$thumbnail --format=png --max-size=1280 $capture
& $renderdocCmd replay --width=640 --height=360 --loops=1 $capture
```

The thumbnail must contain the intended game frame, and the one-loop replay must exit successfully.
An unusually tiny file, a missing thumbnail, or a replay failure is not a valid capture.

## Troubleshooting

| Symptom | Check and action |
|---|---|
| No RenderDoc overlay | Confirm launch-time injection and the three Vulkan frame-capturer log messages. |
| Attach works but capture controls are disabled | RenderDoc control is present, but Vulkan was not intercepted. Restart with the correct layer active. |
| Loader inserts `MirillisActionVulkanLayer.dll` | Set `DISABLE_MIRILLIS_LAYER=1` and repeat the implementation-path check. |
| F12 produces no capture | Confirm `hotkey_capture_frame = f12`, window focus, and the Vulkan frame-capturer messages. |
| Overlay reports `0 captures saved` | Search the target RenderDoc log for errors. Close the GUI if it holds the destination file open. |
| Log reports `errno 13` | Release the old `.rdc` file handle. If necessary, stop the target before moving a verified invalid capture aside, then restart the capture run. |
| A tiny D3D11 capture appears | An overlay or recorder device was captured instead of shadPS4's Vulkan swapchain. Disable unrelated layers and reject the file. |
| Capture pauses for several seconds | This is expected for large frames; wait for `Written to disk` before interacting with the file. |

## Evidence checklist for automated or AI-assisted runs

- Record the RenderDoc version, shadPS4 executable, build type, GPU, and reproduction state.
- Record `vulkanlayer --explain` and the actual instance/device layer DLL paths.
- Confirm that no unrelated Vulkan capture or overlay layer is loaded in the target.
- Confirm the three Vulkan frame-capturer initialization messages before asking for a capture.
- Confirm all four capture/write messages after F12.
- Export and inspect a thumbnail, then complete a one-loop local replay.
- Keep `.rdc` files, thumbnails, RenderDoc logs, game data, and local paths out of commits.
- Remove temporary Loader environment variables and diagnostic files after the investigation.
