# OpenFC2350

<div align="center">

![Platform](https://img.shields.io/badge/FMU-RP2350A%20%7C%20Pico%202W-pink?style=for-the-badge&logo=raspberrypi)
![IO](https://img.shields.io/badge/IO-STM32F103C8T6-03a9f4?style=for-the-badge)
![Firmware](https://img.shields.io/badge/Firmware-PX4%20%7C%20NuttX-orange?style=for-the-badge)

**A custom dual-processor flight controller featuring the RP2350A as FMU and STM32F103 as IO processor, this is a Proof Of Concept Design with an ongoing port of PX4/NuttX firmware.**

</div>

---


>**PCB Layout**
>
> <img width="1364" height="1413" alt="Screenshot From 2026-05-30 13-56-12" src="https://github.com/user-attachments/assets/ac8da70a-9b9a-4cf8-91b2-b8f7a1405c0e" />

>**Schematic Layout**
>
> <img width="1908" height="1324" alt="Screenshot From 2026-05-29 20-03-34" src="https://github.com/user-attachments/assets/57d6c7b1-fa61-4d23-908d-dbb3abacf488" />
>
>

<blockquote>
<b>Final Assembled PoC Board</b>
<table width="100%">
  <tr>
    <td width="50%" align="center">
      <img width="100%" alt="PoC Board Front" src="https://github.com/user-attachments/assets/d80ffe5f-32cb-4edf-b722-2fbdabbe5fd8" />
    </td>
    <td width="50%" align="center">
      <img width="100%" alt="PoC Board Side" src="https://github.com/user-attachments/assets/281692c4-92b3-4af1-a89f-f6743c545c35" />
    </td>
  </tr>
</table>


NOTE: The U1 footprint was included on the PCB solely for SMD soldering practice. It is not connected to anything.
</blockquote>

---
## Overview

**OpenFC2350** is an open-source, research-oriented flight controller built around the **Raspberry Pi Pico 2W (RP2350A)** as the Flight Management Unit (FMU) and an **STM32F103C8T6** as the dedicated IO processor, mirroring the architectural pattern of established autopilot platforms like Pixhawk.

This project has two parallel goals:

1. **Hardware PoC** : Validate the full sensor suite, peripheral connections, and inter-processor communication on a breakout-board assembly before committing to a custom PCB. (Done ✔️)

2. **Firmware PoC** : Port a minimal, functional build of PX4/NuttX to the RP2350A, validating the processor's capability as an FMU: high-rate SPI-DMA IMU polling, FPU-accelerated EKF2 on the Cortex-M33, and PX4IO serial protocol to the STM32 IO processor. (Ongoing 🔄)
   

---

## Hardware Architecture

### FMU - Raspberry Pi Pico 2W (RP2350A)

| Peripheral | Interface | GPIO |
|---|---|---|
| MPU6500 (IMU) | SPI1 | GPIO 10–13, GPIO 14 (IMU INT) |
| BMP280 (Barometer) | I2C1 | GPIO 26, GPIO 27 |
| SD Card | SPI0 | GPIO 2–5 |
| HMC5883L (Magnetometer) | I2C0 | GPIO 20, GPIO 21 |
| uBlox M7N (GPS) | PIO UART | GPIO 18 (RX), GPIO 19 (TX) |
| STM32F103 / PX4IO | UART0 | GPIO 0 (TX), GPIO 1 (RX) |
| Telemetry / NSH Shell | PIO UART | GPIO 7 (TX), GPIO 8 (RX) |
| MAVLink | USB CDC | Onboard USB |
| ARM Button | Push Button | GPIO 22 |


### IO Processor - STM32F103C8T6

| Peripheral | Pin |
|---|---|
| ESC PWM Output (×4) | PA0 – PA3 |
| RC PPM Input | PA6 |
| Passive Buzzer (2.7 kHz) | PB6 |
| Pico 2W / PX4IO UART | PA9 (RX), PA10 (TX) |
| RGB LED | Red: PB0 · Green: PA7 · Blue: PB1 |
| Battery Current Sense | PA5 |
| Battery Voltage Sense | PA4 |

---

## Firmware Architecture

### PX4 / NuttX Port

This project is actively porting **PX4 Autopilot** firmware to run on the RP2350A under the **Apache NuttX RTOS**. This is a non-trivial effort given that the RP2350/RP2040 family lacks native upstream PX4 board support for FMU-class flight management as of now.
The port targets the following milestones:

| Milestone | Status |
|---|---|
| NuttX kernel boot on RP2350A | In Progress |
| PIO UART for Telemetry and GPS | In porgress |
| uORB messaging and Sensors/Peripherals working validation | In Progress |
| EKF2 with M33 FPU acceleration |  In Progress |
| PX4IO protocol over UART0 |  In Progress |
| SD logging (ULog / pyulog) |  In Progress |
| Full quadcopter flight test | In Porgress |
| SMP (dual-core) NuttX config |Planned|

### Key Subsystems

####  FMU Core (RP2350A / PX4 + NuttX)

- **IMU Sampling** — MPU6500 over SPI1 with DMA transfers and interrupt-driven sampling on GPIO 14 (INT pin), potentially targeting ≥1 kHz update rate.
- **State Estimation** — EKF2 using the Cortex-M33's hardware FPU (single + double precision) for quaternion integration and sensor fusion.
- **Sensor Suite** — BMP280 barometer (I2C1), HMC5883L magnetometer (I2C0), uBlox M7N/M8N GPS (PIO UART) all feeding the EKF2 estimator pipeline.
- **PX4IO Protocol** — Serial communication over UART0 to the STM32 IO processor for PWM dispatch, RC input relay, Buzzer and LED alerts/status, and battery telemetry.
- **Data Logging** — SD card logging over SPI0 in ULog format, compatible with QGroundControl and pyulog for post-flight analysis.
- **Ground Station** — MAVLink stream over USB CDC (Pico's native USB), NSH shell over PIO UART for live debugging.

####  IO Processor (STM32F103C8T6 / PX4IO)

- Runs a minimal **PX4IO** firmware build, receiving actuator commands from the FMU over UART and translating them to PWM signals.
- Manages **4× ESC PWM outputs** (A0–A3), **RC PPM decoding** (A6), **passive buzzer** for arming tones (B6), and **RGB status LED** (B0/A7/B1).
- **Battery monitoring** — ADC reads on A4 (voltage) and A5 (current), reported back to FMU via PX4IO protocol.

---

## Team

| Name | GitHub 
|---|---|
| Rishi Patil | `@Rispats` 
| Nikhil Karmali | `@karmalinikhil` 
| Kevin Alex Daniel| `@KevinAlex-08` 
| Krishay Naik | `@krish4yy` 

> The initial idea for this project was inspired by [OpenFC2040](https://github.com/vxj9800/openFC2040). 
