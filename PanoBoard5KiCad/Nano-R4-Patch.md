# PanoBoard V5 — Arduino Nano R4 patch

[Back to PanoBoard](../README.md)

This guide describes how to use an **Arduino Nano R4** instead of the original ATmega328P-based Nano on an existing PanoBoard V5 PCB.

The used signals retain their header positions, so this conversion does not require PCB trace cuts or wire patches. **The five resistor changes below are required before operating the board with the Nano R4.** The existing KiCad schematic and PCB files still describe the original assembly; this guide documents the rework rather than a revised PCB.

## Why migrate to the Nano R4?

The classic Nano remains suitable for basic operation, but its CPU and memory limit more advanced control and diagnostics. The R4 keeps the Nano format and 5 V logic while adding processing and memory headroom.

| Feature | Classic Nano | Nano R4 |
|---|---|---|
| Microcontroller | 8-bit ATmega328P | 32-bit Renesas RA4M1, Arm Cortex-M4 |
| Clock | 16 MHz | 48 MHz |
| SRAM | 2 kB | 32 kB |
| Flash | 32 kB | 256 kB |
| GPIO logic | 5 V | 5 V |
| USB and D0/D1 | USB-to-UART bridge shares the UART | USB-C serial and hardware UART are separate |
| VIN regulation | Linear regulator | Switching regulator |

This provides room for motion control, encoder processing, larger communication buffers, calibration and logging. Keeping 5 V logic allows the existing L298, 5 V encoder wiring and surrounding circuitry to remain in use. The familiar footprint also keeps the conversion small; check USB-C plug and enclosure clearance.

See the [Arduino Nano R4 datasheet](https://docs.arduino.cc/resources/datasheets/ABX00142-datasheet.pdf), particularly sections 2, 3 and 4.

## 1. Reduce the GPIO load

Arduino specifies **8 mA maximum per Nano R4 GPIO**. PanoBoard V5 directly drives several LEDs and the camera optocouplers through 220 Ω resistors. These loads can exceed that limit. **D5 is shared by the shutter optocoupler and its indicator LED**, so their currents must be added. A lower PWM duty cycle reduces average current, not the current during each HIGH pulse.

Replace these resistors:

| Reference | Original | Nano R4 patch | Function |
|---|---:|---:|---|
| R1 | 220 Ω | **1 kΩ** | Sony shutter optocoupler, Nano D5 |
| R2 | 220 Ω | **1 kΩ** | Sony focus optocoupler, Nano D6 |
| R8 | 220 Ω | **1 kΩ** | Shutter indicator LED, also on Nano D5 |
| R11 | 220 Ω | **1 kΩ** | Motor/PWM indicator LED, Nano D9 |
| R12 | 220 Ω | **1 kΩ** | Motor/PWM indicator LED, Nano D3 |

**R10 = 1 kΩ remains unchanged.** Its power indicator is connected to the 5 V rail rather than a GPIO. The V5 resistor footprints are R1210.

At nominal 5 V, the replacement resistors reduce each branch to a few milliamps. For example, assuming an optocoupler forward voltage of 1.1 V and a visible LED forward voltage of 1.8 V:

```text
I_D5 ≈ (5.0 V - 1.1 V) / 1 kΩ + (5.0 V - 1.8 V) / 1 kΩ
     ≈ 3.9 mA + 3.2 mA
     ≈ 7.1 mA
```

This is an estimate, not a guaranteed worst-case bound. Check the combined D5 current at the actual supply voltage, including resistor tolerance and LED forward-voltage variation. If more margin is needed, reduce the indicator current further or use a suitable driver stage; do not exceed the GPIO limit.

The SFH618A optocouplers operate at low LED currents, but camera compatibility still depends on the camera input current and the achieved output LOW level. Verify both shutter and focus after changing R1/R2. See the [Vishay SFH618A/SFH6186 datasheet](https://www.vishay.com/docs/83673/sfh618a.pdf).

## 2. Power supply

The existing PanoBoard 9–12 V input can continue to feed Nano VIN:

```text
PanoBoard input (9–12 V) -> Nano R4 VIN -> onboard converter -> +5 V logic rail
```

The Nano R4 VIN range is 6–21 V and its onboard buck converter generates 5 V. No additional supply modification is required solely for the controller change, provided the connected logic/peripheral load is within the available supply capacity.

The motor supply V+ and its external converter remain separate from the Nano's 5 V logic supply. The stated VIN range is not a motor-voltage rating.

## 3. Header and pin compatibility

| PanoBoard connection | Nano R4 |
|---|---|
| Encoder SDA | A4 |
| Encoder SCL | A5 |
| Bluetooth receive | D0 / RX |
| Bluetooth transmit | D1 / TX |
| Motor direction | D2, D4, D11, D12 |
| Motor enable/PWM | D3, D9 |
| Camera shutter/focus | D5, D6 |
| Power | VIN, +5 V and GND |

D3, D5, D6, D9 and D11 support PWM on the R4.

One header position changes function: the second RESET position of the classic Nano becomes **BOOT** on the R4. Arduino identifies this as position 13 of the R4's JP1 analog/power header; it is **not digital pin D13**. This position is unconnected on the PanoBoard V5 PCB, so it needs no rework here.

See the [Nano R4 pinout](https://docs.arduino.cc/resources/pinouts/ABX00142-full-pinout.pdf) and the [V5 schematic](PanoBoardV5.kicad_sch) / [PCB](PanoBoardV5.kicad_pcb).

## 4. AS5048B encoder I²C pull-ups

Keep the encoder bus on **A4/SDA and A5/SCL**, with the encoders configured for 5 V operation. A level shifter is not needed for that 5 V arrangement. The R4's separate Qwiic connector is a different, 3.3 V interface and is not a replacement for this wiring.

The PanoBoard V5 carrier has no dedicated SDA/SCL pull-ups. Inspect the actual encoder adapter boards; some boards contain pull-ups and others require external ones.

Provide a suitable effective pull-up on each line, typically 2.2–4.7 kΩ for a short bus:

```text
SDA --- pull-up resistor --- +5 V
SCL --- pull-up resistor --- +5 V
```

A pair of external **4.7 kΩ** resistors is a practical starting point if no pull-ups are fitted. If both encoder boards already provide 4.7 kΩ on each line, the parallel combination is approximately 2.35 kΩ. This is normally suitable for a short 5 V bus.

Two 2.2 kΩ sets would produce about 1.1 kΩ; avoid adding unnecessarily strong parallel pull-ups. Check the effective resistance, bus capacitance and signal rise time. Do not rely solely on the MCU's internal pull-ups.

Reference: [AS5048A/AS5048B manufacturer datasheet, hosted by Mouser](https://www.mouser.com/datasheet/2/588/asset_pdf_25493108-3418798.pdf).

## 5. Firmware changes

### Separate Bluetooth UART from USB

On the classic Nano, `Serial` uses the D0/D1 hardware UART. On the R4:

```cpp
Serial   // USB-C serial to the computer
Serial1  // hardware UART on D0/RX and D1/TX
```

When porting an older sketch that talks to the BlueSMiRF on D0/D1, change that communication to `Serial1`. Keep the baud rate matched to the wireless module; changing the controller does not itself require changing the baud rate.

For example, with the current firmware's Bluetooth baud rate:

```cpp
Serial1.begin(57600);  // Bluetooth on D0/D1
Serial.begin(115200); // independent USB diagnostics
```

Route Bluetooth protocol reads and writes to `Serial1`, and USB diagnostics to `Serial`. Do not mix diagnostic text into the control protocol.

[PanoDuinoControl 2.4.75](https://github.com/TomNaiser/PanoDuinoControl) already selects `Serial1` for the regular Nano R4 Bluetooth build. The examples above explain older-sketch porting; they do not require replacing that firmware's transport abstraction.

### I²C

Continue using the standard bus:

```cpp
#include <Wire.h>

Wire.begin(); // A4/SDA and A5/SCL
```

The AS5048B supports 400 kHz Fast-mode. If wiring and rise times permit it, an older sketch can optionally select:

```cpp
Wire.setClock(400000);
```

A clock increase is not required for this migration. Start with a stable bus and verify both encoders before increasing speed. The Qwiic bus uses `Wire1`; keep the existing PanoBoard encoders on `Wire`.

### PWM and motor control

Port through the Arduino API, including `pinMode()`, `digitalWrite()`, `analogWrite()`, `millis()` and `micros()`.

AVR-specific register accesses must be rewritten. Examples include `TCCR1A`, `TCCR1B`, `OCR1A`, `OCR1B`, `TIMSK1`, `PORTB` and `PORTD`; these registers do not exist on the RA4M1.

Re-check PWM frequency, control-loop timing and PID behaviour after the port. Identical Arduino pin numbers do not guarantee the same timer behaviour or PWM frequency as the classic Nano.

## 6. Bring-up

1. With power disconnected, fit the Nano R4 and replace R1, R2, R8, R11 and R12 with 1 kΩ.
2. Initially power the logic without motors connected.
3. Check the +5 V rail, encoder supply and the combined D5 load.
4. Check that SDA and SCL idle near +5 V and that external pull-ups are present.
5. Run an I²C scanner and confirm both configured encoder addresses.
6. Check shutter and focus optocouplers with the camera.
7. Test the L298 direction signals.
8. Check the PWM signals and then test motor outputs at low duty cycle.
9. Connect and test the complete mechanism; re-check position control, PID behaviour and timing under load.

This is a conversion guide, not a new electrical or mechanical acceptance test. Passing a firmware build alone does not validate GPIO currents, camera triggering or loaded motor behaviour.

## Summary

- No PCB trace cuts or wire patches for the used signal positions.
- Replace **R1, R2, R8, R11 and R12: 220 Ω → 1 kΩ**.
- Keep R10 unchanged and verify the combined D5 current.
- Retain the 9–12 V VIN supply and the 5 V A4/A5 encoder bus.
- Check the encoder-board pull-ups; add external ones when absent.
- Use `Serial1` for D0/D1 Bluetooth and `Serial` for USB.
- Replace AVR-specific register code and verify PWM/control timing.
- Check shutter/focus operation and the mechanism after the conversion.

## Related software and references

- [PanoDuinoControl firmware](https://github.com/TomNaiser/PanoDuinoControl)
- [PanoramaSoftware: Android client and detailed system documentation](https://github.com/TomNaiser/PanoramaSoftware)
- [PanoramaControl Windows client](https://github.com/TomNaiser/PanoramaControl)
- [Arduino Nano R4 datasheet](https://docs.arduino.cc/resources/datasheets/ABX00142-datasheet.pdf)
- [Arduino Nano R4 full pinout](https://docs.arduino.cc/resources/pinouts/ABX00142-full-pinout.pdf)
- [AS5048A/AS5048B datasheet, manufacturer document via Mouser](https://www.mouser.com/datasheet/2/588/asset_pdf_25493108-3418798.pdf)
- [Vishay SFH618A/SFH6186 datasheet](https://www.vishay.com/docs/83673/sfh618a.pdf)

