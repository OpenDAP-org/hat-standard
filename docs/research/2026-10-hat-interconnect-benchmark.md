# HAT interconnect benchmark and proposals

Desk research, 2026-10-21. **Status: research input, not a decision.** Nothing here is confirmed by the maintainers; it is written to feed an ADR. Figures that could not be checked against a primary source are marked **UNVERIFIED**. Power estimates for HATs are engineering estimates, marked as such.

Scope: the second stage of the HAT path (device-side piece in the HAT case → HAT PCB), under HCS-004 (target each interface's theoretical maximum), HCS-005 (justify every compromise) and HCS-006 (one interconnect type for every signal, ideally the same part).

> Research limit: the session's web-search budget ran out partway through. Neoconix, Panasonic narrow-pitch, Teenage Engineering, Pixel camera B2B, Surface Connect pinout, Preci-Dip and CFE were **not** researched to primary sources; anything said about them below is marked UNVERIFIED and should be checked before an ADR.

---

## 0. Summary

| Rank | Proposal | One line |
|---|---|---|
| **1** | **All pogo pins, one standardized pin, flat hard-gold pads on the HAT** | The device-side piece carries a 4-row grid of identical SMT pogo pins. The HAT carries only PCB pads. HS pairs are fenced by ground pins of the same part. This approach best fits swap durability, hobbyist HATs, IP65 and second-sourcing. Its SI is the big unknown, so OpenDAP has to characterize it. |
| 2 | Compression interposer (Z-Ray class) captured in the device-side piece, pads on the HAT | Best SI and lowest Z-height, and the HAT side is still bare pads. Against it: a single source, cost, the precision clamping it needs, and an unknown re-mate life. Keep it as the fallback if Proposal 1 fails the PCIe Gen3 / USB3 measurements. |
| 3 | Fine-pitch blind-mate mezzanine (DF40 / BM28 / LSHM class) | Excellent and documented SI with the highest density. But fine-pitch parts are rated for only ~10–30 mating cycles, which disqualifies them for frequent swapping. Viable only if the maintainers accepts "HATs are swapped rarely" or adds a replaceable wear adapter. |
| — | Rejected: magnetic pogo array as a bought part, USB-C as the carrier, contactless (60 GHz) | See §2. |

---

## 1. Context and targets

### 1.1 What the SoC actually exposes

From the board research ([`board/docs/research/2026-10-soc-dac-pmic.md`](../../../board/docs/research/2026-10-soc-dac-pmic.md), not yet a decision):

| Interface | i.MX 8M Plus (primary candidate) | RK3576 (alternative) | Line rate the interconnect must carry |
|---|---|---|---|
| PCIe | 1× Gen3 x1 | Gen2.1 x1 (combo) | **8 GT/s** (Nyquist 4 GHz) |
| USB 3.x | 2× USB 3.0 DRD (5 Gbps) | USB 3.2 Gen1 (5 Gbps) | 5 Gbps today. **Design the channel for 10 Gbps** (Nyquist 5 GHz) so that a future SoC is not capped by the HAT standard (HCS-004) |
| USB 2.0 | yes | yes | 480 Mbps |
| MIPI CSI-2 | 2× 4-lane D-PHY (per-lane rate **UNVERIFIED**, commonly 1.5 Gbps) | 3× 4-lane | **2.5 Gbps/lane** design target (D-PHY v1.2 max), Nyquist 1.25 GHz |
| Ethernet | RGMII MACs | RGMII | See §3. Proposed via a PCIe or USB NIC on the HAT |

The interconnect should therefore be qualified to **at least 10 Gbps NRZ (5 GHz)** per differential pair. That one target covers every interface above at full rate.

### 1.2 Impedance targets

PCIe 85 Ω diff (Raspberry Pi specifies 90 Ω ±10 % on its PCIe FFC [RPi PCIe]), USB 90 Ω, MIPI D-PHY 100 Ω. A single contact geometry cannot hit all three exactly. The proposal is **one geometry tuned to ~90 Ω ±15 %**. This is a small, justified compromise: the contact is a few mm long, so it is electrically short at 1.25 GHz (CSI) and marginal only at 4–5 GHz (PCIe/USB). To be confirmed by TDR (§4.4).

### 1.3 The two-stage path matters for SI

Main board → **FPC (stage 1)** → device-side piece → **interconnect (stage 2)** → HAT. Raspberry Pi caps its PCIe FFC at **50 mm, 90 Ω ±10 %, Gen2 official, Gen3 "not officially supported"** [RPi PCIe]. That is the closest public precedent, and it says Gen3 over an FFC plus one more contact interface is **not free**.

Implications:

- Stage 1 should be **mated once, at assembly** (permanent FPC connector). If the device-side piece travels with each HAT case, every HAT swap re-mates the FPC connector, and ZIF/B2B FPC connectors are rated for tens of cycles. **Open question for the maintainers:** does the device-side piece stay on the chassis, or travel with each HAT case? This report assumes it stays with the chassis, which makes stage 2 the swap interface.
- Keep stage 1 as short as the mechanics allow. Total channel loss (SoC → HAT device) must fit the PCIe Gen3 and USB budgets. Whether a redriver on the device side is allowed under BRD-006 ("directly from the SoC… multiplexers allowed") is an **open question**.

### 1.4 Realistic power needs (engineering estimates, UNVERIFIED unless cited)

| HAT | Load estimate | Notes |
|---|---|---|
| NAS (NVMe M.2 2230 + PCIe switch + 2.5 GbE NIC) | 6–9 W peak → **~2 A at 5 V** after conversion | NVMe 2230 active peaks of ~3–5.5 W, ~1 W switch, ~1 W NIC (all UNVERIFIED). The RPi M.2 HAT+ supplies "up to 3 A" to the M.2 device [SparkFun M.2 HAT+]. |
| SDR | 1.5–3 W → ≤ 0.6 A at 5 V | RTL-SDR class ~1.5 W; HackRF/LimeSDR-Mini class 2–3 W (UNVERIFIED) |
| Multitrack recorder | 1–4 W | 4× 48 V phantom at ≤10 mA each, generated locally, plus preamps/ADC (UNVERIFIED) |
| Camera | 0.5–1.5 W | Sensor + optics; the ISP is on the SoC |
| Oscilloscope / VNA | 2–4 W | UNVERIFIED |
| **Powerbank → device** | **15 W class (5 V 3 A)** both ways | The rail must be bidirectional. Compare the RPi HAT+ "Power HAT", which is source-only and declares `hat_current_supply` in its EEPROM [RPi HAT+]. |

**Proposed power class:** one bidirectional rail (`VHAT`, nominal 5 V, exact window to be set in the board spec) rated **3 A continuous** in either direction, plus a reserve to 5 A. A separate `3V3_AUX` (≤ 0.5 A) is always on for the ID EEPROM and the HAT's housekeeping (same idea as HAT+ STANDBY [RPi HAT+]). The direction and the current limit are declared in the ID EEPROM and negotiated before `VHAT` is enabled.

### 1.5 Analog audio

Proposal: the **recorder's preamps and ADCs live on the HAT**, and the audio crosses the interconnect as **I2S/TDM** (digital). The reason: running microphone-level analog through ~50–100 mm of FPC next to PCIe and USB3 aggressors, then through a contact interface, is the worst place for a low-noise path. A digital link loses nothing. The only analog lines on the interface are **line-level** ones: L/R out from the T1 DAC to the HAT and L/R in to the T1 codec, if it has an ADC (that is a board decision). They get dedicated `AGND` returns that are never shared with digital return pins.

---

## 2. Prior-art benchmark

| System | Interconnect | What it carries | Lessons |
|---|---|---|---|
| **Motorola Moto Mods** | 16 pogo pins + 4 magnets | **USB 3.1 at 5 Gbps natively**, USB 2.0, I2S, and UniPro/M-PHY ("Group B"). CSI (13 MP raw10 @ 24 fps) and DSI (1080p60) go **through the "Moto High Speed Bridge"**, not as native MIPI lanes [element14 MDK basics; Moto Z wiki]. The bridge implementation is UNVERIFIED. | Proof that a consumer back-mounted pogo interface runs **5 Gbps USB3 for years of swaps**. It also shows Motorola avoided native 4-lane MIPI over pogo and used a bridge, which OpenDAP cannot copy (BRD-005/006). |
| **Essential PH-1** | 2 pogo pins (power only) + magnets | Data over a **60 GHz contactless link** (SiBEAM). Keyssa sued over the concept [Wikipedia Essential Phone; Android Police]. | Contactless avoids contact wear, but it puts a bridge chip in the data path and caps throughput. It violates BRD-005 and HCS-004. Rejected. |
| **Microsoft Surface Connect** | 40-pin magnetic connector | USB 3 (not 3.1), DisplayPort, charging [Framework forum; Notebookcheck] | Magnetic, multi-pin, multi-protocol. The pinout is undocumented (UNVERIFIED) and the part is single-vendor. |
| **iPad Smart Connector** | 3 flush contacts + magnets | Power + a proprietary serial link; the spec is MFi-only [mynexttablet; MacRumors] | Flush pads on one side are the most durable and sealable form. Three pins is far too few. |
| **Fairphone 5** | Internal B2B/FPC connectors for camera/USB modules | Repair-grade, not swap-grade [Wikipedia FP5; iFixit] | B2B connectors work for "a few repairs in a lifetime", not for daily swaps. |
| **Framework expansion cards** | USB-C plug on the card into a USB-C receptacle in the bay | Real USB / USB4 / TBT / DP protocols, no off-label pins [Framework GitHub; Framework community] | USB-C works as an accessory interconnect **only when you carry USB-C protocols**. |
| **PinePhone** | 6 pogo pins on the back | INT, SDA, SCL, DCIN, USB-5V, GND; I2C at 400 kHz [pine64 docs; brixit blog] | The closest hobbyist precedent: pogo + back cover, and a real third-party ecosystem (keyboard, LoRa, Qi). But low-speed only. |
| **Nintendo Switch Joy-Con rail** | 10-pin sliding contacts | UART at 1.8 V, power, detect [dekuNukem RE] | Sliding wipe is self-cleaning, but it is low-speed only. |
| **RED DSMC2** | Pogo modules/displays | Display + power [Wooden Camera] | Pro gear accepts pogo for frequent tool-less swaps. HS details UNVERIFIED. |
| **Raspberry Pi HAT+ / M.2 HAT+** | 40-pin header + 16-pin 0.5 mm FFC for PCIe | PCIe x1 Gen2 (Gen3 unofficial), 5 V on 2 pins × 500 mA, PERST/CLKREQ, PWR_EN, DET_WAKE; ID EEPROM on a dedicated I2C (CAT24C32, WP pin) [RPi PCIe; RPi HAT+] | **Reuse the HAT+ ideas**: a dedicated ID I2C, an EEPROM with WP, STANDBY with only an aux rail, power declared in the EEPROM. Its electrical stage is a precedent for how hard Gen3 over FFC is. |
| **Yokowo docking connector** | 16-pin spring connector | **USB 3.1 10 Gbps with no metal shield: four ground pins do the shielding.** 40,000 cycles, 2 A cont / 3 A peak on power/GND, **IPX7** [EEPower; ECN/EEWorld 2017] | **The key data point**: a commercial pogo-style part rated for 10 Gbps, 40k cycles and IPX7, using ground pins of the same contact type as the shield. This is exactly the hybrid in (f). |
| **PIERS 2016 paper** | Pogo connector, 1.8 mm pitch | Covers USB 3.1 Gen1 + Gen2 bandwidth [Wu, Sun, Chen] | Academic confirmation that pogo geometry can reach 10 Gbps when the ground/return design is right. |
| **Teenage Engineering, Pixel camera modules** | — | Not researched (search budget). Phone cameras use 0.35–0.4 mm B2B connectors (UNVERIFIED). | — |

---

## 3. Pin budget

### 3.1 Signal list

#### High-speed differential pairs

| Interface | Pairs | Sideband (counted under low-speed) |
|---|---|---|
| PCIe Gen3 x1 | 3 (TX, RX, REFCLK) | PERST#, CLKREQ#, WAKE# |
| USB 3.x | 2 (SSTX, SSRX) | — |
| USB 2.0 | 1 (D±) | — |
| CSI-2 4-lane | 5 (4 data + clock) | CAM_MCLK, CAM_RST/PWDN |
| **Baseline** | **11** | |
| Reserve (second USB3 port or second CSI/DSI clock-and-2-lanes) | 2 | |
| **Total HS** | **13 pairs = 26 signal contacts** | |

**Ethernet (proposed compromise, HCS-005):** no native Ethernet on the interface. A 2.5 GbE NIC on the HAT over PCIe (or a 5 Gbps USB3 NIC) beats the SoC's RGMII 1 GbE. Native RGMII would cost ~14 single-ended contacts at 125 MHz DDR, with a PHY on the device side. Native MDI would need the PHY and magnetics in the device. Both cost more contacts and space for less speed. This is justified, not just cheaper.

#### Low-speed single-ended (~44)

| Group | Signals | Count |
|---|---|---|
| ID EEPROM I2C (dedicated, HAT+ style) | ID_SDA, ID_SCL | 2 |
| General I2C | SDA, SCL | 2 |
| Camera I2C (CCI) | CAM_SDA, CAM_SCL | 2 |
| SPI | SCLK, MOSI, MISO, CS0, CS1 | 5 |
| UART | TX, RX, RTS, CTS | 4 |
| I2S/TDM | MCLK, BCLK, FS, DOUT0/1, DIN0/1 | 7 |
| SWD/JTAG | TCK/SWCLK, TMS/SWDIO, TDI, TDO/SWO, nRESET | 5 |
| GPIO (PWM-capable) | | 8 |
| Control | PERST#, CLKREQ#, WAKE#, CAM_MCLK, CAM_RST, HAT_DET#, HAT_INT#, HAT_PWR_EN, VHAT_DIR/FAULT | 9 |
| **Total** | | **44** |

#### Power and analog

| Group | 3 A class | 5 A class |
|---|---|---|
| VHAT (bidirectional) | 4 | 6 |
| VHAT return GND | 4 | 6 |
| 3V3_AUX | 2 | 2 |
| Line audio L/R out, L/R in | 4 | 4 |
| AGND (dedicated) | 2 | 2 |

The power counts assume ~2 A-rated contacts derated to ~1 A each with one spare. A 0.3 A mezzanine contact multiplies them by ~3.

### 3.2 Contact totals and size per approach

| Approach | HS incl. grounds | Power | LS + analog + LS grounds | **Total** | Resulting array (estimate) |
|---|---|---|---|---|---|
| (a)/(f) Pogo grid, ground-fenced pairs (≈4 contacts/pair) | 52 | 10 | 44 + 6 + 8 | **~120–128** | 4 rows × 32 at **1.27 mm** ≈ **41 × 4 mm** of contacts (≈ 45 × 8 mm with housing). At 1.0 mm pitch: ≈ 32 × 3 mm |
| Same, but **2 rows only** (the current idea in the requirements context) | | | | ~128 | 2 × 64 at 1.27 mm ≈ **81 mm**. That is too long for an 85.6 mm-class body. **Two rows only work with a reduced signal set, or as two banks.** |
| (c) Mezzanine, linear GSSG (3n+1) | 40 | 22 (0.3 A/pin) | 44 + 6 + 6 | **~118 → 120-pos** | DF40 0.4 mm 2×60 ≈ 26 × 4 mm (UNVERIFIED body size) |
| (d) Interposer, 0.8 mm grid | ~52 | ~16 (current per pin UNVERIFIED) | ~58 | **~126** | 12 × 11 at 0.8 mm ≈ **10 × 9 mm**, 0.33 mm body [ZA8H] |
| (e) USB-C used as generic 24-contact blocks | — | — | — | ~126 | **≥ 6 connectors** (~9 mm wide each), and only 4 SS pairs per connector are HS-grade → impractical |

---

## 4. Candidate approaches

### 4.1 Comparison matrix

| | (a) All pogo, HS-rated pogos | (b) Magnetic pogo array | (c) Blind-mate mezzanine | (d) Compression interposer | (e) USB-C carrying everything | (f) Same pogo part, ground-fenced HS (refines a) |
|---|---|---|---|---|---|---|
| **Rated SI** | Generic pogo: no published S-params. HS-designed parts: **10 Gbps** (Yokowo 16-pin USB 3.1, 4 GND pins as shield) [EEPower]; PIERS 2016 1.8 mm pitch to USB 3.1 Gen2 [PIERS]; coax pogo 50/75 Ω, Promax ≥0.7 mm dia, no GHz figures [Promax]. "Up to 10 Gbps" for HF pogo designs [DigiKey basics] | Off-the-shelf magnetic pogo connectors are typically power/USB2-class (UNVERIFIED) | **DF40 10 Gbps** (USB 3.2 Gen2) [Avnet DF40]; **BM28 10–20 Gbps** [Future BM28]; **LSHM 14 Gbps NRZ** [Samtec LSHM]; SlimStack floating 6 Gbps [EDN] | **ZA8H up to 56 Gbps** [Electropages; Mouser ZA8] | Per USB-C spec, SS pairs are rated for USB4 when the part is rated (UNVERIFIED per part) | Same as (a); OpenDAP must measure |
| **Current/contact** | Mill-Max 854/855: **2 A cont / 3 A peak, 20 mΩ max** [Mill-Max] | Typ. 1–3 A (UNVERIFIED) | DF40 **0.3 A** [Avnet DF40]; BM28 0.3 A signal / 5 A power contacts [Future BM28]; LSHM 2 A [Samtec LSHM] | UNVERIFIED (likely < 1 A) | GCT USB4105: VBUS 5 A total, GND 6.25 A, others **0.25 A** [GCT] | As (a) |
| **Mating cycles** | Mill-Max **1,000,000** [Mill-Max]; Yokowo 40,000 [EEPower] | Typ. 5k–10k+ (UNVERIFIED) | **BM28: 10** [Future BM28]; SlimStack 0.35 mm: **10–30** [Mouser SlimStack]; DF40 ~30 (**UNVERIFIED**); LSHM UNVERIFIED | Few re-mates before replacement (**UNVERIFIED**) | GCT USB4105 **20,000** [GCT] | As (a) |
| **Sealing (IP65)** | Good. The HAT side is bare pads; the pogo side can use individually sealed pins (Yokowo IPX7-class) plus a perimeter gasket | Good (flush) | Poor. Open receptacle cavities; needs a gasket around the mated pair | Medium. Gasket around the clamp | Easy: IPX7/IPX8 receptacles are common (UNVERIFIED per part) | Good |
| **Z-height** | Pogo 2–6 mm (Mill-Max 855 < 5.84 mm engaged [Mill-Max]); HAT side ≈ 0 | 3–6 mm + magnets | **0.6 mm (BM28)**, 1.5–4 mm (DF40), 5–12 mm (LSHM) | **0.33 mm body** [ZA8H] + clamp | ~3.2 mm receptacle + plug engagement, vertical mating needs more (UNVERIFIED) | As (a) |
| **Alignment / tolerance** | Oversized pads absorb ±0.2 mm; dowel pins in the case | Self-aligning magnets | Lead-in guides; BM28 self-aligns ±0.3 mm [Future BM28] | **Precision**: dowels, stiffener, controlled compression (≈0.2 mm deflection, 30 g/pin) [ZA8H] | Tight (±0.1 mm) unless floating | As (a) |
| **Force** | **Spring force adds up**: 60 g at mid-stroke for Mill-Max 855 [Mill-Max] → 128 × 60 g ≈ **7.7 kgf**. Needs a latch or screws; magnets alone won't hold it | Magnet sized to the spring force | Low clamp, high insertion force (LSHM 4–6× typical [Samtec LSHM]) | 128 × 30 g ≈ 3.8 kgf, held by screws | Low | Choose ~25–35 g pins → ~3–4.5 kgf |
| **Cost (UNVERIFIED)** | ~$0.10–0.50 per pin at distributor prices; **HAT side: pads only** | Custom, high | ~$2–5 per pair | ~$20–60+ per interposer + hardware | <$1 per receptacle | As (a) |
| **Availability / second source** | **Excellent**: discrete pogo pins come from Mill-Max, Preci-Dip, CFE, Yokowo, Same Sky, Promax and many LCSC vendors. Standardize the pin **geometry**, not a vendor | Arrays this size are custom → single source | Hirose/Molex/Samtec are single-vendor footprints (pin-compatible clones UNVERIFIED) | **Samtec single source**; Neoconix not researched | Excellent | Excellent |
| **Ease for third-party HAT makers** | **Best**: pads only. Any fab, no extra BOM line. JLCPCB/PCBWay ENIG works; hard gold for durability | Must buy the mating magnetic part | Must place a 0.4 mm-pitch part (JLCPCB can; hobby hand-soldering cannot) | Pads only (best), but the clamp spec is tight | Easy part, but no meaningful pinout | Best |
| **Carries at full rate** | PCIe Gen3, USB3 5 G, CSI 2.5 G: **likely with a fenced design, unproven.** USB 10 G: plausible per Yokowo | USB2 / LS / power only | All of them (documented) | All of them (documented) | USB3/USB4 as USB only. **PCIe and CSI cannot pass natively** (they would need USB4 tunnelling, which the SoC lacks and which violates BRD-005/006) | Same as (a) |
| **HCS-006** | ✅ one part | ✅ | ✅ one part (LSHM is hermaphroditic: literally the same part on both sides) | ✅ | ✅ but useless | ✅ one part |

### 4.2 Notes per approach

**(a)/(f) Pogo + pads.** Put the pogos on the device-side piece and plain pads on the HAT. This puts the wear on the one OpenDAP-controlled, FPC-replaceable part, and makes every HAT free of connector BOM. HS pairs use a **G-S-S-G pattern in the grid, with a ground pin of the same part on every side of a pair**. That is the Yokowo technique, which reached 10 Gbps without a metal shield [EEPower]. Under HCS-006 this means all pogo, **one** pin for signal, ground and power. A coaxial pogo for HS would be a second part: still "all pogo", but it loses "the same part", so avoid it unless measurements force it.

There is no first-make/last-break with equal-height pins under a Z-axis mate. Sequencing is done electrically: `VHAT` stays off until `HAT_DET#` is asserted and the EEPROM has been read. There is soft-start on both sides. The interface is powered-down hot-swap only, so PCIe hot-plug is not required.

**(b) Magnetic pogo array.** Magnets belong in the **HAT case mechanics** (retention and alignment), not in a bought connector. Magnetic pogo connectors at 120+ contacts are custom and single-source. Magnets near the speaker, the compass or the HAT's own magnetics add design constraints. Steel debris collects on the magnets in the field.

**(c) Mezzanine.** Best documented SI and density. But a **10–30-cycle** rating (BM28 10 [Future BM28]; SlimStack 10–30 [Mouser SlimStack]) is incompatible with "frequent swapping". A user who swaps HATs weekly reaches 50 cycles in a year. LSHM has better mechanics and is hermaphroditic, but its 5–12 mm stack and single source count against it.

**(d) Interposer.** Electrically the best (56 Gbps class, 0.33 mm) [Electropages]. It keeps the HAT side as bare pads. Against it: a single source, cost, a clamp that third-party case makers must reproduce precisely, and an unknown re-mate life (UNVERIFIED). Neoconix PCBeam is an alternative interposer family (not researched; UNVERIFIED).

**(e) USB-C.** Excellent part: 20,000 cycles [GCT], sealable, everywhere. But a USB-C connector has 4 HS pairs and very few low-speed pins (CC, SBU and the USB2 pins are rated 0.25 A [GCT]). Carrying ~126 signals needs ≥6 connectors. Off-label pin use also creates a hazard if a user ever plugs a real USB-C cable in. Framework works only because it carries real USB-C protocols [Framework GitHub]. Rejected.

---

## 5. Proposals (ranked; proposals, not decisions)

### Proposal 1: Single standardized pogo pin on the device side, flat hard-gold pads on the HAT, ground-fenced grid

- **Parts:** an OpenDAP "pin envelope" spec: SMT, gold-plated, ~1.27 mm pitch (1.0 mm as a stretch goal), working stroke ~0.5–0.7 mm, 25–35 g at working height, ≥2 A, ≥100k cycles. Candidates to qualify: Mill-Max 0906/085x and 854/855 family (1.27 mm, 1M cycles, 2 A/3 A, 20 mΩ) [Mill-Max]; Yokowo SPC spring connectors (sealed variants) [Yokowo/suasnews]; Preci-Dip and CFE (UNVERIFIED); LCSC-stocked generic pins as the DIY second source. **Qualify at least two vendors against the same envelope.**
- **Layout:** 4 rows × ~32 at 1.27 mm (~41 × 4 mm active) or two 2-row banks. HS pairs sit on the rows nearest the FPC tail, in G-S-S-G with ground pins on all sides. Power gets 4–6 + 4–6 pins at the array ends. Analog sits at the opposite end from HS, with dedicated AGND.
- **HAT side:** pads ≥ Ø0.8 mm on a 1.27 mm grid, hard gold recommended (ENIG acceptable for DIY/low-cycle use; its life is UNVERIFIED). OpenDAP publishes KiCad footprints, a test-coupon stack-up for JLCPCB/PCBWay standard impedance control, and pad/via rules (no via-in-pad stubs on HS pairs).
- **Mechanics:** 2 dowel pins for alignment ±0.1 mm. The HAT case latches or screws to react ~3–4.5 kgf, and the same clamp compresses the IP65 perimeter gasket around the pad field. DIY and Commercial use the same geometry, differing only in tolerance class (T1-004).
- **Sacrifices:**
  1. No vendor S-parameters, so OpenDAP owns characterization.
  2. Possibly ~90 Ω for every pair instead of exact 85/90/100 Ω (§1.2). This is justified because the contact is electrically short.
  3. If measurement shows insufficient Gen3 margin, PCIe link-trains down to Gen2. Today this is a **risk, not a decision**. Under HCS-005 it would be acceptable only with measured evidence.
  4. No native Ethernet (§3.1). Justified because PCIe/USB NICs are faster.
- **Risks:** total spring force; fretting corrosion and contamination on open pads (the gasket mitigates it); pad wear on ENIG; crosstalk between CSI and PCIe; ESD on exposed pads when the HAT is off (TVS on the device side adds capacitance, so budget for it); the stage-1 FPC loss eating the Gen3 budget.

### Proposal 2: Compression interposer captured in the device-side piece, pads on the HAT

- **Parts:** Samtec Z-Ray ZA8H (0.8 mm, 0.33 mm body, up to 56 Gbps, ~30 g/pin, 0.2 mm deflection) [ZA8H; Electropages]. Neoconix as a possible second source (UNVERIFIED).
- **Sacrifices:** single sourcing (a conflict with the availability constraint, justified only if Proposal 1 fails SI); a precision clamp; the interposer becomes a replaceable wear item with an unknown life.
- **Why rank 2:** it is the SI-safe fallback that keeps the third-party HAT side identical (pads), so a switch from P1 to P2 would not orphan HAT designs if the pad grid is chosen compatibly. **Suggest picking a pad grid (0.8/1.0/1.27 mm) that both P1 and P2 can address.**

### Proposal 3: Fine-pitch mezzanine (only with a durability concession)

- **Parts:** Hirose DF40 (0.4 mm, 10 Gbps, 1.5–4 mm stack, 0.3 A) [Avnet DF40]; Hirose BM28 (0.6 mm stack, 10–20 Gbps, 5 A power contacts, ±0.3 mm self-align, **10 cycles**) [Future BM28]; Samtec LSHM (hermaphroditic, 14 Gbps, 2 A, 5–12 mm) [Samtec LSHM].
- **Sacrifices:** mating durability (10–30 cycles) is the big one, justified only if the maintainers rules that HAT swaps are rare. Mitigation: a cheap replaceable adapter board that takes the wear. Power needs ~3× the contacts. JLCPCB can assemble the HAT side, but hobbyists cannot hand-solder it.
- **Why rank 3:** it is the only option with fully documented SI today. It fails the "frequent swapping" constraint unless that is relaxed.

### Not proposed

Magnetic pogo array as a bought part, USB-C as carrier, contactless 60 GHz (Essential), and bridged MIPI (Moto Mods): reasons in §2 and §4.2.

---

## 6. Validation plan (applies to whichever proposal goes forward)

| Step | What | Pass criterion (proposed, to be set in the spec) |
|---|---|---|
| 1 | **2×-thru de-embedding coupons** (IEEE P370-style) on JLCPCB/PCBWay standard stack-ups: HAT pad coupon + device-piece coupon + interconnect | Repeatable fixture removal |
| 2 | **TDR** per pair, mated | 90 Ω ±15 % across the contact region |
| 3 | **4-port VNA S-parameters to ≥10 GHz**: SDD21, SDD11, NEXT/FEXT to the nearest aggressor pairs (CSI↔PCIe↔USB) | Contact-only IL ≤ ~1 dB and RL ≥ ~10–15 dB to 5 GHz (proposed numbers, UNVERIFIED against the PCIe CEM / USB budgets; derive the final values from those specs) |
| 4 | **End-to-end channel** with the real stage-1 FPC: PCIe Gen3 eye/BER (LTSSM reaches Gen3 and stays; AER error counts under fio load on NVMe), USB3 compliance-pattern eye + iperf/fio, CSI-2 with sensor test pattern at max lane rate (CRC/ECC error counters = 0 over ≥1 h) | Full rate with zero corrected-error growth |
| 5 | **Durability**: robotic mate/unmate ≥10k cycles (target 100k), LLCR (EIA-364-23) every 1k cycles, on ENIG and hard-gold pads | ΔR ≤ 10 mΩ (proposed) |
| 6 | **Current / temperature rise** (EIA-364-70) at 3 A and 5 A on VHAT, both directions | ΔT ≤ 30 °C (proposed) |
| 7 | **Environmental**: IP65 test of the mated HAT case (IEC 60529), plus salt/humidity fretting and vibration with a continuity monitor (no >1 µs dropouts, like Mill-Max's 50 G/10 G test [Mill-Max]) | IP65; no discontinuity |
| 8 | **Analog**: noise floor and crosstalk on the line-audio pins with PCIe/USB3 at full load (an AP or a QA40x-class analyzer) | Line-level THD+N / crosstalk budget to be set by the board team |
| 9 | **Hobbyist path**: a third party orders a reference HAT from JLCPCB with standard options and passes steps 2–5 | Pass with no special process |

---

## 7. Open questions for the maintainers

1. Does the device-side piece stay with the chassis (so stage 2 is the swap point), or travel with each HAT case (so the FPC connector re-mates on every swap)?
2. Expected swap frequency. This decides whether Proposal 3 is viable at all.
3. Are redrivers on the device-side piece allowed under BRD-006?
4. VHAT nominal voltage window and current class (3 A vs 5 A).
5. Is IP65 required **with the HAT removed** (the device sealed on its own) or only for the mated assembly?
6. Line-level analog on the interface: yes or no.

---

## Sources

- [RPi PCIe] Raspberry Pi, *Raspberry Pi Connector for PCIe* (RP-008298-DS): <https://pip-assets.raspberrypi.com/categories/892-raspberry-pi-5/documents/RP-008298-DS-1-pcie-connector-standard.pdf> (via <https://datasheets.raspberrypi.com/pcie/pcie-connector-standard.pdf>); summary at CNX Software: <https://cnx-software.com/?p=122019>
- [RPi HAT+] Raspberry Pi, *HAT+ Specification*: <https://datasheets.raspberrypi.com/hat/hat-plus-specification.pdf>
- [SparkFun M.2 HAT+] <https://www.sparkfun.com/raspberry-pi-m-2-hat.html>
- [element14 MDK basics] Moto Mods Development Basics: <https://community.element14.com/products/manufacturers/moto-mods/b/blog/posts/moto-mods-development-basics> ; MDK datasheet (Farnell): <https://www.farnell.com/datasheets/2078628.pdf>
- [Moto Z wiki] <https://en.wikipedia.org/wiki/Moto_Z> ; Tom's Hardware: <https://www.tomshardware.com/news/moto-z-force-smartphones-modules,32029.html>
- [Wikipedia Essential Phone] <https://en.wikipedia.org/wiki/Essential_Phone> ; [Android Police] <https://www.androidpolice.com/2017/10/16/essential-sued-modular-accessory-technology/>
- [Framework forum] Surface Dock connector: <https://community.frame.work/t/surface-dock-connector/8771> ; [Notebookcheck] <https://www.notebookcheck.net/Microsoft-Magnetic-USB-Type-C-possible-successor-of-the-Surface-Connector.388086.0.html>
- [mynexttablet] <https://mynexttablet.com/smart-connector-on-ipads-and-compatible-accessories/> ; [MacRumors] <https://forums.macrumors.com/threads/ipad-pro-smart-connector-potential.1925862/>
- [Wikipedia FP5] <https://en.wikipedia.org/wiki/Fairphone_5>
- [Framework GitHub] <https://github.com/FrameworkComputer/ExpansionCards> ; [Framework community] <https://community.frame.work/t/are-the-usb-c-expansion-cards-actually-any-different/47232>
- [pine64 docs] <https://pine64.org/documentation/PinePhone/Accessory/> ; [brixit blog] <https://blog.brixit.nl/making-a-backcover-extension-for-the-pinephone/> ; <https://tuxphones.com/pinephone-modular-expansion-boards-pogo-pins-i2c/>
- [dekuNukem RE] <https://github.com/dekuNukem/Nintendo_Switch_Reverse_Engineering/>
- [Wooden Camera] <https://woodencamera.com/collections/red/products/red-male-pogo-to-male-lemo-lcd-evf-cable-24-red-dsmc2>
- [EEPower] Yokowo USB 3.1 connector: <https://eepower.com/new-industry-products/usb-3-1-connector-achieves-10gbps-data-transfer-without-a-placing-shield/> ; [ECN/EEWorld] <https://www.eeworldonline.com/news/2017/10/usb-31-connector-yokowo>
- [Yokowo/suasnews] <https://www.suasnews.com/2026/10/yokowos-evolving-connector-product-lineup/>
- [PIERS] Wu, Sun, Chen, "A novel pogo pin connector design for high speed USB3.1 operations", PIERS 2016: <https://ntut.elsevierpure.com/en/publications/a-novel-pogo-pin-connector-design-for-high-speed-usb31-operations/>
- [Promax] <https://promaxpogopin.com/spring-loaded-pin/coaxial-pogo-pins>
- [DigiKey basics] <https://www.digikey.bg/en/articles/the-basics-of-pogo-pin-connectors>
- [Mill-Max] Series 854/855 .050" spring-loaded connectors: <https://mill-max.com/sites/default/files/external/assets/2017-12/Thru-hole-SMT%20Spring%20%20Connectors.pdf> ; 0900/0906 1M cycles: <https://www.mouser.com/new/mill-max/millmax_0900_0906/> ; 085x high-current: <https://www.eenewseurope.com/en/rugged-high-current-spring-pins-rated-at-1000000-cycles-take-9a/>
- [ZA8H] Samtec ZA8 datasheet (Mouser): <https://mouser.com/datasheet/2/527/za8-1349575.pdf> ; [Electropages] <https://www.electropages.com/2015/03/samtec-ultra-low-profile-high-density-interposers-provide-ultimate-design-flexibility>
- [Samtec LSHM] <https://www.samtec.com/products/lshm>
- [Avnet DF40] <https://my.test3.avnet.com/abacus/manufacturers/m/hirose-electric/products/df40-series-0.4mm-connectors> ; RS stock example: <https://uk.rs-online.com/web/p/pcb-sockets/7726752>
- [Future BM28] <https://www.futureelectronics.com/resources/featured-products/hirose-bm28-series-hybrid-board-to-fpc-connectors> ; Mouser: <https://www.mouser.it/hirose-bm28-connectors>
- [Mouser SlimStack] <https://www.mouser.com/new/molex/molex-slimstack-armor/> ; [EDN] <https://edn.com/molex-expands-slimstack-floating-connector-line/>
- [GCT] USB4105: <https://gct.co/connector/usb4105>

**Not checked (budget):** distributor stock and price for the specific pogo SKUs, the DF40 mating-cycle figure, the Z-Ray re-mate life and price, the Neoconix and Panasonic narrow-pitch families, and the i.MX 8M Plus CSI per-lane maximum. These must be verified before an ADR.
