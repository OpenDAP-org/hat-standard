# HAT interface v1

- **Revision:** v1.1 (Draft), 2026-10-24. It replaces v1.0 (2026-10-22, commit `42fcf14`), whose changes are in §3; v1.0 superseded the [HAT interface v0](interface-v0.md), which stays as history. Nothing here is built or measured.
- **Status:** Draft. The pad map freezes only after the signal-integrity test coupon (ADR 0004, §10). v1.1 moves contacts against v1.0; this is allowed because no revision has been frozen and nothing has been built against v1.0 (ADR 0005). After freeze, a change of that kind is a new major revision.
- **Requirements:** HCS-001 to HCS-026, in particular HCS-004 (no cap on high-speed rates), HCS-005 (compromises justified), HCS-006 (one interconnect type), HCS-007 and HCS-017 (identification EEPROM, OpenDAP HAT identity format), HCS-013 (programmable HAT chips), HCS-015 (power classes, Draft), HCS-021 (hardware-agnostic), HCS-022 (reserved contacts), HCS-023 (device-side piece), HCS-024 (compatibility with other standards where cheap), HCS-025 (compatibility levels) and HCS-026 (USB 3 base, PCIe or CSI-2 extension on shared contacts).
- **Decisions:** [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md) (all-pogo interface, fixed device-side piece); [ADR 0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md) (native high-speed and safety lines, low-speed I/O over USB, one USB port); [ADR 0003](../adr/0003-compatibility-levels.md) (compatibility levels and pogo zones); [ADR 0004](../adr/0004-pogo-part-and-field-geometry.md) (pogo part and pitch: Harwin P70, Mill-Max 0965 second source, 2.54 mm); [ADR 0005](../adr/0005-interface-v1-1-shared-extension-and-contact-reduction.md) (v1.1: shared high-speed extension, contact reduction, one flat-cable connector); board ADR 0002 (the signals reach the device-side piece over a flat cable). Justifications (HCS-005): [interface reduction research](../research/2026-10-interface-reduction.md) §4 for v1.0, [v1.1 contact optimisation](../research/2026-10-interface-v1-1-reduction.md) for v1.1.
- **Conventions:** **UNVERIFIED** = no primary source checked. "Device" is the product the HAT attaches to (the T1 today); "HAT" is the accessory. Directions are written from the device: *out* = device to HAT, *in* = HAT to device.

## 1. Scope

This document defines the HAT interface by **compatibility level**, **function** and **interface class**: which signals exist, what each carries, its electrical class, and its contact position. It names no SoC, SoC ball or SoC peripheral instance (HCS-021). A device implements the interface with whatever silicon it has and documents that mapping in its own repository; for the T1 the mapping is in the board repository ([pin plan §4.3](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md), [pinout v1](https://github.com/opendap-project/board/blob/main/docs/specs/pinout-v1.md) HAT section).

The interface has two stages:

1. the **pogo field** on the device-side piece, against flat pads on the HAT (ADR 0001, HCS-014): up to 68 contacts in a 4 × 17 grid at 2.54 mm, in three nested zones (§7.1). This is the interface a HAT designer works with;
2. on the T1, a **flat cable** from the board to the device-side piece, with one Hirose DF40 100-position connector pair at each end carrying signals and power (§7.2). These positions matter to device and device-side piece designers only. Another device may connect its pogo field differently.

## 2. Compatibility levels

| Level | Functions (adds to the level below) | Pogo contacts in the zone | Cumulative | Reserved, cumulative |
|---|---|---|---|---|
| **L1 Core** | Power by class (`HAT_5V`, `HAT_3V3`, `GND`), presence (`HAT_DET#`), interrupt and wake (`HAT_INT#`), ID EEPROM (`ID_*`), general I2C (`I2C_*`), the USB 3 port with its USB 2 half (`USB3_*`), through which every management function reaches the device (§6) | 29 | 29 | 3 (10.3 %) |
| **L2 Control and audio** | Digital audio on the device's DAC clock domain (`I2S_*`), four fast GPIO (`GPIO0`–`GPIO3`) with alternate functions UART, SPI and SWD (§4.3), reset of the HAT's controller (`HAT_RST#`) | 16 | 45 | 6 (13.3 %) |
| **L3 High-speed extension** | One extension of five differential pairs and three sidebands (`EXT0±`–`EXT4±`, `EXT_SB0`–`EXT_SB2`) that carries **PCIe x1 or MIPI CSI-2 up to 4 lanes**, selected by the function the HAT declares (§4.4) | 23 | 68 | 8 (11.8 %) |

- **One field, one pinout:** the three levels share one pogo field and one pinout. Each level is a nested subset of the same fixed positions; there is no connector or pinout per level. An L1 HAT on an L3 device uses only the L1 zone; an L3 HAT on an L1 device finds its L1 contacts in the same place.
- **Compatibility rule (ADR 0003):** a HAT works on any device of its level or higher. A device that receives a HAT needing a higher level, or an extension function it does not offer, says so clearly through Blackbird OS (T1-045: never silent), and powers nothing it cannot support safely. The ID EEPROM is in L1, so every device reads the level and the extension function a HAT needs before it enables `HAT_5V`.
- **L3 devices offer both extension functions.** A device implements L3 only if it can carry both PCIe x1 and CSI-2 with 4 data lanes on the extension; then every L3 HAT works on every L3 device. A device that can offer only one of them declares L2 and describes the one it has in its capability manifest.
- **Declaration:** a device declares the level it implements. A HAT declares the level it needs and, at L3, its extension function in its identity record (HCS-017): the record carries an **interface level** field (`1`, `2` or `3`), an **extension function** field (none, PCIe with its lane count, CSI-2 with its data-lane count) and the interface revision (`1` for this document). The byte layout is defined with the OpenDAP HAT identity format, which is not specified yet (§10).
- **Device-side piece:** a device of level N populates the zones up to N: 29 contacts at L1, 45 at L2, 68 at L3. Unpopulated positions keep the same grid, so the L1 zone sits at the same place on every piece.
- **HAT side:** a HAT carries pads for the zones up to its level. Under the zones above its level it has no exposed copper, so the pogo contacts of a higher-level device touch nothing. An L3 HAT that uses fewer extension lanes than five leaves the pads of the unused pairs unconnected.
- **Conformance:** each level has its own conformance test suite (toolkit-verify), as ADR 0003 sets out; the suites are not written yet.
- **The T1 implements L3.**

## 3. Changes

### 3.1 v1.1 against v1.0

| Item | v1.0 | v1.1 | Reason (ADR 0005, v1.1 research) |
|---|---|---|---|
| L3 | PCIe x1 (3 pairs) and CSI-2 (5 pairs) side by side | One extension of 5 pairs: PCIe x1 **or** CSI-2 up to 4 lanes, by declared function; 2 pairs reserved for a second PCIe lane in PCIe mode | Maintainers' direction (HCS-026); −6 contacts |
| L3 sidebands | `PCIE_PERST#`, `PCIE_CLKREQ#`, `PCIE_WAKE#`; camera reset on a GPIO | `EXT_SB0`–`EXT_SB2`: PCIe sidebands in PCIe mode, sensor reset and two camera GPIO in CSI-2 mode | Same three contacts serve both functions |
| Line out (`LINE_OUT_L`, `LINE_OUT_R`, `AGND` ×2) | 4 | removed | Bit-perfect `I2S_DOUT` on the DAC clock replaces it; no analog through the spring field; −4 |
| SWD (`DBG_SWCLK`, `DBG_SWDIO`) | 2 dedicated | alternate function on `GPIO2`/`GPIO3` (§4.3) | −2; native recovery kept (HCS-013) |
| High-speed fence | 24 grounds, pairs vertical in ground columns | 13 grounds, pairs along rows A and D with shared grounds; power returns under the USB 3 pairs | −11 (**UNVERIFIED** until the coupon) |
| Low-speed grounds | 3 | 0 (the I2S clocks sit next to fence grounds) | −3 |
| Reserved | 14 | 8, the minimum that keeps ≥ 8 % per level after the audio-clock fallback | −6 |
| Pogo field | 4 × 25 = 100 at 2.54 mm | **4 × 17 = 68 at 2.54 mm** (40.64 × 7.62 mm span) | −32 % |
| Interface fault ceilings | `HAT_5V` 3.6 A, `HAT_3V3` 1.8 A, `GND` 5.4 A | 3.0 A, 1.5 A, 4.5 A | Set by the limiters, not by a connector count; the T1's limiters already sit at or below them |
| Flat cable (T1) | DF40 100 (72 used, 24 spare) + DF40 50 (46 of 46) | one DF40 100 (87 used, 9 spare) | Power and signals on one connector; 150 → 100 positions |

### 3.2 v1.0 against v0

| Item | v0 | v1.0 | Path |
|---|---|---|---|
| Levels | none | L1, L2, L3 with pogo zones | §2 |
| Second USB 2.0 port (`USB2B_D±`) | 1 pair | removed | Every USB function uses the one USB port; a HAT that needs a SuperSpeed device and other USB functions carries a USB 3 hub (§6) |
| UART, SPI, slow GPIO (`GPIO4`–`GPIO7`), JTAG data | 4 + 5 + 4 + 2 | removed | USB: CDC-ACM, DLN2, HID, CMSIS-DAP or FTDI-class probe (§6) |
| Debug | `DBG_TCK`, `DBG_TMS`, `DBG_RESET#` | `DBG_SWCLK`, `DBG_SWDIO`, `HAT_RST#` | Two-wire SWD and an open-drain reset (v1.1 moves SWD onto `GPIO2`/`GPIO3`) |
| Camera reset, camera clock | `CAM_RST#`, `CAM_MCLK` | removed | A GPIO (v1.1: `EXT_SB0`); an oscillator on the HAT |
| Digital audio | two clock domains, 11 lines | one port of 8 lines on the device's DAC clock domain | §4.2 |
| Line in | 2 | removed | An ADC on the HAT, through the audio port or USB Audio |

## 4. Interface by function

### 4.1 Contact list

| Level | Function | Signals (direction) | Signal contacts | Electrical class | Notes |
|---|---|---|---|---|---|
| L1 | Power | `HAT_5V`, `HAT_3V3`, `GND` | — | §5 | `HAT_5V` is bidirectional: out for HATs that draw power, in for a HAT that supplies the device (HCS-015, Draft) |
| L1 | Presence | `HAT_DET#` (in) | 1 | 3.3 V, active low; the HAT ties it to `GND`; pull-up on the device | Valid before any HAT power. On the T1 it also proves the flat-cable connector is mated |
| L1 | Interrupt and wake | `HAT_INT#` (in) | 1 | 3.3 V, active low, open drain on the HAT, pull-up on the device | Wake source. Also the interrupt a HAT may assert for events behind its USB functions (§6.4) |
| L1 | Identification | `ID_SCL`, `ID_SDA` | 2 | I2C, 3.3 V, standard or fast mode | Dedicated to the HAT EEPROM at 7-bit address 0x50 (HCS-007, ADR 0002). No other device on this bus. Powered from `HAT_3V3`. It holds the **OpenDAP HAT identity format** (HCS-017, Draft), whose layout is inspired by Raspberry Pi HAT+; staying readable by HAT+ tools is a *should* (HCS-024). Read before `HAT_5V` is enabled, so the power class, level and extension function are known before power is applied |
| L1 | I2C | `I2C_SCL`, `I2C_SDA` | 2 | I2C, 3.3 V | General-purpose bus for kernel devices described in the device tree, and the camera control (CCI) bus. HAT designers keep addresses distinct |
| L1 | USB 3 | `USB3_SSTX±` (out), `USB3_SSRX±` (in), `USB3_D±` (USB 2.0 pair of the same port) | 6 (3 pairs) | USB 3 SuperSpeed + USB 2.0, 90 Ω differential | The only USB port to the HAT. Device is host. TX AC-coupling capacitors on the transmitting side (device for `SSTX`, HAT for `SSRX`). Management functions use it (§6) |
| L2 | Digital audio | `I2S_MCLK`, `I2S_BCLK`, `I2S_FS`, `I2S_DOUT` (all out), `I2S_DIN0`–`I2S_DIN3` (in) | 8 | 3.3 V push-pull | One full-duplex port on the device's DAC clock domain (§4.2) |
| L2 | GPIO | `GPIO0`–`GPIO3` | 4 | 3.3 V | Every line is PWM-capable as an output and interrupt-capable as an input. A device that cannot offer both on all four lines does not meet L2. Alternate functions in §4.3 |
| L2 | Reset | `HAT_RST#` (out) | 1 | 3.3 V, open drain, active low | Resets the HAT's main controller; asserted from device reset until the device releases it. With SWD (§4.3) it reaches a bricked USB bridge MCU too (HCS-013) |
| L3 | High-speed extension | `EXT0±`–`EXT4±` | 10 (5 pairs) | 100 Ω differential (**UNVERIFIED** for PCIe against the PCIe CEM text; set with the FPC stack-up) | Function per pair by mode (§4.4) |
| L3 | Extension sidebands | `EXT_SB0` (out), `EXT_SB1`, `EXT_SB2` | 3 | 3.3 V | Function by mode (§4.4) |
| L1, L3 | High-speed ground fence | `GND` | — | — | 5 pogo contacts in L1, 8 in L3 (§7.1) |
| all | Reserved | `RSVD_1A`–`RSVD_1C`, `RSVD_2A`–`RSVD_2C`, `RSVD_3A`, `RSVD_3B` | 8 | — | §8 |

Native low-speed signals: 22, including `HAT_DET#` (v1.0: 24; v0: 44): 6 in L1, 13 in L2, 3 in L3.

### 4.2 Electrical levels and line rules

- **Logic levels.** Every low-speed signal at the interface is 3.3 V logic. A device whose controllers run at another level translates on its own side; so does a HAT. Every 3.3 V signal on the device side is either isolated or in a defined state while `HAT_DET#` is high (no HAT).
- **Open-drain lines.** `HAT_DET#`, `HAT_INT#` and `HAT_RST#` are active low. `HAT_INT#` is driven by the HAT, `HAT_RST#` by the device, both open drain. Pull-up ownership on `HAT_RST#`, `I2C_*`, `ID_*` and the extension sidebands is open (§10).
- **GPIO.** `GPIO0`–`GPIO3` cover timestamped pulses (for example a GNSS PPS), resets of device-tree-described HAT devices (a modem), PWM and boot straps. Their use is declared in the capability manifest (HCS-009).
- **Digital audio.** The port runs on the device's DAC clock domain, the clock of the device's own playback (ADR 0002, ADR 0003): the device provides `I2S_MCLK`, `I2S_BCLK` and `I2S_FS`, and the HAT is the clock consumer. `I2S_DOUT` and `I2S_DIN0`–`I2S_DIN3` share those clocks, so HAT audio is sample-synchronous with the device's playback. Four input lines carry up to 8 channels as 4 × 2 channels, for example 8 channels at 192 kHz with `I2S_BCLK` at 12.288 MHz. A device declares the formats and rates it supports (TDM included) in its own manifest. A HAT that needs its own audio clock uses USB Audio (asynchronous, the HAT clock is the master). The one-domain rule is conditional on the first T1 prototype (ADR 0002; §10).
- **No analog audio at the interface.** A HAT that renders the device's audio takes it as bit-perfect PCM on `I2S_DOUT`, synchronous with the device's DAC clock, and carries its own converter.
- **High-speed rates.** The interface sets the lane count and the electrical class, not a rate (HCS-004). Each link runs at the highest rate both sides and the channel support; the device and the HAT declare their rates in the capability manifest (HCS-009). Rates are measured on hardware at the end.

### 4.3 Alternate functions on the GPIO lines

`GPIO0`–`GPIO3` carry the alternate functions below, configured by the device's pin multiplexer or by software on a GPIO, with no extra hardware. A HAT that wants one declares it in its identity record, per line. Every other contact keeps a fixed function.

| Alternate | `GPIO0` | `GPIO1` | `GPIO2` | `GPIO3` | Device support |
|---|---|---|---|---|---|
| UART A | TX (out) | RX (in) | — | — | should |
| UART B | — | — | TX (out) | RX (in) | should |
| SPI controller | SCLK (out) | MOSI (out) | MISO (in) | CS0# (out) | should |
| **SWD** | — | — | SWCLK (out) | SWDIO (bidirectional) | **must at L2** |

- **SWD is part of L2.** Two-wire Arm Serial Wire Debug to the HAT's main controller and any SWD target the HAT routes to it, with `HAT_RST#` for reset. Every L2 device offers it on `GPIO2`/`GPIO3` (bit-banged SWD needs only GPIO), so a bricked USB bridge MCU is always recoverable from the device (HCS-013). A HAT that routes SWD to these lines cannot use them, nor UART B, for anything else; it uses CDC-ACM for its console. The SWD clock rate is set by the device and is not specified here.
- **The other alternates are optional for a device.** A device declares which it supports. A device that supports none stays compatible: it treats the lines as plain GPIO. A HAT function that depends on an alternate the device lacks is then unavailable or reached another way (for example over USB), and Blackbird OS says so (T1-045).
- **Defaults:** each line starts as a GPIO input after power-up and changes function only after the device has read the identity record.
- **T1:** the GPIO balls offer UART A, UART B and bit-banged SWD; SPI is not available on them (board pin plan §4.3).

### 4.4 High-speed extension (L3)

The extension carries one function at a time, chosen by the HAT's declared extension function. Both functions start at `EXT0`, so a 2-lane camera and a PCIe x1 HAT use `EXT0`–`EXT2` only.

| Contact | PCIe mode | CSI-2 mode |
|---|---|---|
| `EXT0±` | `PCIE_REFCLK±` (out), 100 MHz from the device | `CSI_D0±` (in) |
| `EXT1±` | `PCIE_TX±` (out) | `CSI_CLK±` (in) |
| `EXT2±` | `PCIE_RX±` (in) | `CSI_D1±` (in) |
| `EXT3±` | reserved: lane 1 TX of a later revision | `CSI_D2±` (in) |
| `EXT4±` | reserved: lane 1 RX of a later revision | `CSI_D3±` (in) |
| `EXT_SB0` | `PCIE_PERST#` (out) | Sensor reset or enable (out, active low) |
| `EXT_SB1` | `PCIE_CLKREQ#` (in, open drain) | Camera GPIO, declared (for example power-down or flash strobe) |
| `EXT_SB2` | `PCIE_WAKE#` (in, open drain) | Camera GPIO, declared (for example frame sync) |

- **PCIe:** the device is root complex and sources the reference clock. TX AC-coupling capacitors on the transmitting side; on the device they sit before any switch, never on the shared path, because CSI-2 is DC-coupled. PCIe x1 in this revision.
- **CSI-2:** MIPI D-PHY, one clock lane and 1, 2 or 4 data lanes as declared; camera control on `I2C_*`. The HAT carries its own sensor clock.
- **Sidebands:** `EXT_SB0` is held asserted by the device until HAT power is good, in both modes. `EXT_SB1` and `EXT_SB2` have pull-ups on the device; in CSI-2 mode they are GPIO, and their use is declared in the manifest.
- **Selection (device duties):** while no L3 HAT is declared, or before the identity record is read, the device keeps the extension pairs disconnected from its PHYs (high impedance) and `EXT_SB0` asserted. After reading the record it selects the declared function, enables `HAT_5V`, connects the pairs, then releases `EXT_SB0`. How a device selects is its own choice: the SoC pin multiplexer where one PHY or pin group serves both functions, or a passive 2:1 switch rated for both protocols at their full rates (the T1: board pinout, ADR 0005). The selection element must not limit either link (board BRD-042).
- **Not concurrent:** one HAT uses PCIe or CSI-2, not both. A HAT that needs both moves one of them to the USB 3 port (for example a UVC camera, or NVMe behind a USB bridge).

## 5. Power and ground

Ratings of the interface itself, all in L1. Device budgets (how much a given device supplies on battery or on external power) are set in that device's documents and declared to the HAT; the HAT power classes (HCS-015, Draft) are written against them.

| Rail | Nominal | Direction | Interface rating (continuous) | Interface fault ceiling | Pogo contacts | DF40 contacts (T1) |
|---|---|---|---|---|---|---|
| `HAT_5V` | 5 V (window **UNVERIFIED**, set with the power classes) | out (classes A/B) or in (class F) | 2.0 A either direction | 3.0 A | 3 | 10 |
| `HAT_3V3` | 3.3 V | out | 1.0 A | 1.5 A | 2 | 5 |
| `GND` (power return) | — | return | 3.0 A | 4.5 A | 4 | 15 |

Margin rule (board BRD-049): each contact at or below **70 % of its rating in continuous use** with all contacts loaded, and at or below **100 % at the fault ceiling**, the highest current a device's limiter may let through in the worst case. v1.1 sets the ceilings from the limiters (ADR 0005); v1.0 had derived them from the DF40 count (3.6, 1.8 and 5.4 A).

| Contact | Rating | Rail | Continuous per contact | With one contact open | At the fault ceiling | Ceiling, one contact open |
|---|---|---|---|---|---|---|
| Pogo (Harwin P70-1000045R drawing) | 2.0 A | `HAT_5V` ×3 | 0.67 A (33 %) | 1.0 A (50 %) | 1.0 A (50 %) | 1.5 A (75 %) |
| | | `HAT_3V3` ×2 | 0.50 A (25 %) | 1.0 A (50 %) | 0.75 A (38 %) | 1.5 A (75 %) |
| | | `GND` ×4 | 0.75 A (38 %) | 1.0 A (50 %) | 1.13 A (56 %) | 1.5 A (75 %) |
| DF40 (Hirose catalog 2020, p. 2) | 0.3 A | `HAT_5V` ×10 | 0.20 A (67 %) | — | 0.30 A (100 %) | — |
| | | `HAT_3V3` ×5 | 0.20 A (67 %) | — | 0.30 A (100 %) | — |
| | | `GND` ×15 | 0.20 A (67 %) | — | 0.30 A (100 %) | — |

- **Pogo count rule (v0 §3):** the fewest contacts that stay at or below 70 % of the pogo rating at the interface rating, plus one for redundancy: `HAT_5V` 2 + 1, `HAT_3V3` 1 + 1, `GND` 3 + 1. Each rail stays inside the rule with one contact open. Fence grounds are not counted as power returns. The second source, Mill-Max 0965, has the same 2 A rating (ADR 0004); any other part is re-checked against its rating.
- **Device duties.** The device limits each rail it sources with a current limiter whose maximum trip current, tolerance included, stays inside the fault ceiling, and limits current drawn from a class F HAT on `HAT_5V` the same way. It enables `HAT_5V` only after reading the HAT EEPROM, and only for a HAT whose level, power class and extension function it supports. A device may set lower limits; the T1's are in the board [power tree §4.6](https://github.com/opendap-project/board/blob/main/docs/specs/power-tree-v0.md).
- **HAT duties.** The HAT powers its EEPROM from `HAT_3V3`. Bulk capacitance on each rail stays at or below 470 µF (board power tree §4.6; **Draft**). USB devices on a HAT are self-powered and take their VBUS-sense input from `HAT_5V` or `HAT_3V3`: the interface has no VBUS contact.
- **Flat cable (T1).** Its power and ground conductors are sized for the interface rating and fault ceilings (board BOM line 102).

## 6. USB port and management functions

### 6.1 One USB port

The HAT has one USB port, in L1: `USB3_SSTX±`, `USB3_SSRX±` and `USB3_D±`, the SuperSpeed lanes and the USB 2.0 pair of the same port. Low-speed functions that v1 removed from the interface (§3.2) reach the device through this port as USB devices on the HAT. This document calls them **management functions**.

- **Simple HATs** connect one USB device (a bridge, an MCU, a probe) to `USB3_D±`. A device with several functions presents them as one composite device (for example DLN2 + CDC-ACM + CMSIS-DAP from one MCU), which needs no hub.
- **HATs that need a SuperSpeed device and management functions at the same time** carry a standard USB 3 hub (USB hub class, mainline Linux `hub` driver) and connect both behind it. Mainline `onboard_usb_dev` handles the reset and supply of several hubs (driver table, **UNVERIFIED** per hub part).
- **USB-native HATs** (for example a NanoVNA- or HackRF-based design) use the same port for their main data.
- **Power:** USB devices on a HAT are self-powered (§5).
- **Availability (T1):** the USB 3 port goes to the HAT (board BRD-044).
- **Manifest:** the HAT lists its USB functions (USB IDs and classes) in its capability manifest (HCS-009). The schema is not defined yet.
- **Flashing at L1:** an L1 device has no native SWD. A programmable chip on an L1 HAT is flashed and debugged through the USB port (a ROM USB bootloader, DFU, or a probe on the HAT, HCS-013). A HAT that needs native SWD for recovery declares L2.

### 6.2 Standard USB classes

Any standard class with a mainline Linux driver is valid. The common ones for management functions ([interface reduction research](../research/2026-10-interface-reduction.md) §3.1):

| Function | USB class | Linux (mainline) |
|---|---|---|
| Serial (console, AT, NMEA) | CDC-ACM | `cdc-acm` |
| Input, sensors | HID | `usbhid`, `hid-sensor-hub` |
| Audio, MIDI | USB Audio Class 2, USB MIDI | `snd-usb-audio` |
| Video | UVC | `uvcvideo` |
| Display | GUD | `gud` |
| Instruments | USBTMC | `usbtmc` |
| Debug and flashing (SWD, JTAG) | CMSIS-DAP v1 (HID) / v2 (bulk) | user space: OpenOCD, openFPGALoader, probe-rs |
| Firmware update | DFU, vendor ROM loaders | user space (`dfu-util`, `picotool`) |

Single-purpose bridge chips with fixed USB IDs and mainline drivers (for example MCP2221A, CP2112, CP2102N) are also valid (research §3.2).

### 6.3 Reference bridge protocol: DLN2

The reference protocol for GPIO, I2C, SPI and ADC behind USB is **DLN2** (ADR 0002).

- **Linux:** mainline drivers `MFD_DLN2` (core), `GPIO_DLN2` (GPIO with interrupts), `I2C_DLN2`, `SPI_DLN2` and `DLN2_ADC`, with no ACPI dependency, so they work on a device-tree system (research §3.2, read 2026-10-22).
- **USB ID:** the mainline `dln2` ID table holds only `a257:2013`. Until OpenDAP's ID patch is merged upstream, Blackbird OS binds other IDs with a udev rule (`new_id`). The USB ID OpenDAP firmware uses is not assigned yet (§10).
- **Firmware:** OpenDAP publishes open DLN2-compatible firmware for common MCUs (ADR 0002). An existing open implementation for the RP2040, `notro/pico-usb-io-board` (CC0, ID `1d50:6170`), also exposes CDC-ACM UARTs.
- **Composite use:** one MCU can expose DLN2, CDC-ACM and CMSIS-DAP at once.

### 6.4 What a function behind USB does not have

| Property | Native line | Behind USB |
|---|---|---|
| Available before Linux | Yes | No: only after the HAT rails are on and the USB device has enumerated |
| Interrupt timing | Hardware interrupt, timestamped | Polled interrupt endpoint (1 ms at full speed, 125 µs at high speed); not usable for PPS |
| Wake from suspend | `HAT_INT#` | USB remote wake-up from the T1's deep suspend is **UNVERIFIED**; a HAT that must wake the device asserts `HAT_INT#` |
| Device-tree description | Normal | Devices on a bridged I2C or SPI bus are created from user space or a driver, not from an overlay (**UNVERIFIED** for device-tree children of `dln2`) |

Functions that need any of these properties use the native lines (§4.1).

## 7. Contact positions

### 7.1 Pogo field (68 contacts, 4 × 17 at 2.54 mm, three zones)

Positions are given looking at the pogo tips on the device-side piece: rows A to D, row A nearest the flat-cable tail; columns 1 to 17, column 1 at the L1 end. The HAT pad map is the mirror image of this table. Every populated position uses the same pogo part (HCS-006): Harwin P70, with Mill-Max 0965 as the second source on the same pads, at **2.54 mm** pitch, the field lying along the body's length (ADR 0004). Contact span 40.64 × 7.62 mm.

| Zone | Positions | Contacts |
|---|---|---|
| L1 | columns 1–7, all rows; B8 | 29 |
| L2 | C8, C9, D8, D9, B10, C10, C12, B13, C13, C14, B16, C16, and column 17 (A–D) | 16 |
| L3 | A8–A16, B9, B11, B12, B14, B15, C11, C15, D10–D16 | 23 |

Layout rules:

- **High-speed pairs lie along the outer rows** A and D, both contacts of a pair in adjacent columns, with a `GND` on each side in the same row; adjacent pairs share that ground (G S S G S S G). The inner rows B and C never hold a pair. The L1 band (USB 3) is fenced on its own, so an L1 piece needs no L3 contact.
- **Next to a pair contact** (the position directly above or below it in rows B and C) sit only `GND`, power or static and slow lines: power returns under the USB 3 SuperSpeed pairs, presence, the sidebands, `HAT_RST#` and reserved contacts. A later revision that assigns a reserved contact there assigns a static or slow function.
- **Power** at the L1 end, every `HAT_5V` and `HAT_3V3` next to a `GND`.
- **Audio clocks** at column 17, the end farthest from the pairs, `I2S_MCLK` and `I2S_BCLK` each next to a fence `GND`.
- **Whether the rows next to the pairs need more grounds** is decided by the coupon, with a costed fallback (§10).

| Col | Row A | Row B | Row C | Row D |
|---|---|---|---|---|
| 1 | GND | HAT_5V | HAT_3V3 | GND |
| 2 | USB3_SSTX+ | GND | HAT_3V3 | USB3_D+ |
| 3 | USB3_SSTX− | GND | HAT_DET# | USB3_D− |
| 4 | GND | HAT_5V | ID_SCL | GND |
| 5 | USB3_SSRX+ | GND | ID_SDA | HAT_INT# |
| 6 | USB3_SSRX− | GND | I2C_SCL | RSVD_1A |
| 7 | GND | HAT_5V | I2C_SDA | RSVD_1B |
| 8 | EXT0+ | RSVD_1C | I2S_DIN0 | I2S_DIN2 |
| 9 | EXT0− | EXT_SB0 | I2S_DIN1 | I2S_DIN3 |
| 10 | GND | GPIO0 | GPIO2 | GND |
| 11 | EXT1+ | EXT_SB1 | GND | EXT3+ |
| 12 | EXT1− | EXT_SB2 | HAT_RST# | EXT3− |
| 13 | GND | GPIO1 | GPIO3 | GND |
| 14 | EXT2+ | RSVD_3A | RSVD_2C | EXT4+ |
| 15 | EXT2− | RSVD_3B | GND | EXT4− |
| 16 | GND | RSVD_2A | RSVD_2B | GND |
| 17 | I2S_MCLK | I2S_DOUT | I2S_FS | I2S_BCLK |

Count by zone:

- **L1 (29):** 6 USB 3 signals + 5 fence `GND` (A1, A4, A7, D1, D4) + `HAT_5V` 3 (B1, B4, B7) + `HAT_3V3` 2 (C1, C2) + power `GND` 4 (B2, B3, B5, B6, under the SuperSpeed pairs) + 6 low-speed signals + 3 reserved.
- **L2 (16):** 8 I2S + 4 GPIO + `HAT_RST#` + 3 reserved.
- **L3 (23):** 10 extension signals + 8 fence `GND` (A10, A13, A16, D10, D13, D16, C11, C15) + 3 sidebands + 2 reserved.

All `GND` contacts are one net; the split into fence and power grounds is a counting convention.

| Group | L1 | L2 | L3 | Total |
|---|---|---|---|---|
| High-speed signals (8 pairs) | 6 | — | 10 | 16 |
| Fence `GND` | 5 | — | 8 | 13 |
| Low-speed signals | 6 | 13 | 3 | 22 |
| `HAT_5V` / `HAT_3V3` / power `GND` | 3 / 2 / 4 | — | — | 3 / 2 / 4 |
| Reserved | 3 | 3 | 2 | 8 (11.8 %) |
| **Total** | **29** | **16** | **23** | **68** |

Spring force (derived): Harwin P70-1000045R, 0.88 N ± 0.29 N at normal working height (drawing P70-10X0045R): 68 × 0.88 N ≈ 60 N (40–80 N) for an L3 piece, 45 × 0.88 N ≈ 40 N (27–53 N) for L2, 29 × 0.88 N ≈ 26 N (17–34 N) for L1. Mill-Max 0965 (115 gf at mid stroke): about 77 N for L3. The HAT retention and the case carry it.

### 7.2 DF40 flat-cable connector (T1; 100 positions, 96 usable)

One Hirose DF40C-100 pair at each end carries signals and power. Numbering follows the Hirose DF40 drawings: contacts 1, 3, 5 … on row A and 2, 4, 6 … on row B, contacts 1 and 2 at the same end (**UNVERIFIED** against drawing EDC3-311351-00; confirm before the FPC layout). The four corner contacts are metal fittings and carry nothing (DF40 catalog 2020, p. 10, note 3). The connector is not polarized (p. 10, note 5): a 180° mating would put `HAT_5V` on signal contacts, so both cable ends are keyed mechanically.

| Row A | Signal | Row B | Signal |
|---|---|---|---|
| 1 | metal fitting | 2 | metal fitting |
| 3–21 (odd) | HAT_5V (×10) | 4–22 (even) | GND (×10) |
| 23–31 (odd) | HAT_3V3 (×5) | 24–32 (even) | GND (×5) |
| 33 | HAT_DET# | 34 | spare |
| 35 | GND | 36 | HAT_INT# |
| 37 | USB3_SSTX+ | 38 | ID_SCL |
| 39 | USB3_SSTX− | 40 | ID_SDA |
| 41 | GND | 42 | I2C_SCL |
| 43 | USB3_SSRX+ | 44 | I2C_SDA |
| 45 | USB3_SSRX− | 46 | RSVD_1A |
| 47 | GND | 48 | RSVD_1B |
| 49 | USB3_D+ | 50 | RSVD_1C |
| 51 | USB3_D− | 52 | spare |
| 53 | GND | 54 | EXT_SB0 |
| 55 | EXT0+ | 56 | EXT_SB1 |
| 57 | EXT0− | 58 | EXT_SB2 |
| 59 | GND | 60 | RSVD_3A |
| 61 | EXT1+ | 62 | RSVD_3B |
| 63 | EXT1− | 64 | spare |
| 65 | GND | 66 | HAT_RST# |
| 67 | EXT2+ | 68 | RSVD_2C |
| 69 | EXT2− | 70 | spare |
| 71 | GND | 72 | GPIO0 |
| 73 | EXT3+ | 74 | GPIO1 |
| 75 | EXT3− | 76 | GPIO2 |
| 77 | GND | 78 | GPIO3 |
| 79 | EXT4+ | 80 | RSVD_2A |
| 81 | EXT4− | 82 | RSVD_2B |
| 83 | GND | 84 | spare |
| 85 | I2S_MCLK | 86 | I2S_DIN0 |
| 87 | GND | 88 | I2S_DIN1 |
| 89 | I2S_BCLK | 90 | I2S_DIN2 |
| 91 | GND | 92 | I2S_DIN3 |
| 93 | I2S_FS | 94 | spare |
| 95 | I2S_DOUT | 96 | spare |
| 97 | spare | 98 | spare |
| 99 | metal fitting | 100 | metal fitting |

Count: power 30 (`HAT_5V` 10, `HAT_3V3` 5, each facing one of 15 `GND`) + `HAT_DET#` + 8 pairs (16) with 9 pair grounds (G S S G in row A) + 21 low-speed + 8 reserved + 2 low-speed grounds = 87; 9 spare (9.4 %). Every reserved pogo contact has a cable position.

## 8. Reserved contacts and spare positions

- **Reserved contacts (HCS-022):** 8 of 68 pogo contacts (11.8 %), spread so that every level keeps at least 8 % of its cumulative field, also after the audio-clock fallback (§10 item 8) takes two at L2 or lower: L1 `RSVD_1A`, `RSVD_1B` (D6, D7), `RSVD_1C` (B8); L2 `RSVD_2A`, `RSVD_2B` (B16, C16, next to the audio clocks, for the fallback) and `RSVD_2C` (C14); L3 `RSVD_3A`, `RSVD_3B` (B14, B15). After the fallback: L2 4 of 45 (8.9 %), L3 6 of 68 (8.8 %). No essential signal was displaced to get them. `RSVD_1C`, `RSVD_2C`, `RSVD_3A` and `RSVD_3B` sit next to pair contacts, so a later assignment there is a static or slow function.
- **Reserved extension lanes:** in PCIe mode `EXT3±` and `EXT4±` are reserved for a second PCIe lane; a later revision may assign them for devices that offer PCIe x2.
- **Rules:** a device leaves reserved contacts unconnected or tied through ≥ 100 kΩ to `GND`; a HAT leaves them unconnected. A later revision of this interface assigns them, inside their level; a HAT declares the interface revision it was built for in its identity record.
- **Spare positions (T1):** the 9 spare positions on the DF40 connector (§7.2) have no pogo contact and no interface function. The board and the device-side piece leave them unconnected. They let a later revision grow, together with a new device-side piece (HCS-023), without a new flat-cable connector.

## 9. Reference HATs on v1.1

Expected use, from the [interface reduction research](../research/2026-10-interface-reduction.md) §6 and the [v1.1 research](../research/2026-10-interface-v1-1-reduction.md) §6. The reference HATs' requirements name the HAT type only; the signal use and level below are engineering judgement, not requirements.

| HAT | Level | Native lines used | USB port | Change against v1.0 | Parts the HAT adds |
|---|---|---|---|---|---|
| Powerbank (PWR-005) | L1 | Power (class F), ID, `HAT_DET#`, `HAT_INT#` (alerts), `I2C_*` (gauge, charger) | Optional: a controls MCU, flashed over USB | Input fault ceiling 3.0 A | None |
| NAS (NAS-005, NAS-006) | L3, PCIe | Extension in PCIe mode (NVMe), `I2C_*` (sensors), `GPIO0` (fan PWM), ID, power class B | USB 3 (for example a 2.5G Ethernet NIC), so NVMe and Ethernet run on two independent links (ADR 0002) | Sidebands renamed `EXT_SB*` | None |
| Camera (CAM-005) | L3, CSI-2 | Extension in CSI-2 mode (4 lanes), `I2C_*` (CCI), `EXT_SB0` (sensor reset), ID; optional I2S for a microphone | Optional: a controls MCU | Reset on `EXT_SB0` instead of `GPIO0`. On-board AI (CAM-004, should) runs on the HAT or on a USB 3 accelerator, not on PCIe beside CSI-2 | An oscillator (24 MHz class) if the sensor module has none |
| Recorder (REC-005) | L2 | I2S port (capture up to 8 channels on 4 lines, `I2S_DOUT` for monitoring), `I2C_*` (codec control), `GPIO`, `HAT_INT#` | Optional: controls as HID; USB Audio as an alternative path with the HAT's own clock | No line out: monitoring through `I2S_DOUT` and the HAT's converter, or the device's own outputs | A monitoring DAC if the HAT has a headphone output |
| Oscilloscope (OSC-005) | L1 | `HAT_INT#`, ID | Sample stream and control from the HAT MCU; flashing and the JTAG path to the FPGA through the MCU or a probe | SWD, if it declares L2, on `GPIO2`/`GPIO3` | A high-speed USB path if the design needs more than full speed, and a JTAG path to the FPGA |
| VNA (VNA-005) | L1 | ID | CDC-ACM, as the NanoVNA does; flashing over USB | None | None |
| Radio (RAD-005) | L1 | ID | HackRF stream (USB generation of the HackRF Pro **UNVERIFIED**); flashing over USB | None | None |

## 10. Open points

1. **Power classes** A, B, F (HCS-015, Draft): current per class against §5, the `HAT_5V` voltage window and the direction handshake.
2. **Identity format** (HCS-017, Draft): byte layout of the interface level, extension function and lane count, interface revision and GPIO alternate-function fields, and HAT+ readability (HCS-024). How a device declares its level, extension functions and supported alternates to Blackbird OS.
3. **Pull-up ownership** on `I2C_*`, `ID_*`, `HAT_INT#`, `HAT_RST#` and the extension sidebands.
4. **High-speed channel budget and fence** (flat cable + DF40 + any selection switch + pogo + HAT pads): DF40 rate rating **UNVERIFIED**; the coupon of ADR 0004 is required before freeze and now also covers the T1's extension switch, D-PHY low-power signalling through it, and the row-based fence of §7.1. Fallback if the coupon requires it: a `GND` next to each pair contact that now holds a static line (B8, B9, B11, B12, B14, B15, C12, C14), with those lines moved to added columns (about 4 × 20 = 80 positions, estimate; research §5.5).
5. **Shared-pair impedance:** 100 Ω differential for both PCIe and CSI-2, confirmed with the FPC stack-up and the coupon.
6. **DF40 contact numbering** (§7.2) against the Hirose drawing.
7. **Interface revision at freeze:** the identity record's interface revision `1` names the pad map frozen at the end of this Draft.
8. **Audio clock domain:** one domain per HAT, on the device's DAC clock, holds only if the first T1 prototype runs the HAT port's transmit and receive from the DAC's clock (ADR 0002). If the test fails, the second domain returns on `RSVD_2A` and `RSVD_2B`; L2 then keeps 4 of 45 reserved (8.9 %).
9. **DLN2 USB ID** used by OpenDAP firmware, and its upstream submission (§6.3).
10. **Capability manifest schema** (HCS-009): how a HAT declares its USB functions, GPIO use, camera sideband use and audio formats.
11. **Conformance suites per level** (ADR 0003, toolkit-verify).

## 11. Sources

- HAT Case Standard requirements: [`../requirements/README.md`](../requirements/README.md); ADR [0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md), [0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md), [0003](../adr/0003-compatibility-levels.md), [0004](../adr/0004-pogo-part-and-field-geometry.md), [0005](../adr/0005-interface-v1-1-shared-extension-and-contact-reduction.md).
- [HAT interface v0](interface-v0.md) (superseded); v1.0 of this document, commit `42fcf14`.
- [v1.1 contact optimisation](../research/2026-10-interface-v1-1-reduction.md); [interface reduction research](../research/2026-10-interface-reduction.md) (§3 USB classes and bridges, §4 justification per removal, §6 reference HATs); [HAT interconnect benchmark](../research/2026-10-hat-interconnect-benchmark.md); [pogo part and geometry research](../research/2026-10-pogo-part-and-geometry.md).
- Board repository: [pin plan §4.3](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md) (T1 mapping), [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md), [BOM v1](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md) (lines 65–70, 102), [power tree v0](https://github.com/opendap-project/board/blob/main/docs/specs/power-tree-v0.md) (§4.6, §4.10).
- Parts library: `Hirose/DF40C-100DS-DP-0.4V.md` (DF40 catalog 2020: 0.3 A per contact, corner metal fittings, sizes), `Harwin/P70-1000045R.md` (drawing P70-10X0045R issue 4: 2.0 A, 0.88 N ± 0.29 N, Ø 2.20 mm pad).
