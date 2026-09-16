# 06 · Substructure matching

Substructure matching is a capability unique to chemistry: you describe "what kind of structure I'm looking for" with a pattern, then search across an entire compound library. Its implementation relies on SMARTS, a pattern language, and it is also the foundation for all structure filtering and alert-structure screening.

## The logical engine of molecular recognition

### 06.1 From SMILES to SMARTS

SMILES describes one specific molecule. But if you want to ask "how many molecules in my library contain a carboxyl group?", you need a language that can describe a whole *class* of molecules. That is what SMARTS (SMILES Arbitrary Target Specification) is for.

The relationship between SMILES and SMARTS ≈ the relationship between a concrete string and a regular expression:

```
SMILES "CC(=O)O"      → "contains the acetic acid structure" (limited to acetic acid itself)
SMARTS "C(=O)[OH]"    → "contains a carboxyl fragment" (matches all carboxylic acids)
SMARTS "[CX4]"        → "contains an sp3 carbon" (logical expression)
SMARTS "[F,Cl,Br,I]"  → "contains a halogen" (logical OR)
SMARTS "a"            → "contains an aromatic atom" (wildcard)
```

SMARTS atomic primitives include:

- Element symbols: `C`, `N`, `O`, `[Na]`, `[Cl]`
- Wildcards: `*` (any atom), `a` (aromatic atom), `A` (aliphatic atom)
- Logical expressions: `,` (OR), `;` (AND), `!` (NOT)
- Degree specification: `D2` (atom connected to two heavy atoms), `X4` (total connectivity = 4)
- Hydrogen count: `H2` (bonded to 2 hydrogens), `h1` (implicit hydrogen count = 1)
- Charge: `+` (positive), `-` (negative)

### 06.2 The chemical meaning of subgraph isomorphism

In computer science, substructure matching corresponds to the **subgraph isomorphism problem**, which is NP-complete. But on chemical molecular graphs, because of their special properties (each atom's degree is bounded by valence, usually ≤ 6), practical algorithms run extremely fast on drug molecules.

The semantics of `mol.HasSubstructMatch(query)`:

- Each atom in the query must match a distinct atom in `mol`
- Each bond in the query must match a bond between the corresponding atoms in `mol`
- Matching rule: properties not specified in the query do not constrain the target (e.g. if the query does not specify aromaticity, the target may be aromatic or aliphatic)

**Key trap:** When the SMILES string `"CO"` is used as a SMARTS query, its meaning is "there exists a C atom and an O atom connected by a single bond" — it does not require the valence of C to match fully, nor that this is the complete fragment in the molecule. This explains why `"CO"` as SMARTS can match `CCO` (ethanol): ethanol does contain a C–O single bond.

### 06.3 MCS — Maximum Common Substructure

This is a core operation in retrosynthesis analysis and scaffold hopping: given two active molecules, find the largest substructure they share. This suggests that the common scaffold may be the core of the pharmacophore.

The parameter choices in `rdFMCS.FindMCS()` reflect chemical judgment:

- `ringMatchesRingOnly=True`: ring atoms can only match ring atoms. Why? An aromatic ring and an aliphatic chain, even with the same number of carbons, behave completely differently chemically.
- `completeRingsOnly=True`: only return complete rings. Why? A half-cut aromatic ring has no chemical meaning and cannot represent a real substructure.
- `timeout` parameter: in the worst case MCS has exponential time complexity and may never finish on a large molecule. The timeout is a necessary engineering safeguard.
