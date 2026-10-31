# Official PSPemu quick-menu Bilinear setting

## Finding

Sony's official PS Vita User's Guide documents a built-in Bilinear Filtering option for PSP game software. While a PSP game is running, hold the PS button to open the quick menu, choose Settings, then enable Bilinear Filtering.

Sony describes the option as smoothing rough graphics when PSP-resolution output is displayed on the higher-resolution Vita screen. The guide also cautions that some games may show little benefit or may look rough even with the option enabled; in those cases, disable it.

Primary source: [PlayStation Vita User's Guide — Playing a game for the PSP system](https://manuals.playstation.net/document/gb/psvita/game/game_psp.html).

## Relevance to Chovy-Sign bubbles

Chovy-Sign prepares PSP content to run through the Vita PSP emulator. It does not replace the emulator renderer. Therefore, the official PSPemu quick-menu option is the first and safest route to test on a Chovy-Sign PSP bubble.

The official Sony guide does not specifically name Chovy-Sign-created bubbles. Availability in those bubbles is therefore strongly plausible but still needs confirmation on the user's actual FW 3.60 setup.

## Safe test

1. Launch one PSP bubble created with Chovy-Sign.
2. While the game is running, hold the PS button to open the Vita quick menu.
3. Open the game's Settings panel.
4. Look for **Bilinear Filtering** and enable it.
5. Return to the game and compare text, edges, HUD, and 3D scenes.
6. Exit and relaunch the same bubble to see whether the setting persists.
7. Repeat with one other PSP bubble only if needed.

This changes an official PSPemu setting; it should not require installing a plugin, changing taiHEN config, editing the ISO, or touching saves/licenses.

## If the option is missing

If the Settings panel or Bilinear Filtering entry does not appear in a Chovy-Sign bubble, record what entries are visible. Only then proceed to the kernel IFTU investigation. Do not install Sharpscale just to test this option.

## Recommendation

Try the Sony-provided quick-menu setting before any plugin work. It is the lowest-risk solution and could make the custom plugin unnecessary.
