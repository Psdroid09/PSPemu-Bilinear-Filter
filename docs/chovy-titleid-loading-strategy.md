# Chovy-Sign plugin loading and scoped test strategy

## Why Adrenaline's loader does not cover normal Chovy bubbles

Adrenaline's kernel plugin hooks `ksceKernelStartPreloadedModules`, obtains the process title ID, and loads `adrenaline_user.suprx` only when that ID equals `PSPEMUCFW`. That is Adrenaline's own application ID. A normal Chovy-Sign bubble is a separately signed PSP title, so it does not match that condition and does not automatically receive Adrenaline's user renderer.

Source: [TheOfficialFloW/Adrenaline `kernel/main.c`](https://github.com/TheOfficialFloW/Adrenaline/blob/master/kernel/main.c).

## Potential scoped loading route

taiHEN supports loading a user plugin for a specific application title ID. That gives a safer experiment than `*ALL`:

- Put the test plugin under a single Chovy bubble's title-ID section.
- On module start, call `taiGetModuleInfo("ScePspemu")`.
- If `ScePspemu` is not present, do nothing and exit initialization.
- Only if it is present, initialize the PSP framebuffer renderer and AA shader.
- Do not add a global `*ALL` plugin and do not install a kernel hook for the first experiment.

Adrenaline itself confirms that the PSPemu process contains a module named `ScePspemu`, since its user module obtains that module's info and hooks its imports. The exact loading order for a normal Chovy bubble still needs validation on FW 3.60.

## Risk boundary

This is an architecture proposal, not a proven implementation. A Vita2D/GXM draw thread would still take over display composition and must synchronize with PSPemu. The experiment must be scoped to one bubble, be removable by deleting one taiHEN title-ID line, and must never run in Settings, LiveArea, or unrelated games.

Do not add the plugin to `*ALL`, do not install a kernel plugin, and do not test against the SD2Vita or storage stack. No test binary currently exists.
