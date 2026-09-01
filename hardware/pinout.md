# Pinout

This document defines the ESP32 pin assignments used by the current quadcopter firmware.

## Motor / ESC Outputs

| Motor / ESC | ESP32 GPIO | Function |
|---|---:|---|
| Motor 1 / ESC 1 | GPIO 27 | Front-right motor |
| Motor 2 / ESC 2 | GPIO 14 | Rear-right motor |
| Motor 3 / ESC 3 | GPIO 12 | Rear-left motor |
| Motor 4 / ESC 4 | GPIO 26 | Front-left motor |

The firmware configures the ESC outputs for a **500 Hz** control frequency with a **1000–2000 µs** pulse range.

## FlySky Receiver Inputs

The receiver uses six PWM channels.

| Receiver Channel | ESP32 GPIO | Control / Purpose |
|---|---:|---|
| CH1 | GPIO 25 | Roll |
| CH2 | GPIO 33 | Pitch |
| CH3 | GPIO 32 | Throttle |
| CH4 | GPIO 35 | Yaw |
| CH5 | GPIO 34 | Auxiliary |
| CH6 | GPIO 13 | Auxiliary |

The firmware measures the receiver pulse widths using GPIO interrupts configured for `CHANGE`.

Typical receiver values are interpreted around a **1500 µs center point**.

## MPU6050

The MPU6050 is connected to the ESP32 through I²C.

| Signal | ESP32 |
|---|---|
| SDA | I²C SDA |
| SCL | I²C SCL |
| VCC | 3.3 V supply |
| GND | GND |

The firmware communicates with the MPU6050 at I²C address **0x68** and sets the I²C clock to **400 kHz**.

The MPU6050 provides:

- Accelerometer X/Y/Z
- Gyroscope X/Y/Z

## BMP280

The BMP280 is also connected through the I²C bus for barometric pressure and altitude/height estimation.

| Signal | ESP32 |
|---|---|
| SDA | Same I²C SDA bus |
| SCL | Same I²C SCL bus |
| VCC | Regulated supply according to module |
| GND | GND |

> The current firmware supplied for this repository does not yet contain BMP280 initialization or measurement code. The hardware is therefore documented here as part of the planned/current hardware architecture, while BMP280 firmware integration should be treated as a separate implementation step.

## LED Indicator

| Function | ESP32 GPIO |
|---|---:|
| Calibration / status LED | GPIO 15 |

GPIO 15 is configured as an output and is toggled during the startup/calibration sequence to provide visual feedback.

## Power Connections

### Battery → XT60 → Power Distribution

```text
LiPo Battery
     │
     ▼
  XT60 Plug
     │
     ▼
Power Distribution Board
     │
     ├── ESC 1
     ├── ESC 2
     ├── ESC 3
     └── ESC 4
```

The PDB distributes the main battery power to the ESCs and provides the regulated 5 V rail used by the ESC-side electronics according to the implemented power architecture.

### Battery → Buck Converter → ESP32

```text
LiPo Battery
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

The buck converter provides the regulated **3.3 V supply for the ESP32**.

## Important Wiring Notes

- Connect all grounds to a common ground reference.
- Do not connect raw LiPo battery voltage directly to the ESP32 3.3 V rail.
- Verify the actual output voltage of the buck converter before connecting the ESP32.
- Keep high-current motor/ESC power wiring physically separated from sensitive sensor wiring where practical.
- Confirm motor rotation direction and propeller orientation before flight.
- Remove propellers during initial electrical, receiver, sensor, and motor-control testing.

## Complete ESP32 Pin Summary

| GPIO | Connected Device | Direction | Purpose |
|---:|---|---|---|
| 12 | ESC 3 | Output | Motor 3 PWM |
| 13 | Receiver CH6 | Input | Auxiliary channel |
| 14 | ESC 2 | Output | Motor 2 PWM |
| 15 | LED | Output | Status/calibration indication |
| 25 | Receiver CH1 | Input | Roll |
| 26 | ESC 4 | Output | Motor 4 PWM |
| 27 | ESC 1 | Output | Motor 1 PWM |
| 32 | Receiver CH3 | Input | Throttle |
| 33 | Receiver CH2 | Input | Pitch |
| 34 | Receiver CH5 | Input | Auxiliary channel |
| 35 | Receiver CH4 | Input | Yaw |

I²C pins for the MPU6050/BMP280 should follow the ESP32 board's configured/default I²C SDA and SCL pins unless explicitly assigned elsewhere in the firmware.
