## Logic Optimization in RTL Synthesis

The main goal of synthesis optimization is to implement the same required functionality using less area, less power, and/or better timing.


---

## Optimization Overview

The optimization flow can be broadly divided into:

```text
                 Logic Optimization
                         |
             +-----------+-----------+
             |                       |
             v                       v
     Combinational              Sequential
      Optimization              Optimization
             |                       |
     +-------+-------+       +-------+---------+-------+
     |               |       |       |         |       |
     v               v       v       v         v       v
Constant         Boolean   Constant  State    Retiming Cloning
Propagation      Logic     Propagation Optimization
                 Optimization
                    |
              K-Map / Quine-
                McCluskey
```

---

## Combinational Logic Optimization

Combinational optimization works on logic whose outputs depend only on the current inputs.

**Examples:**

- `AND`
- `OR`
- `NOT`
- `MUX`
- `NAND`
- `NOR`
- `XOR`
- `XNOR`

**Main techniques:**



## Constant Propagation

Constant propagation is one of the simplest and most useful optimizations.

If the synthesis tool knows that a signal is always 0 or 1, it can replace that signal with the constant and simplify the surrounding logic.

For example:

```verilog
assign y = a & 1'b0;
```

Since:

```text
a & 0 = 0
```

The complete circuit becomes:

```verilog
assign y = 1'b0;
```

The AND gate is no longer required. This can save both area and power.

---

## Boolean Logic Optimization

For example:

```verilog
assign y = a ? (b ? c : (c ? a : 0)) : (!c);
```

The `?:` operator is the ternary conditional operator. It can be understood as a multiplexer:

```text
s ? x : z

if s = 1 -> x
if s = 0 -> z
```

So the complete function is:

```text
a = 0  -> y = !c
a = 1  -> y = c
```

This is the behavior of:

```text
y = a XNOR c
```

Therefore, a complicated nested MUX structure can be reduced to a simple XNOR function.

---

## Sequential Logic Optimization

Sequential logic contains state, usually represented by flip-flops or latches.

Unlike combinational logic:

```text
Output = f(current inputs)
```

sequential logic depends on previous state:

```text
Next State = f(current inputs, current state)
```

**Typical sequential elements include:**

- D flip-flops
- JK flip-flops
- T flip-flops
- Latches
- State machines

Sequential optimization can be divided into:

**Basic**

- Sequential constant propagation

**Advanced**

- State optimization
- Retiming
- Sequential logic cloning / physical-aware synthesis

---

## Sequential Constant Propagation

Sequential constant propagation applies the constant-propagation idea to circuits containing registers.

---

## State Optimization

State optimization is mainly associated with finite-state machines (FSMs).

An FSM can contain states that are:

- Unreachable
- Redundant
- Equivalent
- Never used under legal inputs

The tool can remove or merge unnecessary states.

---

## Retiming

Retiming changes the positions of registers in a sequential circuit while preserving the intended sequential behavior.

---

## Logic Cloning

Suppose one logic block drives many registers:

```text
              +--> FF A
              |
Logic Block --+--> FF B
              |
              +--> FF C
```

The block has high fanout. High fanout can cause:

- Larger load capacitance
- More delay
- More routing congestion
- Higher power

Instead, the synthesis tool can duplicate the logic:

```text
          +--> Logic copy 1 --> FF A
Original -|
          +--> Logic copy 2 --> FF B
          |
          +--> Logic copy 3 --> FF C
```

This is called **logic cloning** or **logic duplication**.

---

## Labs

### Counter 

![Multiple module screenshot](counteroptyosys.PNG)

The first counter code looks at only the 0th bit of `q`. To utilise all bits, use:

```verilog
assign q = (count == 3'b100);
```

So:

```text
when count = 4  -> q = 1
otherwise       -> q = 0
```
![Multiple module screenshot](countermodifiedcode.PNG)
![Multiple module screenshot](countermodifiedy.PNG)

### dff_const1

`D` is always one, but reset makes `q = 0`, so the flip-flop is still required to preserve the asynchronous reset behavior.

![Multiple module screenshot](dffconst1code.PNG)
![Multiple module screenshot](dff_const1.PNG)
![Multiple module screenshot](dffconst1yosys.PNG)

### dff_const2

Change the reset branch so that both branches assign 1. Both cases produce:

```text
q = 1
```

There is no meaningful state anymore. Yosys can therefore reduce the entire sequential circuit to:

```text
1'b1 ---> q
```
![Multiple module screenshot](dffconst2code.PNG)
![Multiple module screenshot](dff_const2.PNG)
![Multiple module screenshot](dffconst2yosys.PNG)

### dff_const3

Because non-blocking assignments use the old value of `q1` on the clock edge, the circuit has genuine sequential behavior. So, two flip-flops are inferred.

![Multiple module screenshot](dffconst3code.PNG)
![Multiple module screenshot](dffconst3.PNG)
![Multiple module screenshot](dffconst3yosys.PNG)

### dff_const4

Set both `q` and `q1` to 1 during reset, and also keep them at 1 afterward. There is no state-dependent behavior left. Yosys can eliminate both registers.

![Multiple module screenshot](dffconst4code.PNG)
![Multiple module screenshot](dffconst4.PNG)
![Multiple module screenshot](dffconst4yosys.PNG)

### dff_const5

One-cycle delay is present, so flops cannot be removed.

![Multiple module screenshot](dffconst5code.PNG)
![Multiple module screenshot](dffconst5.PNG)
![Multiple module screenshot](dffconst5yosys.PNG)

---

## Multiple Modules

### Original Design

The first design contains two submodules:

```verilog
module sub_module1(input a, input b, output y);
    assign y = a & b;
endmodule

module sub_module2(input a, input b, output y);
    assign y = a ^ b;
endmodule
```

The top module is:

```verilog
module multiple_module_opt(
    input a, input b, input c, input d,
    output y
);

wire n1, n2, n3;

sub_module1 U1 (.a(a), .b(1'b1), .y(n1));
sub_module1 U2 (.a(a), .b(1'b0), .y(n2));
sub_module2 U3 (.a(b), .b(d), .y(n3));

assign y = c | (b & n1);

endmodule
```

The first submodule is:

```text
n1 = a & 1'b1;
```

Using the Boolean identity:

```text
A AND 1 = A
```

therefore:

```text
n1 = a
```

The second instance is:

```text
n2 = a & 1'b0;
```

Using:

```text
A AND 0 = 0
```

we get:

```text
n2 = 0
```

But `n2` is never used anywhere else. Therefore, the entire U2 logic can be removed.

The third module calculates:

```text
n3 = b XOR d
```

but `n3` is also never used in the final output. Therefore, this logic can also be removed.

The output becomes:

```text
y = c | (b & a)
```

or:

```text
y = c | (a & b)
```

---

![Multiple module screenshot](mmopt.PNG)
![Multiple module screenshot](mmoptyosys.PNG)

---

### multiple_module_opt2

```verilog
module sub_module(input a, input b, output y);
    assign y = a & b;
endmodule

module multiple_module_opt2(
    input a, input b, input c, input d,
    output y
);

wire n1, n2, n3;

sub_module U1 (.a(a),   .b(1'b0), .y(n1));
sub_module U2 (.a(a),   .b(b),    .y(n2));
sub_module U3 (.a(n2),  .b(d),    .y(n3));
sub_module U4 (.a(n3),  .b(n1),   .y(y));

endmodule
```

**First module:**

```text
U1 (.a(a), .b(1'b0), .y(n1));
```

Therefore:

```text
n1 = a & 0
so: n1 = 0
```

**Second module:**

```text
U2 (.a(a), .b(b), .y(n2));
```

Therefore:

```text
n2 = a & b
```

**Third module:**

```text
U3 (.a(n2), .b(d), .y(n3));
```

Therefore:

```text
n3 = n2 & d
```

Substituting `n2`:

```text
n3 = (a & b) & d
or:  n3 = a & b & d
```

**Fourth module:**

```text
U4 (.a(n3), .b(n1), .y(y));
```

Therefore:

```text
y = n3 & n1
```

But:

```text
n1 = 0
```

Therefore:

```text
y = n3 & 0
and: y = 0
```

**Final result:**

The entire circuit can theoretically be reduced to:

```verilog
assign y = 1'b0;
```

The values of `a`, `b`, `c`, `d` do not matter.

---
![Multiple module screenshot](mmopt2.PNG)
![Multiple module screenshot](mmopt2yosys.PNG)

### opt_check

```verilog
module opt_check(input a, input b, output y);
    assign y = a ? b : 0;
endmodule
```

The `?:` operator is a ternary conditional operator, evaluated as `condition ? value_if_true : value_if_false`.

Therefore:

```text
y = a ? b : 0
```

means:

```text
if a = 1: y = b
if a = 0: y = 0
```

This is exactly equivalent to:

```text
y = a AND b
```

Therefore, Yosys simplifies the conditional expression into an AND gate.

---

![Multiple module screenshot](optcheckcode.PNG)
![Multiple module screenshot](opt_checkshow.PNG)


### opt_check2

The RTL is:

```verilog
module opt_check2(input a, input b, output y);
    assign y = a ? 1 : b;
endmodule
```

This means:

```text
if a = 1: y = 1
if a = 0: y = b
```

This is exactly the OR function:

```text
y = a | b
```

Therefore, Yosys transforms:

```text
a ? 1'b1 : b
```

into:

```text
a | b
```

---
![Multiple module screenshot](optcheck2code.PNG)
![Multiple module screenshot](optcheck2.PNG)


### opt_check4

The RTL is:

```verilog
module opt_check4(input a, input b, input c, output y);
    assign y = a ? (b ? (a & c) : c) : (!c);
endmodule
```

Upon simplification, this is exactly the behavior of:

```text
a XNOR c
```

Therefore:

```text
y = a XNOR c
```

or mathematically:

```text
y = (a & c) | (!a & !c)
```

The original RTL uses `b`, but after Boolean simplification, `b` has no effect on the output.

![Multiple module screenshot](optcheck4xnor.PNG)
![Multiple module screenshot](optcheck4.PNG)
