# IFTU research: PSPemu output filtering

## Why this matters

VitaSDK exposes the kernel IFTU display-plane API. In firmware 3.60, `SceIftuPlaneState` contains the framebuffer dimensions, and `ksceIftuSetInputFrameBuffer(plane, state, bilinear, sync)` accepts `SCE_IFTU_FILTER_MODE_NEAREST = 0` or `SCE_IFTU_FILTER_MODE_BILINEAR = 1`.

Primary reference: [vitasdk/vita-headers — psp2kern/lowio/iftu.h](https://github.com/vitasdk/vita-headers/blob/master/include/psp2kern/lowio/iftu.h).

## Existing direct-IFTU implementation found

[ONElua/AdrenalineBubbleManager](https://github.com/ONElua/AdrenalineBubbleManager) contains `patches/adrbubblebooter-v1.3-direct-iftu.patch` and `patches/README.md` describing a direct IFTU fast path.

The patch:
- Installs a kernel import hook on `SceDisplay` for `SceIftuSetInputFrameBuffer`.
- Checks for a 480×272 input framebuffer.
- Can override the IFTU filter argument before continuing to the original function.
- Exposes a kernel syscall to its user module so the behavior is enabled only under a specific graphics configuration.

The patch's special case changes `bilinear` from 1 to 0 for **Sharp Bilinear Simple at exactly 2× scale**. Its own README says this reproduces exact 2× pixel replication while bypassing the user-mode GXM display queue. This is a nearest-sampling fast path for that particular sharp shader configuration; it is not itself a soft bilinear filter.

## What this proves — and what it does not

**Proven:** On Vita firmware 3.60, a kernel-level hook can intercept the IFTU filter argument passed by `SceDisplay`; the API distinguishes nearest and bilinear filtering. Existing public code uses this hook for PSP-mode compatibility framebuffers.

**Not yet proven:** That a Chovy-Sign-created official PSP bubble on this user's setup follows the same `SceDisplay` → IFTU path with a 480×272 input, or that changing the IFTU argument will alter its visible output as desired. The cited patch is integrated into an Adrenaline Bubble Booter build and is not a drop-in plugin for Chovy-Sign bubbles.

## Candidate experiment (not implemented or installed)

Build a separate, opt-in kernel test plugin that:
1. Hooks only the `SceDisplay` import of `ksceIftuSetInputFrameBuffer`.
2. Logs nothing and writes no persistent files.
3. Leaves all calls unchanged unless the input is identified as the PSPemu compatibility framebuffer.
4. In test mode, changes only the filter argument to `SCE_IFTU_FILTER_MODE_BILINEAR`.
5. Has an immediate disable path and is tested first on a disposable PSP bubble.

Before implementing this, determine a reliable PSPemu-only discriminator. Width/height alone may be insufficient because a system-wide kernel hook must not alter unrelated display planes or native Vita apps. A process/title check or verified framebuffer metadata may be required.

## Important correction about the desired effect

IFTU bilinear filtering is a real hardware display-plane sampling mode. It may soften the final scaled image, but it does not increase the PSP render resolution, add texture detail, or remove aliasing from the original 480×272 image. It can also soften UI/text. The result must be visually compared against the current output before choosing it as the default.

## Safety decision

No plugin or VPK is being distributed at this stage. Do not install the Adrenaline Bubble Manager patch or replace any Adrenaline kernel/user modules on this system for this experiment. Those modules are an ABI-matched pair for that project and are not a validated Chovy-Sign solution.

## References

- [VitaSDK IFTU kernel header](https://github.com/vitasdk/vita-headers/blob/master/include/psp2kern/lowio/iftu.h)
- [Adrenaline Bubble Manager patch README](https://github.com/ONElua/AdrenalineBubbleManager/blob/master/patches/README.md)
- [Direct IFTU patch](https://github.com/ONElua/AdrenalineBubbleManager/blob/master/patches/adrbubblebooter-v1.3-direct-iftu.patch)
- [Adrenaline source](https://github.com/TheOfficialFloW/Adrenaline)

## PSPemu-only process gate (new finding)

VitaSDK's kernel SblACMgr documentation exposes `ksceSblACMgrIsPspEmu(SceUID pid)`, which returns 1 when the target process is PSPemu. On firmware 3.60, process ID zero selects the current process. The related `ksceSblACMgrIsPspEmu2` documentation identifies PSPemu PAIDs as `0x2800000000007009` and `0x2800000000000013`.

Primary reference: [VitaSDK SceSblACMgrKernel documentation](https://docs.vitasdk.org/group__SceSblACMgrKernel.html).

This provides a documented candidate gate: evaluate `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` in the IFTU hook and change the filter argument only when it returns 1. All other calls must pass through unchanged.

### Kernel hook-context caveat

The IFTU import hook runs in the kernel display path. Before implementation, verify that the current-process identity observed at this call site is reliably PSPemu on the target firmware and does not resolve to a display/system worker context. The API's existence alone does not prove that its result will be PSPemu at every invocation. If ambiguous, do not use this gate blindly.

### Proposed minimal plugin behavior

- Install one import hook for `SceDisplay` → `ksceIftuSetInputFrameBuffer`.
- If current process is not PSPemu, call the original function with every argument unchanged.
- If current process is PSPemu, set only the filter argument to `SCE_IFTU_FILTER_MODE_BILINEAR`; preserve plane, state, and sync mode.
- No filesystem logging, no config file initially, no framebuffer rewriting, no scaling/aspect-ratio modifications, and no changes to ISO/license/save data.
- Include a clear disable/uninstall procedure and test on a disposable bubble before normal use.

This is a viable implementation direction, not a tested plugin. It still requires compilation against the correct VitaSDK/taiHEN headers, ABI review, firmware 3.60 testing, and confirmation on a real Chovy-Sign bubble.


## Updated source-level finding: process identity at the IFTU hook

The upstream [Electry/Sharpscale `main.c`](https://github.com/Electry/Sharpscale/blob/source/main.c) was fetched and inspected. Its `sceIftuSetInputFrameBuffer_hook` calls:

```c
ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)
```

inside the IFTU hook itself, to gate PSPemu-specific aspect-ratio handling. This is stronger evidence than merely finding the API in VitaSDK documentation: a real kernel plugin uses the current-process test at this exact hook site.

It does not prove every call is made in the PSPemu context, nor does it prove the Chovy-Sign bubble's exact call sequence. It does, however, make the PSPemu process gate a well-founded implementation candidate rather than a speculative API choice.

## Updated source-level finding: Sharpscale does not force Bilinear

The upstream Sharpscale hook contains:

```c
bilinear = (!ss_config.bilinear && bilinear == 1) ? 0 : bilinear;
```

Therefore:
- Point setting converts incoming bilinear (1) to nearest (0).
- Bilinear setting preserves the caller's incoming value.
- If PSPemu sends nearest (0), Sharpscale's Bilinear setting leaves it at 0.

A custom PSPemu-only hook would need to explicitly set the argument to 1 for the target PSP framebuffer. Sharpscale is useful proof of the hook and process gate, but its Bilinear option alone does not prove the desired result.

## IFTU API details relevant to a safe hook

The current VitaSDK header documents that:
- `ksceIftuSetInputFrameBuffer` takes plane index, a read-only plane-state pointer, filter argument, and sync flag.
- `SCE_IFTU_FILTER_MODE_NEAREST` is 0 and `SCE_IFTU_FILTER_MODE_BILINEAR` is 1.
- A nonnegative filter argument replaces the cached filter setting.
- A negative filter argument preserves the cached filter setting.
- A NULL plane-state pointer reuses cached state.

This means the hook must not alter negative values for unrelated calls. For the target PSPemu call, setting the filter to 1 is a direct documented way to request bilinear, but the target identification must be sufficiently narrow.

Primary source: [VitaSDK `iftu.h`](https://github.com/vitasdk/vita-headers/blob/master/include/psp2kern/lowio/iftu.h).

## Narrowing the target

The current conservative candidate is:
- PSPemu process check succeeds;
- plane state is non-NULL;
- framebuffer dimensions are exactly 480x272.

This is not yet a proven unique signature. It may include PSPemu UI or another compatibility mode if those use the same dimensions. Before a release build, inspect the plane identity and verify actual calls on a disposable PSP bubble. Do not rely on dimensions alone if a stronger verified discriminator is available.
