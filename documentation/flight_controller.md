# Flight Controller

## Overview

The ESP32 acts as the central flight controller of the quadcopter. It continuously reads pilot commands from the FlySky receiver and inertial measurements from the MPU6050, estimates the vehicle attitude, calculates stabilization corrections, and generates control signals for the four ESCs.

The hardware architecture also includes a BMP280 barometric sensor for height/altitude measurement. The current supplied firmware does not yet implement BMP280 data acquisition.

## Flight-Control Flow

```text
FlySky FS-i6X
      │
      ▼
Receiver PWM Inputs
      │
      ▼
     ESP32
      │
      ├──────────────► MPU6050
      │                  │
      │                  ├── Accelerometer
      │                  └── Gyroscope
      │
      ├──────────────► BMP280
      │                  │
      │                  └── Barometric altitude
      │
      ▼
Sensor Processing
      │
      ▼
Complementary Filter
      │
      ▼
Angle Control
      │
      ▼
Rate Control
      │
      ▼
PID Corrections
      │
      ▼
Motor Mixer
      │
      ▼
4 × ESC PWM Outputs
      │
      ▼
4 × Brushless Motors
```

## Receiver Input Processing

The FlySky receiver provides six PWM channels to the ESP32.

| Channel | GPIO | Function |
|---|---:|---|
| CH1 | GPIO 25 | Roll |
| CH2 | GPIO 33 | Pitch |
| CH3 | GPIO 32 | Throttle |
| CH4 | GPIO 35 | Yaw |
| CH5 | GPIO 34 | Auxiliary |
| CH6 | GPIO 13 | Auxiliary |

The firmware uses GPIO interrupts configured on `CHANGE` to measure the duration of each receiver pulse.

The primary control channels are converted into desired roll angle, pitch angle, throttle, and yaw rate commands.

## IMU Processing

The MPU6050 is accessed over I²C at address `0x68`.

The firmware reads:

- Accelerometer X/Y/Z
- Gyroscope X/Y/Z

The gyroscope measurements are converted into angular rates, while the accelerometer measurements are used to calculate roll and pitch angles.

The current firmware uses fixed calibration offsets for the accelerometer and gyroscope.

## Attitude Estimation

The flight controller combines gyroscope integration and accelerometer angle measurements using a complementary filter.

Conceptually:

```text
Gyroscope
   │
   ├── Fast attitude response
   │
   ▼
Gyro-integrated angle ──┐
                       │
                       ▼
                 Complementary
                    Filter
                       ▲
                       │
Accelerometer ─────────┘
   │
   └── Long-term angle reference
```

The implemented filter uses:

```text
0.991 × gyro-based estimate
+
0.009 × accelerometer-based angle
```

The resulting roll and pitch estimates are limited to ±20° in the supplied firmware.

## Cascaded PID Control

The controller uses two levels of PID control for roll and pitch:

```text
Desired Angle
      │
      ▼
 Angle PID
      │
      ▼
Desired Angular Rate
      │
      ▼
 Rate PID
      │
      ▼
Motor Correction
```

The outer angle loop determines the desired angular rate required to reach the commanded attitude.

The inner rate loop then controls the measured angular velocity from the gyroscope.

Yaw is controlled directly through the rate PID loop.

## PID Configuration

The supplied firmware contains separate gains for the angle and rate controllers.

### Roll / Pitch Angle Controller

```text
P = 2
I = 0.5
D = 0.007
```

### Roll / Pitch Rate Controller

```text
P = 0.625
I = 2.1
D = 0.0088
```

### Yaw Rate Controller

```text
P = 4
I = 3
D = 0
```

These values are the gains currently present in the supplied firmware and should be treated as project-specific tuning values.

## Motor Mixing

The four PID corrections are combined with throttle to generate individual motor commands.

```text
Throttle
   │
   ├── Roll correction
   ├── Pitch correction
   └── Yaw correction
          │
          ▼
      Motor Mixer
          │
     ┌────┼────┐
     ▼    ▼    ▼    ▼
    M1   M2   M3   M4
```

The current firmware implements:

```text
Motor 1 = Throttle - Roll - Pitch - Yaw
Motor 2 = Throttle - Roll + Pitch + Yaw
Motor 3 = Throttle + Roll + Pitch - Yaw
Motor 4 = Throttle + Roll - Pitch + Yaw
```

The motor positions are documented as:

- Motor 1: front-right
- Motor 2: rear-right
- Motor 3: rear-left
- Motor 4: front-left

## ESC Control

The ESP32 generates PWM control signals for four ESCs.

```text
GPIO 27 → ESC 1
GPIO 14 → ESC 2
GPIO 12 → ESC 3
GPIO 26 → ESC 4
```

The supplied firmware configures:

```text
ESC frequency : 500 Hz
PWM range     : 1000–2000 µs
Idle value    : 1170 µs
Cutoff value  : 1000 µs
```

Motor commands are limited before being sent to the ESCs.

## Throttle Cutoff

The firmware contains a low-throttle safety condition.

When the throttle receiver value falls below approximately `1030 µs`, the motor outputs are forced to the throttle cutoff value of `1000 µs`.

The PID error and integral states are also reset during this condition.

This prevents accumulated controller state from carrying over while the motors are stopped.

## Control Loop Timing

The control loop uses:

```text
t = 0.004 s
```

which corresponds to a nominal:

```text
1 / 0.004 = 250 Hz
```

control-loop rate.

The loop timing is maintained using `micros()`.

## BMP280 Altitude System

The BMP280 is included in the hardware architecture for barometric height/altitude measurement.

Its intended role is:

```text
BMP280
   │
   ▼
Pressure measurement
   │
   ▼
Altitude estimation
   │
   ▼
Height monitoring
```

The supplied firmware does not currently contain BMP280 initialization, pressure reading, or altitude-hold control logic. Therefore, altitude control should not be claimed as implemented until that firmware functionality is added.

## Startup and Status Indication

GPIO 15 drives the status LED.

During startup, the firmware toggles the LED to provide visual feedback during initialization/calibration.

The controller then initializes the receiver inputs, I²C interface, MPU6050, and ESC outputs before entering the main control loop.

## System Summary

The flight controller can therefore be summarized as:

```text
Pilot Commands
      │
      ▼
FlySky Receiver
      │
      ▼
ESP32 ────────────────┐
 │                    │
 │                 MPU6050
 │                    │
 │             Acc + Gyro Data
 │                    │
 ▼                    ▼
Command Processing → Attitude Estimation
                         │
                         ▼
                    Angle PID
                         │
                         ▼
                     Rate PID
                         │
                         ▼
                    Motor Mixer
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼           ▼
           ESC 1       ESC 2       ESC 3       ESC 4
             │           │           │           │
             ▼           ▼           ▼           ▼
            M1          M2          M3          M4
```

The BMP280 forms the additional barometric sensing path for future height/altitude functionality.
