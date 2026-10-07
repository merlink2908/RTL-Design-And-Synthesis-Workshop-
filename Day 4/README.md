## Gate-Level Simulation (GLS)

GLS checks that the synthesized hardware behaves as intended.

For GLS, the synthesized gate-level Verilog netlist is used as the **Design Under Test (DUT)**.

```text
Gate-Level Netlist + Testbench
              |
              v
          iverilog
              |
              v
            VCD
              |
              v
          GTKWave
```

![GLS screenshot](gls.png)

The same testbench can be reused. Instead of simulating the original RTL, GLS simulates the Verilog representation of the synthesized hardware.

For example, RTL might contain:

```verilog
assign y = (a & b) | c;
```

After synthesis, Yosys may produce a netlist using cells such as:

```text
sky130_fd_sc_hd__and2_1
sky130_fd_sc_hd__or2_1
```

or a library mux cell such as:

```text
sky130_fd_sc_hd__mux2_1
```

The simulator now executes the gate/cell-level implementation.

---

## Why Run GLS?

### 1. Verify Logical Correctness After Synthesis

The synthesized design should remain functionally equivalent to the RTL design.

```text
RTL behavior
     =
Synthesized netlist behavior
```

More precisely, synthesis should preserve the intended functional behavior.

### 2. Verify Implementation-Related Behavior

A gate-level netlist can contain technology-library cells and, when delay information is available, realistic propagation delays.

### 3. Timing Validation

For timing-aware GLS, gate delays can be annotated using an **SDF** (Standard Delay Format) file.

Without delay annotation, GLS primarily checks functional behavior of the synthesized netlist.

---

## Lab 1 — Ternary Operator MUX

![GLS screenshot](tomuxcode.PNG)

This is a 2:1 multiplexer.

GTKWave output:

![GLS screenshot](tomux.PNG)

YOSYS show:

![GLS screenshot](tomuxyosys.PNG)

GLS Output:

![GLS screenshot](tomuxgls.PNG)



---

## Lab 2 — Bad MUX: Missing Sensitivity List

![GLS screenshot](badmuxcode.PNG)

GTKWave Output:

![GLS screenshot](badmux.PNG)

GLS Output:

![GLS screenshot](badmuxgls.PNG)

GLS mismatch is caused by an incomplete sensitivity list.

The problem is:

```verilog
always @(sel)
```

The block is triggered only when `sel` changes. `i0` and `i1` also need to trigger reevaluation — changes in `i0` and `i1` will not be sensed.

### Why Synthesis Can Still Produce a MUX

Synthesis tools do not simply copy the simulator's event-trigger behavior. They analyze the logic described by the procedural statements and infer:

```verilog
if (sel)
    y = i1;
else
    y = i0;
```

which corresponds to:

```text
       i0 ----\
               MUX ----> y
       i1 ----/
       sel ---^
```

Therefore, the synthesized hardware is still a mux.

This creates the situation:

```text
             RTL simulation
i0 change --------X--------> y may not update

             Synthesized hardware
i0 change ------------------> y updates
```

The RTL simulation and synthesized implementation can therefore disagree.

Using:

```verilog
always @(*)
```

automatically builds the sensitivity list from signals used in the block.

---

## Lab 3 — Caveats with Blocking Assignments

**Blocking assignment (`=`)**

The statement updates the left-hand side immediately within the current procedural execution. The line being executed blocks the execution of the code below it — hence the name.

**Non-blocking assignment (`<=`)**

The right-hand side is evaluated when the block executes, but the left-hand-side update is scheduled for a later simulation update event. The order of the lines of code does not matter, as all get evaluated at the same time.

It is recommended to use non-blocking assignments for sequential logic.

---

The procedural order is:

```verilog
d = x & c;
x = a | b;
```

`d` is calculated before `x` is updated. Therefore, during a simulation event, `d` can temporarily use the old value of `x`.

![GLS screenshot](bccode.PNG)

GTKWave Output:

![GLS screenshot](bc.PNG)

YOSYS Output:

![GLS screenshot](bcyosys.PNG)

GLS Output:

![GLS screenshot](bcgls.PNG)

The GLS command is:

```bash
iverilog ../path/to/primitives.v ../path/to/sky130_fd_sc_hd.v filename_net.v tb_filename.v
```
