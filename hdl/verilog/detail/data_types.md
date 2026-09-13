# Data Types

## Overview

Two different things get declared with type-like syntax in Verilog/SystemVerilog, and it's worth
keeping them straight: **signal types** (`wire`, `reg`, `logic`, `bit`, `int`, `integer`, ...) declare
real storage or a real net — something with hardware behind it (or a real variable in simulation).
**Compile-time constants** (`parameter`, `localparam`) and the elaboration-only **`genvar`** are
declared with the same kind of syntax but have no storage or hardware at all — they're resolved away
before simulation or synthesis even runs. This note covers both, since mixing them up is one of the
most common early confusions.

## Net & variable types

| Type | Kind | Notes |
|---|---|---|
| `wire` | 4-state, net | continuous assignment target (`assign`), or module output |
| `reg` | 4-state, variable | Verilog's procedural-assignment type (name is historical — doesn't imply a real register) |
| `logic` | 4-state, variable | SystemVerilog: replaces `reg`/`wire` for most signals; can't be driven by more than one source |
| `bit` | 2-state | SystemVerilog; `0`/`1` only, no `x`/`z` — good for testbench state, not for RTL that must model unknowns |
| `int` / `integer` | 2-state / 4-state | `int` is SV, 2-state, 32-bit; `integer` is Verilog, 4-state, 32-bit |
| `byte`, `shortint`, `longint` | 2-state | SystemVerilog fixed-width integer types (8/16/64-bit) |

### 4-state vs. 2-state

4-state types (`wire`, `reg`, `logic`, `integer`) can hold `0`, `1`, `x` (unknown), or `z`
(high-impedance) per bit — needed for real RTL, where an uninitialized register, an undriven net, or
a tri-state bus genuinely can be unknown, and where a linter's X-propagation check ([linter.md](linter.md))
depends on `x` actually being representable. 2-state types (`bit`, `int`, `byte`, `shortint`,
`longint`) only ever hold `0`/`1` — faster to simulate, but silently turn any `x`/`z` they're assigned
into a `0` or `1`, so they're mainly used in testbench/verification code rather than RTL that must
model unknowns.

## User-defined types: `enum`, `struct`, `union`

### `enum`

```systemverilog
typedef enum logic [1:0] {
  IDLE = 2'b00,
  RUN  = 2'b01,
  DONE = 2'b10
} state_t;

state_t state, next;
```

- Explicit values (`= 2'b00`, ...) are optional — leaving them off assigns `0, 1, 2, ...` in declaration order.
- The base type (before the `{...}` list) defaults to `int` (2-state, 32-bit) if omitted; giving it an
  explicit type and width, as `logic [1:0]` above, is what lets it map directly onto real state-register bits.
- Wrapping it in `typedef` (as above) lets `state_t` be reused for multiple signals or across modules;
  an anonymous `enum {...} state;` works but can't be reused elsewhere.
- `unique`/`priority case` over an enum-typed signal — see [state_machine.md](state_machine.md) and the
  case-statement examples in [reference.md](reference.md) — is especially useful, because the tool can
  check your case list against every value the enum could legally hold.

### `struct` / `union`

```systemverilog
typedef struct packed {
  logic       valid;
  logic [7:0] data;
} pkt_t;

pkt_t pkt_in, pkt_out;
```

`packed` lays the fields out contiguously as one bit vector (so `pkt_in` can also be treated like a
plain `logic [8:0]`) — needed for anything that has to synthesize to real hardware. An *unpacked*
struct/union has no guaranteed bit layout and is mainly for testbench-side bookkeeping. `union` shares
one storage location between its members (only one is meaningfully valid at a time) — rare in RTL,
more common in verification code that needs to reinterpret the same bits multiple ways.

## `parameter` and `localparam` — constants that look like variable declarations

`parameter` and `localparam` are declared with the same kind of syntax as a variable
(`parameter int WIDTH = 8;` reads a lot like `int width = 8;`), which is exactly why they get
confused with variable declarations — but neither creates any storage or hardware. Both are resolved
to fixed values at elaboration time, before simulation or synthesis actually runs, and neither can be
the target of a procedural assignment (a `parameter` can't appear on the left of `=` inside
`always`/`initial`).

The difference between the two is about who is allowed to set the value:

- **`parameter`** — a compile-time constant the *outside world* can override per instance, either at
  instantiation (`fifo #(.WIDTH(16)) u_fifo (...)`) or, in older Verilog, via `defparam` (deprecated —
  avoid it; it can override a parameter from far away in the code with no local visibility). Used for
  module configuration knobs: widths, depths, counts.
- **`localparam`** — a compile-time constant fixed *inside* the module that cannot be overridden from
  outside, even with `#(...)`. Used for internal-only constants, especially ones derived from a
  `parameter`, so a user of the module can change the `parameter` without also needing to (or being
  able to) touch a value that must stay consistent with it.

```systemverilog
module fifo #(
  parameter  int DEPTH = 16,              // overridable — module's public configuration
  parameter  int WIDTH = 8
) (
  input  logic             clk,
  input  logic             rst_n,
  input  logic [WIDTH-1:0] data_in,
  output logic [WIDTH-1:0] data_out
);

  localparam int ADDR_W = $clog2(DEPTH);  // derived from DEPTH — NOT separately overridable

  logic [WIDTH-1:0]  mem [0:DEPTH-1];
  logic [ADDR_W-1:0]  wr_ptr, rd_ptr;
  // ...
endmodule
```

### How this relates to variable declaration — a comparison

| | Signal variable (`logic`/`integer`/`int`/`reg`) | `genvar` | `parameter` | `localparam` |
|---|---|---|---|---|
| Has real storage/hardware? | yes — a real register/net, or a real software variable in simulation | no — elaboration-only loop index | no — compile-time constant | no — compile-time constant |
| Can be assigned at runtime? | yes, in procedural code (`always`/`initial`/tasks) | no — fixed per generate iteration | no | no |
| Overridable from outside the module? | n/a — it's internal state, not configuration | n/a | yes — `#( .NAME(value) )` at instantiation | no — fixed once, inside the module |
| Typical use | data storage, loop index in simulation | generate-block loop index | module configuration (`WIDTH`, `DEPTH`, ...) | internal constants derived from parameters (address widths, opcodes) |

See [reference.md](reference.md) for the procedural-`for`-loop and generate-`for`-loop examples that
use the `integer`/`int` and `genvar` rows of this table.

## Declaring loop / integer variables

```systemverilog
integer i;   // Verilog, 4-state, 32-bit — usable in any procedural block (always/initial/task/function)
int    j;    // SystemVerilog, 2-state, 32-bit — same role, tighter (X/Z-free) semantics
genvar k;    // elaboration-only index — legal ONLY inside a generate block, no storage/hardware
```

See [reference.md](reference.md) for the full procedural-vs-generate `for` loop writeup, including the
two styles of declaring the loop variable (pre-declared vs. inline `for (int i = ...)`) and which lint
rules can forbid the inline form.

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
