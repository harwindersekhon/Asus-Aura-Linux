# Asus-Aura-Linux

Command-line tool to control ASUS AURA ARGB lighting on Linux via USB HID protocol.

Drives ASUS AURA LED Controller (USB 0b05:19af and related product IDs) directly from userspace with zero dependencies beyond Python 3 standard library. Supports AIO pump heads, case fans, and RGB strips that connect to ARGB headers on ASUS mainboards.

## Requirements

- Python 3.6 or later
- Linux with `hidraw` kernel module (usually built-in)
- ASUS mainboard with a USB AURA LED Controller
- User must be in the `wheel` group (or equivalent) for the udev permission grant

## Tested hardware

Developed and verified against exactly one machine:

| | |
|---|---|
| Board | ASUS Prime Z690-P |
| Controller | `0b05:19af` USB AURA LED Controller |
| Firmware | `AULA3-AR32-0207` |
| Topology | 3 addressable channels, up to 120 LEDs each |
| OS | Rocky Linux 10 |

The script also matches product IDs `0x1867`, `0x1872`, `0x1889`, `0x18a3`,
`0x18f3` and `0x1939`, but **none of those are tested**. The protocol quirks
documented below were established on the hardware above and may differ on other
AURA controllers — in particular, controllers whose built-in effect engine does
work would be better served by using it. Reports welcome.

## Installation

1. Copy the `aura` script to your PATH:
```bash
cp aura ~/.local/bin/aura
chmod +x ~/.local/bin/aura
```

Alternatively, install system-wide:
```bash
sudo cp aura /usr/local/bin/aura
sudo chmod +x /usr/local/bin/aura
```

2. Install the udev rule:
```bash
sudo cp 60-aura-led.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

3. Ensure your user is in the `wheel` group:
```bash
groups
```

If not present, add yourself (then log out and log back in):
```bash
sudo usermod -a -G wheel $USER
```

After installation, run `aura info` to verify the device is found. Do not use `sudo aura`—it will fail with "command not found" because sudo resets PATH to exclude user binary directories.

## Usage

### Quick examples

```console
aura info                           # Display controller and channel info
aura static red                     # Solid red on all channels
aura static 0,128,255              # Solid colour by RGB
aura static '#00ffcc'              # Solid colour by hex
aura effect breathing cyan          # Breathing animation on all channels
aura effect rainbow                # Cycling rainbow
aura ruler                         # Count the LEDs actually on each header
aura effect ember --leds 12        # "ember" theme: drifting coals
aura effect ember '#3aa0ff'        # ...the same fire, burning cold
aura pulse                          # Breathing blue (default) on all channels
aura off                            # Turn off all LEDs
aura --channel 0 static white      # Solid white on channel 0 only
aura --device /dev/hidraw5 info    # Use specific device node
```

### Subcommands

| Command | Args | Description |
|---------|------|-------------|
| `info` | none | Probe the controller: read firmware, channel count, LED counts, config table (read-only) |
| `static` | `<color>` | Paint every LED one solid colour (instant) |
| `effect` | `<effect> [color]` | Run an effect; animated ones block until Ctrl-C |
| `direct` | `<color> [--leds N]` | Set every LED to one colour via host direct mode |
| `pulse` | `[color] [--leds N]` | Host-driven breathing animation (default blue, blocks until Ctrl-C) |
| `off` | none | Turn off all LEDs |
| `ruler` | none | Paint four-LED colour bands so you can count what is really on a header |
| `spectrum` | `[--leds N]` | One fixed hue per LED, evenly spaced around the colour wheel (instant) |
| `remember` | `--leds N` | Store the LED count persistently so effects scale correctly by default |

### Global options

These go **before** the subcommand, e.g. `aura --channel 0 static white`.

`--device /dev/hidrawN` : Use a specific hidraw node instead of auto-detecting

`--channel N|all` : Target a single channel (default: `all`). Valid indices depend
on what the controller reports; run `aura info` to see the count.

### Per-command options

`--leds N` : LED count for `effect`, `direct` and `pulse` (default: read from controller)

## How many LEDs? (`--leds`)

**The controller reports its maximum capacity, not what you have plugged in.**
`aura info` says 120 LEDs per channel on every channel because 120 is the most
this controller will drive — a 12-LED fan on that header still reports 120.

Effects that vary *along* the strip — `ember`, `rainbow`, `chase` — spread
themselves across whatever count they are given. Told 120 when only 12 exist,
they render the pattern over 120 positions and the fan shows the first 10% of
it: a nearly flat slice that drifts in brightness. It looks like a solid colour.

Run `aura ruler` to measure. It paints LEDs in blocks of four —
red, green, blue, yellow, magenta, cyan, white, orange, then dim grey past 32 —
so counting the bands that light up gives the real figure to within four. Fans
showing red/green/blue and nothing else have 12 LEDs:

```bash
aura ruler
aura effect ember --leds 12
```

### Making the LED count persistent

Once you know the real count, you can store it so effects use it by default:

```bash
aura remember --leds 12
aura info
aura effect ember  # now uses 12 LEDs without needing --leds
```

The count is stored in `~/.config/aura/config` (or `$XDG_CONFIG_HOME/aura/config`).
It resolves in this order for any command that needs it:

1. Explicit `--leds` flag on the command line (highest priority)
2. Configured value from `aura remember` for that channel or globally
3. Controller's reported maximum (lowest priority)

You can store counts per-channel: `aura --channel 0 remember --leds 16` stores the
count for channel 0 only. `aura remember --leds 12` (without `--channel`) stores a
global default that applies to all channels.

`static`, `pulse` and `off` paint every LED the same colour, so they do not care about the count.

## Colour formats

Colours can be specified by:
- **Name**: `red`, `green`, `blue`, `cyan`, `magenta`, `yellow`, `white`, `black`, `off`, `orange`, `purple`, `pink`
- **Hex**: `#rrggbb` (e.g. `#00ff00` for green)
- **RGB decimal**: `r,g,b` (e.g. `255,0,128`)

Components must be in the range 0–255; out-of-range values are rejected rather than clamped.

## Effects

### Static (instant)
- `static` : Solid colour
- `off` : All LEDs off
- `spectrum` : One fixed hue per LED, evenly spaced around the colour wheel (the still
  counterpart to `rainbow`; with 12 LEDs that is one hue every 30 degrees)

### Animated (run until Ctrl-C)
- `breathing` : Fade in and out smoothly
- `flashing` : Blink on and off
- `cycle` : Cycle through all hues
- `rainbow` : Rainbow gradient across LEDs
- `chase` : Colour chases along the strip
- `flicker` : Random brightness flicker
- `ember` : Drifting coals (see **Themes**)

### Themes

A theme is an effect that is a whole look rather than one colour animated, so it
brings its own palette and needs no colour argument.

- `ember` : A bed of coals. Three waves of different spatial frequency drift
  along the strip at speeds that share no common factor, summing into a heat
  field that is read through a blackbody ramp — `#380500` coal at the cold end,
  orange at the middle, an `#ffaf00` amber tip where two waves crest together.
  The hot end climbs towards amber rather than towards white: lifting every
  channel equally produces a pale peach that a diffused fan ring simply reads as
  white, which is not what a coal looks like. The
  layers take minutes to line up again, so the fire never visibly loops, and a
  slow global breath rides on top.

  Passing a colour rebuilds the ramp around it instead of overriding the look:
  `aura effect ember '#3aa0ff'` gives the same drifting fire in blue. The
  spatial frequencies are whole numbers, so the pattern joins up seamlessly on
  ring-shaped headers (pump heads, fans) as well as along a strip.

  **Pass `--leds`** with the real LED count (see above) or the fan will show one
  flat slice of the fire rather than the whole of it. Short strips cannot sample
  the finer layers — a layer needs two LEDs per cycle — so `ember` drops the ones
  that would alias and shares their weight among the rest. A 12-LED fan runs the
  1- and 3-cycle layers: one flare travelling around the ring with three smaller
  coals turning under it.

Animated effects target 30 FPS on the host. The real rate is lower on strips with
many LEDs, since each frame is several HID packets and the controller needs a
short gap between writes — 3 channels of 120 LEDs measures about 13 FPS. Animation
phase is taken from the clock rather than a frame counter, so an effect runs at
the same speed regardless; a slow strip drops frames instead of slowing down.

## Limitations

**This controller's built-in effect engine does not render while the host holds the device.**

Selecting any built-in effect mode other than direct mode (0xFF) takes the channel away from the host and the LEDs go dark. Therefore, **all effects in this tool are animated on the host and streamed as direct-mode frames**. Animated effects (`breathing`, `flashing`, `cycle`, `rainbow`, `chase`, `flicker`) will block the terminal until you press Ctrl-C.

**Colour is volatile—it lives in the controller's RAM and is lost on power-off.** There is no `--save` flag and no persistence across reboot. To reapply colour automatically at login, create a systemd user service that runs the desired `aura` command at startup.

## Protocol notes

These findings are the result of extensive hardware testing and are useful for anyone porting this tool to other platforms:

1. **Direct mode entry**: Direct mode must be explicitly entered per channel by sending effect mode 0xFF:
   ```
   OP_EFFECT (0x35), channel, 0x00, 0x00, 0xFF
   ```
   Without this, the controller accepts direct frames but silently renders nothing.

2. **Apply flag placement**: In a direct frame, the apply flag (0x80) is ORed **into the channel byte** (packet byte 0x02), not into the LED count byte. Folding the flag into the count makes every frame a no-op because the controller reads a nonsense LED count.

3. **Packet structure**: 
   - Total: 65 bytes
   - Byte 0: Report ID (0xEC)
   - Byte 1: Opcode (0x40 for direct, 0x35 for effect, etc.)
   - Direct frames: max 20 LEDs per packet (60 payload bytes), start-LED at byte 0x03, count at byte 0x04
   - Writes sent back-to-back are dropped; a small delay (1–4 ms) between packets is required

4. **A second apply blanks the first**: The controller applies a direct frame as a unit. Sending
   LEDs 0–11 with the apply flag and then sending a second batch to clear LEDs 12–119 switches OFF
   the 12 that were just set. A short frame must carry its black tail in the SAME batch — which is
   what the `pad_frame` function does.

## Troubleshooting

**No device found**
- Ensure the udev rule is installed and reloaded: `sudo udevadm control --reload-rules && sudo udevadm trigger`
- Verify the ASUS AURA LED Controller is visible: `lsusb | grep 0b05`
- Check that `/dev/hidraw*` exists and is readable by your user

**Permission denied opening /dev/hidraw***
- Reinstall the udev rule and reload: `sudo udevadm control --reload-rules && sudo udevadm trigger`
- Verify your user is in the `wheel` group: `groups`
- Confirm the ACL actually landed: `getfacl /dev/hidraw0` should list your user with `rw-`
- The `uaccess` tag only applies to the logged-in local seat; over SSH you are covered by the `wheel` group fallback instead

**LEDs go dark on their own**
- Another program took the channel. Anything that selects a built-in AURA mode —
  OpenRGB, or the board's own Aura firmware after a reboot — pulls the channel out
  of direct mode, and this controller then renders nothing. Re-run `aura static <color>`
  to take it back.
- Note that stopping an animated effect with Ctrl-C does *not* go dark: the last
  streamed frame stays until something else changes it or the machine powers off.

**sudo aura: command not found**
- Expected, and you do not need `sudo`. It resets PATH to `secure_path`, which on most
  distributions excludes both `~/.local/bin` and `/usr/local/bin`. The udev rule already
  grants you access, so drop the `sudo`. If you genuinely need to run it as root, give the
  full path: `sudo /usr/local/bin/aura static red`.

## Licence

MIT. See LICENSE file in this repository.
