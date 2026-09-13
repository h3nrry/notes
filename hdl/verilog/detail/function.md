# Functions & Tasks

## Verilog functions

```verilog
function [7:0] add_one;
  input [7:0] a;
  begin
    add_one = a + 1; // return value assigned to the function's own name
  end
endfunction
```

Rules:

- Must execute in **zero simulation time** — no `#`, `@`, or `wait` inside.
- Must have **at least one input**.
- Returns exactly **one value**, via an implicit variable that shares the function's name.
- **Static by default**: all calls share one copy of local variables. Declare `function automatic` to get a fresh copy per call (required for recursion).
- Can only be called from procedural code (inside `always`/`initial`) or from a continuous assignment's expression.

## Verilog tasks (for contrast)

- Can contain timing controls (`#`, `@`, `wait`) — i.e. can consume simulation time.
- Can have `input`, `output`, and `inout` arguments, and zero or many "return" values (via outputs).
- Can call other tasks and functions.
- Static by default, same as functions, unless declared `automatic`.

Use a **function** for a same-time-step calculation with one result; use a **task** when you need timing control or multiple outputs.

## What SystemVerilog adds

```systemverilog
function automatic void print_sum(int a, int b = 0, ref int result);
  result = a + b;
  return; // optional for void functions
endfunction
```

- **Explicit return type** before the name (`function int foo(...)`), and a `return expr;` statement can be used instead of (or alongside) assigning the function's own name.
- **`void` functions** — declared with no return value, callable as a statement (e.g. `print_sum(1, 2, sum);`), useful for functions used only for their side effects.
- **Default argument values**: `function void foo(int a, int b = 0);` — lets callers omit trailing args.
- **Pass by reference**: `ref` (read/write) and `const ref` (read-only, avoids copying large types like arrays/structs) argument passing, instead of Verilog's value-only arguments.
- **Default lifetime differs by scope**: at module scope, functions/tasks are still `static` by default (same as Verilog) unless declared `automatic`. Inside a `class`, methods are `automatic` by default, since every object needs its own copy of local variables.
- The zero-time restriction on functions is **unchanged** — SystemVerilog functions still cannot contain time-consuming statements; use a task (or a class method called via `fork`/`join` in testbench code) when you need to consume time.
- SystemVerilog does **not** support true overloading (multiple functions with the same name but different signatures, C++-style) — default argument values cover most of the same use cases instead.

## Quick comparison

| Feature | Verilog | SystemVerilog |
|---|---|---|
| Declare return type | via the function's name/range | explicit type before the name |
| Return statement | none — assign to the function's name | `return expr;` supported (as well as assigning the name) |
| Default lifetime | static (module scope) | static at module scope, automatic inside a class |
| Default argument values | no | yes |
| Pass by reference | no | yes (`ref`, `const ref`) |
| Void (no return value) | not possible — a function always returns a value | yes, `void` functions, callable as a statement |
| Time-consuming statements allowed | no | no (unchanged) |

---

© 2026 Henrry Andrian‍​‌​​‌​​​​‌‌​​‌​‌​‌‌​‌‌‌​​‌‌‌​​‌​​‌‌‌​​‌​​‌‌‌‌​​‌​‌​​​​​‌​‌‌​‌‌‌​​‌‌​​‌​​​‌‌‌​​‌​​‌‌​‌​​‌​‌‌​​​​‌​‌‌​‌‌‌​​​‌‌​​‌​​​‌‌​​​​​​‌‌​​‌​​​‌‌​‌‌​‍. All rights reserved.
