# Ben 10 Chovy-Sign test fixture: identifier verification

Date: 2026-11-01

## User-confirmed fixture

- Game: Ben 10: Protector of Earth (USA)
- User-confirmed PSP content folder: `ux0:pspemu/PSP/GAME/ULUS10307/`
- User confirmed that `EBOOT.PBP` exists in that folder.
- PSP game serial/content ID: `ULUS10307`.

Chovy-Sign's own README says that its output is a **TITLEID** folder which is copied into `ux0:/pspemu/PSP/GAME`, followed by deleting `ux0:/id.dat` and rebooting to refresh the app database. This supports treating `ULUS10307` as the PSP content folder/title identifier for the signed game.

Source: https://github.com/LiEnby/chovy-sign

## Critical distinction for taiHEN configuration

Do not yet assume `ULUS10307` is the process TITLEID seen by taiHEN when the LiveArea bubble is launched.

The Chovy-Sign README describes the PSP content folder, not the exact process TITLEID reported by taiHEN. Other PSP bubble tooling explicitly distinguishes the TITLEID of the LiveArea bubble from the TITLEID of the PSP game being launched. Therefore a `*ULUS10307` section must not be added to `ur0:tai/config.txt` until the process TITLEID has been independently verified.

Reference illustrating the distinction for direct PSP bubbles: https://wololo.net/2017/06/20/adrenaline-bubble-booter-released-boot-psp-games-directly-livearea-on-3-60-with-adrenaline/

## Current decision

- Accept `ULUS10307` as the confirmed PSP content folder for the Ben 10 test fixture.
- Do not ask the user to inspect VitaShell file properties; that route was unavailable in their setup.
- Do not edit the user's taiHEN config or install any plugin.
- Before a title-scoped plugin can be safely configured, obtain the actual LiveArea/process TITLEID from the Vita app database or a harmless runtime probe. If the process TITLEID is shared by all official PSPemu bubbles, a title-scoped taiHEN entry cannot isolate one game by itself; a second-stage in-process check would be required, and the safety implications must be assessed first.

## Project status

No plugin source, SUPRX, VPK, or binary has been built or installed. The next engineering step is to investigate a read-only way to determine the LiveArea/process TITLEID for this Chovy-Sign bubble and whether the official PSP emulator process is shared across bubbles.