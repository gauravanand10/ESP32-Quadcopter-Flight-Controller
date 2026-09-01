# PID Control

## 1. Overview

The ESP32 flight controller uses PID-based feedback control to stabilize the quadcopter.

The supplied firmware implements:

- Roll angle PID
- Pitch angle PID
- Roll rate PID
- Pitch rate PID
- Yaw rate PID
- Integral limiting
- PID output limiting
- Previous-error and previous-integral state tracking

The roll and pitch controllers use a cascaded structure:

```text
Desired Roll/Pitch Angle
          │
          ▼
      Angle PID
          │
          ▼
 Desired Roll/Pitch Rate
          │
          ▼
       Rate PID
          │
          ▼
    Motor Correction
```

Yaw is controlled directly using the yaw-rate PID.

## 2. PID Equation

The implementation follows the conventional PID structure:

```text
PID Output = P + I + D
```

where:

```text
P = Kp × Error

I = Previous Iterm
    + Ki × (Current Error + Previous Error) × (dt / 2)

D = Kd × (Current Error - Previous Error) / dt
```

The controller uses:

```text
dt = 0.004 s
```

The integral and final PID output are constrained to prevent excessive values.

## 3. Angle PID

The outer loop controls the desired roll and pitch attitude.

### Roll

The error is:

```text
ErrorAngleRoll =
    DesiredAngleRoll - complementaryAngleRoll
```

The resulting PID output becomes:

```text
DesiredRateRoll = PIDOutputRoll
```

### Pitch

The error is:

```text
ErrorAnglePitch =
    DesiredAnglePitch - complementaryAnglePitch
```

The resulting PID output becomes:

```text
DesiredRatePitch = PIDOutputPitch
```

Therefore, the angle controller does not directly drive the motors. It generates the target angular rates for the inner rate controller.

## 4. Angle PID Gains

The supplied firmware defines:

```text
PAngleRoll  = 2
IAngleRoll  = 0.5
DAngleRoll  = 0.007
```

Pitch uses the same values:

```text
PAnglePitch = 2
IAnglePitch = 0.5
DAnglePitch = 0.007
```

### Interpretation

- **P** responds to the current attitude error.
- **I** accumulates persistent attitude error.
- **D** responds to the rate of change of attitude error.

## 5. Rate PID

The inner rate loop controls the angular velocity measured by the gyroscope.

### Roll Rate

```text
ErrorRateRoll =
    DesiredRateRoll - RateRoll
```

### Pitch Rate

```text
ErrorRatePitch =
    DesiredRatePitch - RatePitch
```

### Yaw Rate

```text
ErrorRateYaw =
    DesiredRateYaw - RateYaw
```

The rate PID outputs are used as motor-mixer corrections:

```text
InputRoll
InputPitch
InputYaw
```

## 6. Rate PID Gains

The supplied firmware defines the roll rate gains as:

```text
PRateRoll = 0.625
IRateRoll = 2.1
DRateRoll = 0.0088
```

Pitch uses the same values:

```text
PRatePitch = 0.625
IRatePitch = 2.1
DRatePitch = 0.0088
```

Yaw uses:

```text
PRateYaw = 4
IRateYaw = 3
DRateYaw = 0
```

## 7. Integral Limiting

The firmware limits each integral term to:

```text
-400 ≤ Iterm ≤ +400
```

Conceptually:

```text
if Iterm > 400:
    Iterm = 400

if Iterm < -400:
    Iterm = -400
```

This prevents the integral state from growing without bound.

The same limit is applied to the PID output:

```text
-400 ≤ PID Output ≤ +400
```

## 8. Why the Controller Uses Two PID Levels

The cascaded architecture separates two control objectives.

### Outer Angle Loop

The angle loop answers:

> How quickly should the quadcopter rotate to reach the commanded attitude?

```text
Desired Angle
      │
      ▼
Angle Error
      │
      ▼
 Angle PID
      │
      ▼
Desired Rate
```

### Inner Rate Loop

The rate loop answers:

> What motor correction is required to achieve that angular rate?

```text
Desired Rate
      │
      ▼
 Rate Error
      │
      ▼
 Rate PID
      │
      ▼
Motor Correction
```

This creates a hierarchical control structure.

## 9. Roll Control Path

```text
FlySky CH1
    │
    ▼
Desired Roll Angle
    │
    ▼
Roll Angle Error
    │
    ▼
Roll Angle PID
    │
    ▼
Desired Roll Rate
    │
    ▼
Roll Rate Error
    │
    ▼
Roll Rate PID
    │
    ▼
Roll Correction
    │
    ▼
Motor Mixer
```

The MPU6050 gyroscope supplies the measured roll rate, while the complementary filter supplies the measured roll angle.

## 10. Pitch Control Path

```text
FlySky CH2
    │
    ▼
Desired Pitch Angle
    │
    ▼
Pitch Angle Error
    │
    ▼
Pitch Angle PID
    │
    ▼
Desired Pitch Rate
    │
    ▼
Pitch Rate Error
    │
    ▼
Pitch Rate PID
    │
    ▼
Pitch Correction
    │
    ▼
Motor Mixer
```

## 11. Yaw Control Path

Yaw does not use the outer angle loop in the supplied implementation.

```text
FlySky CH4
    │
    ▼
Desired Yaw Rate
    │
    ▼
Yaw Rate Error
    │
    ▼
Yaw Rate PID
    │
    ▼
Yaw Correction
    │
    ▼
Motor Mixer
```

The yaw gyroscope measurement provides the feedback signal.

## 12. Motor Mixer

The three PID corrections are combined with throttle.

The supplied equations are:

```text
MotorInput1 =
    InputThrottle - InputRoll - InputPitch - InputYaw

MotorInput2 =
    InputThrottle - InputRoll + InputPitch + InputYaw

MotorInput3 =
    InputThrottle + InputRoll + InputPitch - InputYaw

MotorInput4 =
    InputThrottle + InputRoll - InputPitch + InputYaw
```

The mixer converts the axis corrections into four motor commands.

```text
                 Throttle
                    │
       ┌────────────┼────────────┐
       │            │            │
      Roll        Pitch         Yaw
       │            │            │
       └────────────┼────────────┘
                    ▼
               Motor Mixer
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼         ▼
         M1        M2        M3        M4
```

## 13. Output Constraints

The motor outputs are constrained before being written to the ESCs.

The upper limit used by the supplied firmware is approximately:

```text
1999 µs
```

The idle value is:

```text
1170 µs
```

The motor cutoff value is:

```text
1000 µs
```

When throttle is below approximately:

```text
1030 µs
```

the firmware forces all four motors to the cutoff value.

## 14. PID State Reset

When the throttle is below the cutoff threshold, the controller resets the previous error and integral states.

The reset includes:

```text
PrevErrorRateRoll
PrevErrorRatePitch
PrevErrorRateYaw

PrevItermRateRoll
PrevItermRatePitch
PrevItermRateYaw

PrevErrorAngleRoll
PrevErrorAnglePitch

PrevItermAngleRoll
PrevItermAnglePitch
```

This prevents previously accumulated controller state from being retained while the motors are stopped.

## 15. Current PID Parameter Summary

| Controller | P | I | D |
|---|---:|---:|---:|
| Roll Angle | 2 | 0.5 | 0.007 |
| Pitch Angle | 2 | 0.5 | 0.007 |
| Roll Rate | 0.625 | 2.1 | 0.0088 |
| Pitch Rate | 0.625 | 2.1 | 0.0088 |
| Yaw Rate | 4 | 3 | 0 |

## 16. Control Timing

The controller uses:

```text
dt = 0.004 s
```

corresponding to a nominal:

```text
250 Hz
```

control-loop frequency.

The derivative and integral calculations use this same time interval.

## 17. PID Tuning

The values documented above are the gains present in the supplied firmware. They should not be treated as universal values for every quadcopter.

PID tuning depends on factors such as:

- Frame size
- Motor characteristics
- Propeller size and pitch
- Total vehicle mass
- Battery voltage
- ESC behavior
- Sensor mounting
- Center of gravity
- Control-loop timing

Tuning should be performed incrementally and with appropriate safety precautions.

## 18. Control-System Summary

```text
                   Desired Angle
                        │
                        ▼
                  ┌───────────┐
                  │ Angle PID │
                  └─────┬─────┘
                        │
                  Desired Rate
                        │
                        ▼
                  ┌───────────┐
                  │  Rate PID │◄──── Gyroscope
                  └─────┬─────┘
                        │
                   Axis Output
                        │
                        ▼
                  ┌───────────┐
                  │   Mixer   │◄──── Throttle
                  └─────┬─────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼         ▼
             ESC1      ESC2      ESC3      ESC4
              │         │         │         │
              ▼         ▼         ▼         ▼
             M1        M2        M3        M4
```

The resulting closed-loop system continuously compares commanded motion with measured vehicle motion and applies corrective motor outputs to stabilize the quadcopter.
