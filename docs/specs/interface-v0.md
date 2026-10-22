# HAT interface v0

> **Superseded by the [HAT interface v1](interface-v1.md) (2026-10-22, [ADR 0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md), [ADR 0003](../adr/0003-compatibility-levels.md)).** Kept as history; do not design against it.

- **Status:** Superseded (was Draft). Nothing here is built or measured. Contact positions follow the current board allocation (board BOM v1.2, line 66) and may be reordered inside a group after the flat-cable signal-integrity test coupon.
- **Date:** 2026-10-22
- **Requirements:** HCS-001 to HCS-022, in particular HCS-004 (no cap on high-speed rates), HCS-006 (one interconnect type), HCS-007 (identification EEPROM), HCS-021 (hardware-agnostic) and HCS-022 (reserved contacts).
- **Decisions:** [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md) (all-pogo interface, fixed device-side piece); board ADR 0002 (the signals reach the device-side piece over a flat cable).
- **Conventions:** **UNVERIFIED** = no primary source checked. "Device" is the product the HAT attaches to (the T1 today); "HAT" is the accessory. Directions are written from the device: *out* = device to HAT, *in* = HAT to device.

## 1. Scope

This document defines the HAT interface by **function and interface class**: which signals exist, what each carries, its electrical class, and its contact position. It names no SoC, SoC ball or SoC peripheral instance. A device implements the interface with whatever silicon it has and documents that mapping in its own repository; for the T1 the mapping is in the board repository ([pin plan §4.3](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md)). A device change, including a different SoC, keeps this interface valid (HCS-021).

The interface has two stages (requirements, Context):

1. a **flat cable** from the device board to the device-side piece, with two Hirose DF40 connector pairs: a 100-contact **signal connector** and a 50-contact **power connector** (§4);
2. the **pogo field** on the device-side piece, against flat pads on the HAT (ADR 0001, HCS-014). Its grid and contact positions are not defined yet (§6).

The contact positions in §4 are those of the flat-cable connectors. They define the functional order that the pogo field will follow.

## 2. Interface classes

| Class | Signals | Electrical class | Notes |
|---|---|---|---|
| Power | `HAT_5V`, `HAT_3V3` | §3 | `HAT_5V` is bidirectional: out for HATs that draw power, in for a HAT that supplies the device (HCS-015, Draft) |
| Ground | `GND` (power and signal returns), `AGND` (analog reference) | — | `AGND` is the return of the line-level analog signals only; it joins `GND` at one point on the device |
| Presence and interrupt | `HAT_DET#` (in), `HAT_INT#` (in) | 3.3 V logic, active low, open drain on the HAT, pull-up on the device | `HAT_DET#` sits on the power connector, so it also proves that connector is mated |
| Identification | `ID_SCL`, `ID_SDA` | I2C, 3.3 V, standard or fast mode | Dedicated to the HAT EEPROM (HCS-007, HCS-017 Draft) at 7-bit address 0x50. No other device on this bus. The EEPROM is powered from `HAT_3V3` |
| I2C | `I2C_SCL`, `I2C_SDA` | I2C, 3.3 V | General-purpose bus, also the camera control (CCI) bus. HAT designers keep addresses distinct. Pull-up ownership is open (§7) |
| SPI | `SPI_SCLK`, `SPI_MOSI`, `SPI_MISO`, `SPI_CS0#`, `SPI_CS1#` | 3.3 V push-pull | Device is the controller |
| UART | `UART_TX` (out), `UART_RX` (in), `UART_RTS#` (out), `UART_CTS#` (in) | 3.3 V | With hardware flow control |
| Digital audio, I2S | `I2S_MCLK`, `I2S_BCLK`, `I2S_FS`, `I2S_DOUT` (out), `I2S_DIN` (in) | 3.3 V push-pull | Stereo I2S with its own clocks; the device can be clock provider or consumer |
| Digital audio, TDM in | `TDM_BCLK_IN`, `TDM_FS_IN`, `TDM_DIN0`–`TDM_DIN3` (all in) | 3.3 V push-pull | Multichannel capture with the HAT as clock provider (4 lines × 2 channels, or TDM on `TDM_DIN0`) |
| Analog audio | `LINE_OUT_L`, `LINE_OUT_R` (out), `LINE_IN_L`, `LINE_IN_R` (in) | Line level, referenced to `AGND` | Levels and impedances are open (§7) |
| USB 3 | `USB3_SSTX±` (out), `USB3_SSRX±` (in), `USB3_D±` (USB 2.0 pair of the same port) | USB 3 SuperSpeed + USB 2.0, 90 Ω differential | Device is host. TX AC-coupling capacitors on the transmitting side (device for `SSTX`, HAT for `SSRX`) |
| USB 2.0, second port | `USB2B_D±` | USB 2.0 high speed, 90 Ω differential | Device is host |
| PCIe | `PCIE_TX±` (out), `PCIE_RX±` (in), `PCIE_REFCLK±` (out), `PCIE_PERST#` (out), `PCIE_CLKREQ#` (in), `PCIE_WAKE#` (in) | PCIe x1, 85–100 Ω differential (**UNVERIFIED** target impedance, set with the FPC stack-up); sidebands 3.3 V | Device is root complex and sources the 100 MHz reference clock. TX AC-coupling capacitors on the transmitting side |
| Camera, MIPI CSI-2 | `CSI_CLK±`, `CSI_D0±`–`CSI_D3±` (all in), `CAM_RST#` (out), `CAM_MCLK` (out, optional) | MIPI D-PHY, 100 Ω differential; `CAM_RST#` 3.3 V; `CAM_MCLK` 3.3 V clock | One clock lane and up to four data lanes. Camera control on `I2C_*`. `CAM_MCLK` is optional: a camera with its own oscillator ignores it |
| Debug | `DBG_TCK` (SWCLK), `DBG_TMS` (SWDIO), `DBG_TDI`, `DBG_TDO` (SWO), `DBG_RESET#` | 3.3 V | SWD or JTAG to chips on the HAT (HCS-013) |
| GPIO | `GPIO0`–`GPIO7` | 3.3 V | `GPIO0`–`GPIO3` are PWM-capable; `GPIO4`–`GPIO7` may be slow (expander-class) on a given device, which declares it |
| Reserved | `RSVD_S0`, `RSVD_S1` (signal connector), `RSVD_P0`–`RSVD_P4` (power connector) | — | §5 |

**High-speed rates.** The interface sets the lane count and the electrical class, not a rate (HCS-004). Each link runs at the highest rate both sides and the channel support; the device and the HAT declare their rates in the capability manifest (HCS-009). Rates are measured on hardware at the end.

**Logic levels.** Every low-speed signal at the interface is 3.3 V logic. A device whose controllers run at another level translates on its own side; so does a HAT. Every 3.3 V signal on the device side is either isolated or in a defined state while `HAT_DET#` is high (no HAT).

## 3. Power

Ratings of the interface itself. Device budgets (how much a given device supplies on battery or on external power) are set in that device's documents and declared to the HAT; the HAT power classes (HCS-015, Draft) are written against them.

| Rail | Nominal | Direction | Interface rating (continuous) | Contacts on the power connector | Per contact at rating |
|---|---|---|---|---|---|
| `HAT_5V` | 5 V (window **UNVERIFIED**, to be set with the power classes) | out (classes A/B) or in (class F) | 2.0 A either direction | 12 | 0.17 A |
| `HAT_3V3` | 3.3 V | out | 1.0 A | 6 | 0.17 A |
| `GND` | — | return | 3.0 A (sum of both rails) | 22 | 0.14 A |

- **Contact derating.** The DF40 contact is rated 0.3 A (Hirose DF40 catalog, 2020, p. 2; no derating curve given). The interface keeps each contact at or below **70 % of its rating (0.21 A) in continuous use** and at or below **100 % (0.3 A) at the worst-case fault ceiling** of the device's current limiters. Fault ceilings: `HAT_5V` ≤ 3.6 A (12 × 0.3 A), `HAT_3V3` ≤ 1.8 A (6 × 0.3 A).
- **Device duties.** The device limits each rail it sources with a current limiter whose maximum trip current stays inside the fault ceiling, and limits current drawn from a class F HAT on `HAT_5V` the same way. It enables `HAT_5V` only after reading the HAT EEPROM.
- **HAT duties.** The HAT powers its EEPROM from `HAT_3V3`. Bulk capacitance on each rail stays at or below 470 µF (inrush budget of the board power tree §4.6; **Draft**).
- **Flat cable.** The power and ground conductors of the flat cable are sized for the interface rating (board BOM line 102).
- **Pogo field.** Each rail gets enough pogo contacts to stay at or below 70 % of the pogo contact rating at the interface rating, with at least one contact more than that minimum for redundancy (counts fixed with the pogo grid, §6).

## 4. Contact positions

Numbering follows the Hirose DF40 drawings: contacts 1, 3, 5 … on row A and 2, 4, 6 … on row B, contacts 1 and 2 at the same end (**UNVERIFIED** against drawing EDC3-311351-00; confirm before the FPC layout). The four corner contacts of each connector are metal hold-downs and carry nothing (DF40 catalog p. 10, note 3).

### 4.1 Signal connector (DF40, 100 contacts, 96 usable)

High-speed pairs run in one row in a ground–signal–signal–ground pattern (12 pairs, 13 grounds). The opposite row carries the slow signals; fast low-speed lines (clocks, SPI, I2S) and the analog lines sit at the far end, with the analog lines last, as the benchmark recommends.

| Row A | Signal | Row B | Signal |
|---|---|---|---|
| 1 | hold-down | 2 | hold-down |
| 3 | GND | 4 | GND |
| 5 | USB3_SSTX+ | 6 | ID_SCL |
| 7 | USB3_SSTX− | 8 | ID_SDA |
| 9 | GND | 10 | HAT_INT# |
| 11 | USB3_SSRX+ | 12 | I2C_SCL |
| 13 | USB3_SSRX− | 14 | I2C_SDA |
| 15 | GND | 16 | UART_TX |
| 17 | USB3_D+ | 18 | UART_RX |
| 19 | USB3_D− | 20 | UART_RTS# |
| 21 | GND | 22 | UART_CTS# |
| 23 | USB2B_D+ | 24 | GND |
| 25 | USB2B_D− | 26 | DBG_TCK |
| 27 | GND | 28 | DBG_TMS |
| 29 | PCIE_TX+ | 30 | DBG_TDI |
| 31 | PCIE_TX− | 32 | DBG_TDO |
| 33 | GND | 34 | DBG_RESET# |
| 35 | PCIE_RX+ | 36 | PCIE_PERST# |
| 37 | PCIE_RX− | 38 | PCIE_CLKREQ# |
| 39 | GND | 40 | PCIE_WAKE# |
| 41 | PCIE_REFCLK+ | 42 | CAM_RST# |
| 43 | PCIE_REFCLK− | 44 | GPIO0 |
| 45 | GND | 46 | GPIO1 |
| 47 | CSI_CLK+ | 48 | GPIO2 |
| 49 | CSI_CLK− | 50 | GPIO3 |
| 51 | GND | 52 | GPIO4 |
| 53 | CSI_D0+ | 54 | GPIO5 |
| 55 | CSI_D0− | 56 | GPIO6 |
| 57 | GND | 58 | GPIO7 |
| 59 | CSI_D1+ | 60 | RSVD_S0 |
| 61 | CSI_D1− | 62 | RSVD_S1 |
| 63 | GND | 64 | SPI_CS1# |
| 65 | CSI_D2+ | 66 | SPI_CS0# |
| 67 | CSI_D2− | 68 | GND |
| 69 | GND | 70 | SPI_MISO |
| 71 | CSI_D3+ | 72 | SPI_MOSI |
| 73 | CSI_D3− | 74 | SPI_SCLK |
| 75 | GND | 76 | GND |
| 77 | GND | 78 | CAM_MCLK |
| 79 | TDM_BCLK_IN | 80 | GND |
| 81 | TDM_FS_IN | 82 | I2S_MCLK |
| 83 | TDM_DIN0 | 84 | I2S_BCLK |
| 85 | TDM_DIN1 | 86 | I2S_FS |
| 87 | TDM_DIN2 | 88 | I2S_DOUT |
| 89 | TDM_DIN3 | 90 | I2S_DIN |
| 91 | GND | 92 | GND |
| 93 | LINE_IN_L | 94 | LINE_OUT_L |
| 95 | AGND | 96 | AGND |
| 97 | LINE_IN_R | 98 | LINE_OUT_R |
| 99 | hold-down | 100 | hold-down |

Count: 12 pairs (24) + 13 pair grounds + 43 low-speed + 6 analog (including 2 `AGND`) + 8 low-speed grounds + 2 reserved = 96.

### 4.2 Power connector (DF40, 50 contacts, 46 usable)

Every power contact faces a ground contact in the other row.

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

## 5. Reserved contacts (HCS-022)

- **Signal connector:** `RSVD_S0`, `RSVD_S1` (contacts 60, 62). They were freed by dropping the two signals the board pin plan had already marked as cheap to drop: a camera power-down line (`CAM_PWDN`; a camera HAT that needs one uses a `GPIO`) and an optional second digital-audio output clocked by the device's DAC (the `I2S_*` group keeps its own output). No essential signal was displaced.
- **Power connector:** `RSVD_P0`–`RSVD_P4` (contacts 5, 31, 45, 47, 48), low-speed only.
- **Rules:** a device leaves reserved contacts unconnected or tied through ≥ 100 kΩ to `GND`; a HAT leaves them unconnected. A later revision of this interface assigns them; a HAT declares the interface revision it was built for in its manifest.

## 6. Pogo field

The grid, pitch, contact positions and keying of the pogo field on the device-side piece are open (HCS requirements, Open questions). They follow the functional grouping of §4: high-speed pairs fenced by grounds on every side, power at the ends, analog at the far end from the high-speed field with its own `AGND`. The power counts follow §3.

## 7. Open points

1. Power classes A, B, F (HCS-015, Draft): current per class against §3, `HAT_5V` voltage window and the direction handshake.
2. Line-level analog: full-scale level, source and load impedance, and whether the device must provide `LINE_IN`.
3. Pull-up ownership on `I2C_*`, `ID_*`, `HAT_INT#` and the PCIe sidebands.
4. High-speed channel budget (flat cable + DF40 + pogo + HAT pads): DF40 rate rating **UNVERIFIED**; a test coupon is required before freeze (board BOM line 66).
5. DF40 contact numbering convention (§4) against the Hirose drawing.

## 8. Sources

- HAT Case Standard requirements: [`../requirements/README.md`](../requirements/README.md).
- HAT interconnect benchmark: [`../research/2026-10-hat-interconnect-benchmark.md`](../research/2026-10-hat-interconnect-benchmark.md).
- Hirose DF40 catalog (May 2020) and drawing EDC3-311351-00, through the parts library sheets `Hirose/DF40C-100DS-DP-0.4V.md` and `Hirose/DF40C-30DS-DP-0.4V.md` (the 50-contact size uses the same catalog ratings).
- Board BOM v1.2 (line 66) and power tree v0 (§4.6), board repository.
