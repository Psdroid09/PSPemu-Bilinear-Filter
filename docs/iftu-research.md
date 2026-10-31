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
