# ADR 0004: Pogo part and field geometry

- **Status:** Accepted
- **Date:** 2026-10-22
- **Amended by:** [ADR 0005](0005-interface-v1-1-shared-extension-and-contact-reduction.md) (2026-10-24): the field is 4 × 17 = 68 with pairs along the rows; part, second source and pitch unchanged.
- **Decided by:** Claude, under the maintainers' delegation (availability first with a second source, small-cost rule, safe power)

## Context

Interface v1 needs 100 positions in nested L1/L2/L3 zones (ADR 0002 amendments, ADR 0003). The [pogo part and geometry research](../research/2026-10-pogo-part-and-geometry.md) found:

- **Mill-Max 855 at 1.27 mm:** the only stocked spring part that reaches 1.27 mm in both directions (field about 32 × 5 mm). Thin stock (0 of the sizes needed, 4-week lead time), no verified second source, and several times the cost of the alternatives.
- **Harwin P70 or Mill-Max 0965 at 2.54 mm:** two makers, stocked, 2 A per contact (the same as the 855, so the BRD-049 power counts do not change). A 4 × 25 field is about 61 × 7.6 mm, which fits only along the body's length.

## Decision

- **Primary:** a 4 × 25 field at **2.54 mm**, oriented along the body's length, with **Harwin P70** springs and **Mill-Max 0965** as the second source on the same pads.
- **Compact option, not adopted now:** the Mill-Max 855 at 1.27 mm, kept documented; it becomes the primary only if it reaches real stock, a second source and a comparable cost.
- **Zones:** the L1 core zone is one contiguous end of the field, so an L1 piece populates only that end (ADR 0003).
- **HAT pads:** flat, hard gold recommended (ENIG acceptable for HATs swapped rarely), with a stiffener under the pad area (HCS-014).
- **Retention:** the HAT retention is sized for the full L3 spring force with margin.
- **Signal integrity:** a test coupon (pogo field, DF40 flat cable and HAT pads: TDR, S-parameters to 10 GHz, crosstalk, real USB 3, PCIe Gen 2 and CSI-2 links) and mating-cycle and contact-resistance tests are required before the pad map freezes. Strips and fences are laid out so every high-speed pair has a ground neighbour on both sides.

## Consequences

- The device-side piece and the HAT case lay out the field lengthwise; the case and the HAT mount are updated.
- Interface v1 keeps its pinout; only the physical grid pitch is fixed here.
