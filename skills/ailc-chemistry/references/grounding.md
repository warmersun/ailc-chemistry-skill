# Visual grounding and honesty (AILC Chemistry)

Shared by the **parent tutor** and **subagent workers**. Wolfram evaluator rules and recipes: `references/wolfram-recipes.md`. Learner reply shape (parent only): `references/reply-format.md`.

## Visual choice matrix

| Intent | Prefer |
|--------|--------|
| Learn a molecule’s structure | `2DAromatic` `MoleculePlot` (default) |
| Functional groups / reactive sites | 2D plot **with highlights** or auto-detected groups |
| Stereochemistry / shape / docking intuition | `3DBallAndStick` (fallback 2D if timeout) |
| Space-filling bulk / steric sense | `3DSpacefilling` |
| Atom/bond property on structure | `2DValuePlot` (e.g. Gasteiger charges) |
| Compare isomers or analogs | Side-by-side 2D plots (≤3) + short name/SMILES list |
| Element focus | Curated Element properties; structure only if a simple allotrope/molecule helps |
| Class survey | Up to 20 names from chemical class; plot 1–2 exemplars, not all |
| Physical appearance of element / compound / reaction product | Real photo of the substance, or labeled AI reconstruction — **not** a fake structure diagram |
| Same structure, larger image | Larger `ImageSize` after a successful plot |

## Structures vs illustrations

| Need | Tool |
|------|------|
| Bonds, atoms, functional groups, stereochemistry | **Wolfram** `MoleculePlot` / `MoleculePlot3D` |
| “What does sulfur look like?” / “copper sulfate crystals” | Photo of the substance or **AI reconstruction** (labeled) |
| “Show the caffeine molecule” | **Never** AI-draw the structure if Wolfram can plot it |

**MUST NOT** use AI image generation as a substitute for a computed molecule plot when the learner needs a reliable structure.

## Illustrating notables

When a teaching turn includes a **notable** reaction or related molecule/compound, illustrate the *physical substance* when it helps. This is separate from structure plots.

**Who does the work:** heavy image find/generate **SHOULD** be done by a **subagent**. The parent briefs the substance (name + preferred physical form) and synthesizes the returned image.

Worker paths:

1. **Find** a suitable real photograph of the briefed substance when one exists (sample, crystal, metal, powder, solution, clear product form).
2. **Generate** with AI when no good real photo exists — label **modern reconstruction** of the physical appearance.

Caption every image (identity, what the photo shows, how it ties to the notable chemistry). **MUST NOT** use a substance photo instead of a molecule plot for structural teaching.

**Do not** default to portraits, period lab scenes, or generic atmosphere shots — the image should show the element, molecule, or compound being discussed.

## AI image generation (non-structure)

### When to use

- Physical appearance of crystals, minerals, element samples, powders, solutions, or recognizable products when a real photo search fails (label reconstruction).

### When not to use

- **Not in place of Wolfram molecule plots.**
- Not as fake spectra, fake chromatograms, or unlabeled “scientific evidence.”
- Not as historical figures, period labs, or generic lab atmosphere when a substance photo is what was briefed.

### Prompt craft (physical appearance)

- Name the substance and form accurately (e.g. yellow sulfur powder, blue copper sulfate pentahydrate crystals, gray iron metal).
- Avoid wrong color, phase, or packaging that would mislead about the chemistry.
- Do not ask the model to “draw accurate bond-line structures” — use Wolfram for that.

## Web encyclopedia lookup

When you need a secondary encyclopedia / background web page (beyond Wolfram):

- Search and cite **[Grokipedia](https://grokipedia.com)** (`site:grokipedia.com` …).
- **Do not** use Wikipedia (en.wikipedia.org or mirrors) for background or secondary synthesis.
- Primary sources, Wolfram, PubChem pages, and museum/photo hosts remain fine; this rule is about encyclopedia search, not structure plots.

## Honesty

- Missing chemical entity → say so; try SMILES/synonym; do not invent.
- Property `Missing[]` → omit or note unavailable; do not fabricate numbers.
- 3D timeout → show 2D and say so.
- Isomer lists may be incomplete or slow — present what returned; do not invent isomers.
- Never invent Cloud/PNG URLs for plots.

## Safety (visual + content)

- Educational structures of common drugs/precursors may be discussed at textbook level; do not provide practical illicit synthesis guidance.
- For toxic/household chemicals, a brief honest hazard note is appropriate when relevant.
