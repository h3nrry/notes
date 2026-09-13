# State Machines (FSM)

## Moore vs Mealy

- **Moore**: outputs depend only on the current state. Slower to react (output changes on the next clock edge after an input changes) but glitch-free and easier to reason about.
- **Mealy**: outputs depend on current state **and** current inputs. Can react within the same clock cycle, but outputs can glitch combinationally with input changes.

## Coding styles

- **1-process**: state register and next-state/output logic combined in a single `always_ff`. Compact, fine for very small FSMs, but mixes sequential and combinational concerns.
- **2-process** (most common, recommended default): one `always_comb` for next-state logic (and Mealy outputs, if any), one `always_ff` for the state register only. Keeps combinational and sequential logic cleanly separated — see [always_comb_ff.md](always_comb_ff.md).
- **3-process**: separate blocks for (1) next-state logic, (2) the state register, (3) Moore output logic. Useful when output logic is heavy and you want to reason about/optimize it independently of next-state logic.

## State encoding

| Encoding | Flops used | Decode logic | Notes |
|---|---|---|---|
| Binary | `ceil(log2(N))` | more | fewest flops; common default for ASIC where flops are relatively costly |
| One-hot | `N` (one per state) | less, very fast | plentiful flops on FPGAs make this a common FPGA default; easy to spot invalid states (should never have 0 or 2+ bits set) |
| Gray | `ceil(log2(N))` | more | only one bit changes between adjacent states; mainly useful when a state's bits cross into another clock domain |

## SystemVerilog specifics

Use a typedef'd enum for the state variable instead of `parameter` constants:

```systemverilog
typedef enum logic [1:0] {IDLE, LOAD, RUN, DONE} state_t;
state_t state, next_state;
```

Benefits: self-documenting waveforms/debug, and `unique case` / `priority case` on the state variable lets the tool flag incomplete or overlapping branches at compile/lint time instead of silently inferring a latch or wrong priority logic.

## Reset

Pick one convention and stay consistent across the codebase:

- **Asynchronous reset**: `always_ff @(posedge clk or negedge rst_n)`, with `if (!rst_n) state <= IDLE; else state <= next_state;` — reacts immediately, but needs a reset synchronizer/de-assertion strategy to avoid recovery/removal timing issues.
- **Synchronous reset**: `always_ff @(posedge clk)`, with `if (!rst_n) state <= IDLE; else state <= next_state;` — simpler timing, but reset only takes effect on a clock edge (needs the clock running to reset).

## Avoiding accidental latches in the next-state logic

Always give `next_state` a default assignment before the `case`, then let the `case` only override it — this guarantees every path assigns `next_state` and prevents latch inference for any state or condition you forgot to handle explicitly:

```systemverilog
always_comb begin
  next_state = state; // default: stay in current state
  unique case (state)
    IDLE: if (start)      next_state = LOAD;
    LOAD: if (load_done)  next_state = RUN;
    RUN:  if (run_done)   next_state = DONE;
    DONE:                 next_state = IDLE;
  endcase
end

always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) state <= IDLE;
  else        state <= next_state;
end
```

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
