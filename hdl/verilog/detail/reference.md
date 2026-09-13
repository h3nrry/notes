# Verilog / SystemVerilog Quick Reference

## How to use this file

This is a syntax cheat sheet for quick lookup while coding — module skeletons, data types, operators, and common constructs in one place. It intentionally stays short: for the *why* behind a rule, follow the link out to the matching deep-dive doc in this folder ([function.md](function.md), [state_machine.md](state_machine.md), [always_comb_ff.md](always_comb_ff.md), [linter.md](linter.md)). When this reference and a deep-dive doc disagree, the deep-dive doc is right — update this file to match.

## Module skeleton (SystemVerilog)

```systemverilog
module my_module #(
  parameter int WIDTH = 8
) (
  input  logic             clk,
  input  logic             rst_n,
  input  logic [WIDTH-1:0] data_in,
  output logic [WIDTH-1:0] data_out
);

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) data_out <= '0;
    else        data_out <= data_in;
  end

endmodule
```

## Data types

| Type | Kind | Notes |
|---|---|---|
| `wire` | 4-state, net | continuous assignment target (`assign`), or module output |
| `reg` | 4-state, variable | Verilog's procedural-assignment type (name is historical — doesn't imply a real register) |
| `logic` | 4-state, variable | SystemVerilog: replaces `reg`/`wire` for most signals; can't be driven by more than one source |
| `bit` | 2-state | SystemVerilog; `0`/`1` only, no `x`/`z` — good for testbench state, not for RTL that must model unknowns |
| `int` / `integer` | 2-state / 4-state | `int` is SV, 2-state, 32-bit; `integer` is Verilog, 4-state, 32-bit |
| `byte`, `shortint`, `longint` | 2-state | SystemVerilog fixed-width integer types (8/16/64-bit) |
| `enum` | user-defined | SystemVerilog; see [state_machine.md](state_machine.md) |
| `struct` / `union` | user-defined | SystemVerilog; group related signals |

### Declaring loop / integer variables

```systemverilog
integer i;   // Verilog, 4-state, 32-bit — usable in any procedural block (always/initial/task/function)
int    j;    // SystemVerilog, 2-state, 32-bit — same role, tighter (X/Z-free) semantics
genvar k;    // elaboration-only index — legal ONLY inside a generate block, no storage/hardware
```

## Operators (most-used)

| Category | Operators |
|---|---|
| Arithmetic | `+  -  *  /  %` |
| Relational | `==  !=  <  <=  >  >=` |
| Case equality (4-state aware) | `===  !==` (exact match incl. `x`/`z`), `==?  !=?` (SV wildcard, `x`/`z` as don't-care) |
| Logical | `&&  \|\|  !` |
| Bitwise | `&  \|  ^  ^~ / ~^  ~` |
| Reduction (unary) | `&a  \|a  ^a` — AND/OR/XOR of all bits of `a` |
| Shift | `<<  >>` (logical), `<<<  >>>` (arithmetic) |
| Concatenation / replication | `{a, b}`, `{4{a}}` |
| Conditional | `sel ? a : b` |

## Common constructs

```systemverilog
// combinational (see always_comb_ff.md)
always_comb begin
  y = a & b;
end

// sequential (see always_comb_ff.md)
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) q <= '0;
  else        q <= d;
end

// case (use unique/priority in SV — see state_machine.md)
unique case (sel)
  2'b00:   y = a;
  2'b01:   y = b;
  default: y = '0;
endcase

priority case (state)     // SV: tool checks branches cover all reachable values, taken in order
  IDLE: next = RUN;
  RUN:  next = DONE;
  default: next = IDLE;
endcase

casez (opcode)             // 'z'/'?' bits in the case items are don't-care (also: casex, x/z/? all don't-care)
  4'b1???: y = alu_out;
  4'b01??: y = mem_out;
  default: y = '0;
endcase
```

### For loop — procedural vs. generate

Two `for` loops that look alike but do different things: a **procedural** `for` runs in simulation
inside `always`/`initial`/a task or function, and needs a real `integer`/`int` variable; a **generate**
`for` runs once at elaboration time and needs a `genvar`, which isn't a variable at all — it has no
storage and disappears once the hardware is built.

```systemverilog
// procedural for — uses integer/int, executes every time the block runs
integer i;                       // or: int i;  (SystemVerilog, 2-state)
always_comb begin
  parity = 1'b0;
  for (i = 0; i < WIDTH; i = i + 1)
    parity = parity ^ data[i];   // one signal, updated WIDTH times per evaluation
end

// generate for — uses genvar, only legal inside generate, runs once at elaboration
generate
  for (genvar g = 0; g < WIDTH; g++) begin : gen_bit
    assign out[g] = in[g] ^ mask[g];   // instantiates WIDTH separate assign statements
  end
endgenerate
```

| | Procedural `for` | Generate `for` |
|---|---|---|
| Loop variable | `integer` / `int` — a real variable | `genvar` — elaboration-only, no storage |
| Runs | every time the enclosing block is evaluated (sim time) | once, at elaboration/compile time |
| Produces | one signal, updated repeatedly | N copies of hardware |
| Legal where | `always`/`initial`/tasks/functions | only inside a `generate` block |

#### Using an integer variable in a `for` loop — two styles

```systemverilog
// 1) declared beforehand — required in plain Verilog, also legal in SV.
//    'i' is visible (and keeps its last value) for the rest of the module/task
//    after the loop ends, and can be reused by a later, separate for loop.
integer i;
initial begin
  for (i = 0; i < 8; i = i + 1)
    $display("i = %0d", i);
end

// 2) declared inline in the loop header — SystemVerilog only.
//    'i' is scoped to just this loop: it doesn't exist before or after it,
//    so it can't collide with another loop's variable of the same name.
initial begin
  for (int i = 0; i < 8; i++)
    $display("i = %0d", i);
end

// nested loops each need their OWN integer variable — reusing one variable
// as both the outer and inner index silently corrupts the outer count.
initial begin
  for (int row = 0; row < 4; row++)
    for (int col = 0; col < 4; col++)
      mem[row][col] = '0;
end
```

Style 2 (inline `int`) is the common modern default for testbenches/`initial` blocks since it can't leak
or clash; style 1 (pre-declared `integer`) is what plain Verilog requires, and is still the only option
in tools/blocks that don't support inline declarations.

**Watch for lint rules that forbid style 2.** Some strict style guides / lint rule decks (in-house
rule sets, or Verible lint rules configured for it) disallow declaring the loop variable inline and
require it pre-declared instead — for consistency with plain-Verilog code, or because older
tools in the flow don't support inline declarations. Check your project's lint config before
defaulting to `for (int i = ...)`; if it's disallowed, fall back to style 1 (pre-declared `integer`/`int`).

```systemverilog
// interface + modport (SV only)
interface bus_if (input logic clk);
  logic [7:0] data;
  logic       valid;
  modport producer (output data, output valid, input clk);
  modport consumer (input  data, input  valid, input clk);
endinterface
```

## Literal / number formats

`<size>'<base><value>`, e.g. `8'hFF` (hex), `4'b1010` (binary), `3'd5` (decimal), `'0` / `'1` (all-zero / all-one, self-sizing, SV only).

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
