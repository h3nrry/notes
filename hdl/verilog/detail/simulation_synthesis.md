# Simulation & Synthesis (Compilation)

## Why these are different from linting

Linting ([linter.md](linter.md)) checks source *statically*, with no notion of time or hardware. Simulation and synthesis both go further, but in different directions: a **simulator** compiles the design into an executable *model of its behavior* and runs it in software against a testbench; a **synthesis/compiler** tool maps the design onto real hardware (a technology library or FPGA fabric). Something can simulate perfectly and still fail to synthesize (or synthesize into something different from what simulated) — that's the whole reason both steps exist.

## Simulation

A simulator elaborates the design + testbench, then runs an event-driven (or cycle-accurate) execution of it, producing waveforms and pass/fail results from assertions/checkers — no hardware is produced.

| Tool | Type | Notes |
|---|---|---|
| Verilator (`verilator --binary`) | free, open-source | compiles Verilog/SV to C++ for very fast simulation; no waveform GUI of its own; the default for CI regressions |
| Icarus Verilog (`iverilog` + `vvp`) | free, open-source | traditional event-driven simulator; lightweight, good for learning/small designs; weaker SystemVerilog support |
| Synopsys VCS | commercial | industry-standard event-driven simulator, full SV/UVM support |
| Cadence Xcelium | commercial | industry-standard event-driven simulator, full SV/UVM support |
| Siemens Questa / ModelSim | commercial | widely used in industry & academia, full SV/UVM support, strong debug GUI |

```sh
# Icarus Verilog
iverilog -g2012 -o sim.out tb.sv dut.sv
vvp sim.out

# Verilator (self-checking testbench in C++, or --binary for a pure-SV flow)
verilator --binary -j 0 -Wall --trace top.sv
./obj_dir/Vtop
```

Dump a waveform (`$dumpfile("wave.vcd"); $dumpvars;` in the testbench, or `--trace` above for Verilator's FST) and view it in a waveform viewer (GTKWave, or the commercial tool's own GUI) when a test fails.

## Synthesis (a.k.a. "compilation" to hardware)

Synthesis parses and elaborates the RTL (the same front end as lint/sim), infers hardware structures (flops, latches, muxes, adders), maps the design onto a target cell library or FPGA primitives, optimizes for area/timing/power, and emits a gate-level netlist or bitstream plus timing reports.

| Tool | Type | Notes |
|---|---|---|
| Yosys | free, open-source | open synthesis suite; commonly paired with NextPNR for iCE40/ECP5 FPGA flows |
| Xilinx/AMD Vivado (`synth_design`) | vendor, free with device | FPGA synthesis + place & route for Xilinx/AMD parts |
| Intel Quartus Prime | vendor, free with device | FPGA synthesis + place & route for Intel/Altera parts |
| Synopsys Design Compiler (`dc_shell`) | commercial | industry-standard ASIC logic synthesis |
| Cadence Genus | commercial | ASIC logic synthesis, alternative to Design Compiler |

```sh
yosys -p "read_verilog top.sv; synth_ice40 -top top; write_json top.json"
```

Not everything that simulates is synthesizable: `#delay` statements, unbounded/dynamic loops, real-number math, and most `initial` blocks (other than an FPGA's power-on-reset idiom) either get ignored or rejected by a synthesis tool, because there's no hardware equivalent. Synthesis warnings about inferred latches or width mismatches are the same classes of bug the linter catches — just reported with the tool's own messaging, and reported later in the flow.

## Where each step catches problems

| Stage | Runs on | Catches | Doesn't catch |
|---|---|---|---|
| Lint ([linter.md](linter.md)) | source, statically | latches, case completeness, blocking/non-blocking misuse, width mismatches | actual functional/timing behavior |
| Simulation | a software model | functional bugs, protocol violations, corner cases exercised by the testbench | anything the testbench doesn't stimulate; real hardware timing/power |
| Synthesis | RTL → real hardware mapping | non-synthesizable constructs, resource usage, static timing (with constraints) | bugs the testbench never triggers; simulation/synthesis mismatch is only caught by re-simulating the gate-level netlist |

## Practical tips

- Run in this order: lint → simulate (RTL) → synthesize → simulate the synthesized netlist (gate-level, with SDF back-annotation) as a final sanity check before tapeout or bitstream generation.
- Keep testbenches self-checking (assertions/scoreboards) rather than relying on manually eyeballing waveforms — it's what makes simulation useful in CI.
- A design that simulates correctly but synthesizes differently almost always traces back to a non-synthesizable construct or a sensitivity-list/latch bug that lint should have flagged first.

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
