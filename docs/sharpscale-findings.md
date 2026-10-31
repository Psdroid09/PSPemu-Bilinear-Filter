# Sharpscale compatibility analysis

## Executive correction

Sharpscale is highly relevant because it hooks the same IFTU call that a custom plugin would need to intercept. However, its Bilinear option is **not a force-bilinear switch**. It preserves the filter value supplied by the caller. Therefore, it cannot guarantee bilinear filtering for a PSPemu bubble if PSPemu submits the call with nearest filtering.

This distinction was verified directly in the upstream source branch `Electry/Sharpscale`, file `main.c`, function `sceIftuSetInputFrameBuffer_hook`:

```c
bilinear = (!ss_config.bilinear && bilinear == 1) ? 0 : bilinear;
```

Meaning:
- Point mode: changes an incoming value of 1 to 0.
- Bilinear mode: leaves the incoming value unchanged (including 0).
- It does not turn an incoming 0 into 1.

The source also confirms that Sharpscale calls `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` from inside this IFTU hook for its PSPemu-specific PS1 aspect-ratio logic. This is strong evidence that the process check is meaningful at this call site, although it is not a substitute for testing on the target firmware and bubble.

## Relevant source behavior

- Kernel module hooks the `SceDisplay` import for IFTU using library NID `0xCAFCFE50` and function NID `0x7CE0C4DA`.
- It also hooks internal SceDisplay framebuffer-preparation functions and reads SceDisplay internal structures/offsets.
- It tracks primary-head and framebuffer metadata to calculate scaling and placement.
- Its PSPemu process check is used only in the aspect-ratio branch; the filter-setting expression itself is applied in the generic scaling path.
- If scaling is not applied, the hook passes through the original filter value.
- The configuration reset sets `bilinear = false`, i.e. Point mode.

## Relevance to Chovy-Sign PSP bubbles

Chovy-Sign prepares/signs PSP content for execution by the Vita PSP emulator; it does not provide its own renderer or graphics-filter setting. A kernel-level IFTU hook is therefore a plausible point below the bubble launcher.

The source evidence establishes:
1. Sharpscale hooks the same IFTU import that is the candidate integration point.
2. Sharpscale's hook can identify PSPemu using `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)`.
3. A custom hook could, in principle, set the IFTU filter to bilinear only when the current process is PSPemu.

Still unverified on hardware:
- Whether every target Chovy-Sign PSP bubble reaches this exact import path on FW 3.60.
- The incoming filter value PSPemu actually supplies for the relevant plane.
- Whether forcing bilinear produces the preferred image on the Vita 1000 panel.
- Whether the same PSPemu process also uses a 480x272 framebuffer for other content (for example PSP shell/UI or PS1 POPS paths).

## Important distinction: output interpolation vs texture filtering

IFTU bilinear filtering affects the final display-plane sampling/scaling. It does not:
- increase the PSP game's internal render resolution;
- add texture detail;
- change the game's own texture filtering;
- repair low-resolution assets.

It can soften jagged edges, but it can also soften text, HUD elements, and pixel-art. The desired outcome must be judged visually.

## Global impact and safety

Sharpscale is a system-wide display plugin, not a per-bubble setting. Its internal hooks and offsets make it a kernel-level change. Its source supports optional file logging; the source build's logging is controlled by compile definitions, so the exact prebuilt binary must be audited before claiming it writes no files.

Do not install Sharpscale solely on the assumption that Bilinear forces the PSPemu filter on. If it is already installed, do not change its configuration until its current settings and effects are recorded.

## Candidate custom implementation

A much smaller plugin may be possible than Sharpscale:
- One kernel import hook on `SceDisplay` / `ksceIftuSetInputFrameBuffer`.
- Call `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)`.
- Require a non-NULL plane state and the expected PSP framebuffer dimensions (initial candidate: 480x272).
- Change only the `bilinear` argument to `SCE_IFTU_FILTER_MODE_BILINEAR`.
- Preserve plane, state, and sync arguments.
- Pass every non-PSPemu or non-matching call through unchanged.
- No config file, file logging, framebuffer rewriting, scaling geometry changes, or ISO/save/license modifications.

This is a design proposal, not a compiled or tested plugin. The 480x272 condition is a conservative candidate discriminator, not yet proven unique to PSP game output. Before release, inspect plane identity and test whether PSP1/POPS or shell surfaces could match it. The hook must also preserve negative filter values for non-target calls because VitaSDK documents negative values as “preserve cached filter.”

## Version / maintenance note

Sharpscale's public source is old (2020) and its binary package must be checked independently for exact firmware compatibility. The source is valuable as a technical reference, not an automatic recommendation for this user's 3.60 setup.

## Recommended test sequence

1. Confirm whether Sharpscale is installed; do not install a duplicate.
2. If installed, record version, configured path, and current mode before changing anything.
3. Use a disposable PSP bubble and capture a baseline image.
4. Compare baseline against a controlled bilinear-only test, keeping all other plugins and clocks unchanged.
5. Check fine text, HUD, gradients, and motion, not just a title screen.
6. Verify native Vita applications remain unchanged with the proposed PSPemu-only filter.
7. Test suspend/resume, returning to LiveArea, and repeated launches.
8. If any regression occurs, remove/disable the test plugin by editing the taiHEN config from VitaShell or recovery access; never modify game data to undo it.

No ISO, license, bubble, or save file should need modification.
