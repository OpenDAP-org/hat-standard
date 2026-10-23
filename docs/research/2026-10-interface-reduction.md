# HAT interface reduction: what can move behind USB

- **Date:** 2026-10-22
- **Status:** Desk research. **Nothing here is a decision.** It proposes a reduced interface (v1) for the maintainers to accept, change or reject. The [HAT interface v0](../specs/interface-v0.md) (Draft) stays in force until they decide.
- **Scope:** how far the HAT contact set can shrink (fewer contacts, less redundancy) by carrying low-speed I/O over USB, without losing what HATs need. It covers both stages: the DF40 flat cable to the device-side piece and the pogo field.
- **Inputs:** HAT Case Standard requirements HCS-001 to HCS-023 ([`../requirements/README.md`](../requirements/README.md)); [HAT interface v0](../specs/interface-v0.md); [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md); [HAT interconnect benchmark](2026-10-hat-interconnect-benchmark.md); board research [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md) and [pin multiplexing v0](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md) (§3.4–3.11, §4.2, §4.3); board [BOM v1](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md) lines 44, 65–70, 93–95, 102 and the power tree §4.6; board requirements BRD-004, BRD-005, BRD-006, BRD-044, BRD-045, BRD-049; the requirements of the seven reference HATs.
- **Method:** mainline Linux read on 2026-10-22 from `torvalds/linux` `master` (top-level `Makefile`: 7.3): the driver sources and their `Kconfig` entries were downloaded and grepped for USB IDs, dependencies and functions. Firmware projects were read from their repositories. No hardware was built or measured. **UNVERIFIED** marks anything not checked against a primary source. HAT needs for future HATs are engineering judgement, not maintainer statements.

---

## 0. Summary

1. **High-speed stays as it is.** USB 3 (with its USB 2 pair), the second USB 2 port, PCIe x1 with REFCLK, and 4-lane CSI-2 cannot move anywhere without breaking HCS-004, BRD-006, BRD-044 or BRD-045. They are 12 pairs and about 48 pogo contacts with their ground fence, half of the field. Merging the second USB 2 port into a hub on the HAT would save one pair, but the reduction makes that port the natural management port for everything that moves to USB, so the report keeps it.
2. **What can move to USB without loss:** UART (CDC-ACM or a USB-UART bridge), SPI, the slow GPIO lines, and JTAG for FPGAs (a USB probe on the HAT). Mainline Linux has class drivers or bridge drivers for all of them. The camera clock and the line-level input can go without USB: a HAT oscillator replaces the first, and a HAT ADC digitises the second.
3. **What must stay native:** power, ground and presence (safety, sequencing); the ID EEPROM bus (read before the HAT has power, and by U-Boot); one interrupt and wake line; the general I2C bus (camera CCI and every DT-described kernel device); the PCIe sidebands; a minimal SWD + reset for direct flashing and recovery (BRD-004, HCS-013); four fast GPIO lines for PWM, timestamped interrupts and resets; one I2S/TDM port; and the analog line output.
4. **Recommended v1 (v1-R):** **97 pogo contacts instead of about 122** (−25, −20 %), with **24 native low-speed signals instead of 44**. The DF40 signal connector needs 71 contacts instead of 96, so it fits the 80-position size with spares. The power connector does not change: its 40 power and ground contacts follow from the DF40 0.3 A rating (BRD-049).
5. **Maximum reduction (v1-M):** **81 pogo contacts**, 12 native low-speed signals. It also moves digital audio to USB Audio Class 2 and flashing to ROM USB bootloaders. It works, but it makes a codec-only audio HAT and a bare-MCU HAT need a USB bridge, and it puts a HAT-side bridge between the SoC and every HAT chip. That conflicts with the wording of BRD-004 and BRD-005 (both Confirmed). Not recommended without a maintainer decision.
6. **Cost moves to the HAT:** a HAT that used UART, SPI, JTAG or the slow GPIO lines adds one bridge part (a USB-UART chip, an MCP2221A-class HID bridge) or an MCU with open firmware: DLN2 protocol for GPIO, I2C, SPI and ADC; CMSIS-DAP for SWD/JTAG; CDC-ACM for serial. Four of the seven reference HATs (NAS, VNA, radio, powerbank) need nothing new. The camera HAT adds one oscillator. The recorder and oscilloscope HATs keep their native paths in v1-R.
7. **Decisions needed:** listed in §9. The main ones: whether HAT-side bridges are allowed under BRD-004/BRD-005, whether the audio port may be one clock domain, and whether the DF40 signal connector shrinks or keeps 100 positions with spares.

---

## 1. Baseline: interface v0

| Group (interface v0) | DF40 signal (96 usable) | DF40 power (46 usable) | Pogo (estimate, §5.4 method) |
|---|---|---|---|
| High-speed pairs (12) | 24 | — | 24 |
| High-speed ground fence | 13 (GSSG row) | — | 24 (about 4 contacts per pair in a grid, benchmark §3.2) |
| Low-speed signals | 43 | 1 (`HAT_DET#`) | 44 |
| Analog (line out L/R, line in L/R, 2 × `AGND`) | 6 | — | 6 |
| Low-speed grounds | 8 | — | 8 |
| Reserved (HCS-022) | 2 | 5 | 7 |
| `HAT_5V` / `HAT_3V3` / `GND` | — | 12 / 6 / 22 | 3 / 2 / 4 |
| **Total** | **96** | **46** | **≈ 122** |

The 43 low-speed signals on the signal connector: ID I2C 2, `HAT_INT#` 1, I2C 2, SPI 5, UART 4, debug 5, PCIe sidebands 3, `CAM_RST#` 1, `CAM_MCLK` 1, GPIO 8, I2S 5, TDM in 6.

The pogo power counts use the board power tree's pogo rating (Harwin P70, 2.0 A per contact, power tree §4.10) with interface v0 §3 rules: at most 70 % of the rating in continuous use, at most 100 % at the fault ceiling, plus one contact for redundancy. `HAT_5V` carries 2.0 A continuous and 3.0 A at the class F fault ceiling (TPS2121 ILM): 2 + 1 = 3. `HAT_3V3` carries 1.0 A (1.15 A ceiling): 1 + 1 = 2. `GND` returns 3.0 A: 3 + 1 = 4. The benchmark and the pin plan estimated 118–128 pogo contacts. 122 sits inside that range.

---

## 2. Needs matrix

### 2.1 Legend

- **N**: needs it natively at the interface.
- **U**: needs the function, but it works as well or better over USB (a USB-class device or a bridge on the HAT).
- **o**: optional, or only some designs need it.
- **—**: not needed.
- **Power class:** HCS-015 (Draft) defines A and B (the HAT draws power) and F (the HAT is a source). A and B have no current figures yet, so "A" means a low draw (≲ 0.5 A at 5 V) and "B" a high draw. The loads are estimates (benchmark §1.4), **UNVERIFIED**.
- **I2S/TDM clock:** "own" means the HAT's own clock (HAT is provider, or the device provides an independent clock). "DAC" means the HAT must run on the T1 DAC's clock domain. Interface v0 already dropped the DAC-clock output (pin plan §3.4), and no HAT below needs it.

### 2.2 Reference HATs

| HAT | Power | ID | USB 2 | USB 3 | PCIe | CSI | I2S/TDM | I2C | SPI | UART | GPIO/IRQ | PWM | ADC | SWD/JTAG | Analog | Ref clock |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Powerbank (PWR-005) | F | N | — | — | — | — | — | N (gauge, charger telemetry) | — | — | N (alert → `HAT_INT#`) | — | — | o (if it has an MCU) | — | — |
| NAS (NAS-005/006) | B | N | o | o (USB NIC) | N (switch + NVMe + NIC) | — | — | o (sensors, fan) | — | — | o | o (fan) | — | — | — | — (REFCLK is part of PCIe) |
| Camera (CAM-005, action camera) | A–B | N | o (controls MCU) | — | — | N (4 lanes) | o (mic → I2S or USB) | N (CCI) | — | — | N (sensor reset) | — | — | — | — | o (sensor MCLK; an on-HAT oscillator works) |
| Recorder (REC-005, Tascam-inspired) | A–B | N | U (UAC2 is an alternative path) | — | — | — | N or U: capture up to 8 ch, own clock | N (codec control) | — | — | o | — | — | o | o (line out to monitor) | — |
| Oscilloscope (OSC-005, DS212-based) | A–B | N | U (sample stream) | o | — | — | — | o | U (FPGA/ADC) | U | o | — | U (on the HAT) | N (MCU + FPGA, HCS-013) | — | — |
| VNA (VNA-005, NanoVNA-based) | A | N | U (CDC-ACM, the NanoVNA protocol) | — | — | — | — | — | — | U | — | — | U (on the HAT) | N (MCU flashing) | — | — |
| Radio (RAD-005, HackRF Pro-based) | A–B | N | U (HackRF streams over USB; HackRF Pro USB generation **UNVERIFIED**) | o | — | — | — | — | — | — | — | — | — | o (CPLD/FPGA via the MCU) | — | o (10 MHz reference: an on-HAT input) |

### 2.3 Plausible future HATs

| HAT | Power | ID | USB 2 | USB 3 | PCIe | CSI | I2S/TDM | I2C | SPI | UART | GPIO/IRQ | PWM | ADC | SWD/JTAG | Analog | Ref clock |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| GNSS | A | N | U (CDC-ACM; mainline `gnss` USB driver for some IDs) | — | — | — | — | o | — | U | N (PPS, timestamped) | — | — | — | — | — |
| Cellular modem (M.2 / LGA) | B | N | U | o | o | — | o (PCM voice) | — | — | — | N (PWRKEY, RESET, W_DISABLE, RI) | — | — | — | — | — |
| LoRa / Meshtastic | A | N | U (MCU node over CDC-ACM, the usual Meshtastic link) | — | — | — | — | — | U (bare SX126x over SPI) | U | o (DIO1, BUSY if bare) | — | — | o | — | — |
| NFC / RFID | A | N | U (USB CCID reader) | — | — | — | — | o (PN7150 class) | U | U | o | — | — | — | — | — |
| Thermal camera | A | N | U (UVC from a bridge MCU) | — | — | o | — | o (MLX90640 class) | U (Lepton VoSPI) | — | o | — | — | — | — | — |
| E-ink display | A | N | U (GUD, R1 format) | — | — | — | — | — | U | — | o (BUSY) | — | — | — | — | — |
| Keyboard / game controller | A | N | U (HID) | — | — | — | — | o (I2C-HID) | — | — | o (wake) | — | — | — | — | — |
| Environmental sensors | A | N | o (HID sensor hub) | — | — | — | — | N (IIO drivers) | — | — | o | — | — | — | — | — |
| Extra battery / charger | F or B | N | — | — | — | — | — | N | — | — | N (alert) | — | — | — | — | — |
| 2.5G Ethernet | B | N | — | o (RTL8156 class) | o (RTL8125 class) | — | — | — | — | — | o | — | — | — | — | — |
| NVMe storage | B | N | — | — | N | — | — | — | — | — | — | — | — | — | — | — |
| USB hub | A–B | N | N | N | — | — | — | — | — | — | — | — | — | — | — | — |
| SDR variants | A–B | N | U | o | o (PCIe SDR) | — | — | — | — | — | — | — | — | o | — | o |
| Logic analyser | A | N | U (fx2lafw class) | o (high rates) | — | — | — | — | — | — | — | — | — | — | — | — |
| MIDI / audio interface | A–B | N | U (USB MIDI, UAC2) | — | — | — | N or U | o | — | U (DIN MIDI is a UART) | — | — | — | — | o | — |
| Haptics | A | N | — | — | — | — | — | N (DRV2605L class, mainline input driver) | — | — | o | o (direct LRA drive) | — | — | — | — |
| Dev / prototyping board | A–B | N | U | o | o | o | o | N | U | U | N (some) | N (some) | U | N | o | — |
| AI accelerator | B | N | o (USB TPU class) | o | N (M.2 accelerator class) | — | — | — | — | — | o | — | — | — | — | — |
| Amplifier (headphone / speaker amp fed by the T1 DAC) | B | N | — | — | — | — | — | o | — | — | o (mute) | — | — | — | N (line out) | — |

### 2.4 What the matrix shows

- **Every HAT needs:** power, ground, presence, the ID EEPROM. Most need an interrupt line.
- **Only native-bus kernel devices need I2C natively:** camera CCI, fuel gauges, sensors, codecs, haptic drivers. These drivers are described in device tree and expect a native adapter. I2C is cheap (2 contacts) and used widely. It stays.
- **No HAT in the list needs native UART or SPI.** Every case has a standard USB alternative that is as good or better: modems, GNSS, LoRa nodes and NFC readers are USB devices in their mainstream form. The exceptions are the dev board and a few bare-chip designs (§8).
- **Native GPIO matters for a few things only:** timestamped pulses (GNSS PPS), resets of DT-described devices (camera sensor, modem), PWM, and boot straps. Four fast lines cover every row.
- **Digital audio** matters for the recorder and the audio-interface class. Both can use either a native port or UAC2 (§4.3).
- **SWD/JTAG** matters for the HATs that carry MCUs or FPGAs (HCS-013): oscilloscope, VNA, radio, dev boards.
- **The analog line output** is the only way an amplifier HAT can use the T1 DAC. The line input has no sink on the T1: the CS43198 is a DAC only, and the microphones are PDM.

---

## 3. What USB offers: classes and bridges

### 3.1 Standard USB classes (no bridge-specific driver)

| Function | USB class | Linux (mainline) | Notes |
|---|---|---|---|
| Serial (console, AT, NMEA, NanoVNA protocol) | CDC-ACM | `cdc-acm` | Any USB MCU. No flow-control contacts needed |
| Network | CDC-NCM / ECM, or vendor NICs | `cdc_ncm`, `r8152` and others | — |
| Audio, capture and playback | USB Audio Class 2, asynchronous | `snd-usb-audio` | The HAT clock is the master, and the host follows through the feedback endpoint. Bit-exact. Up to 8 channels at 192 kHz / 32 bit ≈ 49 Mb/s fits USB 2 high speed |
| MIDI | USB MIDI | `snd-usb-audio` | — |
| Video | UVC | `uvcvideo` | No access to the SoC ISP |
| Display | GUD (vendor class, open protocol) | `gud` (`drivers/gpu/drm/gud`, IDs `1d50:614d`, `16d0:10a9`, `1209:4fb3`; R1 and XRGB1111 formats for monochrome and e-ink) | Open RP2040 firmware: `notro/gud-pico` |
| Input | HID | `usbhid`, `hid-sensor-hub` | Keyboards, controllers, HID sensors |
| Instruments | USBTMC | `usbtmc` | Oscilloscope/VNA option |
| Smart card / NFC | CCID | user space (pcsc-lite) | — |
| Debug and flashing | CMSIS-DAP v1 (HID) / v2 (bulk) | user space: OpenOCD `cmsis-dap`, openFPGALoader `cmsisdap`, probe-rs | Open firmware: Raspberry Pi debugprobe (RP2040/RP2350). SWD and JTAG |
| Firmware update | DFU, UF2/PICOBOOT, vendor ROM loaders | user space (`dfu-util`, `picotool`) | ROM bootloaders cannot be erased: the basis of recovery in v1-M |

### 3.2 USB-to-GPIO/I2C/SPI/UART bridges: mainline status

| Bridge | Functions in mainline | Driver (Kconfig) | Binding | Usable on the T1 (arm64, device tree) | Moved to the HAT |
|---|---|---|---|---|---|
| **DLN2 protocol** (Diolan DLN-2, or an MCU running open DLN2 firmware) | GPIO with interrupts, I2C, SPI, ADC | `MFD_DLN2`, `GPIO_DLN2`, `I2C_DLN2`, `SPI_DLN2`, `DLN2_ADC`; no ACPI dependency | Table holds only `a257:2013`. The open RP2040 firmware (`notro/pico-usb-io-board`, CC0) uses `1d50:6170` and binds through `new_id` from a udev rule | **Yes.** The ID needs a udev rule, or a one-line upstream ID patch carried until merged (OS-023 practice) | An MCU (RP2040/RP2350 class) + crystal + flash, or RP2354 (built-in flash). Same firmware can add CDC-ACM UARTs |
| **Microchip MCP2221A** | I2C, 4 GPIO, ADC and DAC (IIO); its UART is CDC-ACM | `HID_MCP2221` | Fixed Microchip ID | Yes | One SOIC-14/QFN-16 part, USB full speed |
| **Silicon Labs CP2112** | I2C/SMBus, 8 GPIO (gpiochip with IRQ chip) | `HID_CP2112` | Fixed ID | Yes | One QFN-24 part |
| **FTDI FT260** | I2C only in mainline (the driver reads UART mode but registers no tty) | `HID_FT260` | Fixed ID | Yes (I2C) | One part |
| **Microchip MCP2200** | GPIO; UART through CDC-ACM | `HID_MCP2200` | Fixed ID | Yes | One part |
| **WCH CH341A** | UART (`ch341`, serial-mode PID) **or** SPI (`SPI_CH341`, `1a86:5512`); I2C and GPIO are not mainline | `USB_SERIAL_CH341`, `SPI_CH341` | Mode set by a pin | Partly: one function per chip | One part |
| **FTDI FT232H / FT2232H (MPSSE)** | UART (`ftdi_sio`); GPIO through `GPIO_MPSSE` only for two custom IDs (SeaLevel, Bryx); SPI/I2C/JTAG in user space (libftdi, OpenOCD `ftdi`, openFPGALoader) | `USB_SERIAL_FTDI_SIO`, `GPIO_MPSSE` | Fixed IDs | UART and user-space JTAG: yes. Kernel GPIO: only with those IDs | One part + crystal + EEPROM. High speed |
| **Silicon Labs CP2102N / CP210x**, **WCH CH340** | UART (CP210x also has GPIO through `cp210x`) | `USB_SERIAL_CP210X`, `USB_SERIAL_CH341` | Fixed IDs | Yes | One part |
| **i2c-tiny-usb** | I2C | `I2C_TINY_USB` | Two fixed IDs | Yes | MCU firmware |
| **Silicon Labs CP2615** | I2C (it is a USB audio bridge) | `I2C_CP2615` | Fixed ID | Yes | One part |
| **Intel LJCA** | GPIO, I2C, SPI | `USB_LJCA` (+ `GPIO_LJCA`, `I2C_LJCA`, `SPI_LJCA`) | `depends on USB && ACPI`; clients enumerate from ACPI `_ADR`/`_HID` | **No** on a device-tree system | — |
| **Intel USBIO** (`gpio-usbio`, `i2c-usbio`) | GPIO, I2C | `USB_USBIO` | `depends on USB && ACPI` and `X86` (or `COMPILE_TEST`) | **No** | — |

**Reading of the table.** Two paths are clean on the T1. (1) Single-purpose bridge chips with fixed IDs and mainline drivers (MCP2221A, CP2112, CP2102N, CH340, FT232H for UART). (2) One MCU with open firmware that exposes several standard functions at once as a composite device: DLN2 for GPIO, I2C, SPI and ADC; CDC-ACM for serial; CMSIS-DAP for SWD/JTAG. A composite device needs no hub on the HAT. The Intel bridges named in the brief (`gpio-usbio`, `ljca`) are ACPI-only and do not apply to a device-tree board.

### 3.3 Properties a USB-bridged function does not have

| Property | Native | Through a USB bridge |
|---|---|---|
| Available before Linux | Yes (U-Boot reads the ID EEPROM and applies the overlay) | No: only after `HAT_5V`/`HAT_3V3` are on and the device has enumerated |
| Latency per transaction | µs | One USB round trip: about 1 ms at full speed, from 125 µs at high speed (**UNVERIFIED** measured figures for DLN2 and MCP2221 transfers) |
| Interrupt timing | Hardware IRQ, timestamped | Polled interrupt endpoint (1 ms FS, 125 µs HS); not usable for PPS |
| Wake from suspend | `HAT_INT#` sits on a GPIO0 wake pin (pin plan §2.5) | USB remote wake-up from the RK3566's deep suspend is **UNVERIFIED** |
| Device-tree description | Normal (`reset-gpios`, CCI sensors, ASoC codecs, PCIe sidebands) | Awkward: hot-plugged USB children have no stable DT node. Devices on a bridged I2C/SPI bus are created from user space or a driver, not from an overlay (**UNVERIFIED** for DT nodes under `dln2`) |
| Idle power | About zero | A powered MCU or bridge, plus an active USB 2 root port. A suspended USB device may draw up to 2.5 mA (USB 2.0 §7.2.3) |
| Bandwidth | SPI tens of MHz | DLN2 and HID bridges: tens to hundreds of kB/s (**UNVERIFIED**). Bulk classes at high speed: tens of MB/s |

---

## 4. What stays native, and why

### 4.1 Per class

| Class | v0 contacts | v1-R | Why |
|---|---|---|---|
| Power, ground, `HAT_DET#` | power connector | unchanged | Safety and sequencing (BRD-049, HCS-015). Presence must be known before any rail is on |
| ID EEPROM I2C | 2 | 2 | Read before `HAT_5V` is enabled and by U-Boot (HCS-007 to HCS-011, HCS-017 Draft). USB is not up yet |
| `HAT_INT#` | 1 | 1 | Wake source on a GPIO0 pin. It also becomes the interrupt for every bridged function on a HAT |
| General I2C | 2 | 2 | Camera CCI, gauges, sensors, codecs, haptics: DT-described kernel devices. Two contacts |
| PCIe `PERST#`, `CLKREQ#`, `WAKE#` | 3 | 3 | PCIe needs them. `PERST#` is gated with HAT power-good (topology §2.3). `CLKREQ#` enables L1 substates (battery) |
| Debug | 5 | 3: `DBG_SWCLK`, `DBG_SWDIO`, `HAT_RST#` | Two-wire SWD reaches every Cortex-M and the RP2040/RP2350 directly. That includes a bricked bridge MCU and MCUs without a ROM USB loader (BRD-004, HCS-013). JTAG (`TDI`, `TDO`) for FPGAs moves to a USB probe on the HAT, or to the HAT MCU writing the FPGA's flash |
| GPIO | 8 (4 fast PWM, 4 of which 3 are expander-slow) | 4: `GPIO0`–`GPIO3`, all fast, PWM-capable and interrupt-capable | PPS, resets of DT-described devices (absorbs `CAM_RST#`), modem power keys, boot straps, PWM. The slow lines were expander lines; a bridge on the HAT is the same class of latency |
| `CAM_RST#` | 1 | 0 | A camera HAT uses `GPIO0` (declared in the manifest) |
| `CAM_MCLK` | 1 | 0 | Raspberry Pi-style modules carry their own oscillator (pin plan §3.8). A bare sensor adds one oscillator on the HAT, close to the sensor |
| UART | 4 | 0 | CDC-ACM or a USB-UART bridge on the management port. Mainline drivers for all common bridges |
| SPI | 5 | 0 | DLN2 or CH341 SPI for slow devices. GUD for displays. UVC for camera-like SPI sensors. High-rate data uses USB bulk |
| Digital audio | 11 (I2S 5 + TDM in 6) | 8: `I2S_MCLK`, `I2S_BCLK`, `I2S_FS`, `I2S_DOUT`, `I2S_DIN0`–`I2S_DIN3` | One full-duplex port with one clock domain. Either side can be the clock provider. It keeps 8 channels at 192 kHz as 4 lines × 2 channels (BCLK 12.288 MHz). A codec-only audio HAT keeps working without a USB bridge. The T1's I2S2 and I2S1 RX stay in use |
| Line out L/R + 2 `AGND` | 4 | 4 | Analog: the only way an amplifier HAT uses the T1 DAC |
| Line in L/R | 2 | 0 | The T1 has no analog input sink. Capture is digitised on the HAT (benchmark §1.5) |
| High-speed | 12 pairs | 12 pairs | §4.2 |

### 4.2 High-speed lanes: can they merge?

| Option | Saves | Costs | Verdict |
|---|---|---|---|
| Drop the second USB 2 port (`USB2B_D±`) and put a hub on the HAT | 1 pair (≈ 4 pogo, 3 DF40) | All HAT USB 2 traffic shares one 480 Mb/s root port instead of two. The management functions of §3 then share a port with bulk data (SDR streams, audio). The second EHCI controller goes unused. HCS-005 justification needed | **Keep.** With low-speed I/O moving to USB, `USB2B` is the management port: always present, and independent of the USB 3 PHY and its power state |
| Hub or switch in the device-side path | — | BRD-006 (Confirmed) allows only multiplexers in the HAT path (topology §2.1) | Not possible without changing BRD-006. Hubs and switches **on a HAT** are allowed |
| Replace USB 3 with a USB host controller on a PCIe HAT | 3 pairs | Leaves the SoC's only SuperSpeed controller unused. Against BRD-044 (Confirmed) and "100 % of the hardware" | No |
| CSI-2 4 → 2 lanes | 2 pairs | Halves camera bandwidth. Against HCS-004 | No |
| PCIe without REFCLK (SRIS) | 1 pair | RK3566 SRIS support **UNVERIFIED**, and many endpoints do not support it | No |
| Tighter ground fence (shared grounds in the 4-row grid, below 4 per pair) | possibly 8–12 pogo | SI unproven. The benchmark's validation plan (§6) has to show it | Separate SI question. Not counted here |

### 4.3 HCS-004 and HCS-005

- **HCS-004** covers the high-speed interfaces. v1-R and v1-M keep all of them at full lane count, with no change to the channel. Nothing is capped.
- **HCS-005** asks for a full justification of every compromise on the exposed ports. Each removal in §4.1 has its justification in the "Why" column. Two need a maintainer view, because they trade something real:
  - **Audio, one clock domain instead of two.** v0 allows HAT stereo I/O on the device's or HAT's clock **and** at the same time multichannel capture on a second HAT clock. v1-R allows one clock domain per HAT. No reference HAT uses two audio clock domains.
  - **JTAG through a USB probe on the HAT.** A HAT with an FPGA and no MCU adds a probe chip.

---

## 5. Proposal

### 5.1 v1-R (recommended): contact list by function

| Function | Signals | Contacts | Notes |
|---|---|---|---|
| USB 3 | `USB3_SSTX±`, `USB3_SSRX±`, `USB3_D±` | 6 | Unchanged |
| USB 2, management port | `USB2B_D±` | 2 | Unchanged electrically. Recommended as the port for bridges, MCUs and probes |
| PCIe x1 | `PCIE_TX±`, `PCIE_RX±`, `PCIE_REFCLK±` | 6 | Unchanged |
| CSI-2 | `CSI_CLK±`, `CSI_D0±`–`CSI_D3±` | 10 | Unchanged |
| High-speed ground fence | `GND` | 13 DF40 / ≈ 24 pogo | Unchanged |
| Presence | `HAT_DET#` | 1 | Power connector, unchanged |
| Identification | `ID_SCL`, `ID_SDA` | 2 | Unchanged |
| Interrupt and wake | `HAT_INT#` | 1 | Also the aggregate interrupt for bridged functions |
| I2C | `I2C_SCL`, `I2C_SDA` | 2 | Unchanged, including CCI |
| PCIe sidebands | `PCIE_PERST#`, `PCIE_CLKREQ#`, `PCIE_WAKE#` | 3 | Unchanged |
| Debug and recovery | `DBG_SWCLK`, `DBG_SWDIO`, `HAT_RST#` | 3 | `HAT_RST#` replaces `DBG_RESET#`: open drain, resets the HAT's main controller |
| GPIO | `GPIO0`–`GPIO3` | 4 | All fast, PWM- and interrupt-capable. A device that cannot offer that on all four does not meet v1 |
| Digital audio | `I2S_MCLK`, `I2S_BCLK`, `I2S_FS`, `I2S_DOUT`, `I2S_DIN0`–`I2S_DIN3` | 8 | One clock domain. Either side can provide the clock |
| Analog | `LINE_OUT_L`, `LINE_OUT_R`, `AGND` ×2 | 4 | Line in removed |
| Low-speed grounds | `GND` | 5 | Was 8 for 43 signals |
| Reserved (HCS-022) | `RSVD_S0`, `RSVD_S1`, `RSVD_P0`–`RSVD_P4` | 7 | As v0. The signal connector also has 5 free positions (§5.3) |
| Power | `HAT_5V`, `HAT_3V3`, `GND` | DF40 12 / 6 / 22; pogo 3 / 2 / 4 | Unchanged (§5.2) |

Native low-speed signals: 24 (v0: 44).

### 5.2 Power and ground

Power does not change. The `HAT_5V` and `HAT_3V3` ratings (2.0 A and 1.0 A) and their contact counts follow from BRD-049 and the 0.3 A DF40 rating (power tree §4.6: 56 % and 64 % per contact continuous, 83 % at the class F fault ceiling).

Dropping `HAT_3V3` in favour of a small auxiliary rail was considered and **not proposed**. On battery, the direct VSYS → 3.3 V buck is more efficient than VSYS → 5.1 V boost → 3.3 V buck on the HAT: about 0.90 against 0.88 × 0.90 ≈ 0.79 (power tree efficiency figures, **UNVERIFIED**). Most HAT logic and bridge MCUs run at 3.3 V.

One addition HATs must follow: USB devices on a HAT are self-powered and take their VBUS-sense input from `HAT_5V` or `HAT_3V3`. The interface has no VBUS contact, as in v0.

### 5.3 Counts

| | v0 | **v1-R** | v1-M (§5.5) |
|---|---|---|---|
| Native low-speed signals (incl. `HAT_DET#`) | 44 | **24** | 12 |
| Analog (incl. `AGND`) | 6 | **4** | 4 |
| Low-speed grounds | 8 | **5** | 3 |
| Reserved | 7 | **7** | 5 |
| High-speed signal + fence (pogo) | 48 | **48** | 48 |
| Power + power ground (pogo) | 9 | **9** | 9 |
| **Pogo contacts** | **≈ 122** | **97 (−25, −20 %)** | **81 (−41, −34 %)** |
| DF40 signal connector, used | 96 of 96 | **71** | 55 |
| DF40 signal connector, smallest size | 100 | **80 (76 usable, 5 spare)** | 60 (56 usable, 1 spare) |
| DF40 power connector | 46 of 46 (50) | **46 of 46 (50), unchanged** | 46 of 46, with all 5 reserved contacts |
| Spring force at 25–35 g per pin (benchmark §4.1) | 3.1–4.3 kgf | **2.4–3.4 kgf** | 2.0–2.8 kgf |
| Pogo grid at 1.27 mm, 4 rows | 4 × 31 ≈ 39 mm | **4 × 25 ≈ 32 mm** | 4 × 21 ≈ 27 mm |

DF40 signal count for v1-R: 37 (12 pairs in GSSG) + 23 low-speed + 4 analog + 5 grounds + 2 reserved = 71. The family has 60- to 120-position parts (DF40 catalog 2020, p. 10, note 2, in the parts library sheet). Stock of the 80- and 60-position sizes was **not checked** this session. Keeping the 100-position pair and using the 25 spare positions as extra grounds and reserved contacts is also valid. The flat cable is mated once at assembly (30 cycles), so its count matters less than the pogo count, which is the swap interface and sets the size and force of the device-side piece (HCS-023).

### 5.4 T1 mapping consequences (board, for information)

HCS-021 keeps SoC names out of the standard. These notes only check that the T1 can implement v1-R.

- **Freed on the board:** UART4 M0 (1_A4–1_A7), SPI3 M1 (4_C2, 4_C3, 4_C5, 4_C6, 4_D1), the JTAG TDI/TDO balls (1_A2, 1_A3), 4_C4, `REFCLK_OUT` (0_A0), and four TCA6416A lines (three slow GPIO and `CAM_RST#`). Pin plan §3.11 calls the 3.3 V GPIO budget the real bottleneck, so this gives margin back.
- **Audio port:** `I2S_BCLK`/`I2S_FS` would join the I2S2 M1 clock pins and the I2S1 RX clock pins (SCLKRX, LRCKRX) on one net. `I2S_DOUT` would come from I2S2 SDO, and `I2S_DIN0`–`DIN3` would go to I2S1 SDI0–3. Running I2S2 TX and I2S1 RX from one shared clock net, with the SoC as provider on I2S2 and consumer on I2S1 RX, is **UNVERIFIED** in `rockchip_i2s_tdm.c` and must be bench-tested. Fallback: the device acts as consumer only on this port, with the HAT always providing the clock. That fallback loses codec HATs without an oscillator.
- **Parts:** HAT audio translator bits 12 → 8 (BOM line 95; the exact part mix is open). Low-speed ESD on the device-side piece drops from 12 to about 7 TPD4E05U06 and from 4 to 2 TPD1E10B06 (lines 68–69). The PCA9306, eFuses, TVS and AC-coupling capacitors do not change.

### 5.5 v1-M (maximum reduction), for comparison

Native low-speed: `ID_SCL`, `ID_SDA`, `HAT_DET#`, `HAT_INT#` (merged with `PCIE_WAKE#`; both open drain, active low), `I2C_SCL`, `I2C_SDA`, `PCIE_PERST#`, `PCIE_CLKREQ#`, `HAT_RST#`, `HAT_BOOT#` (forces the HAT controller into its ROM USB bootloader), `GPIO0`, `GPIO1` = 12.

On top of v1-R it moves digital audio to UAC2 and all flashing and debugging to ROM bootloaders plus a CMSIS-DAP probe on the HAT. It keeps two GPIO lines (PPS, a sensor reset) and the line output. Further steps were checked and rejected: dropping `USB2B` (§4.2) gives 77, and also dropping the line output gives 73.

What v1-M requires of every programmable HAT: its main controller must have a ROM USB bootloader (RP2040/RP2350 BOOTSEL, STM32 system DFU, ESP32-S3 USB ROM) entered by `HAT_BOOT#`. Every other programmable chip must be reachable from that controller. MCUs without a ROM USB loader (for example the nRF52 family, **UNVERIFIED** for every variant) cannot be the main controller.

---

## 6. Reference HATs on v1-R and what HAT makers add

### 6.1 Reference HATs

| HAT | Uses in v1-R | Change against v0 | Parts added on the HAT |
|---|---|---|---|
| Powerbank | Power (class F), ID, `HAT_DET#`, `HAT_INT#`, I2C (gauge, charger), `GPIO0` (optional), SWD (if it has an MCU) | None | None |
| NAS | PCIe (+ sidebands), optional USB 3, I2C, `GPIO0` (fan PWM), ID, power class B | None | None |
| Camera | CSI-2 4 lanes, I2C (CCI), `GPIO0` as sensor reset, ID; optional `USB2B` for a controls MCU, optional I2S for a microphone | `CAM_RST#` → `GPIO0`; `CAM_MCLK` gone | One oscillator (24 MHz class) if the sensor module has none |
| Recorder | I2S port (capture up to 8 ch / 192 kHz on 4 lines, stereo playback for monitoring), I2C (codec control), `GPIO`, `HAT_INT#`, optional line out, optional `USB2B` (controls as HID; UAC2 as an alternative path) | One clock domain for capture and playback | None, if one clock domain is enough |
| Oscilloscope | `USB2B` (sample stream and control from the HAT MCU), SWD (MCU), JTAG to the FPGA through the MCU or a probe on the HAT, `HAT_INT#`, ID | SPI and UART gone; native JTAG reduced to SWD | A high-speed USB path if the design needs more than full speed (an MCU with an HS PHY, or an FT232H-class FIFO bridge), and a JTAG path to the FPGA |
| VNA | `USB2B` (CDC-ACM, as the NanoVNA does), SWD, ID | None in practice | None (the NanoVNA MCU already has USB) |
| Radio | `USB2B` or USB 3 (HackRF stream), ID, optional SWD | None | None (HackRF is a USB design) |

### 6.2 What a third-party HAT adds, by function it used in v0

| Function used in v0 | v1-R path | Typical added parts |
|---|---|---|
| UART | CDC-ACM from the HAT MCU, or a USB-UART bridge | 0 (MCU already present) or 1 (CP2102N, CH340 class) |
| SPI device | DLN2 firmware on an MCU, CH341A in SPI mode, or the device's USB-native variant | 1 bridge, or an MCU + crystal + flash |
| I2C beyond the native bus, ADC, slow GPIO | MCP2221A, CP2112, or DLN2 on an MCU | 1 |
| JTAG (FPGA) | CMSIS-DAP firmware on the HAT MCU, an FT2232H-class probe, or configuration from the HAT MCU | 0–3 (FT2232H + crystal + EEPROM) |
| Several USB functions on one HAT | One MCU composite device (DLN2 + CDC-ACM + CMSIS-DAP), or a USB 2 hub (USB2514B, GL850G; mainline `onboard_usb_dev` handles their reset and supply) | 0 or 1 hub + crystal |
| Camera MCLK | On-HAT oscillator | 1 |
| Line input | ADC on the HAT, through the I2S port | ADC already needed for any quality capture |

Any HAT that adds a bridge also adds a USB 2 pair route from the pads to the bridge (90 Ω, short) and its VBUS-sense connection (§5.2). In return it loses pads and keep-out area.

---

## 7. Requirements and documents affected

| Item | Status | Effect of v1-R | Effect of v1-M |
|---|---|---|---|
| BRD-004 "The SoC shall reach every chip on the board and on attached HATs directly, for debugging, flashing and operation." | **Confirmed** | Met for flashing (native SWD). For operation, chips behind a HAT-side bridge are reached through it: the maintainers must say whether that counts as "directly" | Same question, and flashing too depends on a HAT-side ROM loader or probe |
| BRD-005 "Helper microcontrollers … shall never sit in a data path or limit capabilities." | **Confirmed** | Written for the board. A bridge MCU on a HAT sits in that HAT's data path. The maintainers must say whether BRD-005 binds HATs | Same, and more central |
| BRD-006, BRD-044, BRD-045 | **Confirmed** | No change | No change |
| HCS-004, HCS-005 | **Confirmed** | High-speed unchanged. Low-speed removals justified in §4.1; audio and JTAG need sign-off (§4.3) | Plus audio over UAC2 and flashing over USB |
| HCS-006 | **Confirmed** | No change | No change |
| HCS-007 to HCS-011 | **Confirmed** | No change (ID bus native) | No change |
| HCS-013 (MCUs, FPGAs, SoCs flashed, operated and debugged from Blackbird OS) | **Confirmed** | Met: SWD native, JTAG through a HAT-side probe. The standard would need a rule that FPGAs are reachable through a CMSIS-DAP or FTDI-class probe or through the HAT MCU | Needs a rule that every programmable HAT has a ROM-loader controller (§5.5) |
| HCS-015 (power classes) | Draft | No change | No change |
| HCS-021, HCS-022 | **Confirmed** | Met: signals by function; 7 reserved contacts kept | Met with 5 reserved |
| HCS-023 (device-side piece as an accessory) | **Confirmed** | Smaller, lighter accessory: about 25 fewer pogo pins and ESD channels, and a shorter pogo field | More so |
| Interface v0 | Draft | Rewritten as v1: §2 classes, §4 positions, §5 reserved; open point 2 closed (no line in) | — |
| Board pin plan §2.1, §2.5, §4.3; BOM lines 66, 68, 69, 93, 95 | Proposed | Audio port remap (§5.4), freed balls, fewer translators and ESD parts, possibly an 80-position DF40 | — |
| Benchmark §3 pin budget | research | Superseded by these counts | — |

---

## 8. HATs that get worse or impossible

| HAT or use | v1-R effect | Alternative |
|---|---|---|
| SPI TFT display HAT (`panel-mipi-dbi` class) | No native SPI. GUD over a full-speed MCU sends about 1.2 Mb per 320 × 240 × 16-bit frame against 12 Mb/s, so about 8 fps uncompressed (GUD compresses, so the real figure may be higher, **UNVERIFIED**) | GUD on an MCU with high-speed USB. E-ink is unaffected (R1 format, slow refresh). DSI on the HAT is already excluded (board ADR 0006) |
| Prototyping / dev HAT expecting Raspberry Pi-style raw pins | SPI, UART and most GPIO come through a bridge, so no kernel-timed bit-banging on those pins | An RP2350 on the HAT (PIO, many GPIO, CMSIS-DAP and DLN2 firmware) plus the four native GPIO lines, I2C and SWD |
| Kernel `serdev` devices (Bluetooth HCI UART, serdev GNSS, UART modules described in DT) | Not possible: `serdev` needs a native, DT-described UART | The USB variant of the same function (`btusb`, CDC-ACM GNSS, USB modems) |
| Bare SPI chips with DT-only kernel drivers (IIO SPI ADCs, some radios) | Through `spi-dln2` they need instantiation from user space or a driver, not an overlay (**UNVERIFIED** for DT under `dln2`) | Choose I2C or USB variants. Put an MCU between the chip and the device |
| Audio HAT with two simultaneous clock domains | Not possible (one clock domain) | UAC2 for one of the two streams |
| FPGA-only HAT without an MCU | Needs a JTAG probe chip | FT2232H-class probe, or a small MCU with CMSIS-DAP |
| Low-latency audio DSP or effects HAT | v1-R keeps the native port, so it is unaffected. **v1-M** adds the UAC2 buffering latency (a few ms, **UNVERIFIED**) | v1-R |
| Codec-only audio HAT (ADC or DAC + crystal, no MCU) | Unaffected in v1-R. **v1-M** makes it impossible without a UAC2 bridge | v1-R |
| HAT that needs a device-supplied clock (bare sensor without an oscillator) | Adds one oscillator | — |
| HAT that relied on the slow GPIO4–7 lines | Must use a bridge, which has the same latency class (expander about 100 µs, USB 125 µs–1 ms) | MCP2221A / CP2112 |
| Analog line-input HAT feeding a device ADC | Not possible, but the T1 has no such ADC anyway | Digitise on the HAT |

No reference HAT becomes impossible under v1-R. Under v1-M, a recorder built as a codec-only design would need a UAC2 bridge.

---

## 9. Decisions needed from the maintainers

1. **Bridges on HATs under BRD-004 and BRD-005 (both Confirmed):** do these requirements bind HAT hardware, and does a standard USB class or mainline bridge driver count as "directly"? Everything in this report that moves off the interface depends on this answer.
2. **Reduction level:** v0 (no change), **v1-R** (recommended, 97 pogo) or v1-M (81 pogo).
3. **Audio port:** accept one clock domain per HAT (8 lines instead of 11), subject to the I2S2 + I2S1 RX bench test (§5.4)? In v1-M: accept UAC2 as the only HAT audio path?
4. **Debug:** two-wire SWD + reset native, JTAG through a HAT-side probe (v1-R); or full JTAG native (v0); or ROM bootloader only (v1-M)?
5. **Line in:** drop it (no sink on the T1)? This closes interface v0 open point 2 for the input side.
6. **GPIO:** four fast lines, with the device obliged to make all four PWM- and interrupt-capable?
7. **DF40 signal connector:** move to the 80-position size (stock to check), or keep 100 with the spare positions as grounds and reserved contacts?
8. **Reserved contacts:** keep 7 as in v0, or use the freed space for more (for example a reserved ground-fenced pair site in the pogo pad map, kept free on HATs, for a future device with a second PCIe or USB 3 lane)?
9. **Reference bridge firmware:** should the standard name a recommended composite firmware (DLN2 + CDC-ACM + CMSIS-DAP) and its USB ID policy (the `dln2` ID table holds only `a257:2013`)? This touches the software stack, which the maintainers have not opened yet.

---

## 10. Sources

### OpenDAP documents

- HAT Case Standard requirements: [`../requirements/README.md`](../requirements/README.md); [HAT interface v0](../specs/interface-v0.md); [ADR 0001](../adr/0001-all-pogo-interface-with-fixed-device-side-piece.md); [HAT interconnect benchmark](2026-10-hat-interconnect-benchmark.md).
- Board: [requirements](https://github.com/opendap-project/board/blob/main/docs/requirements/README.md) (BRD-004, 005, 006, 044, 045, 049); [high-speed I/O topology](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-high-speed-io-topology.md); [pin multiplexing v0](https://github.com/opendap-project/board/blob/main/docs/research/2026-10-pin-multiplexing-v0.md); [BOM v1](https://github.com/opendap-project/board/blob/main/docs/specs/bom-v1.md); [power tree v0](https://github.com/opendap-project/board/blob/main/docs/specs/power-tree-v0.md) §4.6, §4.10; [ADR 0006](https://github.com/opendap-project/board/blob/main/docs/adr/0006-pin-plan-decisions.md).
- Parts library: `Hirose/DF40C-100DS-DP-0.4V.md` (DF40 catalog 2020: 0.3 A per contact, 30 cycles, corner hold-downs, 60- to 120-position parts).
- Reference HAT requirements: `hat-powerbank`, `hat-nas`, `hat-camera`, `hat-recorder`, `hat-oscilloscope`, `hat-vna`, `hat-radio` (`docs/requirements/README.md`).

### Mainline Linux (`torvalds/linux` master, 7.3, read 2026-10-22)

- [`drivers/mfd/dln2.c`](https://github.com/torvalds/linux/blob/master/drivers/mfd/dln2.c), [`gpio-dln2.c`](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-dln2.c), [`i2c-dln2.c`](https://github.com/torvalds/linux/blob/master/drivers/i2c/busses/i2c-dln2.c), [`spi-dln2.c`](https://github.com/torvalds/linux/blob/master/drivers/spi/spi-dln2.c), [`dln2-adc.c`](https://github.com/torvalds/linux/blob/master/drivers/iio/adc/dln2-adc.c)
- [`drivers/usb/misc/usb-ljca.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/misc/usb-ljca.c), [`drivers/usb/misc/usbio.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/misc/usbio.c), [`drivers/gpio/gpio-usbio.c`](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-usbio.c), and `drivers/usb/misc/Kconfig` (`USB_LJCA`, `USB_USBIO`: `depends on USB && ACPI`)
- [`hid-mcp2221.c`](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-mcp2221.c), [`hid-cp2112.c`](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-cp2112.c), [`hid-ft260.c`](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-ft260.c), [`hid-mcp2200.c`](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-mcp2200.c), `drivers/hid/Kconfig`
- [`spi-ch341.c`](https://github.com/torvalds/linux/blob/master/drivers/spi/spi-ch341.c), [`usb/serial/ch341.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/serial/ch341.c), [`usb/serial/cp210x.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/serial/cp210x.c), [`gpio-mpsse.c`](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-mpsse.c), [`i2c-tiny-usb.c`](https://github.com/torvalds/linux/blob/master/drivers/i2c/busses/i2c-tiny-usb.c), [`i2c-cp2615.c`](https://github.com/torvalds/linux/blob/master/drivers/i2c/busses/i2c-cp2615.c)
- [`drivers/gpu/drm/gud/gud_drv.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/gud/gud_drv.c), [`drivers/gnss/usb.c`](https://github.com/torvalds/linux/blob/master/drivers/gnss/usb.c), [`drivers/usb/misc/onboard_usb_dev.h`](https://github.com/torvalds/linux/blob/master/drivers/usb/misc/onboard_usb_dev.h), [`cdc-acm.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/class/cdc-acm.c), [`usbtmc.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/class/usbtmc.c), [`hid-sensor-hub.c`](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-sensor-hub.c)

### Firmware and tools

- Raspberry Pi Pico USB I/O board (DLN2 protocol + CDC UARTs, CC0, `1d50:6170`): <https://github.com/notro/pico-usb-io-board> and its [wiki](https://github.com/notro/pico-usb-io-board/wiki)
- GUD on the Pico: <https://github.com/notro/gud-pico>
- openFPGALoader cable list (`cmsisdap`): <https://github.com/trabucayre/openFPGALoader/blob/master/doc/cable.yml>
- Raspberry Pi debugprobe (CMSIS-DAP firmware): <https://github.com/raspberrypi/debugprobe> (not re-read this session)

**Not checked this session:** DF40 60- and 80-position stock; measured USB-bridge transaction latencies; USB remote wake-up on the RK3566; the HackRF Pro USB generation; DT support for children of `dln2`; whether the RK3566 I2S-TDM block runs I2S2 TX and I2S1 RX from one shared clock net.
