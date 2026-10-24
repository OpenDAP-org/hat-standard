# ADR 0003: Compatibility levels L1–L3

- **Status:** Accepted
- **Date:** 2026-10-22
- **Amended by:** [ADR 0005](0005-interface-v1-1-shared-extension-and-contact-reduction.md) (2026-10-24): L2 without line out, SWD as a GPIO alternate; L3 is one shared extension, PCIe or CSI-2 by declared function.
- **Decided by:** Claude, following the maintainers' direction ("several compatibility layers, L1–L3, like popular standards; USB 3 as the base, CSI and PCIe at other levels")

## Context

Interface v1 ([ADR 0002](0002-interface-v1-native-high-speed-management-over-usb.md)) keeps native only what needs to be native. Not every device or HAT needs all of it. Popular standards scale the same way (Bluetooth profiles, PCIe link widths), so a simple device or HAT implements a small, well-defined core and a capable one adds levels. The standard must stay hardware-agnostic on both sides (HCS-021) and adoptable by other makers.

## Decision

| Level | Adds | Typical HATs |
|---|---|---|
| **L1 Core** | Power by class (HCS-015) and ground, presence detect, interrupt/wake, ID EEPROM (HCS-017), general I2C, the **USB 3 port** with its USB 2 half (low-speed I/O through the management bridge, ADR 0002) | Sensors, GNSS, LoRa, NFC, e-ink, SDR, oscilloscope, VNA, keyboards, powerbank |
| **L2 Control and audio** | L1 + I2S/TDM on the DAC's clock domain, line out, four fast GPIO (PWM and interrupt capable), SWD and reset | Audio interfaces and amplifiers, recorders, instruments, debuggable MCU boards |
| **L3 High speed** | L2 + PCIe + MIPI CSI | NAS (NVMe + Ethernet), cameras, AI accelerators, M.2 radios |

- **Compatibility rule:** a HAT works on any device of its level or higher. A device that receives a HAT needing a higher level says so clearly through Blackbird OS (T1-045: never silent), and powers nothing it cannot support safely.
- **Declaration:** a device declares the level it implements; a HAT declares the level it needs in its identity record (HCS-017).
- **One shared pinout:** all levels use one pogo field and one pinout; each level is a nested subset of the same fixed positions, never a separate connector. The device-side piece keeps one housing and one grid; a lower-level version only populates fewer pins.
- **Alternate functions (should):** low-speed L2 contacts (the fast GPIO) may declare alternate functions, such as native UART or SPI, in the HAT identity record, set by the SoC pinmux without extra hardware. High-speed, power, ground, detect and ID contacts always keep a fixed function. A device without alternate-function support stays compatible and treats them as GPIO.
- **Physical zones:** the pogo field is laid out in zones: L1 contacts form a core zone; L2 and L3 extend it. A device-side piece (or another maker's device) implementing L1 populates only the core zone. Reserved contacts (HCS-022) are distributed so every level keeps room to grow.
- **Conformance:** each level has its own conformance test suite (toolkit-verify), the base for later certification.
- **The T1 implements L3** (T1-045).

## Consequences

- Interface v1 is specified by level and zone.
- Other makers can adopt L1 with a small pogo field and grow later.
- A lower-cost device-side piece implementing only L1 or L2 becomes possible without a separate standard.
