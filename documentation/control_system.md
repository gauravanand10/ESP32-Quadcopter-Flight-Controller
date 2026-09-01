# Control System

## 1. Control Architecture

The quadcopter uses a closed-loop attitude stabilization system. Pilot commands are received from the FlySky receiver, while the MPU6050 provides feedback about the vehicle's motion.

```text
                 ┌─────────────────────┐
                 │   FlySky FS-i6X     │
                 │     Transmitter     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │      Receiver       │
                 └──────────┬──────────┘
                            │
                       PWM Channels
                            │
                            ▼
                 ┌─────────────────────┐
                 │        ESP32        │
                 │   Flight Controller │
                 └──────────┬──────────┘
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
        Desired State                 MPU6050
             │                       Acc + Gyro
             │                             │
             └──────────────┬──────────────┘
                            ▼
                   Attitude Estimation
                            │
                            ▼
                     Angle Controller
                            │
                            ▼
                      Rate Controller
                            │
                            ▼
                       Motor Mixer
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼              ▼
           ESC 1          ESC 2          ESC 3          ESC 4
             │              │              │              │
             ▼              ▼              ▼              ▼
           Motor 1        Motor 2        Motor 3        Motor 4
```

## 2. Command Path

The pilot controls the quadcopter using the FlySky FS-i6X transmitter.

The receiver sends six PWM channels to the ESP32:

| Channel | GPIO | Control |
|---|---:|---|
| CH1 | GPIO 25 | Roll |
| CH2 | GPIO 33 | Pitch |
| CH3 | GPIO 32 | Throttle |
| CH4 | GPIO 35 | Yaw |
| CH5 | GPIO 34 | Auxiliary |
| CH6 | GPIO 13 | Auxiliary |

The firmware measures each PWM pulse using GPIO interrupts.

For roll and pitch, the receiver command is translated into a desired angle:

```text
DesiredAngleRoll  = 0.1 × (CH1 - 1500)
DesiredAnglePitch = 0.1 × (CH2 - 1500)
```

The yaw command is converted into a desired angular rate:

```text
DesiredRateYaw = 0.15 × (CH4 - 1500)
```

Throttle is taken directly from the throttle receiver channel and limited before motor output.

## 3. Sensor Feedback Path

The MPU6050 provides the feedback required by the stabilization system.

```text
             MPU6050
                │
       ┌────────┴────────┐
       ▼                 ▼
Accelerometer         Gyroscope
       │                 │
       ▼                 ▼
 Roll/Pitch Angle     Angular Rate
       │                 │
       └────────┬────────┘
                ▼
        Attitude Estimation
```

The accelerometer provides a long-term reference for roll and pitch, while the gyroscope provides fast angular-rate information.

## 4. Complementary Filter

The firmware combines the gyroscope and accelerometer information using a complementary filter.

The basic implementation is:

```text
Filtered Angle =
    0.991 × (Previous Angle + Gyro Rate × dt)
    + 0.009 × Accelerometer Angle
```

where:

```text
dt = 0.004 s
```

This gives the gyroscope dominant influence over short-term motion while using the accelerometer to correct long-term drift.

The filtered roll and pitch angles are constrained to:

```text
-20° ≤ angle ≤ +20°
```

in the supplied firmware.

## 5. Cascaded Control Structure

Roll and pitch use a cascaded control structure.

```text
Pilot Angle Command
        │
        ▼
 ┌───────────────┐
 │   Angle PID   │
 └───────┬───────┘
         │
         ▼
Desired Angular Rate
         │
         ▼
 ┌───────────────┐
 │    Rate PID   │
 └───────┬───────┘
         │
         ▼
   Axis Correction
         │
         ▼
    Motor Mixer
```

This separates the attitude objective from the fast angular-rate stabilization loop.

## 6. Angle Control

The outer loop compares the desired roll/pitch angle with the filtered measured angle.

```text
Angle Error = Desired Angle - Measured Angle
```

The angle PID then generates a desired angular rate.

For roll:

```text
ErrorAngleRoll =
    DesiredAngleRoll - complementaryAngleRoll
```

For pitch:

```text
ErrorAnglePitch =
    DesiredAnglePitch - complementaryAnglePitch
```

The integral and output terms are limited to prevent excessive controller values.

## 7. Rate Control

The inner loop compares the desired angular rate against the gyroscope measurement.

```text
Rate Error = Desired Rate - Measured Rate
```

For the three axes:

```text
ErrorRateRoll  = DesiredRateRoll  - RateRoll
ErrorRatePitch = DesiredRatePitch - RatePitch
ErrorRateYaw   = DesiredRateYaw   - RateYaw
```

The rate PID outputs become the roll, pitch, and yaw corrections used by the motor mixer.

## 8. Yaw Control

Unlike roll and pitch, yaw is controlled directly through the rate loop.

```text
FlySky Yaw Command
        │
        ▼
Desired Yaw Rate
        │
        ▼
Yaw Rate Error
        │
        ▼
Yaw PID
        │
        ▼
Yaw Motor Correction
```

This correction is added/subtracted from the appropriate motors to generate the required yaw torque.

## 9. Motor Mixing

The controller combines throttle with roll, pitch, and yaw corrections.

The current firmware uses:

```text
Motor 1 = Throttle - Roll - Pitch - Yaw

Motor 2 = Throttle - Roll + Pitch + Yaw

Motor 3 = Throttle + Roll + Pitch - Yaw

Motor 4 = Throttle + Roll - Pitch + Yaw
```

This converts the three-axis control corrections into four individual motor commands.

## 10. Motor Configuration

The firmware identifies the motors as:

```text
             FRONT
       M1              M4
       ↺                ↻

       M2              M3
       ↻                ↺
             REAR
```

The code comments identify:

```text
Motor 1 → Front-right → Counter-clockwise
Motor 2 → Rear-right  → Clockwise
Motor 3 → Rear-left   → Counter-clockwise
Motor 4 → Front-left  → Clockwise
```

The actual motor/propeller orientation must be verified on the physical quadcopter before flight.

## 11. Output Limiting

Motor commands are constrained before being sent to the ESCs.

The supplied firmware limits the upper motor command to approximately:

```text
1999 µs
```

and applies an idle value of:

```text
1170 µs
```

when the calculated command falls below the idle threshold.

When the throttle channel is below approximately:

```text
1030 µs
```

all motor outputs are forced to:

```text
1000 µs
```

and the relevant PID state variables are reset.

## 12. Control-Loop Timing

The controller defines:

```text
t = 0.004 s
```

Therefore the nominal control-loop frequency is:

```text
1 / 0.004 = 250 Hz
```

The loop uses `micros()` to maintain the timing interval.

The 250 Hz loop repeatedly performs:

```text
1. Read MPU6050
2. Apply sensor calibration
3. Calculate accelerometer angles
4. Update complementary filter
5. Read receiver commands
6. Calculate desired angles/rates
7. Execute angle PID
8. Execute rate PID
9. Calculate motor mixer outputs
10. Apply limits and cutoff
11. Update ESC PWM outputs
12. Maintain loop timing
```

## 13. Barometric Height Path

The BMP280 is intended to provide an additional altitude/height measurement path.

```text
BMP280
   │
   ▼
Pressure Measurement
   │
   ▼
Altitude Estimation
   │
   ▼
Height Monitoring
```

The current supplied firmware does not yet implement this path in the control loop. Therefore, the present controller should be described as an attitude/rate stabilization controller rather than an implemented altitude-hold controller.

## 14. Closed-Loop Behavior

The complete feedback system operates continuously:

```text
        Pilot Input
             │
             ▼
      Desired Attitude
             │
             ▼
       PID Controller
             │
             ▼
       Motor Commands
             │
             ▼
        Quadrotor
             │
             ▼
       Vehicle Motion
             │
             ▼
          MPU6050
             │
             ▼
    Measured Attitude/Rate
             │
             └──────────────► PID Controller
```

Any difference between the desired and measured state produces a corrective control output.

## 15. Summary

The control system consists of:

- FlySky FS-i6X pilot command input
- ESP32 real-time control processing
- MPU6050 inertial feedback
- Complementary roll/pitch attitude estimation
- Cascaded angle and rate PID for roll and pitch
- Rate PID for yaw
- Four-motor mixer
- 500 Hz ESC PWM output
- Throttle limiting and cutoff
- 250 Hz nominal control loop
- BMP280 hardware path for future altitude functionality
