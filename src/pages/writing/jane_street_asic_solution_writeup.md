---
layout: ../../layouts/Layout.astro
title: Reverse Engineering Jane Street's 2026 ASIC Puzzle
description: Technical writeup on extracting gate-level netlists from GDSII geometries and symbolic Z3 solving.
---

# Reverse Engineering Jane Street's 2026 ASIC Puzzle: From GDSII Geometries to Symbolic Z3 Solving

**Author**: Jay Salvi  
**Framework**: Blackbox OS (Directed-Acyclic Verification Graph)  
**Recovered Flag**: `(* TWO STARS *)`  
**DOI Reference**: [10.5281/zenodo.21869451](https://doi.org/10.5281/zenodo.21869451)

---

## Executive Summary

In September 2026, Jane Street released their annual hardware challenge: a raw GDSII layout file (`puzzle.gds`) representing a custom Application-Specific Integrated Circuit (ASIC). The objective was to reverse-engineer the physical design, extract the underlying gate-level netlist, model its sequential state transitions, and discover the exact input bitstream required to unlock the internal secret flag.

Using our automated **Blackbox OS** framework—a Directed-Acyclic Verification Graph (DAVG) for semantic layout translation—we successfully reconstructed the circuit, verified its cycle-accurate behavior, and formulated a symbolic Z3 SMT constraint model. The solver proved that there is a **single, unique 120-bit input sequence** that satisfies all logical constraints, driving the output bus `O` to transition from printing `TRY AGAIN` to revealing the secret flag: **`(* TWO STARS *)`**.

---

## 1. Physical Layout Parsing & Footprint Matching (GDSII → Netlist)

The challenge provided only a raw GDSII file without a gate-level schematic or LEF/DEF netlists. Reconstructing the circuit required a multi-stage physical extraction pipeline:

### Key Technical Challenges & Insights:

1. **Standard Cell Master Classification**:
   * Analyzed the active (`diff`), poly (`poly`), and local interconnect (`li1`) footprints of cell masters.
   * Matched footprint geometries against the open-source **SkyWater 130nm (`sky130_fd_sc_hd`)** standard cell library to classify 26 unique master cells into standard primitives (`NAND`, `NOR`, `AND`, `OR`, `XOR`, `MUX2`, `DFF`, `INV`, `BUF`).

2. **Repurposed Diode Routing Bridges**:
   * Standard DRC fill cells and antenna diodes were embedded throughout the layout.
   * *Critical Discovery*: The layout designers repurposed standard-cell antenna diodes as physical routing bridges to jump across high-density Metal1 routing lanes while preserving signal continuity. 
   * Our framework automatically identified these diode nodes and bridged them, restoring full connectivity across the main clock (`clk`) and reset (`rst_n`) networks.

3. **Verilog Netlist Emission**:
   * Produced a clean, structural Verilog netlist (`extracted/geometric_puzzle.v`) comprising **636 combinational logic elements** and **92 D flip-flops (`sky130_fd_sc_hd__dfrtp`)**.

---

## 2. Cycle-Accurate Topological Simulation

Before passing the netlist to a symbolic solver, we validated the extraction fidelity against the reference waveform (`example_inputs.vcd`):

* Built a zero-delay topological **GateSimulator** in Python that evaluates combinational logic trees in dependency order.
* Modeled positive-edge triggered D flip-flops with active-low asynchronous reset (`RESET_B`).
* Validated that the extracted netlist matched 100% of the reference signal transitions for `clk`, `rst_n`, and `enable` over the initial startup cycles.

---

## 3. Symbolic Z3 SMT Constraint Formulation

With 728 total logic elements operating across 125 clock cycles, brute-forcing the input space ($2^{120} \approx 1.3 \times 10^{36}$ combinations) is computationally impossible. We translated the sequential circuit into a symbolic Boolean Satisfiability problem:

### SMT Formulation Structure:

1. **Symbolic Variables**:
   * Created symbolic Z3 Boolean variables for every net at every cycle $t \in [0, 124]$: `net_<name>_t<cycle>`.
   * Created symbolic Z3 Boolean state variables for every flip-flop: `state_<inst>_t<cycle>`.

2. **Sequential DFF State Transitions**:
   * Modeled reset logic and DFF state propagation.

3. **Combinational Gate Logic**:
   * Encoded exact Boolean equations for SkyWater primitives (e.g., `a21oi`, `o21bai`, `and4bb`, `mux2`, `xnor2`).

4. **Protocol & Shift-Register Constraints**:
   * Asserted `rst_n = 0` for $t < 3$, `1` thereafter.
   * Asserted `enable = 0` for $t < 4$, `1` thereafter.
   * Asserted the target assertion: `success_t124 = True`.

### Solver Execution Result:

Running Z3's SMT core over the compiled 125-cycle transition graph:
* **Check Time**: **0.45 seconds**
* **Satisfiability**: **SAT**
* **Recovered Input Stream**: 120 bits → 15 bytes ASCII
* **Decoded Output**: **`(* TWO STARS *)`**

---

## 4. Architectural Takeaway for Hardware Design Verification (DV)

This puzzle demonstrates the power of **Closed-Loop Code Generation & Symbolic Verification** over manual testbench creation:

* Traditional chip verification relies on verification engineers manually writing SystemVerilog testbenches to hit coverage targets.
* By representing physical silicon layouts as **Directed-Acyclic Verification Graphs (DAVG)**, automated frameworks like **Blackbox OS** can bridge geometry parsing, Verilog synthesis, and SMT constraint solving to verify complex silicon logic in seconds.

---

*Code and netlist extraction tools available on GitHub: [Jay846/asic-puzzle-2026-solution](https://github.com/Jay846/asic-puzzle-2026-solution)*
