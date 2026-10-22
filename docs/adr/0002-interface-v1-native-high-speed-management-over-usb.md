# ADR 0002: Interface v1: native high-speed and safety lines, low-speed I/O over USB

- **Status:** Accepted
- **Date:** 2026-10-22
- **Decided by:** Claude, under the maintainers' delegation (HCS-021 hardware-agnostic, HCS-022 reserved contacts, build on standards, BRD-049 safe power)

## Context

Interface v0 carries about 122 pogo contacts, 44 of them native low-speed signals. The [interface reduction research](../research/2026-10-interface-reduction.md) checked every reference HAT and a wide set of plausible future HATs against what each truly needs, and which classes can move behind USB with mainline Linux drivers. The pogo field is also the largest cost line of the device-side piece.

## Decision

1. **Interface v1 is the research's recommended level (v1-R), without the second USB 2 port:** about 93 pogo contacts (about −24 %), 24 native low-speed signals.
2. **Native, unchanged:** all high-speed pairs of the USB 3 port (with its USB 2 half), PCIe and MIPI CSI (HCS-004), power and ground with their BRD-049 margins, `HAT_DET#`, the ID EEPROM bus (read before the HAT is powered, so the power class is known before power is applied; passive HATs need nothing else; the OpenDAP identity format, HCS-017, with HAT+ compatibility as a should, HCS-024; U-Boot reads it too), `HAT_INT#` (wake source and the interrupt for anything behind a bridge), one general I2C (kernel devices described in the device tree, camera control), the PCIe sidebands, SWD and reset (direct flashing and recovery of any HAT MCU, including a bricked bridge, HCS-013), four fast GPIO lines (every one PWM- and interrupt-capable), one I2S/TDM port of 8 lines on the DAC's clock domain, and line out with its analog ground.
3. **Over USB:** UART, SPI, slow GPIO and JTAG move behind USB. There is **one USB port to the HAT**: the USB 3 port with its own USB 2 half (both buses belong to one USB 3 port and stay). The separate second USB 2 port is removed. A HAT that needs a USB 3 device and a management bridge at the same time carries a standard USB 3 hub (mainline driver); simple HATs connect their bridge directly. The reference bridge protocol is **DLN2**, which has mainline Linux drivers for GPIO with interrupts, I2C, SPI and ADC; OpenDAP publishes open DLN2-compatible firmware for common MCUs, and sends the missing USB ID upstream without waiting for it (udev rule meanwhile). Standard USB classes (CDC-ACM, HID, USB Audio, CMSIS-DAP) are equally valid.
4. **PCIe stays, next to USB 3.** On the RK3566 they come from separate PHYs (Multi-PHY2 and Multi-PHY1), so a HAT gets two independent 5 Gb/s channels at once: the NAS HAT (NAS-005, NAS-006) runs NVMe on PCIe and 2.5G Ethernet on USB 3 without sharing a lane, and BRD-045 rules out an on-board PCIe switch. PCIe-only devices (M.2 AI accelerators such as Hailo-8/8L, M.2 Wi-Fi 6E/7 cards) and FPGA endpoints need it, and dropping it would cap HATs to one 5 Gb/s link (T1-045, HCS-004).
5. **Removed:** the camera reference clock (camera HATs carry their own oscillator), `CAM_RST#` (becomes a fast GPIO), and line in (the T1 has no input to receive it; recording HATs carry their own ADC).
6. **One audio clock domain per HAT,** conditional on the RK3566 running the HAT I2S transmit and the DAC path from one shared clock (test on the first prototype). If the test fails, the second domain returns on reserved contacts.
7. **Scope of BRD-004 and BRD-005:** they govern the T1 main board. HAT hardware follows HCS-013: every programmable HAT chip is flashable and debuggable from OpenDAP OS, natively over SWD or through the management port.
8. **Board-to-piece connector:** the DF40 signal connector stays at 100 positions, with the freed positions as spares; the cost difference is small (maintainers' rule on small-cost hardware) and spares keep the interface extensible.
9. **Reserved pogo contacts:** at least 8 % of the field stays reserved (HCS-022).

## Consequences

- No reference HAT breaks; the camera HAT adds an oscillator.
- Some HAT types get harder (SPI TFT displays, Raspberry-Pi-style prototyping HATs, kernel serial devices needing a native UART, FPGA-only HATs without a probe chip): they use a USB bridge, a standard USB class, or the fast GPIO and I2C lines.
- The interface spec moves from v0 to v1 (Draft) with the contact list by function; the board pin plan maps the RK3566 onto it.
