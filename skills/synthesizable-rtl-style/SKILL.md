---
name: synthesizable-rtl-style
description: >
  Apply this RTL coding style when writing, reviewing, or refactoring synthesizable
  SystemVerilog. The style emphasizes explicit hardware intent, predictable tool
  behavior, hierarchy-aware naming, explicit instance-boundary connectivity,
  width/sign correctness, and conservative constructs for ASIC-oriented flows.
  Do not apply it to testbenches, UVM, verification frameworks, verification
  functions, or other non-synthesizable code.
---

# Synthesizable RTL Style

## When to use this skill

Use this skill when:

- writing synthesizable SystemVerilog RTL;
- reviewing or refactoring RTL for clarity and tool robustness;
- defining module, instance, signal, and hierarchy naming;
- writing combinational or sequential logic;
- writing case statements and one-hot control logic;
- handling arithmetic width, carry, signedness, constants, and X values;
- instantiating submodules;
- writing generate blocks;
- deciding whether to use macros, packages, assertions, or explicit connectivity;
- preparing RTL intended to pass through simulation, lint, CDC, formal, synthesis, DFT, STA, P&R, gate simulation, and ECO/debug flows.

The goal is not merely to write code that synthesizes. The goal is to make hardware intent explicit, expose real hardware structure, and minimize ambiguity across tools and reviewers.

---

## Core principles

Always optimize the RTL for these three goals:

1. **Make intent explicit.**
2. **Expose hardware structure.**
3. **Minimize ambiguity.**

When reviewing RTL, ask:

- What hardware does this code describe?
- Where is priority encoded?
- Which operations are actually parallel?
- Which values are state?
- What is the exact intermediate width?
- What is the exact signedness?
- Who owns this signal hierarchically?
- Will simulation, lint, synthesis, STA, DFT, gate simulation, and debug interpret the code consistently?

Think of the RTL as entering this flow:

```text
RTL
 │
 ├── Simulation
 ├── Lint
 ├── CDC
 ├── Formal
 ├── Synthesis
 ├── DFT
 ├── STA
 ├── P&R
 ├── Gate Simulation
 └── ECO / Debug
```

Coding style is part of the design methodology.

---

## 1. Prefer `logic`

Use `logic` for ordinary point-to-point signals, ports, and procedural variables.

```systemverilog
logic        valid;
logic [31:0] data;
```

Use `wire` only when the signal genuinely requires multiple drivers, such as special wired connections, external IP cases, or tri-state behavior.

Do not use declaration assignment to represent a continuous connection:

```systemverilog
logic [31:0] data = 32'b0;
```

Use an explicit continuous assignment instead:

```systemverilog
logic [31:0] data;

assign data = 32'b0;
```

For structs and unions used in RTL, prefer fully packed representations so layout is explicit and tool interpretation is predictable.

---

## 2. Use `always_comb` and `always_ff`

Prefer:

```systemverilog
always_comb
always_ff @(posedge clk)
```

Do not prefer legacy forms such as:

```systemverilog
always @(*)
always @(posedge clk)
```

Use the SystemVerilog constructs because they explicitly communicate combinational and sequential intent to the tools.

---

## 3. Fully assign combinational outputs

Every signal driven in an `always_comb` block must receive a value on every control path.

Prefer a block-level default assignment:

```systemverilog
always_comb begin
    next_state = state_q;

    if (condition) begin
        next_state = STATE_RUN;
    end
end
```

Do not rely on incomplete assignment:

```systemverilog
always_comb begin
    if (condition) begin
        next_state = STATE_RUN;
    end
end
```

An incomplete path implies storage and can infer a latch.

Prefer block-level defaults over relying on a `case` `default` branch purely to complete assignments.

---

## 4. Avoid reading and writing the same signal in one `always_comb`

Avoid:

```systemverilog
always_comb begin
    foo = foo << 1;
end
```

Prefer a distinct next-value signal:

```systemverilog
logic next_foo;

always_comb begin
    next_foo = foo << 1;
end
```

This avoids sensitivity and interpretation problems around self-dependence inside `always_comb`.

---

## 5. Prefer `assign` for simple combinational relationships

For simple dataflow:

```systemverilog
assign hit = valid && ready;
```

Prefer `assign` over wrapping the same expression in `always_comb`.

Use `always_comb` when the logic genuinely needs procedural control flow such as:

```text
if
case
priority selection
multiple dependent operations
```

For complex or reusable conditions, expose them as named signals:

```systemverilog
assign hit_condition = valid && ready && !flush;

always_comb begin
    ...
    if (hit_condition) begin
        ...
    end
end
```

This improves control-flow readability and waveform observability.

---

## 6. Expose parallelism

Do not write inherently parallel hardware as if it were sequential software.

Avoid:

```systemverilog
always_comb begin
    set = 1'b0;

    if (a && b)
        set = 1'b1;

    if (c || d)
        set = 1'b1;

    if (e)
        set = 1'b1;
end
```

Prefer:

```systemverilog
assign set =
       (a && b)
    || (c || d)
    || e;
```

Or expose intermediate conditions:

```systemverilog
assign cond_a = a && b;
assign cond_b = c || d;

assign set = cond_a || cond_b || e;
```

If there is no true dependency, do not create artificial procedural dependency.

---

## 7. Keep `always_comb` blocks small when natural, but do not force one signal per block

It is often useful to keep separate signals in separate combinational blocks:

```systemverilog
always_comb begin
    foo = ...;
end

always_comb begin
    bar = ...;
end
```

This can improve:

- signal ownership;
- waveform debug;
- dependency analysis;
- simulator optimization;
- avoidance of false combinational-loop or UNOPTFLAT-style issues.

However, this is not a hard rule.

If several outputs are tightly coupled by one control decision, it is acceptable to drive them together in one block.

Do not split tightly related logic merely for stylistic purity.

---

## 8. `always_ff` may contain local update logic

Do not require every register to use an external `*_d` signal.

This is acceptable:

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        cnt_q <= '0;
    end
    else begin
        if (clear) begin
            cnt_q <= '0;
        end
        else if (enable) begin
            cnt_q <= cnt_q + 1'b1;
        end
    end
end
```

Local register-update semantics such as:

```text
clear
enable
increment
```

may remain inside `always_ff`.

Split out `*_d` / `*_q` only when it is useful, such as for:

- large combinational cones;
- complex arithmetic;
- shared calculations;
- long priority trees;
- complex FSM next-state logic;
- reusable conditions;
- explicit debug observation points.

Treat D/Q separation as a tool, not a mandatory style rule.

---

## 9. Reset is architecture-driven

Do not require every register to be reset.

Control-oriented state often needs explicit initialization, for example:

```text
FSM state
valid
pending
protocol control
interrupt status
```

Many datapath registers may not need reset if validity/control already guarantees that their contents are ignored until valid.

Examples include:

```text
DSP datapath
pipeline data
temporary operands
large register arrays
```

Evaluate reset decisions against:

```text
Reset architecture
X methodology
DFT
Reset tree
Area
Power
Routing
Timing
Recovery/removal
Reset-domain crossing
```

Do not mechanically apply reset to every flop.

---

## 10. Use `if / else if` for real priority

Prefer:

```systemverilog
if (req0) begin
    grant = 2'd0;
end
else if (req1) begin
    grant = 2'd1;
end
else if (req2) begin
    grant = 2'd2;
end
```

when the intended priority is:

```text
req0 > req1 > req2
```

Use ordinary `if / else if` as the default way to express priority logic.

Do not use `priority case` broadly. Use it only when case-form priority is genuinely useful and the priority semantics are part of the intended architecture.

Also remember that `if` is X-optimistic in simulation: an X condition is typically treated as false.

---

## 11. Do not unnecessarily mix control flow and Boolean computation

Avoid burying temporary Boolean computation inside control flow when it obscures the structure.

For example:

```systemverilog
if (enabled) begin
    flag = x && y;

    if (flag) begin
        out = a;
    end
end
```

If the intermediate signal has no independent value, this can be written directly:

```systemverilog
if (enabled && x && y) begin
    out = a;
end
```

If the expression is complex, reused, or useful in waveforms, expose it:

```systemverilog
assign hit = x && y;

if (enabled && hit) begin
    ...
end
```

Keep the control structure easy to understand and cover.

---

## 12. Use plain `case` by default

Use ordinary:

```systemverilog
case (...)
```

as the default selection construct.

Do not add `unique` or `priority` merely as style decoration.

A plain `case` is preferred when the RTL should describe only the explicit selection behavior without granting synthesis additional assumptions about completeness, exclusivity, or reachability.

---

## 13. Use `unique case` only for proven architectural assumptions

Use:

```systemverilog
unique case (...)
```

only when all of the following are true:

1. the listed legal cases are mutually exclusive;
2. the legal input/state space is completely covered;
3. unmatched encodings are not part of defined functional behavior;
4. synthesis is allowed to rely on these assumptions for optimization.

Treat `unique` as an architectural contract, not as a routine coding-style keyword.

Do not combine `unique case` with a fallback that suggests unmatched encodings have defined recovery behavior unless that combination is intentionally and fully understood.

If illegal or unmatched encodings have real functional recovery behavior, prefer ordinary `case` and describe that behavior explicitly.

---

## 14. Do not use `full_case` or `parallel_case`

Do not rely on synthesis pragmas such as:

```text
// synopsys full_case
// synopsys parallel_case
```

Do not replace them mechanically with `unique` or `priority`.

Use plain `case`, `if / else if`, or `unique case` according to the actual functional and architectural intent.

---

## 15. Avoid `unique0` unless the tool flow is verified and the semantics are required

Do not use `unique0` by default.

If the current tool flow has been verified for `unique0` and zero-or-one-match semantics are genuinely required, it may be used.

Otherwise prefer plain `case` or another explicit structure that directly reflects the intended behavior.

---

## 16. One-hot selection may use `unique case (1'b1)` only when the one-hot property is proven

For one-hot state or control vectors, this form may be used:

```systemverilog
unique case (1'b1)
    state_q[0]: ...
    state_q[1]: ...
    state_q[2]: ...
endcase
```

but only when the one-hot property and complete legal coverage are established architectural assumptions.

If those assumptions are not guaranteed, do not add `unique` merely to express the coding pattern.

---

## 17. Prefer a block-level default path over a `case default`

When combinational logic has a natural fallback, hold, or default behavior, assign it at the beginning of the `always_comb` block.

Prefer:

```systemverilog
always_comb begin
    next_state = state_q;

    case (state_q)
        IDLE: begin
            if (i_start)
                next_state = RUN;
        end

        RUN: begin
            if (i_done)
                next_state = IDLE;
        end
    endcase
end
```

over using `default` only to complete assignment:

```systemverilog
always_comb begin
    case (state_q)
        IDLE: begin
            ...
        end

        RUN: begin
            ...
        end

        default:
            next_state = state_q;
    endcase
end
```

The preferred interpretation is:

> establish the normal default path first, then let the `case` describe overrides.

Do not add `default` mechanically for latch avoidance or stylistic completeness.

Use a `case default` only when the unmatched encoding itself has distinct functional meaning that should be expressed explicitly.

---

## 18. Do not modify correct RTL only to close coverage holes

If coverage reports an unreachable or structurally impossible branch as a hole, do not change otherwise-correct RTL merely to make the report green.

Prefer an appropriate coverage waiver or constraint when the design intent is already correct.

---

## 19. Keep large logic out of `case` branches when possible

Avoid deeply nested control structure such as:

```systemverilog
case
  state A:
      case
          ...
      endcase
  state B:
      ...
endcase
```

Factor complex selection into separate combinational logic when that improves readability:

```systemverilog
always_comb begin
    ...
    start_next = ...;
end

always_comb begin
    unique case (state_q)
        START: next_state = start_next;
        ...
    endcase
end
```

Keep the main control structure easy to scan.

---

## 20. Do not use wildcard values in `case` items

Do not use:

```systemverilog
casex
casez
case (...) inside
```

Do not use `?`, `x`, or `z` bits in `case` item literals to represent wildcard or don't-care behavior.

Prefer ordinary `case` with exact, fully known values. If masked selection is genuinely required, expose the mask comparisons as named Boolean conditions and use `if / else if` so the matching behavior is explicit.

For example, avoid:

```systemverilog
case (status) inside
    3'b00?: ...
    3'b01?: ...
    3'b1?0: ...
    3'b1?1: ...
endcase
```

Prefer:

```systemverilog
logic status_is_00;
logic status_is_01;
logic status_is_1_0;
logic status_is_1_1;

assign status_is_00  = (status[2:1] == 2'b00);
assign status_is_01  = (status[2:1] == 2'b01);
assign status_is_1_0 = (status[2] == 1'b1) && (status[0] == 1'b0);
assign status_is_1_1 = (status[2] == 1'b1) && (status[0] == 1'b1);

always_comb begin
    ...

    if (status_is_00) begin
        ...
    end
    else if (status_is_01) begin
        ...
    end
    else if (status_is_1_0) begin
        ...
    end
    else if (status_is_1_1) begin
        ...
    end
end
```

This keeps wildcard behavior out of `case` statements and makes every ignored bit explicit in the comparison logic. Do not allow input X/Z values to become accidental matches.

---

## 21. Make expression associativity explicit

Use parentheses in complex arithmetic and Boolean expressions.

Prefer:

```systemverilog
assign hit =
       (a && b)
    || (c && d);
```

Make nested ternaries easy to read:

```systemverilog
assign result =
    cond_a              ? value_a :
    (cond_b && !cond_c) ? value_b :
                          value_c;
```

Do not force reviewers to reconstruct operator precedence mentally.

---

## 22. Make bit widths explicit

Always reason about intermediate expression width.

For example:

```systemverilog
logic [15:0] a;
logic [15:0] b;

assign result = (a + b) >> 1;
```

must be reviewed for whether `a + b` retains the carry before shifting.

If the intended operation is:

\[
result = \frac{a+b}{2}
\]

make the extension explicit:

```systemverilog
logic [16:0] sum;

assign sum    = {1'b0, a} + {1'b0, b};
assign result = sum >> 1;
```

Or use an explicit cast.

Do not rely unnecessarily on implicit SystemVerilog expression-sizing rules.

---

## 23. Explicitly capture carry, even when unused

Prefer:

```systemverilog
logic unused_co;

assign {unused_co, result} = a + b;
```

over silently truncating:

```systemverilog
assign result = a + b;
```

The `unused_` prefix communicates that the discarded carry is intentional.

---

## 24. Use signed types for signed arithmetic

Prefer:

```systemverilog
logic signed [15:0] a;
logic signed [15:0] b;
```

over manual sign extension scattered throughout the RTL.

Be careful when signed and unsigned operands mix:

```systemverilog
logic signed [3:0] a;
logic signed [3:0] b;
logic              ci;

sum = a + b + ci;
```

An unsigned operand such as `ci` may affect the signedness of the whole expression.

Do not rely on implicit signedness propagation when it matters.

---

## 25. Do not unnecessarily split one arithmetic cone

If the intended combinational operation is:

```systemverilog
sum = a + b + c;
```

do not split it without a structural reason:

```systemverilog
tmp = a + b;
sum = tmp + c;
```

and do not split the same arithmetic cone across modules without reason.

Allow synthesis to see the full expression when possible so it can optimize adders, carry structures, or compressor trees.

A real pipeline boundary is a valid reason to split:

```text
a + b
  │
 FF
  │
 + c
```

---

## 26. Make constants explicit

Avoid magic numbers:

```systemverilog
if (cnt == 37)
```

Use a named constant when the value has design meaning:

```systemverilog
localparam logic [5:0] p_cnt_max = 6'd37;
```

Or use a package constant.

Be explicit about literal width where relevant.

Remember that:

```systemverilog
'1
```

means “all bits set to one at the destination width,” not numeric one.

For example:

```systemverilog
logic [15:0] data;

assign data = '1;
```

produces:

```text
16'hffff
```

`'0` is generally less ambiguous because it produces all zeros at any width.

---

## 27. Preserve enum type safety

If a port or signal uses an enum type, connect or assign values of the same enum type when possible.

Prefer:

```systemverilog
.foo_state(pkg::IDLE)
```

over:

```systemverilog
.foo_state(2'b00)
```

Do not depend unnecessarily on implicit enum conversion.

---

## 28. Do not use X as a routine synthesis don't-care

Avoid:

```systemverilog
default: data = 'x;
```

when the only intent is to give synthesis optimization freedom.

RTL simulation sees X, while silicon can only implement 0 or 1.

Use X-based optimization only when there is a demonstrated benefit and the verification methodology explicitly supports it.

---

## 29. Naming must encode useful hierarchy

Signal naming should improve:

```text
waveform search
hierarchy trace
flattened-netlist trace
gate-level debug
ECO
```

Naming should make direction and hierarchy ownership clear.

---

## 30. Current-module ports use `i_` and `o_`

All ordinary module inputs use:

```text
i_<semantic>
```

All ordinary module outputs use:

```text
o_<semantic>
```

Example:

```systemverilog
module audio_top (
    input  logic [23:0] i_data,
    input  logic        i_valid,

    output logic [23:0] o_data,
    output logic        o_valid
);
```

Do not use:

```text
data_i
data_o
```

for current-module ports.

---

## 31. Clock and reset do not use `i_`

Clock and reset are exceptions to the ordinary port-prefix rule.

Use:

```systemverilog
clk
rst_n
```

For multiple domains:

```systemverilog
clk_sys
clk_audio
clk_i2s

rst_sys_n
rst_audio_n
rst_i2s_n
```

Do not use:

```systemverilog
i_clk
i_rst_n
```

---

## 32. Instance names use `u_<name>`

Use:

```systemverilog
filter u_filter (...);
eq     u_eq     (...);
drc    u_drc    (...);
```

This makes instance names immediately recognizable in hierarchy and debug tools.

---

## 33. Instance-boundary signals use `<inst>_i_<semantic>` and `<inst>_o_<semantic>`

For `u_filter`, use:

```systemverilog
logic [23:0] filter_i_data;
logic        filter_i_valid;

logic [23:0] filter_o_data;
logic        filter_o_valid;
```

Interpret direction from the instantiated block's point of view:

```text
filter_i_xxx → enters u_filter
filter_o_xxx → leaves u_filter
```

Example:

```systemverilog
filter u_filter (
    .clk     (clk),
    .rst_n   (rst_n),

    .i_data  (filter_i_data),
    .i_valid (filter_i_valid),

    .o_data  (filter_o_data),
    .o_valid (filter_o_valid)
);
```

---

## 34. Explicitly create logic between module ports and instance ports

Do not connect a top-level module port directly to a submodule port when applying this style.

Avoid:

```systemverilog
filter u_filter (
    .i_data(i_data),
    .o_data(o_data)
);
```

Prefer:

```systemverilog
logic [23:0] filter_i_data;
logic [23:0] filter_o_data;

assign filter_i_data = i_data;
assign o_data        = filter_o_data;

filter u_filter (
    .i_data(filter_i_data),
    .o_data(filter_o_data)
);
```

This creates an explicit boundary between the current module interface and the instantiated-module interface.

Input flow:

```text
i_data
   │
 assign
   ▼
filter_i_data
   │
   ▼
u_filter.i_data
```

Output flow:

```text
u_filter.o_data
   │
   ▼
filter_o_data
   │
 assign
   ▼
o_data
```

---

## 35. Preserve both sides of instance-to-instance connectivity

For:

```text
HPF → EQ
```

do not directly connect:

```systemverilog
eq u_eq (
    .i_data(hpf_o_data)
);
```

Prefer separate source-side and destination-side signals:

```systemverilog
logic [23:0] hpf_o_data;
logic [23:0] eq_i_data;

assign eq_i_data = hpf_o_data;
```

This explicitly preserves both hierarchy-boundary names:

```text
u_hpf
   │
   ▼
hpf_o_data
   │
 assign
   ▼
eq_i_data
   │
   ▼
u_eq
```

Synthesis may collapse these nets, but the source RTL should preserve both ownership views.

---

## 36. Use `assign` to express hierarchy connectivity

In this style, `assign` serves two purposes:

1. simple dataflow;
2. explicit hierarchy-boundary connectivity.

Example:

```systemverilog
assign hpf_i_data  = i_data;
assign eq_i_data   = hpf_o_data;
assign drc_i_data  = eq_o_data;
assign o_data      = drc_o_data;
```

These assignments should make the major datapath visible even before inspecting the instance declarations.

---

## 37. Do not hide logic inside module instantiations

Avoid:

```systemverilog
foo u_foo (
    .i_enable(a && b && !c),
    .i_data  (x + y)
);
```

Prefer:

```systemverilog
logic        foo_i_enable;
logic [31:0] foo_i_data;

assign foo_i_enable = a && b && !c;
assign foo_i_data   = x + y;

foo u_foo (
    .i_enable(foo_i_enable),
    .i_data  (foo_i_data)
);
```

Module instantiation should primarily express connectivity.

This improves:

```text
Lint
CDC
Waveform
ECO
Netlist trace
Port-width review
```

---

## 38. Avoid `.*` in synthesizable design RTL

Use explicit named port connections:

```systemverilog
.i_data  (filter_i_data),
.i_valid (filter_i_valid),
.o_data  (filter_o_data)
```

Do not use `.*` in ordinary synthesizable design RTL.

`.*` may be acceptable in:

```text
testbench
very simple wrappers
highly mechanical interface shells
```

The design-RTL default is explicit connectivity.

---

## 39. Name all generate scopes

Do not write:

```systemverilog
for (genvar i = 0; i < 4; i++) begin
```

Write:

```systemverilog
for (genvar i = 0; i < 4; i++) begin : g_channel
```

This gives stable hierarchy such as:

```text
g_channel[0]
g_channel[1]
g_channel[2]
g_channel[3]
```

instead of tool-generated names such as:

```text
genblk1
genblk2
```

Explicitly name generate `for`, `if`, and `else` scopes.

---

## 40. Minimize use of the preprocessor

Prefer:

```systemverilog
parameter
localparam
generate
```

over:

```text
`define
`ifdef
```

Avoid:

```systemverilog
`ifdef FEATURE
    ...
`endif
```

when the same design choice can be represented through elaboration:

```systemverilog
generate
    if (p_feature) begin : g_feature
        ...
    end
endgenerate
```

Macros do not have normal module scope, can pollute later compilation units, and can make behavior depend on file ordering.

If a local macro is unavoidable:

```systemverilog
`define LOCAL_XXX
...
`undef LOCAL_XXX
```

so it does not leak farther than intended.

---

## 41. Use ``default_nettype none``

RTL files should use:

```systemverilog
`default_nettype none
```

This prevents undeclared identifiers or misspelled connection names from silently creating implicit wires.

---

## 42. Naming conventions for parameters, state, active-low signals, types, and unused signals

Use:

```text
p_xxx       parameter / localparam
xxx_q       registered value
xxx_n       active-low
xxx_t       typedef type
unused_xxx  intentionally unused signal
g_xxx       generate scope
u_xxx       instance
```

For intentionally discarded values:

```systemverilog
logic unused_co;

assign {unused_co, result} = a + b;
```

The naming itself should communicate that the unused value is intentional.

---

## 43. Pipeline stage naming is useful, but do not over-encode

Reasonable examples:

```text
p1_valid
p2_valid_q
p3_data_q
```

Avoid names that attempt to encode every property at once, such as:

```text
filter_o_p2_data_valid_q_n
```

Optimize names for searchability and semantic clarity, not maximum information density.

---

## 44. Recommended module organization

Use a consistent module structure such as:

```text
1. Module ports

2. Parameters / localparams

3. typedef / enum

4. Instance-boundary logic
   xxx_i_*
   xxx_o_*

5. Local combinational signals

6. Registers / state
   *_q

7. Boundary assign / simple assign

8. always_comb

9. always_ff

10. Submodule instances

11. Assertions / bind externally
```

The exact ordering may be adapted consistently across the project, but the project should use one predictable structure.

---

## 45. Use packages for shared definitions, but avoid wildcard imports

Shared:

```text
type
constant
function
task
```

may live in packages.

Prefer explicit qualification:

```systemverilog
core_pkg::fifo_width
core_pkg::state_t
core_pkg::gray_code(...)
```

over:

```systemverilog
import core_pkg::*;
```

Explicit qualification makes origin clear, limits namespace pollution, and improves search/debug.

---

## 46. Keep assertions separate from synthesizable RTL when practical

Prefer verification-only assertions in a separate file, for example:

```text
block_assert.sv
```

and attach them through `bind`.

This makes it easier to use different assertion sets for:

```text
RTL simulation
formal
gate simulation
```

Do not mix large verification-only property sets into the synthesizable implementation unless there is a clear reason.

---

## 47. Make unused or dangling intent explicit

If a signal or port is intentionally unused, do not leave the intent ambiguous.

Use a clear name such as:

```text
unused_xxx
```

This simplifies:

```text
lint waivers
review
integration
```

and distinguishes intentional unused logic from accidental omission.

---

## 48. Formatting consistency matters

Use a consistent formatting style.

Prefer:

- spaces instead of tabs;
- controlled line length;
- line breaks for complex expressions;
- `begin/end` for `if`, `else`, and `always_*` blocks;
- consistent formatting across the project.

The important property is consistency.

---

## 49. Canonical rule summary

| Category | Rule |
|---|---|
| RTL mindset | Describe real hardware, not software execution |
| Tool philosophy | Minimize ambiguity across simulation, synthesis, lint, and downstream flows |
| Ordinary signals | Default to `logic` |
| Multiple drivers | Use `wire` only when needed |
| Combinational logic | Use `always_comb` |
| Sequential logic | Use `always_ff` |
| Simple dataflow | Prefer `assign` |
| `always_comb` completeness | Assign every driven signal on every control path |
| Combinational self-dependence | Avoid reading and writing the same signal in one block |
| Parallelism | Express inherently parallel logic as parallel logic |
| Comb block size | Prefer small blocks when natural; do not force one signal per block |
| `always_ff` | Local update logic is allowed |
| D/Q split | Use when useful; do not require it |
| Reset | Architecture-driven; do not reset every flop by default |
| Priority | Prefer ordinary `if / else if`; use `priority case` only when case-form priority is genuinely required |
| Case | Use plain `case` by default |
| `unique` | Use only when mutual exclusivity and complete legal coverage are proven architectural assumptions |
| `unique0` | Avoid unless the tool flow is verified and the semantics are required |
| One-hot | Use `unique case (1'b1)` only when one-hot behavior and complete legal coverage are proven |
| Old case pragmas | Do not use `full_case` / `parallel_case` |
| Default path | Prefer a block-level default assignment at the start of `always_comb` |
| `case default` | Use only when unmatched encoding has distinct functional meaning |
| Coverage holes | Prefer waiver/constraint over changing correct RTL |
| Wildcard match | Do not use `casex`, `casez`, `case inside`, or `?`/`x`/`z` pattern bits in case items; use exact `case` values or explicit Boolean mask conditions |
| Expressions | Make associativity explicit with parentheses |
| Width | Make intermediate widths explicit |
| Carry | Explicitly capture unused carry with `unused_` |
| Signed arithmetic | Use signed types and explicit casts where needed |
| Arithmetic | Do not unnecessarily split one combinational arithmetic cone |
| Constants | Use explicit width/type and avoid magic numbers |
| Enum | Preserve enum type safety |
| X | Do not routinely use X as a synthesis don't-care |
| Current-module input | `i_xxx` |
| Current-module output | `o_xxx` |
| Clock | `clk` / `clk_xxx`, without `i_` |
| Reset | `rst_n` / `rst_xxx_n`, without `i_` |
| Instance | `u_xxx` |
| Instance input net | `xxx_i_name` |
| Instance output net | `xxx_o_name` |
| Module ↔ instance | Explicit logic + `assign` |
| Instance ↔ instance | Preserve both source-output and destination-input names |
| Instance hookup | Connect named signals only; do not hide expressions in ports |
| `.*` | Do not use in ordinary design RTL |
| Generate | Name all scopes with `g_xxx` |
| Parameter | `p_xxx` |
| Register | `_q` |
| Active-low | `_n` |
| Type | `_t` |
| Intentional unused | `unused_xxx` |
| Macros | Minimize; prefer parameter/generate |
| Default net type | Use ``default_nettype none`` |
| Package | Prefer explicit `pkg::symbol` over `::*` |
| Assertions | Prefer separate assertion files + bind |
| Formatting | Be consistent |

---

## 50. Canonical example

```systemverilog
`default_nettype none

module audio_top #(
    parameter int p_data_w = 24
) (
    input  logic                clk,
    input  logic                rst_n,

    input  logic [p_data_w-1:0] i_data,
    input  logic                i_valid,

    output logic [p_data_w-1:0] o_data,
    output logic                o_valid
);


// -----------------------------------------------------------------------------
// HPF interface
// -----------------------------------------------------------------------------

logic [p_data_w-1:0] hpf_i_data;
logic                hpf_i_valid;

logic [p_data_w-1:0] hpf_o_data;
logic                hpf_o_valid;


// -----------------------------------------------------------------------------
// EQ interface
// -----------------------------------------------------------------------------

logic [p_data_w-1:0] eq_i_data;
logic                eq_i_valid;

logic [p_data_w-1:0] eq_o_data;
logic                eq_o_valid;


// -----------------------------------------------------------------------------
// Module / instance connectivity
// -----------------------------------------------------------------------------

assign hpf_i_data  = i_data;
assign hpf_i_valid = i_valid;

assign eq_i_data   = hpf_o_data;
assign eq_i_valid  = hpf_o_valid;

assign o_data      = eq_o_data;
assign o_valid     = eq_o_valid;


// -----------------------------------------------------------------------------
// Instances
// -----------------------------------------------------------------------------

hpf #(
    .p_data_w (p_data_w)
) u_hpf (
    .clk      (clk),
    .rst_n    (rst_n),

    .i_data   (hpf_i_data),
    .i_valid  (hpf_i_valid),

    .o_data   (hpf_o_data),
    .o_valid  (hpf_o_valid)
);


eq #(
    .p_data_w (p_data_w)
) u_eq (
    .clk      (clk),
    .rst_n    (rst_n),

    .i_data   (eq_i_data),
    .i_valid  (eq_i_valid),

    .o_data   (eq_o_data),
    .o_valid  (eq_o_valid)
);

endmodule
```

This should read structurally as:

```text
TOP
 i_data
   │
assign
   ▼
hpf_i_data
   │
 u_hpf
   │
hpf_o_data
   │
assign
   ▼
eq_i_data
   │
 u_eq
   │
eq_o_data
   │
assign
   ▼
 o_data
```

Port direction, instance ownership, hierarchy boundaries, and major dataflow should be visible directly in the RTL source.

The key project-specific deviations from stricter style guides are:

- reset is architecture-driven rather than mandatory for every register;
- `always_ff` may contain reasonable local update logic;
- hierarchy-boundary naming and explicit `assign` connectivity are preferred and intentionally retained in source RTL.
