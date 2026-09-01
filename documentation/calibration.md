# Calibration

## 1. Overview

Calibration is performed during flight-controller initialization to establish sensor offsets before the main control loop begins.

The current firmware uses predefined calibration values for:

- Roll gyroscope rate
- Pitch gyroscope rate
- Yaw gyroscope rate
- Accelerometer X
- Accelerometer Y
- Accelerometer Z

An LED connected to GPIO 15 provides visual indication during the startup/calibration sequence.

## 2. Startup Sequence

The controller performs the following high-level initialization sequence:

```text
Power ON
   │
   ▼
ESP32 Startup
   │
   ▼
LED Initialization / Indication
   │
   ▼
Receiver GPIO Initialization
   │
   ▼
Receiver Interrupt Setup
   │
   ▼
I²C Initialization
   │
   ▼
MPU6050 Initialization
   │
   ▼
ESC PWM Initialization
   │
   ▼
Motor Outputs Set to 1000 µs
   │
   ▼
Calibration Values Loaded
   │
   ▼
Main Control Loop
```

## 3. LED Calibration / Status Indication

GPIO 15 is configured as an output and used as a visual status indicator.

During startup, the firmware repeatedly toggles this GPIO with approximately 100 ms delays.

Conceptually:

```text
LED OFF → ON → OFF → ON → ...
```

This provides visual feedback while the controller is being initialized.

A later revision can assign explicit LED patterns to individual states such as:

```text
Startup
Sensor initialization
Calibration
Receiver ready
Armed
Fault
```

The current firmware does not define these as separate named states.

## 4. Gyroscope Calibration Values

The current firmware contains the following gyroscope calibration offsets:

```text
RateCalibrationRoll  = -2.43
RateCalibrationPitch = -0.48
RateCalibrationYaw   = -0.83
```

During operation, these values are subtracted from the raw angular-rate measurements:

```text
RateRoll  -= RateCalibrationRoll
RatePitch -= RateCalibrationPitch
RateYaw   -= RateCalibrationYaw
```

This compensates for the measured gyro bias used by the current setup.

## 5. Accelerometer Calibration Values

The current firmware contains:

```text
AccXCalibration = -0.03
AccYCalibration = -0.04
AccZCalibration =  0.14
```

These values are subtracted from the converted accelerometer readings:

```text
AccX -= AccXCalibration
AccY -= AccYCalibration
AccZ -= AccZCalibration
```

The corrected accelerometer values are then used to calculate roll and pitch angles.

## 6. MPU6050 Initialization

The MPU6050 is connected through I²C and uses address:

```text
0x68
```

The firmware initializes the sensor by waking it from its default sleep state.

The I²C clock is configured to:

```text
400 kHz
```

The accelerometer and gyroscope configuration registers are written during sensor setup and during sensor acquisition.

## 7. Accelerometer Angle Calculation

After calibration, accelerometer measurements are used to estimate roll and pitch.

Roll is calculated approximately as:

```text
AngleRoll =
    atan(AccY / sqrt(AccX² + AccZ²)) × 57.29
```

Pitch is calculated approximately as:

```text
AnglePitch =
    -atan(AccX / sqrt(AccY² + AccZ²)) × 57.29
```

These values provide the accelerometer-based attitude reference for the complementary filter.

## 8. Complementary Filter Initialization

The filtered angles start from:

```text
complementaryAngleRoll  = 0
complementaryAnglePitch = 0
```

During every control cycle, the controller combines gyroscope integration with the accelerometer angle:

```text
Roll:

complementaryAngleRoll =
    0.991 × (previous roll angle + RateRoll × dt)
    + 0.009 × AngleRoll
```

```text
Pitch:

complementaryAnglePitch =
    0.991 × (previous pitch angle + RatePitch × dt)
    + 0.009 × AnglePitch
```

with:

```text
dt = 0.004 s
```

## 9. Angle Limiting

The filtered roll and pitch angles are constrained to ±20°:

```text
-20° ≤ complementaryAngleRoll  ≤ +20°

-20° ≤ complementaryAnglePitch ≤ +20°
```

This limits the attitude values used by the supplied control implementation.

## 10. Receiver Initialization

The six receiver input GPIOs are configured with pull-ups:

```text
GPIO 25 → CH1
GPIO 33 → CH2
GPIO 32 → CH3
GPIO 35 → CH4
GPIO 34 → CH5
GPIO 13 → CH6
```

Interrupts are attached to each channel using `CHANGE`.

The pulse width is measured between the rising and falling edges of each receiver signal.

## 11. ESC Initialization

The firmware allocates ESP32 PWM timers and attaches four servo outputs for the ESCs.

Each ESC output is configured with:

```text
Frequency: 500 Hz
Minimum:   1000 µs
Maximum:   2000 µs
```

During initialization, all four motor outputs are first set to:

```text
1000 µs
```

This establishes the motor-control output at the cutoff level before normal control begins.

## 12. Throttle Cutoff and Controller Reset

The flight controller contains a low-throttle condition:

```text
ReceiverValue[2] < 1030 µs
```

When this condition is true:

```text
Motor 1 = 1000 µs
Motor 2 = 1000 µs
Motor 3 = 1000 µs
Motor 4 = 1000 µs
```

The controller also resets previous PID errors and integral states.

This includes both the angle-controller and rate-controller state variables.

## 13. Calibration Values in the Current Firmware

| Parameter | Current value |
|---|---:|
| Rate calibration roll | -2.43 |
| Rate calibration pitch | -0.48 |
| Rate calibration yaw | -0.83 |
| Accelerometer X calibration | -0.03 |
| Accelerometer Y calibration | -0.04 |
| Accelerometer Z calibration | 0.14 |

These are the values currently hard-coded in the supplied firmware.

## 14. Recommended Physical Calibration Procedure

For a new hardware build, the hard-coded values should not automatically be reused.

A practical calibration procedure is:

```text
1. Place quadcopter on a stable, level surface.
2. Remove all propellers.
3. Power the flight controller.
4. Keep the frame completely stationary.
5. Collect multiple accelerometer and gyroscope samples.
6. Calculate the sensor offsets.
7. Store the resulting calibration values.
8. Restart the controller.
9. Verify that gyro rates are close to zero while stationary.
10. Verify that calculated roll/pitch angles are close to the expected level orientation.
```

The current firmware does not implement an automatic sample-and-average calibration routine; it uses fixed calibration constants.

## 15. BMP280 Calibration / Altitude Reference

The BMP280 is intended to provide barometric altitude information.

A future altitude subsystem can establish a reference pressure at startup:

```text
Power ON
   │
   ▼
BMP280 Pressure Reading
   │
   ▼
Reference / Ground Pressure
   │
   ▼
Relative Altitude
```

However, the supplied firmware does not currently initialize or calibrate the BMP280.

Therefore, BMP280 altitude calibration should be considered a future firmware feature rather than an implemented part of the current calibration sequence.

## 16. Calibration Verification Checklist

Before attempting flight:

- [ ] Quadrotor is level during sensor calibration
- [ ] Propellers are removed during bench testing
- [ ] Gyroscope values remain close to zero while stationary
- [ ] Roll angle behaves correctly when the frame is tilted
- [ ] Pitch angle behaves correctly when the frame is tilted
- [ ] Receiver CH1–CH4 values respond correctly
- [ ] Throttle cutoff works below approximately 1030 µs
- [ ] All four motors reach the 1000 µs cutoff condition
- [ ] Motor numbering is correct
- [ ] Motor rotation directions are correct
- [ ] ESC PWM outputs are verified
- [ ] Buck converter output is verified at 3.3 V
- [ ] Sensor supply voltage and wiring are verified
- [ ] BMP280 hardware wiring is verified if installed

## 17. Calibration Flow Summary

```text
                POWER ON
                   │
                   ▼
             ESP32 Startup
                   │
                   ▼
            LED Indication
                   │
                   ▼
        Receiver Initialization
                   │
                   ▼
          I²C / MPU6050 Setup
                   │
                   ▼
          Load Sensor Offsets
                   │
                   ▼
          ESC Output Setup
                   │
                   ▼
        Motors → 1000 µs
                   │
                   ▼
          Sensor Measurements
                   │
                   ▼
        Apply Calibration Data
                   │
                   ▼
       Attitude Estimation Begins
                   │
                   ▼
          PID Control Enabled
                   │
                   ▼
            Main Control Loop
```

## 18. Important Notes

The calibration constants in this repository describe the specific sensor setup represented by the supplied firmware.

If the MPU6050 is replaced, remounted, or its orientation changes, the calibration values should be re-established.

Likewise, the current BMP280 hardware is documented for altitude/height sensing, but automatic barometric calibration and altitude-hold control are not yet implemented in the supplied flight-controller firmware.
