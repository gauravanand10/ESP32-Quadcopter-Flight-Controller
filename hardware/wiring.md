# Wiring

This document describes the overall electrical wiring architecture of the ESP32-based quadcopter flight controller.

> **Safety:** Perform all wiring and initial testing with the propellers removed. Verify supply voltages and polarity before connecting the ESP32, sensors, receiver, or ESCs.

## 1. Overall Power Architecture

The LiPo battery is the primary power source.

```text
                    LiPo Battery
                         │
                         ▼
                      XT60 Plug
                         │
                         ▼
              Power Distribution Board
                    │           │
          Main ESC power        │
                    │            │
          ┌─────────┼────────────┼─────────┐
          ▼         ▼            ▼         ▼
        ESC 1     ESC 2        ESC 3     ESC 4
          │         │            │         │
         M1        M2           M3        M4

                         │
                         ▼
                   Buck Converter
                         │
                         ▼
                    Stable 3.3 V
                         │
                         ▼
                       ESP32
```

The PDB distributes the main battery power to the four ESCs and provides the regulated 5 V rail used by the ESC-side electronics according to the implemented power architecture.

The buck converter provides a stable **3.3 V rail for the ESP32**.

## 2. ESP32 ↔ MPU6050

The MPU6050 is the primary inertial sensor for the flight controller.

```text
ESP32                         MPU6050
─────                         ───────
I²C SDA  ───────────────────► SDA
I²C SCL  ───────────────────► SCL
3.3 V     ───────────────────► VCC
GND       ───────────────────► GND
```

The MPU6050 provides:

- 3-axis accelerometer measurements
- 3-axis gyroscope measurements

The firmware uses these measurements for attitude estimation and stabilization.

The current firmware communicates with the MPU6050 at address **0x68** and configures the I²C bus for **400 kHz**.

## 3. ESP32 ↔ BMP280

The BMP280 is used for barometric pressure and altitude/height estimation.

```text
ESP32                         BMP280
─────                         ──────
I²C SDA  ───────────────────► SDA
I²C SCL  ───────────────────► SCL
Supply    ──────────────────► VCC
GND       ───────────────────► GND
```

The BMP280 shares the I²C bus with the MPU6050.

> **Firmware status:** The supplied flight-controller code does not currently initialize or read the BMP280. This wiring documents the hardware architecture; BMP280 altitude processing can be integrated into a later firmware revision.

## 4. ESP32 ↔ FlySky Receiver

The FlySky FS-i6X transmitter communicates wirelessly with its receiver.

The receiver outputs six PWM channels to the ESP32.

```text
FlySky Receiver                 ESP32
────────────────                 ─────
CH1  ─────────────────────────► GPIO 25   Roll
CH2  ─────────────────────────► GPIO 33   Pitch
CH3  ─────────────────────────► GPIO 32   Throttle
CH4  ─────────────────────────► GPIO 35   Yaw
CH5  ─────────────────────────► GPIO 34   Auxiliary
CH6  ─────────────────────────► GPIO 13   Auxiliary
GND  ─────────────────────────► GND
VCC  ─────────────────────────► Receiver supply
```

The firmware measures the PWM pulse widths using GPIO interrupts.

The control inputs are centered around approximately **1500 µs**, with throttle handled separately.

## 5. ESP32 ↔ ESCs

Each ESC receives a motor-control PWM signal from the ESP32.

```text
ESP32 GPIO 27 ───────────────► ESC 1 signal
ESP32 GPIO 14 ───────────────► ESC 2 signal
ESP32 GPIO 12 ───────────────► ESC 3 signal
ESP32 GPIO 26 ───────────────► ESC 4 signal

ESP32 GND ───────────────────► ESC signal ground/reference
```

### ESC mapping

| ESC | ESP32 GPIO | Motor position |
|---|---:|---|
| ESC 1 | GPIO 27 | Front-right |
| ESC 2 | GPIO 14 | Rear-right |
| ESC 3 | GPIO 12 | Rear-left |
| ESC 4 | GPIO 26 | Front-left |

The firmware uses:

- ESC frequency: **500 Hz**
- Minimum pulse: **1000 µs**
- Maximum pulse: **2000 µs**
- Idle throttle: **1170 µs**
- Motor cutoff: **1000 µs**

## 6. ESC ↔ Brushless Motors

Each ESC controls one brushless motor.

```text
ESC 1 ─────────► Motor 1
ESC 2 ─────────► Motor 2
ESC 3 ─────────► Motor 3
ESC 4 ─────────► Motor 4
```

The three motor wires of each brushless motor connect to the corresponding three-phase outputs of its ESC.

Motor rotation direction must match the propeller configuration used by the flight controller.

## 7. Battery ↔ XT60 ↔ PDB

The XT60 connector is the main battery interface.

```text
LiPo Battery
     │
     │ High-current battery connection
     ▼
   XT60
     │
     ▼
    PDB
```

The PDB then distributes power to the ESCs.

**Check battery polarity carefully before powering the system.**

## 8. PDB ↔ ESC Power

```text
PDB
 │
 ├────────► ESC 1 power
 │
 ├────────► ESC 2 power
 │
 ├────────► ESC 3 power
 │
 └────────► ESC 4 power
```

The PDB is responsible for distributing the battery power to the four ESCs and providing the regulated **5 V rail for the ESC-side electronics** according to the implemented design.

## 9. Buck Converter ↔ ESP32

The buck converter is used for the ESP32's regulated supply.

```text
Battery / PDB supply
        │
        ▼
 Buck Converter
        │
        │ 3.3 V regulated
        ▼
      ESP32
```

Before connecting the ESP32:

1. Power the buck converter without the ESP32 connected.
2. Measure its output with a multimeter.
3. Confirm the output is **3.3 V**.
4. Confirm polarity.
5. Connect the regulated output to the ESP32 supply rail.

## 10. LED Status / Calibration Wiring

The firmware uses GPIO 15 for the status LED.

```text
ESP32 GPIO 15 ─────────► LED circuit
ESP32 GND      ─────────► LED return/reference
```

The LED provides visual indication during the startup and calibration sequence.

The current firmware performs a series of GPIO 15 transitions during initialization.

## 11. Common Ground

All low-voltage control electronics should share a common ground reference.

```text
                ┌── MPU6050 GND
                │
ESP32 GND ──────┼── BMP280 GND
                │
                ├── Receiver GND
                │
                └── ESC signal GND
```

The high-current battery/ESC power path should be kept appropriately separated from sensitive sensor wiring while maintaining the required electrical reference between control electronics and ESC signal grounds.

## 12. Complete System Wiring

```text
                         ┌─────────────────────┐
                         │    FlySky FS-i6X    │
                         │     Transmitter     │
                         └──────────┬──────────┘
                                    │ Wireless
                                    ▼
                         ┌─────────────────────┐
                         │   FlySky Receiver   │
                         └──────────┬──────────┘
                                    │
                         CH1–CH6 PWM signals
                                    │
                                    ▼
       ┌─────────────────────────────────────────────────┐
       │                      ESP32                       │
       │                                                 │
       │  Receiver Inputs      GPIO 25,33,32,35,34,13  │
       │                                                 │
       │  MPU6050 ───────────── I²C                     │
       │  BMP280  ───────────── I²C                     │
       │                                                 │
       │  ESC Outputs ───────── GPIO 27,14,12,26       │
       │  LED ───────────────── GPIO 15                │
       └───────────────┬─────────────────────────────────┘
                       │
                 Motor PWM signals
                       │
        ┌──────────────┼───────────────┐
        ▼              ▼               ▼              ▼
      ESC 1          ESC 2           ESC 3          ESC 4
        │              │               │              │
        ▼              ▼               ▼              ▼
      Motor 1        Motor 2         Motor 3        Motor 4

LiPo Battery
     │
     ▼
   XT60
     │
     ▼
    PDB
     │
     ├──────────────► ESC 1
     ├──────────────► ESC 2
     ├──────────────► ESC 3
     ├──────────────► ESC 4
     │
     └──────────────► Buck Converter ───► 3.3 V ───► ESP32
```

## 13. Pre-Power Checklist

- [ ] XT60 polarity verified
- [ ] Battery voltage appropriate for the ESCs and power system
- [ ] Buck converter output verified at 3.3 V
- [ ] 5 V PDB/regulator output verified where applicable
- [ ] ESP32 supply polarity verified
- [ ] MPU6050 wiring verified
- [ ] BMP280 wiring verified
- [ ] Receiver channel mapping verified
- [ ] ESC signal mapping verified
- [ ] Common signal ground verified
- [ ] Motor order verified
- [ ] Motor rotation direction verified
- [ ] Propellers removed for first power-up and bench testing
- [ ] Throttle cutoff behavior verified before installing propellers
