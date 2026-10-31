# PSPemu anti-aliasing feasibility research

## User goal

Reduce visible stair-stepping on diagonal and curved edges in PSP games launched as Chovy-Sign bubbles from LiveArea on a PS Vita 1000 (3G), FW 3.60 Ensō.

The goal is not image sharpening. The user rejected Sony's native PSPemu Bilinear Filtering because it made the image look horrible. Leave that setting off. Bilinear output-plane interpolation is not equivalent to anti-aliasing geometry.

## New verified finding: Vita GXM supports MSAA modes

The VitaSDK SceGxm documentation exposes:
- `SCE_GXM_MULTISAMPLE_NONE`
- `SCE_GXM_MULTISAMPLE_2X`
- `SCE_GXM_MULTISAMPLE_4X`

The multisample mode is supplied as part of `SceGxmRenderTargetParams.multisampleMode`. Fragment-program creation also takes a `SceGxmMultisampleMode` argument. See the official VitaSDK API documentation: https://docs.vitasdk.org/group__SceGxmUser.html and the enum listing https://docs.vitasdk.org/globals_eval_s.html.

This proves the Vita's GXM API exposes MSAA configuration for applications using GXM. It does **not** prove that Sony's closed PSPemu renderer uses GXM in an interceptable way, that it creates a multisampled render target, or that a plugin can safely retrofit MSAA after the render target and fragment programs have already been created.

## Techniques

### MSAA — strongest theoretical match for 3D edges

MSAA samples polygon coverage during rasterization and resolves the result. It targets geometric edges more directly than a display scaling filter. On Vita, the public GXM API documents 2x and 4x modes.

Main blocker: MSAA must be selected consistently when creating the render target and matching fragment programs. A simple late kernel hook on the IFTU display plane cannot enable it. Retrofitting MSAA into an already-created PSPemu rendering pipeline is likely to require intercepting or modifying renderer setup, not merely changing the final framebuffer filter.

### FXAA — possible post-process, but unproven in PSPemu

FXAA analyzes the completed image and smooths high-contrast edges. It could theoretically be applied after PSPemu renders the frame, but needs access to the framebuffer plus a programmable GPU pass. No stable PSPemu frame interception point or shader injection route has been demonstrated.

FXAA may soften text/HUD as well as jagged edges, so quality must be compared carefully.

### SSAA / supersampling — likely costly

Render at a higher internal resolution and downsample. This can reduce aliasing but requires control over the emulator's internal rendering resolution and additional rendering/memory bandwidth. No supported stock PSPemu setting for this has been identified.

## PPSSPP comparison

PPSSPP documents higher internal rendering resolution and MSAA (currently dependent on its graphics backend); it also describes FXAA as a post-processing option. These capabilities are part of PPSSPP's own renderer and cannot be assumed to exist in Sony's PSPemu. Reference: https://www.ppsspp.org/docs/settings/graphics/.

## What Chovy-Sign does

Chovy-Sign signs/prepares PSP content to run through the Vita's PSP emulator. It does not replace PSPemu with PPSSPP or supply a custom renderer. Therefore, changing the Chovy-Sign bubble's packaging alone is not known to add MSAA/FXAA support. Reference: https://github.com/LiEnby/chovy-sign.

## Recommended technical investigation

1. Identify the PSPemu graphics stack and whether it uses SceGxm or a private/legacy rendering path.
2. Determine whether PSPemu creates GXM render targets and fragment programs, and whether the calls can be intercepted safely before resource creation.
3. If GXM is used, investigate whether a process-scoped hook can alter render-target and fragment-program multisample configuration consistently. This is a high-risk research path and must not be attempted on the user's working Vita without a reproducible rollback.
4. If the PSPemu path cannot be safely modified for MSAA, investigate a completed-frame FXAA pass only if a stable framebuffer interception and GPU shader execution route can be proven.
5. Benchmark frame pacing and GPU/memory load on Vita 1000 before any release.

## Safety and project constraints

- PSPemu-only. Native Vita games/apps must not be affected.
- No Adrenaline.
- No ISO, license, bubble, or save modification.
- No persistent logging or extra configuration unless proven necessary.
- Do not install a kernel plugin on the user's working console until the hook point, process filtering, rollback, and performance are demonstrated.
- Do not claim MSAA/FXAA works in PSPemu merely because Vita GXM or PPSSPP supports it.
- Do not release a plugin that only forces IFTU bilinear; the user already rejected its image quality.

## Status

- User's target (reduce jagged edges): confirmed.
- Native Bilinear Filtering present in Chovy-Sign bubbles: confirmed by user.
- Native bilinear visual quality: rejected by user.
- Vita GXM MSAA 2x/4x API support: confirmed in VitaSDK documentation.
- PSPemu's actual graphics backend and GXM render-target setup: not yet established.
- Safe MSAA injection into PSPemu: not established.
- PSPemu framebuffer access for FXAA: not established.
- Plugin source/binary/VPK: not implemented.
