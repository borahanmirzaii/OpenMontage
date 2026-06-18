# OpenMontage — Design Philosophy

> Why this repo looks the way it does. Read `ONBOARDING.md` for *what* the
> system is; read this for *why* it's shaped that way — so you draw the same
> lines the next time you extend it.

---

## The thesis in one line

> **The agent absorbs the glue. What survives is irreducible capability and hard contracts.**

Everything in between — the orchestration that used to be the bulk of the
codebase — moves out of code and into **prose (skills)** and **data (YAML)**.

---

## Why glue was always the tax, never the value

The oldest truth in software: *glue code is most of the code.* Parsing,
branching, sequencing, retry, fallback routing, "if this provider is down try
that one" — the actual capability (call FLUX, run FFmpeg) is a thin sliver. The
rest is orchestration scaffolding.

That scaffolding was never the value. **It was the tax** — the unavoidable cost
you paid to ship the value.

An LLM-driven agent changes the economics: **the glue tax goes toward zero.**
The model *is* the interpreter that used to live in your `switch` statements.
So the code collapses to the two things that genuinely have to be code:

1. **Irreducible capability** — `BaseTool.execute()`. Something must actually
   hit the API or shell out to FFmpeg.
2. **Hard contracts** — JSON schemas, governance rules. Something must
   *guarantee* correctness where it matters.

Everything between those two evaporates into skills (Markdown) and pipeline
manifests (YAML).

---

## Quantitative → Qualitative

This is the conceptual heart of the shift.

| | Imperative code | Instruction-driven |
|---|---|---|
| You write… | every case, branch, edge | the intent + the quality bar |
| The model of work | **quantitative** — you *enumerate* the combinatorial space by hand | **qualitative** — you *specify judgment*; the agent fills the space |
| A missing case is… | a bug you ship | a situation the agent reasons about |
| Changing behavior | edit + recompile + redeploy | edit a Markdown / YAML file |

Imperative code forces you to *count* — every fallback, every permutation. Miss
one, you get a bug. Instruction-driven lets you state things like *"concepts
must be genuinely different"* or *"motion is a hard requirement"* and trust the
agent to fill the combinatorial space those statements imply.

That is the move from quantitative to qualitative: **you stopped enumerating
cases and started specifying judgment.**

---

## The trap: "no state machine, just instructions"

It is tempting to conclude that the agent makes structure obsolete — that
instructions and tools *suffice* on their own.

They do not. And the repo proves it by **keeping** the structure, just moving it:

- The state machine didn't disappear. It moved **from code into data** — the
  pipeline manifest (`pipeline_defs/*.yaml`) is a declarative state machine:
  `research → proposal → script → scene_plan → assets → edit → compose → publish`.
- The contracts didn't disappear. They *hardened* — JSON schemas at every seam
  (`schemas/artifacts/`), plus governance rules enforced as CRITICAL violations.

> A YAML state machine is not *less* than a coded one. It is a state machine you
> can edit without recompiling, that a non-coder can author, and that the agent
> can read as part of its own context. You kept the rigor **and** gained the
> malleability. That is the win — not the absence of structure.

---

## The actual design principle

The thing that makes the agent powerful — **interpretation, judgment** — is the
same thing that makes it nondeterministic and untrustworthy *where mistakes are
expensive*. So the governing rule is:

> **Spend determinism where mistakes hurt. Spend flexibility where judgment helps.**

OpenMontage encodes this boundary almost perfectly:

| Where | What it uses | Why |
|---|---|---|
| Creative stages (`script`, `scene_plan`) | freeform agent + human approval | judgment *is* the value; nondeterminism is acceptable |
| Money / irreversible (paid generation, render) | hard schemas + governance + `decision_log` audit trail | a wrong call costs real dollars and can't be undone — pin it down |

This is why "instructions and tools suffice" is only *half* true. They suffice
**because** the contracts and human-approval gates catch the cases where the
agent's qualitative judgment would otherwise run off the rails. Strip those out
and you have a demo, not a production system.

The genius is that it *looks* like "just instructions and tools" but is actually
**a very deliberate boundary between the soft and the hard.**

---

## What the new craft is

The senior-engineering skill shifts. It used to be *writing control flow.* Now
it is two harder things:

### 1. Drawing the line

Decide what to delegate to the agent's judgment vs what to nail into a contract.
Draw it wrong and the system is either **unsafe** (too much freedom near money/
irreversibility) or **brittle** (too much rigidity near creative work). The line
is the architecture.

### 2. Authoring the specification

Skills and schemas *are* the program now.

- A **stale skill** is a bug — the agent will faithfully execute outdated intent.
- A **loose schema** is a security hole — it's the only thing standing between
  judgment and a costly mistake.
- The **prose is load-bearing.** Grep `skills/` as readily as you grep `tools/`.

---

## How to apply this when you extend the repo

When you add a capability or a pipeline, ask in order:

1. **Is this irreducible capability?** → it's a `BaseTool`. Keep it dumb: do the
   thing, return a `ToolResult`. No creative or routing logic inside.
2. **Is this a decision the agent should make?** → it's a skill (Markdown).
   Express intent and quality bars, not branches.
3. **Is this a guarantee that must hold no matter what the agent decides?** → it's
   a schema or a governance rule. Make it hard, make it fail loudly.
4. **Is this sequencing / stage flow?** → it's the pipeline manifest (YAML), not
   Python.

If you ever find yourself writing orchestration `if/else` in Python, stop — that
glue belongs in a skill or a manifest. If you find yourself trusting the agent
with something irreversible and unverified, stop — that needs a contract.

The discipline is the boundary. Everything good about this system lives there.

---

*Companion to [`ONBOARDING.md`](ONBOARDING.md) · written on the `onboarding` branch.*
