# Implementation readiness and go/no-go (2026-11-01)

## Decision

Proceed with a title-scoped **user SUPRX** experiment first. Do not create a project SKPRX and do not add a global `*ALL` entry.

## Loading path now supported by source evidence

taiHEN's `start_preloaded_modules_patched()` runs the original start-preloaded-modules routine, obtains the process TITLEID, then calls `taiLoadPluginsForTitleForKernel(pid, titleid, 0)`. Its comment describes this as occurring after the default modules have been loaded.

Adrenaline's user module independently demonstrates that it can call `taiGetModuleInfo("ScePspemu")` at module start and then install hooks against ScePspemu. This supports the hypothesis that a title plugin can see ScePspemu when loaded for a Chovy-Sign PSP title. It is not yet an on-device verification for every Chovy-Sign bubble.

## Intended minimal plugin behavior

The first build must be a diagnostic-free, display-inert probe:

- It is registered only under one test bubble's TITLEID section in `ur0:tai/config.txt`.
- It queries its own process TITLEID and `ScePspemu` module information.
- It rejects unsupported module NIDs; initial target is FW 3.60 retail only.
- It does not create a drawing thread.
- It does not allocate a framebuffer texture.
- It does not read or write `0x74000000`.
- It does not hook IFTU, display, shell, or kernel functions.
- It creates no files and writes no logs.
- On any failed precondition, it returns without side effects.

A no-op probe alone does not prove display capture; it only validates the per-title loading boundary and module visibility.

## Second-stage renderer integration

Adrenaline's relevant sequence is:

1. Its user module locates ScePspemu and obtains module segment information.
2. It hooks ScePspemu's `sceCompatWaitSpecialRequest` import.
3. The hook initializes Adrenaline before forwarding the original special request.
4. It starts a dedicated Vita2D/GXM drawing thread.
5. The thread copies the PSP framebuffer at `0x74000000` (`0x88000` bytes) using DMA to a texture.
6. It draws the texture through the selected GXM shader and calls `sceCompatLCDCSync()` for PSP presentation.

This sequence is evidence for a possible integration route, not a safe recipe to copy. The frame layout, cache visibility, DMA timing, synchronization, and restoration behavior must be checked against the exact FW 3.60 Chovy-Sign process before any drawing thread is introduced.

## Why the current build is not ready

The project does not yet have:
- a tested title-specific SUPRX;
- a VitaSDK build configuration or CI result for such a module;
- an on-device confirmation of the Chovy bubble TITLEID and taiHEN load timing;
- a verified restoration path for PSPemu's original display;
- suspend/resume and LiveArea recovery tests.

A generated ELF or SUPRX without these checks would be a nonfunctional or risky artifact, not a release.

## One required test fixture detail

The exact TITLEID of the chosen Chovy-Sign test bubble is required to write a safe, title-only config entry and verify the intended scope. It can be read from the bubble's `ux0:app/<TITLEID>/sce_sys/param.sfo` path or from the app's LiveArea details. Do not guess the ID.

## Release gate

No release until:
- the per-title probe is confirmed to load only in the selected PSP bubble;
- the renderer can present and restore frames without black screen, tearing, or hang;
- exit, suspend/resume, and return to LiveArea are clean;
- image quality is visibly better than nearest while avoiding the blur the user rejected;
- frame pacing is acceptable on PS Vita 1000;
- removal is one title-specific config line plus the module file.

## Current state

Research and source-level lifecycle analysis are complete enough to define the first probe. No source or binary plugin has yet been compiled, installed, or tested on hardware.
