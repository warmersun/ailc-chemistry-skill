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

Highlight keys should match names in **Functional-group pattern constants** below (`functionalGroupMoleculePatterns`, bonds, amino acids). After those constants are evaluated:

```wl
mol = toMolecule["ChemicalName", "D-ribose"];
highlight = <|"hydroxyl" -> hydroxyl, "carbonyl" -> carbonyl|>;
(* equivalent: KeyTake[functionalGroupMoleculePatterns, {"hydroxyl", "carbonyl"}] *)
MoleculePlot[mol, highlight,
  PlotTheme -> {"HeavyAtom", "Aromatic"},
  PlotLegends -> Automatic,
  ImageSize -> Large
]
```

Without the library, pass SMARTS to `MoleculePattern` (see auto-detect recipe).

### Auto-detect functional groups present (keys only)

When the full plugin pattern library (**Functional-group pattern constants**) is loaded, prefer:

```wl
groupsFound = Keys @ Select[functionalGroupMoleculePatterns, MoleculeContainsQ[mol, #] &];
```

Otherwise detect with a small SMARTS dictionary (extend as needed):

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

## Functional-group pattern constants (plugin library)

Source: `Molecule Patterns` / `constants` sections in `AILCBioChem.nb`, `AILCChemistry.nb`, and `AILCChemistryFunctionGroups.nb`. Each group is a `MoleculePattern` built from `Atom` / element strings and `Bond` (order `"Single"` / `"Double"` / `"Triple"` / `"Aromatic"` where specified).

**How to use**

1. Evaluate the constant block below once in the Wolfram Language evaluator (or paste only the groups you need).
2. Highlight with association keys matching `functionalGroupMoleculePatterns` / `bondsMoleculePatterns` / `aminoAcidMoleculePatterns`.
3. Detect groups present: `Keys @ Select[functionalGroupMoleculePatterns, MoleculeContainsQ[mol, #] &]`.

```wl
mol = toMolecule["ChemicalName", "D-ribose"];
highlight = KeyTake[functionalGroupMoleculePatterns, {"hydroxyl", "aldehyde", "carbonyl"}];
(* or auto: *)
groupsFound = Keys @ Select[functionalGroupMoleculePatterns, MoleculeContainsQ[mol, #] &];
highlight = KeyTake[functionalGroupMoleculePatterns, groupsFound];
MoleculePlot[mol, highlight,
  PlotTheme -> {"HeavyAtom", "Aromatic"},
  PlotLegends -> Automatic,
  ImageSize -> Large
]
```

Alkyls (`methyl` … `alkyl20`) are helpers only — **excluded** from `functionalGroupMoleculePatterns` in the notebooks.

### Pattern constants

```wl
carbonyl = MoleculePattern[{Atom[_], "C", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}]}];

alkyl[n_Integer] := MoleculePattern[
Flatten[{Atom[_], Table[{"C", "H", "H"}, n], "H"}],
Flatten[{Bond[{1, 2}], Table[{Bond[{i, i+1}], Bond[{i, i+2}], Bond[{i, i+3}]}, {i, 2, 1+n*3, 3}]}]
]

alkane[n_Integer] := Molecule[
Flatten[{"H", Table[{"C", "H", "H"}, n], "H"}],
Flatten[{Bond[{1, 2}], Table[{Bond[{i, i+1}], Bond[{i, i+2}], Bond[{i, i+3}]}, {i, 2, 1+n*3, 3}]}]
]

(* alkyls will be excluded from functional groups *)
methyl = alkyl[1];
ethyl = alkyl[2];
propyl = alkyl[3];
butyl = alkyl[4];
pentyl = alkyl[5];
hexyl = alkyl[6];
heptyl = alkyl[7];
octyl = alkyl[8];
nonyl = alkyl[9];
decyl = alkyl[10];
undecyl = alkyl[11];
dodecyl = alkyl[12];
tridecyl = alkyl[13];
teradecyl = alkyl[14];
alkyl15 = alkyl[15];
alkyl16 = alkyl[16];
alkyl17 = alkyl[17];
alkyl18 = alkyl[18];
alkyl19 = alkyl[19];
alkyl20 = alkyl[20];

alkenyl = MoleculePattern[{Atom[_], Atom[_], Atom[_], Atom[_], "C", "C"}, {Bond[{5, 6}, "Double"], Bond[{5, 1}], Bond[{5, 2}], Bond[{6, 3}], Bond[{6, 4}]}];
alkynyl = MoleculePattern[{Atom[_], Atom[_], "C", "C"}, {Bond[{3, 4}, "Triple"], Bond[{3, 1}], Bond[{4, 2}]}];
phenyl = MoleculePattern[{Atom[_], "C", "C", "C", "C", "C", "C", "H", "H", "H", "H", "H"}, {Bond[{1, 2}], Bond[{2, 3}, "Aromatic"], Bond[{3, 4}], Bond[{4, 5}, "Aromatic"], Bond[{5, 6}], Bond[{6, 7}, "Aromatic"], Bond[{7, 2}], Bond[{3, 8}], Bond[{4, 9}], Bond[{5, 10}], Bond[{6, 11}], Bond[{7, 12}]}];

halo = MoleculePattern[{"C", Atom["F" | "Cl" | "Br" | "I"]}, {Bond[{1, 2}]}];

fluoro = MoleculePattern[{"C", "F"}, {Bond[{1, 2}]}];

chloro = MoleculePattern[{"C", "Cl"}, {Bond[{1, 2}]}];

bromo = MoleculePattern[{"C", "Br"}, {Bond[{1, 2}]}];

iodo = MoleculePattern[{"C", "I"}, {Bond[{1, 2}]}];

hydroxyl = MoleculePattern[{"O", "H", Atom[_]}, {Bond[{1, 2}], Bond[{1, 3}]}];

ketone = MoleculePattern[{"C", "O", Atom[_], Atom[_]}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{1, 4}]}];

aldehyde = MoleculePattern[{"C", "O", Atom[_], "H"}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{1, 4}]}];

haloformyl = MoleculePattern[{"C", "O", Atom[_], Atom["F" | "Cl" | "Br" | "I"]}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{1, 4}]}];

carbonateEster = MoleculePattern[{Atom[_], "O", "C", "O", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{3, 5}], Bond[{5, 6}]}];

carboxylate = MoleculePattern[{Atom[_], "C", "O", Atom["O", "FormalCharge" -> -1]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{2, 4}]}];

carboxyl = MoleculePattern[{Atom[_], "C", "O", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];
carboalkoxy = MoleculePattern[{Atom[_], "C", "O", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

hydroperoxy = MoleculePattern[{Atom[_], "O", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}]}];

peroxy = MoleculePattern[{Atom[_], "O", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}]}];

ether = MoleculePattern[{Atom[_], "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}]}];

hemiacetal = MoleculePattern[{Atom[_], "C", "H", "O", Atom[_], "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{2, 4}], Bond[{4, 5}], Bond[{2, 6}], Bond[{6, 7}]}];

hemiketal = MoleculePattern[{Atom[_], "C", Atom[_], "O", Atom[_], "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{2, 4}], Bond[{4, 5}], Bond[{2, 6}], Bond[{6, 7}]}];

acetal = MoleculePattern[{"C", "H", "O", Atom[_], "O", Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{1, 3}], Bond[{3, 4}], Bond[{1, 5}], Bond[{5, 6}], Bond[{1, 7}]}];

ketal = MoleculePattern[{Atom[_], "C", "O", Atom[_], "O", Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}], Bond[{2, 5}], Bond[{5, 6}], Bond[{2, 7}]}];

orthoester = MoleculePattern[{Atom[_], "C", "O", Atom[_], "O", Atom[_], "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}], Bond[{2, 5}], Bond[{5, 6}], Bond[{2, 7}], Bond[{7, 8}]}];

methylenedioxy = MoleculePattern[{"O", "C", "H", "H", "O"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{2, 4}], Bond[{2, 5}], Bond[{1, 5}]}];

orthoCarbonateEster = MoleculePattern[{"C", "O", Atom[_], "O", Atom[_], "O", Atom[_], "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{1, 4}], Bond[{4, 5}], Bond[{1, 6}], Bond[{6, 7}], Bond[{1, 8}], Bond[{8, 9}]}];

carboxylicAnhydride = MoleculePattern[{Atom[_], "C", "O", "O", "C", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}], Bond[{5, 6}, "Double"], Bond[{5, 7}]}];

carboxamide = MoleculePattern[{"C", "O", "N", Atom[_], Atom[_], Atom[_]}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{3, 4}], Bond[{3, 5}], Bond[{1, 6}]}];

amidine = MoleculePattern[{"C", "N", "N", Atom[_], Atom[_], Atom[_], Atom[_]}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{2, 4}], Bond[{3, 5}], Bond[{3, 6}], Bond[{1, 7}]}];

primaryAmine = MoleculePattern[{Atom[_], "N", "H", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{2, 4}]}];

secondaryAmine = MoleculePattern[{Atom[_], Atom[_], "N", "H"}, {Bond[{1, 3}], Bond[{2, 3}], Bond[{3, 4}]}];

tertiaryAmine = MoleculePattern[{Atom[_], Atom[_], Atom[_], "N"}, {Bond[{1, 4}], Bond[{2, 4}], Bond[{3, 4}]}];

ammoniumIon = MoleculePattern[{Atom[_], Atom[_], Atom[_], Atom[_], Atom["N", "FormalCharge" -> 1]}, {Bond[{1, 5}], Bond[{2, 5}], Bond[{3, 5}], Bond[{4, 5}]}];

hydrazone = MoleculePattern[{Atom[_], Atom[_], Atom["C"], Atom["N"], Atom["N"], Atom["H"], Atom["H"]}, {Bond[{1, 3}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{4, 5}], Bond[{5, 6}], Bond[{5, 7}]}];

primaryKetimine = MoleculePattern[{Atom[_], Atom["C"], Atom["N"], Atom["H"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{3, 4}], Bond[{2, 5}]}];

secondaryKetimine = MoleculePattern[{Atom[_], Atom["C"], Atom["N"], Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{3, 5}]}];

primaryAldimine = MoleculePattern[{Atom[_], Atom["C"], Atom["N"], Atom["H"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{3, 5}]}];

secondaryAldimine = MoleculePattern[{Atom["C"], Atom["N"], Atom["H"], Atom[_], Atom[_]}, {Bond[{1, 2}, "Double"], Bond[{1, 3}], Bond[{2, 4}], Bond[{1, 5}]}];

imide = MoleculePattern[{Atom["N"], Atom["C"], Atom["C"], Atom["O"], Atom["O"], Atom[_], Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{1, 3}], Bond[{2, 4}, "Double"], Bond[{3, 5}, "Double"], Bond[{1, 6}], Bond[{2, 7}], Bond[{3, 8}]}];

azide = MoleculePattern[{Atom["N"], Atom["N", "FormalCharge" -> 1], Atom["N", "FormalCharge" -> -1], Atom[_]}, {Bond[{1, 4}], Bond[{1, 2}, "Double"], Bond[{2, 3}]}];

azo = MoleculePattern[{Atom["N"], Atom["N"], Atom[_], Atom[_]}, {Bond[{1, 3}], Bond[{1, 2}, "Double"], Bond[{2, 4}]}];

cyanate = MoleculePattern[{Atom["O"], Atom["C"], Atom["N"], Atom[_]}, {Bond[{1, 4}], Bond[{1, 2}], Bond[{2, 3}, "Triple"]}];

isocyanate = MoleculePattern[{Atom["N"], Atom["C"], Atom["O"], Atom[_]}, {Bond[{1, 4}], Bond[{1, 2}, "Double"], Bond[{2, 3}, "Double"]}];

nitrate = MoleculePattern[{Atom[_], Atom["O"], Atom["N", "FormalCharge" -> 1], Atom["O"], Atom["O", "FormalCharge" -> -1]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{3, 5}]}];

nitrile = MoleculePattern[{Atom[_], Atom["C"], Atom["N"]}, {Bond[{1, 2}], Bond[{2, 3}, "Triple"]}];

isonitrile = MoleculePattern[{Atom[_], Atom["N", "FormalCharge" -> 1], Atom["C", "FormalCharge" -> -1]}, {Bond[{1, 2}], Bond[{2, 3}, "Triple"]}];

nitrosooxy = MoleculePattern[{Atom[_], Atom["O"], Atom["N"], Atom["O"]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"]}];

nitro = MoleculePattern[{Atom[_], Atom["N", "FormalCharge" -> 1], Atom["O"], Atom["O", "FormalCharge" -> -1]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}, "Single"]}];

nitroso = MoleculePattern[{Atom[_], Atom["N"], Atom["O"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"]}];

aldoxime = MoleculePattern[{Atom[_], Atom["C"], Atom["N"], Atom["O"], Atom["H"], Atom["H"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{3, 4}], Bond[{4, 5}], Bond[{2, 6}]}];

ketoxime = MoleculePattern[{Atom[_], Atom["C"], Atom["N"], Atom["O"], Atom["H"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{3, 4}], Bond[{4, 5}], Bond[{2, 6}]}];

pyridyl2 = MoleculePattern[
{Atom[_], "C", "N", "C", "C", "C", "C", "H", "H", "H", "H"},
{Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Aromatic"], Bond[{4, 5}], Bond[{5, 6}, "Aromatic"], Bond[{6, 7}], Bond[{7, 2}, "Aromatic"], Bond[{4, 8}], Bond[{5, 9}], Bond[{6, 10}], Bond[{7, 11}]}];

pyridyl3 = MoleculePattern[
{Atom[_], "C", "C", "N", "C", "C", "C", "H", "H", "H", "H"},
{Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Aromatic"], Bond[{4, 5}], Bond[{5, 6}, "Aromatic"], Bond[{6, 7}], Bond[{7, 2}, "Aromatic"], Bond[{3, 8}], Bond[{5, 9}], Bond[{6, 10}], Bond[{7, 11}]}];

pyridyl4 = MoleculePattern[
{Atom[_], "C", "C", "C", "N", "C", "C", "H", "H", "H", "H"},
{Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Aromatic"], Bond[{4, 5}], Bond[{5, 6}, "Aromatic"], Bond[{6, 7}], Bond[{7, 2}, "Aromatic"], Bond[{3, 8}], Bond[{4, 9}], Bond[{6, 10}], Bond[{7, 11}]}];

carbamate = MoleculePattern[{Atom[_], Atom["O"], Atom["C"], Atom["O"], Atom["N"], Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{3, 5}], Bond[{5, 6}], Bond[{5, 7}]}];

sulfhydryl = MoleculePattern[{Atom[_], Atom["S"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}]}];

sulfide = MoleculePattern[{Atom[_], Atom["S"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}]}];

disulfide = MoleculePattern[{Atom[_], Atom["S"], Atom["S"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}]}];

sulfinyl = MoleculePattern[{Atom[_], Atom["S"], Atom["O"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}]}];

sulfonyl = MoleculePattern[{Atom[_], Atom["S"], Atom["O"], Atom["O"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}, "Double"], Bond[{2, 5}]}];

sulfino = MoleculePattern[{Atom[_], Atom["S"], Atom["O"], Atom["O"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

sulfonicAcid = MoleculePattern[{Atom[_], Atom["S"], Atom["O"], Atom["O"], Atom["O"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}, "Double"], Bond[{2, 5}], Bond[{5, 6}]}];

sulfonate = MoleculePattern[{Atom[_], Atom["S"], Atom["O"], Atom["O"], Atom["O"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}, "Double"], Bond[{2, 5}], Bond[{5, 6}]}];

thiocyanate = MoleculePattern[{Atom[_], Atom["S"], Atom["C"], Atom["N"]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Triple"]}];

isothiocyanate = MoleculePattern[{Atom[_], Atom["N"], Atom["C"], Atom["S"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{3, 4}, "Double"]}];

carbonothioylThione = MoleculePattern[{Atom[_], Atom["C"], Atom["S"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}]}];

carbonothioylThial = MoleculePattern[{Atom[_], Atom["C"], Atom["S"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}]}];

carbothioicSacid = MoleculePattern[{Atom[_], Atom["C"], Atom["O"], Atom["S"], Atom["H"]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

carbothioicOacid = MoleculePattern[{Atom[_], "C", "S", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

thiolester = MoleculePattern[{Atom[_], Atom["C"], Atom["O"], Atom["S"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

thionoester = MoleculePattern[{Atom[_], Atom["C"], Atom["S"], Atom["O"], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

carbodithioicAcid = MoleculePattern[{Atom[_], "C", "S", "S", "H"}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

carbodithio = MoleculePattern[{Atom[_], "C", "S", "S", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

phosphino = MoleculePattern[{Atom[_], Atom[_], Atom[_], "P"}, {Bond[{1, 4}], Bond[{2, 4}], Bond[{3, 4}]}];

phosphono = MoleculePattern[{Atom[_], "P", "O", "O", "H", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}], Bond[{2, 6}], Bond[{6, 7}]}];

phosphate = MoleculePattern[{Atom[_], "O", "P", "O", "O", "H", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{3, 5}], Bond[{5, 6}], Bond[{3, 7}], Bond[{7, 8}]}];

phosphatePhosphodiester = MoleculePattern[{Atom[_], "O", "P", "O", "O", "H", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{3, 5}], Bond[{5, 6}], Bond[{3, 7}], Bond[{7, 8}]}];

borono = MoleculePattern[{Atom[_], "B", "O", "H", "O", "H"}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}], Bond[{2, 5}], Bond[{5, 6}]}];

boronate = MoleculePattern[{Atom[_], "B", "O", Atom[_], "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}], Bond[{2, 5}], Bond[{5, 6}]}];

borino = MoleculePattern[{Atom[_], Atom[_], "B", "O", "H"}, {Bond[{1, 3}], Bond[{2, 3}], Bond[{3, 4}], Bond[{4, 5}]}];

borinate = MoleculePattern[{Atom[_], Atom[_], "B", "O", Atom[_]}, {Bond[{1, 3}], Bond[{2, 3}], Bond[{3, 4}], Bond[{4, 5}]}];

alkyllithium = MoleculePattern[{Atom[_], Atom["Li"]}, {Bond[{1, 2}]}];

alkylmagnesiumHalide = MoleculePattern[{Atom[_], Atom["Mg"], Atom["Cl" | "Br" | "I"]}, {Bond[{1, 2}], Bond[{2, 3}]}];

alkylaluminium = MoleculePattern[{Atom[_], Atom[_], Atom[_], "Al", Atom[_], Atom[_], Atom[_], "Al"}, {Bond[{1, 4}], Bond[{2, 4}], Bond[{3, 4}], Bond[{4, 8}], Bond[{5, 8}], Bond[{6, 8}], Bond[{7, 8}]}];

silylEther = MoleculePattern[{Atom[_], Atom[_], Atom[_], Atom["Si"], Atom["O"], Atom[_]}, {Bond[{1, 4}], Bond[{2, 4}], Bond[{3, 4}], Bond[{4, 5}], Bond[{5, 6}]}];

peptideBond = MoleculePattern[{"H", "N", "C", "O", Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}], Bond[{3, 4}, "Double"], Bond[{2, 5}], Bond[{3, 6}]}];

esterBond = MoleculePattern[{Atom[_], "C", "O", "O", Atom[_]}, {Bond[{1, 2}], Bond[{2, 3}, "Double"], Bond[{2, 4}], Bond[{4, 5}]}];

etherBond = MoleculePattern[{Atom["O"], Atom[_], Atom[_]}, {Bond[{1, 2}], Bond[{1, 3}]}];

functionalGroupMoleculePatterns = <|
"carbonyl" -> carbonyl,
"alkenyl" -> alkenyl,
"alkynyl" -> alkynyl,
"phenyl" -> phenyl,
"halo" -> halo,
"fluoro" -> fluoro,
"chloro" -> chloro,
"bromo" -> bromo,
"iodo" -> iodo,
"hydroxyl" -> hydroxyl,
"ketone" -> ketone,
"aldehyde" -> aldehyde,
"haloformyl" -> haloformyl,
"carbonateEster" -> carbonateEster,
"carboxylate" -> carboxylate,
"carboxyl" -> carboxyl,
"carboalkoxy" -> carboalkoxy,
"hydroperoxy" -> hydroperoxy,
"peroxy" -> peroxy,
"ether" -> ether,
"hemiacetal" -> hemiacetal,
"hemiketal" -> hemiketal,
"acetal" -> acetal,
"ketal" -> ketal,
"orthoester" -> orthoester,
"methylenedioxy" -> methylenedioxy,
"orthoCarbonateEster" -> orthoCarbonateEster,
"carboxylicAnhydride" -> carboxylicAnhydride,
"carboxamide" -> carboxamide,
"amidine" -> amidine,
"primaryAmine" -> primaryAmine,
"secondaryAmine" -> secondaryAmine,
"tertiaryAmine" -> tertiaryAmine,
"ammoniumIon" -> ammoniumIon,
"hydrazone" -> hydrazone,
"primaryKetimine" -> primaryKetimine,
"secondaryKetimine" -> secondaryKetimine,
"primaryAldimine" -> primaryAldimine,
"secondaryAldimine" -> secondaryAldimine,
"imide" -> imide,
"azide" -> azide,
"azo" -> azo,
"cyanate" -> cyanate,
"isocyanate" -> isocyanate,
"nitrate" -> nitrate,
"nitrile" -> nitrile,
"isonitrile" -> isonitrile,
"nitrosooxy" -> nitrosooxy,
"nitro" -> nitro,
"nitroso" -> nitroso,
"aldoxime" -> aldoxime,
"ketoxime" -> ketoxime,
"pyridyl2" -> pyridyl2,
"pyridyl3" -> pyridyl3,
"pyridyl4" -> pyridyl4,
"carbamate" -> carbamate,
"sulfhydryl" -> sulfhydryl,
"sulfide" -> sulfide,
"disulfide" -> disulfide,
"sulfinyl" -> sulfinyl,
"sulfonyl" -> sulfonyl,
"sulfino" -> sulfino,
"sulfonicAcid" -> sulfonicAcid,
"sulfonate" -> sulfonate,
"thiocyanate" -> thiocyanate,
"isothiocyanate" -> isothiocyanate,
"carbonothioylThione" -> carbonothioylThione,
"carbonothioylThial" -> carbonothioylThial,
"carbothioicSacid" -> carbothioicSacid,
"carbothioicOacid" -> carbothioicOacid,
"thiolester" -> thiolester,
"thionoester" -> thionoester,
"carbodithioicAcid" -> carbodithioicAcid,
"carbodithio" -> carbodithio,
"phosphino" -> phosphino,
"phosphono" -> phosphono,
"phosphate" -> phosphate,
"phosphatePhosphodiester" -> phosphatePhosphodiester,
"borono" -> borono,
"boronate" -> boronate,
"borino" -> borino,
"borinate" -> borinate,
"alkyllithium" -> alkyllithium,
"alkylmagnesiumHalide" -> alkylmagnesiumHalide,
"alkylaluminium" -> alkylaluminium,
"silylEther" -> silylEther
|>;

bondsMoleculePatterns = <|
"peptide" -> peptideBond,
"ester" -> esterBond,
"ether" -> etherBond
|>;

aminoAcidMoleculePatterns = AssociationThread[
{"A", "C", "D", "E", "F", "G", "H", "I", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "Y"} -> Map[MoleculePattern, {
"[NX3][CX4H]([CH3X4])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][SX2])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CX3](=[OX1])[OX2H])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][CX3](=[OX1])[OX2H])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][cX3]1[cX3H][cX3H][cX3H][cX3H][cX3H]1)[CX3](=[OX1])",
"[NX3][CX4H2][CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][cX3]1[cX3H][nX3H][cX3H][nX2]1)[CX3](=[OX1])",
"[NX3][CX4H]([CHX4]([CH3X4])[CH2X4][CH3X4])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][CH2X4][CH2X4][NX3H2])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CHX4]([CH3X4])[CH3X4])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][SX2][CH3X4])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CX3](=[OX1])[NX3H2])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][CH2X4][CH2X4][NX3H][CX3](=[OX1])[CHX4]1[CHX4]([CH3X4])[CH2X4][CHX3]=[NX2]1)[CX3](=[OX1])",
"[NX3]1[CX4H2][CX4H2][CX4H2][CX4H]1[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][CX3](=[OX1])[NX3H2])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][CH2X4][CH2X4][NHX3][CH0X3](=[NX2H])[NH2X3])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][OX2H])[CX3](=[OX1])",
"[NX3][CX4H]([CHX4]([CH3X4])[OX2H])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][#34X2])[CX3](=[OX1])",
"[NX3][CX4H]([CHX4]([CH3X4])[CH3X4])[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][cX3]1[cX3H][nX3H][cX3]2[cX3H][cX3H][cX3H][cX3H][cX3]12)[CX3](=[OX1])",
"[NX3][CX4H]([CH2X4][cX3]1[cX3H][cX3H][cX3]([OX2H])[cX3H][cX3H]1)[CX3](=[OX1])"
}]
];

recognizedMoleculePatterns = Join[functionalGroupMoleculePatterns, bondsMoleculePatterns, aminoAcidMoleculePatterns];

peptideMoleculePatterns = Join[<|"peptide bond" -> peptideBond|>, aminoAcidMoleculePatterns];
```

### Association indexes (keys for highlights)

| Association | Keys |
|-------------|------|
| `functionalGroupMoleculePatterns` | 86 groups: `carbonyl`, `alkenyl`, `alkynyl`, `phenyl`, `halo`…`silylEther` (see `<|…|>` above) |
| `bondsMoleculePatterns` | `peptide`, `ester`, `ether` → `peptideBond`, `esterBond`, `etherBond` |
| `aminoAcidMoleculePatterns` | 1-letter: `A C D E F G H I K L M N O P Q R S T U V W Y` (SMARTS → `MoleculePattern`) |
| `recognizedMoleculePatterns` | `Join` of the three associations above |
| `peptideMoleculePatterns` | `<|"peptide bond" -> peptideBond|>` plus amino acids |

If the full library is too large to paste, fall back to the small SMARTS dictionary in **Auto-detect functional groups** above.

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
