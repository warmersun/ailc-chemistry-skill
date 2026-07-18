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
| Everyday use / history / lab scene | Real photo or labeled AI reconstruction — **not** a fake structure diagram |
| Same structure, larger image | Larger `ImageSize` after a successful plot |

## Structures vs illustrations

| Need | Tool |
|------|------|
| Bonds, atoms, functional groups, stereochemistry | **Wolfram** `MoleculePlot` / `MoleculePlot3D` |
| “What does a 19th-century lab look like?” | Photo or **AI reconstruction** (labeled) |
| “Show the caffeine molecule” | **Never** AI-draw the structure if Wolfram can plot it |

**MUST NOT** use AI image generation as a substitute for a computed molecule plot when the learner needs a reliable structure.

## Illustrating story elements

When a teaching turn includes a **story** element (discovery, everyday scene, famous chemist, industrial plant, …), illustrate the *scene* when it helps. This is separate from structure plots.

**Who does the work:** heavy image find/generate **SHOULD** be done by a **subagent**. The parent briefs the scene and synthesizes the returned image.

Worker paths:

1. **Find** a suitable real image when one exists (museum, lab glassware photo, historical portrait).
2. **Generate** with AI when no good real image exists — label **modern reconstruction**.

Caption every image (what / how it ties to the chemistry). **MUST NOT** use a story illustration instead of a molecule plot for structural teaching.

## AI image generation (non-structure)

### When to use

- Historical figures, lab atmosphere, industrial context, sensory everyday scenes (coffee cup + plant — not the molecule graph).
- Macro photos of crystals/minerals when a real photo search fails (label reconstruction).

### When not to use

- **Not in place of Wolfram molecule plots.**
- Not as fake spectra, fake chromatograms, or unlabeled “scientific evidence.”

### Prompt craft (chemistry-adjacent scenes)

- State era/setting accurately if historical.
- Avoid wrong glassware, modern PPE in period scenes, or impossible lab setups unless the goal is humor and labeled as such.
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
