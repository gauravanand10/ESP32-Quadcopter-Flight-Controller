# Hardware Components

This document lists the major hardware used in the ESP32 Quadcopter Flight Controller.

## 1. ESP32

The ESP32 acts as the main flight-controller microcontroller. It processes receiver commands, reads the IMU and barometric sensor, performs attitude estimation and PID calculations, and generates control signals for the four ESCs.

## 2. MPU6050 IMU

The MPU6050 provides:

- 3-axis accelerometer data
- 3-axis gyroscope data

The accelerometer is used for attitude/angle estimation, while the gyroscope provides angular-rate measurements for the stabilization control loop.

The MPU6050 communicates with the ESP32 through the I²C interface.

## 3. BMP280 Barometric Sensor

The BMP280 is used for barometric pressure measurement and altitude estimation.

It provides altitude-related information that can be used for height monitoring and can be extended later for altitude-hold functionality.

The BMP280 communicates with the ESP32 through I²C.

## 4. Brushless Motors

Four brushless motors provide the thrust required for flight.

The motors are arranged as a quadcopter configuration, with two motors rotating clockwise and two rotating counter-clockwise to balance reaction torque.

## 5. Electronic Speed Controllers (ESCs)

Four ESCs independently control the four brushless motors.

The ESP32 sends the required motor-control PWM signals to the ESCs.

The firmware uses a 500 Hz ESC control frequency and a 1000–2000 µs control range.

## 6. Power Distribution Board (PDB)

The power distribution board distributes battery power to the four ESCs.

It also provides a regulated 5 V supply for the ESC-side electronics according to the implemented power architecture.

## 7. XT60 Power Connector

An XT60 connector is used as the main battery power connection.

It provides a secure high-current connection between the LiPo battery and the quadcopter power distribution system.

## 8. Buck Converter

A buck converter is used to step down the battery voltage and provide a stable 3.3 V supply for the ESP32.

The regulated supply helps maintain reliable operation of the flight-controller electronics.

## 9. FlySky FS-i6X Transmitter and Receiver

The FlySky FS-i6X radio system provides pilot control commands.

The receiver provides six PWM channels to the ESP32:

| Channel | ESP32 GPIO | Function |
|---|---:|---|
| CH1 | GPIO 25 | Roll |
| CH2 | GPIO 33 | Pitch |
| CH3 | GPIO 32 | Throttle |
| CH4 | GPIO 35 | Yaw |
| CH5 | GPIO 34 | Auxiliary |
| CH6 | GPIO 13 | Auxiliary |

The receiver pulse widths are measured using GPIO interrupts in the firmware.

## 10. LED Indicator

An onboard LED indicator is used to provide visual feedback during the controller startup and calibration sequence.

The LED indicates the initialization/calibration state before normal flight-controller operation.

## 11. LiPo Battery

The LiPo battery is the primary power source for the quadcopter.

Battery power is routed through the XT60 connector and power distribution system before being supplied to the ESCs and regulated electronics.

---

## Hardware Summary

| Component | Purpose |
|---|---|
| ESP32 | Main flight controller |
| MPU6050 | Accelerometer + gyroscope |
| BMP280 | Barometric altitude/height sensing |
| 4 × BLDC Motors | Thrust generation |
| 4 × ESCs | Motor speed control |
| Power Distribution Board | Battery power distribution |
| XT60 Connector | Main battery connection |
| Buck Converter | Stable 3.3 V supply for ESP32 |
| FlySky FS-i6X + Receiver | Wireless pilot control |
| LED | Calibration/startup indication |
| LiPo Battery | Main power source |
