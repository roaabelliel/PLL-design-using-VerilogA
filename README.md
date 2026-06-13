# PLL Design and Verilog-A Behavioral Modeling
**Course:** Analog/Mixed-Signal Simulation and Modeling | IEEE ASU Student Branch   
**Tool:** Cadence Virtuoso (Spectre simulator)  
**Methodology:** Top-down design using Verilog-A behavioral models + transistor-level refinement

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [PLL Architecture](#pll-architecture)
3. [Repository Structure](#repository-structure)
4. [Block-by-Block Design](#block-by-block-design)
   - [Phase-Frequency Detector (PFD)](#1-phase-frequency-detector-pfd)
   - [Charge Pump (CHP)](#2-charge-pump-chp)
   - [Voltage-Controlled Oscillator (VCO)](#3-voltage-controlled-oscillator-vco)
   - [Frequency Divider](#4-frequency-divider)
5. [Integration & Simulation](#integration--simulation)
   - [All Verilog-A PLL](#all-verilog-a-pll)
   - [Mixed PLL (Transistor-Level Divider)](#mixed-pll-transistor-level-divider)
6. [Key Results](#key-results)
7. [Design Parameter](#design-parameter)
8. [How to Run](#how-to-run)

---

## Project Overview

This project applies a **top-down design methodology** to a Phase-Locked Loop (PLL) — one of the most fundamental mixed-signal building blocks in clock generation, frequency synthesis, and CDR circuits. Each PLL block is first modeled behaviorally in **Verilog-A**, individually verified, then integrated into a full closed-loop system. In the final stage, the Verilog-A frequency divider is replaced with a **transistor-level schematic** to demonstrate mixed abstraction co-simulation using Cadence's Hierarchy Editor and config view.

---

## PLL Architecture

```
              ┌─────────────────────────────────────────┐
              │                                         │
REF ──► [PFD] ──► [Charge Pump] ──► [Loop Filter] ──► [VCO] ──► OUTPUT
              ▲                                         │
              │              [÷N Divider] ◄─────────────┘
              └─────────────────────────────────────────┘
```

| Block | Function |
|-------|----------|
| PFD | Detects phase/frequency error between REF and feedback; outputs UP/DN pulses |
| Charge Pump | Converts UP/DN pulses to a current that charges/discharges the loop filter |
| Loop Filter | Converts charge pump current to a smooth VCO control voltage |
| VCO | Generates output frequency proportional to control voltage; gain = Kvco |
| ÷N Divider | Divides output frequency by N; feeds back to PFD input |

---

## Repository Structure

```
PLL-design-using-VerilogA/
│
├── verilog_a/
│   ├── pfd.vams              # Phase-Frequency Detector
│   ├── charge_pump.vams      # Charge Pump
│   ├── vco.vams              # Voltage-Controlled Oscillator
│   └── divider.vams          # Frequency Divider (÷N)
│
├── schematics/
│   ├── dff_transistor/       # Transistor-level DFF for divider
│   └── divider_transistor/   # Transistor-level ÷N divider
│
├── testbenches/
│   ├── tb_pfd/               # PFD standalone testbench
│   ├── tb_chp/               # Charge pump standalone testbench
│   ├── tb_vco/               # VCO tuning curve testbench
│   ├── tb_divider/           # Divider testbench (Verilog-A & transistor)
│   └── tb_pll_full/          # Full closed-loop PLL testbench
│
├── configs/
│   ├── pll_all_verilog_a/    # Config view: all behavioral blocks
│   └── pll_mixed/            # Config view: behavioral PFD/CHP/VCO + transistor divider
│
├── results/
│   ├── control_voltage/      # Vctrl vs. time plots (lock acquisition)
│   ├── frequency_plots/      # REF vs. output frequency after lock
│   └── log_files/            # Spectre simulation logs with runtime
│
└── README.md
```

---

## Block-by-Block Design

### 1. Phase-Frequency Detector (PFD)

**Function:** Compares the rising edges of the reference clock (REF) and the divided feedback signal (FB). Generates a UP pulse if REF leads FB, or a DN pulse if FB leads REF. Both pulses reset when both are high (after a short delay to avoid dead zone).

**Verilog-A model — key behaviour:**
```
if (REF rises before FB)  → UP pulse width ∝ phase error
if (FB  rises before REF) → DN pulse width ∝ phase error
at lock: UP = DN ≈ short reset pulses (no net charge injected)
```

**Verification:** Testbench applies REF and FB with a known phase offset → observe UP/DN pulse widths proportional to the offset (reproduces behaviour analogous to Fig. 6.13/6.14 of the reference).

---

### 2. Charge Pump (CHP)

**Function:** Steered by the UP/DN signals. When UP is active: sources current +Icp into the loop filter. When DN is active: sinks current −Icp from the loop filter. At lock, the net average current is zero.

**Verilog-A model — key behaviour:**
```
UP = 1 → I_out = +Icp
DN = 1 → I_out = −Icp
UP = DN = 0 → I_out = 0
```

**Verification:** Drive UP/DN with controlled pulse widths → verify output current matches ±Icp (analogous to Fig. 6.18).

---

### 3. Voltage-Controlled Oscillator (VCO)

**Function:** Outputs a sinusoidal (or digital) signal whose frequency is linearly controlled by an input voltage.

$$f_{out} = f_0 + K_{VCO} \cdot V_{ctrl}$$

**Your Kvco:**
```
Kvco = (random_number / 100) × 1e9  Hz/V
       where random_number ∈ [50, 400]
```
> Replace `random_number` in the line above with your actual generated value and update `vco.vams` accordingly.

**Verification:** Sweep Vctrl from 0 V to Vdd → plot fout vs. Vctrl → single tuning curve. Add a marker at Vctrl corresponding to the target output frequency (analogous to Fig. 6.11).

**Analytical lock voltage:**
$$V_{ctrl,lock} = \frac{f_{target} - f_0}{K_{VCO}}$$

Compare this value against the Vctrl cursor reading from the closed-loop simulation (deliverable 12 & 13).

---

### 4. Frequency Divider

**Function:** Divides the VCO output frequency by integer N, producing the feedback signal for the PFD. At lock: fout = N × fref.

#### Stage A — Verilog-A model
Behaviorally counts N VCO cycles, then toggles output → ideal divide-by-N.

**Verification:** Apply a known frequency → verify output period = N × input period (analogous to Fig. 6.20).

#### Stage B — Transistor-level schematic
Built from **cascaded DFFs** (master-slave topology). Each DFF divides by 2; chain of k DFFs → divide by 2^k.

**Transistor-level DFF:** Document the schematic view clearly — show all transistors, clock connections (CK and CKB), D input, and Q/QB outputs.

**Verification:** Use the same testbench as Stage A — the waveform should be identical, confirming functional equivalence between behavioral and physical implementations.

---

## Integration & Simulation

### All Verilog-A PLL

**Setup in Cadence:**
1. Create a top-level schematic instantiating PFD, CHP, loop filter (passive RC), VCO, and divider — all using their Verilog-A cell views
2. Open Hierarchy Editor → set all blocks to `veriloga` view
3. Create a config view pointing to this hierarchy
4. Run transient simulation until Vctrl settles (loop locks)

**Expected outputs:**
- Vctrl vs. time: starts away from lock voltage, converges and settles → add cursor to show steady-state Vctrl
- REF and feedback frequency overlay: both converge to same frequency at lock → add A/B markers to confirm fFB = fREF

**Record the simulation runtime** from the Spectre log file (deliverable 15).

---

### Mixed PLL (Transistor-Level Divider)

**Setup in Cadence:**
1. In Hierarchy Editor, change **only the divider** from `veriloga` → `schematic` view
2. All other blocks remain on their Verilog-A views
3. Re-run the same transient simulation

**Expected outputs:**
- Vctrl vs. time: same locking behaviour as all-Verilog-A case (analogous to Fig. 6.23)
- REF and output frequency: same lock confirmed with A/B markers (analogous to Fig. 6.24)
- **Simulation runtime increases** compared to the all-Verilog-A case — record from log file

**Deliverable 20 — Comparison & Comment:**

| Configuration | Simulation Time |
|---------------|----------------|
| All Verilog-A | (record here) |
| Mixed (transistor divider) | (record here) |

The mixed simulation is slower because Spectre must now solve the full transistor-level SPICE equations for every DFF at every time step, whereas the Verilog-A model is a simple event-driven counter. This illustrates why top-down methodology starts behavioral — fast iteration — then progressively refines individual blocks to transistor level only when needed.

---

## Key Results

Fill in after running simulations:

| Parameter | Value |
|-----------|-------|
| Reference frequency (fref) | |
| Divide ratio (N) | |
| Target output frequency (fout = N × fref) | |
| Kvco | Hz/V |
| Analytically calculated Vctrl at lock | V |
| Simulated Vctrl at lock (cursor) | V |
| Discrepancy | % |
| All Verilog-A simulation time | s |
| Mixed simulation time | s |
| Simulation slowdown factor | × |

---

## Design Parameter

Your personal Kvco must be calculated once and used consistently across all simulations:

```
1. Visit: https://www.calculator.net/random-number-generator.html
            ?slower=50&supper=400&ctype=1&s=1922&submit1=Generate
2. Record the generated number (e.g., 237)
3. Kvco = 237 / 100 * 1e9 = 2.37e9 Hz/V
4. Update this value in vco.vams and in the results table above
```

---

## How to Run

### Individual Block Testbenches

```
1. Open Cadence Virtuoso
2. Navigate to testbenches/ → open desired testbench schematic
3. Launch ADE L → set up transient simulation
4. Run and verify waveform matches expected behaviour (see each block section above)
5. Save screenshots for report deliverables
```

### Full PLL Simulation

```
1. Open tb_pll_full/ testbench schematic
2. Tools → Hierarchy Editor
3. Select config: pll_all_verilog_a (or pll_mixed for Stage B)
4. Set all view mappings as required
5. Launch ADE → transient sim (run long enough for PLL to lock, typically several reference cycles)
6. Plot: Vctrl vs. time, and REF + feedback frequency
7. Check Spectre log for simulation time
```

### Switching to Transistor-Level Divider

```
In Hierarchy Editor:
  divider → change view from "veriloga" to "schematic"
  (all other blocks remain "veriloga")
Save config → re-run simulation → compare Vctrl and runtime
```
