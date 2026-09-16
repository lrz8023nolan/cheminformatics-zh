# 11 · Scaffolds and R-groups

When medicinal chemists discuss molecules, they talk about "swap this scaffold for that substituent — how does activity change?" Decomposing a molecule into scaffold and R-groups turns that discussion into a computable operation — and is also the basis for automatically extracting structure–activity relationships from compound libraries.

## The "backbone" concept of medicinal chemistry

### 11.1 The theoretical basis of the Murcko scaffold

In their 1996 *Journal of Medicinal Chemistry* paper, Bemis and Murcko proposed a far-reaching framework: systematically decompose a drug molecule into "scaffold + side chains."

**Decomposition rules:**

1. remove all non-ring, single-bonded side-chain atoms that are not connecting rings
2. keep all ring systems and the atoms connecting different rings (linkers)
3. what remains is the molecule's "framework"

```
Aspirin: CC(=O)OC1=CC=CC=C1C(=O)O
Murcko scaffold: c1ccccc1    (benzene ring—the only ring system)
```

**Why this concept matters:** the scaffold represents the molecule's "irreducible core" — if two drug molecules share the same scaffold, they very likely bind the same target family. Scaffold hopping — finding molecules with different scaffolds but the same activity — is one of the core strategies of modern drug discovery.

### 11.2 The use of the generic scaffold

A generic scaffold replaces all atoms with carbon and all bonds with single bonds. What results is a "topological map of molecular shape" that ignores heteroatoms and bond orders.

**Application scenario:** chemical-series clustering — identifying groups of molecules with the same "scaffold topology," even when their heteroatoms and functional groups differ.

### 11.3 R-group decomposition — the mathematical formalization of SAR

R-group decomposition systematically splits a set of molecules sharing a common scaffold into a "core + substituent table at each position."

**Mathematically, this is a graph-matching problem:**

- find the subgraph isomorphism of the core scaffold in each molecule
- label the matched atoms
- collect the unmatched subgraphs at each position (attachment point) as R-groups

**Output format** — using the benzene ring as the core, with benzene derivatives as the example:

| Molecule | R1 | R2 | R3 |
|----------|-----|-----|-----|
| Fluorobenzene | F | H | H |
| m-Fluorotoluene | F | CH₃ | H |
| m-Fluorochlorobenzene | F | Cl | H |

This table is the starting point for structure–activity relationship (SAR) analysis: you can directly compare "how much does activity change when R1 goes from F to Cl."

### 11.4 MMPA — the purest unit of SAR analysis

Matched Molecular Pair Analysis (MMPA) finds pairs of molecules that differ in only one structural transformation.

**Definition of an MMP:** for two molecules A and B, there exists a common substructure M such that A = M ∪ g1 and B = M ∪ g2 (g1 ≠ g2).

**Chemical meaning:** if A and B differ in activity, that difference can be attributed only to the g1 → g2 transformation. This is the cleanest structure–activity signal.

**The practical value of MMPA:** synthetic chemists often ask "if I replace -H with -CH₃ at this position, will activity change?" — MMPA can answer such questions systematically using existing data in a database.
