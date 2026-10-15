# ZN6 uaEFI Adapter

A DIY adapter board that plugs the [rusEFI](https://rusefi.com) **uaBrain** into the factory ECU connector of the **ZN6 platform** (Toyota 86 / Scion FR-S / Subaru BRZ, 2012–2020).

![Board render](ZN6_UAEFI_Adapter.png)

## What it does

The board has the same 135-pin ECU header as the factory ECU, so it plugs straight into the car's harness. Power, ground and CAN are hardwired. Every other ECU pin is mirrored to a labelled solder-pad field, and every uaBrain pin is broken out to labelled headers. You connect the two with jumper wires.

That means you can wire up **any engine swap** on the factory body harness. You aren't locked into the FA20 pinout: route each harness pin to whatever uaBrain function your engine needs.

My own build is an **EZ36** (Subaru 3.6 L H6) swap. The pin plan for it is in the `H6 Tribeca` sheet of [`Wiring.xlsx`](Wiring.xlsx).

On top of the uaBrain's own I/O, the board adds:

- **Constant 3.3 V RTC supply**: a regulator fed from the car's always-on battery feed keeps the uaBrain real-time clock running with the key off.
- **8-channel I/O expander**: an ADS7128 on the uaBrain I2C bus gives 6 analog/digital inputs and 2 protected high-current low-side outputs.

> **Status: REV A, untested.** Check the pinout against your own harness before powering anything.

### ECU pin numbering

The factory ECU uses four connectors, which the TE 6376430-2 combines into one 135-position header. The pad numbers in the schematic map to the factory connector pins like this:

| Factory connector | Pins | Pad numbers |
|---|---|---|
| A (A33) | A1–A31 | 1–31 |
| B (A34) | B1–B35 | 32–66 |
| C (A35) | C1–C35 | 67–101 |
| D (A36) | D1–D34 | 102–135 |

For example, `B28` is pad 31 + 28 = **59**. The factory function of every pin is in the `BRZ FRS Pinout 2018` sheet of [`Wiring.xlsx`](Wiring.xlsx).

## Fixed pins

These connections are hardwired on the board and can't be changed with jumper wires.

### ECU side (AC1 / AC2)

| ECU pin | Pad | Factory function | Board net | Goes to |
|---|---|---|---|---|
| A1 | 1 | Switched power supply | `+12V` | uaBrain M1 (+12 V main), TVS D2 |
| A2 | 2 | Keep-alive memory power (battery) | `+12V_CONSTANT` | D1 → 3.3 V LDO → `VBAT` (TP1), TVS D4 |
| A4 | 4 | EC (ECU ground) | `GND` | Ground |
| A18 | 18 | CAN Lo | `CAN1-` | uaBrain N6 (CAN1 L), ESD D5 |
| A19 | 19 | CAN Hi | `CAN1+` | uaBrain N5 (CAN1 H), ESD D5 |
| A27 | 27 | Ignition switch | `12V_KEY` | uaBrain M2 (key voltage sense), TVS D3 |
| A28 | 28 | Sensor ground (shield) | `GND` | Ground |
| B1 | 32 | Power ground | `GND` | Ground |
| B2 | 33 | Power ground | `GND` | Ground |
| B3 | 34 | Power ground | `GND` | Ground |
| B29 | 60 | Sensor ground (shield) | `GND` | Ground |
| B30 | 61 | Shield | `GND` | Ground |
| B35 | 66 | Sensor ground (shield) | `GND` | Ground |
| D3 | 104 | Power ground | `GND` | Ground |
| D4 | 105 | Power ground | `GND` | Ground |

The factory CAN bus on A18/A19 is wired to the uaBrain's **CAN1** (the "CAN2" in Toyota's naming). All other ECU pins are free to jumper.

### uaBrain side

| uaBrain pin | Function | Connected to |
|---|---|---|
| M1 | +12 V main | ECU A1 |
| M2 | Key voltage | ECU A27 |
| N5 | CAN1 H | ECU A19 |
| N6 | CAN1 L | ECU A18 |
| C3, C8, G8, K3, K8, M3, M4, P5 | Power ground | `GND` (ECU grounds above) |
| L3 | +5 V sensor supply | Expander AVDD, input pull-ups, J14 `5V` |
| L4 | GNDA | Expander ground, J12 `GNDA` |
| L5 | +3.3 V analog | Expander DVDD |
| L6 | SDA | Expander I2C |
| L8 | SCL | Expander I2C |

Every other uaBrain pin (injectors, coils, low-sides, H-bridges, VR/Hall inputs, analog inputs, wideband, EGT, USB, etc.) is free and broken out on J1, J2, J4 and J5.

## Constant 3.3 V RTC supply

A MIC5205-3.3 LDO runs from the constant battery feed (ECU A2) through reverse-polarity diode D1. Its output is the `VBAT` net on **TP1**, labelled `VBAT CONST 3V3`.

**Run a jumper wire from TP1 to the uaBrain's VBAT pad** to keep its RTC running while the car is off.

## 8-channel I/O expander

A TI **ADS7128** 8-channel 12-bit ADC/GPIO sits on the uaBrain I2C bus (L6/L8) at address **0x16**, which is set by R1/R9. It's powered from the uaBrain 5 V sensor rail (AVDD) and 3.3 V analog rail (DVDD).

| Channel | Use | Connector |
|---|---|---|
| AIN0–AIN5 | Inputs, analog or digital, 0–5 V, low speed. 10 k pull-up to 5 V, 4.7 k / 10 nF RC filter. | J13 `ADIN 0`–`5` |
| AIN6 | GPIO output that drives the gate of `LS_AUX_1` | J3 pin 1 |
| AIN7 | GPIO output that drives the gate of `LS_AUX_2` | J3 pin 2 |

`LS_AUX_1` and `LS_AUX_2` are the two channels of an onsemi **NCV8402ADD** protected low-side driver:

- 42 V clamp, 2 A+ per channel
- Short-circuit protection, and thermal shutdown with auto-restart
- Built-in clamp for inductive loads such as relays and solenoids

J14 (6× `5V`) and J12 (6× `GNDA`) sit next to J13, so each input has its own supply and ground for a sensor or switch.

> rusEFI firmware support for an external ADS7128 is still being confirmed.

## Open questions

These are still being checked with the rusEFI folks (see [`Questions for RUSEFI.txt`](Questions%20for%20RUSEFI.txt)):

- Is uaBrain `IN_PPS2` shared with `IN_AUX2`?
- Low Side 7 (G6) doesn't appear to be mapped in firmware yet.
- Are any uaBrain low-side outputs safe to switch a constantly powered load, such as the EFI relay?
- Pin-by-pin verification of `Wiring.xlsx`.

## Repository contents

| File | Contents |
|---|---|
| `ZN6_UAEFI_Adapter.kicad_pro/.kicad_sch/.kicad_pcb` | KiCad project, main schematic and PCB |
| `ADS712.kicad_sch` | Sub-sheet for the I/O expander and low-side driver |
| `ZN6_UAEFI_Adapter.pdf` | Schematic PDF |
| `Wiring.xlsx` | Factory ZN6 ECU pinout, engine junction (AC1/AC2) mapping, EZ30/EZ36 pinouts, uaBrain pin list, 2018 cluster pinout |
| `ZN6.pretty`, `6376430-2/`, `ADS7128IRTER/`, `hw-uaBrain.*`, `ZN6_Adapter.kicad_sym` | Footprint and symbol libraries |

## References

- [rusEFI uaEFI / uaBrain](https://github.com/rusefi/rusefi/wiki)
- [ZN6 2018 wiring diagram](https://drive.google.com/file/d/1gPjDucamgb3sumqR6xQpNT1HQZo3DOKR/view)
- [FT86Club ECU pinout thread](https://www.ft86club.com/forums/showthread.php?t=143079)
- [TE 6376430-2 connector](https://www.digikey.com/en/products/detail/te-connectivity-amp-connectors/6376430-2/18111199)

## License

[MIT](LICENSE) © 2026 Jacky4566 
