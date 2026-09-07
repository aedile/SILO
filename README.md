# SILO

**Atari's 1980 Missile Command, emulated on an ESP32-C6 Fiesta medal.** A 6502,
the MADSEL video side-channel, POKEY sound, and tilt-driven trackball — running
at the cabinet's own 61.035 Hz with every frame drawn.

A San Antonio Fiesta medal is a collectible pin. This one plays Missile Command.

---

## 🎮 Quick Start Guide

### How to Play

You hold the medal upright, like a phone.

| Control | What it does |
|---|---|
| **Tilt left / right** | Move the crosshair sideways |
| **Tilt toward / away from you** | Move the crosshair up and down |
| **Middle button** | Launch a missile |
| **Power button, short press** | Insert a coin and start |
| **Power button, hold 1 second** | Power off |

The tilt is measured against **however you are holding it right now**, not
against gravity. A neutral pose is captured the first time the medal reads its
sensor, and again every time you coin up — so if the crosshair drifts, press the
power button to re-centre.

The middle button cycles through the three missile bases: left, then centre,
then right, then back. The cabinet had a button per base and thirty missiles a
wave; one button that rotates keeps all thirty reachable instead of stranding
twenty of them.

### Charging

USB-C. The medal runs from its battery when unplugged. Holding the power button
for a second cuts the battery rail — do that before storing it, or it will sit
and drain.

### Troubleshooting

**The crosshair drifts on its own.** Press the power button to coin up, which
re-captures the neutral pose. If it still drifts, you are probably holding it at
a steeper angle than when the pose was taken.

**Nothing happens when I press the middle button.** The base it picked may be
out of missiles. Press again to advance to the next one.

**It won't turn on.** Charge it. If it still won't, hold the reset button on the
bottom edge.

---

## 🔨 Building Your Own

You need a **Waveshare ESP32-C6-LCD-1.69**, the Missile Command ROM set, Docker,
and esptool.

```sh
git clone https://github.com/aedile/SILO.git
cd SILO
python3 tools/convert_roms.py /path/to/missile
docker run --rm -v "$PWD":/project -w /project espressif/idf:v5.3.4 \
    idf.py -B build_docker build
cd build_docker && esptool --chip esp32c6 -p /dev/cu.usbmodemXXXX \
    -b 460800 write_flash @flash_args
```

Flashing has to run **from inside `build_docker`** — the paths in `flash_args`
are relative to it — and **from the host**, because Docker Desktop on macOS
cannot reach USB.

### The ROMs

Not included, and never will be. You need MAME's `missile` set: the six 2 KB
program ROMs `035820.02` through `035825.02`, and the 32-byte write-mask PROM
`035826.01`. The converter checks every CRC and refuses anything that does not
match.

`tools/convert_roms.py` writes `main/roms/missile_roms.h`, which is gitignored.

---

## 🔬 Technical Details

### The original hardware

One 6502 at 1.25 MHz, one POKEY, a trackball and three fire buttons. That is the
whole board — and the interesting part is what is missing from it.

**There is no tilemap, no sprites and no graphics ROM.** The picture is a
256×231 bitmap, two bits per pixel, living in the CPU's own 16 KB of RAM. Work
RAM and video RAM are the same chips.

Writing a pixel does not look like writing memory. The schematics call the
mechanism **MADSEL**: when the CPU fetches an opcode whose low five bits are 1 —
the indexed-indirect `(zp,X)` addressing modes — a counter is armed, and the bus
access exactly five cycles later is diverted away from normal address decoding
and into video RAM as a two-bit pixel write. The bottom 32 rows get a third bit
from a scattered set of addresses, which is what gives the ground and the cities
their extra colours, and costs the CPU an extra cycle each time.

The CPU also runs at half speed over those bottom 32 lines, while the video
circuit has the RAM to itself.

### Two things that had to be right before anything appeared

The screen was blank white for a long time, and both causes are worth recording.

**VBLANK is scanlines 0–23, not the bottom of the frame.** IN1 bit 7 is active
over the first 24 lines. The interrupt handler only counts a frame tick when it
sees that bit set. With it wrong, the game booted, ran real code, executed
millions of instructions, and sat in its wait loop forever drawing nothing. It
looked like a dead emulator; it was a live one being told the frame never ended.

**The I/O decode has to honour the mirrors, and the second DIP bank is not a
port.** The R8 switches are read through the POKEY's ALLPOT register rather than
through any address of their own.

### Making it fit in 160 MHz

The first working build ran at 44 of the 61 frames it needed, with CPU emulation
alone eating 858 ms of every second.

The problem was the CPU core. A cycle-stepped 6502 is the honest way to model
the chip, and MADSEL genuinely is a cycle-level effect — but the core we started
with carries its whole pin state in a `uint64_t`, and on a 32-bit RISC-V every
pin poke costs two instructions. It worked out at roughly **200 CPU clocks per
emulated 6502 cycle**, about four times the budget.

`core/m6502fast.h` replaces it with an instruction-stepped core. That sounds
like it should make MADSEL harder and it makes it easier: the eight `(zp,X)`
opcodes are all six cycles long and their data access is always the sixth, so at
instruction granularity MADSEL is simply *the data access of a `(zp,X)`
instruction*. Emulation dropped from 858 ms/s to about 350.

Before trusting it, it was checked two ways: it passes **Klaus Dormann's 6502
functional test** in full, decimal mode included, and the old cycle-stepped core
was rebuilt alongside it to confirm the two render frame-for-frame identically.

The other third came from skipping the main loop's wait for the frame tick. That
is only sound while `$9F` is zero — the loop is `LSR $9F / BCC`, and `LSR`
rewrites the byte, so any other value would fall out of the loop on a later pass.

| | before | after |
|---|---|---|
| Emulation | 858 ms/s | ~350 ms/s |
| Frames emulated | 44/s | 61/s |
| Frames drawn | 15/s | 61/s |

### Video

The visible window is lines 25–255, which is where MAME's cliprect sits. The
cabinet was a horizontal 4:3 monitor, so with the medal upright the picture is
letterboxed to 240×180 with a 50-row bar above and below. The enclosure does not
have to change.

### Audio

One POKEY. The board's audio HAL had a startup fault worth knowing about: for
about two seconds after I2S starts, its accounting of consumed DMA samples runs
ahead of reality, the mixer over-renders roughly eighteenfold, and the driver
drops the excess — which sounds like the first two seconds of every boot are
badly distorted. The fix is a pending-sample buffer, flushed before anything new
is rendered.

---

## 📁 Project Structure

```
core/           platform-independent emulation, shared with the host harness
  missile.c       memory map, MADSEL, IRQ timing, video and audio glue
  m6502fast.h     instruction-stepped NMOS 6502
  pokey.c         POKEY sound
main/           the ESP32 application
  main.cpp        frame pacing and catch-up
  render.cpp      letterboxed 240×180 presentation
  input.cpp       tilt-to-trackball and the button
components/     display, IMU and audio HAL for the Waveshare board
host/           builds the same core on a Mac or Linux box; frames to PPM, audio to WAV
tools/          ROM converter
```

## 💻 Running It on Your Computer

The core is plain C with no ESP-IDF in it, so it builds and runs on a desktop.
This is how nearly all the debugging was done.

```sh
cd host && make
./harness /tmp/out 20 --every 1 --r10 0x82 --wav /tmp/mc.wav \
    --script "2.0:coin=1,2.2:coin=0,3.0:start=1,3.2:start=0,6.0:tx=3,7.2:fire1=1"
```

Frames come out as PPM, audio as a WAV. `--r10` and `--r8` set the DIP banks
(`0x82` is free play). The script takes `coin`, `start`, `fire1`–`fire3`, and
`tx`/`ty` for trackball counts per frame.

## ⚙️ Configuration

| What | Where |
|---|---|
| DIP switches | `mc_set_dips()` in `main/main.cpp` — `0x82, 0x73` is free play, English, 6 cities |
| Tilt sensitivity | `DEADBAND_DEG`, `FULL_DEG`, `MAX_COUNTS` in `main/input.cpp` |
| Tilt direction | `X_SIGN`, `Y_SIGN` in `main/input.cpp` |
| Screen brightness | `DISPLAY_BRIGHTNESS_ACTIVE` in `components/display/include/display.h` |

## 📌 Status and Known Gaps

Runs at full speed with sound, every frame drawn, nothing skipped or dropped.

- Cocktail flip-screen is implemented but untested — there is no second player
  on a pin.
- The two unused IN1 inputs read as zero, matching MAME.
- Sound is one POKEY at a fixed mix level; the cabinet had a volume pot.

## 📄 Legal Notice

### ROM files

No ROMs here. Missile Command is © 1980 Atari. Dumping a board you own is one
thing; downloading a set you do not is another, and that is between you and
Atari's current owners. This project ships a converter, not a game.

### Third-party code

See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). The machine model and the
POKEY are written from MAME (BSD-3-Clause).

### Disclaimer

Not affiliated with, endorsed by, or connected to Atari, its successors, or the
Fiesta San Antonio Commission.

## 🙏 Credits

The MAME team, whose `missile.cpp` is the reason MADSEL is documented at all.
Klaus Dormann, whose 6502 functional test caught what guessing would not.

## 📜 License

[0BSD](LICENSE) — no attribution required, no conditions.
