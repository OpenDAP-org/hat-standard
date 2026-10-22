# Pogo part and field geometry for HAT interface v1

- **Status:** Research input, not a decision. It edits no specification, ADR or BOM line. Figures without a primary source are marked **UNVERIFIED**. Derived numbers (sums, spans, forces) are computed here from the cited figures and say so.
- **Date:** 2026-10-22. All stock and lead-time figures were read on this date.
- **Question:** which spring contact and which field geometry carry the 100-position HAT field (ADR 0002 amendment: 4 × 25, pitch and part open; [interface v1](../specs/interface-v1.md) §10 item 7).
- **Inputs:** [interface v1](../specs/interface-v1.md) (§2 zones, §4.4 counts, §5 power, §7.1 field); [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md), [ADR 0002](../adr/0002-interface-v1-native-high-speed-management-over-usb.md) and its amendments, [ADR 0003](../adr/0003-compatibility-levels.md); requirements HCS-004, HCS-006, HCS-012, HCS-014, HCS-018 (Withdrawn), HCS-023 ([`../requirements/README.md`](../requirements/README.md)); [interconnect benchmark](2026-10-hat-interconnect-benchmark.md); [interface reduction](2026-10-interface-reduction.md); t1 ADR 0003 (body 59.5 × 94.3 × 24 mm), T1-010, T1-028, T1-029, T1-038; case requirements (CASE-015 wall thickness Withdrawn, CASE-024); board [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md); parts library sheet `Harwin/P70-1000045R.md`.
- **Research limit:** the web-search budget of this session was exhausted before the survey. Sources were reached by direct URL only: Harwin and Samtec sites, the Mill-Max metric catalog mirrored by DigiKey (mill-max.com returned HTTP 503), and DigiKey product pages. Yokowo, Interplex, Würth, CnC Tech, Preci-Dip multi-way strips and a Harwin "P50" family could not be checked (§2.3).

## 0. Summary

1. **Part:** Mill-Max **series 855**, SMT, double-row spring-loaded strips on a 1.27 mm grid (rows 1.27 mm apart inside a strip), 0.48 mm plungers, 0.7 mm mid stroke, 2 A continuous and 3 A peak, 20 mΩ maximum, up to 100,000 cycles, 1 pF maximum (Mill-Max metric catalog p. 19 and 19.1). It is the only stocked spring part found that reaches 1.27 mm in both directions; the Harwin P70-1000045R on the BOM needs a Ø 2.20 mm pad and cannot.
2. **Geometry:** 4 × 25 positions built from **four strips**: two for rows A–B (the high-speed band), two for rows C–D, split at the boundary between the L1 core zone and the L2/L3 zone. All contacts sit on one 1.27 mm lattice; the gap between the two strip pairs is 2.54 mm (one empty row), and so is the gap at the zone boundary (one empty column). Contact span **31.75 × 5.08 mm**, strip envelope about **33.4 × 6.9 mm** (derived, §4). It fits across the 59.5 mm body with room for alignment pins and a retention feature at each end.
3. **Zones:** each zone maps onto whole strips, so an L1 device-side piece carries only the two core strips (32 contacts) and the core zone stays at the same place on every piece (ADR 0003).
4. **Power:** the rating equals the P70's 2.0 A, so the contact counts and margins of interface v1 §5 (BRD-049) are unchanged.
5. **Second source:** no second 1.27 mm double-row spring strip was found at an authorized distributor in this session. The pad map is chosen so that any double-row 1.27 mm strip whose housing reaches at most 1.27 mm past its outer contacts drops in. A second part must be qualified before the pad map is frozen (§7). **Samtec FSI** (1.00 mm one-piece compression interface, published 24 Gb/s differential rating) is the documented-SI fallback, but it is not pad-compatible and is a compression contact, not a pogo pin (§2.2).
6. **High speed:** no spring-pin maker publishes S-parameters for these parts. A ground-fenced 1.27 mm grid at 5 Gb/s (USB 3 Gen 1, PCIe 2.1, CSI-2 at 2.5 Gb/s per lane) is plausible, not proven; a test coupon is required before freeze (§5).
7. **Force:** about 59 N for a full L3 field at mid stroke (about 47–71 N over a ±0.25 mm stack tolerance), about 19 N for an L1 piece (derived, §6). The P70 at the same count gives about 88 N.

## 1. Constraints used

| Constraint | Source | Effect on the choice |
|---|---|---|
| One interconnect type, the same part for every contact | HCS-006, ADR 0001 | Power, ground, analog and high-speed contacts are one part |
| Springs on the device side, flat pads on the HAT | HCS-014 (Draft), ADR 0001 | The HAT pad map is the standard; the spring part can change behind it |
| 100 positions, 4 × 25 | ADR 0002 amendment | Field size |
| Nested zones L1 ⊂ L2 ⊂ L3, contiguous, core zone at a fixed place | ADR 0003, interface v1 §2, §7.1 | Strip split follows the zone boundary |
| Each pair fenced by ground columns | interface v1 §7.1 | Row A/B of one strip carry the pairs |
| Power margins (≤ 70 % continuous, ≤ 100 % at the fault ceiling, one contact open) | BRD-049, interface v1 §5 | Contact rating ≥ 2.0 A keeps the v1 counts |
| Off-the-shelf parts with real availability, no custom parts | T1-010, T1-028, T1-029, T1-038 | Excludes custom pogo housings and made-to-order arrays |
| Body 59.5 × 94.3 × 24 mm | t1 ADR 0003 | Field width across the body ≤ about 55 mm (§3) |
| Case wall thickness | CASE-015 **Withdrawn**; no value set | Assumed 1.5–2.0 mm for the fit check (**UNVERIFIED**) |
| Slider lock | HCS-018 **Withdrawn** (2026-10-21) | No lock geometry is fixed; §6 gives the force any retention must carry |

## 2. Survey of spring contacts

### 2.1 Parts checked against a primary source

| Part | Form | Grid / pitch | Device-side pad | Heights and travel | Force | Current, resistance | Cycles | High-speed data | Mount | DigiKey, 2026-10-22 |
|---|---|---|---|---|---|---|---|---|---|---|
| **Mill-Max 855-22-0XX-30-001101** | Double-row strip, 04–40 contacts | 1.27 × 1.27 mm | Ø 1.02 mm (catalog) | Free height 4.98 mm; mid stroke 0.7 mm, max 1.4 mm; housing 3.05 mm wide, length (contacts/2 × 1.27) + 0.38 mm | 25 gf initial, 60 gf at 0.7 mm | 2 A cont., 3 A peak; 20 mΩ max | Up to 100,000 | 1 pF max capacitance only | SMT; 855 also on tape and reel (`-002191`, 660 per reel) | 40-pos: 33 in stock, 4 weeks; 20-pos: 78, 4 weeks; 16-, 18-, 32-pos: 0 in stock, factory order (16-pos 4 weeks) |
| Mill-Max 854-22-0XX-30-001101 | Single-row strip, 02–20 | 1.27 mm | Ø 1.02 mm | As 855; housing 2.21 mm wide | As 855 | As 855 | Up to 100,000 | As 855 | SMT | 2-pos: 37 (variant `-041101`) |
| Mill-Max 838-22-0XX-30-001101 | Double-row strip, 04–20 | 2.00 mm | Ø 1.68 mm | 0.7 mm mid, 1.4 mm max | 25 / 60 gf | 2 A / 3 A; 20 mΩ | Up to 1,000,000 | 1 pF max | SMT, tape and reel | 20-pos: 52 in stock, 4 weeks |
| Mill-Max 0965-0-15-20-80-14-11-0 | Discrete pin | Pad Ø 1.83 mm, so ≥ about 2.0 mm pitch | Ø 1.83 mm | Working 1.93–2.54 mm, recommended 2.24 mm | 30 gf initial, 115 gf mid | 2 A | 1,000,000 | None | SMT | 41,785 in stock, 4 weeks |
| Harwin P70-1000045R (BOM line 67) | Discrete pin | Pad Ø 2.20 mm, so 2.54 mm pitch | Ø 2.20 mm | Working 2.70–3.00 mm, normal 2.90 mm | 0.88 N ± 0.29 N at 2.90 mm | 2.0 A; 30 mΩ max (part drawing; family sheet: 50 mΩ max) | 10,000 | None | SMT, tape and reel | 5,006 in stock, 24 weeks |
| **Samtec FSI-150-03-G-D** | One-piece compression interface, 100 positions | 1.00 mm, 2 rows 3.55 mm apart (mating pads) | Samtec footprint | Body 3.00 mm; free 3.62 mm, so about 0.62 mm compression (derived) | 30 gf minimum normal force | 2.0 A (spec), 2.8 A (catalog, 2 pins powered); LLCR Δ ≤ 15 mΩ | 10,000 without environment, 100 with | **Published**: 3 mm body, −3 dB at 12 GHz differential, rated 24 Gb/s | SMT | Listed in stock, quantity not shown; `-AD` variant 0, 10 due 2026-11-09, 4 weeks |

Notes:

- **Mill-Max 855.** The catalog states the strips are "modular contacts for use on 1,27 grid". Two strips butted end to end leave 1.65 mm between their end contacts (housing overhang 0.825 mm per end, derived from the length formula), and two strips side by side leave at least 1.78 mm between rows (overhang 0.89 mm per side, derived from the 3.05 mm width). A single 4-row 1.27 mm grid is therefore not possible with this part; §4 builds the field around that. Plunger tip Ø 0.48 mm (catalog: "standard 0,48 dia. spring pin plungers", p. 21.3). The working height at mid stroke, 4.28 mm, is derived (4.98 − 0.7).
- **Suffixes.** DigiKey lists `-001101`, `-004101`, `-041101` and `-051101` for the same contact counts; the catalog explains only `-001101` (tube) and `-002191` (tape and reel). The others are **UNVERIFIED**.
- **Samtec FSI.** The mating board needs 30 µin gold pads, 120–200 Knoop (ASTM B488 type 2, grade C), and a stiffener is recommended (mating-card layout FSI-XXX-XX-XX-X, rev. F). Contacts per row: 05, 10, 20, 30, 50 (catalog FSI06_10).
- **Harwin family sheet** (PD012): pogo pins 1 A or 2 A, 50 mΩ max, 10,000 cycles, 0.39–1.32 N at working height. No high-speed data.

### 2.2 Fit against the constraints

| Part | 1.27 mm field possible | Fits 4 × 25 across the body | One part for all contacts | Second source at the same pads | Availability | Verdict |
|---|---|---|---|---|---|---|
| Mill-Max 855 | Yes (two-row strips, §4) | Yes | Yes | None verified (§7) | Authorized distributor, thin stock, 4-week factory lead time | **Recommended** |
| Mill-Max 838 (2.00 mm) | No | Tight (§3) | Yes | None verified | As 855 | Not preferred: larger field, same sourcing |
| Harwin P70, Mill-Max 0965 (discrete) | No (2.54 mm) | No; only along the 94.3 mm length | Yes | **Yes**: both are 2 A SMT pins on a 2.54 mm grid | Best of the set | Fallback geometry if the body orientation allows it (§3) |
| Samtec FSI-150 | No (1.00 mm, 2 × 50) | Across the body only without screw inserts (§3) | Yes | No (Samtec only) | Distributor stock thin | **SI fallback**, not a drop-in: different pad map, hard-gold pads on every HAT, and a stamped compression contact rather than a pogo pin, which ADR 0001 would have to cover |

### 2.3 Not checked (UNVERIFIED)

- **Preci-Dip:** DigiKey lists only discrete spring pins (90625-AS, 90629-AS, 90637-AS; 5 A, 10,000–20,000 cycles); a multi-way 1.27 mm strip was not found.
- **CnC Tech, Würth Elektronik, Yokowo, Interplex:** product pages not reachable this session. The benchmark cites a Yokowo 16-pin spring connector at 10 Gb/s, 40,000 cycles, which is a fixed multi-pin part and not a field building block.
- **Harwin P50:** no such family was found on the Harwin pages read; Harwin's pogo range is discrete pins only (PD012).
- **LCSC generic 1.27 mm double-row pogo strips:** commonly offered, but no part was located through the JLCPCB parts API, so stock and authorized status are unknown.

## 3. Geometries against the body

Contact span = (columns − 1) × pitch by (rows − 1) × pitch, uniform grid, derived. The body is 59.5 mm wide; with 1.5–2.0 mm walls (**UNVERIFIED**) the inner width is about 55.5–56.5 mm, and each end of the field needs room for an alignment pin and a retention feature (about 4–6 mm per end, engineering estimate).

| Geometry | Pitch | Contact span (mm) | Across the 59.5 mm width | Along the 94.3 mm length | Part | Zones |
|---|---|---|---|---|---|---|
| 4 × 25 uniform | 1.27 | 30.48 × 3.81 | Fits | Fits | None stocked (855 strips cannot butt at 1.27 mm) | Contiguous |
| **4 × 25 in four 855 strips** (§4) | 1.27, one empty row and one empty column | **31.75 × 5.08** | **Fits** (envelope 33.4 × 6.9) | Fits | Mill-Max 855 | Contiguous, one strip pair per zone group |
| 5 × 20 | 1.27 | 24.13 × 5.08 | Fits | Fits | Odd row count does not map onto two-row strips | Needs a new zone map |
| 2 × 50 | 1.27 | 62.23 × 1.27 | No | Fits | 855 strips (max 20 columns each) | Contiguous, long |
| 2 × 50 | 1.00 | 49.0 × 3.55 (FSI pads) | Only without screw inserts (alignment holes 57.24 mm apart) | Fits | Samtec FSI-150 | Contiguous |
| 4 × 25 | 2.00 | 48.0 × 6.0 | Tight (about 3–4 mm per end) | Fits | Mill-Max 838 (max 10 columns per strip) | Contiguous |
| 5 × 20 | 2.54 | 48.26 × 10.16 | Tight | Fits | P70 / 0965 | Needs a new zone map |
| 4 × 25 | 2.54 | 60.96 × 7.62 | No (as the ADR 0002 amendment notes) | Fits | P70 / 0965 | Contiguous |
| 2 × 50 | 2.54 | 124.46 × 2.54 | No | No | — | — |

## 4. Recommended field

### 4.1 Strip layout

Rows A and B form the high-speed band (pairs stand in one column, positive in A, negative in B, interface v1 §7.1). Rows C and D carry the slow and analog lines. The field is split between the L1 core zone and the L2/L3 zone:

| Strip | Rows | Columns | Contacts | Part (catalog code) | Populated on |
|---|---|---|---|---|---|
| S1 | A–B | L2/L3 side | 2 × 17 = 34 | 855-22-034-30-001101 | L3 pieces (and L2, §4.3) |
| S2 | C–D | L2/L3 side | 2 × 17 = 34 | 855-22-034-30-001101 | L2 and L3 pieces |
| S3 | A–B | core | 2 × 8 = 16 | 855-22-016-30-001101 | All pieces |
| S4 | C–D | core | 2 × 8 = 16 | 855-22-016-30-001101 | All pieces |

This assumes the 25th column joins the L2/L3 side, as the ADR 0002 amendment reserves the four extra positions in L2. If the spec places it in the core instead, the split becomes 16 + 9 columns (855-22-032 and 855-22-018); the dimensions below change by one column on each side of the gap. The 34-contact size is inside the catalog range (04–40, even counts) but its DigiKey listing was not checked.

### 4.2 Lattice and dimensions

All contacts sit on one 1.27 mm lattice (derived):

- **Rows** (y): A 0, B 1.27, C 3.81, D 5.08 mm. The empty row at 2.54 mm leaves room for two strip housings (0.89 mm overhang each, 0.76 mm clearance).
- **Columns** (x): L2/L3 side at 0 to 20.32 mm (17 columns); one empty column at 21.59 mm; core at 22.86 to 31.75 mm (8 columns). The empty column leaves room for two strip ends (0.825 mm overhang each, 0.89 mm clearance).
- **Contact span:** 31.75 × 5.08 mm. **Strip envelope:** about 33.4 × 6.9 mm. With an alignment pin and a retention feature at each end, the device-side field is about 44 × 7 mm (estimate), leaving roughly 5–6 mm per side inside a 55.5 mm inner width.
- **Second-source rule:** any double-row 1.27 mm strip whose housing extends at most 1.27 mm beyond its outer contact centres, sideways and lengthwise, fits the same pads.

### 4.3 Zones

- **L1 piece:** S3 + S4 only, 32 contacts. The core zone keeps the same coordinates on every piece (ADR 0003).
- **L2 piece:** S2 + S3 + S4. S2 also carries the L3 positions of row C (even columns: PCIe sidebands and `RSVD_S0`–`RSVD_S4`). On an L2 piece those contacts are fitted but left unconnected; on an L3 HAT they touch pads the device does not drive. Interface v1 §2 says unpopulated positions keep the grid; it does not yet say whether a fitted but unconnected contact is acceptable. This is for the spec owner.
- **L3 piece:** all four strips, 100 contacts.

### 4.4 Inputs for the spec (not edits)

- The gap between S1 and S3 falls between the last L3 column and the first L1 column. In the v1 table, column 16 is the `PCIE_TX±` pair and column 17 is `GND`. With the gap there, `PCIE_TX±` would have its nearer fence ground 2.54 mm away on one side instead of 1.27 mm. The strips should end and start on ground columns, so every pair keeps 1.27 mm ground neighbours on both sides. Where the 25th column goes decides this.
- Row C under the pairs is 2.54 mm from row B instead of 1.27 mm. That lowers the coupling between the pairs and the slow lines under them (§5).

## 5. Signal integrity

**Needs:** USB 3 Gen 1 at 5 Gb/s and PCIe 2.1 at 5 GT/s (Nyquist 2.5 GHz), CSI-2 D-PHY at 2.5 Gb/s per lane (Nyquist 1.25 GHz) (board high-speed I/O topology §1, §2.3). HCS-004 sets no cap, so a later device with faster lanes must also be considered; nothing here supports more than 5 Gb/s.

**What is published:**

- Mill-Max 855: 1 pF maximum capacitance, no S-parameters or bandwidth.
- Harwin P70, Mill-Max 0965: nothing at high frequency.
- Samtec FSI 3 mm: −3 dB at 12 GHz differential, rated 24 Gb/s ("FSI 3mm Connector Speed Rating", report revision 2011-05-10). That figure is for the connector on Samtec's test boards, not for a channel with a flat cable.
- Precedents (benchmark §2, secondary sources): Motorola Moto Mods ran USB 3.1 at 5 Gb/s over 16 pogo pins; Yokowo reports 10 Gb/s with four ground pins as the shield; a PIERS 2016 paper reports USB 3.1 rates on a 1.8 mm-pitch pogo design.

**Assessment of the fence pattern by geometry:**

| Geometry | Pair layout | Fence | Comment |
|---|---|---|---|
| 4 × 25 in 855 strips (recommended) | Vertical pair in one column of S1/S3, rows 1.27 mm apart | Ground columns 1.27 mm away on both sides, rows A and B (G-S-G per row, G-S-S-G per pair across the column) | Interface v1 §7.1 pattern intact inside each strip. Pair length about 4.3 mm at working height, about 0.04 λ at 2.5 GHz in air (derived): electrically short at 5 Gb/s, marginal beyond. Impedance unknown; small plungers at 1.27 mm are likely above 90–100 Ω (**UNVERIFIED**, TDR needed) |
| 4 × 25 uniform 1.27 | As above | As above | No stocked part |
| 2 × 50 (1.27 or 1.00) | Pair across the two rows | Ground columns on both sides | Works, but row 2 has no slow-line buffer; length is the problem, not SI |
| 2.00 or 2.54 mm grids | As the recommended layout | Ground columns 2.0–2.54 mm away | Looser coupling to the fence, larger pads (Ø 1.68–2.20 mm) with more pad capacitance on every high-speed line; workable at 5 Gb/s is plausible but less likely than at 1.27 mm (engineering judgement) |
| Samtec FSI 2 × 50 | Pair across rows or along a row | Ground positions in the row | The only option with published SI |

**Confidence:** moderate for 5 Gb/s on the recommended field, low for anything faster. Nothing has been measured, and the channel also includes the DF40 flat cable (board ADR 0002), the HD3SS3212 mux on USB 3, ESD arrays at the field (BOM line 68) and the HAT's pads.

**Test coupon (required before freeze, interface v1 §10 item 5):**

1. Device-side coupon with S1 + S3 on the real stack-up and the DF40 flat cable; HAT-side coupon with the pad map of §6; 2×-thru de-embedding structures (benchmark §6, steps 1–4).
2. TDR per pair, mated at nominal, minimum and maximum compression.
3. 4-port S-parameters to at least 10 GHz: SDD21, SDD11, NEXT and FEXT between neighbouring pairs and from the pairs to row C.
4. End-to-end: USB 3 Gen 1 and PCIe Gen 2 link training and error counters under load; CSI-2 test pattern at 2.5 Gb/s per lane with zero CRC/ECC errors.
5. The coupon also decides the open fence questions of interface v1 §10 item 5, and whether `PCIE_TX±` next to the strip gap is acceptable (§4.4).

A second coupon with Samtec FSI-150 on the same flat cable gives a documented reference if the pogo field fails.

## 6. Pads and force

### 6.1 Device side (spring part)

- Solder pads Ø 1.02 mm on the 1.27 mm lattice (Mill-Max catalog p. 19), 0.25 mm between pads.
- Four strips must be coplanar within the stroke budget. The catalog does not give a coplanarity figure for series 855 (it gives 0.13 mm for 838 and 856, for up to 20 positions; **UNVERIFIED** for 855). A rigid device-side piece and a reflow fixture are assumed.

### 6.2 HAT side (HCS-014)

- **Pad:** round, **Ø 0.90 mm**, non-solder-mask-defined, on the mirrored lattice of §4.2. With the 0.48 mm plunger tip this allows ±0.21 mm of radial misalignment with the whole tip on the pad (derived). The case must hold the HAT to about ±0.1 mm with alignment pins (benchmark §5, proposal 1), which leaves margin for plunger wobble.
- **Finish:** electrolytic hard gold recommended for HATs that are swapped often; ENIG acceptable for low-cycle and DIY HATs, with its wear life **UNVERIFIED** (benchmark §6 step 5 measures it). Samtec specifies 30 µin hard gold for its compression contacts on mating pads; no spring-pin maker states a pad finish requirement.
- **High-speed pads:** no via in the pad, no stub; ground pads of the fence connected with their own vias to the reference plane.
- **Zones above the HAT's level:** no exposed copper (interface v1 §2).

### 6.3 Force budget

Per contact (Mill-Max catalog): 25 gf at initial height, 60 gf at 0.7 mm. Between those points the spring is taken as linear (about 50 gf/mm, derived; **UNVERIFIED** beyond mid stroke).

| Field | Contacts | At 0.7 mm (nominal) | At 0.45 mm | At 0.95 mm |
|---|---|---|---|---|
| L3 (all strips) | 100 | 58.8 N | 46.6 N | 71.1 N |
| L2 (S2, S3, S4) | 66 | 38.8 N | 30.8 N | 46.9 N |
| L1 (S3, S4) | 32 | 18.8 N | 14.9 N | 22.8 N |
| L3 with Harwin P70 (2.54 mm, for comparison) | 100 | 88 N at 2.90 mm (59–117 N with the ±0.29 N spread) | — | — |

- **Working point:** 0.7 mm compression (working height about 4.28 mm), stack tolerance ±0.25 mm, which stays inside the 1.4 mm maximum stroke and above the 25 gf preload.
- **Retention:** whatever holds the HAT (HCS-018 is withdrawn, so no slider lock is required) should react at least 1.5 × the maximum, about **107 N** for an L3 field, with no loss of contact under the shock and vibration tests of benchmark §6 step 7 (engineering margin, not a sourced figure).
- **HAT PCB deflection:** about 71 N spread over a 33 × 7 mm area pushes the HAT board away at the field. The HAT case should support the board next to the pad field so it deflects less than about 0.1 mm (engineering estimate; Samtec recommends a stiffener behind FSI mating pads for the same reason).

## 7. Sourcing and second source

- **Mill-Max 855** is a catalog part sold through DigiKey, an authorized distributor. Stock on 2026-10-22 is thin (33 of the 40-position size, 78 of the 20-position, 0 of the 16-, 18- and 32-position), with a 4-week factory lead time. That is a catalog product with a short lead time, not a custom part (T1-038), but prototype quantities should be ordered early. T1-029 asks for availability to be validated before adoption; it is met on lead time, not on shelf stock.
- **No drop-in second source verified.** The pad map rule of §4.2 is written so a second double-row 1.27 mm strip can be qualified without changing a single HAT. Candidates to check: CnC Tech, Würth Elektronik and Preci-Dip multi-way spring strips, and LCSC-stocked 1.27 mm double-row strips (all **UNVERIFIED**). Until one passes the coupon and durability tests, the pogo field is single-source on the device side only; HATs carry no part from it.
- **Fallbacks, in order:**
  1. **Samtec FSI-150-03-G-D** (1.00 mm, 2 × 50, published SI) if the 1.27 mm pogo field fails the coupon. It changes the HAT pad map, requires hard gold on every HAT and needs ADR 0001 to cover a compression contact.
  2. **4 × 25 at 2.54 mm with Harwin P70-1000045R or Mill-Max 0965** (two vendors, best stock) if the device-side piece can lie along the 94.3 mm length. The field is 60.96 × 7.62 mm and the force about 88–113 N (derived from the figures in §2.1).

## 8. Power check (BRD-049)

The Mill-Max 855 is rated 2 A continuous and 3 A peak per contact, the same 2.0 A that interface v1 §5 used for the P70. The pogo counts (`HAT_5V` 3, `HAT_3V3` 2, `GND` 4) and their margins are unchanged: at most 38 % continuous, 50 % with one contact open, and 90 % at the fault ceiling with one contact open. Contact resistance falls from 30 mΩ to 20 mΩ maximum. Temperature rise at 2 A per contact in a 1.27 mm field is not published (**UNVERIFIED**); benchmark §6 step 6 measures it.

## 9. Open points

1. Qualify a second 1.27 mm double-row spring strip against §4.2 (§7).
2. Test coupon and durability tests (§5, benchmark §6), on ENIG and hard-gold pads.
3. Where the 25th column goes, and whether the strip gap may sit next to a pair (§4.4).
4. Whether fitted but unconnected contacts are allowed on L2 pieces (§4.3).
5. Coplanarity of four 855 strips after reflow (§6.1).
6. Case wall thickness and the retention design, now that HCS-018 is withdrawn (§1, §6.3).
7. Spring force beyond mid stroke (§6.3).

## 10. Sources

All accessed 2026-10-22.

- Mill-Max Mfg. Corp., *Metric Catalog* (PDF modified 2023-08-15), copy from the DigiKey mirror <https://mm.digikey.com/Volume0/opasdata/d220001/medias/docus/7161/Mill%20Max%20catalog.pdf>, sha256 `2a6aeebf356b2da46e2018bf84b292f3271c9774232d81a2da9ade6e57649f33`: series 854/855 (pp. 18, 19, 19.1), 836/838 (p. 19.2), 856 (p. 21.3). mill-max.com returned HTTP 503 on this date.
- Harwin, *Pogo Pins and Contact Pads*, product datasheet PD012 <https://content.harwin.com/m/8d9ee00ee190470b/original/PD012-Product-Datasheet-POGOPINS-Pogo-Pins-and-Contact-Pads-EZi.pdf>, sha256 `e15e0a6fa22aba172cfca65a8d52dfa1eaac85eee982063ef8acbe9739787e7b`; parts library sheet `Harwin/P70-1000045R.md` (drawing P70-10X0045R, issue 4).
- Samtec, FSI product page <https://www.samtec.com/products/fsi>; product specification *FSI 1,00 mm One Piece Interface*, rev. D, 2015-10-30 <https://suddendocs.samtec.com/productspecs/fsi.pdf>; catalog page <https://suddendocs.samtec.com/catalog_english/fsi06_10.pdf>; *FSI 3mm Connector Speed Rating* <https://suddendocs.samtec.com/testreports/connector-speed_fsi-03mm_web.pdf>; print FSI-1XX-03-X-D-X-XX-XX-X-XX rev. AG <https://suddendocs.samtec.com/prints/fsi-1xx-03-x-d-x-xx-xx-x-xx-mkt.pdf>; mating-card layout rev. F <https://suddendocs.samtec.com/prints/fsi-mating%20board.pdf>.
- DigiKey product and search pages (stock, lead time, attributes): P70-1000045R; 0965-0-15-20-80-14-11-0; 855-22-040-30-001101; 855-22-020-30-001101; 855-22-016-30-001101; search `855-22-032-30` and `855-22-018-30`; 838-22-020-30-001101; FSI-150-03-G-D; FSI-150-03-G-D-AD <https://www.digikey.com/en/products/detail/samtec-inc/FSI-150-03-G-D-AD/7259308>; search "spring loaded connector 1.27mm surface mount 2 row" (Mill-Max only); search "preci-dip spring loaded 1.27mm".
- Rockchip RK3568 datasheet V1.0 and the RK3566 brief, as cited in the board [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md) (lane rates).
- Secondary, through the [interconnect benchmark](2026-10-hat-interconnect-benchmark.md): Moto Mods, Yokowo 16-pin, PIERS 2016.
