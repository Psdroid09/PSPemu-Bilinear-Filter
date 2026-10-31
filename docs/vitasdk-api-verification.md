# VitaSDK 3.60 IFTU and PSPemu API verification

This note records the exact public VitaSDK declarations and firmware-3.60 NIDs relevant to the proposed PSPemu-only bilinear hook. It is source-level verification, not proof that a Chovy-Sign bubble reaches the hook on hardware.

## 1. IFTU filter contract

Source: [vitasdk/vita-headers — psp2kern/lowio/iftu.h](https://github.com/vitasdk/vita-headers/blob/e66ebe90b73d1fa4cce005a5b2072cec27322544/include/psp2kern/lowio/iftu.h).

The header defines:

- `SCE_IFTU_FILTER_MODE_NEAREST = 0`
- `SCE_IFTU_FILTER_MODE_BILINEAR = 1`

The function prototype is:

```c
int ksceIftuSetInputFrameBuffer(
    unsigned int plane_index,
    const SceIftuPlaneState *plane_state,
    int bilinear,
    int sync
);
```

The header's FW 3.60 notes say:

- A nonnegative `bilinear` value replaces the cached filter setting.
- A negative `bilinear` value preserves the cached setting.
- The state pointer is read-only and the function copies the complete 0x54-byte state when it is non-NULL.
- A zero framebuffer pixel format disables the input plane.

The proposed hook must therefore change only the filter argument to `SCE_IFTU_FILTER_MODE_BILINEAR` for a positively identified PSPemu game-output call. Every non-target call must retain all original arguments, including negative filter values.

## 2. PSPemu process discriminator

Source: [vitasdk/vita-headers — psp2kern/kernel/acmgr.h](https://github.com/vitasdk/vita-headers/blob/e66ebe90b73d1fa4cce005a5b2072cec27322544/include/psp2kern/kernel/acmgr.h).

The header declares:

```c
int ksceSblACMgrIsPspEmu(SceUID pid);
```

Its FW 3.60 header comment states that process ID zero selects the current process. `SCE_KERNEL_PROCESS_ID_SELF` is defined as zero in [psp2common/types.h](https://github.com/vitasdk/vita-headers/blob/e66ebe90b73d1fa4cce005a5b2072cec27322544/include/psp2common/types.h).

## 3. Firmware 3.60 import identifiers

Source: [vitasdk/vita-headers — db/360/SceSblACMgr.yml](https://github.com/vitasdk/vita-headers/blob/e66ebe90b73d1fa4cce005a5b2072cec27322544/db/360/SceSblACMgr.yml).

| Module / library | Kernel | NID |
|---|---:|---:|
| SceSblACMgr / SceSblACMgrForDriver | Yes | `0x9AD8E213` |
| `ksceSblACMgrIsPspEmu` | — | `0xFD00C72A` |

The IFTU import used by the Sharpscale hook is the SceDisplay import with library NID `0xCAFCFE50` and function NID `0x7CE0C4DA`. This pair is visible in the upstream-style Sharpscale hook registration; it must still be checked against the exact VitaSDK import database/toolchain used for the build.

## 4. What this verifies — and what it does not

Verified from source:

- The filter values and their meaning.
- The exact const-qualified IFTU hook signature.
- The FW 3.60 PSPemu process-check declaration and current-process ID convention.
- The FW 3.60 SceSblACMgrForDriver library and function NIDs.

Not verified:

- That a Chovy-Sign PSP bubble on the user's FW 3.60 device calls this SceDisplay import.
- The actual framebuffer dimensions, plane index, or incoming filter value for the bubble's game output.
- Whether 480x272 is unique to PSP game output rather than PSP shell/UI or PS1 POPS.
- That a plugin compiled with these imports loads correctly alongside the user's installed kernel plugins.
- That bilinear produces a preferred image or has no measurable performance cost on the target device.

## 5. Implementation rule

Do not treat the provisional `PSPemu + 480x272` check as release-safe until it is observed on hardware. If a diagnostic build is eventually needed, it must be temporary, reversible, and must not write persistent logs. The final plugin should have no file logging, no configuration file, no worker thread, no framebuffer copy, and no changes to ISO, license, bubble, or save data.
