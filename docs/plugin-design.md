# Proposed minimal PSPemu-only IFTU filter plugin

## Objective

Request hardware IFTU bilinear sampling for PSP game output while leaving every non-target display call unchanged. The intended use is Chovy-Sign PSP bubbles launched from LiveArea, without Adrenaline.

## Verified source-level API contract

The VitaSDK FW 3.60 headers define:

- `SCE_IFTU_FILTER_MODE_NEAREST = 0`.
- `SCE_IFTU_FILTER_MODE_BILINEAR = 1`.
- `ksceIftuSetInputFrameBuffer(unsigned int plane_index, const SceIftuPlaneState *plane_state, int bilinear, int sync)`.
- A nonnegative filter argument replaces the cached setting; a negative value preserves it.
- `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` checks whether the current process is PSPemu; FW 3.60 uses process ID zero for the current process.

The relevant FW 3.60 SceSblACMgrForDriver library NID is `0x9AD8E213`, and `ksceSblACMgrIsPspEmu` has function NID `0xFD00C72A`. The SceDisplay IFTU import pair used by the Sharpscale hook is library NID `0xCAFCFE50` and function NID `0x7CE0C4DA`.

Full declarations, source links, and verification limits: [`docs/vitasdk-api-verification.md`](vitasdk-api-verification.md).

## Evidence already available

- Sharpscale's upstream IFTU hook calls `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` inside the hook.
- Sharpscale and the Adrenaline Bubble Booter patch hook the SceDisplay import for `ksceIftuSetInputFrameBuffer`.
- The IFTU API allows the filter argument to be explicitly set to bilinear without changing framebuffer geometry or copying image data.

## Proposed call policy

For each intercepted call:

1. If the current process is not PSPemu, forward every argument unchanged.
2. If the plane-state pointer is NULL, initially forward unchanged; the cached state is not inspectable at this point.
3. If the framebuffer dimensions or plane identity do not match the target signature verified on hardware, forward unchanged.
4. If the call is positively identified as PSP game output, replace only the filter argument with `SCE_IFTU_FILTER_MODE_BILINEAR`.
5. Always forward to the original function exactly once and return its result.

The earlier `PSPemu + non-NULL state + 480x272` signature remains a candidate only. It is not release-safe until the actual Chovy-Sign bubble call and plane identity are observed on FW 3.60. Do not assume 480x272 uniquely identifies PSP game output; PSP shell/UI or PS1 POPS may use related display paths.

## Why a custom hook instead of Sharpscale

Sharpscale is a broader scaler with multiple hooks, internal SceDisplay offsets, configuration storage, and a global display policy. Its Bilinear option preserves the incoming filter value; it does not force a 0 to 1. A minimal plugin can avoid scaling geometry changes and can explicitly request bilinear only for PSPemu calls that match a validated target.

## Implementation constraints

- Kernel plugin (`.skprx`) loaded from `*KERNEL`.
- Use taiHEN's kernel import hook API.
- Use the verified FW 3.60 SceDisplay and SceSblACMgr import identifiers documented above.
- Use the exact VitaSDK 3.60-compatible function signature, including a const plane-state pointer.
- No file logging, debug log file, config file, syscall export, user plugin, shader, framebuffer copy, or extra background thread.
- On hook-install failure, fail module startup cleanly and release any partially installed hook.
- On module stop, release the hook before returning.
- Do not patch SceDisplay internal offsets or inject code/data into SceDisplay.
- Do not write ISO, license, savedata, or Chovy-Sign bubble files.

## Remaining blockers before compiling a user-facing VPK

1. Confirm the exact PSPemu IFTU call, plane identity, framebuffer dimensions, and incoming filter value on FW 3.60 using a disposable bubble or a controlled temporary diagnostic build.
2. Confirm whether PSPemu's game-output signature is distinct from PSP shell/UI and PS1 POPS content.
3. Confirm the linked kernel import NIDs and function prototype against the exact selected VitaSDK revision and build toolchain.
4. Review taiHEN hook chaining behavior and ensure the hook calls the continuation exactly once.
5. Build with a reproducible VitaSDK toolchain and inspect the resulting SELF/SKPRX imports.
6. Test on the user's exact FW 3.60 + Ensō environment with a disposable bubble before any normal library.
7. Test launch/exit loops, suspend/resume, LiveArea return, native Vita apps, PSP UI, and PS1 POPS.
8. Verify removal by deleting only the plugin line and rebooting; no recovery should require touching SD2Vita content.

## Safety classification

- Source-level feasibility: strong.
- PSPemu process discrimination at this hook: supported by upstream Sharpscale usage.
- FW 3.60 IFTU and SceSblACMgr API declarations/NIDs: verified against VitaSDK source.
- Chovy-Sign bubble path on FW 3.60: not yet confirmed on hardware.
- Exact framebuffer/plane signature: provisional.
- Visual benefit: unknown until compared on a real Vita.
- Release readiness: not ready; no plugin binary exists.

A successful compile is not proof of runtime safety or visible improvement. Do not distribute or install a release build until the hardware blockers above are resolved.
