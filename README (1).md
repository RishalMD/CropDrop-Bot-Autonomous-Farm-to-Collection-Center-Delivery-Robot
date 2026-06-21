# CropDrop Bot – Autonomous Farm-to-Collection Center Delivery Robot

## Overview

**CropDrop Bot** is an autonomous mobile robot developed as part of the **e-Yantra Robotics Competition (CropDrop Theme)** organized by **IIT Bombay**.

The robot simulates an automated agricultural logistics system where crop crates are transported from farms to collection centers. It autonomously follows predefined paths, identifies crop crates based on color, picks them up using an electromagnet, and delivers them to the correct destination while maintaining efficient navigation.

The project was first implemented and tested in **CoppeliaSim** and later deployed on physical hardware using an **STM32F103RB** microcontroller.

---

## Team Information

- **Team ID:** 4036

| Name | Role |
|------|------|
| Rishal Mohammed Karuvanthodikayil | Team Leader |
| Saurav Mohan | Member |
| Sanjeev Mohan | Member |
| Richu Mathew Shaji | Member |

---

## Problem Statement

Design an autonomous delivery robot capable of:

- Following complex white-line and black-line trajectories
- Navigating curves, zig-zags, intersections, and dotted paths
- Identifying color-coded crop crates
- Picking up crates from farm locations
- Delivering crates to designated drop zones
- Returning to the farm for subsequent deliveries
- Completing deliveries within specified time constraints

---

## System Architecture

The robot consists of five main subsystems:

### Navigation System
- 5-channel IR Line Sensor Array
- PID-based line following
- Node detection logic
- White-line and black-line navigation

### Object Detection System
- IR Proximity Sensor
- Box presence detection

### Color Recognition System
- TCS3200 Color Sensor
- RGB frequency measurement
- Red, Green, and Blue crate classification

### Pick-and-Place System
- Electromagnet-based gripping mechanism
- MOSFET driver controlled through STM32

### Control Unit
- STM32 Nucleo F103RB
- PWM motor control
- ADC sensor acquisition
- UART debugging

---

## Hardware Components

| Component | Purpose |
|-----------|---------|
| STM32 Nucleo F103RB | Main Controller |
| N20 Motors | Robot Locomotion |
| L298N Motor Driver | Motor Control |
| IR Sensor Array | Line Tracking |
| TCS3200 | Color Detection |
| IR Proximity Sensor | Box Detection |
| Electromagnet | Box Pickup |
| MOSFET Driver | Electromagnet Switching |
| LiPo Battery | Power Supply |
| RGB LED | Visual Feedback |

---

## Software Tools

- STM32CubeIDE
- STM32 HAL Libraries
- Embedded C
- CoppeliaSim
- UART Debugging using PuTTY

---

## Navigation Algorithm

The robot uses a **PID-based line-following algorithm**. Five IR sensors continuously monitor the position of the line. Sensor values are normalized using experimentally obtained black and white calibration values.

**Weighted average for line position:**

$$\text{Position} = \frac{\sum (\text{Sensor}_i \times \text{Weight}_i)}{\sum \text{Sensor}_i}$$

**Weights:** `[-2, -1, 0, 1, 2]`

**Error calculation:**
```
Error = Setpoint - Position
```

**PID correction:**
```
Correction = (Kp × Error) + (Ki × Integral) + (Kd × Derivative)
```

**Motor speed adjustment:**
```
Left Speed  = Base Speed - Correction
Right Speed = Base Speed + Correction
```

This allows smooth tracking through curves, zig-zag paths, sharp turns, and dotted sections.

---

## Color Detection

The **TCS3200** sensor measures RGB frequencies. The dominant frequency determines crate color:

| Color | Condition |
|-------|-----------|
| Red Box | Red > Green AND Red > Blue |
| Green Box | Green > Red AND Green > Blue |
| Blue Box | Blue > Red AND Blue > Green |

Detected color is stored for delivery decisions.

---

## Box Pickup Mechanism

When the proximity sensor detects a crate:

1. Robot stops
2. Electromagnet activates
3. Box is picked
4. Color is identified
5. Robot performs a 180° turn
6. Robot proceeds toward the collection center

---

## Delivery Logic

Each crate has a designated drop location:

| Crate | Action |
|-------|--------|
| Red Box | Turn right at first city node → Drop at second node |
| Blue Box | Turn left at second city node → Drop at third node |
| Green Box | Turn right at third city node → Drop at fourth node |

After delivery:
1. Electromagnet deactivates
2. Robot performs a 180° turn
3. Returns to farm

---

## State Management

| Variable | Purpose |
|----------|---------|
| `city` | Farm/City detection |
| `pick` | Box picked status |
| `returns` | Return journey status |
| `box_count` | Number of deliveries |
| `white_node` | Farm node count |
| `count` | City node count |
| `colour` | Current box color |

This enables autonomous decision making without external intervention.

---

## Features

- Autonomous line following
- PID-based navigation
- White-line and black-line tracking
- Real-time color detection
- Automatic box pickup
- Automatic box delivery
- Return-to-origin navigation
- Embedded implementation on STM32
- Simulation and hardware validation

---

## Results

- Successfully completed all competition tasks in simulation
- Successfully implemented the robot on hardware using STM32F103RB
- Achieved reliable line following using PID control
- Successfully detected and sorted color-coded crates
- **Finished in the Top 30 Teams** in the e-Yantra Robotics Competition
