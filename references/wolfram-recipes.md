# Wolfram Language recipes for AILC Chemistry

Evaluate with the Wolfram Language evaluator (`wolfram__WolframLanguageEvaluator`). Always resolve chemical/element names with `Interpreter["Chemical"|"Element"]` or `\[FreeformPrompt]` — never invent `Entity["Chemical", "…"]` canonical names by hand.

When a plot is returned as a markdown image link, paste that link into the user-facing reply.

These recipes mirror the **Chemistry** chapter of `AILCBioChem.nb` (`moleculePlot`, `getEntityPropertyList`, `getEntityValue`, `getListOfChemicals`, `getIsomers`). Do **not** use Biology `anatomyPlot` APIs here.

**Typical order:** resolve molecule → 2D structure plot (optional highlights) → curated properties → isomers/class peers if useful.

---

## WL evaluator rules

- Prefer returning the **plot expression itself** (`MoleculePlot`, `MoleculePlot3D`, …) so results can surface as markdown image links.
- On name failure: re-resolve with `Interpreter["Chemical"]` → `\[FreeformPrompt]` → try SMILES/InChI if the learner has one → synonym.
- Raise `timeConstraint` (e.g. 90–120) for 3D plots, large molecules, or isomer searches.
- 3D plots can be slow; fall back to 2D HeavyAtom+Aromatic if `TimeConstrained` fails (30s default in the original plugin).
- Prefer `ImageSize -> Large` (or larger) for readable structures.

---

## Shared helpers

Paste at the top of a multi-step evaluation when useful.

```wl
(* Chemical entity from free text *)
toChemical[s_String] := Module[{c},
  c = Quiet @ Interpreter["Chemical"][s];
  If[FailureQ[c] || MissingQ[c],
    \[FreeformPrompt][s, Entity],
    c
  ]
]

(* Element entity from free text *)
toElement[s_String] := Module[{e},
  e = Quiet @ Interpreter["Element"][s];
  If[FailureQ[e] || MissingQ[e],
    \[FreeformPrompt][s, Entity],
    e
  ]
]

(* Build a Molecule from the same type tags as moleculePlot in AILCBioChem.nb *)
toMolecule[type_String, mol_String] := Quiet @ Molecule @ Switch[type,
  "ChemicalName", Interpreter["Chemical"][mol],
  "SMILES" | "InChI", mol,
  "DNA", BioSequence["DNA", mol],
  "RNA", BioSequence["RNA", mol],
  "Peptide", BioSequence["Peptide", mol],
  "PubChemCompoundID", ExternalIdentifier["PubChemCompoundID", mol],
  "PubChemSubstanceID", ExternalIdentifier["PubChemSubstanceID", mol],
  "ChEMBLID", ExternalIdentifier["ChEMBLID", mol],
  _, Interpreter["Chemical"][mol]
]
```

### Input `type` values (from plugin `moleculePlot`)

| type | Meaning |
|------|---------|
| `ChemicalName` | Common / IUPAC-ish name via `Interpreter["Chemical"]` (default) |
| `SMILES` | SMILES string |
| `InChI` | InChI string |
| `PubChemCompoundID` | PubChem CID |
| `PubChemSubstanceID` | PubChem SID |
| `ChEMBLID` | ChEMBL id |
| `DNA` / `RNA` / `Peptide` | BioSequence strings (molecular view only — not anatomy) |

### `plotTheme` values

| Theme | Renders with |
|-------|----------------|
| `2D` | `MoleculePlot`, HeavyAtom or AllAtom |
| `2DAromatic` (**default**) | `MoleculePlot` with Aromatic theme |
| `3DBallAndStick` | `MoleculePlot3D` BallAndStick |
| `3DSpacefilling` | Spacefilling |
| `3DTubes` | Tubes |
| `3DWireframe` | Wireframe |
| `2DValuePlot` | `ResourceFunction["MoleculeValuePlot"]` + `valuePlotProperty` |
| `BioSequence` | `BioSequencePlot` for DNA/RNA/Peptide types only |

### `viewPoint3D`

`Automatic` | `Above` | `Below` | `Front` | `Back` | `Left` | `Right`

### `valuePlotProperty` (for `2DValuePlot`)

Atom: `FormalCharge`, `GasteigerPartialCharge`, `HydrogenCount`, `ImplicitHydrogenCount`, `ImplicitValence`, `OrbitalHybridization`, `OuterShellElectronCount`, `UnpairedElectronCount`, `Valence`  
Bond: `BondLength`, `BondType`  
Element: `AtomicRadius`, `ValenceElectronCount`, `VanDerWaalsRadius`

---

## Molecule plot (core — mirrors `moleculePlot`)

### Basic 2D aromatic (default teaching view)

```wl
mol = toMolecule["ChemicalName", "caffeine"];
If[MoleculeQ[mol],
  MoleculePlot[mol,
    PlotTheme -> {"HeavyAtom", "Aromatic"},
    PlotLabel -> Replace[MoleculeName[mol], _?FailureQ -> "caffeine"],
    ImageSize -> Large
  ],
  Failure["InterpretationFailure", <|"MessageTemplate" -> "Could not interpret molecule"|>]
]
```

### 2D with explicit functional-group highlights

Highlight keys should match known pattern names when possible (see **Recognized highlight keys** below). Otherwise pass SMARTS to `MoleculePattern`.

```wl
mol = toMolecule["ChemicalName", "D-ribose"];
(* Example: highlight carbonyl-like pattern via SMARTS if named patterns unavailable *)
highlight = <|
  "carbonyl" -> MoleculePattern["[CX3]=[OX1]"]
|>;
MoleculePlot[mol, highlight,
  PlotTheme -> {"HeavyAtom", "Aromatic"},
  PlotLegends -> Automatic,
  ImageSize -> Large
]
```

### Auto-detect functional groups present (keys only)

When the full plugin pattern library is **not** loaded, detect with a small SMARTS dictionary (extend as needed):

```wl
mol = toMolecule["ChemicalName", "caffeine"];
fgSMARTS = <|
  "hydroxyl" -> "[OX2H]",
  "carbonyl" -> "[CX3]=[OX1]",
  "carboxyl" -> "[CX3](=O)[OX2H1]",
  "ether" -> "[OD2]([#6])[#6]",
  "primaryAmine" -> "[NX3;H2;!$(NC=O)]",
  "secondaryAmine" -> "[NX3;H1;!$(NC=O)]",
  "tertiaryAmine" -> "[NX3;H0;!$(NC=O);!$(N=O)]",
  "amide" -> "[NX3][CX3](=[OX1])",
  "nitro" -> "[$([NX3](=O)=O),$([NX3+](=O)[O-])]",
  "nitrile" -> "[NX1]#[CX2]",
  "thiol" -> "[SX2H]",
  "phenyl" -> "c1ccccc1",
  "alkene" -> "[CX3]=[CX3]",
  "alkyne" -> "[CX2]#[CX2]"
|>;
groupsFound = Keys @ Select[fgSMARTS, MoleculeContainsQ[mol, MoleculePattern[#]] &];
highlight = KeyTake[fgSMARTS, groupsFound] /. s_String :> MoleculePattern[s];
{
  groupsFound,
  MoleculePlot[mol, highlight,
    PlotTheme -> {"HeavyAtom", "Aromatic"},
    PlotLegends -> Automatic,
    ImageSize -> Large
  ]
}
```

### 3D ball-and-stick

```wl
mol = toMolecule["ChemicalName", "water"];
TimeConstrained[
  MoleculePlot3D[mol,
    PlotTheme -> "BallAndStick",
    IncludeHydrogens -> True,
    ViewPoint -> {1.3, -2.4, 2.},
    ImageSize -> Large
  ],
  30,
  MoleculePlot[mol, PlotTheme -> {"HeavyAtom", "Aromatic"}, ImageSize -> Large]
]
```

### From SMILES

```wl
mol = Molecule["CC(=O)Oc1ccccc1C(=O)O"]; (* aspirin *)
MoleculePlot[mol, PlotTheme -> {"HeavyAtom", "Aromatic"}, ImageSize -> Large]
```

### 2D value plot (partial charges, etc.)

```wl
mol = toMolecule["ChemicalName", "acetic acid"];
ResourceFunction["MoleculeValuePlot"][mol, "GasteigerPartialCharge"]
```

---

## Recognized highlight keys (plugin library)

From `functionalGroupMoleculePatterns` / bonds / amino acids in `AILCBioChem.nb`. Use these names when the host has the plugin patterns loaded; otherwise use SMARTS as above.

**Functional groups (sample):**  
`carbonyl`, `alkenyl`, `alkynyl`, `phenyl`, `halo`, `fluoro`, `chloro`, `bromo`, `iodo`, `hydroxyl`, `ketone`, `aldehyde`, `haloformyl`, `carbonateEster`, `carboxylate`, `carboxyl`, `carboalkoxy`, `hydroperoxy`, `peroxy`, `ether`, `hemiacetal`, `hemiketal`, `acetal`, `ketal`, `orthoester`, `methylenedioxy`, `orthoCarbonateEster`, `carboxylicAnhydride`, `carboxamide`, `amidine`, `primaryAmine`, `secondaryAmine`, `tertiaryAmine`, `ammoniumIon`, `hydrazone`, `primaryKetimine`, `secondaryKetimine`, `primaryAldimine`, `secondaryAldimine`, `imide`, `azide`, `azo`, `cyanate`, `isocyanate`, `nitrate`, `nitrile`, `isonitrile`, `nitrosooxy`, `nitro`, `nitroso`, `aldoxime`, `ketoxime`, `pyridyl2`, `pyridyl3`, `pyridyl4`, `carbamate`, `sulfhydryl`, `sulfide`, `disulfide`, `sulfinyl`, `sulfonyl`, `sulfino`, `sulfonicAcid`, `sulfonate`, `thiocyanate`, `isothiocyanate`, `carbonothioylThione`, `carbonothioylThial`, `carbothioicSacid`, `carbothioicOacid`, `thiolester`, `thionoester`, `carbodithioicAcid`, `carbodithio`, `phosphino`, `phosphono`, `phosphate`, `phosphatePhosphodiester`, `borono`, `boronate`, `borino`, `borinate`, `alkyllithium`, `alkylmagnesiumHalide`, `alkylaluminium`, `silylEther`

**Bonds:** `peptide`, `ester`, `ether` (bond patterns)

**Amino acids (1-letter):** `A C D E F G H I K L M N O P Q R S T U V W Y`

---

## Properties (mirrors `getEntityPropertyList` / `getEntityValue`)

### List non-graphical properties of a chemical or element

```wl
e = toChemical["water"]; (* or toElement["carbon"] *)
assoc = EntityValue[e, "NonMissingPropertyAssociation"];
Select[Keys[assoc],
  FreeQ[assoc[#], Image | Graphics | GraphicsComplex | Graphics3D] &&
  !MatchQ[assoc[#], _MeshRegion | _Image] &
]
```

### Fetch a few properties (curate — max ~3–5 for teaching)

```wl
e = toChemical["caffeine"];
EntityValue[e,
  {"MolecularFormula", "MolarMass", "MeltingPoint", "BoilingPoint", "Density"},
  "NonMissingPropertyAssociation"
]
```

Always prefer a **short curated list** for the learner. Full property lists are for the agent’s internal choice, not dumps.

---

## List chemicals in a class (mirrors `getListOfChemicals`)

```wl
class = "AminoAcids"; (* must be a member of EntityClassList["Chemical"] *)
classlist = Last /@ EntityClassList["Chemical"];
If[MemberQ[classlist, class],
  EntityValue[
    Take[EntityList[EntityClass["Chemical", class]], UpTo[20]],
    {"Name", "SMILES"},
    "PropertyAssociation"
  ],
  Failure["InterpretationFailure",
    <|"MessageTemplate" -> "Unknown chemical class", "MessageParameters" -> <|"class" -> class|>|>]
]
```

Useful class names vary by Knowledgebase version — if unsure, evaluate `Last /@ EntityClassList["Chemical"]` once and pick a nearby class.

---

## Isomers (mirrors `getIsomers`)

```wl
mol = Molecule["CCO"]; (* ethanol — will list structural isomers of C2H6O etc. per FindIsomers *)
If[MoleculeQ[mol],
  isomers = FindIsomers[mol];
  MoleculeValue[isomers, {"Name", "SMILES"}, "PropertyAssociation"],
  Failure["InterpretationFailure", <|"MessageTemplate" -> "Not a molecule"|>]
]
```

Optional: plot the first few isomers in a row (keep ≤3 for teaching).

```wl
mol = Molecule["CCO"];
isomers = Take[FindIsomers[mol], UpTo[3]];
MoleculePlot[#, PlotTheme -> {"HeavyAtom", "Aromatic"}, ImageSize -> Medium] & /@ isomers
```

---

## Element quick facts

```wl
el = toElement["iron"];
EntityValue[el,
  {"AtomicNumber", "AtomicMass", "ElectronConfigurationString", "MeltingPoint", "BoilingPoint", "Density"},
  "NonMissingPropertyAssociation"
]
```

---

## Wolfram|Alpha fallback

When structured entities fail but a quick fact is enough:

- Use `wolfram__WolframAlpha` with queries like `"caffeine melting point"`, `"structure of ibuprofen"`.
- Prefer `WolframLanguageEvaluator` + recipes above for **plots you control** (highlights, themes).

Always call `wolfram__WolframContext` when the topic is new so chemistry entity coverage is current.

---

## Entity types

| Type | Use for |
|------|---------|
| `Chemical` | Compounds, named molecules, many materials |
| `Element` | Periodic-table elements |
| `Molecule` (expression) | Structure ops: plot, substructure, isomers |
| `ExternalIdentifier` | PubChem / ChEMBL ids |

---

## Failure patterns

| Symptom | What to try |
|---------|-------------|
| Name not interpreted | Synonym, formula, SMILES, PubChem CID |
| Empty plot / not `MoleculeQ` | Wrong type tag; try `ChemicalName` vs `SMILES` |
| 3D timeout | Fall back to 2D aromatic; mention it |
| `FindIsomers` empty/slow | Smaller formula; raise timeConstraint; teach with known isomer pairs instead |
| Property `Missing` | Drop it; pick another; say data unavailable |

---

## Plugin parity note

The cloud `APIFunction`s in `AILCBioChem.nb` return JSON with `image` URL, `functionGroups`, `moleculeName`, and assistant notes. Under Wolfram MCP you achieve the same **teaching outcome** by:

1. Returning `MoleculePlot` / `MoleculePlot3D` for the image link  
2. Computing `functionGroups` / properties as separate expressions  
3. Putting captions and group names in the worker bullet list for the parent to weave in  

Do not invent Cloud object URLs.
