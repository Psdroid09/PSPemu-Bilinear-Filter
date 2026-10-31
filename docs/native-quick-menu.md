# Official PSPemu quick-menu Bilinear setting

## Finding

Sony's official PS Vita User's Guide documents a built-in Bilinear Filtering option for PSP game software. While a PSP game is running, hold the PS button to open the quick menu, choose Settings, then enable Bilinear Filtering.

Sony describes the option as smoothing rough graphics when PSP-resolution output is displayed on the higher-resolution Vita screen. The guide also cautions that some games may show little benefit or may look rough even with the option enabled; in those cases, disable it.

Primary source: [PlayStation Vita User's Guide — Playing a game for the PSP system](https://manuals.playstation.net/document/gb/psvita/game/game_psp.html).

## Test result on the target setup

The user confirmed that the Bilinear Filtering option **does appear** in a Chovy-Sign PSP bubble, but reported that the image looks horrible with it enabled.

Conclusion: the setting is available through the official PSPemu quick menu, but it does not meet this project's visual-quality goal on the user's setup. Do not recommend leaving it enabled. Return it to its previous/off state before comparing any further filter.

This is an important result: the existence of a bilinear toggle does not mean bilinear is the desired solution.

## Technical interpretation

Bilinear filtering interpolates neighboring samples. It can reduce harsh pixel edges, but it also softens fine text, HUD elements, and textures. The user's visual test rejects this output as too blurry/poor.

The native toggle is therefore a useful baseline only, not the final filter.

## Safe test

1. Launch one PSP bubble created with Chovy-Sign.
2. While the game is running, hold the PS button to open the Vita quick menu.
3. Open the game's Settings panel.
4. Toggle Bilinear Filtering only for an A/B comparison.
5. The user found the enabled result unacceptable; leave it disabled for normal play.
6. Do not change plugins, clock speeds, ISO files, licenses, or save data as part of this comparison.

## Next research direction

Investigate whether a sharper reconstruction filter can be applied to the PSPemu output. The IFTU API exposes nearest and bilinear modes, but no documented sharp-bilinear or shader mode. A sharper filter would likely require access to PSPemu's composition/rendering path or another verified programmable stage; a simple IFTU argument change cannot provide that effect.

Do not build a plugin that merely forces IFTU bilinear: the user's direct test has already shown that this does not satisfy the goal.

## Recommendation

Keep native Bilinear Filtering off. Continue source-level investigation for a sharper PSPemu-only solution, and do not install a kernel plugin until its exact hook point and visual behavior are demonstrated.
