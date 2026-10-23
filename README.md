# DCLK-Flasher

Web firmware flasher for the **DCLK Dual MIDI Clock** (.axs instruments).
Runs in Chrome or Edge and talks to the STM32F411 Black Pill's built-in ROM
DFU bootloader over WebUSB. No drivers on Mac, a one-time WinUSB driver
(Zadig) on Windows, a udev rule on Linux.

It uses the same DfuSe engine as the MC8P MkII flasher (`dfuse.js`, unchanged),
so all of the Black Pill USB workarounds carry over: blank words are skipped,
failing blocks are split down until they get through, half-received blocks
are aborted, every byte is read back, and bad sectors are repaired.

## Repo layout

| File | What it is |
| --- | --- |
| `index.html` | The flasher page |
| `dfuse.js` | STM32 DfuSe over WebUSB: HEX/BIN parsing, erase, write, verify, start |
| `manifest.json` | Which firmware the "Latest release" option flashes |
| `firmware/` | DCLK release builds |
| `test/` | Simulated STM32F411 ROM bootloader and tests (`npm test`) |
| `test/fixtures/` | MC8P MkII binaries the library tests were built around |

## Hosting

Turn on GitHub Pages (Settings > Pages > Deploy from branch > `main` / root).
WebUSB needs HTTPS, which Pages provides.

## Releasing new firmware

1. In Arduino IDE, use an upload method with **no bootloader offset**
   (STM32CubeProgrammer DFU or SWD, not HID Bootloader), so the image is
   linked at `0x08000000`.
2. **Sketch > Export Compiled Binary** and copy the `.hex` into `firmware/`.
   Quick check: after `:020000040800F2`, the next line should start
   `:10000000`, not `:10400000`.
3. Update `manifest.json`: the `path` (if the file name changed) and `version`.
4. Run `npm test` (Node 18+, no packages). The "DCLK release" group checks
   the image starts at `0x08000000`, has a valid vector table, stays out of
   the last flash sector, and flashes cleanly on the simulated flaky board.
5. Commit and push.

Optional manifest fields:

- `"video": "https://youtu.be/..."` shows the tutorial button.
- `"note": "..."` inside a build is appended to the success message.
- A `.bin` part needs `"offset"` (relative to `0x08000000`); `.hex` parts
  carry their own addresses, so prefer `.hex`.

## Settings survive a flash

The flasher only erases the sectors the firmware covers. The STM32 core's
emulated EEPROM (where DCLK saves LED brightness and MIDI options) lives in
the last sector, `0x08060000`, so it is left alone. If the firmware ever
grows past 384 KB the release test will fail before that becomes a problem.

## Debug options

- `?block=256` in the page URL forces smaller USB transfers.
- **Download Log** saves the full session log for bug reports.
