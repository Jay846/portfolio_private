---
layout: ../../layouts/Layout.astro
title: "Reverse Engineering Jane Street's 2026 ASIC Puzzle: From GDSII Geometries to Symbolic Z3 Solving"
author: "Jay Salvi"
date: "Sept 14, 2026"
---

# Reverse Engineering Jane Street's 2026 ASIC Puzzle: From GDSII Geometries to Symbolic Z3 Solving

**Author**: Jay Salvi  
**Framework**: Blackbox OS (Directed-Acyclic Verification Graph)  
**Recovered Flag**: `(* TWO STARS *)`  
**DOI Reference**: [10.5281/zenodo.21869451](https://doi.org/10.5281/zenodo.21869451)

---

## Executive Summary

In August 2026, Jane Street released their annual hardware challenge: a raw GDSII layout file (`puzzle.gds`) representing a custom Application-Specific Integrated Circuit (ASIC) built on SkyWater's open 130nm process (`sky130_fd_sc_hd`). The objective was to reverse-engineer the physical silicon geometry, extract the underlying gate-level netlist, model its sequential state transitions, and discover the input bitstream required to unlock the secret output flag.

Using our automated **Blackbox OS** framework—a Directed-Acyclic Verification Graph (DAVG) for semantic layout translation—we successfully reconstructed the 728-cell circuit, verified its cycle-accurate behavior, and formulated a symbolic Z3 SMT constraint model. The solver proved that there is a **single, unique 120-bit input sequence** (representing the 15-byte ASCII string `(* TWO STARS *)`) that satisfies all logical constraints, driving the output bus `O` to transition from printing `TRY AGAIN` to revealing the secret flag: **`(* TWO STARS *)`**.

---

## 1. Physical Layout Parsing & Geometry Extraction (GDSII $\rightarrow$ Netlist)

The challenge provided only a raw GDSII file (`puzzle.gds`, ~1.4 MB containing ~50,000 layer polygons) without gate-level schematics or LEF/DEF netlists. Reconstructing the circuit required a multi-stage physical extraction pipeline:

```
[GDSII Layout: puzzle.gds]
           │
           ▼
[Polygon Extraction & Layer Mapping]
           │
           ▼
[Standard Cell Master Footprinting]
           │
           ▼
[Multi-Layer Via & Contact Overlaps]
           │
           ▼
[Infrastructure Recovery & Diode Bridging]
           │
           ▼
[Synthesized Netlist: 636 Comb Gates + 92 DFFs]
```

### Key Technical Challenges & Insights:

1. **Standard Cell Master Classification**:
   * Analyzed the active (`diff`), poly (`poly`), and local interconnect (`li1`) footprints of cell masters.
   * Matched footprint geometries against the open-source **SkyWater 130nm (`sky130_fd_sc_hd`)** standard cell library to classify 26 unique master cells into standard primitives (`NAND`, `NOR`, `AND`, `OR`, `XOR`, `MUX2`, `DFF`, `INV`, `BUF`).
   * Dropped 676 well taps and 204 decoupling capacitors that carry no logic signals.

2. **Multi-Layer Conductor & Via Overlap Extraction**:
   * Evaluated bounding-box overlaps across 6 conductor layers (`li1`, `met1` through `met5`) and 5 cut/via layers using spatial indexing (Union-Find structure over 33,323 via cuts).
   * Achieved 100% via landing fidelity: 33,323 out of 33,323 cuts landed precisely on metal geometries on both top and bottom layers.

3. **Repurposed Diode Routing Bridges**:
   * Standard DRC fill cells and antenna diodes were embedded throughout the layout.
   * *Critical Discovery*: The layout designers repurposed standard-cell antenna diodes as physical routing bridges to jump across high-density Metal1 routing lanes while preserving signal continuity. 
   * Our framework automatically identified these diode nodes and bridged them, restoring full connectivity across the main clock (`clk`) and reset (`rst_n`) networks.

4. **Netlist Emission**:
   * Produced a clean, structural Verilog netlist (`extracted/geometric_puzzle.v`) comprising **636 combinational logic elements** and **92 D flip-flops (`sky130_fd_sc_hd__dfrtp`)** across 734 nets with zero combinational loops.

---

## 2. Protocol Specification & The 120-Bit vs 121-Bit Nuance

### Interface Clocking Protocol
Tracing the signal transitions reveals the exact interface timing:
* **Cycles 0–2**: `rst_n = 0` (Asynchronous active-low reset).
* **Cycle 3**: `rst_n = 1` (De-assert reset).
* **Cycles 4–124**: `enable = 1` (Serial bitstream shifted in on input pin `I` over 120 clock transitions).
* **Cycle 125**: `enable = 0` (Message evaluation and output generation begins).

### Demystifying 120 Bits vs. 121 Bits ($11 \times 11$ Grid)
A common point of confusion among solvers is the relationship between **120 bits** and **121 bits**:

* **Grid Dimension**: $11 \times 11 = 121$ cells (Star Battle Matrix)
* **Shift Window**: Cycles 4 to 124 = 120 clock transitions = 15 ASCII Bytes × 8 bits = 120 bits

```
15 ASCII Bytes × 8 bits/byte = 120 bits
String: "(* TWO STARS *)"  (Length: 15 characters)
```

* **Why 120 bits?** The ASCII message bus feeds 15 characters of 8 bits each ($15 \times 8 = 120$ bits). The 15-character string **`(* TWO STARS *)`** occupies exactly 120 bits in memory.
* **Why 121 bits inside the silicon?** The internal circuit maps the 120 shifted bits plus 1 implicit initial boundary state bit into an $11 \times 11$ 2D grid array representing a **Star Battle** puzzle matrix.

---

## 3. The Puzzle Inside the Silicon: Star Battle Verification Engine

Tracing back from the `success` net reveals that the set condition is a wide AND-tree decomposing into groups of 2-bit counter comparisons:

1. **Row & Column Counters**:
   * 11 row counters check that every row contains **exactly 2 stars** (`counter == 2`).
   * 11 column counters check that every column contains **exactly 2 stars** (`counter == 2`).
2. **Region Counters**:
   * 11 irregular, orthogonally contiguous 2D regions (partitioned across sizes 14, 21, 7, 5, 28, 8, 11, 9, 6, 8, 4 totaling 121 cells) each check that the region contains **exactly 2 stars**.
3. **No-Touch Constraint**:
   * Adjacency check logic asserts that no two stars touch horizontally, vertically, or diagonally.

The name **`(* TWO STARS *)`** was the hint all along: the chip is a hardware evaluator for an 11×11 2-Star Battle puzzle!

```
. . . . . . . * . * .
* . . . . * . . . . .
. . . . . . . * . * .
* . * . . . . . . . .
. . . . * . * . . . .
. . * . . . . . * . .
. . . . * . . . . . *
. * . . . . * . . . .
. . . * . . . . . . *
. . . . . * . . * . .
. * . * . . . . . . .
```

---

## 4. Hidden Silicon Easter Eggs & Unreachable Dead Strings

By probing the output generator ROM logic and state transitions, we uncovered several hidden easter eggs encoded into the chip by Jane Street engineers:

1. **`TRY AGAIN`**: Standard response for any incorrect input bitstream.
2. **`EMPTY SKY`**: Reachable when all 121 grid bits are set to `0`.
3. **`BIG BANG`**: Reachable when all 121 grid bits are set to `1`.
4. **`(* TWO STARS *)`**: The unique correct solution flag.
5. **`TWO"NOT TOUCH`**: A secret dead string encoded in the output generator ROM. *Z3 SMT proof*: Constraining the output bus to `TWO"NOT TOUCH` over reachable register states returned `UNSAT`—proving it is a dead ROM string only discoverable via static netlist reverse-engineering!
6. **VCD Metadata**: The provided VCD wrong-attempt recording is timestamped `Sat Dec 31 23:59:60 2016` (the official 2016 leap second!) with version header `"Leave no stone unturned!"`.
7. **Warmup Constant**: The warmup design's magic condition $A + B = 496$ uses $496$, the 3rd perfect number.

---

## 5. Symbolic Z3 SMT Constraint Formulation & Solving

With $2^{120} \approx 1.3 \times 10^{36}$ possible input bitstreams, brute force is intractable. We unrolled the 728 logic elements over 125 clock cycles into a Tseitin-encoded Boolean Satisfiability (SAT) formulation:

$$\text{state}_{i, t+1} = \text{If}(\text{reset\_b}_t == 0, \, 0, \, D_{i, t-1})$$

### Solver Execution Result:
* **Solver**: Z3 / CaDiCaL SMT Engine
* **Execution Time**: **0.45 seconds**
* **Satisfiability**: **SAT** (Adding blocking clause returned **UNSAT**, proving uniqueness!)
* **Decoded Output**: **`(* TWO STARS *)`**

---

## 6. Architectural Takeaway for Hardware Design Verification (DV)

This challenge demonstrates the power of **Closed-Loop Code Generation & Symbolic Verification** over manual testbench creation:

* Traditional chip verification relies on verification engineers manually writing SystemVerilog testbenches to hit coverage targets.
* By representing physical silicon layouts as **Directed-Acyclic Verification Graphs (DAVG)**, automated frameworks like **Blackbox OS** can bridge geometry parsing, Verilog synthesis, and SMT constraint solving to verify complex silicon logic in seconds.

---

*Code and netlist extraction tools available on GitHub: [Jay846/asic-puzzle-2026-solution](https://github.com/Jay846/asic-puzzle-2026-solution)*
