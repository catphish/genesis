# FPGA Sega Genesis / Mega Drive: Scope

An FPGA reimplementation of the Sega Genesis / Mega Drive that runs original cartridges and controllers, with HDMI output.

## Design goals

- **Faithful cartridge bus first.** The cartridge slot behaves exactly like the original's 68000 bus, including bus arbitration, interrupts, /DTACK timing and the /TIME region, so anything that worked in the original slot works here.
- **Upgrade the output, not the console.** Video improvements happen after the VDP and are on by default.
- **Behaviour-changing enhancements are opt-in.** Anything that alters how the console behaves (e.g. removing the sprite-per-line limit) is an option, off by default.

## Feature set

- **Original cartridges** through a native cartridge slot. On-cart hardware (mappers, battery saves, SVP, lock-on) and flash carts work as on original hardware.
- **Cartridge audio input.** The slot's SL1/SR1 analogue inputs are digitised by a PCM1808 ADC and mixed into the HDMI audio, so a flash cart that plays Mega-CD audio can be heard.
- **Master System cartridges** through a Power Base Converter. The slot is wired for Mode 4 in full; the gateware (Z80 bus master, VDP Mode 4) is later work.
- **Two original controller ports** with per-pin direction control, as on the original I/O chip, so pads, 6-button pads, mice, multitaps and other port peripherals all work.
- **HDMI output** carrying digital video and audio at a 27 MHz pixel clock, in the standard format of each region: **720×480p60 (CTA-861 VIC 2/3, 858×525) for NTSC** and **720×576p50 (VIC 17/18, 864×625) for PAL**. Each is mandatory for TVs of its region, and both are within the ECP5 -6 datasheet limits (270 Mb/s per TMDS lane). The console is locked to the HDMI frame rate, so it runs slightly fast: +0.03% on NTSC (0.5 cent sharp) and +0.60% on PAL (10 cents sharp). No frames are repeated and no frame buffer is needed, only a couple of line buffers.
- **Scaler:** 2× line doubling in both formats. H40 (320 wide) becomes 640 pixels with 40-pixel side borders; H32 (256 wide) is scaled 2.5× horizontally to the same 640, which is non-integer and is smoothed by the blending filter. 224 lines become 448 and 240 lines (PAL V30) become 480. The 448/480-line interlaced modes are handled by bob or weave (see Open items). Optional horizontal blending filter to reproduce composite-video dither blending.
- **Cycle-accurate implementation** of all console chips in the FPGA.
- **Power switch and console reset button**, as on the original.
- **Single USB-C port** for power and programming. The console requires a USB PD charger offering 9 V; any 5 V source is enough for programming.

### Not supported

- 32X (its four video pins on the slot are left unconnected)
- Mega-CD / Sega CD (no expansion port). Mega-CD games on an FPGA flash cart are covered by the cartridge audio input.
- Loading games from SD card (use an off-the-shelf flash cart)
- Analog video or audio output
- CRT light guns (Menacer, Justifier)

## Architecture

| Block | Implementation |
|---|---|
| 68000 CPU | FPGA (fx68k) |
| Z80 CPU | FPGA (T80) |
| VDP, YM2612, SN76489, bus logic | FPGA, from the open-source MiSTer Genesis core |
| HDMI encoder and scaler | FPGA (e.g. hdl-util/hdmi encoder; custom line-doubling scaler), driving TMDS directly from bank 0 through series resistors |
| Z80 RAM, CRAM/VSRAM, VDP line and sprite buffers | FPGA block RAM |
| VDP VRAM and 68000 work RAM | External async SRAM on a **dedicated** bus, private to the FPGA |
| Cartridge audio ADC | PCM1808, I2S into the FPGA, resampled to 48 kHz and mixed |

All clocks derive from a single 27 MHz oscillator. No Sega firmware is required (TMSS is omitted).

### Clocking

HDMI is the timing master, and the console is clocked to complete exactly one frame per HDMI frame:

- **One PLL:** 27 MHz in, phase detector at 27 MHz, × 20 = 540 MHz VCO (ECP5 range 400–800 MHz). Its outputs are 135 MHz (÷4) for the TMDS output DDR stage (270 Mb/s per lane), 54 MHz (÷10) for the console and 27 MHz (÷20) for the pixel clock. Both regions use the same PLL settings; only the video timing differs. The second PLL is free.
- **Clock swallowing:** no PLL ratio can produce the console clocks below, because they contain the factor 131 (NTSC) or 313 (PAL). A fixed pattern therefore skips individual 54 MHz pulses through the ECP5's glitch-free `DCCA` clock gate. This pauses the whole console together, and cycles per frame stay exact.
- **Audio ADC clock:** SCKI is derived from a clean clock, never the swallowed 54 MHz, whose missing pulses would show up as conversion jitter. 27 MHz ÷ 2 = 13.5 MHz gives 256 fs at fs = 52.73 kHz (or 135 MHz ÷ 11 = 12.27 MHz for fs ≈ 47.94 kHz).

| | NTSC | PAL |
|---|---|---|
| Output | 720×480p, 858 × 525, 59.94 Hz | 720×576p, 864 × 625, 50.00 Hz |
| Console master clock, average | 53.7087 MHz | 53.5230 MHz |
| Native master clock | 53.693175 MHz | about 53.203 MHz |
| Native frame rate | 59.92 Hz | 49.70 Hz |
| Pulses skipped | 0.54% | 0.88% |
| Speed-up | +0.03% (0.5 cent sharp) | +0.60% (10 cents sharp) |

Audio is resampled to HDMI's 48 kHz in any case, so swallowing adds no audible effect beyond the pitch shift above. Both regions use one oscillator and one bitstream. Other modes, such as exact native speed with a few extra blanking lines, are gateware options to test on real TVs and need no hardware change.

**Oscillator decoupling: one 100 nF at the pins, no ferrite bead.** Lattice's checklist (FPGA-TN-02038, Figure 10.1) shows a 600 Ω bead with 1 µF + 100 nF, but the margins here don't need it. Epson asks for 0.01–0.1 µF. The PLL's phase detector runs at 27 MHz, where the ECP5 accepts 1,000 ps peak-to-peak input period jitter (tIPJIT, fPFD ≥ 20 MHz). The SG2016CAN's own jitter is 15–16 ps peak-to-peak (1.8 ps RMS) at 3.3 V, so the PLL, not the oscillator, sets HDMI clock quality. Lattice's Versa board does fit a bead, but on a SERDES reference clock, which this design doesn't use. Keep Y1 and its trace to E8 (PCLKT1_1) short and away from the SRAM and cartridge buses.

### Main memory

VRAM (64 KB) and 68000 work RAM (64 KB) both live in the external SRAM, on a bus
that is **private to the FPGA** and shares nothing with the cartridge slot.

The reason is capacity. The LFE5U-25F has 1,008 Kbit of block RAM as 56 x 18
Kbit blocks, and VRAM alone would take 32 of those 56 — before the Z80 RAM,
the VDP's line and sprite buffers, the fx68k microcode ROMs, the FM tables and
the scaler's line buffers. Holding VRAM and work RAM on-chip needs about
1.15 Mbit, which does not fit. Moving both out leaves roughly 25 of the 56
blocks in use, with headroom for the optional blending filter and the
sprite-per-line option.

Keeping the bus dedicated rather than shared with the cartridge is what makes
this work, and it also removes complexity rather than adding it:

- A 68000 cartridge cycle occupies the bus for about 520 ns (four clocks at
  7.67 MHz). On a shared bus that would stall every VDP fetch for the duration,
  which the VDP's display fetches cannot tolerate. A private bus lets the VDP
  read VRAM while the 68000 is still waiting on the cartridge.
- The cartridge data transceiver's direction then follows the 68000 bus cycle
  alone, instead of arbitrating between cartridge and SRAM accesses.
- No address or data multiplexing between the two buses.

The console core already has the interface for this. Its VRAM ports are
request/acknowledge handshakes (`vram_req`/`vram_ack`, and a 32-bit
`vram32_req`/`vram32_ack` prefetch path) because upstream drives VRAM from
SDRAM. A 10 ns asynchronous SRAM with fixed latency is simpler to drive than
the SDRAM those ports were written for.

**Bandwidth.** One 16-bit access every two MCLK cycles gives about 26.9 M
accesses/s, 53.8 MB/s. Demand is about 7.7 M accesses/s, 15.3 MB/s:

| Client | Accesses/s | MB/s |
|---|---|---|
| VDP scroll planes (H40) | 5.03 M | 10.1 |
| VDP sprites | 1.68 M | 3.4 |
| 68000 work RAM | 0.96 M | 1.9 |
| **total** | **7.67 M** | **15.3** |

That is 29% utilisation and 3.5x headroom. Single-cycle accesses are possible
(108 MB/s) but leave only 8.5 ns of the 18.5 ns clock period for FPGA
clock-to-output, board flight time and input setup once the SRAM's 10 ns is
accounted for. **Design for two-cycle accesses**; treat single-cycle as an
optimisation to attempt only if something later needs it.

**Arbitration.** One controller, priority ordered: **68000 first**, then VDP
32-bit prefetch, then VDP 16-bit.

That ordering is deliberate and is the opposite of the obvious choice. On real
hardware work RAM sits on the 68000 bus and VRAM is private to the VDP, so the
two **never contend**. Putting both in one SRAM introduces a contention the
original does not have, and the 68000 is the side where it shows: a read must
resolve within 261 ns of address-valid or /DTACK slips and a wait state appears
that the real console would not have inserted. A VDP burst of five slots ahead
of the CPU costs 186 ns and breaks that budget.

Giving the 68000 priority costs almost nothing in the other direction. It asks
for only 4% of the available slots, so the VDP is delayed by at most one 37 ns
slot, and the VDP's request/acknowledge prefetch path exists precisely to absorb
latency — upstream drives it from SDRAM, refresh stalls included. CPU timing is
observable by software and by the cartridge; a 37 ns VDP delay is not.

During VDP DMA the VDP does hog memory, but real hardware stalls the 68000
during DMA as well, so contention there is roughly faithful anyway.

**Map.** 128 KB of the 512 KB part is used:

| Range | Size | Contents |
|---|---|---|
| `0x00000-0x0FFFF` | 64 KB | VDP VRAM |
| `0x10000-0x1FFFF` | 64 KB | 68000 work RAM |
| `0x20000-0x7FFFF` | 384 KB | spare |

The spare space is a reason to keep the 256K x 16 part rather than drop to
64K x 16, which would fit the 128 KB exactly with no margin. A field store for
properly weaving the 448-line interlaced mode would need about 160 KB and fits
in what is left.

### Cartridge bus level shifting

The slot carries every Mega Drive signal: VA1–VA23, VD0–VD15, /AS, /CE_0,
/TIME, /CAS0, /UWR, /LWR, VCLK, /DTACK, /M3, /CART, /MRES, /VRES, /ASEL and
/CAS2, plus the analogue audio inputs SL1 and SR1. The 32X video pins EDCLK,
/HSYNC, /VSYNC and /YS (b12–b15) are **left unconnected**.

**Cartridge input levels are TTL.** Original cartridges drive with 5 V TTL
or NMOS outputs, and the 68000 itself only guarantees VOH ≥ 2.4 V, so every
input from the slot must accept VIH = 2.0 V.

#### Fast lines: three SN74ALVC164245DGG

Port A is VCCA = 3.3 V (FPGA side), port B is VCCB = 5 V (slot side). The B
inputs are TTL-level (VIH 2.0 V, VIL 0.8 V) and the B outputs reach
VOH ≥ 4.3 V. Per 8-bit half: /OE = L and DIR = L passes B → A (inbound);
/OE = L and DIR = H passes A → B (outbound); /OE = H isolates.

| Device | Half | Lines | DIR | /OE |
|---|---|---|---|---|
| U72 | both | VD0–VD15 | `CART_DATA_DIR` (K2) | `CART_DATA_OE` (K1) |
| U83 | both | VA1–VA16 | `CART_DIR` (K14) | `CART_OE` (J16) |
| U71 | 1 | VA17–VA19, /CAS0, /CE_0, /CAS2, VCLK, /LWR | `CART_DIR` | `CART_OE` |
| U71 | 2 | VA20–VA23, /UWR, /AS, /ASEL, /TIME | `CART_VA_HI_DIR` (J15) | `CART_OE` |

U71's halves split on the Master System boundary: every line that stays a
console output in Mode 4 is in half 1, and the lines that must stop being
driven in Mode 4 (VA21–VA23) are in half 2, whose other lines are unused in
Mode 4.

#### Idle state: disabled, inbound, pulled up

- **Each /OE net has a 4.7 kΩ pull-up to +3V3** (R77 `CART_DATA_OE`, R79
  `CART_OE`). With the FPGA unconfigured, all three devices are therefore
  disabled, which follows TI's power-up guidance (SCAS416Q §10: power VCCA
  first, tie /OE to VCCA so it ramps with it).
- **Value:** 4.7 kΩ, not 10 kΩ, because the pull-up must beat the ECP5's
  configuration pull-down. At the 150 µA worst case it leaves /OE at about
  2.6 V, above the 2.0 V VIH. 10 kΩ would leave only 1.8 V.
- **Each DIR has a 10 kΩ pull-down** (R82, R84, R87), so the devices default
  to inbound, which is TI's "keep DIR low".
- **Every slot-side line has a 47 kΩ pull-up to +5 V** (48 lines).
- The ALVC's input stages are always active on both ports, so nothing may
  float, and nothing does:
  - The slot side is held high by its pull-ups.
  - The FPGA side is held low by the ECP5's weak pull-downs (30–150 µA)
    during power-up and configuration, and is then driven by the gateware.
- The cartridge sees an idle bus (all strobes high) from power-on, through
  configuration and through any JTAG reconfiguration, which is how the
  original's bus looks with the 68000 in reset.
- With the 5 V rail off (a 5 V-only source), both ports stay high-impedance
  rather than driving the A side from an unpowered B side.

#### Gateware rules for the transceivers

- **/OE must be inactive while DIR changes** (no break-before-make). Turning
  the data bus around is therefore /OE high, change DIR, /OE low: at least two
  54 MHz cycles (46–56 ns), comfortably inside the 521 ns bus cycle.
- **At start-up**, /OE is already high from its pull-up. Drive it high, set the
  output banks outbound, then lower /OE before driving any A-side line.
- **U71 half 2 goes outbound only once /M3 confirms Mega Drive mode** (see
  below).
- **Pulse /VRES once at start-up.**
- **`PULLMODE=NONE`** on every FPGA pin with an external pull resistor.

#### Open-drain and status lines: BSS138

/DTACK, /MRES, /VRES, /CART and /M3 each go through a BSS138 bidirectional
shifter, with its gate on +3V3:

| Line | Slot-side pull-up | FPGA-side pull-up |
|---|---|---|
| /DTACK | 2.2 kΩ to +5 V, matching the original (Mega Drive II R15) | 10 kΩ to +3V3 |
| /MRES, /VRES, /CART, /M3 | 10 kΩ to +5 V | 10 kΩ to +3V3 |

The FPGA drives these **open-drain** (0 or Z, never 1) and can read them
back, so the console can both assert reset and see a cartridge asserting it.
/CART and /M3 are inputs only.

#### Master System mode and the /M3 interlock

A Power Base Converter pulls /M3 low. In Mode 4 it also pulls **VA22 to GND**,
and **VA21** too when running from the card slot, and **VA23** becomes the Pause
button, active low. So in Mode 4 those three are inputs to the console: VA21
selects card or cartridge, VA23 is Pause, VA22's purpose is unknown. Their
47 kΩ slot-side pull-ups make them read high when the converter is not pulling
them. Every other Mode 4 role is still a console output: `/CE_0` and `/CAS2`
become the chip enables for `$0000-$7FFF` and `$8000-$BFFF`, `/CAS0` is `/OE`,
`/LWR` is write, VA17/VA18 are the card and cartridge enables and VA19 is
`/IORQ`. `/VRES` is held low throughout.

Driving VA21–VA23 with a converter inserted would be a transceiver output into
ground. So `/M3` is a **safety interlock**, not just a mode strap, and it acts
through **DIR as well as /OE**: U71 half 2 idles inbound (its DIR pull-down)
and is turned outbound only once `m3_ok` is true. `m3_ok` is /M3 through a
two-stage synchroniser and an ~4.9 ms debounce, **fall immediate, rise
debounced**, and resets **low**, so an adapter already inserted at power-up
needs no race to be won. On the way up DIR reaches "out" with /OE inactive;
on the way down /OE goes inactive at once and DIR follows. Hot-plugging a
converter into a running console still crashes whatever was playing, which
beats damaging a transceiver.

Until there is a Master System machine in the gateware, Mode 4 still means
"drive nothing". When there is, it becomes: half 2 inbound, the rest of the bus
enabled, `/VRES` held low. `/CART` plus /M3 then distinguishes "no cartridge",
"Mega Drive cartridge" and "Master System cartridge".

The Mode 4 mapping follows Plutiedev's cartridge-slot page; an earlier forum
source disagreed on VA21 and the address offset. Verify against a real
converter before relying on it.

#### What the cartridge sees during console-internal cycles

The cartridge gets the full 68000 address on VA1–VA23 for **every** cycle, not
just its own, exactly as on the original. The data bus is the interesting case,
because work RAM now lives in a private SRAM rather than on the shared bus.

On the original, a 68000 work RAM read does put its data on the slot's VD lines,
because the slot shares the copper. But the console also tells the cartridge to
ignore it: **`/CAS0`, the cartridge's own output enable, is not asserted for
`$E00000-$FFFFFF`**, and `/CE_0` covers only `$000000-$3FFFFF`. So during a work
RAM access the cartridge is explicitly told the cycle is not its own. No
well-behaved cartridge can depend on that data.

So mirroring SRAM read data back onto the slot is **not required for
correctness**. It is cheap to do anyway, and the timing is not close:

| | ns |
|---|---|
| budget, address-valid to data latched (end of S6) | 391 |
| budget, address-valid to `/DTACK` for zero wait states | 261 |
| actual path: decode, SRAM, register, transceiver turnaround, propagation | **137** |

The outward direction on the data transceiver is needed regardless, for 68000
**writes** into cartridge space — `/TIME` mapper registers and on-cart save RAM.
Mirroring is just one more condition on a path that has to exist.

**Derive the outward enable from the same decode that produces `/CE_0` and
`/CAS0`.** If the "drive VD outward" condition and "the cartridge may drive"
condition can ever disagree, the transceiver's B side fights a cartridge ROM
output at 5 V. One decoder, so they cannot disagree.

### Cartridge audio input

SL1 (b1) and SR1 (b3) feed a **PCM1808** stereo ADC (U84), so cartridge audio,
such as Mega-CD audio from an FPGA flash cart, reaches HDMI.

Per channel, slot to ADC: a bidirectional H3V3L10B ESD diode at the pin (the
cart's audio can swing below 0 V), a 47 kΩ load to GND that holds an empty slot
quiet, and a 10 µF coupling capacitor into VINL/VINR (60 kΩ input impedance,
0.27 Hz high-pass). TI's 1 µF would give 2.7 Hz; 10 µF is used because the
coupling capacitor is a class-2 ceramic with about 2.5 V DC across it, and the
larger value keeps the signal voltage across it (and so its distortion)
negligible: about 1% of the signal at 20 Hz instead of 13%. Inputs settle in
about 0.6 s after power-up. Full scale is 0.6 × VCC = 3 Vpp; the PCM1808's own
anti-alias filter is relied on, with no external RC.

| Pin | Connection |
|---|---|
| VCC (3) | +5V, 100 nF + 2.2 µF. Unfiltered: noise from the buck is an accepted risk |
| VDD (4) | +3V3, 100 nF + 2.2 µF |
| VREF (1) | 100 nF + 10 µF to GND. This is the reference itself, so it keeps its full 10 µF |
| MD1, MD0 (11, 10) | GND: slave mode, FPGA supplies all clocks |
| FMT (12) | GND: I2S, 24-bit |
| SCKI (6) | FPGA J12, 4.7 kΩ pull-down. A halted SCKI holds the ADC in reset |
| LRCK (7), BCK (8) | FPGA H13, J13 (internal pull-downs) |
| DOUT (9) | FPGA H12 |

### Controller port level shifting

**One SN74LVC1T45 per signal pin, 14 in total.** On the original console every
controller pin is individually direction-programmable (one control-register
bit per pin), and real peripherals use that: TH is an output for pads and an
input for light guns; TR and TL are outputs for the mouse, multitaps and the
EA 4-Way Play; UP and DOWN are driven by the Activator and 4-Way Play; TL and
TR become serial TxD and RxD. Only pins 3 and 4 have no known use as outputs,
and they get a direction pin anyway. A banked transceiver flips a whole port
together, and TI's per-bit-direction parts stop at 3.6 V, so single-bit
dual-supply parts are the only fit. This is the same architecture as the
Mega Sg (14 × 1T45, series resistors, TVS diodes).

Per pin, FPGA to DE-9:

```
FPGA data ─── A [LVC1T45] B ──┬── 220 Ω ── ESD ── DE-9 pin
FPGA dir ──┬─ DIR             │
          10k                10k
           │                  │
          GND               +5 V
```

- **VCCA 3.3 V, VCCB the console 5 V rail.** The 5 V output high clears a
  stock pad's 74HC157 VIH of about 3.5 V, which 3.3 V would not.
- **Inputs are CMOS-level** (VIH = 0.7 × VCCB ≈ 3.5 V). Controllers are 5 V
  CMOS and drive to the rail, so this is accepted.
- **10 kΩ pull-up to +5 V on the port side.** Undriven inputs read high on the
  original, an empty port reads all ones, and detection routines depend on it.
  No pull-ups on the 3.3 V side; the transceiver passes the pulled-up level.
- **Pull-down on each DIR**, so every pin listens until the FPGA is configured
  — the original's reset state (control register = 0). The 1T45 has no /OE;
  with the 5 V rail down its VCC isolation holds both sides high impedance.
- **220 Ω series, 0805** on the port side. Software can and does drive a pin the
  peripheral is also driving (4-Way Play detection does this against a normal
  pad); 220 Ω limits a 5 V-versus-0 V fight to ~23 mA, inside the 1T45's
  ±32 mA rating, and still leaves a pulled-down input at ~0.1 V. A sustained
  fight dissipates about 116 mW, so these are 0805 (125 mW) rather than 0402
  (62.5 mW), which also suits their place right behind the ESD diodes.
- **ESD diode on every pin**, a TPD1E05U06 at the DE-9 side of the
  220 Ω resistor. See ESD protection.
- **Pin 5 (+5 V)** through a 200 mA polyfuse, with 10 µF on the port side.

FPGA pins: `ctrlN_d[6:0]` and `ctrlN_dir[6:0]` per port, bit order matching the
port data register {TH, TR, TL, D3, D2, D1, D0} — 28 in total, all in bank 1
(top). They run at microsecond rates, so any bank will do.

### Front panel

- **Power switch (SW4):** a latching push switch that turns a P-channel
  MOSFET high-side switch (Q6, AO3407A) on and off. Q6 sits in the input path
  after the JP1 selector, so it cuts everything, STM32 included; with the
  switch off the console cannot be programmed either. SW4 only pulls Q6's gate
  to GND against R148 (47k), so it carries about 106 µA (5 V source) to 190 µA
  (9 V) rather than the console's input
  current (up to about 1 A at 9 V), which no latching push switch at JLC is
  rated for. Q6's source is on the supply side, so its body diode blocks when
  off; Vgs is −9 V on (−5 V on a 5 V source), within its ±20 V rating even at
  the 17 V TVS clamp. R148 is 47k rather than 100k because ALPS specifies a
  minimum switching load of 50 µA at 3 V, which 100k would only just meet from
  a 5 V source.
- **Power and reset buttons** are a matched ALPS SPPH4 pair (same body,
  plunger and cap): SPPH410100 latching for SW4, SPPH410200 momentary for SW1.
- **Reset button (SW1):** to GND on FPGA ball D7 (net `RESET`), with a 10k
  pull-up to +3V3 (R149). The external pull-up gives 330 µA through the
  switch, above ALPS's 50 µA minimum; the ECP5's internal pull-up (30–150 µA)
  might not be. Gateware sets `PULLMODE=NONE` on D7 and debounces it. Before
  configuration R149 outweighs the pin's weak pull-down, so it reads released.
- **STM32 buttons:** SW2 resets the STM32; SW3 pulls BOOT0 high for the ROM
  DFU bootloader. Both are Omron B3U-1000P tactile switches.

### Gateware integration work

The console logic comes from existing open-source cores (GPL). Project work is integration:

- Port the MiSTer Genesis core from Intel/Quartus primitives (block RAM, PLLs, DDR I/O) to ECP5.
- Replace the core's SDRAM ROM interface, mapper and SVP emulation with a real 68000 bus interface to the cartridge transceivers. Nuked-MD-FPGA serves as the reference for exact bus and glue-logic behaviour.
- Write the external memory controller and arbiter: VRAM and 68000 work RAM in one async SRAM on a dedicated bus, priority 68000 > VDP prefetch > VDP (see Main memory). The core's existing `vram_req`/`vram_ack` and `vram32_*` handshakes are the attachment points.
- Transceiver sequencing, the /M3 interlock and the /VRES start-up pulse (see Cartridge bus level shifting). Open-drain drive on the BSS138 lines.
- Connect the I/O port registers to the real controller pins (including TH).
- PCM1808 I2S receiver (the FPGA is the clock master), resampler to 48 kHz, and mixer.
- Reset button: `PULLMODE=NONE` on D7 (external 10k pull-up R149), debounce, console soft reset.
- HDMI output, scaler and blending filter: 480p60/576p50 timing, 2× line doubling, 2.5× horizontal scaling for H32, interlace bob/weave, and HDMI audio. Read the EDID at hot-plug and fall back to 60 Hz for PAL games on sinks without 50 Hz support. The TMDS pins follow the rules in the HDMI section.
- Power-good handling. While PG is low:
  - the console is held in reset
  - the cartridge bus is disabled
  - the power LED shows red

  When PG goes high, after a short debounce, reset is released and the LED turns green.

The sources mix SystemVerilog and VHDL. Use Lattice Diamond, or Yosys with the GHDL plugin plus sv2v or yosys-slang.

The combined design is GPL; publishing the FPGA source is required if boards are ever distributed.

## Configuration and programming

- **Boot:** the FPGA boots in Master SPI mode from a QSPI flash (single image; no dual-boot). Quad reads enabled for fast boot.
  - CFG[2:0] = 010, strapped directly (CFG0 N10 and CFG2 R10 to GND, CFG1 P10 to +3V3).
  - Flash /CS is on **CSSPIN (N8)**, not SN (R8).
  - **22 Ω series resistors on all six flash lines**, at the FPGA end, and a **1 kΩ pull-up on MCLK**, per the Lattice checklist. 10 kΩ pull-ups on /CS, IO2 and IO3.
  - The W25Q32JVZPIQ (4 MB, WSON-8 6×5 mm) has QE = 1 as a factory-fixed default, as do all Winbond "IQ" ordering options, so /HOLD is disabled and IO2/IO3 are free for quad mode. 4 MB holds a 25F bitstream (under 1 MB) several times over, and a 45F bitstream as well.
  - PROGRAMN, INITN and DONE: 4.7 kΩ pull-ups. TCK: 4.7 kΩ pull-down; TDI, TMS, TDO: 4.7 kΩ pull-ups.
- **USB programming:** an STM32G431 on the USB-C data lines acts as a USB-to-JTAG adapter for openFPGALoader (DirtyJTAG port or CMSIS-DAP). It writes the configuration flash through the FPGA's JTAG port, so a bad bitstream is always recoverable. JTAG is on PA4–PA7 (TMS, TCK, TDO, TDI), the SPI1 pins, so SPI can accelerate shifting.
- **Reload:** the STM32 drives PROGRAMN (PA3) and reads INITN (PA1) and DONE (PA2).
- **STM32 recovery:** through its ROM USB DFU bootloader, using the BOOT0 button.
- **No debug pads.** There is no SWD, UART or JTAG header or pad set. All FPGA JTAG access goes through the STM32, and the STM32 is programmed only over USB DFU. This is an accepted risk; see Design decisions.
- **No STM32–FPGA data link** beyond JTAG and the configuration pins. None is planned; the ECP5's `JTAGG` primitive could carry user data over the existing JTAG wires if one is ever needed.

### Debugging

A powered USB-C hub whose downstream ports provide both data and PD (including 9 V) powers the console while the PC talks to the STM32. Verify the hub's 9 V capability with a PD tester first.

As a fallback, the board carries an unpopulated debug power input (J4). Moving JP1's shunt from VBUS to J4 lets a bench supply run the console while the PC still talks to the STM32 over USB data; nothing on the board senses VBUS.

## Power

### Power chain

USB-C VBUS (5 V, or 9 V after PD negotiation) or the debug input J4 is selected by a 3-pin header JP1 with a jumper shunt (default: VBUS) onto the node `POWER_IN`, where a single TVS (D4) protects whichever input is selected, then passes through the power switch (Q6, controlled by SW4) to the input node VCC. There are no OR diodes:

| Rail | Regulator | Input | Supplies |
|---|---|---|---|
| 3.3 V | TPS563257 buck (U74) | VCC | FPGA I/O, SRAM, flash, STM32, transceiver A-sides, PCM1808 VDD |
| 1.1 V | TPS563257 buck (U81) | 3.3 V | FPGA core |
| 2.5 V | LP5907 LDO (U70) | 3.3 V | FPGA VCCAUX (<100 mA) |
| 5.0 V (console) | TPS563257 buck (U80) | VCC | Cartridge, controller ports, transceiver B-sides, HDMI +5 V, PCM1808 VCC |

- **Bucks:** 5.00 V (220k/30k, 22 pF feed-forward), 3.327 V (150k/33k, 33 pF; 135k is a rare value, and 150k/33k keeps the 33 pF feed-forward zero at 32 kHz against TI's 36 kHz), 1.107 V (8.45k/10k), all per TI's Table 8-2. Inductors 2.2 µH, 2.2 µH, 1 µH (Sunlord MWSA0503S; see Component choices).
- **Inductor current rating:** set by the TPS563257's overcurrent limit, not the load. The limit acts on *valley* current, up to 5.0 A (3.1 A min, 4.1 A typ), so in an overload or short the peak is that plus the ripple: about 6.2 A for L1 (ripple 0.95 A nominal, 1.2 A at −20% L, from 9 V), 6.1 A for L2 (0.84 / 1.05 A) and 5.8 A for L3 (0.61 / 0.76 A, from 3.3 V). Hence Isat ≥ about 6.5 A for all three, and ≥3 A thermal (the buck's rating); normal loads are about 1 A (3.3 V), 1.5 A (5 V) and 0.5 A (1.1 V).
- **3.3 V enable:** 100k/47k on EN: on at VCC ≈ 3.7 V, off at ≈ 3.1 V.
- **Console 5 V enable:** 53.6k/10k on EN: on at VCC ≈ 7.6 V, off at ≈ 6.4 V. With a 5 V-only source the rail stays off; everything else powers up and programming works.
- **5 V waits for 3.3 V:** U74's open-drain PG drives U80's EN node (net `5V_EN`), wired-AND with the divider.
  - PG holds EN low until 3.3 V reaches 90% of its setpoint, plus a 1 ms delay. PG sinks 4 mA at 0.4 V, well below EN's 1.0 V off threshold.
  - Once released, the divider sets the threshold as before. EN never exceeds about 2.7 V even at 17 V, within PG's 6 V maximum.
  - This guarantees the ALVC164245's required order, VCCA (3.3 V) first, for every source, including a 9 V bench supply on J4, which would otherwise start both bucks together.
  - A 3.3 V fault (undervoltage or overvoltage) also drops 5 V.
- **Input selection, not OR-ing:** JP1 connects either VBUS or the debug input, never both, so there is no diode drop and neither source can back-feed the other. Bridging both sides of JP1 would tie a bench supply to the USB source, so it ships with one shunt, on VBUS. The shunt is not on the JLC BOM and is ordered separately.
- **1.1 V from 3.3 V:** the TPS563257's UVLO (rising 2.8–3.0 V, falling ≤2.8 V) leaves ≥200 mV of margin on a worst-case 3.2 V rail. It has its own 10 µF at its VIN pin.
- **Input capacitance:** U80 and U74 share 2 × 4.7 µF + 2 × 100 nF on VCC, which keeps the total within USB's 10 µF limit on VBUS. TI asks for more than 10 µF per device; this is an accepted risk.
- **Input transients:** USB-C sources always start at 5 V and slew to 9 V under PD control, so the 17 V input rating has ample margin.
- **No VBUS overvoltage protection:** the TVS clamps short surges only. A faulty charger holding VBUS above the bucks' 17 V rating would destroy them. This risk is accepted.
- **Power-good:** the console 5 V buck's PG output goes to FPGA ball A8, with a 10 kΩ pull-up to 3.3 V (see Gateware integration work).
- **Polyfuses:** one 200 mA (nSMD020) on each controller port's pin 5, and **one 1 A polyfuse (F1) feeding both cartridge +5 V pins** (J1 a2 and a31 joined behind it). A single fuse avoids two parallel polyfuses sharing current unevenly and tripping in cascade. Confirm the rating by measuring flash and SVP carts on the prototype.
- **Partial power-down:** with the 5 V rail off, the 1T45s isolate the controller ports (VCC isolation), and the ALVC164245s sit disabled (both ports high-impedance) on their /OE pull-ups.
- **Sequencing:** the ECP5 has no constraints between its rails. The one board-level constraint is 3.3 V before 5 V, for the ALVC164245s, which the PG-to-EN link enforces.
- **Budget:** ~1.4 A at 5 V (~7 W), including a cartridge allowance; under 1 A drawn from VBUS at 9 V.

### Decoupling

- **100 nF per IC supply pin.** Extra capacitors only where a datasheet asks for them.
- **Bulk capacitance is pooled per rail, not placed per pin.** Where a datasheet asks for 10 µF, a 2.2 µF beside the pin is fitted and the rest is shared with the rail's distributed bulk.
- **Buck output filters are sized on effective capacitance** (after DC-bias derating), as TI's Table 8-2 is. The 5 V output uses 2 × 10 µF (C39, C114) in place of TI's 22 µF: both give about 9 µF effective at 5 V.
- **LP5907:** 2.2 µF on input and output (C18, C19), not 1 µF. The datasheet requires at least 0.7 µF effective over all conditions, which a 1 µF X5R at 3.3 V can miss in the worst case.
- **STM32 VDDA/VREF+:** 2.2 µF (C13). ST asks for 1 µF; more is fine while VREFBUF is unused (VREFBUF accepts only 0.5–1.5 µF).
- **FPGA:** a top-side ring, with 8 × 100 nF + 5 × 10 µF on VCCIO (13 pins), 6 × 100 nF + 3 × 10 µF on VCC (1.1 V) and 2 × 100 nF + 4.7 µF on VCCAUX.
- **Connector power pins get bulk capacitance only** (10 µF), to buffer powered or passive devices plugged into them. No 100 nF is fitted at connectors.

### PD (STM32)

- **Negotiation:** the STM32's built-in UCPD peripheral negotiates USB PD and requests 9 V.
- **Cold start:** with no battery, every plug-in is a dead-battery start. PA9 and PA10 (UCPD1_DBCC1/2) are linked to CC1 and CC2, so the STM32 presents its 5.1 kΩ Rd while unpowered and the source applies 5 V. Firmware must program UCPD as a sink before setting `UCPD1_DBDIS`, or Rd disappears and the source removes VBUS.
- **CC protection:** ESD diodes only, plus 220 pF 50 V on each CC line for the USB PD capacitance range (200–600 pF, including about 60–90 pF from the STM32). The CC pins connect directly to the connector, so a CC-to-VBUS short at 9 V (for example, during an angled unplug) can damage the STM32. This risk is accepted.
- **Diagnostics:** none beyond USB. The firmware is simple, and there is no VBUS sense divider: the UCPD contract state is enough.
- **Debugger caveat:** if the STM32 is halted in a debugger, the source may hard-reset the PD contract, dropping the console 5 V rail. The console then resets cleanly via PG.

### Power LED (bi-colour, FPGA-driven)

- **Red:** PG low. No 9 V available, or the 5 V rail has dropped.
- **Green:** PG high. Console running.
- **Dark:** the FPGA is not yet configured.

Each colour is driven from its own FPGA pin (A7 red, B7 green) through 270 Ω (R58, R61), giving about 5 mA per colour from a forward voltage of about 1.9 V. The LED is a red / **yellow-green** pair, not red / emerald green: emerald greens need 2.8–3.4 V and would barely light from a 3.3 V pin. 5 mA is well inside the LED's 30 mA and the ECP5's 8 mA LVCMOS33 drive, and keeps bank 0 far below Lattice's 10 mA-per-I/O average (electromigration) limit alongside the HDMI pins. Final brightness is to be judged on the prototype; gateware PWM can dim it.

## ESD protection

**Every external signal pin gets an ESD diode; power pins don't.**

- **Signal pins:** one TPD1E05U06 per pin (71 in total), placed right at the connector with its ground pad on a via straight to the ground plane: 53 cartridge signals, 14 controller lines, and USB D+, D−, CC1 and CC2. One part number serves all of them, as every line is 5 V or below. On the controller ports the diode sits at the DE-9, ahead of the 220 Ω resistor. On the cartridge slot, route each line past its diode before the transceiver or BSS138.
- **Cartridge audio:** SL1 and SR1 each get a bidirectional H3V3L10B (3.3 V working, 3.5–5.5 V breakdown, 10 pF typical / 20 pF max, ≤0.5 µA leakage), because the audio swings below 0 V. The ADC's 3 Vpp full scale is ±1.5 V at the pin, well inside the 3.3 V working voltage.
- **Unused 32X pins (b12–b15):** unconnected and unprotected, as nothing is attached.
- **HDMI:** covered by the TPD12S016.
- **USB VBUS and the debug input:** one SMAJ10A (D4) on `POWER_IN`, after JP1 and ahead of Q6, for surges as well as ESD. It protects whichever input JP1 selects and clamps whether Q6 is on or off, so it also protects Q6. A bench supply connected backwards to J4 forward-biases D4, and the supply's current limit takes over.
- **Power pins (cartridge +5 V, controller pin 5, HDMI +5 V):** no diode. A rail's own capacitance absorbs an ESD pulse: an 8 kV IEC discharge carries about 1.2 µC (150 pF × 8 kV), which raises 10 µF by about 0.12 V.
- **Connector shells:** DE-9 and USB-C shells are grounded.
- **Accepted risk:** a CC-to-VBUS short at 9 V exceeds the diodes' 6.5–8.5 V breakdown, so the diodes don't protect against it.

## HDMI

**TMDS is driven directly by the FPGA from bank 0 (top), DC-coupled through a 270 Ω series resistor per line.** No transmitter or redriver chip. HDMI transmitter chips were rejected on cost and JLCPCB stock (ADV7513, IT66121) or for lacking audio (TFP410), and the DP159-class level shifters on cost once the output rate dropped to 270 Mb/s.

| Lane | FPGA balls (true/comp) | Series R | Connector | TPD12S016 |
|---|---|---|---|---|
| D2 | E4 / D4 | R56 / R57 | J2.1 / J2.3 | 23 / 22 |
| D1 | A3 / A4 | R59 / R60 | J2.4 / J2.6 | 21 / 20 |
| D0 | A2 / B3 | R62 / R63 | J2.7 / J2.9 | 18 / 17 |
| CLK | C4 / B4 | R64 / R65 | J2.10 / J2.12 | 16 / 15 |

- **Why bank 0:** top and bottom banks have a programmable clamp (default OFF; left/right banks are always clamped) and are hot-socket rated: ±1 mA maximum pin leakage with VCCIO absent (IDK_HS, datasheet Table 3.5), against 18–33 mA on left/right banks. A powered TV therefore can't back-feed the board through its termination. LVCMOS33D emulated differential works on every top/bottom pair, and the generic DDR output stage (500 Mb/s) and LVCMOS33 (~300 Mb/s) cover 270 Mb/s on the -6. 720p (742.5 Mb/s) is not possible on these banks.
- **How it makes TMDS levels:** the sink terminates each line with 50 Ω to its 3.3 V (AVcc). A leg driven low sinks about 3.3 V / (270 + 50 + driver) ≈ 9.5 mA, giving a 485 mV swing nominal. Across sink AVcc 3.135–3.465 V, 45–55 Ω termination, ±1% resistors and a 10–33 Ω driver, the swing stays within 402–573 mV (HDMI: 400–600 mV). The low corner is tight; 255 Ω (E96) would centre it at about 500 mV if prototype measurements call for it. One leg of each pair is always low, so the sink supplies about 38 mA in total; each resistor dissipates about 24 mW.
- **The TPD12S016 sits on the FPGA side of the resistors.** An ESD strike at the connector passes through 270 Ω before U82 clamps it. Its TMDS channels add about 1 pF and nA-level leakage.
- **Known deviations from a true current-mode TMDS driver** (accepted; see decision 17):
  - **High level:** a push-pull high meets the sink's AVcc ± 10 mV only if the board's 3.3 V and the sink's AVcc are within about 75 mV. Receivers are differential, so this affects formal compliance testing, not the picture.
  - **Board-off leakage:** the ±1 mA hot-socket figure is a maximum; HDMI allows about 200 µA (AVcc ± 10 mV into 50 Ω). Typical leakage is expected to be far lower.
  - **Back-powering a TV that is off:** a high leg sources about 10 mA into a sink whose AVcc is unpowered, which an open-drain TMDS driver never does. HPD can still read high then, because the sink's EDID and HPD run from the source's +5 V. The receiver-sense rule below prevents it.
- **Gateware rules for the TMDS pins:**
  - `IO_TYPE=LVCMOS33D`, `CLAMP=OFF` stated explicitly, `SLEWRATE=FAST`, `DRIVE=12` or `16`, ODDRX1 output stage clocked at 135 MHz.
  - **Receiver sense before driving:** tri-state the TMDS pins with weak pull-downs enabled and read them back. A powered sink holds them near 3.2 V; an unpowered one lets them fall low. Drive only when HPD is high **and** the lines read high; return to tri-state when HPD drops.
- **TPD12S016:**
  - DDC and CEC level shifting, with internal pull-ups on both sides.
  - HPD buffering.
  - The HDMI +5 V load switch, fed from the console +5 V rail.
  - VCCA on +3V3. All its FPGA-side signals are in bank 0: CEC D5, SCL C5, SDA B5, HPD A5.
  - The FPGA drives LS_OE (E6) and CT_HPD (A6). Both are low before configuration, so the sink sees no +5 V until the FPGA is running.

## Component choices

Part numbers, LCSC numbers and footprints are being assigned in the schematic (the `LCSC` and `Footprint` fields are authoritative). Passives and connectors chosen so far are listed under Passive parts below; inductors, the remaining discretes, switches and some connectors are still open.

| Function | Part | Notes |
|---|---|---|
| FPGA | Lattice **LFE5U-25F-6BG256C** | ECP5, 24K LUTs, ~1 Mbit block RAM, 197 I/O, caBGA-256 (0.8 mm). The design target: the smallest and slowest part supported. Gateware is built and timed for it (`SPEED := 6`). See FPGA alternates. **The prototype is built with the LFE5U-25F-7BG256I (LCSC C1550762)** because the -6BG256C (C1521614) is out of stock; U68's value and LCSC field deliberately differ. Footprint `Package_BGA:BGA-256_14.0x14.0mm_Layout16x16_P0.8mm_Ball0.45mm_Pad0.32mm_NSMD`, checked against Lattice FPGA-DS-02053 §93 (14 mm, 0.8 mm pitch, 0.45 mm nominal ball, MO-275 JJAB-2) and all 256 ball names. |
| VRAM + 68000 work RAM | ISSI **IS61WV25616BLL-10TLI** | 256K×16 async SRAM, 10 ns, 3.3 V, TSOP-II-44. LCSC C53825. 128 KB used of 512 KB, on a dedicated 39-pin bus (A0–A17, D0–D15, /CE /OE /WE /UB /LB). Keep the 256K×16 rather than 64K×16: see Main memory. |
| Configuration flash | Winbond **W25Q32JVZPIQ** | 4 MB QSPI NOR, WSON-8 6×5 mm, QE = 1 factory-fixed. Exposed pad (3.40 × 4.30 mm) unconnected internally; tied to GND. |
| Programming / PD MCU | ST **STM32G431CB** | 48-pin, 128 KB flash, 32 KB RAM. USB FS on PA11/PA12 (crystal-less), UCPD CC1/CC2 on PB6/PB4, dead-battery DBCC on PA9/PA10, ROM DFU bootloader. |
| Cartridge audio ADC | TI **PCM1808PWR** | 24-bit stereo ADC, 5 V analogue / 3.3 V digital, slave mode, I2S. TSSOP-14. |
| Signal ESD | TI **TPD1E05U06DPY** (×71) | One per external signal pin. Single-line, 5.5 V working, 6.5 V minimum breakdown, 0.42 pF, ±12 kV contact / ±15 kV air (IEC 61000-4-2). X1SON 0402; use TI's DPY land pattern. |
| Audio ESD | **H3V3L10B** (×2) | Hongjiacheng (LCSC C20615781), footprint `Diode_SMD:D_SOD-882`. Bidirectional, 3.3 V working, 3.5–5.5 V breakdown, clamps at 6 V (1 A) and 11 V (8 A, 8/20 µs), ±30 kV IEC 61000-4-2, 20 pF max. DFN1006-2L. |
| 5 V, 3.3 V and 1.1 V regulators | TI **TPS563257DRLR** (×3) | 3–17 V input, 3 A synchronous buck, 1.2 MHz, forced continuous mode, precision EN, open-drain PG, SOT-563. D-CAP3: output capacitors and feed-forward per datasheet tables. |
| Buck inductors | Sunlord **MWSA0503S** (×3), molded metal-alloy, shielded, 5.4 × 5.2 × 3 mm | L1, L2: **MWSA0503S-2R2MT** (2.2 µH, LCSC C408408); L3: **MWSA0503S-1R0MT** (1.0 µH, LCSC C408355). JLC lists 7 A / 7 A and 29 mΩ for the 2.2 µH, 10 A / 8.5 A and 14 mΩ for the 1.0 µH, against a worst-case overload peak of about 6.2 A (see Power); molded alloy saturates softly. One footprint for all three, `Inductor_SMD:L_Sunlord_MWSA0503S`, so the three buck layouts are identical. Confirm which listed figure is Isat and which is Irms on Sunlord's datasheet. |
| 2.5 V regulator | TI **LP5907MFX-2.5** | LDO from 3.3 V, <100 mA |
| Input selector | **JP1**, 1×3 2.54 mm header, XFCN PZ254V-11-03P (LCSC C2937625) | VBUS (default) or debug input to Q6. Pin 2 is the common. Jumper shunt ordered separately. Footprint `PinHeader_1x03_P2.54mm_Vertical`. |
| Debug power input | **J4**, 1×2 2.54 mm header, XFCN PZ254V-11-02P (LCSC C492401), not fitted | Pin 1 GND, pin 2 +9V. Footprint `PinHeader_1x02_P2.54mm_Vertical`. |
| Power switch FET | AOS **AO3407A** (LCSC C15155, Q6) | P-channel, −30 V, 4.3 A, 48 mΩ at −10 V, ±20 V gate, SOT-23 (`Q_PMOS_GSD`: 1 G, 2 S, 3 D). About 50 mW at 1 A. Gate pull-up R148 (47k) to the source. |
| Input TVS | **SMAJ10A** (×1, D4, on `POWER_IN`, ahead of Q6) | Diodes Inc. SMAJ10A-13-F (LCSC C134939), footprint `Diode_SMD:D_SMA`, pad 1 (cathode) on `POWER_IN`. Unidirectional, 400 W. 10 V standoff (above 9 V PD's 9.45 V), 11.1–12.3 V breakdown, 17.0 V clamp at 23.5 A (confirmed on the datasheet), which just meets the bucks' 17 V rating. |
| Polyfuses | **nSMD020-30V** (×2), **1 A** (×1) | nSMD020-30V, TECHFUSE (LCSC C69680), footprint `Fuse:Fuse_1206_3216Metric` (KiCad's pads leave a 1.55 mm gap where the datasheet suggests 2.0 mm; either solders): ratings identical to the Sea & Land / Alpha-Top datasheet: 0.20 A hold, 0.46 A trip, 30 V, 0.35 Ω minimum and 2.7 Ω maximum one hour after reflow, so up to about 0.5 V drop at the hold current; controller pin 5 ×2. Cartridge +5 V (F1): TECHFUSE **nSMD100-16V** (LCSC C70082), same maker and footprint: 1.0 A hold, 1.8 A trip, 16 V, 55–270 mΩ, so at most about 80 mV at typical cart currents and 0.27 V at 1 A. TECHFUSE and Alpha-Top (Sea & Land Alliance) are the same company. |
| Cartridge level shifting | TI **SN74ALVC164245DGG** (×3) | TTL-level B inputs, VOH ≥ 4.3 V (VCCB 4.5 V, −100 µA). VCCA 2.3–3.6 V, VCCB 3–5.5 V; TI's power-up order (VCCA first) is met because 3.3 V always precedes 5 V. Data (U72), VA1–VA16 (U83), the rest (U71). TSSOP-48. |
| Cartridge open-drain lines | onsemi **BSS138** (×5, LCSC C52895) | /DTACK, /MRES, /VRES, /CART, /M3. SOT-23, 1 G, 2 S, 3 D. |
| Controller port level shifting | TI **SN74LVC1T45DBV** (×14) | One per signal pin, each with its own DIR from the FPGA. |
| HDMI protection and interface | TI **TPD12S016** | ESD, +5 V switch, HPD/DDC level shifting |
| HDMI TMDS series resistors | **270 Ω** 1% (×8: R56, R57, R59, R60, R62–R65) | Set the TMDS swing with the sink's 50 Ω termination; see HDMI |
| Cartridge connector | WingTAT **ED64MEBGFBK** (LCSC C5242019) | 2×32, 2.54 mm, rows 5.08 mm, 1.57 mm card (Mega Drive carts are 1.6 mm), two Ø3.5 mm mounting ears 95.88 mm apart. Gold flash (1 µin) contacts; no thicker plating is stocked. EDAC 345/395-064-520-202 (Ø3.25 mm ears, flush; about £10 at DigiKey) is dimensionally interchangeable. Project footprint `genesis:CardEdge_MegaDrive_2x32_P2.54mm_Ears` fits both, with Ø3.4 mm NPTH ear holes at ±47.94 mm and copper keep-outs for M3 hardware; EDAC's STEP model is attached. WingTAT's ears sit about 5.4 mm above the board (spacer or case boss needed); EDAC's are flush. |
| Controller connectors | Foxconn **DT10121-H4W2-4F** DE-9 male right-angle ×2 (LCSC C426221, J5, J6) | Board locks, no screw locks (suits Sega pads), shell grounded, 2 A contacts, gold-flash contacts rated 500 cycles (DT10123 is the 30 µin version, if stocked). Footprint `Connector_Dsub:DSUB-9_Pins_Horizontal_P2.77x2.84mm_EdgePinOffset7.70mm_Housed_MountingHolesOffset9.12mm`, matching Foxconn's layout: 2.77 / 2.84 mm pins, board-lock holes 24.99 mm apart, 1.42 mm behind the 6–9 row. Place the PCB edge 8.08 mm from the 6–9 row per Foxconn, not at the footprint's 7.70 mm edge line. Check a real Sega controller fits. |
| HDMI connector | Molex **208658-1001** (LCSC C916313), type A, right-angle SMD (`datasheets/2086581051_sd.pdf`, which covers the whole 208658 family) | Footprint `Connector_Video:HDMI_A_Molex_208658-1001_Horizontal`, matching Molex's recommended layout: 19 pads 0.30 × 2.60 mm at 0.5 mm pitch, shell slots 14.50 mm apart (0.90 × 2.70 mm rear, 0.90 × 1.70 mm front, 5.96 mm apart), PCB edge 9.55 mm from the rear slots. 0.76 µm gold contacts; the -1051 is the gold-flash version on the same footprint. |
| USB connector | Molex **217179-0001** (LCSC C3197684), 16-pin USB 2.0 Type-C receptacle (`datasheets/2171790001_sd.pdf`) | Both D+/D− pairs connected; CC direct to the STM32 UCPD and DBCC pins. Project footprint `genesis:USB_C_Receptacle_Molex_217179-0001`, drawn from Molex's recommended layout: signal pads as the common 16-pin pattern (1.15 mm long), shell slots 8.64 mm apart with the rear pair 4.00 mm behind the front (0.6 × 1.4 mm front / 0.6 × 1.8 mm rear holes), no locating pegs. KiCad's G-Switch GT-USB-7010ASV footprint does not fit it (rear legs 4.20 mm, different slot sizes). SBU pads A8/B8 are unconnected. 5 A, 10,000 cycles. No 3D model yet. |
| Clocking | **One** 27 MHz CMOS oscillator, Epson **SG2016CAN** 27.000000 MHz TJGA (X1G0048010061, LCSC C881727, Y1) | 2.0 × 1.6 mm, 1.6–3.63 V, ±50 ppm over −40 to 85 °C (HDMI allows ±0.5%), 15–16 ps p-p jitter, 3 ms start-up. Pin 1 is standby (high or open runs), tied to +3V3. Footprint `Crystal:Crystal_SMD_SeikoEpson_FA128-4Pin_2.0x1.6mm`: KiCad has no SG2016 oscillator footprint, and this one matches Epson's recommended land (0.90 × 0.80 mm pads at 1.40 × 1.10 mm) to within 0.05 mm, with the same pin order. The only clock source, on PCLK ball E8 (PCLKT1_1). One ECP5 PLL makes the 135 MHz TMDS, 54 MHz console and 27 MHz pixel clocks (see Clocking). One 100 nF at the pins, no ferrite bead. |
| Power LED | Lite-On **LTST-C295KGKRKT** (LCSC C364568, D1) | Red (631 nm) and yellow-green (571 nm) in one 1.6 × 0.8 mm package, as two separate LEDs: green anode 1 / cathode 3, red anode 2 / cathode 4; both cathodes to GND (symbol `LED_Dual_AAKK`). Vf 2.0 V typ, 2.4 V max at 20 mA; 35 / 45 mcd at 20 mA. Footprint `LED_SMD:LED_LiteOn_LTST-C295K_1.6x0.8mm`, matching Lite-On's suggested pads (0.4 × 0.5 mm at ±0.35 / ±0.6 mm). Top-view: needs a light pipe or window above it. FPGA-driven from PG. |
| Switches | SW4 power: ALPS **SPPH410100** (latching, LCSC C370973); SW1 reset: ALPS **SPPH410200** (momentary, LCSC C202363); SW2, SW3: Omron **B3U-1000P** (LCSC C231329) | SPPH4: 0.1 A 30 V max, 50 µA 3 V min, 10,000 operations, 2.2 mm travel; project footprint `genesis:SW_ALPS_SPPH4_Vertical` (both poles paralleled: pad 1 = commons 2/5, pad 2 = NO contacts 3/6, NC 1/4 unconnected; location lug on the NO side). B3U: `Button_Switch_SMD:SW_SPST_B3U-1000P`. |
| Mounting holes | 6 (H1–H6) | Placeholders: style and quantity to be decided at layout. |

### Passive parts

All chosen to be JLCPCB parts; X5R/X7R MLCCs, ±1% thick-film resistors.

| Part | LCSC | Used for | Why |
|---|---|---|---|
| 100 nF 50 V X7R 0402 (Samsung CL05B104KB54PNC) | C307331 | All 75 decoupling caps | One part everywhere; 50 V covers the VCC caps (9 V, transients to the 17 V TVS clamp) |
| 10 µF 25 V X5R 0805 (Samsung CL21A106KAYNNNE) | C15850 | All bulk on +5V, +3V3, +1V1; PCM1808 VREF; audio coupling | No 10 µF sits above 5 V. 0805 per TI's buck table; keeps about 4–5 µF at 5 V, 6 µF at 3.3 V |
| 4.7 µF 25 V X5R 0805 (Samsung CL21A475KAQNNNE) | C1779 | Buck inputs on VCC (C22, C37); C12; VCCAUX (C91) | 25 V for the 9 V input and its transients. At 9 V each keeps only about half its value, so decision 2's shortfall is larger than the nominal figures suggest |
| 2.2 µF 16 V X5R 0603 (Samsung CL10A225KO8NNNC) | C23630 | ALVC and PCM1808 supplies; LP5907 in/out; STM32 VDDA | All on 5 V or less |
| 220 pF 50 V X7R 0402 | C1530 | USB CC lines | |
| 33 pF / 22 pF 50 V C0G 0402 | C1562 / C1555 | 3.3 V / 5 V buck feed-forward | |
| Resistors, 0402 ±1% | various (UniOhm 0402WGF series) | Everything except below | |
| 220 Ω 0805 ±1% (UniOhm 0805W8F2200T5E) | C17557 | Controller series resistors | 125 mW for a sustained 5 V-vs-0 V fight (about 116 mW) |

There is no 1 µF capacitor on the board: every former 1 µF position moved to 2.2 µF or 10 µF.

### FPGA alternates

Any of these can be fitted in place of the LFE5U-25F-6BG256C without changes to the schematic, footprint, pin map or bitstream. They share its die and caBGA-256 package. Gateware stays built for the -6 target, so it meets timing on all of them.

| Part | Speed grade | Temperature | Relative to the target |
|---|---|---|---|
| LFE5U-25F-7BG256C | -7 | Commercial, 0 to 85 °C junction | Faster |
| LFE5U-25F-8BG256C | -8 | Commercial | Faster |
| LFE5U-25F-7BG256I | -7 | Industrial, –40 to 100 °C junction | Faster in practice; see note |
| LFE5U-25F-8BG256I | -8 | Industrial | Faster in practice; see note |

**Not an alternate: LFE5U-25F-6BG256I.** Lattice publishes commercial timing and says industrial timing is "typically slower" at the same grade, so the -6I may be slightly slower than the -6C target.

**Industrial parts at -7 and -8:** one speed grade faster should outweigh the industrial derating, but Lattice only gives industrial numbers through Diamond, so this isn't confirmed from the datasheet.

**Also not alternates:**
- The LFE5U-45F is pin-identical in caBGA-256 (confirmed from the Trellis database; see `fpga/docs/pinout.md`) but needs a different bitstream (`DEVICE=45k`).
- The LFE5UM and LFE5UM5G (SERDES) parts have a different pinout.

The schematic symbol was drawn for the 45F. Its 197 user I/O balls and their functions were checked against the 25F pin database, and they match.

## PCB

- 6 layers, with adjacent ground and power planes on thin dielectric.
- Single-sided SMT assembly (JLCPCB). Through-hole connectors via JLC through-hole assembly or hand soldering.
- BGA decoupling in a top-side ring around the package. Unpopulated 0402 footprints under the BGA on the bottom side as a fallback.
- X-ray inspection of the BGA.
- HDMI routed as 100 Ω differential pairs, length-matched within each pair. Place the 270 Ω resistors and the TPD12S016 close to the connector; keep the FPGA-to-resistor run short.
- All FPGA I/O banks at 3.3 V; HDMI pairs use LVCMOS33D on true/complement pin pairs in bank 0.
- **Bank allocation:** bank 0 HDMI, LEDs, PG and reset button; bank 1 controller ports and the clock input; banks 2/3 SRAM, audio ADC, the address-transceiver controls and /MRES, /VRES; bank 6 cartridge address; bank 7 cartridge data, strobes, the data-transceiver controls and the other BSS138 lines; bank 8 configuration flash.
- **FPGA I/O:** 153 of 197 used, 44 spare.

  | Group | Pins |
  |---|---|
  | Cartridge bus, including 5 transceiver controls | 58 |
  | SRAM | 39 |
  | Controller ports | 28 |
  | HDMI (8 TMDS + 6 control) | 14 |
  | Flash (excluding dedicated MCLK) | 5 |
  | Audio ADC | 4 |
  | Clock, PG, LED ×2, reset button | 5 |
- Keep the PCM1808 and its input network beside slot pins b1/b3, away from the bus lines and the bucks.
- KiCad SRAM footprint: `Package_SO:TSOP-II-44_10.16x18.41mm_P0.8mm`.
- Configuration flash (WSON-8): no exposed vias under the centre pad, per Winbond. Include U69 in the X-ray inspection, as its joints can't be inspected visually.

## Design decisions and accepted risks

Recorded from the full schematic review. Each was considered and either accepted or deferred deliberately.

| # | Item | Decision |
|---|---|---|
| 1 | Part numbers | The `value` field carries the part number. Authoritative manufacturer and LCSC part numbers for all components are added in the next phase. |
| 2 | Buck input capacitance | 2 × 4.7 µF shared between U80 and U74, below TI's >10 µF per device, to respect USB's 10 µF VBUS limit. **Accepted risk.** |
| 3 | FPGA VCCIO decoupling | 8 × 100 nF plus a ring of 5 × 10 µF for the 13 VCCIO pins. **Accepted.** |
| 4 | Debug access | No STM32 SWD or UART pads: the firmware is simple. **Accepted risk.** FPGA JTAG is essential and is assumed always reachable through the STM32. |
| 5 | Flash lines | 22 Ω series on all six flash lines and a 1 kΩ MCLK pull-up, per the Lattice checklist. **Done.** |
| 6 | Front panel | Power switch and console reset button. **Done.** |
| 7 | STM32–FPGA link | None: no planned use. Skipped to reduce layout complexity. |
| 8 | Mounting holes | 6 placeholders. Style and quantity are decided at layout. |
| 9–11 | Passive specifications, MPNs, LCSC numbers, capacitor voltage ratings, inductor saturation current | **Done**, except the mounting holes (at layout). |
| 12 | Rail capacitance | Pooled bulk per rail, as engineered. **Accepted.** |
| 13 | Connector power pins | Bulk capacitance only, no 100 nF. **Accepted.** |
| 14 | Input OR diodes | Removed. A header JP1 with a shunt selects VBUS or the debug input. **Done.** |
| 15 | ALVC164245 /OE | **Changed to a 4.7 kΩ pull-up to +3V3** (R77, R79), per TI. The 47 kΩ slot pull-ups keep the cart bus idle with the transceivers disabled, so enabling them gained nothing. 4.7 kΩ beats the ECP5's configuration pull-down. DIR keeps its 10 kΩ pull-down. Power sequencing (3.3 V first) is enforced by U74 PG to U80 EN. |
| 16 | Cartridge +5 V | One 1 A polyfuse feeding both pins. **Done**; confirm with measured cart current. |
| 17 | HDMI TMDS | Output 480p60 (NTSC) / 576p50 (PAL) at 27 MHz, so 270 Mb/s per lane, within the -6 datasheet limits. TMDS driven directly from bank 0 (hot-socket rated, clamp off) through 270 Ω series resistors into the sink's 50 Ω termination. Transmitter chips rejected on cost and stock (ADV7513, IT66121) or for lacking audio (TFP410); DP159-class redrivers not needed at this rate. The high-level tolerance, the board-off leakage maximum and back-powering an unpowered TV are **accepted deviations**, the last handled by a gateware receiver-sense rule. **Done.** |
| 18 | PCM1808 VCC | Fed straight from +5V, no filter. Noise is an **accepted risk.** |
| — | Controller input levels | CMOS (VIH ≈ 3.5 V), not TTL. **Accepted.** |
| — | CC-to-VBUS short, VBUS overvoltage | **Accepted risks** (see PD and Power). |

## Open items

- **HDMI on the prototype:** measure the TMDS swing and eye at the connector into a real sink, and confirm 255 Ω vs 270 Ω. Check that the receiver-sense read-back distinguishes powered and unpowered sinks.
- **Cartridge +5 V polyfuse:** F1 is a 1 A hold part (nSMD100-16V); measure flash cart and SVP cart current draw on prototype hardware to confirm the rating.
- **Parts still to assign:** mounting holes (H1–H6), at layout.
- **Deferred schematic items:** test points on VCC, +5V, +3V3, +2V5, +1V1 and PG_5V; reference re-annotation.
- **FPGA docs are out of date:** `fpga/docs/pin-constraints.md` and `constraints/pinmap.csv` predate the bank re-plan (HDMI to bank 0, controllers to bank 1, cartridge bus to banks 6/7, clock to E8). They also put `flash_cs_n` on R8 (it is N8), describe FET-gate `cart_mres_drive`/`cart_vres_drive` (now open-drain through BSS138) and the old transceiver controls, and lack CART_DIR, the audio ADC and the reset button. Regenerate both from the netlist.
- Synthesize the MiSTer Genesis core (fx68k, T80, VDP, sound) against the LFE5U-25F to confirm fit. Fallbacks: the pin-identical LFE5U-45F (supply is limited), or two 25Fs split into system and video halves.
- Confirm toolchain language support (Diamond vs. Yosys with GHDL and sv2v/yosys-slang).
- Gateware: gate the data transceiver's `/OE` on cartridge presence as well as PG and the `/M3` interlock. Presence is **`/CART` low or `/M3` low**: either alone means something is in the slot, which covers a Power Base Converter whether or not it grounds `/CART`.
- Confirm the JLCPCB assembly tier supports the layer count, BGA pitch and X-ray inspection.
- Choose the STM32 JTAG firmware (DirtyJTAG port or CMSIS-DAP); confirm openFPGALoader support.
- Implement the STM32 PD sink firmware (9 V request).
- Verify the debug hub offers 9 V PD with data on its downstream ports (PD tester).
- Source and fit-check the cartridge slot (card thickness, height, retention).
- Decide on PAL / region support. This no longer affects the hardware: the 27 MHz oscillator covers both regions.
- Close timing on the dedicated SRAM bus. Two-cycle accesses at 54 MHz are the design target; confirm FPGA clock-to-output, board flight time and input setup against the 10 ns part, and keep the bus short.
- Decide whether to weave the 448-line interlaced mode using a field store in the spare 384 KB of SRAM, or leave it to the scaler.
