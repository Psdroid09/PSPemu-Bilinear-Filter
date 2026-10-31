# Sharpscale compatibility analysis

## Key finding

Sharpscale already implements the relevant final-output filtering path. It hooks `SceDisplay`'s import of `ksceIftuSetInputFrameBuffer` and has a `bilinear` configuration option. Its configuration app exposes two values: `Point` and `Bilinear`.

Sources:
- [Sharpscale source: `sharpscale.c`](https://github.com/psv-plugins-archive/sharpscale/blob/master/sharpscale.c)
- [Sharpscale config app: `config-app/main.c`](https://github.com/psv-plugins-archive/sharpscale/blob/master/config-app/main.c)
- [Sharpscale README](https://github.com/psv-plugins-archive/sharpscale/blob/master/README.md)
- [Sharpscale changelog](https://github.com/psv-plugins-archive/sharpscale/blob/master/CHANGELOG.md)

## Code-path observations

- The kernel module tracks the active primary display head and framebuffer metadata through SceDisplay hooks.
- It hooks `SceDisplay` import library NID `0xCAFCFE50`, function NID `0x7CE0C4DA` (`ksceIftuSetInputFrameBuffer`).
- In its filter logic, when the config is Point (`bilinear == false`), it changes an incoming IFTU bilinear request (`bilinear == 1`) to nearest (`0`).
- When config is Bilinear (`bilinear == true`), it leaves an incoming bilinear request unchanged.
- The plugin's PSP emulator detection is used for its PS1 aspect-ratio behavior. The filter toggle itself is applied in the general display-plane scaling path, not behind a Chovy-Sign or Adrenaline-only check.
- The README explicitly describes Bilinear as bilinear interpolation and Point as nearest-neighbour; when scaling is not applied, the system default bilinear interpolation is used.

## Relevance to Chovy-Sign PSP bubbles

This is the strongest existing candidate for the user's desired effect without starting Adrenaline. Chovy-Sign creates PSP content that runs through the Vita's PSP emulator; Sharpscale operates below the app at the SceDisplay/IFTU output stage. Its generic IFTU hook is therefore likely to see the PSPemu compatibility framebuffer too.

However, this remains a source-level compatibility assessment, not a test on the user's console. The exact Chovy-Sign bubble path, firmware-specific SceDisplay behavior, and visual result must be confirmed on the Vita.

## Important limitations

- This is **output scaling interpolation**, not texture filtering inside the PSP emulator. It softens the final scaled image; it cannot add detail or raise the internal PSP rendering resolution.
- Sharpscale's `bilinear` setting is global to the display path, not documented as a per-PSP-bubble option. Other scaled applications may also be affected.
- The kernel plugin uses SceDisplay hooks and internal offsets. Treat it as a kernel-level modification and verify the exact release and firmware compatibility before installation.
- The source contains file-logging initialization calls, but `CMakeLists.txt` defaults `FILE_LOGGING` to `0`, which disables file logging in a default source build. The config app inherits that setting. The exact prebuilt package should still be checked before claiming that it writes no files.
- Sharpscale's default/reset config sets `bilinear = false` (Point). To test the smoother mode, the configuration app must show `Bilinear` as selected.

## Version and maintenance note

The mirrored source repository has no GitHub Releases entries and its latest listed source commit is from August 2020. The published Sharpscale 1.50 package page lists version 1.50 and a release date of August 24, 2020. This is mature but old software, not a recently maintained plugin. Firmware 3.60 compatibility should be verified against the exact binary/build rather than assumed from the generic description.

## Recommended test sequence

1. First check whether Sharpscale is already installed; do not install a duplicate kernel plugin.
2. If present, record the current settings and confirm the loaded plugin path/version.
3. Change only `Scaling algorithm` from Point to Bilinear in the Sharpscale configuration app.
4. Test one non-critical Chovy-Sign PSP bubble and compare fine text/edges with the previous setting.
5. Check one native Vita app/game afterward for unwanted softness or visual regression.
6. If the result is worse, restore Point. No ISO, license, save, or bubble files should need modification.

Do not combine this test with other plugin changes or overclock changes. Do not use the AdrBubbleBooter direct-IFTU patch as a substitute for Sharpscale; it is a separate Adrenaline Bubble Booter modification.