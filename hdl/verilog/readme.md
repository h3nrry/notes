# Verilog vs SystemVerilog

SystemVerilog (IEEE 1800) is a superset of Verilog (IEEE 1364): essentially all legal Verilog is legal SystemVerilog, but SystemVerilog adds a large set of design and verification features on top. File extension is `.v` for Verilog, `.sv` for SystemVerilog.

## Key differences

| Aspect | Verilog (IEEE 1364) | SystemVerilog (IEEE 1800) |
|---|---|---|
| Scope | RTL design + basic simulation | RTL design + full verification/testbench language |
| Data types | `reg`, `wire`, `integer`, `real`, `time` | adds `logic`, `bit`, `byte`, `shortint`, `int`, `longint`, `struct`, `union`, `enum`, `string`, dynamic/associative arrays, queues |
| Value system | 4-state only (`0,1,x,z`) | 4-state types kept (`logic`, `reg`) + new 2-state types (`bit`, `int`, ...) |
| Procedural blocks | `always @(...)`, `initial` | adds `always_comb`, `always_ff`, `always_latch` (intent is explicit — see [detail/always_comb_ff.md](detail/always_comb_ff.md)) |
| Module connectivity | ports + parameters only | adds `interface` / `modport` to bundle related signals |
| Code reuse | `` `include``, macros | adds `package`, `import` |
| Functions/tasks | limited — see [detail/function.md](detail/function.md) | default arg values, pass-by-`ref`, `void` functions, explicit `return` |
| Control flow | `case`, `if` | adds `unique case`, `priority case`, `foreach`, enhanced `for`, `do...while` |
| Verification | none built-in | classes (OOP), `rand`/constraints, `covergroup`, assertions (`assert`, `cover`, `property`, `sequence`), mailboxes, semaphores — basis of UVM |
| Assertions | none | immediate assertions (`assert (expr)`) and concurrent assertions (SVA: `property`/`sequence`) |

In short: Verilog is the RTL modeling language; SystemVerilog keeps that but folds in stronger typing, clearer procedural intent, and a full verification language on top.

## Details

Deeper notes on specific topics live in [`detail/`](detail/):

- [Functions & tasks](detail/function.md) — Verilog function/task rules and what SystemVerilog adds
- [State machines](detail/state_machine.md) — FSM styles, encoding, reset conventions
- [always_comb / always_ff (combinational vs sequential)](detail/always_comb_ff.md) — blocking vs non-blocking, latch inference, why SystemVerilog split `always` into three

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
