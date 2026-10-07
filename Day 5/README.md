##  `if` and `case` Statements

`if` and `case` statements are commonly used inside an `always` block to describe combinational logic.

For example, a multiplexer can be described using either `if` or `case`.

### Using `if`

```verilog
always @(*) begin
    if (sel)
        y = b;
    else
        y = a;
end
```

This describes:

```text
             ┌─────┐
a ──────────►│     │
             │ MUX ├────► y
b ──────────►│     │
             └──┬──┘
                │
               sel
```

The behavior is:

```text
sel = 0  →  y = a
sel = 1  →  y = b
```

This is a 2:1 multiplexer.

Consider:

```verilog
always @(*) begin
    if (cond1)
        y = c1;
    else if (cond2)
        y = c2;
    else if (cond3)
        y = c3;
    else
        y = c4;
end
```

The conditions are evaluated in order:

```text
if cond1
   │
   ├── TRUE  → c1
   │
   └── FALSE
          │
          ▼
       cond2?
          │
          ├── TRUE  → c2
          │
          └── FALSE
                 │
                 ▼
              cond3?
                 │
                 ├── TRUE  → c3
                 │
                 └── FALSE → c4
```

Therefore, if multiple conditions are true at the same time, the first condition has priority. This can result in a priority multiplexer structure.

---

### `case` Statement

A case statement is useful when a signal selects one of several values.

**Example:**

```verilog
always @(*) begin
    case (sel)
        2'b00: y = c1;
        2'b01: y = c2;
        2'b10: y = c3;
        2'b11: y = c4;
    endcase
end
```

This describes a 4:1 multiplexer.

```text
sel = 00 → y = c1
sel = 01 → y = c2
sel = 10 → y = c3
sel = 11 → y = c4
```

---

## Incomplete Assignments and Latch Inference

Consider:

```verilog
always @(*) begin
    if (sel)
        y = b;
end
```

There is no assignment to `y` for `sel = 0`. The synthesizer has to preserve the previous value of `y`. That requires storage. Therefore, a **latch** is inferred.

> To prevent unwanted latches, use the whole if-else condition, or give a default assignment.

Consider:

```verilog
always @(*) begin
    case (sel)
        2'b00: y = c1;
        2'b01: y = c2;
    endcase
end
```

There is no assignment to `y` for `sel = 2'b10` or `sel = 2'b11`. Therefore, the synthesizer may infer a latch. Specify all cases and a default case to avoid this.

Consider:

```verilog
always @(*) begin
    case (sel)

        2'b00: begin
            x = a;
            y = b;
        end

        2'b01: begin
            x = c;
        end

        default: begin
            x = d;
            y = e;
        end

    endcase
end
```

For `sel = 2'b01`: `x` is assigned, but `y` is not. Therefore `y` may infer a latch.

Every output assigned inside a combinational `always` block must receive a value on every possible execution path.

---

## `for` Loops in RTL

A procedural `for` loop can be used inside an `always` block.

**Example:**

```verilog
integer i;

always @(*) begin

    for (i = 0; i < 8; i = i + 1) begin

        if (i == sel)
            y = input[i];

    end

end
```

The loop is used to express repetitive logic.

---

##  `generate for`

A generate for loop is different. It is used to replicate hardware structures during elaboration. Generate loops must be outside the `always` block.

**Example:**

```verilog
genvar i;

generate

    for (i = 0; i < 8; i = i + 1) begin : gen

        and_gate u_and (
            .a(a[i]),
            .b(b[i]),
            .y(y[i])
        );

    end

endgenerate
```

This creates eight instances of `and_gate`.

---

##  Difference Between `for` and `generate for`

| Procedural `for` | `generate for` |
|---|---|
| Usually inside `always` | Outside procedural blocks |
| Used for repetitive procedural operations | Used to replicate hardware/instances |
| Uses an integer variable | Uses `genvar` |
| Executed as part of procedural evaluation | Expanded during elaboration |

---

## Labs

### Bad Case

![screenshot](badcasecode.PNG)
![screenshot](badcase.PNG)
![screenshot](badcaseyosys.PNG)

GLS Output:

![screenshot](badcasegls.PNG)

When `sel = 2'b11`, `y` gets no new assignment, so synthesis infers a latch. The "hold the previous value" behavior is the indication for a latch.

### Complete Case

![screenshot](compcasecode.PNG)
![screenshot](compcase.PNG)
![screenshot](compcaseyosys.PNG)

Every possible `sel` value has an assignment with the help of default statement. Therefore, no latch is inferred.

### Incomplete case

![screenshot](incompcasecode.PNG)
![screenshot](incompcase.PNG)
![screenshot](incompcaseyosys.PNG)

`sel` is 2 bits, so there are 4 possibilities, but only two are stated, so a latch will be inferred.

### Demux Case

![screenshot](demuxcasecode.PNG)
![screenshot](demuxcase.PNG)

There are 8 possible values of `sel`, and all 8 are covered. No latch is inferred.

### Demux generate

![screenshot](demuxgencode.PNG)
![screenshot](demuxgen.PNG)

Demux using `generate`.

### Incomplete if

![screenshot](incompifcode.PNG)
![screenshot](incompif.PNG)
![screenshot](incompifyosys.PNG)

There is no `else`. Because `y` has to remember its old value, synthesis creates a latch.

### Incomplete if2

![screenshot](incompif2code.PNG)
![screenshot](incompif2.PNG)
![screenshot](incompif2yosys.PNG)

This involves an incomplete `if-else if`, so it creates a latch.

### Mux generate

![screenshot](muxgencode.PNG)
![screenshot](muxgen.PNG)

`mux_generate`, a 4-to-1 MUX written using a generate loop, has a complete selection for all 4 values of `sel`, so it works as intended.

### Partial Case

![screenshot](partialcasecode.PNG)
![screenshot](partialcase.PNG)

This is a case statement with multiple outputs, `y` and `x`. `x` is not assigned for `sel = 01`. So `x` has to remember its previous value, and a latch is created. `y`, however, gets an assignment for every case, so no latch for `y`.

### Ripple Carry Adder

![screenshot](rcacode.PNG)
![screenshot](rcs.PNG)

Instead of writing eight instances manually:

```verilog
full_adder fa0 (...);
full_adder fa1 (...);
full_adder fa2 (...);
...
full_adder fa7 (...);
```
we can use `generate for` loops to generate eight full-adder instances.


