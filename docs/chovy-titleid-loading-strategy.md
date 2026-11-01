# Chovy-Sign plugin loading and scoped test strategy

## Corrected finding: a separate project kernel loader is not automatically required

The earlier design treated Adrenaline's custom kernel loader as the only way to load a renderer into a PSP title. That was too narrow.

taiHEN itself has a title-plugin loading stage. In the public taiHEN source at commit
`309b3800bcb8ebbd5e4f5e5e920af3da3590b829`, `start_preloaded_modules_patched()`:

1. Calls the original `ksceKernelStartPreloadedModules` first.
2. Reads the process TITLEID.
3. Calls `taiLoadPluginsForTitleForKernel(pid, titleid, 0)`.

The source comment explicitly says this event occurs after the default list of modules has been loaded for the process. This makes a title-specific user plugin in taiHEN's normal config a plausible first loading mechanism, without writing a separate project-specific SKPRX.

Source: [yifanlu/taiHEN hen.c at the inspected commit](https://github.com/yifanlu/taiHEN/blob/309b3800bcb8ebbd5e4f5e5e920af3da3590b829/hen.c).

## Why this is promising for PSPemu

Adrenaline's user module calls `taiGetModuleInfo("ScePspemu", ...)` during its own `module_start`, then installs imports/hooks against that already-loaded module. Its source therefore demonstrates that a user module can inspect and hook ScePspemu once loaded.

The likely scoped experiment is:

- Put the future user plugin under the exact TITLEID section for one Chovy-Sign PSP bubble in `ur0:tai/config.txt`.
- Do not put it under `*ALL`, `*KERNEL`, or a project-wide kernel plugin section.
- At `module_start`, query `ScePspemu`; if unavailable, fail closed and return without creating threads or touching display state.
- Confirm the process is PSPemu before doing anything else.
- Keep the plugin removable by deleting its single title-specific config line and its module file.

The taiHEN lifecycle is strong evidence for the load timing, but the exact behavior on a real Chovy-Sign bubble with FW 3.60 still needs on-device confirmation. No assumption should be made that every Chovy-Sign package uses an identical title/process setup.

## What Adrenaline's kernel plugin does differently

Adrenaline's `kernel/main.c` also hooks `ksceKernelStartPreloadedModules`, but its callback loads Adrenaline's user module only for its own TITLEID, `PSPEMUCFW`, and loads its VSH module into the shell process. This is a custom loader for Adrenaline's own app, not proof that a new project must copy that kernel hook.

Source: [TheOfficialFloW/Adrenaline kernel/main.c](https://github.com/TheOfficialFloW/Adrenaline/blob/7d382b7837d9d211d830017ba7aee982fa49a8c6/kernel/main.c).

## Remaining hard problem: safe display composition

Title-scoped loading solves only *where* the user module may start. It does not solve how to replace PSPemu's display output.

Adrenaline's source shows that its renderer is initialized by hooking PSPemu's `sceCompatWaitSpecialRequest` import, before forwarding the original request. It then creates a dedicated Vita2D/GXM draw thread. Each PSP frame is copied from `0x74000000` (`0x88000` bytes) into a texture with DMA, rendered with a selected shader, and synchronized with `sceCompatLCDCSync()`.

This is not a generic framebuffer read. It depends on the PSPemu lifecycle, framebuffer layout, DMA/cache behavior, and display synchronization. Reproducing it incorrectly can cause tearing, black screen, hangs, or suspend/resume faults.

Relevant sources:
- [Adrenaline user/main.c](https://github.com/TheOfficialFloW/Adrenaline/blob/7d382b7837d9d211d830017ba7aee982fa49a8c6/user/main.c)
- [Adrenaline user/menu.c](https://github.com/TheOfficialFloW/Adrenaline/blob/7d382b7837d9d211d830017ba7aee982fa49a8c6/user/menu.c)
- [Adrenaline adrenaline_compat.h](https://github.com/TheOfficialFloW/Adrenaline/blob/7d382b7837d9d211d830017ba7aee982fa49a8c6/adrenaline_compat.h)

## Safe implementation sequence

1. Build a minimal title-specific user plugin that only identifies the process and reports whether `ScePspemu` is present; no display hooks, threads, file logging, or framebuffer access.
2. Confirm the exact Chovy-Sign bubble TITLEID and module load behavior on FW 3.60.
3. Add a reversible hook around the PSPemu startup boundary only after the minimal load test works.
4. Prove the original display path can be restored on exit and suspend/resume before enabling a shader.
5. Add the Advanced AA GXM pass only after frame capture and synchronization are proven.
6. Keep all testing scoped to one bubble. Never add a global kernel hook merely to simplify loading.

## Current status

- taiHEN title-plugin loading after default modules: confirmed in source.
- ScePspemu lookup/hooking from an Adrenaline user module: confirmed in source.
- Title-specific user plugin as a Chovy-Sign test route: plausible, not yet tested on-device.
- Stable framebuffer capture and display replacement for an ordinary Chovy-Sign bubble: not yet proven.
- Project SKPRX/SUPRX/VPK: not implemented.
- No changes have been made to the user's Vita.
