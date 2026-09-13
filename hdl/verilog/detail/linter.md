# Linting

## Why lint

A linter checks source code *statically* (no simulation) for patterns that are almost always bugs or bad practice — before you burn time in simulation or synthesis. Most of the rules a linter enforces are things already covered elsewhere in these notes: unintended latches ([always_comb_ff.md](always_comb_ff.md)), incomplete `case` statements ([state_machine.md](state_machine.md)), and blocking/non-blocking misuse ([always_comb_ff.md](always_comb_ff.md)).

## Common tools

| Tool | Type | Notes |
|---|---|---|
| Verilator (`verilator --lint-only`) | free, open-source | very fast, widely used as a first pass; also a full simulator |
| Verible (`verible-verilog-lint`) | free, open-source | style + lint rules, configurable rule set, good for style consistency across a team |
| Yosys (`read_verilog` + checks) | free, open-source | synthesis tool with some lint-style checks as a side effect of elaboration |
| Synopsys SpyGlass | commercial | deep, sign-off-quality lint (CDC, RDC, low-power, testability rule decks) |
| Cadence HAL / JasperGold (lint app) | commercial | similar sign-off-quality lint, integrates with formal |
| Siemens Questa Lint / Autocheck | commercial | lint + auto-generated formal checks (X-propagation, FSM reachability) |

Open-source tools (Verilator, Verible) are normally enough for day-to-day RTL hygiene; commercial sign-off lint decks (SpyGlass etc.) get added at a project level for tapeout-quality checks (clock-domain crossing, reset-domain crossing, low power).

## What a linter typically catches

- **Unintended latch inference** — an `always_comb`/`always @(*)` block that doesn't assign an output on every path (missing `else`, incomplete `case`). See [always_comb_ff.md](always_comb_ff.md).
- **Blocking (`=`) vs non-blocking (`<=`) misuse** — blocking assignment inside `always_ff`, non-blocking inside `always_comb`, or mixing both in one block.
- **Incomplete / overlapping `case` statements** — especially useful combined with `unique`/`priority case` (SystemVerilog), which asks the tool to actively check this. See [state_machine.md](state_machine.md).
- **Multiple drivers** — the same signal assigned from more than one `always` block or both a continuous assignment and a procedural block.
- **Unused or undriven signals** — a declared signal that's never read, or never written.
- **Width mismatches** — assigning a wider or narrower value than the target without an explicit cast, which can silently truncate or zero/sign-extend.
- **Implicit net declarations** — using a signal name without declaring it (Verilog silently makes it a 1-bit `wire`) — usually a typo. `` `default_nettype none `` at the top of a file turns this into a compile error.
- **X-propagation risk** — reset or enable logic that could let an unknown (`x`) value propagate into control logic.
- **Inline `for`-loop variable declarations** — some strict style guides/lint rule decks forbid `for (int i = ...)` and require the loop variable pre-declared instead, for portability or house-style consistency. See [reference.md](reference.md).

## Running a quick local check (example: Verilator)

```sh
verilator --lint-only -Wall my_module.sv
```

`-Wall` turns on the full warning set; individual warnings can be waived inline with a pragma comment (`/* verilator lint_off WIDTH */ ... /* verilator lint_on WIDTH */`) when a specific instance is a deliberate, reviewed exception rather than a bug.

## Practical tips

- Run lint on every file before it goes into simulation — it's much faster than a sim cycle and catches the same classic bugs earlier.
- Fix the root cause instead of waiving a warning where possible; reserve waivers for cases you've actually reviewed and understand.
- Turn on `unique`/`priority case` in SystemVerilog RTL specifically so the lint/synthesis tool double-checks case completeness, not just style.

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
