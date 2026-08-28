# RP2350 HDMI Hexpansion — Design Plan

Working name: **Hexi-GFX**. A Tildagon hexpansion that turns the EMF badge into a
machine with a real display output: an RP2350 driving DVI/HDMI over HSTX, with its
own flash and PSRAM, presenting itself to the badge as a normal hexpansion by
emulating the identification EEPROM in firmware.

Status: **design plan / pre-schematic**. Nothing here has been built yet. Every
figure marked *(est.)* is a modelled estimate, not a quote.

---

## 1. What the badge actually gives us

All of this is taken from the official hardware sources
(`emfcamp/badge-2024-hardware`) rather than from memory, because the whole design
hangs off it.

### 1.1 Hexpansion edge connector (20 pads, SFP-style)

Pads 1–10 are on the bottom copper, 11–20 on the top copper.

| Pad | Signal | Pad | Signal |
|----:|--------|----:|--------|
| 1 | GND | 11 | GND |
| 2 | LS_A | 12 | HS_F |
| 3 | LS_B | 13 | HS_G |
| 7 | SDA | 14 | GND |
| 8 | SCL | 15 | +3V3 |
| 9 | HEXP_DET | 16 | +3V3 |
| 10 | LS_C | 17 | GND |
| 11 | LS_D | 18 | HS_H |
| 12 | LS_E | 19 | HS_I |
| 13 | GND | 20 | GND |

Source: `hexpansion/hexpansion.kicad_sch` + `tildagon.pretty/hexpansion-edge-connector.kicad_mod`.

### 1.2 What each signal is worth to us

* **+3V3 only — there is no 5 V rail on the connector.** Each port is fed through
  an MT9700 current-limited load switch with an 11k3 set resistor, i.e. **~600 mA**.
  The badge enables the port in software, so we are dead until the badge decides
  to power us. Anything needing 5 V (the HDMI +5 V pin) has to be boosted on-board.
* **HS_F..HS_I go straight to dedicated ESP32-S3 GPIOs.** No mux, no expander.
  From the badge firmware's port table:

  | Port | hs_1 | hs_2 | hs_3 | hs_4 |
  |------|-----:|-----:|-----:|-----:|
  | 1 (A) | 39 | 40 | 41 | 42 |
  | 2 (B) | 35 | 36 | 37 | 38 |
  | 3 (C) | 34 | 33 | 47 | 48 |
  | 4 (D) | 11 | 14 | 13 | 12 |
  | 5 (E) | 18 | 16 | 15 | 17 |
  | 6 (F) | 3 | 4 | 5 | 6 |

  `hs_1..hs_4` correspond to `HS_F..HS_I` on the connector. These are matrix-routed
  (not IOMUX) on the ESP32-S3, so a hardware SPI master on them tops out around
  **40 MHz** — call it **2.5–5 MB/s** in practice. This is the single most important
  number in the whole design (see §3).
* **LS_A..LS_E hang off AW9523B I/O expanders** behind the I2C bus. They are slow —
  hundreds of microseconds per toggle. Useless for data, perfect for reset and
  attention lines.
* **SDA/SCL are per-port**, isolated behind a TCA9548A mux on the badge, with the
  badge providing the 2k2 pull-ups. So we get a private I2C bus and can own
  address 0x50 without colliding with anything else.
* **HEXP_DET** has a weak pull-up on the badge; we tie it to GND. That is what makes
  the badge power the port at all.

### 1.3 Mechanical — the hexagon, and how much of it we get

**The hexpansion is a regular hexagon, 32 mm across flats, with the insertion tab on one
flat.** From `hexpansion.kicad_pcb`: vertices at (0, 9.66), (16, 0), (32, 9.66),
(32, 26.98), (16, 36.64), (0, 26.98), tab tip at x = 0 spanning y = 12.07–24.57.

That means **each flat is only ~18.5 mm long** (side = W/√3 for a hexagon of width W).
This is the number that governs the whole physical design, and it is much smaller than
the 32 mm bounding-box width suggests.

* **Board thickness 1.0 mm** (template stackup is 0.91 mm core + 2×35 µm Cu ≈ 0.98 mm).
  Not negotiable — the badge's SFP-style receptacle expects a 1.0 mm card edge.
* **ENIG only. HASL is explicitly forbidden** — the pads must be flat to make contact.
* Two M2 mounting-hole pads, which is what will take the strain of a hanging cable.

#### How big can a hexpansion be?

From `tildagon-base.kicad_pcb`, the six SFP connectors sit at **radius 26.075 mm** from
the badge centre, 60° apart. For a hexpansion of width `W` across flats, the tile centre
is at `R = 26.075 + W/2`, and two adjacent tile centres are `D = 2·R·sin(30°) = R` apart.
Two hexagons of width `W` touch when `D = W`, so:

> **gap between neighbouring hexpansions = 26.075 − W/2 mm**

| W across flats | Flat length | Area | Gap to neighbours |
|---------------:|------------:|-----:|------------------:|
| 32 (template) | 18.5 mm | 8.9 cm² | 10.1 mm |
| 40 | 23.1 mm | 13.9 cm² | 6.1 mm |
| **44 (chosen)** | **25.4 mm** | **16.8 cm²** | **4.1 mm** |
| 48 | 27.7 mm | 20.0 cm² | 2.1 mm |
| 52.2 | 30.1 mm | 23.6 cm² | 0 — touching |

This assumes the tab tip sits at the connector footprint origin. Insertion depth puts the
tab-side flat a millimetre or two further out, which makes `R` larger and the real gaps
slightly **bigger** than the table says — so these figures are conservative.

Five of the six flats are free. The two flanking the tab are effectively unusable (they
face the badge), leaving **the outer flat and the two adjoining it** — see §4.4.

### 1.4 The EEPROM contract

The badge scans the port's I2C bus and decides addressing mode by what answers:

* `0x50`–`0x57` all answer → 1-byte addressing (24C16-style, ≤ 2 KiB)
* only `0x50` answers → **2-byte addressing**, up to 64 KiB
* only `0x57` answers → 2-byte addressing

It then reads 32 bytes from offset 0 and parses a header packed as
`<4s4sHHIHHH9s` + 1 checksum byte:

| Offset | Size | Field |
|-------:|-----:|-------|
| 0 | 4 | magic, ASCII `THEX` |
| 4 | 4 | manifest version, ASCII `2024` or `2026` |
| 8 | 2 | `fs_offset` |
| 10 | 2 | `eeprom_page_size` |
| 12 | 4 | `eeprom_total_size` |
| 16 | 2 | `vid` |
| 18 | 2 | `pid` |
| 20 | 2 | `unique_id` |
| 22 | 9 | `friendly_name`, NUL-padded ASCII |
| 31 | 1 | checksum: seed `0x55`, XOR of bytes 1–30 |

If the header validates, the badge mounts a **LittleFS** starting at `fs_offset` and
runs the app it finds there. Block size is 512 bytes for EEPROMs ≥ 8 KiB.

VID/PID have to be requested from the badge team ("UHB-IF") — **do this early**, it is
the one item with an external dependency and zero cost.

---

## 2. Product definition

**Primary mode: the badge's own screen, on a monitor.** The hexpansion mirrors the
240×240 round display, pixel-doubled to 480×480 and circular-masked to match what people
actually see on the badge. **Every existing Tildagon app gets HDMI output with no code
changes** — no API to learn, no per-app work. That property is the product.

Everything else supports it:

1. **DVI/HDMI** from the RP2350's HSTX peripheral, on a **mini-HDMI (Type C)** receptacle.
2. **Emulates the identification EEPROM** at `0x50` in RP2350 firmware, serving a 64 KiB
   LittleFS image that contains the badge-side MicroPython driver app. The RP2350 owns
   that image, so the driver updates itself with the hexpansion firmware — no separate
   EEPROM flashing step, ever.
3. A **SPI link on the four HS pins** carrying framebuffer updates, or drawing commands.
4. **16 MB QSPI flash**, **8 MB QSPI PSRAM** and a **microSD socket** for the advanced
   mode's assets. The mirroring path needs none of it (§3.1).
5. **Display-list mode** for apps that want more than 240×240 — the RP2350 holds the
   assets and renders (§3.2).
6. **USB-C power** so it does not flatten the badge battery — powering this board only,
   never the badge, under any condition (§4.5) — plus UF2, a USB CDC console, and a
   **3-pin SWD connector** for a Raspberry Pi Debug Probe.
7. **Two Qwiic / STEMMA QT sockets**, which the badge can reach through the card as an
   I2C bridge.
8. A **44 mm across-flats hexagon** rather than the 32 mm template — the smallest size at
   which the connectors fit (§4.4).

The badge stays the computer. This is a display, and secondarily a graphics card.

## 3. Two modes, and the bandwidth that shapes them

### 3.1 Primary — mirroring the badge screen

The data path already exists in the badge firmware. From
`badge-2024-software/drivers/gc9a01/display.c`:

```c
EXT_RAM_BSS_ATTR
static uint8_t tildagon_fb[240 * 240 * 2];          // :34
...
tildagon_ctx = ctx_new_for_framebuffer(tildagon_fb, 240, 240, 480,
                                       CTX_FORMAT_RGB565_BYTESWAPPED);   // :41
...
void tildagon_blit_fb(void) { flow3r_bsp_display_send_fb(tildagon_fb, 16); }  // :64
```

One static **115,200-byte framebuffer**, RGB565 **byteswapped**, living in the ESP32's
PSRAM. ctx renders into it; `tildagon_end_frame()` blits it to the panel. **Every app
funnels through `display.end_frame(ctx)`** — a single choke point, always populated, in a
fixed format. Nothing better could be asked for.

The byteswap costs us nothing: the RP2350 can swap during DMA or in the HSTX command
stream.

#### The one blocker

**The framebuffer is not exposed to MicroPython.** The `display` module publishes
`gfx_init`, `bsp_init`, `splash`, `get_fps`, `get_ctx`, `end_frame` and `hexagon` — and no
framebuffer accessor. A hexpansion app cannot read those bytes today.

This needs a small upstream change to `badge-2024-software`: a `display.get_fb()` returning
a `memoryview` of `tildagon_fb`, which is on the order of ten lines of C. **Raise it in
week 1 alongside the VID/PID request** — it is short work, but it is on someone else's
schedule, and mirroring does not exist without it. See risk 4.

#### Link arithmetic

| | Bytes/frame | @5 MB/s | @2.5 MB/s |
|---|---:|---:|---:|
| Full 240×240 frame, RGB565 | 115,200 | **43 fps** | **22 fps** |
| Half the screen changed | 57,600 | 87 fps | 43 fps |
| Typical static UI, ~10% dirty | 11,520 | 434 fps | 217 fps |

**But the link is almost certainly not the ceiling.** Mirroring cannot show more frames
than the badge draws, and a ctx-rendered MicroPython app on an ESP32-S3 is unlikely to be
doing forty. The firmware already tracks this — `st3m_gfx_fps()`, exposed as
`display.get_fps()` — so Phase 0 measures it across real apps rather than guessing. Expect
**the badge's own render rate to be what you see**, and design the link to stay out of
the way rather than to chase a number.

Dirty-rectangle updates still help, but note the badge does not track dirty regions — ctx
redraws the whole frame. Finding them means a block-`memcmp` against a second copy of the
buffer, which is cheap in MicroPython (comparing `memoryview` slices drops into C) but
needs another 115 KB on the badge. Worth having; not worth having in v1.

#### Scaling

| Scale | Output | Status |
|-------|--------|--------|
| **2×** | 480×480 pillarboxed in 640×480 @60 | **v1 mode.** Standard timings, integer nearest-neighbour, sharp and near-free |
| 3× | 720×720 pillarboxed in 1280×720 | Exact integer fit for 720p height — elegant, but needs the overclocked 74.25 MHz pixel clock |

#### Two consequences that de-risk the design

* **Memory leaves the critical path.** Source 115 KB, double-buffered 230 KB — comfortably
  inside the RP2350's 520 KB of SRAM. **The mirroring mode never touches PSRAM**, so the
  one video mode rated "needs measurement" (§3.3) no longer gates the primary product.
* **The badge-side driver becomes trivial** — read the buffer, push it, repeat. The
  display-list API becomes an opt-in extra rather than something every app author must
  learn.

#### The round display

The panel is circular, so the corners of that square buffer hold whatever ctx drew but are
never visible on the badge. On HDMI they would be. **The circular mask is on by default**,
so the monitor shows what the wearer sees; a runtime toggle reveals the full square for
debugging.

### 3.2 Advanced — the display list

For apps that want more than 240×240, the arithmetic is unforgiving. 40 MHz SPI is ~5 MB/s
at best; a 640×480 16 bpp frame is 614 KB, so pushing full frames gives **8 fps** before
any MicroPython overhead. So at full resolution **the badge sends a display list, not
pixels** — the RP2350 holds the assets and renders. Asset upload is the only bulk
transfer and happens once; a full-screen 60 fps scene then costs a few hundred bytes per
frame.

| Class | Commands |
|-------|----------|
| **Mirror** | `mirror_config` (scale, mask, background) · `mirror_frame` · `mirror_rect` · `resync` |
| Mode | `set_mode`, `get_edid`, `blank`, `set_palette` |
| Assets | `upload_asset` (to flash or PSRAM), `free_asset`, `list_assets` |
| Draw | `clear`, `rect`, `line`, `blit`, `blit_rect`, `blit_scaled`, `text`, `tilemap`, `sprite_batch` |
| Frame | `present` (flip), `vsync_wait`, `set_layer` |
| Audio | `queue_pcm` (HDMI audio data islands — no extra pins needed) |
| Bridge | `i2c_scan`, `i2c_txn` — badge reaches the Qwiic bus through the card |
| System | `ping`, `status`, `reset`, `enter_bootloader`, `fw_update` |

`resync` is a full-frame refresh on demand, kept for robustness rather than bandwidth: a
dropped or corrupted update must not be able to leave a permanent artifact on screen.

### 3.3 Achievable video modes

| Mode | Source | Colour | Buffering | Confidence |
|------|--------|--------|-----------|-----------|
| 240×240 ×2 → 480×480 in 640×480 @60 | mirror | 16 bpp | 2×115 KB in SRAM | **High — v1 primary** |
| 320×240 doubled to 640×480 @60 | display list | 16 bpp | 2×150 KB in SRAM | High |
| 640×480 @60 | display list | 8 bpp palette | 307 KB SRAM + PSRAM back buffer | High |
| 640×480 @60 | display list | 16 bpp | PSRAM-backed, line-buffer DMA ~37 MB/s | Medium — needs measurement, but no longer gates v1 |
| 240×240 ×3 → 720×720 in 1280×720 | mirror | 16 bpp | 2×115 KB in SRAM | Low — needs overclocked 720p timings |

## 4. Hardware architecture

```
      ┌─────────── hexpansion edge connector (1.0 mm, ENIG) ───────────┐
      │  +3V3 (600 mA)   SDA/SCL   HS_F..HS_I   LS_A..LS_E   DET→GND   │
      └───┬──────────────┬───────────┬────────────┬────────────────────┘
          │              │           │            │
     ┌────▼────┐   ┌─────▼─────┐ ┌───▼────┐  ┌────▼─────┐
     │ bulk +  │   │ I2C0      │ │ SPI0   │  │ RUN /    │
     │ ferrite │   │ target    │ │ slave  │  │ IRQ /    │
     └────┬────┘   │ @0x50     │ │ 20–23  │  │ BOOTSEL  │
          │        │ GPIO24/25 │ └───┬────┘  └────┬─────┘
          │        └─────┬─────┘     │            │
          │              └───────────┴────────────┘
          │                          │
  ┌───────▼───────┐          ┌───────▼────────┐        ┌──────────────┐
  │ TPS2116 mux   │          │   RP2350A      │◄──QSPI─┤ W25Q128 16MB │
  │ prio USB, rev-│          │   QFN-60       │  CS0   └──────────────┘
  │ blocking BOTH │          │  L1 3.3 µH     │        ┌──────────────┐
  └───────┬───────┘          │  Y1 12 MHz     │◄──QSPI─┤ APS6404 8 MB │
          │                  └─┬──┬──┬──┬──┬──┘  CS1   └──────────────┘
          │      HSTX 12–19    │  │  │  │  │ SWD
          │             ┌──────▼┐ │  │  │  └──────► J3  3-pin JST-SH debug
          │             │2× ESD │ │  │  │           SWCLK · GND · SWDIO
          │             │arrays │ │  │  └─SPI1 8–11─► microSD  (J6)
          │             └──────┬┘ │  └────I2C1 6/7──► J4,J5 Qwiic ×2
          │                    │  │                   4k7 pull-ups on jumper
          │                    │  └─PIO I2C 3/4────► PCA9306 ──► DDC / EDID
          └──────── +5V ───────┴─────────────────────────────► J1 mini-HDMI (C)

  J2 USB-C ─► TVS + 5k1 CC ─► buck 5V→3V3 ─► mux input A (priority)
              USB D± ─────────────────────────► RP2350
  3V3_LOCAL ─► TPS61023 boost ─► V5_HDMI  (only 5 V source; never muxed to VBUS)
  SW1 BOOT   SW2 RESET   badge-rail sense ─► GPIO
  LED1 power (always on)   LED2 status (GPIO1)   LED3 SK6805 RGB

  Flats:  outer = J1 + J2   ·   side A = J4 + J5   ·   side B = J6
```

### 4.1 RP2350A pin budget (30 GPIO, QFN-60)

Every peripheral group below was checked against the RP2350's pin-mux tables — SPI0/SPI1
and I2C0/I2C1 can only appear on fixed pin groups, and that, not pin count, is what
constrains this map.

| Pin | Function |
|------|----------|
| 0 | PSRAM chip select (QMI CS1 — GPIO0/8/19 are the only options on RP2350A, and 19 is taken by HSTX) |
| 1 | Status LED (LED2) |
| 2 | HDMI hot-plug detect, via 100k/100k divider from +5 V |
| 3, 4 | DDC/EDID via **PIO I2C** through the PCA9306 level shifter |
| 5 | microSD card-detect |
| 6, 7 | **I2C1 (hardware) → Qwiic sockets J4/J5** |
| 8–11 | **SPI1 → microSD** (8 MISO, 9 CS, 10 SCK, 11 MOSI) |
| **12–19** | **HSTX → 4 TMDS pairs (clock + 3 data)** — fixed by silicon |
| 20–23 | SPI0 slave ↔ HS_F(SCK) / HS_G(MOSI) / HS_H(MISO) / HS_I(CS) |
| 24, 25 | I2C0 target ↔ badge SDA / SCL (the emulated EEPROM) |
| 26 | → LS_B, attention/IRQ to the badge |
| 27 | ← LS_C, bootloader-entry request from the badge |
| 28, 29 | ← LS_D / LS_E, spare |
| RUN pin | ← LS_A (badge-controlled reset) **and** SW2 reset button |
| SWCLK / SWDIO | → J3, Raspberry Pi 3-pin debug connector |
| USB D± | → J2 USB-C |
| BOOTSEL | → SW1 |

Note the mapping trades JTAG on port 1 (GPIO39–42 are the ESP32's JTAG pins) — use
port 2 or 6 during badge-side debugging.

**Why the SD card forced a reshuffle.** SPI0 is already the badge slave on GPIO20–23, so
the card needs SPI1 — whose only groups are {8,9,10,11}, {12–15} and {24–27}. HSTX owns
12–19 and the badge I2C target owns 24/25, so **{8,9,10,11} is the only possibility**.
That displaced the Qwiic bus to I2C1 on GPIO6/7 and DDC to a PIO I2C on GPIO3/4. The one
casualty is the UART0 test points, which were already redundant with the USB CDC console.

**Why Qwiic gets the hardware I2C and DDC gets PIO:** user-attached Qwiic devices are the
unpredictable ones — arbitrary addresses, arbitrary clock stretching, arbitrary bus
speeds — so they get the real peripheral. EDID is a one-shot low-rate read at mode-set
time whose timing we fully control, and PIO I2C on RP2350 handles it comfortably. Keeping
them on separate buses also avoids the address collision that would otherwise be
guaranteed: monitor EDID EEPROMs live at 0x50, and so do plenty of Qwiic boards.

### 4.2 Debug and control surface

| Item | Detail |
|------|--------|
| J2 USB-C | Power input, UF2 mass-storage firmware update, USB CDC serial console |
| J3 SWD | 3-pin JST-SH 1.0 mm, **Raspberry Pi debug connector standard** (SWCLK · GND · SWDIO) — a Debug Probe plugs straight in |
| SW1 BOOT | Hold at power-up for UF2 bootloader |
| SW2 RESET | Pulls RUN low; shares the pin with the badge's LS_A reset line |
| LED1 | Power — always on when the rail is up. High-efficiency part on 4k7, ~0.4 mA |
| LED2 | Status/user, GPIO1 |
| LED3 | SK6805 RGB, optional |
| GPIO3 test pads | Spare, brought out for bring-up probing |

Three independent ways in — USB, SWD, and the badge's own RUN control — means there is no
plausible state this board gets into that you cannot recover from.

### 4.3 Qwiic bus

Two JST-SH 4-pin sockets wired in **parallel on one bus**, which is the standard Qwiic
chaining idiom: the hexpansion can sit mid-chain rather than only at the end. 4k7
pull-ups to 3V3 with a solder jumper so they can be removed when the card joins a chain
that already has them.

Budget roughly **100 mA** of the port's headroom for attached devices (§4.5 leaves about
that). A Qwiic device that pulls more than that will trip the badge's port switch, not
ours — worth a line in the user documentation.

### 4.4 Flat allocation — the real mechanical problem

At the template's 32 mm the outer flat is 18.5 mm and **mini-HDMI + USB-C do not fit on
it** (19.5 mm of connector body before margins). Growing the board to **44 mm across
flats** gives a 25.4 mm flat and makes the layout work:

| Flat | Contents | Used |
|------|----------|-----:|
| **Outer** (opposite the tab) | J1 mini-HDMI 10.5 + J2 USB-C 9.0, ~2 mm margins | 25.5 of 25.4 mm |
| **Side A** | J4 + J5 Qwiic, 6.2 each | 18.4 of 25.4 mm |
| **Side B** | microSD socket, ~15 mm | 19.0 of 25.4 mm |
| Two tab-side flats | Face the badge — unusable | — |

Fat-cable connectors have to be on the outer flat: with a 4.1 mm gap to the neighbouring
hexpansion, a USB-C or HDMI plug body simply will not fit beside a side flat. JST-SH
Qwiic cables (~4–5 mm wide, thin and flexible) are borderline in that gap and fine when
the adjacent bay is empty, which is the common case. Same for the SD slot: card insertion
needs finger room that a populated neighbouring bay does not leave.

Two things to be honest about:

* **The outer flat closes with no slack** — 25.5 mm of connector-plus-margin on a 25.4 mm
  flat. It works with ~1.9 mm end margins and 2 mm between connectors, and the corner
  radii give a little back, but this is a placement study, not an assumption.
* **microSD is the first thing to cut** if the layout fights back. With 16 MB of flash and
  a USB-C port for loading assets, it is a convenience, not a requirement.

#### If you want real slack: a radial wedge instead of a hexagon

The 60° sectors diverge, so available width grows with radius: at radius `r` from the badge
centre a shape can be about `r` wide (half-width `r·sin 30° = r/2` each side) before it
fouls its neighbours. Keeping the tab-end profile of the template hexagon and flaring out
to `r ≈ 70 mm` gives roughly a **60 mm outer edge** — everything on one face with room
to spare, for about the same radial extent as the 44 mm hexagon.

The costs are that it stops looking like a Tildagon hexagon, and more mass hangs off a
1.0 mm card edge and two M2 screws. **Recommendation: build the 44 mm hexagon; hold the
wedge in reserve for if the outer-flat placement study fails.**

### 4.5 Power domains and badge isolation

> **Design rule: the hexpansion never powers the badge.** Not in normal operation, not
> during insertion or removal, not under a single-component failure. USB-C powers this
> board only. The badge's BQ25895 stays the sole authority over the badge's own rails.

Everything below exists to enforce that rule.

| Rail | Source | Notes |
|------|--------|-------|
| `VBUS` | USB-C J2 | 5 V. TVS + USBLC6 ESD, 5k1 CC1/CC2 pulldowns |
| `3V3_USB` | Buck from `VBUS` | The local rail when USB is present |
| `3V3_BADGE` | Edge pads 15/16 | 600 mA through the badge's MT9700 load switch |
| `3V3_LOCAL` | TPS2116 mux, priority to `3V3_USB` | Powers everything on the board |
| `V5_HDMI` | Boost from `3V3_LOCAL` | The only 5 V source on the board — never muxed with `VBUS` |

#### The four backfeed paths

| # | Path | If unhandled | Mitigation |
|--:|------|--------------|------------|
| 1 | 5 V `VBUS` reaches the badge's 3V3 rail | Destroys the ESP32-S3, both AW9523s, the TCA9548A, the display and the IMU | `VBUS` never leaves the USB domain: it feeds only the buck. The TPS2116 blocks reverse current on its badge input, so even a shorted buck cannot push 5 V outward |
| 2 | USB-derived 3V3 backfeeds pads 15/16 | Flows through the badge's load-switch body diode into `3V3_SYS`. The badge loses the ability to power the port off, and its whole rail is back-powered from your USB supply | TPS2116 priority mux with true reverse-current blocking and break-before-make. When USB is present the badge input is hard-isolated |
| 3 | Our GPIOs drive badge pins while the badge is unpowered | Phantom-powers the badge through ESP32/AW9523 ESD diodes — latch-up, pin damage | Only two pins can source: MISO (HS_H) and the LS_B interrupt. Firmware holds both tri-stated until the badge-rail sense reads live. 33 Ω series on the HS lines caps fault current |
| 4 | The boost drives `VBUS` backwards | 5 V on the USB-C connector: spec violation, hazard to whatever is plugged in | Deleted by construction — the HDMI 5 V rail is never muxed with `VBUS` |

#### Why a mux and not a pair of FETs

The earlier revision said "supply OR-ing (Q1/Q2)" without a topology, and two P-FETs
described that loosely do not do the job: **a single P-FET always has a body diode**, and
whichever way you orient it one direction conducts. Blocking both ways needs back-to-back
FETs plus their gate drive — four parts and a switchover behaviour you have to get right
yourself.

A **TPS2116** does it in one part: two inputs, automatic priority, genuine reverse-current
blocking on both, break-before-make. `3V3_USB` is input A with priority; `3V3_BADGE` is
input B. Q1/Q2 come out of the BOM.

Path 2 deserves emphasis. A high-side P-FET load switch conducts OUT→IN through its body
diode when the output is pulled above the input, unless the part explicitly blocks reverse
current — and the MT9700/AAT4610 class does not advertise that. So without the mux, running
on USB back-feeds the badge's `3V3_SYS` **even with the port switched off in software**.
The datasheet in `badge-2024-hardware/datasheets/` is a scanned image and cannot be
searched, so this is assumed until confirmed — which is the correct posture regardless.

#### Path 3 is mostly free

Of our badge-facing signals, SDA and SCL are open-drain — they only ever pull low, so they
cannot inject current into an unpowered bus. SCK, MOSI and CS are inputs to us. That leaves
**exactly two sourcing outputs**: MISO on HS_H, and the attention line on LS_B. So:

* Divide `3V3_BADGE` through 100k/100k to a GPIO as a **badge-presence sense**.
* Firmware tri-states MISO and LS_B until that sense reads live, and re-tri-states them if
  it drops. This doubles as the RP2350 knowing whether it is in a badge or standalone.
* **33 Ω series on each HS line.** It caps fault current and doubles as source-series
  termination, which 40 MHz edges want anyway.

#### USB-C sink requirements

* **5k1 CC1/CC2 pulldowns.** Without them a Type-C source delivers nothing at all.
* Nothing on the board may present as a source.
* **TVS on `VBUS`** so a misbehaving PD supply cannot exceed the buck's input rating.

### 4.6 Points that need care

* **Mini-HDMI Type C does not share Type A's pin assignment.** The 19 pins are reordered.
  Wire the schematic from the chosen connector's datasheet — do not adapt a Type A
  footprint's netlist, this is a classic and expensive mistake.
* **RP2350-E9 erratum.** Internal pull-downs latch on floating inputs. Use external
  pulls on anything that can float — the LS lines, CS, and both I2C pairs — and do not
  rely on internal pull-downs anywhere.
* **TMDS drive.** The Pico DVI Sock / Adafruit HSTX-DVI approach — GPIOs driven directly
  into the sink's 50 Ω terminations with drive strength set to 12 mA, no buffers — is
  proven and is what we use. Series-resistor footprints stay on the board as 0 Ω, as
  insurance.
* **HDMI +5 V** must be supplied to the sink, 55 mA minimum, or many monitors
  will not present EDID. Boost it, and gate the boost's EN pin from a GPIO so it comes
  up *after* firmware, not during inrush.
* **ESD on TMDS** needs sub-pF capacitance. `TPD4E05U06` (0.4 pF) is the right family;
  a generic USBLC6 at 3 pF is not.
* **Inrush.** The MT9700 will trip at 600 mA. Keep total bulk capacitance modest
  (≈ 40 µF) and stagger the boost enable, or the hexpansion will fail to power on.

### 4.7 Power budget

| Rail / block | Est. current @3V3 |
|--------------|------------------:|
| RP2350 @150 MHz, both cores + HSTX + DMA | 60 mA |
| TMDS drive, 4 pairs | 50 mA |
| PSRAM active | 30 mA |
| Flash (bursty) | 15 mA |
| HDMI +5 V @55 mA through boost (85% eff.) | 98 mA |
| microSD during transfers (bursty, idle ≈ 0.2 mA) | 50 mA |
| Level shifter, LEDs, misc | 11 mA |
| **Total typical, SD idle, no Qwiic devices** | **~264 mA** |
| **Peak, SD streaming** | **~400 mA** |
| Headroom left for attached Qwiic devices | ~100 mA (shared with SD bursts) |

Comfortably inside 600 mA — but ~0.9 W is a real load on a badge battery. Expect
**2–3 hours** of badge runtime with the GFX card live on badge power alone.

On USB the same 264 mA at 3.3 V is ~0.87 W, so `VBUS` draws about **195 mA at 5 V** through
a 90 %-efficient buck — well inside a USB 2.0 port's 500 mA, no PD negotiation needed. With
USB-C plugged in the hexpansion powers itself and **draws nothing at all from the badge**;
the TPS2116 hard-isolates the badge input rather than merely sharing the load (§4.5).

---

## 5. Faking the EEPROM

The RP2350 brings up I2C0 as a target at `0x50` with 16-bit addressing and answers as a
64 KiB EEPROM:

* `fs_offset` = 64, `eeprom_page_size` = 64, `eeprom_total_size` = 65536
* Header as §1.4, manifest `2026`, assigned VID/PID, `unique_id` per board from the
  RP2350's unique chip ID
* Reads served from a flash-backed image with a RAM shadow; writes accepted with
  page-write semantics and ACK polling, then committed to flash

**The one genuine risk is a start-up race.** The badge enables port power and scans I2C
shortly after. If the RP2350 has not brought up its I2C target yet, the hexpansion
enumerates as "no EEPROM" and the app never runs. Three mitigations, all cheap:

1. Bring the I2C target up in the first few hundred microseconds of `main()`, before
   PSRAM init, HSTX, or anything else. Target **< 20 ms** from power-good.
2. **Clock-stretch** while flash is being read. The badge is a normal I2C master and will
   wait.
3. **Keep a real 24C64 footprint on the board (DNP) with a solder jumper.** If emulation
   turns out to be flaky in the field, populate a Zetta ZD24C64A, move the jumper, and
   the hexpansion enumerates from silicon while the RP2350 moves to a secondary address.
   Twelve cents of insurance against a bug that would otherwise brick the product.

A nice second-order benefit: because writes are accepted, the badge can push a firmware
image *into* the fake EEPROM and the RP2350 can flash itself from it. Field updates with
no cable.

---

## 6. Bill of materials

Per board, 92 fitted placements (U8 is DNP and excluded from the totals). Prices are
LCSC/JLCPCB indicative in USD *(est.)* — **verify at order time**,
they move constantly and the RP2350A in particular has swung between $1.03 and $1.71.

| # | Ref | Part | Package | Qty | @10 | @50 |
|--:|-----|------|---------|----:|----:|----:|
| 1 | U1 | RP2350A (C42411118) | QFN-60 7×7 | 1 | 1.45 | 1.20 |
| 2 | U2 | W25Q128JVSIQ 16 MB QSPI flash | SOIC-8 | 1 | 1.10 | 0.90 |
| 3 | U3 | APS6404L-3SQR-ZR 8 MB QSPI PSRAM (C3040877) | SOP-8 | 1 | 1.15 | 1.00 |
| 4 | U4 | TPS61023 boost, 3V3→5V | SOT-563 | 1 | 0.55 | 0.45 |
| 5 | U5 | PCA9306 DDC level shifter | VSSOP-8 | 1 | 0.32 | 0.26 |
| 6 | U6,U7 | TPD4E05U06 ESD array, 0.4 pF | USON-10 | 2 | 0.76 | 0.60 |
| 7 | U8 | ZD24C64A fallback EEPROM — **DNP** | SOP-8 | 1 | 0.11 | 0.09 |
| 8 | J1 | **Mini-HDMI Type C receptacle**, right-angle | SMD | 1 | 0.65 | 0.52 |
| 9 | J2 | USB-C 16P receptacle (power + UF2 + CDC) | SMD | 1 | 0.30 | 0.24 |
| 10 | J3 | **JST-SH 3-pin, RPi debug standard** (SM03B-SRSS-TB) | SMD | 1 | 0.12 | 0.10 |
| 14 | J4,J5 | **JST-SH 4-pin Qwiic** (SM04B-SRSS-TB) | SMD | 2 | 0.26 | 0.22 |
| 11b | J6 | **microSD socket, push-push** | SMD | 1 | 0.35 | 0.30 |
| 12 | D1 | USBLC6-2SC6 USB ESD | SOT-23-6 | 1 | 0.10 | 0.08 |
| 13 | U9 | **TPS2116 power mux** — priority + reverse blocking | SOT-23-6 | 1 | 0.55 | 0.45 |
| 13b | U10 | **Buck 5V→3V3**, ≥500 mA (AP3417 / TPS62203) | SOT-23-5 | 1 | 0.35 | 0.28 |
| 13c | L3 | 2.2 µH shielded (USB buck) | 0805 | 1 | 0.09 | 0.07 |
| 13d | D2 | TVS on VBUS (SMF5.0A) | SOD-123 | 1 | 0.05 | 0.04 |
| 14 | Y1 | 12 MHz crystal, ±30 ppm | 3225 | 1 | 0.16 | 0.13 |
| 15 | L1 | 3.3 µH shielded (RP2350 VREG_LX) | 0805 | 1 | 0.10 | 0.08 |
| 16 | L2 | 2.2 µH shielded (boost) | 0805 | 1 | 0.09 | 0.07 |
| 17 | FB1,FB2 | Ferrite bead 600 Ω @100 MHz | 0402 | 2 | 0.04 | 0.03 |
| 18 | C | 22 µF bulk / 10 µF / 1 µF / 100 nF / 15 pF | 0805–0402 | 34 | 0.34 | 0.26 |
| 19 | R | Pulls, dividers, Qwiic 4k7, SD pulls, **5k1 CC**, **badge sense**, **33 Ω HS series**, LED | 0402 | 30 | 0.10 | 0.06 |
| 20 | SW1,SW2 | **Tactile switches — BOOT and RESET** | 3×2 mm | 2 | 0.12 | 0.10 |
| 21 | LED1,LED2 | **Power (always on) + status** | 0603 | 2 | 0.04 | 0.04 |
| 22 | LED3 | SK6805 RGB status | 2427 | 1 | 0.08 | 0.06 |
| | | **Total parts / board** | | **92 fitted** | **$9.20** | **$7.53** |

Machine-readable version: [`bom.csv`](bom.csv).

Component-level notes:

* GD25Q128 substitutes for the W25Q128 at ~$0.85 if flash pricing bites.
* The `-ZR` PSRAM part is the one to buy — the `-SN` variant is 2.5× the price for the
  same die.
* U8 stays unpopulated in normal builds; it exists so the EEPROM-emulation risk in §5
  has a hardware answer.
* J3/J4/J5 are all JST SH 1.0 mm — one connector family, two part numbers, which keeps
  the feeder count down.

---

## 7. Costing: 10 / 20 / 50 boards

Turnkey PCBA at JLCPCB, delivered to the UK. Fee structure from JLCPCB's published
rates (setup ≈ $8/side, feeder ≈ $1.50 per extended part type, stencil ≈ $1.50,
SMT labour ≈ $0.0017/joint with a $0.48/board floor). Board is a **44 mm across-flats
hexagon** (44 × 50.8 mm bounding box, 16.8 cm²), 4-layer, 1.0 mm, ENIG,
impedance-controlled, ~360 solder joints, ~19 extended part types,
**double-sided assembly**.

| Line | 10 boards | 20 boards | 50 boards |
|------|----------:|----------:|----------:|
| PCB fab (4L, 1.0 mm, ENIG, imp. ctrl) | $42 | $64 | $135 |
| Stencils (2 sides) | $3 | $3 | $3 |
| SMT setup (2 sides) | $16 | $16 | $16 |
| Feeder fees (~19 extended parts) | $29 | $29 | $29 |
| SMT labour | $6 | $12 | $31 |
| X-ray (QFN-60) contingency | $10 | $10 | $10 |
| Components (incl. MOQ overshoot) | $120 | $201 | $395 |
| **Subtotal, ex-works** | **$226** | **$335** | **$619** |
| Air freight to UK | $24 | $28 | $38 |
| UK import VAT @20% | $50 | $73 | $131 |
| Courier disbursement fee | $15 | $15 | $15 |
| **Delivered total** | **$315** | **$451** | **$803** |
| **Per board** | **$31.50** | **$22.55** | **$16.06** |
| *Per board, £ @1.27* | *£24.80* | *£17.76* | *£12.65* |

### How the price has moved

| Revision | What changed | 50-board, per board |
|----------|--------------|--------------------:|
| Mini-HDMI, USB-C, SWD, buttons, Qwiic — 32 mm hexagon | — | $13.28 |
| Grown to 44 mm across flats, microSD restored | +89% board area, SD socket | $15.20 (+$1.92) |
| **Power-domain isolation** | **TPS2116 mux, USB buck, TVS, CC and sense resistors** | **$16.06 (+$0.86)** |

Two of those are worth reading as value, not cost. **$1.92 nearly doubled the board area**
and is what makes the connectors fit at all — PCB fabrication is only 14–17% of the run, so
area is among the cheapest things you can buy here, and far cheaper than a respin. And
**$0.86 is what stands between a USB cable and a dead badge**; the alternative is not a
cheaper board, it is an unprotected one.

The fixed-cost shape is unchanged: **$58 of the cost is fixed** (setup, feeders, stencils,
X-ray) regardless of quantity. At 10 boards you pay $5.80/board just to turn the machine
on; at 50 it is $1.16. If there is any chance of wanting 50, order 50.

### One-off NRE, not included above

| Item | Est. |
|------|-----:|
| 5-board prototype spin (same process) | $220 |
| Second spin, assume one is needed | $220 |
| Raspberry Pi Debug Probe, mini-HDMI cable, test monitor, M2 hardware | $55 |
| **Total NRE** | **~$495** |

Fully loaded, a 50-board run lands at roughly **$26/board**; a 20-board run at
**$47/board**.

### Should you hand-assemble instead?

| | 10 boards | 50 boards |
|---|---:|---:|
| Turnkey PCBA, delivered | $315 | $803 |
| PCB + parts + own reflow, delivered | ~$250 | ~$709 |
| Saving | $65 | $94 |
| Your time | ~10 h | ~36 h |

**No.** You would be buying your own labour at under $3/hour to hand-place a 0.4 mm-pitch
QFN-60, an HDMI shell, a microSD cage and three JST-SH connectors, with yield risk on top.
Use turnkey.

### Other vendors

* **PCBWay** — comparable, usually 10–20% more, better on odd outlines and small runs.
* **Aisler** (EU) — no import VAT surprise, much friendlier for a UK/EU maker, but
  roughly 2× on assembly and a thinner parts library.
* **Eurocircuits** — excellent quality, ~3× the cost. Not for this.

Order **10% spare bare PCBs** with every run; they cost almost nothing and save a whole
lead time when you kill a board on the bench.

---

## 8. Delivery plan

### Phase 0 — De-risk on hardware you already have (2–3 weekends, ~$40)

Before drawing a single schematic symbol. Wire a Pico 2 + DVI Sock to a badge port using
a protoboard hexpansion (`codemyriad/protogon` or DanNixon's devkit) and settle everything
that could kill the design:

1. **EEPROM emulation enumerates reliably.** Cold-plug the hexpansion 50 times. Measure
   the worst-case time from port power-on to the badge's first I2C read, and the RP2350's
   time-to-I2C-ready. If these do not have clear daylight between them, §5's fallback
   becomes the primary plan and the schematic changes.
2. **SPI link speed.** Sweep 10→40 MHz over the HS pins with the real edge connector in
   circuit. Record the highest reliable rate and the MicroPython-side throughput.
   Then push a real 115,200-byte frame and time it end to end.
3. **What the badge actually renders.** Log `display.get_fps()` across a handful of real
   apps. If it comes in around 15 fps, the link has enormous headroom and the mirror
   design gets simpler; if it is near 40, dirty-rect support moves into v1. This is the
   measurement that decides how much work §3.1 needs.
4. **Prototype `display.get_fb()`** against a local firmware build, so the upstream ask is
   a tested patch rather than a feature request.
5. **HSTX DVI at 240×240 pixel-doubled**, running from the badge's 3V3 with a current
   meter inline, to validate §4.7.
6. **Backfeed check.** With the badge powered off and USB-C live, measure the current into
   pads 15/16 and the voltage on the badge's `3V3_SYS`. Both must read zero. Repeat with
   the badge on and the port disabled in software. This is the test that proves §4.5, and
   it is worth building a jig for.

Everything downstream is cheap to change now and expensive to change later. Do not skip
this phase.

### Phase 1 — Schematic + layout (3–4 weeks part-time)

Start from `emfcamp/badge-2024-hardware/hexpansion` so the outline, tab, and mounting
holes are right by construction. 4-layer stackup: L1 signal/TMDS, L2 solid GND, L3
power, L4 signal. Length-match TMDS pairs, keep them on L1 over the L2 ground plane, and
keep the runs under ~30 mm (which the board size makes easy).

**Do the flat-allocation placement study first** (§4.4) — the outer flat carries
mini-HDMI plus USB-C at 25.5 mm on a 25.4 mm flat, so connector placement, not routing,
is what decides whether this board closes. If it does not, either drop microSD and move
USB-C to a side flat, or switch to the radial wedge outline. Request VID/PID **and open the `display.get_fb()` conversation upstream** in week 1 —
both are short work on someone else's schedule.

### Phase 2 — Prototype run (5 boards, ~2 weeks lead time)

### Phase 3 — Firmware (4–6 weeks part-time)

Pico SDK in C. Order of work: badge-presence sense and tri-state-by-default on MISO/LS_B
(this comes first — it is a safety interlock, not a feature) → boot + flash/PSRAM bring-up → I2C target and EEPROM
emulation → HSTX DVI output → SPI slave with DMA (PIO-based, so clock-stretch and
back-pressure are under our control) → the mirror path (receive frame, pixel-double,
mask, present) → the drawing engine → PIO-I2C DDC/EDID and mode negotiation → Qwiic bus and the I2C bridge commands → microSD → HDMI audio if time allows.

### Phase 4 — Badge-side driver and the upstream patch (1–2 weeks)

Two deliverables. **A PR to `badge-2024-software`** exposing `tildagon_fb` as
`display.get_fb()` — prototyped back in Phase 0, so this is submitting tested work rather
than asking for a feature. And the **mirror app**: read the buffer after each
`end_frame`, push it over SPI, done. Baked into the LittleFS image the fake EEPROM serves,
plus a companion app in this repo's index so people can install it without owning the
hardware yet.

The display-list driver is a later, separate piece of work — nothing depends on it for the
primary mode.

### Phase 5 — Production run (10/20/50 per §7)

---

## 9. Risk register

| # | Risk | Severity | Mitigation |
|--:|------|----------|------------|
| 1 | EEPROM emulation loses the enumeration race at power-on | **High** | Fast-boot I2C target; clock stretching; DNP 24C64 fallback footprint. Proven or disproven in Phase 0. |
| 2 | Outer flat does not close — 25.5 mm of connectors on a 25.4 mm flat | **High** | Placement study is the first task in Phase 1. Fallbacks in order: drop microSD and move USB-C to a side flat; then the radial wedge outline (§4.4). |
| 3 | USB backfeed reaches the badge's 3V3 rail | **High** | TPS2116 priority mux with reverse blocking on both inputs; `VBUS` confined to the USB domain; HDMI 5 V never muxed with `VBUS`. Bench-proven in Phase 0. |
| 4 | **`display.get_fb()` never lands upstream**, so nothing can read the badge framebuffer | **High** | Ask in week 1 with a tested patch from Phase 0; it is ~10 lines of C at a single choke point. If it is refused, the product falls back to display-list mode only — still a working graphics card, but it loses the "every app for free" property that makes mirroring the headline. Do not discover this late. |
| 5 | Phantom-powering the badge through signal pins | Medium | Only MISO and LS_B can source; both firmware tri-stated until the badge-presence sense reads live; 33 Ω series on the HS lines. |
| 6 | Buck fails short, putting 5 V on `3V3_LOCAL` | Medium | The mux's reverse blocking keeps it off the badge. On-board damage is accepted; add a 3.6 V clamp on `3V3_LOCAL` if the layout leaves room. |
| 7 | 600 mA budget or inrush trips the port switch | Medium | Modest bulk capacitance, staged boost enable, measured in Phase 0. Qwiic devices documented as sharing ~100 mA of headroom. |
| 8 | Mini-HDMI wired as if it were Type A | Medium | Different pin assignment; netlist from the connector datasheet, and check it at review. |
| 9 | Neighbouring hexpansion sits 4.1 mm away, blocking side-flat cables | Medium | Fat-cable connectors are all on the outer flat by design; Qwiic and SD documented as needing the adjacent bay free. |
| 10 | Cable strain on the edge connector — worse on an 89% larger board | Medium | Both M2 mounting holes populated; strain-relief loop documented for users. Mini-HDMI reduces leverage but has lower retention force than Type A. |
| 11 | Badge battery life halves when the card runs on badge power | Medium | USB-C input with the TPS2116 mux — on USB the badge supplies nothing at all. |
| 12 | Badge renders too slowly for mirroring to look good | Low | Inherent: you see what the badge draws, and no link speed changes that. Measured in Phase 0; document the expectation rather than engineering against it. |
| 13 | SPI link slower than 40 MHz in practice | Low | Mirroring needs 115 KB/frame and is bounded by the badge anyway; the display-list path already assumes 2.5 MB/s. |
| 14 | TMDS signal integrity on a 1.0 mm 4-layer board | Low | Short runs, controlled impedance, proven direct-drive topology. |
| 15 | VID/PID not assigned in time | Low | Ask in week 1; costs nothing. |
| 16 | RP2350-E9 pull-down erratum bites on LS/CS/I2C lines | Low | External pull resistors everywhere it matters. |

---

## 10. Decisions taken

| Decision | Outcome |
|----------|---------|
| **Primary mode** | **Mirroring the badge's 240×240 screen**, pixel-doubled to 480×480 in 640×480. Display-list rendering becomes the advanced, opt-in path. |
| **Circular mask** | **On by default** — the monitor shows what the wearer sees. Runtime toggle reveals the full square for debugging. |
| Video connector | **Mini-HDMI Type C.** Cables are common (Pi Zero, most cameras), and on an 18.5–25.4 mm flat the 5 mm it saves over Type A is decisive. |
| USB-C | **In** — power, UF2, and a CDC console. |
| Debug | **3-pin JST-SH SWD** to the Raspberry Pi debug connector standard, so a Debug Probe plugs straight in. |
| Buttons | **BOOT and RESET**, both on the top face. |
| LEDs | **Power (always on) + status on GPIO1**, plus an optional SK6805 RGB. |
| Qwiic | **Two JST-SH 4-pin sockets in parallel** on hardware I2C1 (GPIO6/7), with jumper-removable 4k7 pull-ups. Both on side flat A. |
| Badge power | **The hexpansion never powers the badge** — not in normal use, not during hot-plug, not under a single-component failure. Enforced by the TPS2116 mux and firmware tri-stating, not by convention. |
| Board size | **44 mm across flats** (was the 32 mm template). Smallest size where mini-HDMI and USB-C fit on the outer flat. Costs ~$2/board. |
| microSD | **Back in**, on SPI1 GPIO8–11, on the second side flat. First thing to cut if the placement study fails. |

### Still open

* **Run size.** The fixed-cost curve in §7 says 50 if there is any chance of 50.
* **Whether the outer flat closes at 44 mm**, or the board wants the wedge outline. Decided by the placement study, not now.
* **Whether microSD earns its place.** 16 MB of flash plus USB-C asset loading may make it redundant.
