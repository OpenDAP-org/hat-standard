# HAT interface v1

- **Status:** Draft. Nothing here is built or measured. It supersedes the [HAT interface v0](interface-v0.md), which stays as history. Contact positions may be reordered inside a group after the signal-integrity test coupon (§10).
- **Date:** 2026-10-22
- **Requirements:** HCS-001 to HCS-025, in particular HCS-004 (no cap on high-speed rates), HCS-005 (compromises justified), HCS-006 (one interconnect type), HCS-007 and HCS-017 (identification EEPROM, OpenDAP HAT identity format), HCS-013 (programmable HAT chips), HCS-015 (power classes, Draft), HCS-021 (hardware-agnostic), HCS-022 (reserved contacts), HCS-023 (device-side piece), HCS-024 (compatibility with other standards where cheap) and HCS-025 (compatibility levels).
- **Decisions:** [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md) (all-pogo interface, fixed device-side piece); [ADR 0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md) (interface v1: native high-speed and safety lines, low-speed I/O over USB, one USB port); [ADR 0003](../adr/0003-compatibility-levels.md) (compatibility levels L1–L3 and pogo zones); board ADR 0002 (the signals reach the device-side piece over a flat cable). The justification of each removal (HCS-005) is in the [interface reduction research](../research/2026-10-interface-reduction.md) §4.
- **Conventions:** **UNVERIFIED** = no primary source checked. "Device" is the product the HAT attaches to (the T1 today); "HAT" is the accessory. Directions are written from the device: *out* = device to HAT, *in* = HAT to device.

## 1. Scope

This document defines the HAT interface by **compatibility level**, **function** and **interface class**: which signals exist, what each carries, its electrical class, and its contact position. It names no SoC, SoC ball or SoC peripheral instance (HCS-021). A device implements the interface with whatever silicon it has and documents that mapping in its own repository; for the T1 the mapping is in the board repository ([pin plan §4.3](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md)).

The interface has two stages:

1. the **pogo field** on the device-side piece, against flat pads on the HAT (ADR 0001, HCS-014): up to 96 contacts in a 4 × 24 grid, in three zones (§7.1). This is the interface a HAT designer works with;
2. on the T1, a **flat cable** from the board to the device-side piece, with two Hirose DF40 connector pairs: a 100-position **signal connector** and a 50-position **power connector** (§7.2, §7.3). These positions matter to device and device-side piece designers only. Another device may connect its pogo field differently.

## 2. Compatibility levels

| Level | Functions (adds to the level below) | Zone | Pogo contacts in the zone | Cumulative | Reserved, cumulative |
|---|---|---|---|---|---|
| **L1 Core** | Power by class (`HAT_5V`, `HAT_3V3`, `GND`), presence (`HAT_DET#`), interrupt and wake (`HAT_INT#`), ID EEPROM (`ID_*`), general I2C (`I2C_*`), the USB 3 port with its USB 2 half (`USB3_*`), through which every management function reaches the device (§6) | Core zone: columns 17–24, all rows | 32 | 32 | 3 (9.4 %) |
| **L2 Control and audio** | Digital audio on the device's DAC clock domain (`I2S_*`), line out (`LINE_OUT_*`, `AGND`), four fast GPIO (`GPIO0`–`GPIO3`), SWD and reset (`DBG_*`, `HAT_RST#`) | L2 zone: row C odd columns 1–15, row D columns 1–16 | 24 | 56 | 5 (8.9 %) |
| **L3 High speed** | PCIe x1 with its sidebands (`PCIE_*`), MIPI CSI-2 4-lane (`CSI_*`) | L3 zone: rows A–B columns 1–16, row C even columns 2–16 | 40 | 96 | 10 (10.4 %) |

- **One field, one pinout:** the three levels share one pogo field and one pinout. Each level is a nested subset of the same fixed positions; there is no connector or pinout per level. An L1 HAT on an L3 device uses only the core zone; an L3 HAT on an L1 device finds its L1 contacts in the same place.
- **Compatibility rule (ADR 0003):** a HAT works on any device of its level or higher. A device that receives a HAT needing a higher level says so clearly through Blackbird OS (T1-045: never silent), and powers nothing it cannot support safely. The ID EEPROM is in L1, so every device can read the level a HAT needs before it enables `HAT_5V`.
- **Declaration:** a device declares the level it implements. A HAT declares the level it needs in its identity record (HCS-017): the record carries an **interface level** field (`1`, `2` or `3`) next to the interface revision (`1` for this document). The byte layout is defined with the OpenDAP HAT identity format, which is not specified yet (§10).
- **Device-side piece:** a device of level N populates the zones up to N. An L1 device-side piece carries only the core zone (32 contacts). An L2 piece carries 56. Unpopulated positions keep the same grid, so the core zone sits at the same place on every piece.
- **HAT side:** a HAT carries pads for the zones up to its level. Under the zones above its level it has no exposed copper, so the pogo contacts of a higher-level device touch nothing.
- **Conformance:** each level has its own conformance test suite (toolkit-verify), as ADR 0003 sets out; the suites are not written yet.
- **The T1 implements L3.**

## 3. Changes from v0

| Item | v0 | v1 | Path in v1 |
|---|---|---|---|
| Levels | none | L1, L2, L3 with pogo zones | §2 |
| Second USB 2.0 port (`USB2B_D±`) | 1 pair | removed | Every USB function uses the one USB port (`USB3_*`); a HAT that needs a SuperSpeed device and other USB functions carries a USB 3 hub (§6) |
| UART (`UART_TX`, `UART_RX`, `UART_RTS#`, `UART_CTS#`) | 4 | removed | USB: CDC-ACM or a USB-UART bridge (§6) |
| SPI (`SPI_SCLK`, `SPI_MOSI`, `SPI_MISO`, `SPI_CS0#`, `SPI_CS1#`) | 5 | removed | USB: DLN2 bridge, or the device's USB variant (§6) |
| Slow GPIO (`GPIO4`–`GPIO7`) | 4 | removed | USB: DLN2 or a HID GPIO bridge (§6) |
| JTAG data (`DBG_TDI`, `DBG_TDO`) | 2 | removed | USB: CMSIS-DAP or FTDI-class probe on the HAT, or configuration from the HAT MCU |
| Debug | `DBG_TCK`, `DBG_TMS`, `DBG_RESET#` | `DBG_SWCLK`, `DBG_SWDIO`, `HAT_RST#` | Two-wire SWD and an open-drain reset of the HAT's main controller |
| Camera reset (`CAM_RST#`) | 1 | removed | `GPIO0`–`GPIO3`, declared in the manifest (a camera HAT is L3, so it has L2's GPIO) |
| Camera clock (`CAM_MCLK`) | 1 | removed | An oscillator on the HAT |
| Digital audio | I2S with either side as clock provider (5) + TDM in on a HAT clock (6): two clock domains | one port of 8 lines on the device's DAC clock domain | §4.2 |
| Line in (`LINE_IN_L`, `LINE_IN_R`) | 2 | removed | An ADC on the HAT, through the audio port or USB Audio |
| GPIO alternate functions | none | UART or SPI on `GPIO0`–`GPIO3`, declared by the HAT (should) | §4.3 |
| Reserved | 7 (2 signal + 5 power connector) | 10 (5 + 5), spread over the levels | §8 |
| Pogo field | not defined (estimate ≈ 122) | 96 contacts in zones, positions fixed | §7.1 |
| DF40 signal connector | 96 of 96 used | 72 used, 24 spare | §7.2 |
| DF40 power connector | 46 of 46 | unchanged | §7.3 |

## 4. Interface by function

### 4.1 Contact list

| Level | Function | Signals (direction) | Signal contacts | Electrical class | Notes |
|---|---|---|---|---|---|
| L1 | Power | `HAT_5V`, `HAT_3V3`, `GND` | — | §5 | `HAT_5V` is bidirectional: out for HATs that draw power, in for a HAT that supplies the device (HCS-015, Draft) |
| L1 | Presence | `HAT_DET#` (in) | 1 | 3.3 V, active low; the HAT ties it to `GND`; pull-up on the device | On the T1 it sits on the DF40 power connector, so it also proves that connector is mated |
| L1 | Interrupt and wake | `HAT_INT#` (in) | 1 | 3.3 V, active low, open drain on the HAT, pull-up on the device | Wake source. Also the interrupt a HAT may assert for events behind its USB functions (§6.4) |
| L1 | Identification | `ID_SCL`, `ID_SDA` | 2 | I2C, 3.3 V, standard or fast mode | Dedicated to the HAT EEPROM at 7-bit address 0x50 (HCS-007). No other device on this bus. Powered from `HAT_3V3`. It holds the **OpenDAP HAT identity format** (HCS-017, Draft: reverse-domain ID, power class, interface level, Ed25519 signature, capability manifest hash; this document adds the interface revision and the GPIO alternate functions, §4.3), whose layout is inspired by Raspberry Pi HAT+; staying readable by HAT+ tools is a *should* (HCS-024). Read before `HAT_5V` is enabled, so the power class and level are known before power is applied |
| L1 | I2C | `I2C_SCL`, `I2C_SDA` | 2 | I2C, 3.3 V | General-purpose bus for kernel devices described in the device tree, and the camera control (CCI) bus. HAT designers keep addresses distinct |
| L1 | USB 3 | `USB3_SSTX±` (out), `USB3_SSRX±` (in), `USB3_D±` (USB 2.0 pair of the same port) | 6 (3 pairs) | USB 3 SuperSpeed + USB 2.0, 90 Ω differential | The only USB port to the HAT. Device is host. TX AC-coupling capacitors on the transmitting side (device for `SSTX`, HAT for `SSRX`). Management functions use it (§6) |
| L2 | Digital audio | `I2S_MCLK`, `I2S_BCLK`, `I2S_FS`, `I2S_DOUT` (all out), `I2S_DIN0`–`I2S_DIN3` (in) | 8 | 3.3 V push-pull | One full-duplex port on the device's DAC clock domain (§4.2) |
| L2 | Analog audio | `LINE_OUT_L`, `LINE_OUT_R` (out); `AGND` ×2 | 4 | Line level, referenced to `AGND` | `AGND` is the return of the line output only and joins `GND` at one point on the device. Levels and impedances open (§10) |
| L2 | GPIO | `GPIO0`–`GPIO3` | 4 | 3.3 V | Every line is PWM-capable as an output and interrupt-capable as an input. A device that cannot offer both on all four lines does not meet L2 |
| L2 | Debug and recovery | `DBG_SWCLK` (out), `DBG_SWDIO` (bidirectional), `HAT_RST#` (out) | 3 | 3.3 V; `HAT_RST#` open drain, active low | Two-wire SWD to the HAT's main controller and any SWD target the HAT routes to it. `HAT_RST#` resets the HAT's main controller. They reach a bricked USB bridge MCU too (HCS-013) |
| L3 | PCIe | `PCIE_TX±` (out), `PCIE_RX±` (in), `PCIE_REFCLK±` (out) | 6 (3 pairs) | PCIe x1, 85–100 Ω differential (**UNVERIFIED** target impedance, set with the FPC stack-up) | Device is root complex and sources the 100 MHz reference clock. TX AC-coupling capacitors on the transmitting side |
| L3 | PCIe sidebands | `PCIE_PERST#` (out), `PCIE_CLKREQ#` (in), `PCIE_WAKE#` (in) | 3 | 3.3 V | The device holds `PCIE_PERST#` asserted until HAT power is good |
| L3 | Camera, MIPI CSI-2 | `CSI_CLK±`, `CSI_D0±`–`CSI_D3±` (all in) | 10 (5 pairs) | MIPI D-PHY, 100 Ω differential | One clock lane and up to four data lanes. Camera control on `I2C_*`, sensor reset on a `GPIO` |
| L1, L3 | High-speed ground fence | `GND` | — | — | 8 pogo contacts in L1, 16 in L3 (§7.1) |
| L2 | Low-speed grounds | `GND` | — | — | 3 pogo contacts |
| all | Reserved | `RSVD_S0`–`RSVD_S4`, `RSVD_P0`–`RSVD_P4` | 10 | — | §8 |

Native low-speed signals: 24, including `HAT_DET#` (v0: 44): 6 in L1, 15 in L2, 3 in L3.

### 4.2 Electrical levels and line rules

- **Logic levels.** Every low-speed signal at the interface is 3.3 V logic. A device whose controllers run at another level translates on its own side; so does a HAT. Every 3.3 V signal on the device side is either isolated or in a defined state while `HAT_DET#` is high (no HAT), as in v0.
- **Open-drain lines.** `HAT_DET#`, `HAT_INT#` and `HAT_RST#` are active low. `HAT_INT#` is driven by the HAT, `HAT_RST#` by the device, both open drain. Pull-up ownership on `HAT_RST#`, `I2C_*`, `ID_*` and the PCIe sidebands is open (§10).
- **GPIO.** `GPIO0`–`GPIO3` cover timestamped pulses (for example a GNSS PPS), resets of device-tree-described HAT devices (a camera sensor, a modem), PWM and boot straps. Their use is declared in the capability manifest (HCS-009).
- **Digital audio.** The port runs on the device's DAC clock domain, the clock of the device's own playback (ADR 0002, ADR 0003): the device provides `I2S_MCLK`, `I2S_BCLK` and `I2S_FS`, and the HAT is the clock consumer. `I2S_DOUT` and `I2S_DIN0`–`I2S_DIN3` share those clocks, so HAT audio is sample-synchronous with the device's playback. Four input lines carry up to 8 channels as 4 × 2 channels, for example 8 channels at 192 kHz with `I2S_BCLK` at 12.288 MHz. A device declares the formats and rates it supports (TDM included) in its own manifest. A HAT that needs its own audio clock uses USB Audio (asynchronous, the HAT clock is the master). The one-domain rule is conditional on the first T1 prototype (ADR 0002; §10).
- **Debug.** `DBG_SWCLK` and `DBG_SWDIO` follow the Arm Serial Wire Debug two-wire convention. The SWD clock rate is set by the device and is not specified here.
- **High-speed rates.** The interface sets the lane count and the electrical class, not a rate (HCS-004). Each link runs at the highest rate both sides and the channel support; the device and the HAT declare their rates in the capability manifest (HCS-009). Rates are measured on hardware at the end.

### 4.3 Alternate functions on the GPIO lines (should)

`GPIO0`–`GPIO3` **should** be able to carry the alternate functions below, configured by the device's pin multiplexer with no extra hardware. A HAT that wants one declares it in its identity record, per line. Every other contact keeps a fixed function: high-speed pairs, power, ground, `HAT_DET#`, `ID_*` and all the rest of §4.1.

| Alternate | `GPIO0` | `GPIO1` | `GPIO2` | `GPIO3` |
|---|---|---|---|---|
| UART A | TX (out) | RX (in) | — | — |
| UART B | — | — | TX (out) | RX (in) |
| SPI controller | SCLK (out) | MOSI (out) | MISO (in) | CS0# (out) |

- **Device support is optional.** A device declares which alternates it supports. A device that supports none stays compatible: it treats the lines as plain GPIO. A HAT function that depends on an alternate the device lacks is then unavailable or reached another way (for example over USB), and Blackbird OS says so (T1-045).
- **Levels and defaults:** the alternates belong to L2. Each line starts as a GPIO input after power-up and changes function only after the device has read the identity record.
- **T1:** the GPIO balls offer UART A and UART B; SPI is not available on them (board pin plan §4.3).

### 4.4 Counts

| | Pogo L1 | Pogo L2 | Pogo L3 | **Pogo total** | DF40 signal connector | DF40 power connector |
|---|---|---|---|---|---|---|
| High-speed signals (11 pairs) | 6 | — | 16 | **22** | 22 | — |
| High-speed ground fence | 8 | — | 16 | **24** | 13 | — |
| Low-speed signals | 6 | 15 | 3 | **24** | 23 | 1 (`HAT_DET#`) |
| Analog (line out L/R + 2 `AGND`) | — | 4 | — | **4** | 4 | — |
| Low-speed grounds | — | 3 | — | **3** | 5 | — |
| `HAT_5V` / `HAT_3V3` / power `GND` | 3 / 2 / 4 | — | — | **3 / 2 / 4** | — | 12 / 6 / 22 |
| Reserved | 3 | 2 | 5 | **10 (10.4 %)** | 5 | 5 |
| **Assigned** | **32** | **24** | **40** | **96** | **72 of 96 usable** | **46 of 46 usable** |
| Spare (no interface function) | — | — | — | 0 | 24 | 0 |

ADR 0002 estimated about 93 pogo contacts (7 reserved, about 22 fence grounds, 5 low-speed grounds). The zoned grid gives 96: 10 reserved, so that every level keeps at least 8 % (ADR 0002, ADR 0003); 24 fence grounds, because the L1 band must be fenced on its own; and 3 low-speed grounds, because the L1 lines sit next to power returns.

## 5. Power and ground

Ratings of the interface itself, all in L1. Device budgets (how much a given device supplies on battery or on external power) are set in that device's documents and declared to the HAT; the HAT power classes (HCS-015, Draft) are written against them. Power is unchanged from v0 §3, and the pogo counts are now fixed.

| Rail | Nominal | Direction | Interface rating (continuous) | Interface fault ceiling | Pogo contacts | DF40 power contacts (T1) |
|---|---|---|---|---|---|---|
| `HAT_5V` | 5 V (window **UNVERIFIED**, set with the power classes) | out (classes A/B) or in (class F) | 2.0 A either direction | 3.6 A | 3 | 12 |
| `HAT_3V3` | 3.3 V | out | 1.0 A | 1.8 A | 2 | 6 |
| `GND` (power return) | — | return | 3.0 A | 5.4 A | 4 | 22 |

Margin rule (board BRD-049): each contact at or below **70 % of its rating in continuous use** with all contacts loaded, and at or below **100 % at the worst-case fault ceiling** of the device's current limiters. The fault ceilings come from the DF40 contacts (12 × 0.3 A and 6 × 0.3 A), as in v0; the `GND` ceiling is the sum of both rails.

| Contact | Rating | Rail | Continuous per contact | With one contact open | At the fault ceiling | Ceiling, one contact open |
|---|---|---|---|---|---|---|
| Pogo (Harwin P70-1000045R drawing) | 2.0 A | `HAT_5V` ×3 | 0.67 A (33 %) | 1.0 A (50 %) | 1.2 A (60 %) | 1.8 A (90 %) |
| | | `HAT_3V3` ×2 | 0.50 A (25 %) | 1.0 A (50 %) | 0.9 A (45 %) | 1.8 A (90 %) |
| | | `GND` ×4 | 0.75 A (38 %) | 1.0 A (50 %) | 1.35 A (68 %) | 1.8 A (90 %) |
| DF40 (Hirose catalog 2020, p. 2) | 0.3 A | `HAT_5V` ×12 | 0.17 A (56 %) | — | 0.30 A (100 %) | — |
| | | `HAT_3V3` ×6 | 0.17 A (56 %) | — | 0.30 A (100 %) | — |
| | | `GND` ×22 | 0.14 A (45 %) | — | 0.25 A (82 %) | — |

- **Pogo count rule (v0 §3):** the fewest contacts that stay at or below 70 % of the pogo rating at the interface rating, plus one for redundancy: `HAT_5V` 2 + 1, `HAT_3V3` 1 + 1, `GND` 3 + 1. Each rail stays inside the rule with one contact open. The fence and low-speed grounds are not counted as power returns. If another pogo part is qualified (§10), the counts are re-checked against its rating.
- **Device duties.** The device limits each rail it sources with a current limiter whose maximum trip current stays inside the fault ceiling, and limits current drawn from a class F HAT on `HAT_5V` the same way. It enables `HAT_5V` only after reading the HAT EEPROM, and only for a HAT whose level and power class it supports. A device may set lower limits; the T1's are in the board [power tree §4.6](https://github.com/opendap-project/board/blob/main/docs/specs/power-tree-v0.md).
- **HAT duties.** The HAT powers its EEPROM from `HAT_3V3`. Bulk capacitance on each rail stays at or below 470 µF (board power tree §4.6; **Draft**). USB devices on a HAT are self-powered and take their VBUS-sense input from `HAT_5V` or `HAT_3V3`: the interface has no VBUS contact.
- **Flat cable (T1).** Its power and ground conductors are sized for the interface rating (board BOM line 102).

## 6. USB port and management functions

### 6.1 One USB port

The HAT has one USB port, in L1: `USB3_SSTX±`, `USB3_SSRX±` and `USB3_D±`, the SuperSpeed lanes and the USB 2.0 pair of the same port. Low-speed functions that v1 removed from the interface (§3) reach the device through this port as USB devices on the HAT. This document calls them **management functions**.

- **Simple HATs** connect one USB device (a bridge, an MCU, a probe) to `USB3_D±`. A device with several functions presents them as one composite device (for example DLN2 + CDC-ACM + CMSIS-DAP from one MCU), which needs no hub.
- **HATs that need a SuperSpeed device and management functions at the same time** carry a standard USB 3 hub (USB hub class, mainline Linux `hub` driver) and connect both behind it. Mainline `onboard_usb_dev` handles the reset and supply of several hubs (driver table, **UNVERIFIED** per hub part).
- **USB-native HATs** (for example a NanoVNA- or HackRF-based design) use the same port for their main data.
- **Power:** USB devices on a HAT are self-powered (§5).
- **Availability:** on the T1 the SuperSpeed lanes can be switched to the main USB-C port when the HAT does not declare SuperSpeed use; `USB3_D±` stays with the HAT ([board high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md) §3).
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

### 7.1 Pogo field (96 contacts, 4 × 24, three zones)

Positions are given looking at the pogo tips on the device-side piece: rows A to D, row A nearest the flat-cable tail; columns 1 to 24. The HAT pad map is the mirror image of this table. Every populated position uses the same pogo part (HCS-006). The pitch is open (§10).

| Zone | Positions | Contacts |
|---|---|---|
| L1 core | columns 17–24, rows A–D | 32 |
| L2 | row C, odd columns 1–15; row D, columns 1–16 | 24 |
| L3 | rows A–B, columns 1–16; row C, even columns 2–16 | 40 |

Layout rules:

- **High-speed band (rows A and B, columns 1–23):** each pair stands in one column, positive contact in row A and negative contact in row B. Odd columns are ground columns, so every pair has a ground column on each side. The L1 band (USB 3, columns 17–23) is fenced on its own, so an L1 piece needs no L3 contact.
- **Row C under a pair** (even columns 2–22) carries only static or slow lines: presence, the ID bus, the PCIe sidebands and reserved contacts. A later revision that assigns a reserved contact there assigns a static or slow function. Whether row C needs grounds under the pairs is decided by the coupon (§10).
- **Power** at the L1 end: `HAT_5V` in column 24, `HAT_3V3` at D21–D22, each next to `GND`.
- **Analog** at the end of row D farthest from the core zone, each line next to an `AGND` or `GND`, with reserved contacts above it and a static line (`HAT_RST#`) between it and the I2S clocks.

| Col | Row A | Row B | Row C | Row D |
|---|---|---|---|---|
| 1 | GND | GND | RSVD_P3 | AGND |
| 2 | CSI_D3+ | CSI_D3− | RSVD_S0 | LINE_OUT_L |
| 3 | GND | GND | RSVD_P4 | AGND |
| 4 | CSI_D2+ | CSI_D2− | RSVD_S1 | LINE_OUT_R |
| 5 | GND | GND | GPIO0 | GND |
| 6 | CSI_D1+ | CSI_D1− | RSVD_S2 | HAT_RST# |
| 7 | GND | GND | GPIO1 | I2S_MCLK |
| 8 | CSI_D0+ | CSI_D0− | RSVD_S3 | GND |
| 9 | GND | GND | GPIO2 | I2S_BCLK |
| 10 | CSI_CLK+ | CSI_CLK− | RSVD_S4 | I2S_FS |
| 11 | GND | GND | GPIO3 | I2S_DOUT |
| 12 | PCIE_REFCLK+ | PCIE_REFCLK− | PCIE_PERST# | I2S_DIN0 |
| 13 | GND | GND | DBG_SWCLK | I2S_DIN1 |
| 14 | PCIE_RX+ | PCIE_RX− | PCIE_CLKREQ# | I2S_DIN2 |
| 15 | GND | GND | DBG_SWDIO | I2S_DIN3 |
| 16 | PCIE_TX+ | PCIE_TX− | PCIE_WAKE# | GND |
| 17 | GND | GND | HAT_INT# | RSVD_P0 |
| 18 | USB3_D+ | USB3_D− | HAT_DET# | RSVD_P1 |
| 19 | GND | GND | I2C_SCL | RSVD_P2 |
| 20 | USB3_SSRX+ | USB3_SSRX− | ID_SCL | GND |
| 21 | GND | GND | I2C_SDA | HAT_3V3 |
| 22 | USB3_SSTX+ | USB3_SSTX− | ID_SDA | HAT_3V3 |
| 23 | GND | GND | GND | GND |
| 24 | HAT_5V | HAT_5V | HAT_5V | GND |

Count by zone:

- **L1 (32):** 6 USB 3 signals + 8 fence `GND` (columns 17, 19, 21, 23, rows A–B) + `HAT_5V` 3 + `HAT_3V3` 2 + power `GND` 4 (C23, D20, D23, D24) + 6 low-speed signals + 3 reserved.
- **L2 (24):** 8 I2S + 4 GPIO + 3 debug and reset + 4 analog + 3 low-speed `GND` (D5, D8, D16) + 2 reserved.
- **L3 (40):** 16 PCIe and CSI-2 signals + 16 fence `GND` (columns 1–15 odd, rows A–B) + 3 PCIe sidebands + 5 reserved.

All `GND` contacts are one net; the split into fence, power and low-speed grounds is a counting convention.

Spring force with the P70-1000045R (0.88 N ± 0.29 N at normal working height, drawing P70-10X0045R): 96 × 0.88 N ≈ 84 N (57–112 N) for an L3 piece, 32 × 0.88 N ≈ 28 N (19–37 N) for an L1 piece. The HAT retention and the case carry it.

### 7.2 DF40 signal connector (T1; 100 positions, 96 usable)

Numbering follows the Hirose DF40 drawings: contacts 1, 3, 5 … on row A and 2, 4, 6 … on row B, contacts 1 and 2 at the same end (**UNVERIFIED** against drawing EDC3-311351-00; confirm before the FPC layout). The four corner contacts are metal hold-downs and carry nothing (DF40 catalog p. 10, note 3). Every signal kept from v0 keeps its v0 position, so the flat cable changes only where v1 removed or added a signal. "spare" marks a freed position (§8).

| Row A | Signal | Row B | Signal |
|---|---|---|---|
| 1 | hold-down | 2 | hold-down |
| 3 | GND | 4 | GND |
| 5 | USB3_SSTX+ | 6 | ID_SCL |
| 7 | USB3_SSTX− | 8 | ID_SDA |
| 9 | GND | 10 | HAT_INT# |
| 11 | USB3_SSRX+ | 12 | I2C_SCL |
| 13 | USB3_SSRX− | 14 | I2C_SDA |
| 15 | GND | 16 | spare |
| 17 | USB3_D+ | 18 | spare |
| 19 | USB3_D− | 20 | spare |
| 21 | GND | 22 | spare |
| 23 | spare | 24 | GND |
| 25 | spare | 26 | DBG_SWCLK |
| 27 | GND | 28 | DBG_SWDIO |
| 29 | PCIE_TX+ | 30 | spare |
| 31 | PCIE_TX− | 32 | spare |
| 33 | GND | 34 | HAT_RST# |
| 35 | PCIE_RX+ | 36 | PCIE_PERST# |
| 37 | PCIE_RX− | 38 | PCIE_CLKREQ# |
| 39 | GND | 40 | PCIE_WAKE# |
| 41 | PCIE_REFCLK+ | 42 | spare |
| 43 | PCIE_REFCLK− | 44 | GPIO0 |
| 45 | GND | 46 | GPIO1 |
| 47 | CSI_CLK+ | 48 | GPIO2 |
| 49 | CSI_CLK− | 50 | GPIO3 |
| 51 | GND | 52 | spare |
| 53 | CSI_D0+ | 54 | spare |
| 55 | CSI_D0− | 56 | spare |
| 57 | GND | 58 | spare |
| 59 | CSI_D1+ | 60 | RSVD_S0 |
| 61 | CSI_D1− | 62 | RSVD_S1 |
| 63 | GND | 64 | RSVD_S2 |
| 65 | CSI_D2+ | 66 | RSVD_S3 |
| 67 | CSI_D2− | 68 | RSVD_S4 |
| 69 | GND | 70 | spare |
| 71 | CSI_D3+ | 72 | spare |
| 73 | CSI_D3− | 74 | spare |
| 75 | GND | 76 | spare |
| 77 | spare | 78 | spare |
| 79 | spare | 80 | GND |
| 81 | spare | 82 | I2S_MCLK |
| 83 | I2S_DIN0 | 84 | I2S_BCLK |
| 85 | I2S_DIN1 | 86 | I2S_FS |
| 87 | I2S_DIN2 | 88 | I2S_DOUT |
| 89 | I2S_DIN3 | 90 | spare |
| 91 | GND | 92 | GND |
| 93 | spare | 94 | LINE_OUT_L |
| 95 | AGND | 96 | AGND |
| 97 | spare | 98 | LINE_OUT_R |
| 99 | hold-down | 100 | hold-down |

Count: 11 pairs (22) + 13 pair grounds + 23 low-speed + 4 analog (including 2 `AGND`) + 5 low-speed grounds + 5 reserved = 72; 24 spare. The two grounds that framed the removed `USB2B_D±` pair (21, 27) stay, so every pair keeps a ground on each side in the row; the pair's positions (23, 25) are spare.

Position changes against v0: `DBG_TCK` → `DBG_SWCLK` (26), `DBG_TMS` → `DBG_SWDIO` (28), `DBG_RESET#` → `HAT_RST#` (34), `TDM_DIN0`–`TDM_DIN3` → `I2S_DIN0`–`I2S_DIN3` (83–89), `SPI_CS1#` → `RSVD_S2` (64), `SPI_CS0#` → `RSVD_S3` (66), v0 ground 68 → `RSVD_S4`. The v0 low-speed grounds at 76 and 77 are spare.

### 7.3 DF40 power connector (T1; 50 positions, 46 usable)

Unchanged from v0. Every power contact faces a ground contact in the other row.

| Row A | Signal | Row B | Signal |
|---|---|---|---|
| 1 | hold-down | 2 | hold-down |
| 3 | HAT_DET# | 4 | GND |
| 5 | RSVD_P0 | 6 | GND |
| 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29 | HAT_5V (×12) | 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30 | GND (×12) |
| 31 | RSVD_P1 | 32 | GND |
| 33, 35, 37, 39, 41, 43 | HAT_3V3 (×6) | 34, 36, 38, 40, 42, 44 | GND (×6) |
| 45 | RSVD_P2 | 46 | GND |
| 47 | RSVD_P3 | 48 | RSVD_P4 |
| 49 | hold-down | 50 | hold-down |

Count: `HAT_5V` 12 + `HAT_3V3` 6 + `GND` 22 + `HAT_DET#` 1 + reserved 5 = 46.

## 8. Reserved contacts and spare positions

- **Reserved contacts (HCS-022):** 10 of 96 pogo contacts (10.4 %), spread so that every level keeps at least 8 % of its cumulative field (ADR 0002, ADR 0003): L1 `RSVD_P0`–`RSVD_P2` (D17–D19); L2 `RSVD_P3`, `RSVD_P4` (C1, C3); L3 `RSVD_S0`–`RSVD_S4` (C2–C10, even). No essential signal was displaced to get them. On the T1 each has a flat-cable position: `RSVD_S0`–`RSVD_S4` on the signal connector (60–68, even), `RSVD_P0`–`RSVD_P4` on the power connector (5, 31, 45, 47, 48, low-speed only). The L3 reserved contacts sit under high-speed pairs, so a later assignment there is a static or slow function.
- **Rules:** a device leaves reserved contacts unconnected or tied through ≥ 100 kΩ to `GND`; a HAT leaves them unconnected. A later revision of this interface assigns them, inside their level; a HAT declares the interface revision it was built for in its identity record.
- **Spare positions (T1):** the 24 spare positions on the DF40 signal connector (§7.2) have no pogo contact and no interface function. The board and the device-side piece leave them unconnected. They let a later revision grow, together with a new device-side piece (HCS-023), without a new flat-cable connector (ADR 0002).

## 9. Reference HATs on v1

Expected use, from the [interface reduction research](../research/2026-10-interface-reduction.md) §6, updated for the single USB port and the levels. The reference HATs' requirements name the HAT type only; the signal use and level below are engineering judgement, not requirements. Levels follow the typical HATs of ADR 0003.

| HAT | Level | Native lines used | USB port | Change against v0 | Parts the HAT adds |
|---|---|---|---|---|---|
| Powerbank (PWR-005) | L1 | Power (class F), ID, `HAT_DET#`, `HAT_INT#` (alerts), `I2C_*` (gauge, charger) | Optional: a controls MCU, flashed over USB | No GPIO or SWD at L1 | None |
| NAS (NAS-005, NAS-006) | L3 | PCIe and sidebands (NVMe), `I2C_*` (sensors), `GPIO0` (fan PWM), ID, power class B | USB 3 (for example a 2.5G Ethernet NIC), so NVMe and Ethernet run on two independent links (ADR 0002) | None | None |
| Camera (CAM-005) | L3 | CSI-2 4 lanes, `I2C_*` (CCI), `GPIO0` (sensor reset), ID; optional I2S for a microphone | Optional: a controls MCU | `CAM_RST#` → `GPIO0`; `CAM_MCLK` gone | An oscillator (24 MHz class) if the sensor module has none |
| Recorder (REC-005) | L2 | I2S port (capture up to 8 channels on 4 lines, playback for monitoring), `I2C_*` (codec control), `GPIO`, `HAT_INT#`, optional line out | Optional: controls as HID; USB Audio as an alternative path with the HAT's own clock | The ADCs run as clock consumers of the device's DAC clock domain | None, if consumer mode is enough |
| Oscilloscope (OSC-005) | L1 | `HAT_INT#`, ID | Sample stream and control from the HAT MCU; flashing and the JTAG path to the FPGA through the MCU or a probe | SPI, UART and JTAG gone; SWD only if it declares L2 | A high-speed USB path if the design needs more than full speed, and a JTAG path to the FPGA |
| VNA (VNA-005) | L1 | ID | CDC-ACM, as the NanoVNA does; flashing over USB | None in practice | None |
| Radio (RAD-005) | L1 | ID | HackRF stream (USB generation of the HackRF Pro **UNVERIFIED**); flashing over USB | None in practice | None |

No reference HAT needs the removed second USB 2.0 port: each has at most one USB device, or puts a hub on the HAT.

## 10. Open points

1. **Power classes** A, B, F (HCS-015, Draft): current per class against §5, the `HAT_5V` voltage window and the direction handshake.
2. **Identity format** (HCS-017, Draft): byte layout of the interface level, interface revision and GPIO alternate-function fields, and HAT+ readability (HCS-024). How a device declares its level and supported alternates to Blackbird OS.
3. **Line output:** full-scale level, source and load impedance (v0 open point 2, output side; the input side is closed by its removal).
4. **Pull-up ownership** on `I2C_*`, `ID_*`, `HAT_INT#`, `HAT_RST#` and the PCIe sidebands.
5. **High-speed channel budget** (flat cable + DF40 + pogo + HAT pads): DF40 rate rating **UNVERIFIED**; a test coupon is required before freeze (board BOM line 66). The coupon also decides the fence pattern of §7.1 (vertical pairs, row C under the pairs without grounds) and whether some spare DF40 positions become grounds.
6. **DF40 contact numbering** (§7.2) against the Hirose drawing.
7. **Pogo pitch and part.** The research assumed a 1.27 mm pitch (4 × 24 ≈ 29 × 3.8 mm of contact centres). The pin on the board BOM (Harwin P70-1000045R, line 67) has a Ø 2.00 mm flange and a Ø 2.20 mm recommended pad, so it cannot sit on a 1.27 mm grid; at a pitch that fits it (for example 2.54 mm) the field is about 58 × 7.6 mm. The pitch is set when the pogo part is qualified (benchmark proposal 1: qualify two vendors against one envelope).
8. **Audio clock domain:** one domain per HAT, on the device's DAC clock, holds only if the first T1 prototype runs the HAT port's transmit and receive from the DAC's clock (ADR 0002). If the test fails, the second domain returns on reserved contacts. Two of the five L2-or-lower reserved contacts would leave L2 at 3 of 56 (5.4 %), below 8 %, unless the field grows.
9. **DLN2 USB ID** used by OpenDAP firmware, and its upstream submission (§6.3).
10. **Capability manifest schema** (HCS-009): how a HAT declares its USB functions, GPIO use and audio formats.
11. **Conformance suites per level** (ADR 0003, toolkit-verify).

## 11. Sources

- HAT Case Standard requirements: [`../requirements/README.md`](../requirements/README.md); [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md); [ADR 0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md); [ADR 0003](../adr/0003-compatibility-levels.md).
- [HAT interface v0](interface-v0.md) (superseded).
- [Interface reduction research](../research/2026-10-interface-reduction.md) (§3 USB classes and bridges, §4 justification per removal, §6 reference HATs); [HAT interconnect benchmark](../research/2026-10-hat-interconnect-benchmark.md) (pogo grid and validation plan).
- Board repository: [pin plan §4.3](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md) (T1 mapping), [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md), [BOM v1](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md) (lines 66–69, 102), [power tree v0](https://github.com/opendap-project/board/blob/main/docs/specs/power-tree-v0.md) (§4.6, §4.10).
- Parts library: `Hirose/DF40C-100DS-DP-0.4V.md` (DF40 catalog 2020: 0.3 A per contact, corner hold-downs), `Harwin/P70-1000045R.md` (drawing P70-10X0045R issue 4: 2.0 A, 0.88 N ± 0.29 N, Ø 2.20 mm pad).
