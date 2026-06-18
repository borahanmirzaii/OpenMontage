# OpenMontage — Developer Onboarding

> A senior-dev orientation to *how this repo thinks*, not just what it contains.
> Read this once, and the rest of the codebase stops looking strange.

---

## TL;DR — the one idea that explains everything

> **The AI agent is the runtime. Python is only a library of capabilities + a persistence layer.**

Everything you'd normally write as orchestration code — control flow, branching,
retries, "creative" decisions, review logic — lives in **instructions**
(YAML manifests + Markdown skills) that the model reads and executes.

If you arrive expecting a `main.py` orchestrator with a `StateMachine` class,
you will waste an hour hunting for it. **It does not exist, on purpose.**

```
The state machine  = a YAML manifest (pipeline_defs/*.yaml)
The executor       = the LLM following AGENT_GUIDE.md
The "functions"    = tools (Python BaseTool subclasses)
The "stdlib docs"  = skills (Markdown)
```

Internalize that mapping and the whole repo snaps into focus.

---

## Why build it this way? (the intuition)

Traditional video tools hardcode the pipeline: *generate script → call TTS →
render → mux*. The moment you want a different creative treatment, you edit code.

OpenMontage flips it. The **pipeline is data** (YAML), the **know-how is prose**
(Markdown), and the **only code is the irreducible capability** (call ElevenLabs,
run FFmpeg, render Remotion). The bet: an LLM following good written instructions
makes better creative + routing decisions than branching `if/else` ever could —
*and* you can change behavior by editing a Markdown file instead of shipping code.

Three consequences fall out of that bet:

1. **No Python orchestrator, no Python reviewer, no Python handlers.**
   This is enforced as a rule, not a style preference. Python that makes a
   creative or routing decision is treated as a *bug*.
2. **Capability is discovered at runtime, never hardcoded.**
   The same repo behaves differently on two machines, because "what can I do"
   is a live function of which API keys + binaries are present.
3. **Every seam is a typed contract.**
   Stages hand off via canonical JSON artifacts validated against
   `schemas/artifacts/`. A stage is swappable as long as it emits a valid artifact.

---

## The three knowledge layers (the spine)

```
Layer 1   tools/            →  WHAT exists, is it available, what's it cost   (Python — runtime truth)
Layer 2   skills/           →  HOW OpenMontage uses it in a pipeline           (Markdown — project conventions)
Layer 3   .agents/skills/   →  HOW the underlying tech/API actually works      (Markdown — vendor knowledge)
```

The bridge from Layer 1 → Layer 3 is **one field** on every tool:
`agent_skills: [...]`. That is the pointer the agent follows to load
provider-specific prompting *before* it ever calls a model.

> 🔑 **Intuition:** the gap between a generic prompt and a "cinematic" one is
> mechanically *just this field*. Layer 2 tells you *what & when*; Layer 3 tells
> you *how*. Skip Layer 3 and you get usable-but-bland output.

**The reading order the repo wants from you:**

```
registry  →  Layer 2 pipeline/stage skill  →  Layer 3 vendor skill  →  only then .py
```

Source-diving into `.py` is explicitly *sanctioned for debugging and audits* —
just not for normal usage. (When a skill and the code disagree, the code wins,
and you should fix the skill afterward.)

---

## The four primitives you actually need

### 1. `BaseTool` — the capability contract · `tools/base_tool.py`

Every tool is an ABC subclass. The interesting part is **not** `execute()` —
it's the rich **self-describing contract** each tool carries:

| Field | Why it matters |
|---|---|
| `runtime` | `LOCAL` / `LOCAL_GPU` / `API` / `HYBRID` — drives cost + availability story |
| `dependencies` | `env:FOO`, `cmd:ffmpeg`, `python:torch` — checked live by `check_dependencies()` |
| `determinism` | `DETERMINISTIC` / `SEEDED` / `STOCHASTIC` — reproducibility contract |
| `fallback_tools` | graceful degradation path |
| `agent_skills` | the Layer 1 → Layer 3 bridge (see above) |
| `cost` / `estimate_cost()` | feeds budget governance |

`get_status()` literally *tries* `check_dependencies()` and reports
`AVAILABLE` / `UNAVAILABLE` / `DEGRADED`. **Availability is computed, not
declared** — that's what makes the "0/13 configured" preflight menus honest.

> ⚠️ **Gotchas that will bite you:**
> - Tool classes are **PascalCase with NO `Tool` suffix** → `VideoCompose`, not `VideoComposeTool`.
> - You invoke via `.execute(dict)` → returns `ToolResult`. **Never `.run()`.**
> - `ToolResult` has `.success`, `.data`, `.error`, `.cost_usd`, `.seed`, `.model`.

### 2. `ToolRegistry` — discovery · `tools/tool_registry.py`

`discover()` walks the `tools/` package tree with `pkgutil` and auto-registers
every concrete `BaseTool` subclass.

> 🔑 **Intuition:** drop a new tool file into the right package and it is
> discovered with **zero wiring**. No central registration list to edit.

The agent queries it through:
- `provider_menu_summary()` — the human-ready preflight rollup (use this first)
- `capability_catalog()` — tools grouped by capability (tts, image_generation, …)
- `provider_catalog()` — tools grouped by provider (elevenlabs, openai, ffmpeg, …)

### 3. The **selector pattern** — capability routing

For multi-provider families there is a **router tool + N provider tools**:

| Selector | Routes to |
|---|---|
| `tts_selector` | ElevenLabs / Google / OpenAI / Piper / Doubao |
| `image_selector` | FLUX / Imagen / DALL-E / Recraft … |
| `video_selector` | Kling / LTX / Seedance / Wan / Hunyuan … |

Selectors **auto-discover** their providers via `registry.get_by_capability(...)`.
Add a provider → it appears in the selector with **no selector edits**.
Routing priority: **user preference > availability > discovery order.**

### 4. Pipeline manifest — the declarative state machine · `pipeline_defs/*.yaml`

This is where the "orchestrator" actually lives. Each stage declares:

| Manifest key | Meaning |
|---|---|
| `skill` | the director skill that teaches the agent HOW |
| `produces` | the canonical output artifact |
| `required_artifacts_in` | input contract from prior stages |
| `tools_available` / `required_tools` / `optional_tools` | capability scope |
| `review_focus` | what self-review checks |
| `success_criteria` | the gate to pass the stage |
| `human_approval_default` | checkpoint policy |

**The state machine:**

```
research → proposal → script → scene_plan → assets → edit → compose → publish
```

> 🔑 **Intuition:** *creative* stages (idea, script, scene_plan) gate on human
> approval; *technical* stages (assets, edit, compose) auto-proceed. The split
> reflects where human judgement actually adds value.

---

## The governance layer (the "production-grade" part)

This is what separates OpenMontage from a toy agent demo. Three hard rules,
enforced as **contract violations** (a reviewer flags them as CRITICAL):

| Rule | What it prevents |
|---|---|
| **Announce before spend** | Agent must state tool / provider / model / cost *before* any paid call |
| **No silent substitution** | If a locked render runtime is unavailable, the agent must escalate a *structured blocker* — not quietly swap engines |
| **Present both composition runtimes** | When Remotion *and* HyperFrames both exist, the agent must surface both — never pick a silent default |

`render_runtime` is **locked at proposal** and carried through `edit_decisions`
unchanged. `video_compose` dispatches on that field to one of three engines:

| Engine | Best for | Requires |
|---|---|---|
| **FFmpeg** | cuts, concat, trim, subtitle burn | `ffmpeg` (always present) |
| **Remotion** | React scenes: stat cards, charts, callouts, spring transitions | Node + `remotion-composer/` |
| **HyperFrames** | HTML/CSS/GSAP: kinetic type, product promos, SVG rigs | Node ≥ 22 + FFmpeg + `npx` |

A `decision_log` artifact records `options_considered` / `rejected_because` —
a literal audit trail for *why* each provider/runtime was chosen.

> 🔑 **Intuition:** "motion is a hard requirement" — a brief that promises moving
> shots may **not** be silently downgraded to a Ken Burns slideshow. If the chosen
> engine is unavailable, the agent stops and asks; it does not improvise a cheaper path.

---

## How to navigate the repo

| You want… | Go to |
|---|---|
| The agent contract you're bound by | `AGENT_GUIDE.md` |
| Architecture truth (no duplication) | `PROJECT_CONTEXT.md` + `docs/ARCHITECTURE.md` |
| What tools exist *right now* | `registry.discover()` — never trust docs |
| How a stage should behave | `skills/pipelines/<pipeline>/<stage>-director.md` |
| Checkpoint / review policy | `skills/meta/` + manifest `human_approval_default` |
| Add a tool | subclass `BaseTool`, drop in right `tools/<family>/`, set contract fields — discovery is automatic |
| Add a pipeline | YAML manifest + director skills + contract tests |
| Persistence / state | `lib/checkpoint.py`; checkpoints at `pipelines/<project_id>/checkpoint_<stage>.json` |
| Budget governance | `tools/cost_tracker.py` (estimate → reserve → reconcile) |

---

## Mandatory preflight (run this first, every session)

```bash
python -c "
from tools.tool_registry import registry
import json
registry.discover()
print(json.dumps(registry.provider_menu_summary(), indent=2))
"
```

It returns four fields to translate into plain language for the user:

- `composition_runtimes` — booleans for ffmpeg / remotion / hyperframes
- `capabilities[]` — `configured / total` counts per capability family
- `setup_offers[]` — unavailable tools fixable with a 1-minute env-var
- `runtime_warnings[]` — silent-failure signals to surface verbatim

---

## The honest "watch out for" list

- **The intelligence is in Markdown, not code.** Bugs often live in stale skill
  files, not `.py`. Grep `skills/` as readily as `tools/`.
- **Beta pipelines** (`talking-head`, `clip-factory`, `character-animation`, …)
  aren't fully audited — expect rough edges; tell the user.
- **Never write ad-hoc scripts to call tools directly.** That's the cardinal sin
  (Rule Zero). Everything routes through a pipeline — or the reviewer flags it.
- **`projects/` is gitignored.** All generated assets are regenerable; never committed.

---

## Add-a-tool, in practice

1. Inherit from `tools/base_tool.py::BaseTool`.
2. Put it in the right capability package (`tools/audio/`, `tools/video/`, …).
3. Prefer the **selector + provider** pattern (one router, one tool per backend).
4. Set every contract field — especially `capability`, `provider`, `supports`,
   `fallback_tools`, `agent_skills`.
5. Implement `execute()` → return a `ToolResult`.
6. Let `tool_registry.discover()` find it. **No ad-hoc imports.**
7. Add a JSON schema in `schemas/tools/` if I/O is complex.
8. Add tests *after* the runtime path is correct.

---

## Where to go deeper

- **Operating it** → run the preflight, then walk a pipeline end-to-end.
- **Extending tools** → trace `video_compose`'s runtime-routing (the most intricate tool).
- **Agent-contract internals** → `lib/checkpoint.py` + `skills/meta/reviewer.md` + `checkpoint-protocol.md`.

---

*Generated as part of the `onboarding` branch.*
