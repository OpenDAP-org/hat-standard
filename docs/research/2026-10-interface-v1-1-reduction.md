# HAT interface v1.1: contact optimisation

- **Status:** Research input for [ADR 0005](../adr/0005-interface-v1-1-shared-extension-and-contact-reduction.md). Figures without a primary source are marked **UNVERIFIED**. Derived numbers (sums, ratios, forces) are computed here from the cited figures and say so.
- **Date:** 2026-10-24. Web pages were read on this date.
- **Question:** the maintainers (2026-10-24) say the interface still uses too many contacts in places, and set the direction: **USB 3 is the base (L1); the high-speed extension is PCIe or MIPI CSI-2 (camera in) on the same reused contacts, selected by the function the HAT declares** — not PCIe and CSI-2 side by side as in v1's L3. They clarified board BRD-042: a standard high-speed switch is acceptable if it does not limit the link and uses standard parts and protocols. What is the smallest interface that keeps every reference HAT working at its level?
- **Inputs:** [interface v1](../specs/interface-v1.md) as committed in `42fcf14` (here "v1.0"); [requirements](../requirements/README.md); ADR [0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md)–[0004](../adr/0004-pogo-part-and-field-geometry.md); [interface reduction](2026-10-interface-reduction.md), [pogo part and geometry](2026-10-pogo-part-and-geometry.md), [interconnect benchmark](2026-10-hat-interconnect-benchmark.md); board [BOM v1.9](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md) lines 65–70, 102, 112, 135; board [pinout v1](https://github.com/opendap-project/board/blob/main/docs/specs/pinout-v1.md) HAT section; board [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md); board requirements BRD-006, BRD-042, BRD-044, BRD-045, BRD-049; parts library `Hirose/DF40C-100DS-DP-0.4V.md`, `Hirose/DF40C-50DS-DP-0.4V.md`, `Harwin/P70-1000045R.md`; the reference HATs' requirements (camera, NAS, oscilloscope, powerbank, radio, recorder, VNA).
- **Primary sources read for this note:** Hirose DF40 catalog, edition May 1, 2020 (cached copy, sha256 `d49c00f0…dd370`): p. 1 (size and stacking-height table), p. 2 (ratings), p. 3 (product number structure), p. 4 (receptacle table), pp. 6–7 (DF40HC receptacles), p. 10 (header table and notes); TI product pages for TMUXHS4412, HD3SS3212, HD3SS3412, TMUX646 and TS5MP646 (ti.com, 2026-10-24).
- **Research limit:** the web-search budget of this session was exhausted; sources were reached by direct URL only. onsemi FSA644 and Diodes PI3WVR-family switches could not be read and are not assessed. No datasheet curve was read for any switch: only the figures on the product pages.

## 0. Summary

1. **The extension shares five pairs.** PCIe x1 needs 3 pairs (TX, RX, REFCLK), CSI-2 needs a clock and up to 4 data lanes (5 pairs). One extension of **5 pairs** carries either: CSI-2 on all five, PCIe x1 on three, and the other two kept for a second PCIe lane in a later revision. v1.0 carried 8 pairs for the same functions. The three sidebands serve both functions by declared mode (PERST#, CLKREQ#, WAKE# for PCIe; sensor reset and two camera GPIO for CSI-2).
2. **The T1 needs a switch; one standard part carries both protocols.** On the RK3566, PCIe (combphy2) and CSI-2 (its own D-PHY) are separate PHYs, so no SoC pin-mux can join them. TI **TMUXHS4412** (4-channel passive 2:1, −3 dB at 13 GHz, −1.3 dB at 8 GHz, 0–1.8 V common mode, listed for PCIe Gen 1–4 and MIPI DSI/CSI) has five times the bandwidth of the PCIe Gen2 Nyquist frequency (2.5 GHz) and ten times that of a 2.5 Gb/s D-PHY lane (1.25 GHz). Two parts put all five pairs through matched switch channels. It meets the BRD-042 criterion on the figures available; return loss and crosstalk at 2.5 GHz go to the coupon.
3. **Pogo field: 100 → 68 positions (4 × 17 at 2.54 mm), −32 %.** By level, cumulative: L1 32 → 29, L2 60 → 45, L3 100 → 68. The cuts: lane sharing (−6), a shared-ground fence with pairs lying along the rows (24 → 13 grounds, −11), line out and its analog grounds (−4), SWD as a GPIO alternate (−2), low-speed grounds folded into the fence (−3), reserved 14 → 8 (−6). Every level keeps ≥ 8 % reserved, also after the audio-clock fallback takes two.
4. **Flat cable: one DF40C-100 pair per end instead of a 100 and a 50.** 87 of 96 usable contacts with 9 spare (9.4 %). Power moves onto the signal connector with 10 `HAT_5V`, 5 `HAT_3V3` and 15 power `GND` contacts, which holds BRD-049 because the interface fault ceilings come down to what the T1's limiters already enforce (3.0 A, 1.5 A, 4.5 A). **There is no high-current DF40 in the catalog**: "DF40HC" is the style code for a ≥ 2.5 mm stack without reinforcing metal fittings, rated 0.3 A like the rest.
5. **Low-speed ESD lines:** v1.0's real list is 24 native low-speed signals; BOM line 69's 29 is those plus the 5 reserved contacts that had signal-connector positions. v1.1 has **22 low-speed signals + 8 reserved = 30 lines**, all protected (TPD4E05U06 × 8, unchanged quantity); the 2 line-out protectors go. High-speed ESD falls from 11 to 8 pairs (TPD4E05U06 × 6 → × 4).
6. **Kept, with reasons:** the dedicated ID bus, separate presence and interrupt lines, native SWD (as an alternate), `HAT_RST#`, all eight I2S lines, four fast GPIO, the pogo power counts.
7. **Signal integrity is UNVERIFIED** for the new fence, the switch and the DF40 until the ADR 0004 coupon. The pinout keeps a costed fallback (§5.5) so the coupon can reverse the fence change before the pad map freezes.

## 1. Where v1.0 spends its contacts

Pogo field, v1.0 §4.4 (100 positions):

| Group | Contacts | Share | Driver |
|---|---|---|---|
| High-speed signals | 22 | 22 % | 11 pairs: USB 3 (3), PCIe (3), CSI-2 (5) |
| High-speed ground fence | 24 | 24 % | Pairs stand vertically (A+/B−), so each ground column costs two contacts; the L1 band is fenced on its own |
| Low-speed signals | 24 | 24 % | L1 6, L2 15, L3 3 |
| Analog | 4 | 4 % | Line out L/R and two `AGND` |
| Low-speed grounds | 3 | 3 % | I2S returns |
| Power | 9 | 9 % | `HAT_5V` 3, `HAT_3V3` 2, `GND` 4 |
| Reserved | 14 | 14 % | ≥ 8 % per level, plus 4 added by the 4 × 25 grid |

Ground is 31 of 100 contacts (fence 24, low-speed 3, power 4). The fence, the parallel PCIe and CSI-2 lanes and the reserve are the three large items.

Flat cable (T1), v1.0 §7.2–§7.3: a DF40C-100 signal pair (72 of 96 used, 24 spare) and a DF40C-50 power pair (46 of 46): 150 positions, 118 used.

## 2. The shared high-speed extension

### 2.1 Width

| Function | Pairs | Direction (device view) |
|---|---|---|
| PCIe x1 | TX, RX, REFCLK = 3 | out, in, out |
| CSI-2 D-PHY, 4 lanes | CLK + D0–D3 = 5 | all in |
| CSI-2, 2 lanes | CLK + D0–D1 = 3 | all in |

Options:

| Extension width | PCIe | CSI-2 | Pogo signal contacts | Verdict |
|---|---|---|---|---|
| 3 pairs | x1 | 2 lanes | 6 | Caps every camera HAT at 2 lanes, half the RK3566's CSI-2 (4 × 2.5 Gb/s, board topology §2): against HCS-004 |
| **5 pairs** | x1 (2 pairs kept for x2) | **4 lanes** | 10 | Full CSI-2 of the T1; PCIe x1 of the T1; x2 room for another device |
| 8 pairs (v1.0) | x1 | 4 lanes, both at once | 16 | What v1.0 does; the maintainers' direction removes it |

**Decision input: 5 pairs.** In PCIe mode the two extra pairs are reserved for a second PCIe lane (TX1, RX1) that a later revision may assign; a device with PCIe x2 (not the T1) would then use them without new contacts. This keeps the standard ready for a stronger SoC (HCS-021) at no contact cost.

Lane map (both functions start at the same pairs, so a 2-lane camera or a PCIe x1 HAT uses only `EXT0`–`EXT2`):

| Pair | PCIe mode | CSI-2 mode |
|---|---|---|
| `EXT0±` | `PCIE_REFCLK±` (out) | `CSI_D0±` (in) |
| `EXT1±` | `PCIE_TX±` (out) | `CSI_CLK±` (in) |
| `EXT2±` | `PCIE_RX±` (in) | `CSI_D1±` (in) |
| `EXT3±` | reserved (lane 1 TX, later revision) | `CSI_D2±` (in) |
| `EXT4±` | reserved (lane 1 RX, later revision) | `CSI_D3±` (in) |

### 2.2 Sidebands

| Contact | PCIe mode | CSI-2 mode | Electrical |
|---|---|---|---|
| `EXT_SB0` | `PCIE_PERST#` (out) | Sensor reset or enable (out, active low) | 3.3 V, device output; held asserted until HAT power is good, in both modes |
| `EXT_SB1` | `PCIE_CLKREQ#` (in, open drain) | Camera GPIO (declared: power-down, strobe, frame sync) | 3.3 V, pull-up on the device |
| `EXT_SB2` | `PCIE_WAKE#` (in, open drain) | Camera GPIO (declared) | 3.3 V, pull-up on the device |

v1.0 used `GPIO0` for the camera's sensor reset. With `EXT_SB0` the camera keeps all four L2 GPIO lines, and the reset inherits the power-good gating that PERST# already has on the T1 (board BOM line 135).

### 2.3 How a device realises the selection

- **Same PHY:** where a SoC offers PCIe and CSI-2 from one PHY or one pin group, the pin multiplexer selects; no part is added.
- **Separate PHYs (the T1):** the RK3566 takes PCIe from combphy2 and CSI-2 from its own D-PHY (board high-speed I/O topology §2: "PCIe 2.1 x1 uses combphy2"; "MIPI CSI-2 RX … one D-PHY"). No pin-mux joins them. The two choices are a passive 2:1 switch, or routing both PHYs to the same contacts, which leaves an unterminated stub on every shared pair and is not acceptable. **The T1 needs a switch.**

BRD-006 allows multiplexers. BRD-042, as the maintainers clarified on 2026-10-24, rejects anything that limits or interferes with the maximum the hardware offers, or that is not standard. The criterion used here: the switch must pass **PCIe Gen2 (5 GT/s, Nyquist 2.5 GHz)** and **D-PHY at the RK3566's 2.5 Gb/s per lane (Nyquist 1.25 GHz)**, including D-PHY low-power signalling (single-ended, up to about 1.2 V; level **UNVERIFIED** against the MIPI D-PHY text), with margin in insertion loss, return loss, crosstalk and bandwidth, and be a catalog part used for those protocols.

| Part | Channels | −3 dB BW | Insertion loss | Common-mode range | Listed protocols | Status | Verdict |
|---|---|---|---|---|---|---|---|
| **TI TMUXHS4412** | 4 diff., 2:1 | 13 GHz | −1.3 dB at 8 GHz | 0–1.8 V | PCIe Gen 1–4, USB 3.2, USB4, MIPI DSI/CSI, LVDS, … | Active | **Meets the criterion on the published figures.** 5.2× the PCIe Nyquist, 10× the D-PHY Nyquist; the LP levels sit inside the common-mode range. 320 µA active, 0.1 µA standby (≈ 1 mW at 3.3 V, derived) |
| TI HD3SS3212 | 2 diff., 2:1 | > 8 GHz | −1.6 dB at 5 GHz | 0–2 V | USB 3.1, MIPI DSI/CSI, PCIe Gen II and III, LVDS | Active | Meets it too; needs three parts for five pairs. Alternative, not a same-footprint second source. Already studied by the board for the USB 3 lane (topology §4, option U2) |
| TI HD3SS3412 | 4 diff., 2:1 | > 8 GHz | −1.5 dB at 4 GHz; crosstalk −35 dB, return loss −11 dB at 4 GHz | < 2 V | PCIe Gen III, USB 3.0, DP 1.2 (no MIPI listed) | Active | Not chosen: MIPI is not a listed use |
| TI TMUX646 | 5 diff., 2:1 | 6 GHz | not on the page; RON 6 Ω typ. | — | MIPI CSI/DSI, C-PHY/D-PHY | Active | Not chosen: MIPI only, PCIe not a listed use |
| TI TS5MP646 | 5 diff., 2:1 | 3 GHz | not on the page | — | MIPI | NRND | Rejected: not for new designs |
| onsemi FSA644, Diodes PI3WVR family | — | — | — | — | — | — | Not read (research limit) |

- **One switch type for both protocols.** TI lists both PCIe and MIPI CSI on the TMUXHS4412 and the HD3SS3212 pages. A remark that HD3SS-class switches suit CML but not D-PHY does not match those pages: both list MIPI DSI/CSI, and both common-mode ranges cover the LP levels. The behaviour of LP-mode signalling through the switch (both lines driven high single-ended at 10 Mb/s class rates) is still **UNVERIFIED** until the datasheet and the coupon.
- **Two TMUXHS4412 per T1, all five pairs switched.** Three pairs need a 2:1 path (PCIe or CSI-2). Passing `EXT3` and `EXT4` through the remaining channels too (port B only) gives every CSI-2 lane the same delay and loss, so lane-to-clock skew does not depend on which lane was switched, and in PCIe mode the CSI receiver is disconnected from `EXT3`/`EXT4`. Cost: one more part than the minimum. A single part plus length compensation on `EXT4` is possible but makes the skew budget depend on the switch's propagation delay (**UNVERIFIED**).
- **AC coupling:** PCIe TX capacitors (board BOM line 70) sit between the SoC and the switch, never on the shared path, because CSI-2 is DC-coupled.
- **Impedance:** the shared pairs use one target. D-PHY is 100 Ω ± 10 % differential (board interface-standards layout research); 100 Ω is taken for the shared pairs, which PCIe Gen2 tolerates (**UNVERIFIED** against the PCIe CEM text; set with the FPC stack-up and confirmed on the coupon).
- **Reset and absent states:** the switch is disabled while no L3 HAT is declared, so no PHY sees the pogo field; the device reads the identity record, selects, enables `HAT_5V`, then enables the switch, then releases `EXT_SB0`.
- **Budget:** the switch adds its insertion loss to the flat cable, DF40 and pogo channel. The coupon of ADR 0004 now includes the switch.

### 2.4 Requirement checks

- **HCS-004 (no cap):** each function keeps its full width (PCIe x1, CSI-2 4 lanes) and rate; the switch's published bandwidth is well above both Nyquist frequencies. What is lost is concurrency: one HAT cannot use PCIe and CSI-2 at the same time (§6). This is the maintainers' direction, and USB 3 stays alongside either function.
- **HCS-005:** §6 lists what gets harder and the path each case takes.
- **BRD-045 (PCIe straight to the HAT):** a passive switch is not a PCIe device; the link stays root complex to endpoint. The board owner confirms the reading with BRD-042 as clarified.
- **HCS-021:** the extension is defined by function and lane, not by PHY; a device may realise it by pin-mux, by switch or by dedicated PHYs.

## 3. Low-speed reuse, one by one

| Proposal | Saves (pogo / DF40) | Keeps every reference HAT? | Cost | Verdict |
|---|---|---|---|---|
| GPIO0–3 alternates UART, SPI (v1.0) | — | Yes | — | Kept |
| **SWD on `GPIO2`/`GPIO3` as an alternate** instead of dedicated `DBG_SWCLK`/`DBG_SWDIO` | 2 / 2 | Yes (the oscilloscope is L1; no reference HAT needs SWD and UART B together) | A HAT that uses SWD loses UART B and two of its four GPIO; it uses CDC-ACM for its console. On the T1, SWD is already bit-banged (pinout `HAT_DBG_SWCLK` "bit-banged SWD"), so any 3.3 V GPIO ball can carry it | **Adopted** |
| SWD only over the HAT's USB | 2 / 2 | Yes | A bricked USB bridge MCU without a ROM USB loader becomes unrecoverable from the device, against HCS-013 | Rejected; the alternate keeps native recovery |
| Presence and interrupt on one line | 1 / 1 | Yes | `HAT_DET#` must be valid before any HAT power (a strap to `GND`); `HAT_INT#` is driven by a powered HAT. One line makes removal and an interrupt indistinguishable and breaks the power-up order | Rejected |
| ID EEPROM on the general I2C at a reserved address | 2 / 2 | Yes | The identity read gates `HAT_5V` (BRD-049). On a shared bus it depends on every third-party device being well-behaved while only `HAT_3V3` is on: a device on `HAT_5V` can clamp the bus through its protection diodes, a hung device blocks the read, and addresses 0x50–0x57, common on EEPROMs and sensors, would be lost to every HAT. HAT+ keeps a separate ID bus (HCS-024). HCS-007 itself only requires an EEPROM; the dedicated bus comes from ADR 0002 item 2, which a change would amend | Rejected: two contacts do not pay for a power-safety dependency |
| **Line out removed** (`LINE_OUT_L/R`, two `AGND`) | 4 / 4 | Yes (the recorder's optional monitoring uses `I2S_DOUT` or the T1's own outputs) | A HAT that wants the T1's analog output takes the bit-perfect, sample-synchronous `I2S_DOUT` on the DAC clock and carries its own converter, or uses the T1's headphone outputs. The board never had a source for it (pin plan: "source is a board decision"); a tap on the DAC's analog output would load the audiophile path, and a line-level signal through a spring field beside 5 Gb/s lanes and power returns is the worst place for it | **Adopted** |
| **Sidebands shared by mode** (§2.2) | 1 / 1 against v1.0 + `CAM_RST#` habit | Yes | None for PCIe; a camera gets reset and two GPIO next to its lanes | **Adopted** |
| I2S data lines 4 → 2 with TDM | 2 / 2 | — | The HAT port runs on the DAC's bit clock, which the DAC (clock master) sets; 8 channels at 192 kHz need 4 lines at 2 channels each (v1.0 §4.2) | Rejected: audio is not traded |
| Fewer fast GPIO | — | — | PPS, PWM and resets need native timing (v1.0 §6.4) | Kept at 4 |
| `HAT_RST#` as a GPIO alternate | 1 / 1 | — | It must be in a defined, asserted state from device reset, independent of any HAT declaration | Kept |

## 4. Pogo field

### 4.1 Fence

In v1.0 each pair stands in one column (positive in row A, negative in row B) with a ground column on each side: every ground column is two contacts, and the L1 band needs its own. Laying each pair **along a row** (G S S G S S G) lets adjacent pairs share one ground, and the row next to the pairs carries power returns or static lines:

| Pattern | Grounds for 8 pairs | Each pair has |
|---|---|---|
| v1.0 vertical, ground columns, 11 pairs | 24 | a two-contact ground column each side |
| **Along rows A and D, shared grounds** | **13** fence (11 in rows A and D, 2 in row C over the row-D extension pairs) + the 4 power returns under the USB 3 pairs | a ground on each side in its row; power returns (USB 3 SS), ground or static lines in the adjacent row |

This keeps ADR 0004's rule ("every high-speed pair has a ground neighbour on both sides"). Pairs sit in the two outer rows (A and D), three rows (7.62 mm) apart; the inner rows (B, C) never hold a pair. Positions orthogonally next to a pair contact hold only `GND`, power or static and slow lines (the v1.0 row-C rule, now for rows B and C). The four power returns lie under the USB 3 SuperSpeed pairs, so they serve as the pairs' reference row. **UNVERIFIED** until the coupon.

### 4.2 Reserved contacts

Rule: ≥ 8 % of every cumulative level (ADR 0002), also after the audio-clock fallback takes two reserved contacts at L2 or lower (ADR 0002 amendment). Minimum by level, derived: L1 needs 3 of 29 (10.3 %); L2 needs 6 of 45 (13.3 %, 8.9 % after the fallback); L3 needs 8 of 68 (11.8 %, 8.8 % after the fallback). v1.1 holds exactly these: 3 + 3 + 2 = **8** (v1.0: 14). The 2 extension pairs reserved for a second PCIe lane are a functional reserve on top (§2.1).

### 4.3 Count and grid

| Group | v1.0 | v1.1 | Change and reason |
|---|---|---|---|
| High-speed signals | 22 | 16 | −6: extension shares 5 pairs (§2) |
| Fence ground | 24 | 13 | −11: shared grounds along the rows (§4.1) |
| Low-speed signals | 24 | 22 | −2: SWD becomes an alternate (§3) |
| Analog | 4 | 0 | −4: line out removed (§3) |
| Low-speed ground | 3 | 0 | −3: the clocks sit next to fence grounds |
| Power (`HAT_5V`/`HAT_3V3`/`GND`) | 3 / 2 / 4 | 3 / 2 / 4 | Unchanged: the 2 + 1, 1 + 1, 3 + 1 rule (v0 §3) |
| Reserved | 14 | 8 | −6: the rule minimum (§4.2) |
| **Total** | **100** | **68** | **−32 %** |

68 = 4 × 17 at 2.54 mm: contact span 40.64 × 7.62 mm (derived; v1.0 60.96 × 7.62 mm). The pitch and part of ADR 0004 do not change.

By level (in-zone / cumulative / reserved cumulative):

| Level | v1.0 | v1.1 |
|---|---|---|
| L1 | 32 / 32 / 3 (9.4 %) | 29 / 29 / 3 (10.3 %) |
| L2 | 28 / 60 / 9 (15.0 %) | 16 / 45 / 6 (13.3 %) |
| L3 | 40 / 100 / 14 (14.0 %) | 23 / 68 / 8 (11.8 %) |

### 4.4 Spring force (retention)

Derived from the part figures (P70-1000045R drawing: 0.88 N ± 0.29 N at 2.90 mm; Mill-Max 0965: 115 gf at mid stroke, pogo research §2):

| Piece | Contacts | Harwin P70 | Mill-Max 0965 |
|---|---|---|---|
| L3 (v1.0) | 100 | 88 N (59–117 N) | 113 N |
| **L3 (v1.1)** | **68** | **59.8 N (40.1–79.6 N)** | **76.7 N** |
| L2 (v1.1) | 45 | 39.6 N (26.6–52.7 N) | 50.8 N |
| L1 (v1.1) | 29 | 25.5 N (17.1–33.9 N) | 32.7 N |

The retention and the HAT's stiffener carry about a third less.

## 5. Flat cable (T1)

### 5.1 DF40 sizes and the "high-current" question

- **Sizes** (catalog 2020 p. 1, standard type, 1.5 mm stack): 10, 12, 20, 24, 30, 34, 40, 50, 60, 70, 80, 90, 100 positions (44 exists as a header and at 2.0/3.0 mm, not as a 1.5 mm receptacle). On every DF40C header the **four corner contacts are metal fittings** and carry nothing (p. 10, note 3): usable = positions − 4.
- **DF40HC is not a high-current part.** In the product number structure (p. 3) "C" and "HC" are both "without reinforcing metal fitting", "the H denotes a stacking height 2.5 mm or above"; the DF40HC(2.5)/(3.0)/(3.5)/(4.0) receptacle tables (pp. 6–7) are taller stacks of the same contact. The only rating in the catalog is 0.3 A (p. 2). A separate Hirose high-current DF40 product was not found and is **UNVERIFIED**; it is not considered.

### 5.2 Power contacts against BRD-049 (0.3 A per contact)

The v1.0 interface fault ceilings (3.6 A, 1.8 A, 5.4 A) were defined from the DF40 contact count (12 × 0.3 A, 6 × 0.3 A), not from a limiter. The T1's limiters are lower: `HAT_5V` 1.66 A out (eFuse #A) and 3.0 A in (TPS2121 ILM), `HAT_3V3` 1.15 A (eFuse #B) (board BOM line 66). Setting the interface ceilings to **3.0 A, 1.5 A and 4.5 A** keeps every rating and every T1 limit, and lets the counts drop to the rule minimum:

| Rail | Continuous | Fault ceiling (v1.1) | Contacts | Continuous per contact | At the ceiling | T1 limiter |
|---|---|---|---|---|---|---|
| `HAT_5V` | 2.0 A | 3.0 A | 10 | 0.20 A (67 %) | 0.30 A (100 %) | 3.0 A in: 100 %; 1.66 A out: 55 % |
| `HAT_3V3` | 1.0 A | 1.5 A | 5 | 0.20 A (67 %) | 0.30 A (100 %) | 1.15 A: 77 % |
| `GND` (power) | 3.0 A | 4.5 A | 15 | 0.20 A (67 %) | 0.30 A (100 %) | 4.15 A: 92 % |

The 100 % at the ceiling holds only if the TPS2121's 3.0 A is its worst-case maximum trip; if its tolerance takes it higher, the T1 sets ILM lower (board change). The pogo power contacts gain margin with the lower ceilings (spec §5).

### 5.3 Options

| Option | Signals | Power | Positions | Pairs per end | Spare |
|---|---|---|---|---|---|
| v1.0 | DF40C-100 (72 of 96) | DF40C-50 (46 of 46) | 150 | 2 | 24 + 0 |
| Separate, sized | 56 + 8 % → ≥ 61 → DF40C-70 (66 usable) | 31 (30 + `HAT_DET#`) → DF40C-40 (36 usable) | 110 | 2 | 10 + 5 |
| **One connector** | **87 → ≥ 94 with 8 % → DF40C-100 (96 usable)** | (included) | **100** | **1** | **9 (9.4 %)** |
| One connector, v1.0 ceilings | 56 + 37 = 93 | (included) | 100 | 1 | 3 (3.1 %): fails the reserve rule |

Signal count: 16 pair signals + 9 pair grounds (one row, G S S G …) + 21 low-speed + `HAT_DET#` + 8 reserved (every reserved pogo contact gets a cable position) + 2 low-speed grounds = 57; with 30 power contacts, 87. **One DF40C-100 pair per end** gives the smallest total and halves the mating operations. Power and returns face each other across the rows, as in v1.0's power connector. `HAT_DET#` keeps proving the (now single) connector is mated. The DF40C-100DP header's stock risk (Digi-Key 0, Mouser 10,937 on 2026-10-22, BOM line 66) is unchanged.

### 5.4 ESD

| | v1.0 (BOM v1.9) | v1.1 |
|---|---|---|
| High-speed (line 68) | 11 pairs, 22 lines: TPD4E05U06 × 6 | 8 pairs, 16 lines: TPD4E05U06 × 4 |
| Low-speed (line 69) | 29 lines (24 low-speed + 5 reserved with signal-connector positions; the 5 power-connector reserved and 4 unplaced reserved had none): TPD4E05U06 × 8 | 30 lines (22 low-speed + 8 reserved, all with cable positions): TPD4E05U06 × 8 (32 channels) |
| Line out (line 69) | TPD1E10B06 × 2 | none |

v1.0's real low-speed list is 24 (6 + 15 + 3, spec §4.1); the 29 on line 69 is not wrong, it adds the 5 wired reserved contacts. v1.1 protects every wired contact, reserved included, since a later assignment should not need a new piece for ESD.

### 5.5 Reversibility by the coupon

The coupon (ADR 0004) tests the v1.1 fence and a **ground-backed variant**: a `GND` in the adjacent row under each pair contact that now holds a static line (B8, B9, B11, B12, B14, B15 under row A; C12, C14 over row D: 8 positions). If the coupon requires it, those 8 lines move to added columns, about 4 × 20 = 80 positions (estimate), still 20 below v1.0. The pad map freezes only after the coupon, as ADR 0004 already requires.

## 6. Reference HATs

| HAT | Level, v1.1 | Uses | Change against v1.0 |
|---|---|---|---|
| Powerbank (PWR-005) | L1 | Power class F, ID, `HAT_DET#`, `HAT_INT#`, I2C | Input ceiling 3.0 A (the T1 already limits at 3.0 A) |
| NAS (NAS-005, NAS-006) | L3, PCIe | NVMe on the extension (PCIe x1), 2.5G Ethernet on USB 3: two independent 5 Gb/s links, as v1.0 | None |
| Camera (CAM-005) | L3, CSI-2 | CSI-2 4 lanes on the extension, CCI on `I2C_*`, sensor reset on `EXT_SB0`, two camera GPIO | Reset moves from `GPIO0` to `EXT_SB0`; all four GPIO free. **CAM-004 (should, Draft, on-board AI):** an M.2 PCIe accelerator cannot run beside CSI-2; AI runs on the HAT (a sensor module or SoC with its own NPU) or on a USB 3 accelerator behind the HAT's USB 3 port |
| Recorder (REC-005) | L2 | I2S (8 in, 2 out), I2C, GPIO, `HAT_INT#` | Monitoring through `I2S_DOUT` to the HAT's own converter, or the T1's headphone outputs; no line out |
| Oscilloscope (OSC-005) | L1 | ID, `HAT_INT#`, USB | None; SWD at L2 is now an alternate |
| VNA (VNA-005), Radio (RAD-005) | L1 | ID, USB | None |

No reference HAT loses a function it needs. Harder for future HATs: PCIe and CSI-2 at once (a camera recording to NVMe: one of the two moves to USB 3, as UVC or a USB NVMe bridge); a HAT amplifier fed by the T1's analog output (uses I2S and its own converter); SWD and UART B at once (console over CDC-ACM).

## 7. Levels

- **L1 Core:** unchanged in scope; the USB 3 port is the base.
- **L2 Control and audio:** I2S on the DAC clock, four fast GPIO (alternates UART, SPI, SWD), `HAT_RST#`. Line out and the dedicated SWD contacts leave.
- **L3 High-speed extension:** one extension of 5 pairs and 3 sidebands, PCIe x1 or CSI-2 up to 4 lanes, selected by the HAT's declared function. An L3 device supports **both** functions, so the "level or higher" rule stays exact; a device that can offer only one declares L2 and describes the one it has in its manifest.
- One field, one pinout, nested zones (ADR 0003).

## 8. Version

The change moves L3 contacts and functions and shrinks the field, so it is breaking against the v1.0 pad map. It is published as **v1.1 (Draft)**, not v2: v1 was never frozen, no device or HAT exists against it, and the interface revision a HAT writes in its identity record becomes binding only at freeze (§10 of the spec). After freeze, a change of this kind would be v2.

## 9. Sources

- Hirose Electric, *DF40 Series* catalog, May 1, 2020, pp. 1–4, 10 (cached copy `~/.cache/opendap/parts/Hirose/DF40C-100DS-DP-0.4V/df40_catalog.pdf`, sha256 `d49c00f06336c89fbf2eda5d8db7c452d3376bbd53dd1732d125b480159dd370`).
- Texas Instruments product pages, read 2026-10-24: <https://www.ti.com/product/TMUXHS4412>, <https://www.ti.com/product/HD3SS3212>, <https://www.ti.com/product/HD3SS3412>, <https://www.ti.com/product/TMUX646>, <https://www.ti.com/product/TS5MP646>.
- Harwin P70-10X0045R drawing, issue 4 (parts library `Harwin/P70-1000045R.md`); Mill-Max 0965 figures from the [pogo part and geometry research](2026-10-pogo-part-and-geometry.md) §2.
- Board repository: [requirements](https://github.com/opendap-project/board/blob/main/docs/requirements/README.md) (BRD-006, BRD-042, BRD-045, BRD-049), [BOM v1](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md) lines 65–70, 102, 135; [pinout v1](https://github.com/opendap-project/board/blob/main/docs/specs/pinout-v1.md) HAT section; [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md) §2, §4; [pin multiplexing v0](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md) §4.3.
