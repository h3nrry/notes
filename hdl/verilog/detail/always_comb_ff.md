# always: combinational vs sequential (comb vs ff)

## Verilog: one construct, `always @(...)`

- `always @(*)` — combinational logic. The simulator/tool infers the sensitivity list from every signal read on the right-hand side. Must use **blocking** assignments (`=`), and every output must be assigned on every possible path (all `if`/`case` branches covered) or a latch is inferred.
- `always @(posedge clk)` — sequential logic. Must use **non-blocking** assignments (`<=`).
- `always @(posedge clk or negedge rst_n)` — sequential logic with an asynchronous reset.

The limitation: `always @(...)` doesn't encode *intent*. Nothing stops you from writing blocking assignments inside a clocked block, or an incomplete `if` inside a combinational block — the tool just infers whatever the code implies, bug or not.

## SystemVerilog: explicit intent

- **`always_comb`** — combinational logic only. Automatically computes its sensitivity list (like `@(*)`, but also re-evaluates once at time 0 without waiting for an input to change first). Tools flag it as an error/warning if you use `<=` inside, or if the same signal is also driven by another block. Use blocking (`=`) assignments only.
- **`always_ff`** — sequential (flip-flop) logic only. Tools warn on blocking (`=`) assignments, or on mixing `=` and `<=`. Use non-blocking (`<=`) assignments only, and put exactly one clock edge (plus, optionally, one asynchronous reset edge) in the event control.
- **`always_latch`** — an intentional level-sensitive latch. Rare in synchronous FPGA/ASIC design; if a tool infers a latch from `always_comb` or plain `always @(*)`, it's almost always a bug (a missing `else` or incomplete `case`), not a real intended latch.

## Golden rules

1. **Combinational block**: assign every output on every branch (always include `else` / a `default` case) — this is what avoids unintended latches. Use `=`.
2. **Sequential block**: use `<=` for every assignment, so every right-hand-side read in the block sees the value from *before* the clock edge, regardless of statement order.
3. **Never mix** `=` and `<=` in the same `always`/`always_ff`/`always_comb` block.
4. **Never drive the same signal from two different always blocks** (including two `always_comb` blocks) — SystemVerilog tools flag this as multi-driven.
5. Prefer `always_comb` / `always_ff` / `always_latch` over plain `always @(...)` whenever the toolchain supports SystemVerilog — the extra checking catches classic bugs (missing `else` → latch, blocking assignment in sequential logic, incomplete sensitivity list) at compile/lint time instead of in synthesis or in silicon.

## Comparison

| Construct | Language | Logic type | Assignment | Sensitivity |
|---|---|---|---|---|
| `always @(*)` | Verilog / SV | combinational | blocking `=` | inferred automatically |
| `always @(posedge clk)` | Verilog / SV | sequential | non-blocking `<=` | explicit edge |
| `always_comb` | SV only | combinational | blocking `=` | inferred + checked |
| `always_ff` | SV only | sequential | non-blocking `<=` | explicit edge, checked |
| `always_latch` | SV only | level-sensitive latch | blocking `=` | inferred + checked |

## Example: the classic latch bug

```systemverilog
// BUG: no else branch -> q holds its old value when sel==0 -> latch inferred
always_comb begin
  if (sel) q = d;
end

// FIXED: every branch assigns q -> combinational, no latch
always_comb begin
  if (sel) q = d;
  else     q = '0;
end
```

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
