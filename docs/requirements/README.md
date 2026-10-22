# HAT Case Standard requirements

Requirements for the HAT Case Standard.

| ID | Requirement | Priority | Status |
|---|---|---|---|
| HCS-001 | The standard shall cover both sides of the HAT Case: the device (T1) side and the accessory side. | must | Confirmed |
| HCS-002 | The standard shall be usable by third-party accessory developers. | must | Confirmed |
| HCS-003 | A HAT shall replace the basic back cover of the core with a HAT case. | must | Confirmed |
| HCS-004 | Each exposed high-speed interface (PCIe, USB, camera, …) shall target the theoretical maximum the hardware and software can deliver, with no cap set up front. Actual speeds are measured at the end. | must | Confirmed |
| HCS-005 | Any compromise on the exposed ports shall be fully justified. | must | Confirmed |
| HCS-006 | The HAT interface shall use a single interconnect type for every signal: either all pogo pins or all connectors, ideally of the same part. | must | Confirmed |
| HCS-007 | A HAT shall identify itself through an on-board EEPROM. | must | Confirmed |
| HCS-008 | There shall be no central registry of manufacturer or vendor IDs. | must | Confirmed |
| HCS-009 | The EEPROM shall carry a capability manifest in a standard schema. | must | Confirmed |
| HCS-010 | The capability manifest shall be signed, and OpenDAP OS shall verify it. | must | Confirmed |
| HCS-011 | The capability manifest shall work with the OpenDAP OS package repository mechanism (reverse-domain, well-known HTTP, through the package endpoint). | must | Confirmed |
| HCS-012 | The device-side piece shall be fixed to the T1; only the HAT is swapped. | must | Confirmed |
| HCS-013 | The standard shall support HATs carrying microcontrollers, FPGAs or another SoC, which OpenDAP OS can flash, operate and debug. | must | Confirmed |
| HCS-014 | HATs shall be passive at the interface: flat contact pads on the HAT, springs only on the device side. | must | Draft |
| HCS-015 | Power classes: A and B (the HAT draws power) and F (the HAT is a power source). | must | Draft |
| HCS-016 | Power-up sequence: presence detect → 3.3 V → read EEPROM → apply overlay → 5 V → switches and high-speed links. | must | Withdrawn |
| HCS-017 | The EEPROM shall use the OpenDAP HAT identity format: reverse-domain ID, power class, Ed25519 signature and the hash of the capability manifest. | must | Draft |
| HCS-018 | The device-side piece shall be fixed with four M2 screws, and a spring-loaded steel slider lock shall hold the HAT, released by a button on the right edge. | should | Withdrawn |
| HCS-019 | The lock may use a metal part, as the exception to polymer-first cases. | should | Confirmed |
| HCS-020 | The device-side piece of the HAT case shall be screwed to the T1. | must | Confirmed |
| HCS-021 | The standard shall be hardware-agnostic on both sides: it shall not depend on the board or SoC behind the device side, nor on the HAT's hardware, and shall stay valid if the device's board changes completely. Signals are defined by function and interface class, never by a specific SoC pin. | must | Confirmed |
| HCS-022 | The interface shall keep a few reserved contacts for future revisions, without displacing essential signals to get them. | must | Confirmed |
| HCS-023 | The device-side piece (pogo field and its flat cable) shall be an independent accessory of the HAT standard, fitted to the T1 by the user or at assembly and then fixed to it (HCS-012, HCS-020). A T1 without it works fully, and the back cover closes it. | must | Confirmed |
| HCS-024 | The HAT standard is OpenDAP's own. It takes inspiration from established standards (Raspberry Pi HAT+ and others) and should support them where that costs little, for example by keeping the identity layout compatible with HAT+ so existing tools can read it. | should | Confirmed |

## Open questions

- Exposed port set and pinout: a functional allocation on the flat-cable connectors is in the [HAT interface v0](../specs/interface-v0.md) (Draft, HCS-021, HCS-022); the pogo-field layout is still open.
- Mechanical details of the device-side piece and the HAT case.

## Context

- HCS-017 and HCS-024 (maintainers, 2026-10-22): the standard is OpenDAP's own; compatibility with Raspberry Pi HAT+ and other standards is a *should*, not a constraint.
- HCS-022 is must since 2026-10-22 ([priority audit](https://github.com/opendap-project/t1/blob/main/docs/research/2026-10-requirement-priorities.md)): contacts cannot be added once third parties build HATs.
- The connection from the board to a HAT has two stages: a flat cable from the board to the device-side piece of the HAT case, then from that piece to the HAT hardware.
- The second stage is all pogo pins ([ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md)). The benchmark estimates about 120–128 contacts; two rows would be about 81 mm long, more than the device width, so its proposal is a 4 × 32 grid at 1.27 mm (about 41 × 4 mm). Grid, pitch and pinout are not defined yet.
- The flat cable from the board to the device-side piece is specified by the `board` repository.
- HCS-014 to HCS-018: Adopted from the earlier experiments on 2026-10-21 as **Draft**: consistent with the decisions already taken and with the product philosophy, pending the maintainers' confirmation. Part availability still has to be checked against the sourcing rule (T1-010).
- Not adopted from the experiments: their 42-contact connector (Raspberry Pi 40 pins + GND + PRESENT#) carried no PCIe or camera lanes, which conflicts with HCS-004 and HCS-006.
- Withdrawn on 2026-10-21 after the [experiment review](https://github.com/opendap-project/t1/blob/main/docs/research/2026-10-experiment-review.md): HCS-016, HCS-018.
- The device-side piece goes wherever it fits best inside the T1. The HAT Case Standard is defined once the rest of the device is settled.
