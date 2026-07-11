# CropDrop Bot – Autonomous Farm-to-Collection Center Delivery Robot

> Autonomous line-following delivery robot built for the **e-Yantra Robotics Competition (CropDrop Theme), IIT Bombay**. Simulates an agricultural logistics pipeline — crop crates are detected by colour, picked up via electromagnet, and delivered to designated collection points — implemented on **STM32F103RB**.

## Overview

CropDrop Bot is an autonomous mobile robot that simulates an agricultural logistics system, transporting crop crates from a farm location to the correct collection center. The robot:

- Follows complex line trajectories (curves, zig-zags, intersections, dotted paths)
- Identifies crop crates using colour sensing
- Picks up crates using an electromagnet
- Delivers crates to colour-specific drop zones
- Returns to the farm for subsequent deliveries

The project was first developed and validated in **CoppeliaSim**, then deployed on physical hardware using an **STM32F103RB** microcontroller.

---

## System Architecture

The robot is built around five core subsystems:

| Subsystem | Function |
|---|---|
| **Navigation System** | 5-channel IR line sensing, PID-based line following, node detection |
| **Object Detection** | IR proximity sensor for crate presence detection |
| **Colour Recognition** | TCS3200 sensor for RGB-based crate classification |
| **Pick-and-Place** | Electromagnet + MOSFET driver for crate handling |
| **Control Unit** | STM32F103RB — PWM motor control, ADC acquisition, UART debug |

---

## Hardware Used

| Component | Purpose |
|---|---|
| STM32F103RB (Nucleo) | Main controller |
| N20 Motors | Robot locomotion |
| L298N Motor Driver | Motor direction & speed control |
| IR Sensor Array (5-channel) | Line detection |
| TCS3200 | Colour detection |
| IR Proximity Sensor | Box/crate detection |
| Electromagnet | Crate pickup mechanism |
| MOSFET Driver | Electromagnet switching |
| LiPo Battery | Power supply |
| RGB LED | Visual status feedback |

## Software Tools

- STM32CubeIDE
- STM32 HAL Libraries
- Embedded C
- CoppeliaSim
- UART Debugging via PuTTY

---

## Navigation & Line Following

### IR Sensor Acquisition (ADC + DMA)

The 5-channel IR array is read using the STM32 ADC peripheral, configured in:

- **Scan Conversion Mode** — sequentially samples all 5 channels
- **Continuous Conversion Mode** — ADC restarts automatically after each sequence
- **DMA Mode + Circular Mode** — transfers converted values to RAM without CPU intervention, wrapping automatically once the buffer is full

**Why DMA circular mode?**

Without DMA, every ADC conversion requires a CPU interrupt and a manual read:

```
ADC Conversion Complete → CPU Interrupt → Read ADC Value
```

With circular DMA, values are streamed directly to a memory buffer, freeing the CPU for control-loop computation:

```
ADC → DMA → RAM Buffer (auto-wraps at end)

IR_Buffer[0] → Sensor 1
IR_Buffer[1] → Sensor 2
IR_Buffer[2] → Sensor 3
IR_Buffer[3] → Sensor 4
IR_Buffer[4] → Sensor 5
```

This enables continuous, real-time sensor acquisition with minimal CPU overhead.

### Line Following Algorithms

Two approaches were implemented and evaluated:

#### 1. PID-Based Line Following

Sensor readings are normalized against calibrated black/white references, and the line position is computed as a weighted average:

```
Position = Σ(Sensor_i × Weight_i) / Σ(Sensor_i)
Weights: [-2, -1, 0, 1, 2]

Error = Setpoint - Position
Correction = Kp·Error + Ki·Integral + Kd·Derivative

LeftSpeed  = BaseSpeed - Correction
RightSpeed = BaseSpeed + Correction
```

This allows smooth tracking through curves, zig-zags, sharp turns, and dotted line sections.

#### 2. Reinforcement Learning–Based Navigation

A separate RL-based approach was implemented for autonomous navigation, where the robot learns suitable motor actions based on sensor states and reward signals — evaluated as an alternative to the deterministic PID controller.

---

## STM32 Timer Peripherals

Timers are used extensively across the project. Each timer relies on:

- **Counter Register (CNT)**
- **Auto-Reload Register (ARR)**
- **Capture/Compare Registers (CCR1–CCR4)**

### PWM Mode — Motor Control

The timer counter free-runs between `0` and `ARR`. PWM frequency and duty cycle are derived as:

```
PWM Frequency = Timer Clock / [(PSC + 1)(ARR + 1)]
Duty Cycle    = (CCR / (ARR + 1)) × 100
```

- Changing **ARR** → changes PWM frequency
- Changing **CCR** → changes duty cycle

PWM output drives the **L298N** motor driver for DC motor speed control.

```
STM32 Timer PWM → L298N Motor Driver → DC Motors
```

The L298N handles direction via H-bridge switching, and speed via PWM duty cycle.

### Input Capture Mode — Frequency Measurement

Used to measure external digital signal timing via a timer channel:

```
External Edge → Timer captures CNT value → Stored in CCR register
```

**Example:**
```
1st rising edge → CNT = 5000 → CCR1 = 5000
2nd rising edge → CNT = 7000 → CCR1 = 7000

Ticks = Capture_2 - Capture_1
Time  = Ticks × TimerTick
```

Applications of input capture include frequency measurement, pulse-width measurement, RPM sensing, and ultrasonic distance measurement — used here for **TCS3200 colour frequency measurement**.

### Interrupt Flow (Input Capture)

```
Input Edge → CNT copied to CCR → CCxIF flag set
           → CCxIE enabled → Timer Interrupt → NVIC → CPU
```

```c
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    value = HAL_TIM_ReadCapturedValue(&htim3, TIM_CHANNEL_1);
}
```

---

## Colour Detection System

### TCS3200 Sensor

The TCS3200 uses an array of photodiodes, each paired with an optical filter, grouped as:

- Red-filtered photodiodes
- Green-filtered photodiodes
- Blue-filtered photodiodes
- Clear (unfiltered) photodiodes

The active filter group is selected via the sensor's **S2 / S3** control pins.

### Light-to-Frequency Conversion

```
Light → Filtered Photodiode → Photocurrent
      → Current-to-Frequency Converter → Square Wave Output

f_OUT ∝ I_photo ∝ Light Intensity
```

### Frequency Measurement via Input Capture

The TCS3200 `OUT` pin is connected to an STM32 timer channel configured in Input Capture mode:

```
TCS OUT:  ____|‾‾‾‾|____|‾‾‾‾|____|‾‾‾‾|____
               ↑        ↑        ↑
            Capture  Capture  Capture
```

At each edge:
1. Timer captures `CNT` into `CCR`
2. Capture interrupt fires
3. HAL callback reads the captured value
4. Frequency is computed:

```
Frequency = 1 / [(CCR_2 - CCR_1) × TimerTick]
```

This measurement is repeated per filter (Red → Green → Blue), and the dominant frequency determines the crate's colour.

---

## Box Detection & Pickup Sequence

```
Detect Box → Stop Robot → Activate Electromagnet
           → Identify Colour → Navigate to Destination
```

The electromagnet is switched via a MOSFET driver controlled by the STM32.

---

## Delivery Logic

| Crate | Action |
|---|---|
| Red | Turn right at 1st city node → drop at 2nd node |
| Blue | Turn left at 2nd city node → drop at 3rd node |
| Green | Turn right at 3rd city node → drop at 4th node |

After delivery: electromagnet deactivates → robot performs a 180° turn → returns to farm for the next crate.

---

## State Management

| Variable | Purpose |
|---|---|
| `city` | Farm/city zone detection |
| `pick` | Box picked status |
| `returns` | Return journey status |
| `box_count` | Number of deliveries completed |
| `white_node` | Farm node count |
| `count` | City node count |
| `colour` | Currently detected box colour |

This state machine enables fully autonomous decision-making without external intervention.

---

## Features

- Autonomous line following (white-line & black-line)
- PID-based navigation
- Real-time colour detection via input capture
- DMA-driven multi-channel IR sensor acquisition
- Automatic crate pickup and delivery
- Return-to-origin navigation
- Simulation (CoppeliaSim) and hardware (STM32) validation

## Results

- Successfully completed all competition tasks in simulation
- Successfully deployed and validated on physical hardware (STM32F103RB)
- Achieved reliable line following using PID control
- Successfully detected and sorted colour-coded crates
- Finished in the **Top 30 Teams** at the e-Yantra Robotics Competition, IIT Bombay

---

## Key Learning Outcomes

- STM32 ADC configuration (scan, continuous, circular DMA)
- Timer PWM generation for motor control
- Timer input capture & frequency measurement
- Interrupt-driven embedded programming & NVIC handling
- Sensor interfacing (IR array, TCS3200, IR proximity)
- PID controller design and tuning
- Reinforcement learning applied to robotic navigation
- Real-time embedded system design on STM32

---

## Team

**Team ID:** 4036

| Name | Role |
|---|---|
| Rishal Mohammed Karuvanthodikayil | Team Leader |
| Saurav Mohan | Member |
| Sanjeev Mohan | Member |
| Richu Mathew Shaji | Member |

---

## License

This project was developed for academic and competition purposes as part of the e-Yantra Robotics Competition, IIT Bombay.
