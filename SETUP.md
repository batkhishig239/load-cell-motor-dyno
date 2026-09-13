# Setup Guide

Step-by-step instructions for wiring the rig, flashing the firmware, and getting live data into LabVIEW.

## Contents

- [Prerequisites](#prerequisites)
- [1. Wire the hardware](#1-wire-the-hardware)
- [2. Flash the firmware](#2-flash-the-firmware)
- [3. Calibrate the load cell](#3-calibrate-the-load-cell)
- [4. Configure LabVIEW](#4-configure-labview)
- [5. Run a test](#5-run-a-test)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- STM32F103-based board (as pictured in the README) with a USB-to-serial or ST-Link connection available
- HX711 load-cell amplifier breakout
- A load cell, mounted so the motor's output shaft presses on its exposed sensing side
- STM32CubeIDE (or your preferred STM32 toolchain) to build and flash `main.c`
- A PC with LabVIEW and NI-VISA installed
- A USB-to-UART adapter (unless your board exposes USART2 directly over USB)

## 1. Wire the hardware

| Signal | STM32F103 pin (per `main.c`) | Connects to |
|---|---|---|
| `CK` (clock, output) | `CK_Pin` / `CK_GPIO_Port` | HX711 `SCK` |
| `DO` (data, input) | `DO_Pin` / `DO_GPIO_Port` | HX711 `DOUT` |
| `LED` | `LED_Pin` / `LED_GPIO_Port` | Onboard status LED (toggles on every calibration sample and every telemetry send) |
| USART2 TX/RX | USART2 pins | USB-to-UART adapter → PC |

The HX711 itself connects to the load cell's four wires (E+/E-, A+/A-) per the HX711 breakout's silkscreen, and takes its own VCC/GND from the STM32 board's 3.3V/5V and GND.

Double-check `CK`/`DO` pin assignments against your actual CubeMX pinout before flashing — the table above reflects the pin *names* used in `main.c`, not fixed physical pin numbers, since those depend on your board's CubeMX configuration.

## 2. Flash the firmware

1. Open the project in STM32CubeIDE.
2. Confirm the clock configuration matches your board (the provided `SystemClock_Config` targets an HSI-sourced 84 MHz PLL — adjust if your board uses an external crystal).
3. Build and flash via ST-Link (or your board's bootloader) as usual.
4. Open a serial terminal at **115200 baud, 8N1** on the USART2 port to confirm you're seeing a stream of integers once the board boots.

## 3. Calibrate the load cell

Calibration happens automatically at boot: `Calibrate_HX711()` takes 10 samples with the LED toggling on each one, averages them, and stores the result as `average_offset`. Every subsequent reading has this offset subtracted, so:

- Make sure the load cell is **unloaded** (nothing pressing on it) when the board powers on or resets — this is when calibration runs.
- If you need to re-zero mid-session, reset the board with no load applied.
- The raw values you'll see are HX711 counts, not physical force units yet — converting counts → Newtons/torque is handled on the LabVIEW side using your load cell's known scale factor.

## 4. Configure LabVIEW

1. Open `Torque_measuring_device.vi`.
2. Install/confirm **NI-VISA** is installed so the VI can see serial ports.
3. Set **VISA resource name** to the COM port your STM32 enumerates as (check Device Manager on Windows or `ls /dev/tty.*` on macOS/Linux).
4. Confirm the serial settings match the firmware: **115200 baud**, and **termination character enabled**, set to `0xA` (`\n`, matching the `\r\n` the firmware appends to each sample).
5. Enter your load cell's scale factor / lever-arm distance wherever the VI expects it, so the raw counts convert correctly into `Torque, Nm`.

## 5. Run a test

1. Mount the motor so its shaft is just touching (not yet loaded onto) the load cell.
2. Run the VI — you should see the `read buffer` and `element` list start filling with live values, and the `Electric_value` plot moving.
3. Start the motor and let it spin up against the load cell. Watch the `Torque, Nm` plot, `Max_torque`, and `mean` update live.
4. Press **STOP** in the VI to end the run once the motor reaches steady state.
5. Separately, measure the motor's no-load RPM with a tachometer/RPM counter — this is not currently captured by the VI, so log it by hand alongside the run's `Max_torque` value.

## Troubleshooting

- **No data in LabVIEW / VI hangs on read:** wrong COM port or baud rate, or the termination character isn't set to `0xA`.
- **Readings look pinned at a constant offset:** the load cell had something pressing on it during the boot-time calibration window — power-cycle with the load cell unloaded.
- **Noisy or jumpy readings:** check the HX711's wiring for loose connections, and confirm `CK`/`DO` timing isn't being disturbed by other GPIO activity sharing the same port.
- **`HX711_cycles_max` reads unexpectedly high:** the 20 ms telemetry interval assumes each HX711 read finishes in ~8 ms; if reads are taking longer, the loop may be falling behind — check for blocking code elsewhere in the loop.
