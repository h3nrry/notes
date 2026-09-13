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

## Arrays: 1D and 2D declarations

A dimension written *before* the variable name is **packed** (part of the type — one contiguous bit
vector you can slice); a dimension written *after* the name is **unpacked** (a list of separate,
independently-addressable elements). This is what actually decides whether something is "an array" in
the everyday sense.

### 1D

```systemverilog
logic [7:0] byte_val;        // packed 1D — a single 8-bit vector, ONE element
logic [7:0] mem [0:15];      // unpacked 1D array — 16 separate 8-bit elements, indexed mem[i]
logic       flags [0:7];     // unpacked 1D array — 8 separate 1-bit elements
```

`byte_val` isn't really an array — it's one 8-bit value. `mem` is the real 1D array: 16 independent
8-bit words, the shape already used for the register file in the [`fifo`](#parameter-and-localparam--constants-that-look-like-variable-declarations)
and [`packet_counter`](#how-to-use-it--a-worked-example) examples above.

### 2D

```systemverilog
logic [3:0][7:0] packed2d;         // packed 2D — one 32-bit vector, sliceable a byte at a time
logic [7:0]      matrix [0:3][0:3]; // unpacked 2D array — 16 separate 8-bit elements, matrix[row][col]
logic            bitmap [0:7][0:7]; // unpacked 2D array — 64 separate 1-bit elements
```

`packed2d` is still just one 32-bit vector under the hood — `packed2d[1]` gives you bits `[15:8]` — so
it synthesizes as a flat bus. `matrix` is 16 genuinely independent 8-bit storage elements addressed by
two indices; whether a synthesis tool maps a 2D *unpacked* array like this onto a single block RAM or
flattens/splits it varies by tool and target, so check that if you're relying on it for memory.

### How to initialize a 2D array variable

- **Nested `for` loop in an `initial` block** — most portable, works in plain Verilog too, and is
  simulation-only unless your synthesis tool happens to honor `initial`-block contents for a memory
  (common for FPGA block RAM, generally not for ASIC):

  ```systemverilog
  logic [7:0] matrix [0:3][0:3];

  initial begin
    for (int row = 0; row < 4; row++)
      for (int col = 0; col < 4; col++)
        matrix[row][col] = 8'(row * 4 + col);
  end
  ```

- **SystemVerilog array-literal assignment (`'{...}`)** — declare and initialize in one step, for
  unpacked arrays only. Each `'{...}` is one row; nesting them one level gives the 2D shape:

  ```systemverilog
  logic [7:0] matrix [0:1][0:1] = '{
    '{8'h01, 8'h02},   // row 0
    '{8'h03, 8'h04}    // row 1
  };
  ```

  `'{default: value}` fills the rest of a level with one value (e.g. `'{8'h01, default: 0}` for a row);
  nesting `default` two levels deep for a full 2D array gets unwieldy, so the explicit nested-literal
  form above is usually clearer once there's more than one row.

- **`$readmemh` / `$readmemb`** — the standard way to load real memory contents from a hex/binary text
  file, and the one initialization method most synthesis tools also recognize for pre-loading block
  RAM:

  ```systemverilog
  logic [7:0] mem [0:15];             // 1D memory array
  initial $readmemh("mem_init.hex", mem);
  ```

  `$readmemh`/`$readmemb` only fill one unpacked dimension directly, so a genuinely 2D memory is
  usually modeled as a 1D array addressed as `mem[row*COLS + col]`, or loaded with one `$readmemh` call
  per row from separate files.

| | Packed | Unpacked |
|---|---|---|
| Dimension position | before the name: `logic [7:0] x;` | after the name: `logic x [0:7];` |
| Represents | one contiguous, sliceable bit vector | separate, independently-addressable elements |
| Combines to 2D as | `logic [3:0][7:0] x;` | `logic x [0:3][0:7];` (packed + unpacked can also combine: `logic [7:0] x [0:15];`) |
| Typical use | a bus/word you slice | memory arrays, register files, lookup tables |

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

## How to use it — a worked example

Everything above, put together in one module: `parameter`/`localparam` for configuration, `enum` for
named states, a packed `struct` for a grouped input, `logic` for real storage, and `genvar` for
structural repetition.

```systemverilog
module packet_counter #(
  parameter  int DATA_W   = 8,             // parameter: caller can override per instance
  parameter  int MAX_PKTS = 16
) (
  input  logic              clk,
  input  logic              rst_n,
  input  logic              pkt_valid,
  input  logic [DATA_W-1:0] pkt_data,
  output logic [DATA_W-1:0] parity_out,
  output logic              full
);

  localparam int CNT_W = $clog2(MAX_PKTS + 1);   // localparam: derived from MAX_PKTS, fixed inside

  typedef enum logic [1:0] {                     // enum: named states instead of raw bits
    IDLE, COUNTING, FULL
  } state_t;

  typedef struct packed {                        // struct: group related fields as one bit vector
    logic              valid;
    logic [DATA_W-1:0] data;
  } pkt_t;

  state_t           state, next_state;           // logic-based variables: real storage
  logic [CNT_W-1:0] pkt_count;
  pkt_t             pkt_in;

  assign pkt_in.valid = pkt_valid;
  assign pkt_in.data  = pkt_data;
  assign full          = (state == FULL);

  // sequential: real storage updated on the clock
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      state     <= IDLE;
      pkt_count <= '0;
    end else begin
      state <= next_state;
      if (pkt_in.valid && state != FULL)
        pkt_count <= pkt_count + 1;
    end
  end

  // combinational: enum-driven case — unique case checks it covers every state_t value
  always_comb begin
    unique case (state)
      IDLE:     next_state = pkt_in.valid ? COUNTING : IDLE;
      COUNTING: next_state = (pkt_count == MAX_PKTS - 1) ? FULL : COUNTING;
      FULL:     next_state = FULL;
      default:  next_state = IDLE;
    endcase
  end

  // generate: structural repetition — genvar 'g' has no storage, disappears after elaboration
  generate
    for (genvar g = 0; g < DATA_W; g++) begin : gen_parity
      assign parity_out[g] = pkt_in.data[g] ^ pkt_in.valid;
    end
  endgenerate

endmodule
```

Instantiating it overrides the `parameter`s but not `CNT_W` — that stays derived and internal:

```systemverilog
packet_counter #(
  .DATA_W  (16),
  .MAX_PKTS(32)
) u_pc (
  .clk       (clk),
  .rst_n     (rst_n),
  .pkt_valid (valid),
  .pkt_data  (data),
  .parity_out(parity),
  .full      (full)
);
```

| Construct in the example | What it is | Section |
|---|---|---|
| `DATA_W`, `MAX_PKTS` | `parameter` — configurable per instance | [parameter and localparam](#parameter-and-localparam--constants-that-look-like-variable-declarations) |
| `CNT_W` | `localparam` — derived, not overridable | same section |
| `state_t` | `enum` — named states, pairs with `unique case` | [enum](#enum) |
| `pkt_t` | packed `struct` — groups fields as one bit vector | [struct / union](#struct--union) |
| `state`, `pkt_count`, `pkt_in` | `logic` — real storage/hardware | [Net & variable types](#net--variable-types) |
| `g` | `genvar` — elaboration-only, no storage | [comparison table](#how-this-relates-to-variable-declaration--a-comparison) |

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
