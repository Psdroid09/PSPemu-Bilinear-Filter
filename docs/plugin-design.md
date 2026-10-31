# Proposed minimal PSPemu-only IFTU filter plugin

## Objective

Request hardware IFTU bilinear sampling for the PSP emulator's normal 480x272 framebuffer, while leaving all non-target display calls unchanged. The intended use is Chovy-Sign PSP bubbles launched from LiveArea, without Adrenaline.

## Evidence already available

- VitaSDK defines nearest = 0 and bilinear = 1.
- VitaSDK documents the filter argument as a cached setting: nonnegative values replace it; negative values preserve it.
- VitaSDK documents PID 0 as the current process for APIs expecting a process ID.
- Sharpscale's upstream IFTU hook calls `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` inside the hook.
- Sharpscale and the Adrenaline Bubble Booter patch both hook the SceDisplay import for `ksceIftuSetInputFrameBuffer`, using library NID `0xCAFCFE50` and function NID `0x7CE0C4DA`.

## Proposed call policy

For each intercepted call:

1. If the current process is not PSPemu, forward every argument unchanged.
2. If the plane-state pointer is NULL, initially forward unchanged; the cached state is not inspectable at this point.
3. If the framebuffer dimensions do not match the target signature, forward unchanged.
4. If the framebuffer matches the verified PSP target, replace only the filter argument with `SCE_IFTU_FILTER_MODE_BILINEAR`.
5. Always forward to the original function exactly once and return its result.

Initial target signature: PSPemu process + non-NULL state + 480x272 framebuffer. This signature is provisional until checked on hardware. A plane-index constraint may be added only after the PSPemu plane is identified reliably.

## Why a custom hook instead of Sharpscale

Sharpscale is a broader scaler with multiple hooks, internal SceDisplay offsets, configuration storage, and a global display policy. Its Bilinear option preserves the incoming filter value; it does not force a 0 to 1. A minimal plugin can avoid scaling geometry changes and can explicitly request bilinear only for PSPemu calls that match the validated target.

## Implementation constraints

- Kernel plugin (`.skprx`) loaded from `*KERNEL`.
- Use taiHEN's kernel import hook API.
- Import `ksceSblACMgrIsPspEmu` from the SblACMgr kernel/driver library.
- Hook the SceDisplay IFTU import (library NID `0xCAFCFE50`, function NID `0x7CE0C4DA`).
- Use the exact VitaSDK 3.60-compatible function signature, including a const plane-state pointer.
- No file logging, debug log file, config file, syscall export, user plugin, shader, framebuffer copy, or extra background thread.
- On hook-install failure, fail module startup cleanly and release any partially installed hook.
- On module stop, release the hook before returning.
- Do not patch SceDisplay internal offsets or inject code/data into SceDisplay.
- Do not write ISO, license, savedata, or Chovy-Sign bubble files.

## Remaining blockers before compiling a user-facing VPK

1. Confirm the exact PSPemu IFTU call and incoming filter value on FW 3.60 using a disposable bubble or a controlled diagnostic build.
2. Confirm whether PSPemu's 480x272 framebuffer is unique to the PSP game output or also used by PSP shell/UI and PS1 POPS content.
3. Confirm plane identity and whether the target call uses a non-NULL state pointer on initial submission.
4. Confirm the linked kernel import NIDs and function prototype against the selected VitaSDK revision.
5. Review taiHEN hook chaining behavior and ensure the hook calls the continuation exactly once.
6. Build with a reproducible VitaSDK toolchain and inspect the resulting SELF/SKPRX imports.
7. Test on the user's exact FW 3.60 + Ensō environment with a disposable bubble before any normal library.
8. Test launch/exit loops, suspend/resume, LiveArea return, native Vita apps, PSP UI, and PS1 POPS.
9. Verify removal by deleting only the plugin line and rebooting; no recovery should require touching SD2Vita content.

## Safety classification

- Source-level feasibility: strong.
- PSPemu process discrimination at this hook: supported by upstream Sharpscale usage.
- Chovy-Sign bubble path on FW 3.60: not yet confirmed on hardware.
- Exact framebuffer/plane signature: provisional.
- Visual benefit: unknown until compared on a real Vita.
- Release readiness: not ready; no plugin binary exists.

A successful compile is not proof of runtime safety or visible improvement. Do not distribute or install a release build until the hardware blockers above are resolved.
