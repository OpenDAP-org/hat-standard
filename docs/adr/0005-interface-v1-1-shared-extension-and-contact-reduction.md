# ADR 0005: Interface v1.1: shared high-speed extension and contact reduction

- **Status:** Accepted
- **Date:** 2026-10-24
- **Decided by:** Claude, under the maintainers' delegation, following their direction of 2026-10-24 (USB 3 is the base; the extension is PCIe or CSI-2 on the same reused contacts, selected by declared function; reduce contacts where the design still uses too many; board BRD-042 means no switch that limits the link or is not standard)
- **Amends:** [ADR 0002](0002-interface-v1-native-high-speed-management-over-usb.md) items 2 (line out, dedicated SWD), 8 (flat-cable connector) and the 4 × 25 amendment; [ADR 0003](0003-compatibility-levels.md) L2 and L3 scope; [ADR 0004](0004-pogo-part-and-field-geometry.md) field size and fence rule. Part and pitch of ADR 0004 stand.

## Context

[Interface v1](../specs/interface-v1.md) v1.0 used 100 pogo contacts, a DF40 100 and a DF40 50 per cable end. Ground was 31 % of the field, PCIe and CSI-2 ran side by side on 8 pairs, and 14 contacts were reserved. The [v1.1 contact optimisation](../research/2026-10-interface-v1-1-reduction.md) checked each group against what the reference HATs and the requirements need, and found:

- PCIe x1 needs 3 pairs and CSI-2 at 4 lanes 5; one extension of 5 pairs carries either (research §2.1).
- On the T1, PCIe and CSI-2 come from separate PHYs, so selection needs a switch. TI TMUXHS4412 (passive 2:1, 4 channels, −3 dB at 13 GHz, −1.3 dB at 8 GHz, 0–1.8 V common mode, listed for PCIe Gen 1–4 and MIPI CSI) has 5× the PCIe Gen2 Nyquist and 10× the D-PHY 2.5 Gb/s Nyquist (research §2.3).
- Pairs laid along the rows with shared grounds need 13 fence grounds instead of 24 (research §4.1, **UNVERIFIED** until the coupon).
- Line out has no source on the board and is replaced by bit-perfect I2S on the DAC clock; dedicated SWD can be a GPIO alternate because the T1 already bit-bangs it (research §3).
- "DF40HC" is a tall-stack style code, not a high-current contact; every DF40 is 0.3 A (catalog 2020, pp. 2, 3). Power fits on the signal connector once the interface fault ceilings follow the limiters rather than a connector count (research §5).
- Merging presence with interrupt, or the ID EEPROM onto the general I2C, saves 1 and 2 contacts but makes the power-up safety order depend on third-party behaviour (research §3).

## Decision

1. **Interface v1.1 (Draft)** replaces v1.0 in place, with v1.0 kept as history (spec §3.1, commit `42fcf14`). It is breaking against the v1.0 pad map; it stays revision 1 because nothing was frozen or built against v1.0, and the identity record's revision becomes binding at freeze.
2. **Levels:** L1 Core (unchanged; USB 3 is the base); L2 Control and audio (I2S on the DAC clock, four fast GPIO, `HAT_RST#`); **L3 High-speed extension**: five pairs `EXT0±`–`EXT4±` and three sidebands `EXT_SB0`–`EXT_SB2`, carrying **PCIe x1 or CSI-2 up to 4 lanes** by the function the HAT declares. In PCIe mode `EXT3`/`EXT4` are reserved for a second PCIe lane. An L3 device offers both functions. One field, one pinout, nested zones.
3. **Sidebands by mode:** `EXT_SB0` is PERST# or the sensor reset, held asserted until HAT power is good; `EXT_SB1`/`EXT_SB2` are CLKREQ#/WAKE# or camera GPIO.
4. **Selection:** a device keeps the extension disconnected until it has read the identity record; it selects by SoC pin-mux, or by a passive switch that does not limit either link. The T1 uses **two TI TMUXHS4412**, all five pairs through matched channels; HD3SS3212 (× 3) is the alternative.
5. **Removed:** line out with its two `AGND`; dedicated `DBG_SWCLK`/`DBG_SWDIO` (SWD becomes a **mandatory L2 alternate** on `GPIO2`/`GPIO3`); low-speed grounds.
6. **Kept:** the dedicated ID bus, separate `HAT_DET#` and `HAT_INT#`, `HAT_RST#`, all eight I2S lines, four GPIO, the pogo power counts (3 / 2 / 4).
7. **Pogo field: 4 × 17 = 68 at 2.54 mm**, pairs along rows A and D with shared grounds, power returns under the USB 3 SuperSpeed pairs. L1 29, L2 45, L3 68 cumulative. **Reserved 8**: 3 / 3 / 2, the minimum that keeps every level ≥ 8 % after the audio-clock fallback.
8. **Fault ceilings:** `HAT_5V` 3.0 A, `HAT_3V3` 1.5 A, `GND` 4.5 A (were 3.6 / 1.8 / 5.4 A). Continuous ratings unchanged.
9. **Flat cable (T1): one DF40C-100 pair per end** for signals and power: 87 of 96 used, 9 spare (9.4 %); power 10 / 5 / 15 at 67 % continuous and 100 % at the ceilings.
10. **Signal integrity stays UNVERIFIED.** The ADR 0004 coupon now covers the row fence, the switch (including D-PHY low-power signalling through it) and a ground-backed fallback (about 4 × 20 = 80 positions) that reverses the fence change if needed before the pad map freezes.

## Consequences

| | v1.0 | v1.1 |
|---|---|---|
| Pogo L1 / L2 / L3 (cumulative) | 32 / 60 / 100 | **29 / 45 / 68** |
| High-speed pairs | 11 | 8 |
| Fence grounds | 24 | 13 |
| Low-speed signals | 24 | 22 |
| Analog | 4 | 0 |
| Reserved | 14 (14 %) | 8 (11.8 %) |
| Field span | 60.96 × 7.62 mm | 40.64 × 7.62 mm |
| Spring force, L3, P70 | 88 N | 60 N |
| DF40 per cable end | 100 + 50 (118 used) | 100 (87 used) |
| ESD lines, high-speed / low-speed / analog | 22 / 29 / 2 | 16 / 30 / 0 |

- Every reference HAT keeps working at its level (research §6). The camera's sensor reset moves to `EXT_SB0`; its on-board AI (CAM-004, should) cannot use PCIe beside CSI-2 and runs on the HAT or behind USB 3. The recorder monitors through `I2S_DOUT`.
- Harder for future HATs: PCIe and CSI-2 at once (one moves to USB 3), an analog amplifier fed by the device's DAC (uses I2S and its own converter), SWD and UART B at once.
- The device-side piece, the HAT case and the case mount follow the shorter field.
- Open: identity-record fields for the extension function (spec §10 item 2), the coupon (§10 item 4).

## Board changes (for the board owner)

The board repository is not edited by this ADR. These are the changes interface v1.1 needs:

### BOM v1.9

| Line | Now | Change |
|---|---|---|
| 66 | DF40C-100DP/-100DS (signals) + DF40C-50DP/-50DS (power), 2 + 2 per end | **DF40C-100DP-0.4V(51) + DF40C-100DS-0.4V(51), 1 + 1 per end**, signals and power; DF40C-50 pair removed. Allocation: interface v1.1 §7.2 (87 used, 9 spare). Per contact at 0.3 A: `HAT_5V` × 10 2.0 A = 0.20 A (67 %); `HAT_3V3` × 5 1.0 A = 0.20 A (67 %); `GND` × 15 3.0 A = 0.20 A (67 %); at the interface ceilings 3.0 / 1.5 / 4.5 A: 100 %. T1 limiters: eFuse #A 1.66 A out = 55 %, TPS2121 3.0 A in = 100 %, eFuse #B 1.15 A = 77 %, `GND` 4.15 A = 92 %. `HAT_DET#` at 33 proves the single connector mated. Both ends keyed (not polarized) |
| 67 | P70-1000045R × 100 (4 × 25) | **× 68 (4 × 17 at 2.54 mm, 40.64 × 7.62 mm span)**; Mill-Max 0965 second source unchanged; force ≈ 60 N (40–80 N), 0965 ≈ 77 N |
| 68 | TPD4E05U06DQAR × 6 (11 pairs) | **× 4**: 8 pairs (USB 3 SS × 2, USB 2, `EXT0`–`EXT4`), 16 lines on 16 channels |
| 69 | TPD4E05U06DQAR × 8 (29 lines) + TPD1E10B06DPYR × 2 (line out) | **TPD4E05U06DQAR × 8** for 30 lines (22 low-speed + 8 reserved, all with cable positions) on 32 channels; **TPD1E10B06 × 0** (line out removed) |
| 70 | 100 nF × 4 (USB 3 SSTX, PCIe TX) | Quantity unchanged; the PCIe TX pair sits between the SoC and the extension switch, never on the shared `EXT1` path |
| new | — | **HAT extension switch: TI TMUXHS4412 × 2** (4-channel passive 2:1, WQFN-42 3.5 × 9 mm, Active; −3 dB 13 GHz, −1.3 dB at 8 GHz, 0–1.8 V common mode, PCIe Gen 1–4 and MIPI CSI listed; 320 µA active, 0.1 µA standby). Common ports to `EXT0`–`EXT4`; port A: PCIe REFCLK, TX, RX on `EXT0`–`EXT2`; port B: CSI D0, CLK, D1, D2, D3 on `EXT0`–`EXT4`; three channels unused. Alternative: HD3SS3212 × 3. Stock and price **UNVERIFIED**; return loss and crosstalk at 2.5 GHz from the datasheet, then the coupon |
| 44, 45 | eFuse #A, TPS2121 | Confirm each maximum trip current, tolerance included, is ≤ 3.0 A (`HAT_5V`, both directions); if TPS2121's ILM tolerance exceeds 3.0 A, lower its ILM setting. eFuse #B ≤ 1.5 A (1.15 A holds) |
| 102 | HAT flat cable | One DF40C-100 at each end; power conductors sized for 2.0 / 1.0 / 3.0 A continuous and 3.0 / 1.5 / 4.5 A fault; shared `EXT` pairs at 100 Ω differential |
| 135 | SN74LVC1G57 G14 (PERST# gate) | Unchanged part; output net becomes `HAT_EXT_SB0` (PERST# or sensor reset, both held until HAT power good) |

### Pinout v1, HAT section

| Ball | Now | Change |
|---|---|---|
| A22 (GPIO1_A0) | `HAT_DBG_SWCLK` | Freed; proposed `HAT_EXT_SEL` (switch select). `_u` ball: an external pull sets the reset state |
| B22 (GPIO1_A1) | `HAT_DBG_SWDIO` | Freed; proposed `HAT_EXT_EN` (switch enable): switch disabled at reset and in standby (GPIO1 loses state, HDG p. 35), held by an external pull that wins over the internal pull-up |
| 1P3 (GPIO3_C4), 1P4 (GPIO3_C5) | `HAT_GPIO2`, `HAT_GPIO3` (UART B alternates) | Add bit-banged SWD alternates: SWCLK on `HAT_GPIO2`, SWDIO on `HAT_GPIO3` |
| B26 (GPIO1_B2) | `SOC_PCIE_PERST_N` | `SOC_EXT_SB0`: PERST# in PCIe mode, sensor reset in CSI-2 mode (into G14, out as `HAT_EXT_SB0`) |
| 1B13 (GPIO1_B0) | `PCIE_CLKREQ_N` | `HAT_EXT_SB1`: PCIE20_CLKREQn_M2 in PCIe mode, GPIO in CSI-2 mode |
| A26 (GPIO1_B1) | `PCIE_WAKE_N` | `HAT_EXT_SB2`: PCIE20_WAKEn_M2 in PCIe mode, GPIO in CSI-2 mode |
| 1K19/1K20, AA37/AA38, AB37/AB38 | `HAT_PCIE_REFCLK/TX/RX` to the DF40 | To switch port A of `EXT0`, `EXT1`, `EXT2` |
| AP18/AR18, 1V9/1U9, AR17/AP17, AP15/AR15, AR14/AP14 | `HAT_CSI_D0/CLK/D1/D2/D3` to the DF40 | To switch port B of `EXT0`, `EXT1`, `EXT2`, `EXT3`, `EXT4` |
| AG37 (GPIO0_A5) | `HAT_DET_N` on the power connector | Same ball; DF40 position 33 of the single connector |
| — | Pin plan §4.3 row `LINE_OUT_L/R`, `AGND` | Remove (no ball was assigned) |

**PCB design constraints:** `EXT0`–`EXT4` at 100 Ω ± 10 % differential from the switch common port to the DF40; CSI-2 lane-to-clock matching includes the switch (all five lanes through one channel each); no branch on the common side; the unused port branches as short as placement allows; PCIe TX AC caps on the SoC side of the switch; REFCLK through the switch like the data pairs.

**Feature blocks:** `blk.hat-signals`: interface becomes one DF40C-100 (signals and power), the two extension switches, `EXT_SB0`–`EXT_SB2`, SWD as a GPIO alternate; absent state adds "extension switch disabled, `HAT_EXT_SB0` asserted"; alternates add HD3SS3212 × 3; bound by adds HCS-026 and this ADR. `blk.hat-5v` and `blk.hat-3v3`: connector is the single DF40C-100; the interface fault ceilings are 3.0 A and 1.5 A. `blk.hat-audio`: unchanged.
