# Adrenaline PSP framebuffer and filtering path audit

## Why this matters

The user wants to reduce jagged edges in PSP games launched as Chovy-Sign LiveArea bubbles, without using Adrenaline. Adrenaline is still useful as a technical reference because its source demonstrates how PSPemu output can be captured and processed on Vita.

This audit is based on TheOfficialFloW/Adrenaline source:
- `user/menu.c`
- `user/main.c`
- `adrenaline_compat.h`

Upstream repository: https://github.com/TheOfficialFloW/Adrenaline

## Confirmed framebuffer access pattern

Adrenaline defines:
- `SCE_PSPEMU_FRAMEBUFFER = 0x74000000`
- `SCE_PSPEMU_FRAMEBUFFER_SIZE = 0x88000`

In its `AdrenalineDraw` thread, it:
1. Initializes Vita2D/GXM.
2. Allocates a PSP-sized texture (512 x 272).
3. Creates GXM shaders for Original, Sharp Bilinear, Advanced AA, and LCD3x.
4. Each frame, copies the PSP framebuffer from `SCE_PSPEMU_FRAMEBUFFER` into its texture with `sceDmacMemcpy`.
5. Draws that texture through its selected GXM shader to the Vita display.
6. Synchronizes using `sceCompatLCDCSync()` for PSP output.

This is direct source evidence that a Vita-side GXM renderer can sample the PSPemu framebuffer and apply a shader before displaying it.

## How Adrenaline integrates with PSPemu

Adrenaline does not simply install a generic post-processing plugin. Its source:
- obtains ScePspemu module information and internal segment addresses;
- resolves internal PSPemu functions using fixed offsets;
- hooks multiple imports in the ScePspemu module, including SceCompat and SceShellUtil;
- replaces the PSPemu settings handler with `ScePspemuCustomSettingsHandler`;
- creates a dedicated `AdrenalineDraw` thread;
- controls PSPemu display behavior and uses PSPemu-specific address conversion/cache functions.

This is a substantial integration with PSPemu, not a safe generic overlay.

## What this means for Chovy-Sign bubbles

Chovy-Sign bubbles run under the official PSPemu path, but do not automatically include Adrenaline's custom settings handler, draw thread, configuration, or display replacement.

A plugin that wants to apply FXAA to a standalone bubble would need to reproduce enough of Adrenaline's display integration to:
- access the current PSP frame safely;
- know exactly when the PSP frame is complete and stable;
- run a Vita GXM shader over it;
- present the processed frame without racing PSPemu or causing tearing;
- restore Sony's normal display path on exit, suspend, crash, or plugin unload.

Merely reading `0x74000000` is not sufficient: timing, cache coherency, process permissions, framebuffer format/stride, and display ownership must be handled correctly. Adrenaline's own source uses a DMA copy and compatibility synchronization, and hooks several PSPemu imports to coordinate this.

## Relevance of Adrenaline's Advanced AA option

Adrenaline includes an **Advanced AA** option and creates a dedicated GXM shader pair named `advanced_aa_v` / `advanced_aa_f`. The menu code selects that shader when the option is chosen.

The checked-in headers expose compiled shader binaries, not the original readable shader source. The source confirms the existence and use of the shader, but does not by itself establish the exact algorithm (for example, do not label it FXAA without inspecting/decompiling the shader or finding its original source).

The shader and renderer are part of Adrenaline's GPL-3.0 project. Any code reuse would require preserving and complying with its license; this research does not copy its binary shader or code into this repository.

## Feasibility assessment

This is the strongest technical lead found so far:
- A PSPemu framebuffer address is used by working Vita software.
- A Vita-side GXM pass can process the PSP image.
- Adrenaline demonstrates an Advanced AA shader stage.

However, the integration relies on substantial PSPemu hooks and a custom display loop. It is not yet a proven low-risk standalone plugin for ordinary Chovy-Sign bubbles.

### Recommended next steps

1. Reverse-engineer the exact PSPemu display callback/synchronization path used by Adrenaline, without copying its fixed offsets blindly.
2. Determine whether the PSPemu framebuffer address, dimensions, format, and stride are stable on FW 3.60 and in normal Chovy-Sign bubbles.
3. Inspect the Advanced AA shader algorithm/source if available, respecting GPL-3.0; otherwise implement a clean-room FXAA-like shader for GXM.
4. Build a proof-of-concept in a separate test environment that only captures and displays a PSP frame, with no persistent config or logging.
5. Add edge detection/FXAA only after stable capture and synchronization are demonstrated.
6. Test frame pacing, tearing, suspend/resume, LiveArea return, and PSP game compatibility before considering a release.

## Safety status

- No plugin/VPK exists.
- No code has been installed on the user's Vita.
- Do not write to the PSP framebuffer from an unverified thread.
- Do not hook PSPemu using Adrenaline's fixed offsets without revalidating every target offset against FW 3.60.
- Do not claim that the standalone Chovy-Sign path can use Adrenaline's Advanced AA option as-is.
