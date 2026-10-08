# 24-Hour Digital Railway Clock — FPGA Implementation

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Verilog](https://img.shields.io/badge/HDL-Verilog-blue.svg)](source/Digital_Clock.v)
[![FPGA](https://img.shields.io/badge/FPGA-Cyclone%20IV%20E-green.svg)](quartus-project/)
[![Quartus](https://img.shields.io/badge/Tool-Quartus%20Prime%2021.1-orange.svg)](https://www.intel.com/content/www/us/en/software/programmable/quartus-prime/overview.html)

A fully functional **24-hour digital clock** implemented in Verilog HDL and
synthesized on an Intel Cyclone IV E FPGA (Terasic DE2-115). The design uses
a hardware clock-divider to produce a precise 1 Hz signal from the 50 MHz
on-board oscillator, maintaining hours, minutes, and seconds in 24-hour
(railway-style) format with midnight rollover.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Hardware Used](#hardware-used)
- [Software Used](#software-used)
- [Project Structure](#project-structure)
- [Design Methodology](#design-methodology)
- [Block Diagram](#block-diagram)
- [RTL Design](#rtl-design)
- [Simulation](#simulation)
- [Hardware Demonstration](#hardware-demonstration)
- [How to Run](#how-to-run)
- [Results](#results)
- [Applications](#applications)
- [Future Improvements](#future-improvements)
- [Learning Outcomes](#learning-outcomes)
- [Author](#author)
- [License](#license)

---

## Overview

This project implements a real-time **24-hour digital clock** on an FPGA using
the Verilog hardware description language. The design is divided into two
always blocks:

1. **Clock Divider** — divides the 50 MHz FPGA system clock down to a stable
   1 Hz square-wave using a 26-bit prescaler counter.
2. **Timekeeping Logic** — tracks seconds (0–59), minutes (0–59), and hours
   (0–23), cascading carries upward and rolling over at midnight.

The railway clock concept refers to the use of unambiguous 24-hour (0–23)
notation rather than 12-hour AM/PM format, which is the standard for rail,
aviation, and military applications worldwide.

---

## Features

- **50 MHz → 1 Hz clock division** using a 26-bit hardware prescaler
- **24-hour timekeeping** (HH:MM:SS, 00:00:00 → 23:59:59) with automatic midnight rollover
- **Active-high synchronous reset** — clears all time registers to 00:00:00
- **Fully synthesisable RTL** — no behavioural-only constructs
- **ModelSim testbench** — verifies reset behaviour and time progression
- **Clean, single-module top level** with clearly separated clock-divider and counter logic
- Targets the **Terasic DE2-115** board (EP4CE115F29C7, Cyclone IV E)

---

## Hardware Used

| Component        | Specification                                      |
|------------------|----------------------------------------------------|
| FPGA Board       | Terasic DE2-115 Development and Education Board    |
| FPGA Device      | Intel Cyclone IV E — EP4CE115F29C7                |
| Package          | FBGA-780                                          |
| Logic Elements   | 114,480 LEs (Cyclone IV E)                        |
| On-board Clock   | 50 MHz crystal oscillator                         |
| Speed Grade      | 7                                                 |
| Core Voltage     | 1.2 V                                             |
| I/O Standards    | 3.3 V LVCMOS                                      |

---

## Software Used

| Tool                          | Version / Notes                              |
|-------------------------------|----------------------------------------------|
| Intel Quartus Prime           | 21.1.0 Lite Edition (Build 842, Oct 2021)   |
| ModelSim-Altera               | Included with Quartus Prime 21.1             |
| HDL Language                  | Verilog-2001 (IEEE 1364-2001)               |
| Operating System (dev)        | Windows 10/11                                |

---

## Project Structure

```
24-hour-railway-clock-fpga/
│
├── README.md                          ← This file
├── LICENSE                            ← MIT License
├── .gitignore                         ← Quartus / ModelSim build artifact exclusions
│
├── source/
│   ├── Digital_Clock.v                ← Top-level RTL design (clock divider + timekeeping)
│   └── tb_Digital_Clock.v             ← ModelSim testbench
│
├── simulation/
│   └── run_simulation.do              ← ModelSim TCL script (compile + simulate)
│
├── quartus-project/
│   ├── Digital_Clock.qpf              ← Quartus project file
│   └── Digital_Clock.qsf             ← Quartus settings file (device, EDA tool config)
│
├── hardware/
│   └── pin_assignments.md             ← Recommended DE2-115 pin mapping
│
├── docs/
│   ├── 24_Hour_Railway_Clock.pdf      ← Project report
│   ├── 24_Hour_Railway_Clock.pptx     ← Presentation slides
│   └── Project_Report.docx            ← Full written report (Word format)
│
└── images/
    ├── block-diagram.png              ← System-level block diagram
    └── hardware-demo.jpg              ← Hardware demonstration photo
```

---

## Design Methodology

The design follows a **hierarchical, synchronous RTL** approach, structured
entirely within a single `Digital_Clock` module for clarity and portability.

### Clock Division

The FPGA's 50 MHz oscillator is too fast to tick a clock directly.  A
**26-bit counter** accumulates rising edges of `Clk_50MHz`.  When the count
reaches 25,000,000 (half of 50 M), the 1 Hz internal signal `Clk_1sec` is
toggled and the counter resets.  This yields:

```
f_out = f_in / (2 × 25,000,000) = 50,000,000 / 50,000,000 = 1 Hz
```

### Timekeeping

A separate `always` block is sensitive to the rising edge of `Clk_1sec`.
Each tick:
- **Seconds** increments by 1.
- When seconds reaches 59, it resets to 0 and **minutes** increments.
- When minutes reaches 59, it resets to 0 and **hours** increments.
- When hours reaches 23, it resets to 0 (midnight rollover).

All registers are cleared to zero on assertion of the active-high `reset`.

### Signal Widths

| Register  | Width  | Range   | Reasoning                        |
|-----------|--------|---------|----------------------------------|
| `seconds` | 6 bits | 0 – 59  | 2⁶ = 64 ≥ 60                    |
| `minutes` | 6 bits | 0 – 59  | 2⁶ = 64 ≥ 60                    |
| `hours`   | 5 bits | 0 – 23  | 2⁵ = 32 ≥ 24                    |
| `counter` | 26 bits| 0–25 M  | 2²⁶ = 67,108,864 ≥ 25,000,000  |

---

## Block Diagram

```
                    ┌─────────────────────────────────────┐
                    │           Digital_Clock              │
                    │                                      │
  Clk_50MHz ───────►│  ┌──────────────────────┐           │
                    │  │   Clock Divider       │           │
  reset     ───────►│  │  (26-bit prescaler)   ├──► Clk_1sec (1 Hz)
                    │  └──────────────────────┘           │
                    │               │                      │
                    │               ▼                      │
                    │  ┌──────────────────────┐           │
                    │  │   Timekeeping Logic  │           │
                    │  │  (HH:MM:SS counters) │           │
                    │  └──────────┬───────────┘           │
                    │             │                        │
                    └─────────────┼────────────────────────┘
                                  │
                        ┌─────────┼─────────┐
                        ▼         ▼         ▼
                    hours[4:0] min[5:0] sec[5:0]
                    (0–23)     (0–59)   (0–59)
```

---

## RTL Design

The top-level module exposes three ports:

```verilog
module Digital_Clock (
    input        Clk_50MHz,    // 50 MHz system clock
    input        reset,        // Active-high reset
    output reg [5:0] seconds,
    output reg [5:0] minutes,
    output reg [4:0] hours
);
```

The two-always-block architecture keeps the clock-division logic cleanly
separated from the timekeeping counters, which simplifies timing analysis
and makes each block independently readable and testable.

See [`source/Digital_Clock.v`](source/Digital_Clock.v) for the complete
annotated implementation.

---

## Simulation

The testbench (`tb_Digital_Clock.v`) instantiates the DUT, drives a 50 MHz
clock with a 10 ns toggle, asserts reset for 100 ns, then deasserts it and
runs for 100 ms of simulated time to observe time progression.

### Running in ModelSim

```tcl
# From the ModelSim console, with the simulation/ folder as working directory:
do run_simulation.do
```

The `run_simulation.do` script:
1. Creates and compiles Altera simulation libraries.
2. Compiles `Digital_Clock.v` and `tb_Digital_Clock.v`.
3. Launches simulation with full visibility (`+acc`).
4. Adds all signals to the waveform viewer.
5. Runs to completion.

### Expected Waveform Behaviour

| Time (ns)    | `reset` | `seconds` | `minutes` | `hours` |
|-------------|---------|-----------|-----------|---------|
| 0 – 100      | `1`     | `0`       | `0`       | `0`     |
| 100 +        | `0`     | incrementing each 1 Hz tick → |

---

## Hardware Demonstration

The hardware demo photo is available in [`images/hardware-demo.jpg`](images/hardware-demo.jpg).

The compiled `.sof` bitstream was programmed to the Terasic DE2-115 board
via the USB-Blaster interface using the Quartus Prime Programmer. Output
signals can be observed on the LEDR LEDs or connected to a downstream
BCD-to-seven-segment decoder.

---

## How to Run

### Prerequisites

- Intel Quartus Prime 21.1 Lite Edition (free from [intel.com](https://fpgasoftware.intel.com))
- Terasic DE2-115 board (or any Cyclone IV E target with a 50 MHz clock)
- USB-Blaster cable for programming
- ModelSim-Altera (included with Quartus Prime)

### Step 1 — Open the Quartus Project

```
File → Open Project → quartus-project/Digital_Clock.qpf
```

### Step 2 — Add Pin Assignments

Open **Assignments → Pin Planner** and apply the mappings from
[`hardware/pin_assignments.md`](hardware/pin_assignments.md), or paste the
`set_location_assignment` lines directly into `Digital_Clock.qsf`.

### Step 3 — Compile

```
Processing → Start Compilation   (or Ctrl+L)
```

A successful compile produces `output_files/Digital_Clock.sof`.

### Step 4 — Program the Device

1. Connect DE2-115 via USB-Blaster.
2. Open **Tools → Programmer**.
3. Click **Add File** → select `output_files/Digital_Clock.sof`.
4. Click **Start**.

### Step 5 — Verify Output

- Press `KEY[0]` to apply reset (clears time to 00:00:00).
- Release reset; LED indicators should begin changing every second.
- Count the LED toggles to confirm 1-second timing accuracy.

### Step 6 — Run Simulation (Optional)

```bash
# In ModelSim, navigate to the simulation/ directory, then:
do run_simulation.do
```

---

## Results

- The design compiled successfully targeting EP4CE115F29C7 with **zero
  critical warnings**.
- Functional RTL simulation in ModelSim confirmed correct reset behaviour
  and second-by-second time progression.
- The 26-bit prescaler produces a 1 Hz clock with a theoretical accuracy
  limited only by the 50 MHz crystal tolerance (±50 ppm on DE2-115).
- Synthesis resource utilisation: minimal — the design uses a handful of
  registers and LUTs, well within the Cyclone IV E's 114,480 LEs.

---

## Applications

- Railway and transit station time displays
- Industrial process timers and elapsed-time indicators
- Digital wristwatch and wall-clock prototypes
- Real-time clock (RTC) module replacement in embedded FPGA systems
- Teaching platform for synchronous FSM design and clock-domain management

---

## Future Improvements

- **BCD decoder + 7-segment display driver** — output HH:MM:SS on the
  DE2-115's six 7-segment displays (HEX0–HEX5).
- **Set-time interface** — use push-buttons (KEY[1]/KEY[2]) to increment
  hours and minutes at runtime.
- **Alarm functionality** — comparator logic to drive a buzzer or LED at a
  pre-set time stored in registers.
- **AM/PM toggle** — switchable 12/24-hour display mode via SW[0].
- **UART output** — transmit HH:MM:SS over UART for logging or display on
  a PC terminal.
- **PLL-based clock** — replace the counter divider with an on-chip PLL for
  improved frequency accuracy and jitter reduction.
- **Testbench enhancement** — add self-checking assertions and randomised
  reset sequences for robust verification.

---

## Learning Outcomes

Through this project, the following digital design concepts were applied and
reinforced:

- **RTL design hierarchy** — separating functional blocks into distinct
  always processes for clarity and maintainability.
- **Clock domain management** — deriving a slow clock enable from a fast
  system clock using a prescaler.
- **Synchronous reset** — understanding active-high reset propagation through
  all state-holding elements.
- **Counter-based finite state machines** — cascaded counters with carry
  conditions implementing the timekeeping FSM.
- **FPGA tool flow** — end-to-end: HDL authoring → synthesis → place & route
  → programming in Quartus Prime.
- **Functional simulation** — writing a self-contained testbench and verifying
  design behaviour in ModelSim before hardware deployment.

---

## Author

| Field          | Details                                             |
|---------------|------------------------------------------------------|
| Name           | Thirugnanam V S                                    |
| Student ID     | 24BVD1004                                          |
| Programme      | B.Tech Electronics Engineering (VLSI Design & Technology) |
| Institution    | VIT Chennai                                        |
| Course         | Digital System Design (DSD)                        |
| Batch          | Batch 5                                            |
| Year           | 2024–2025                                          |

---

## License

This project is released under the **MIT License** — see [`LICENSE`](LICENSE)
for the full text.

You are free to use, modify, and distribute this design for personal,
educational, or commercial purposes with attribution.

---

*For questions or suggestions, feel free to open an issue or pull request.*
"# 24-hour-railway-clock-fpga" 
