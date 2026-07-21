# AILC Chemistry

**AI Learning Companion — Chemistry**: an agent skill for interactive chemistry tutoring grounded in the [Wolfram Knowledgebase](https://www.wolfram.com/knowledgebase/) via Wolfram MCP.

It turns open-ended questions about molecules, functional groups, elements, and properties into **structure plots**, curated facts, isomer/class comparisons, and notable reactions/molecules with real-world photos — without dumping property JSON or inventing structure image URLs.

Molecule-plot behavior is adapted from the Chemistry chapter of the Wolfram plugin notebook `AILCBioChem.nb` (`moleculePlot`, property helpers, isomers, chemical class lists). Anatomy / biology body plots from that notebook are **out of scope**.

## What it does

- Resolves chemicals and elements and draws **2D/3D molecule plots** (aromatic 2D by default)
- **Highlights functional groups** (named patterns or SMARTS) and lists groups found
- Pulls **curated properties** (formula, mass, melting point, …) — not full dumps
- Finds **isomers** and samples chemicals in a **class**
- Grounds teaching with a **notable reaction or related molecule**, illustrated by a real substance photo when available
- Keeps the chat **fast**: dispatch parallel Hermes sub-agents first (they inherit Wolfram + `skill_view`), then high-level framing while plots finish in the background
- Stays honest about missing data, plot timeouts, and dual-use safety limits

## Repository layout

```text
.
├── SKILL.md                      # Parent tutor + orchestration (entry point)
└── references/
    ├── worker-brief.md           # Subagent load order + output contract
    ├── wolfram-recipes.md        # Sole home for WL evaluator rules + recipes
    ├── grounding.md              # Visual matrix, AI illustration, honesty
    ├── reply-format.md           # Learner-facing structure (parent teaching turns)
    └── pedagogy.md               # Context pass, levels, flow
```

Parent tutors from `SKILL.md`. Subagents are told (in `delegate_task` context) to `skill_view` `worker-brief.md`, then `wolfram-recipes.md` / `grounding.md` as needed — they do not load the full tutor skill body.

## Requirements

| Dependency | Role |
|------------|------|
| **Wolfram MCP** | Chemicals, elements, `Molecule` / `MoleculePlot` / `MoleculePlot3D`, properties, isomers |
| **skills** toolset | `skill_view` for worker brief and recipes (parent and children) |

If Wolfram is unavailable, the skill falls back to careful prose and does not invent structure image URLs.

## Install (Hermes)

Register this repo as a [Hermes skills tap](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills#publishing-a-custom-skill-tap), then install the skill. Use the `owner/repo` form — not a full GitHub URL.

```bash
hermes skills tap add warmersun/ailc-chemistry-skill
hermes skills install warmersun/ailc-chemistry-skill/skills/ailc-chemistry --category ailc
```

After that, `/ailc-chemistry` is available in chat. To pick up upstream changes later:

```bash
hermes skills update ailc-chemistry
```

`tap add` only registers the source; `install` is what copies `SKILL.md` and `references/` into `~/.hermes/skills/`. Prefer the install identifier above over `hermes skills search` — custom taps are easy to miss in search (Hermes may skip GitHub when its index is available, and `--source github` can time out while scanning the large default taps).

Ensure Wolfram MCP is configured (`WolframContext`, `WolframAlpha`, `WolframLanguageEvaluator` or equivalent).

## Usage

Trigger when the learner wants to explore chemistry, for example:

- “Draw caffeine and show its functional groups”
- “3D structure of aspirin”
- “What are the isomers of C4H10?”
- “Properties of iron”
- “Show me amino acids in the chemical class”
- `/ailc-chemistry`

Typical turn shape:

1. `delegate_task` fan-out (each child context mandates `skill_view` of worker-brief + wolfram-recipes)  
2. High-level framing in plain language after the handle returns  
3. Later synthesis in the reply format: headed sections, full sentences that gloss terms, short tables when useful, one strong structure visual, a notable beat with substance photo when available, a next hook  

## Entity / input types (Wolfram)

Canonical list and evaluator rules live in [`references/wolfram-recipes.md`](references/wolfram-recipes.md). Names are always resolved with free-form / typed interpreters — never hand-written canonical entity IDs.

Inputs supported (aligned with plugin `moleculePlot`):

| type | Example |
|------|---------|
| `ChemicalName` | `caffeine`, `D-ribose` |
| `SMILES` | `CCO` |
| `InChI` | InChI string |
| PubChem / ChEMBL IDs | external identifiers |
| `Peptide` / `DNA` / `RNA` | short sequences as molecules (not anatomy) |

## Design principles

- **Learner-facing only** — no pedagogy jargon or orchestration labels in replies  
- **Compute over guess** — structures from Wolfram when bonds and groups matter  
- **DRY references** — WL rules only in wolfram-recipes; shared visuals/honesty in grounding; workers get a thin brief  
- **Curate** — few strong artifacts beat long dumps  
- **Photo-first for physical appearance** — real substance photos when available; AI images labeled modern reconstructions, never substitutes for molecule plots  

## License

[MIT](LICENSE) — Copyright (c) 2026 Sic
