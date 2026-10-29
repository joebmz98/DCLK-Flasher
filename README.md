# DCLK-Flasher

Web tools for the **DCLK Dual MIDI Clock** (.axs instruments, STM32F411CEU6
Black Pill): a firmware flasher and a settings editor. Nothing to install.
They run in **Chrome or Edge on a computer** (not Firefox, Safari or phones)
and need a USB-C data cable.

Opening the site shows a start page with two tools:

| Tool | Opens | Talks to DCLK over | DCLK needs to be |
| --- | --- | --- | --- |
| **Update firmware** | `index.html#flash` | WebUSB, to the STM32 ROM DFU bootloader | In DFU mode |
| **Change settings** | `editor.html` | Web MIDI + SysEx | Running normally, firmware v1.3 or later |

The start page reads `manifest.json` and shows the latest release version on
the badge and the flasher card.

## Change settings (DCLK Editor)

### Using it

1. Plug DCLK into the computer and let it start up as normal.
2. Open **Change settings** and click **Connect DCLK**.
3. Choose **Allow** when the browser asks to control MIDI devices (the editor
   needs SysEx access). It finds the MIDI port named DCLK.
4. Change anything you like. Changes reach DCLK straight away.

**Try demo** runs the whole page against a simulated DCLK with no hardware.
The demo's pretend USB host plays at 124 BPM: 16 bars, then 2 bars stopped,
so you can see how outputs follow the host in USB clock mode.

### What it can change

Everything the panel's MIDI, LEDS and SCREEN menus change:

| Setting | Panel name | Options |
| --- | --- | --- |
| Clock source | CLK SRC | **Internal** (DCLK's own tempo) or **USB in** (follow MIDI clock from the USB host) |
| Tempo | encoder | 30 to 300 BPM (internal clock only) |
| Host transport | EXT TPT | **Follow host** (outputs run with the host's Start, Stop and Continue) or **Free** (incoming clock sets the tempo only) |
| Wait length | WAIT LEN | 1, 2, 4, 8 or 16 bars |
| Quantised stop | STOP Q | On or off |
| MIDI OUT 1 / 2 divide | CH1 DIV / CH2 DIV | /1 /2 /3 /4 /6 /8 |
| USB clock out follows | USB OUT | OUT 1 or OUT 2 (off while following USB clock) |
| LED brightness | LEDS | Low, Mid, High |
| Screen saver | SAVER | Off, 1, 5, 10 or 30 min |

It can also start and stop each output, with the same quantised behaviour as
the CH1 and CH2 buttons.

### How changes behave

- **Live:** every change is sent to DCLK as you make it. There is no separate
  save step.
- **Saved automatically:** DCLK writes its settings to flash once every output
  is stopped (when following a host, once the host stops), the same rule as
  the panel. Flash writes pause the CPU, so this never happens mid-clock. The
  page shows whether changes are still waiting to be saved.
- **In sync with the panel:** the editor polls DCLK several times a second.
  It shows the tempo (or the measured USB tempo), the beat and bar position,
  the state of both outputs, and any setting changed on the panel.
- **Reset to defaults** stops every clock and restores the factory settings
  (click twice to confirm).

### Can't connect?

- Use Chrome or Edge on a computer.
- If MIDI access was blocked, allow it again from the icon in the address bar.
- Close any app holding DCLK's MIDI port (DAW, MIDI monitor). On Windows only
  one program can use a port at a time.
- Older firmware does not answer the editor: update to v1.3 or later with the
  flasher first. Saved settings survive the update (see below).
- Previews embedded in another site can't reach MIDI devices. Open the page
  in its own tab.

### SysEx protocol

Messages are `F0 7D 44 43 <cmd> <data...> F7` (7D = non-commercial ID,
`44 43` = "DC"). Firmware 1.3 implements protocol version 1. Values are
14-bit, sent as `<lo 7 bits> <hi 7 bits>`.

| Cmd | Request | Reply |
| --- | --- | --- |
| `01` | GET_INFO | `41`: protocol, firmware version string |
| `02` | GET_STATE | `42`: protocol, 10 settings, output flags, USB flags, measured USB tempo ×10, beat, bar |
| `03` | SET `<param> <lo> <hi>` | `42` |
| `04` | TRANSPORT `<out> <action>` (out 0, 1 or 2 = both; action 0 toggle, 1 start, 2 stop) | `42` |
| `06` | FACTORY RESET `44 45 46` ("DEF") | `42` |

Settings in order: 0 tempo, 1 clock source, 2 wait length (bars), 3 stop Q,
4 OUT 1 divide, 5 OUT 2 divide, 6 USB out, 7 LED level, 8 screen saver
(minutes), 9 host transport (1 = follow host). The full description is at the
top of the firmware sketch.

## Update firmware (flasher)

### Using it

1. Connect DCLK straight to the computer (not a hub). Stop any running clock
   and close apps using it as a MIDI device.
2. Enter DFU mode: **hold BOOT0, tap NRST**, keep holding BOOT0 for about a
   second, then let go (or hold BOOT0 while plugging in). The OLED and LEDs
   stay off. It can take two or three tries.
3. Pick the firmware and click **Flash Firmware**, then choose
   **STM32 BOOTLOADER** in the popup.
4. Wait for **Done**. DCLK restarts by itself; if not, press NRST once.

The page won't leave the flasher while a flash is running.

### Options

- **Latest release:** the files listed in `manifest.json`, downloaded from the
  site.
- **Custom file:** a `.hex` or `.bin` you built. Prefer `.hex`, which carries
  its own address. For a `.bin`, enter the start address (default
  `0x08000000`; small numbers such as `0x4000` are taken as offsets from
  the start of flash).
- **Verify after writing** (on by default) reads back every byte written and
  repairs any bad sector.
- **Start firmware when done** (on by default).

It uses the same DfuSe engine as the MC8P MkII flasher (`dfuse.js`), so the
Black Pill USB workarounds carry over: blank words are skipped, failing blocks
are split down until they get through, half-received blocks are aborted,
every byte is read back and bad sectors are repaired. A few `WARN` lines in
the log are normal.

### Troubleshooting

- **STM32 BOOTLOADER not in the list:** the board isn't in DFU mode yet. If
  the computer lists DCLK as a MIDI device, it's still running its normal
  firmware. Try again, and try another cable (many USB-C cables are charge
  only).
- **Windows, "Access denied" / "Unable to claim interface":** install the
  WinUSB driver once with [Zadig](https://zadig.akeo.ie/) (Options > List All
  Devices > STM32 BOOTLOADER > WinUSB > Replace Driver).
- **Linux, can't open the device:** add a udev rule, e.g.
  `SUBSYSTEM=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="df11", MODE="0666"`
  in `/etc/udev/rules.d/50-stm32-dfu.rules`, then replug.
- **Flashed but doesn't start:** release BOOT0 and press NRST. A custom file
  must be built to start at `0x08000000` (no HID or UF2 bootloader offset);
  the flasher warns you when it isn't.
- `?block=256` on the flasher URL forces smaller USB transfers.
- **Download Log** saves the whole session log for bug reports.

## Settings survive a flash

The flasher only erases the sectors the firmware covers. The STM32 core's
emulated EEPROM, where DCLK saves its settings (tempo, clock source, MIDI
options, LEDs, screen saver), lives in the last sector, `0x08060000`, so it
is left alone. Settings saved by older firmware carry over: the clock source
settings added in v1.3 start at Internal / Follow host. If the firmware ever
grows past 384 KB the release test fails before that becomes a problem.

## Repo layout

| File | What it is |
| --- | --- |
| `index.html` | Start page (choose a tool) and the flasher, at `#flash` |
| `editor.html` | DCLK Editor, including the demo device |
| `dfuse.js` | STM32 DfuSe over WebUSB: HEX/BIN parsing, erase, write, verify, start |
| `manifest.json` | Version and firmware parts the "Latest release" option flashes |
| `firmware/` | DCLK release builds |
| `test/` | Simulated STM32F411 ROM bootloader and flasher tests (`npm test`) |
| `test/fixtures/` | MC8P MkII binaries the library tests were built around |
| `theme.css` | Shared 1-bit pixel theme for both pages (fonts, frames, buttons, controls) |
| `assets/fonts/` | Self-hosted pixel fonts and their licences (`LICENSES.txt`) |
| `assets/walker.png` | The walking character from the DCLK front panel, as a 1-bit sprite |

## Look and feel

Both pages follow the DCLK / MC8P MkII front panels: black and white only,
white "silkscreen" lines on black, stepped pixel corners, a starfield and
square-wave clock traces. One art pixel is 2 CSS px.

- **Fonts:** Alagard for the wordmark and headings (the DCLK panel's own
  lettering), Silkscreen for labels and buttons, Pixelify Sans for body text,
  VT323 for the debug terminal. All are self-hosted, so the pages work with no
  third-party requests. Alagard and Silkscreen are sized on their pixel grids
  (Alagard 32/64/96 px, Silkscreen 16/32 px) to stay sharp.
- **Frames:** `.frame`, `.frame-dots`, `.frame-double` and `.frame-fill` are
  9-slice SVGs in `theme.css` (`--px-*`). Dotted means available, solid means
  hover or active, filled (inverted) means selected or important.
- **Editor screen:** the editor draws DCLK's 128x32 main screen into a canvas
  with the Picopixel bitmaps from Adafruit GFX, the same font and layout the
  firmware uses (`BPM: 120` at size 4, four beat bars). The CH1 / CH2 buttons
  are the panel's double-ring touch pads: hollow when stopped, filled while
  running, blinking while waiting for the boundary.

## Hosting

Turn on GitHub Pages (Settings > Pages > Deploy from branch > `main` / root).
WebUSB and Web MIDI both need HTTPS, which Pages provides.

## Releasing new firmware

1. In Arduino IDE, build with the **DCLK Dual MIDI Clock** board option and an
   upload method with **no bootloader offset** (STM32CubeProgrammer DFU or
   SWD, not HID Bootloader), so the image is linked at `0x08000000`.
2. **Sketch > Export Compiled Binary** and copy the `.hex` into `firmware/`.
   Quick check: after `:020000040800F2`, the next line should start
   `:10000000`, not `:10400000`.
3. Update `manifest.json`:
   - `version`: shown on the start page, the badge and the release option.
   - `builds[0].parts`: each file to flash, in order. A `.hex` needs only
     `path`; a `.bin` also needs `offset` (e.g. `"0x4000"`). Parts must not
     overlap.
   - `note` (optional): shown after a successful flash of the release.
   - `video` (optional): a tutorial link, shown as a button on the flasher.
4. If the editor protocol changed, update `editor.html` to match and the
   "firmware v1.x or later" wording on both pages.
5. Run `npm test` (Node 18+, no packages). The "DCLK release" group checks
   the image starts at `0x08000000`, has a valid vector table, stays out of
   the last flash sector, and flashes cleanly on the simulated flaky board.
6. Commit and push.
