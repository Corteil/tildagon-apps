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
| 4 | SDA | 14 | GND |
| 5 | SCL | 15 | +3V3 |
| 6 | HEXP_DET | 16 | +3V3 |
| 7 | LS_C | 17 | GND |
| 8 | LS_D | 18 | HS_H |
| 9 | LS_E | 19 | HS_I |
| 10 | GND | 20 | GND |

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

### 1.3 Mechanical

* Template outline: **32.0 × 36.6 mm** overall, including the 6.25 mm × 12 mm insertion tab.
* **Board thickness 1.0 mm** (template stackup is 0.91 mm core + 2×35 µm Cu ≈ 0.98 mm).
  This is not negotiable — the badge's SFP-style receptacle expects a 1.0 mm card edge.
* **ENIG only. HASL is explicitly forbidden** — the pads must be flat to make contact.
* Two M2 mounting-hole pads, which is what will take the strain of a hanging HDMI cable.

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

A hexpansion that:

1. Outputs **DVI/HDMI** from the RP2350's HSTX peripheral.
2. Carries **16 MB QSPI flash** (boot + sprite/font/asset storage) and **8 MB QSPI PSRAM**
   (framebuffers, off-screen surfaces).
3. **Emulates the identification EEPROM** at `0x50` in RP2350 firmware, serving a 64 KiB
   LittleFS image that contains the badge-side MicroPython driver app. The RP2350 owns
   that image, so the driver updates itself with the hexpansion firmware — no separate
   EEPROM flashing step, ever.
4. Accepts drawing commands from the badge over a **SPI link on the four HS pins**.
5. Optionally takes **USB-C power** so it does not flatten the badge battery, and so the
   RP2350 can be re-flashed by UF2 without a programmer.

The badge stays the computer. This is a graphics *card*, not a co-processor that does
the badge's job for it.

---

## 3. The bandwidth reality, and what it forces

40 MHz SPI is ~5 MB/s at absolute best. A 640×480 16 bpp frame is 614 KB. Pushing whole
frames from the badge would give **8 fps**, and that is before MicroPython overhead.

So the architecture is settled by arithmetic: **the badge sends a display list, not
pixels.** The RP2350 holds the assets and does the rendering.

Protocol sketch (4-byte header + payload, full duplex so the RP2350 can return status
on MISO in the same transaction):

| Class | Commands |
|-------|----------|
| Mode | `set_mode`, `get_edid`, `blank`, `set_palette` |
| Assets | `upload_asset` (to flash or PSRAM), `free_asset`, `list_assets` |
| Draw | `clear`, `rect`, `line`, `blit`, `blit_scaled`, `text`, `tilemap`, `sprite_batch` |
| Frame | `present` (flip), `vsync_wait`, `set_layer` |
| Audio | `queue_pcm` (HDMI audio islands) |
| System | `ping`, `status`, `reset`, `enter_bootloader`, `fw_update` |

Asset upload is the only bulk transfer and it happens once, not per frame. After that a
full-screen 60 fps scene costs the badge a few hundred bytes per frame. That is the
design working *with* the 5 MB/s ceiling instead of against it.

### Achievable video modes

| Mode | Colour | Buffering | Confidence |
|------|--------|-----------|-----------|
| 320×240 pixel-doubled to 640×480 @60 | 16 bpp | double-buffered in SRAM (2×150 KB) | **High — this is the target for v1** |
| 640×480 @60 | 8 bpp palettised | single buffer in SRAM (307 KB) + PSRAM back buffer | High |
| 640×480 @60 | 16 bpp | PSRAM-backed, line-buffer DMA (~37 MB/s sustained) | Medium — needs measurement |
| 800×600 / 720p30 | reduced depth | PSRAM-backed, system clock overclocked | Low — stretch goal |

Ship v1 on the first row. Everything else is a firmware update.

---

## 4. Hardware architecture

```
        ┌─────────────── hexpansion edge connector (1.0 mm, ENIG) ───────────────┐
        │  +3V3 (600 mA)   SDA/SCL   HS_F..HS_I   LS_A..LS_E   HEXP_DET→GND      │
        └───┬──────────────┬───────────┬────────────┬──────────────────────────┬─┘
            │              │           │            │                          │
       ┌────▼────┐   ┌─────▼─────┐ ┌───▼────┐  ┌────▼─────┐                    │
       │ bulk +  │   │ I2C0      │ │ SPI0   │  │ RUN /    │                    │
       │ ferrite │   │ target    │ │ slave  │  │ IRQ /    │                    │
       └────┬────┘   │ @0x50     │ │ 20–23  │  │ BOOTSEL  │                    │
            │        │ 24 / 25   │ └───┬────┘  └────┬─────┘                   GND
            │        └─────┬─────┘     │            │
            │              └───────────┴────────────┘
            │                          │
    ┌───────▼───────┐          ┌───────▼────────┐        ┌──────────────┐
    │ TPS61023      │          │   RP2350A      │◄──QSPI─┤ W25Q128 16MB │
    │ 3V3→5V, 55 mA │          │   QFN-60       │  CS0   └──────────────┘
    └───────┬───────┘          │  L1 3.3 µH     │        ┌──────────────┐
            │                  │  Y1 12 MHz     │◄──QSPI─┤ APS6404 8 MB │
            │                  └───┬────────┬───┘  CS1   └──────────────┘
            │              HSTX 12–19│    I2C1│10/11 + HPD
            │                  ┌─────▼──┐  ┌──▼────────┐
            │                  │ 2× ESD │  │ PCA9306   │  DDC level shift
            │                  │ arrays │  │ + divider │
            │                  └─────┬──┘  └──┬────────┘
            └────────── +5V ─────────┴────────┴──────► HDMI Type A receptacle
```

### 4.1 RP2350A pin budget (30 GPIO, QFN-60)

| GPIO | Function |
|------|----------|
| 0 | PSRAM chip select (QMI CS1 — GPIO0/8/19 are the only options on RP2350A, and 19 is taken by HSTX) |
| 1 | Status LED |
| 2 | HDMI hot-plug detect, via 100k/100k divider from +5 V |
| 3–7 | Spare / optional microSD on SPI |
| 8, 9 | UART0 debug pads |
| 10, 11 | I2C1 → DDC/EDID through PCA9306 level shifter |
| **12–19** | **HSTX → 4 TMDS pairs (clock + 3 data)** — fixed by silicon |
| 20–23 | SPI0 slave ↔ HS_F(SCK) / HS_G(MOSI) / HS_H(MISO) / HS_I(CS) |
| 24, 25 | I2C0 target ↔ badge SDA / SCL (the emulated EEPROM) |
| 26 | → LS_B, attention/IRQ to the badge |
| 27 | ← LS_C, bootloader-entry request from the badge |
| 28, 29 | ← LS_D / LS_E, spare |
| RUN pin | ← LS_A, badge-controlled reset (this is worth having: the badge can always recover a wedged GFX card) |

Note the mapping trades JTAG on port 1 (GPIO39–42 are the ESP32's JTAG pins) — use
port 2 or 6 during badge-side debugging.

### 4.2 Points that need care

* **RP2350-E9 erratum.** Internal pull-downs latch on floating inputs. Use external
  pulls on anything that can float — the LS lines, CS, and the DDC pair — and do not
  rely on internal pull-downs anywhere.
* **TMDS drive.** The Pico DVI Sock / Adafruit HSTX-DVI approach — GPIOs driven directly
  into the sink's 50 Ω terminations with drive strength set to 12 mA, no buffers — is
  proven and is what we use. Series-resistor footprints stay on the board as 0 Ω, as
  insurance.
* **HDMI +5 V (pin 18)** must be supplied to the sink, 55 mA minimum, or many monitors
  will not present EDID. Boost it, and gate the boost's EN pin from a GPIO so it comes
  up *after* firmware, not during inrush.
* **ESD on TMDS** needs sub-pF capacitance. `TPD4E05U06` (0.4 pF) is the right family;
  a generic USBLC6 at 3 pF is not.
* **Inrush.** The MT9700 will trip at 600 mA. Keep total bulk capacitance modest
  (≈ 40 µF) and stagger the boost enable, or the hexpansion will fail to power on.

### 4.3 Power budget

| Rail / block | Est. current @3V3 |
|--------------|------------------:|
| RP2350 @150 MHz, both cores + HSTX + DMA | 60 mA |
| TMDS drive, 4 pairs | 50 mA |
| PSRAM active | 30 mA |
| Flash (bursty) | 15 mA |
| HDMI +5 V @55 mA through boost (85% eff.) | 98 mA |
| Level shifter, LED, misc | 10 mA |
| **Total typical** | **~265 mA** |
| **Peak** | **~350 mA** |

Comfortably inside 600 mA — but ~0.9 W is a real load on a badge battery. Expect
**2–3 hours** of badge runtime with the GFX card live. That is the argument for the
USB-C input: with it, the hexpansion powers itself and the badge is untouched.

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

Per board. Prices are LCSC/JLCPCB indicative in USD *(est.)* — **verify at order time**,
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
| 8 | J1 | HDMI Type A receptacle, SMD right-angle | — | 1 | 0.60 | 0.48 |
| 9 | J2 | USB-C 16P receptacle (power + UF2) | — | 1 | 0.30 | 0.24 |
| 10 | D1 | USBLC6-2SC6 USB ESD | SOT-23-6 | 1 | 0.10 | 0.08 |
| 11 | Q1,Q2 | AO3401 P-FET, supply OR-ing | SOT-23 | 2 | 0.12 | 0.10 |
| 12 | Y1 | 12 MHz crystal, ±30 ppm | 3225 | 1 | 0.16 | 0.13 |
| 13 | L1 | 3.3 µH shielded (RP2350 VREG_LX) | 0805 | 1 | 0.10 | 0.08 |
| 14 | L2 | 2.2 µH shielded (boost) | 0805 | 1 | 0.09 | 0.07 |
| 15 | FB1,FB2 | Ferrite bead 600 Ω @100 MHz | 0402 | 2 | 0.04 | 0.03 |
| 16 | C | 22 µF 6.3 V X5R bulk | 0805 | 2 | 0.10 | 0.08 |
| 17 | C | 10 µF | 0603 | 4 | 0.08 | 0.06 |
| 18 | C | 1 µF | 0402 | 4 | 0.04 | 0.03 |
| 19 | C | 100 nF decoupling | 0402 | 18 | 0.07 | 0.05 |
| 20 | C | 15 pF crystal load | 0402 | 2 | 0.01 | 0.01 |
| 21 | R | assorted pulls / dividers / LED | 0402 | 16 | 0.05 | 0.03 |
| 22 | SW1 | Tactile switch, BOOTSEL | 3×2 mm | 1 | 0.06 | 0.05 |
| 23 | LED1 | Green indicator | 0603 | 1 | 0.02 | 0.02 |
| 24 | LED2 | SK6805 RGB status | 2427 | 1 | 0.08 | 0.06 |
| | | **Total parts / board** | | **66 placements** | **$7.45** | **$6.20** |

Machine-readable version: [`bom.csv`](bom.csv).

Component-level notes:

* GD25Q128 substitutes for the W25Q128 at ~$0.85 if flash pricing bites.
* The `-ZR` PSRAM part is the one to buy — the `-SN` variant is 2.5× the price for the
  same die.
* U8 stays unpopulated in normal builds; it exists so the EEPROM-emulation risk in §5
  has a hardware answer.

---

## 7. Costing: 10 / 20 / 50 boards

Turnkey PCBA at JLCPCB, delivered to the UK. Fee structure from JLCPCB's published
rates (setup ≈ $8/side, feeder ≈ $1.50 per extended part type, stencil ≈ $1.50,
SMT labour ≈ $0.0017/joint with a $0.48/board floor). Board is 32 × 36.6 mm,
4-layer, 1.0 mm, ENIG, impedance-controlled, ~280 solder joints, ~14 extended part
types, **double-sided assembly**.

| Line | 10 boards | 20 boards | 50 boards |
|------|----------:|----------:|----------:|
| PCB fab (4L, 1.0 mm, ENIG, imp. ctrl) | $28 | $42 | $80 |
| Stencils (2 sides) | $3 | $3 | $3 |
| SMT setup (2 sides) | $16 | $16 | $16 |
| Feeder fees (~14 extended parts) | $21 | $21 | $21 |
| SMT labour | $5 | $10 | $24 |
| X-ray (QFN-60) contingency | $10 | $10 | $10 |
| Components (incl. MOQ overshoot) | $97 | $161 | $326 |
| **Subtotal, ex-works** | **$180** | **$263** | **$480** |
| Air freight to UK | $22 | $25 | $32 |
| UK import VAT @20% | $40 | $58 | $102 |
| Courier disbursement fee | $15 | $15 | $15 |
| **Delivered total** | **$257** | **$361** | **$629** |
| **Per board** | **$25.70** | **$18.05** | **$12.58** |
| *Per board, £ @1.27* | *£20.24* | *£14.21* | *£9.91* |

The shape of that curve is the whole story: **$66 of the cost is fixed** (setup, feeders,
stencils, X-ray) regardless of quantity. At 10 boards you pay $6.60/board just to turn the
machine on; at 50 it is $1.32. If there is any chance of wanting 50, order 50.

### One-off NRE, not included above

| Item | Est. |
|------|-----:|
| 5-board prototype spin (same process) | $160 |
| Second spin, assume one is needed | $160 |
| Test monitor + HDMI cables + M2 hardware | $40 |
| **Total NRE** | **~$360** |

Fully loaded, a 50-board run lands at roughly **$20/board**; a 20-board run at
**$36/board**.

### Should you hand-assemble instead?

| | 10 boards | 50 boards |
|---|---:|---:|
| Turnkey PCBA, delivered | $257 | $629 |
| PCB + parts + own reflow, delivered | ~$205 | ~$530 |
| Saving | $52 | $99 |
| Your time | ~8 h | ~30 h |

**No.** You would be buying your own labour at under $4/hour to hand-place a 0.4 mm-pitch
QFN-60 and an HDMI shell, with yield risk on top. Use turnkey.

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
a protoboard hexpansion (`codemyriad/protogon` or DanNixon's devkit) and prove the three
things that could kill the design:

1. **EEPROM emulation enumerates reliably.** Cold-plug the hexpansion 50 times. Measure
   the worst-case time from port power-on to the badge's first I2C read, and the RP2350's
   time-to-I2C-ready. If these do not have clear daylight between them, §5's fallback
   becomes the primary plan and the schematic changes.
2. **SPI link speed.** Sweep 10→40 MHz over the HS pins with the real edge connector in
   circuit. Record the highest reliable rate and the MicroPython-side throughput.
3. **HSTX DVI at 320×240 doubled**, running from the badge's 3V3 with a current meter
   inline, to validate §4.3.

Everything downstream is cheap to change now and expensive to change later. Do not skip
this phase.

### Phase 1 — Schematic + layout (3–4 weeks part-time)

Start from `emfcamp/badge-2024-hardware/hexpansion` so the outline, tab, and mounting
holes are right by construction. 4-layer stackup: L1 signal/TMDS, L2 solid GND, L3
power, L4 signal. Length-match TMDS pairs, keep them on L1 over the L2 ground plane, and
keep the runs under ~30 mm (which the board size makes easy). Request VID/PID in week 1.

### Phase 2 — Prototype run (5 boards, ~2 weeks lead time)

### Phase 3 — Firmware (4–6 weeks part-time)

Pico SDK in C. Order of work: boot + flash/PSRAM bring-up → I2C target and EEPROM
emulation → HSTX DVI output → SPI slave with DMA (PIO-based, so clock-stretch and
back-pressure are under our control) → the drawing engine → EDID/mode negotiation →
HDMI audio if time allows.

### Phase 4 — Badge-side driver (1–2 weeks)

MicroPython app baked into the LittleFS image the fake EEPROM serves, plus a companion
app published to this repo's index so people can install it without owning the hardware
yet.

### Phase 5 — Production run (10/20/50 per §7)

---

## 9. Risk register

| # | Risk | Severity | Mitigation |
|--:|------|----------|------------|
| 1 | EEPROM emulation loses the enumeration race at power-on | **High** | Fast-boot I2C target; clock stretching; DNP 24C64 fallback footprint. Proven or disproven in Phase 0. |
| 2 | SPI link slower than 40 MHz in practice | Medium | Display-list architecture already assumes 2.5 MB/s. Degrades to fewer draw calls, not to a broken product. |
| 3 | 600 mA budget or inrush trips the port switch | Medium | Modest bulk capacitance, staged boost enable, measured in Phase 0. |
| 4 | Badge battery life halves when the card is running | Medium | USB-C power input with supply OR-ing. |
| 5 | TMDS signal integrity on a 1.0 mm 4-layer board | Low | Short runs, controlled impedance, proven direct-drive topology. |
| 6 | VID/PID not assigned in time | Low | Ask in week 1; costs nothing. |
| 7 | Cable strain on the edge connector | Medium | Both M2 mounting holes populated; strain-relief loop documented for users. |
| 8 | RP2350-E9 pull-down erratum bites on LS/CS lines | Low | External pull resistors everywhere it matters. |

---

## 10. Open decisions

These need your call before Phase 1 starts:

1. **HDMI Type A vs micro-HDMI Type D.** Type A (assumed above, 15.5 mm wide) means no
   adapter cable but more leverage on the connector. Type D is 8.5 mm and much lighter,
   but everyone needs a special cable. Both fit the 32 mm width; Type A plus USB-C is
   ~26.5 mm of the outer edge, which is tight but workable.
2. **USB-C: in or out?** It costs ~$0.40 and some edge space, and it buys self-powering
   plus solder-free firmware updates. Recommended in.
3. **microSD on GPIO3–7?** Space is the constraint, not cost. Probably not for v1 —
   16 MB of flash goes a long way when you are not storing video.
4. **Run size.** The fixed-cost curve in §7 says 50 if there is any chance of 50.
