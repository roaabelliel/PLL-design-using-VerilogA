
# PLL Design and Verilog-A Behavioral Modeling
**Course:** Analog/Mixed-Signal Simulation and Modeling | IEEE ASU Student Branch 
**Tool:** Cadence Virtuoso (Spectre simulator) 
**Methodology:** Top-down design using Verilog-A behavioral models + transistor-level refinement 

---

## Table of Contents

1. [Project Overview](https://www.google.com/search?q=%23project-overview)
2. [PLL Architecture](https://www.google.com/search?q=%23pll-architecture)
3. [Repository Structure](https://www.google.com/search?q=%23repository-structure)
4. [Block-by-Block Design](https://www.google.com/search?q=%23block-by-block-design)
* [Phase-Frequency Detector (PFD)](https://www.google.com/search?q=%231-phase-frequency-detector-pfd)
* [Charge Pump (CHP)](https://www.google.com/search?q=%232-charge-pump-chp)
* [Voltage-Controlled Oscillator (VCO)](https://www.google.com/search?q=%233-voltage-controlled-oscillator-vco)
* [Frequency Divider](https://www.google.com/search?q=%234-frequency-divider)


5. [Integration & Simulation](https://www.google.com/search?q=%23integration--simulation)
* [All Verilog-A PLL](https://www.google.com/search?q=%23all-verilog-a-pll)
* [Mixed PLL (Transistor-Level Divider)](https://www.google.com/search?q=%23mixed-pll-transistor-level-divider)


6. [Key Results](https://www.google.com/search?q=%23key-results)
7. [Design Parameters](https://www.google.com/search?q=%23design-parameters)
8. [How to Run](https://www.google.com/search?q=%23how-to-run)

---

## Project Overview

This project applies a **top-down design methodology** to a Phase-Locked Loop (PLL) — one of the most fundamental mixed-signal building blocks in clock generation, frequency synthesis, and high-speed data transceiver circuits. Each PLL block is first modeled behaviorally in **Verilog-A**, individually verified, then integrated into a full closed-loop system. In the final stage, the Verilog-A frequency divider is replaced with a **transistor-level schematic** to demonstrate mixed-abstraction co-simulation using Cadence's Hierarchy Editor and config view.

---

## PLL Architecture

```
              ┌─────────────────────────────────────────┐
              │                                         │
REF ──► [PFD] ──► [Charge Pump] ──► [Loop Filter] ──► [VCO] ──► OUTPUT (960 MHz)
        ▲                                               │
        │                                               │
        │              [÷8 Divider] ◄───────────────────┘
        └───────────────────────────────────────────────┘

```

| Block | Function |
| --- | --- |
| **PFD** | Detects phase/frequency error between REF ($120\text{ MHz}$) and feedback ($120\text{ MHz}$); outputs UP/DN pulses.

 |
| **Charge Pump** | Converts UP/DN pulses to a current ($I_{CHP} = 10\ \mu\text{A}$) that charges/discharges the loop filter.

 |
| **Loop Filter** | Passive $2^{\text{nd}}$ order low-pass filter ($R_z = 10.9\text{ k}\Omega$, $C_z = 20.6\text{ pF}$, $C_p = 0.662\text{ pF}$) converting pump current to a smooth VCO control voltage ($V_c$).

 |
| **VCO** | Generates an output frequency proportional to control voltage ($K_{VCO\_mean} = 1.9\text{ GHz/V}$).

 |
| **÷N Divider** | Divides high-speed output clock frequency by a fixed ratio of $N=8$ to feed back to the PFD input.

 |

---

## Repository Structure

```
PLL-design-using-VerilogA/
│
├── verilog_a/
│   ├── pfd.vams              # Phase-Frequency Detector model
│   ├── charge_pump.vams      # Charge Pump model
│   ├── vco.vams              # Voltage-Controlled Oscillator model
│   └── divider.vams          # Frequency Divider (÷N) behavioral model
│
├── schematics/
[cite_start]│   ├── dff_transistor/       # Transistor-level Transmission-Gate DFF [cite: 83]
[cite_start]│   └── divider_transistor/   # Transistor-level cascaded ÷8 divider [cite: 85]
│
├── testbenches/
[cite_start]│   ├── tb_pfd/               # PFD standalone testbench [cite: 70]
[cite_start]│   ├── tb_chp/               # Charge pump standalone testbench [cite: 73]
[cite_start]│   ├── tb_vco/               # VCO tuning curve testbench [cite: 75]
[cite_start]│   ├── tb_divider/           # Divider testbench (Verilog-A & transistor) [cite: 80, 85]
[cite_start]│   └── tb_pll_full/          # Full closed-loop PLL testbench [cite: 87]
│
└── README.md

```

---

## Block-by-Block Design

### 1. Phase-Frequency Detector (PFD)

**Function:** Compares the rising edges of the reference clock (REF) and the divided feedback signal (FB). Generates an UP pulse if REF leads FB, or a DN pulse if FB leads REF. Both pulses reset when both are high to mitigate the dead-zone effect.

**Verilog-A model — key behavior:**

* 
`REF` rises before `FB` $\rightarrow$ `UP` pulse width proportional to phase error.


* 
`FB` rises before `REF` $\rightarrow$ `DN` pulse width proportional to phase error.


* At lock: `UP` and `DN` show equal, ultra-short reset pulses with zero net charge injected into the pump.



**Verification:** Testbench applies inputs with controlled phase offsets . stand-alone simulations show clean, non-overlapping transient pulse widths corresponding directly to leading phase trends.

---

### 2. Charge Pump (CHP)

**Function:** Controlled by `UP`/`DN` digital signals. When `UP` is high, it sources current into the loop filter; when `DN` is high, it sinks current out of the filter. The current amplitude is fixed at $I_{CHP} = 10\ \mu\text{A}$.

**Verilog-A model — key behavior:**

```
UP = 1, DN = 0  →  I_out = +10 uA
UP = 0, DN = 1  →  I_out = -10 uA
UP = 0, DN = 0  →  I_out = 0 (High Impedance State)

```

**Verification:** Standalone transient sweeps trace a linear input-output tracking characteristic, verifying current matching tolerances against the control pulses.

---

### 3. Voltage-Controlled Oscillator (VCO)

**Function:** Outputs a periodic clock whose frequency is linearly controlled by the incoming input loop voltage ($V_{ctrl}$).

$$f_{out} = f_0 + K_{VCO} \cdot V_{ctrl}$$

**Your Kvco:** * **$K_{VCO}$ Parameter:** Set to **$600\text{ MHz/V} \rightarrow 6\text{ GHz/V}$** ($K_{VCO\_mean} = 1.9\text{ GHz/V}$).

* 
**Free-Running Frequency ($f_0$):** Set to $500.14\text{ MHz}$ at $V_{ctrl} = 0.2\text{ V}$.



**Verification:** Standalone $V_{ctrl}$ sweeps track an output tuning path mapping $500.14\text{ MHz}$ at $0.2\text{ V}$, passing through $1.0001\text{ GHz}$ at $0.6\text{ V}$, up to $1.5002\text{ GHz}$ at $1.0\text{ V}$.

**Analytical Lock Voltage Calculation:**
For a target output frequency of $f_{target} = 960\text{ MHz}$:


$$V_{ctrl,lock} = \frac{960\text{ MHz} - 500.14\text{ MHz}}{600\text{ MHz/V}} + 0.2\text{ V} \approx 0.966\text{ V}$$

---

### 4. Frequency Divider

**Function:** Divides the high-frequency VCO output clock ($960\text{ MHz}$) down to match the reference clock frequency ($120\text{ MHz}$) at the PFD interface.

#### Stage A — Verilog-A model

Counts $N=8$ incoming VCO edges before toggling output, providing an ideal, noise-free, division behavior.

**Verification:** Standalone testing confirms that for a $4\text{ ns}$ cycle input window, a perfectly stable $16\text{ ns}$ period waveform output is generated ($N=4$ configuration verified in standalone validation stages).

#### Stage B — Transistor-level schematic

Built out of **cascaded D-Flip-Flops (DFF)** implemented using $0.13\ \mu\text{m}$ CMOS transmission-gate master-slave architecture. The sequential flip-flops are chained together to implement the full division scale.

* 
**Transistor-level DFF:** Employs complementary NMOS/PMOS transmission pass-gates to isolate the master stage latch from the slave phase based on `clk` and `clkb`. It includes asynchronous reset routing to secure initial state alignment.



**Verification:** Mixed-level simulation waveforms trace exact physical functional matching, capturing timing-accurate propagation delays and real rise/fall transition tails.

---

## Integration & Simulation

### All Verilog-A PLL

Integrated with all block cell-views set to `veriloga` inside the Cadence Hierarchy Editor. The loop tracking rapidly drives $V_{ctrl}$ to settle firmly at the expected lock location.

* 
**Reference Frequency ($f_{ref}$):** $120\text{ MHz}$ 


* 
**Output Lock Frequency ($f_{out}$):** $960\text{ MHz}$ 


* **Simulation Performance:** Executes very fast due to idealized event-driven behavioral expressions.

---

### Mixed PLL (Transistor-Level Divider)

Using the Hierarchy Editor, the divider cell-view is changed from `veriloga` to `schematic` while the PFD, CHP, and VCO remain on behavioral blocks.

The mixed-signal loop correctly locks to the same output parameters, but captures high-frequency non-idealities and real circuit loading effects.

**Abstraction Runtime Comparison Matrix:**

| Configuration | Simulation Execution Performance Type |
| --- | --- |
| **All Verilog-A** | **Fast / Behavioral Mode** |
| **Mixed (Transistor Divider)** | **Heavy Transistor-Level Computation Scale** |

The mixed simulation requires extra computing time because Spectre solves full SPICE multi-node matrix models for the physical transmission-gate DFFs at every time step, showing the value of starting with a top-down behavioral approach.

---

## Key Results

| Parameter | Value |
| --- | --- |
| Reference Frequency ($f_{ref}$) | <br>$120\text{ MHz}$ 

 |
| Divide Ratio ($N$) | <br>$8$ (Fixed) 

 |
| Target Output Frequency ($f_{out}$) | <br>$960\text{ MHz}$ 

 |
| VCO Gain ($K_{VCO\_mean}$) | <br>$1.9\text{ GHz/V}$ 

 |
| Charge Pump Current ($I_{CHP}$) | <br>$10\ \mu\text{A}$ 

 |
| Loop Filter Parameters | <br>$R_z = 10.9\text{ k}\Omega,\ C_z = 20.6\text{ pF},\ C_p = 0.662\text{ pF}$ 

 |
| Loop Damping Factor ($\zeta$) | <br>$1.2$ (Stable/Overdamped Link) 

 |
| Target Loop Phase Margin ($\phi_m$) | <br>$70^{\circ}$ 

 |

---

## Design Parameters

The loop parameters were selected to achieve high loop stability with a $70^{\circ}$ Phase Margin and a damping factor of $\zeta = 1.2$ using a passive lead-lag loop filter configuration:

* 
**$R_z$ (Loop Filter Zero Resistor):** $10.9\text{ k}\Omega$ 


* 
**$C_z$ (Loop Filter Zero Capacitor):** $20.6\text{ pF}$ 


* 
**$C_p$ (Loop Filter Pole Capacitor):** $0.662\text{ pF}$ 



---

## How to Run

### Individual Block Testbenches

1. Launch Cadence Virtuoso.
2. Open the desired block testbench view (e.g., `ieee_PFD_va_tb`, `ieee_CHP_va_tb`, `ieee_VCO_va_tb`, or `ieee_Divider_va_tb`).


3. Open **ADE L** or **ADE XL**, choose the `spectre` simulator, and set up a transient analysis run.


4. Execute simulation and verify the behavior matches the report plots.



### Full Closed-Loop PLL Simulation

1. Navigate to the top-level full system verification library `ieee_pll_schematic`.


2. Open the cell using the **Hierarchy Editor (config view)**.


3. Select your desired abstraction mode by mapping the divider to either `veriloga` or `schematic`.


4. Open **ADE XL** to load loop variables, set transient options to allow tracking lock acquisition, and recompile.


5. Plot $V_{ctrl}$ versus time to view loop lock convergence.
