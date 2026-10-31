# Proposed PSPemu-only image-filter investigation

## Objective

Find a sharper, more pleasing smoothing method for PSP games launched as Chovy-Sign bubbles from LiveArea, without Adrenaline and without changing ISO, license, or save data.

Target environment: PS Vita 1000, hardware 3G, firmware 3.60 with Ensō.

## User's native-filter test

The user confirmed that Sony's quick-menu **Bilinear Filtering** option is present in Chovy-Sign PSP bubbles. The user also tested it and reported that the image looks horrible.

Therefore, forcing IFTU bilinear is no longer an acceptable project outcome. The native toggle should be left off for normal use. See [`native-quick-menu.md`](native-quick-menu.md).

## Verified source-level APIs

VitaSDK defines:

- `SCE_IFTU_FILTER_MODE_NEAREST = 0`.
- `SCE_IFTU_FILTER_MODE_BILINEAR = 1`.
- `ksceIftuSetInputFrameBuffer(unsigned int plane_index, const SceIftuPlaneState *plane_state, int bilinear, int sync)`.
- A nonnegative filter argument replaces the cached setting; a negative argument preserves it.
- `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` identifies whether the current process is PSPemu.

The FW 3.60 SceSblACMgrForDriver library NID is `0x9AD8E213`, and `ksceSblACMgrIsPspEmu` has function NID `0xFD00C72A`. The SceDisplay IFTU import pair used by Sharpscale is library NID `0xCAFCFE50` and function NID `0x7CE0C4DA`.

Detailed declarations and verification limits: [`vitasdk-api-verification.md`](vitasdk-api-verification.md).

## What IFTU can and cannot do

The documented IFTU filter argument offers nearest and bilinear. A simple hook that changes nearest to bilinear only reproduces the blurry result the user rejected.

IFTU's documented filter argument does not expose sharp-bilinear, CRT/LCD shaders, edge-aware reconstruction, or a programmable custom kernel. Do not describe a bilinear-only hook as a sharper filter.

## Candidate directions, in order of investigation

1. **Check PSPemu's own graphics/composition path.** Determine whether its renderer has a supported filter or texture-sampling setting beyond the quick-menu bilinear switch.
2. **Investigate a programmable composition stage.** A sharp-bilinear-like filter requires shader or equivalent programmable sampling. Identify whether PSPemu exposes an appropriate hook/API before considering code.
3. **Review existing Vita display projects only as references.** Sharpscale is useful for understanding IFTU hooks, but its bilinear option preserves the incoming filter value and does not provide the desired sharp shader. Do not install it just for this experiment.
4. **Only then decide whether a custom plugin is feasible.** A kernel import hook alone may not be sufficient for a programmable filter. Do not assume a custom shader can be injected safely into PSPemu.

## Constraints

- PSPemu-only effect: native Vita games and apps must remain untouched.
- No Adrenaline requirement.
- No ISO, license, bubble, or save modification.
- No persistent file logging, configuration file, worker thread, or framebuffer copy unless a proven implementation requires it and the cost is justified.
- No installation on the user's working Vita until the hook path and rollback are established.
- Keep the native Bilinear Filtering option off after the user's negative visual test.

## Remaining blockers

1. Find a supported or demonstrated programmable filter path in PSPemu.
2. Establish whether PSPemu's renderer can be intercepted without patching unstable internal offsets.
3. Determine whether a sharp reconstruction filter can run within Vita 1000 performance limits.
4. Prototype offline or in a disposable environment first.
5. Build and test a reversible plugin only if the prior checks support it.

## Status

- Native quick-menu Bilinear availability in Chovy-Sign bubbles: confirmed by user.
- Native Bilinear visual quality: rejected by user.
- VitaSDK IFTU nearest/bilinear contract: verified from source.
- Sharp-bilinear/shader path in stock PSPemu: not yet identified.
- Custom plugin source: not implemented.
- Binary/VPK: does not exist.

A plugin that only forces IFTU bilinear must not be released as the solution.
