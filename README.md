# Synthesizable RTL Style

A portable Agent Skill for writing, reviewing, and refactoring synthesizable SystemVerilog RTL with explicit hardware intent and predictable behavior across ASIC-oriented tool flows.

The package follows the `SKILL.md`-based Agent Skills format. It is tested with OpenAI Codex and can also be used by tools that support this format.

## Scope

Use this skill for synthesizable design RTL, including:

- combinational and sequential logic;
- FSMs, priority logic, and one-hot control;
- arithmetic width, carry, and signedness;
- hierarchy-aware naming and explicit instance-boundary connectivity;
- generate blocks, packages, and synthesis-safe constructs;
- RTL intended for simulation, lint, CDC, formal, synthesis, DFT, STA, place-and-route, gate simulation, and ECO/debug.

Do not apply this skill to:

- testbenches;
- UVM components;
- verification frameworks;
- verification-only functions, tasks, classes, checkers, or modules;
- other non-synthesizable code.

## Key conventions

- Prefer `logic`, `always_comb`, `always_ff`, and explicit `assign` statements.
- Fully assign combinational outputs and make priority and parallelism explicit.
- Treat reset as an architectural decision rather than resetting every register mechanically.
- Make arithmetic widths, carries, signedness, constants, and enum types explicit.
- Use `i_` and `o_` for ordinary module ports, `u_` for instances, and named signals at instance boundaries.
- Avoid `casex`, `casez`, wildcard `case inside`, and `?`/`x`/`z` pattern bits in case items.
- Prefer exact `case` values or named Boolean mask conditions with `if / else if`.
- Avoid hidden expressions in instance port connections and avoid `.*` in ordinary design RTL.
- Name all generate scopes and use ``default_nettype none``.

The complete rules and canonical example are in [`SKILL.md`](skills/synthesizable-rtl-style/SKILL.md).

## Installation

### Install with Codex

Give Codex the URL of this repository and ask:

> Install the `synthesizable-rtl-style` skill from this repository.

The skill directory to install is:

```text
skills/synthesizable-rtl-style
```

### Manual Codex installation

Copy the skill directory into your personal Codex skills directory:

```text
skills/synthesizable-rtl-style
    -> ~/.codex/skills/synthesizable-rtl-style
```

Restart or begin a new Codex turn after installation so the skill can be discovered.

### Other Agent Skills-compatible tools

Install the following directory according to the skill-discovery instructions of the target tool:

```text
skills/synthesizable-rtl-style
```

## Repository layout

```text
.
|-- README.md
|-- LICENSE
`-- skills/
    `-- synthesizable-rtl-style/
        `-- SKILL.md
```

## License

This project is licensed under the MIT License. See [`LICENSE`](LICENSE).
