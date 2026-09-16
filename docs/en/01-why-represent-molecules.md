# 01 · Why represent molecules at all

## From chemical intuition to machine language

### 01.1 The core problem: computers don't recognize molecules

Show a chemist the structure of a benzene ring and they know instantly what it is. But a computer only understands numbers. So the first — and most fundamental — question in cheminformatics is:

> How do you describe a molecule completely, uniquely, and compactly in a form a computer can process?

It sounds simple, but it is full of traps. An ethanol molecule has at least six "drawings":

```
CCO     C-O-C WRONG!    OCC      C(O)C     [CH3][CH2][OH]      ethanol (English name)
```

The second one is wrong, but the other four strings describe the same molecule. If the computer treats `CCO` and `OCC` as two different things, the same molecule gets registered multiple times in a database, and your machine-learning model receives contradictory labels — that is where the trouble starts.

### 01.2 Linear notation: the design philosophy of SMILES

SMILES (Simplified Molecular Input Line Entry System) was proposed by David Weininger in the 1980s. Its core design trade-off is:

| Trade-off dimension | SMILES's choice | Why |
|----------|--------------|--------|
| Readability vs. machine optimization | Leans human-readable | Chemists need to be able to write and read it by hand |
| Flexibility vs. uniqueness | Allows multiple writings of the same molecule | Makes human input convenient (start from any atom) |
| Compactness vs. completeness | Uses implicit hydrogens, omits single bonds | Reduces storage (a typical molecule is only ~1.6 bytes/atom) |

**Graph traversal is the core mechanism.** The essence of SMILES is a depth-first search over the molecular graph, encoding the visited path as a string. So starting from ethanol:

- Start traversal at C1 → `CCO`
- Start traversal at O → `OCC`

Different starting points yield different SMILES, but they all point to the same molecular graph.

### 01.3 Canonical SMILES

If you could mandate "always start from this atom, traverse in this direction," you would guarantee the same molecule always produces the same string. That is the heart of canonicalization.

Weininger's CANGEN algorithm has two steps:
1. **CANON step:** assign each atom a unique "canonical label" based on topological invariants of its atomic properties (atomic number, degree, charge, and so on).
2. **GENES step:** starting from the atom with the smallest canonical label, traverse the graph by fixed rules to generate the string.

**But that does not mean "all problems are solved."** Different software (RDKit, OpenBabel, CDK) uses different canonicalization algorithms, so the same molecule can yield different canonical SMILES in different tools. InChI (IUPAC International Chemical Identifier) was designed precisely to solve this cross-tool consistency problem.

### 01.4 Beyond SMILES: a hierarchy of representations

```
1D representation (scalar descriptors):  MW=46.07, LogP=-0.18
    ↑ most information loss, but simplest
2D representation (topology graph + fingerprints):  CCO, molecular graph, ECFP4 fingerprint
    ↑ the main arena of cheminformatics
3D representation (conformer + shape):  atomic coordinates (x,y,z), electrostatic potential grid
    ↑ richest information, but most computationally expensive
```

This part focuses on 2D representation: starting from SMILES, build the molecular graph, and extract fingerprints and descriptors.

---

## Where these principles inform your decisions

### 01.5 Key decision points in your pipeline

In real work you keep hitting the following decision points:

| Decision point | What you weigh | Cost of getting it wrong |
|--------|-------------|-----------|
| Standardize the data? | Compute cost vs. data consistency | The model learns standardization artifacts instead of real chemical laws |
| Which fingerprint? | Information coverage vs. interpretability vs. compute speed | Miss key structure–activity relationships |
| Tanimoto or Dice? | Sensitivity to molecular size | Similarity ranking falls out of order |
| fpSize 1024 or 2048? | Collision probability vs. storage/compute cost | For large molecule libraries, the 1024-bit collision rate can exceed 20% |

### 01.6 Why understand the principles, not just memorize the operations

**No one tells you "what is right" — you have to define it yourself.**

No off-the-shelf standardization pipeline? You need to understand the chemical rationale behind each step (Chapter 03) so you can explain to collaborators that "this preprocessing is sound."

No fixed fingerprint choice? You need to understand what information each fingerprint captures (Chapter 04) so you can make an informed choice based on the task goal (QSAR? similarity search? virtual screening?).

No one tells you how to define "whether two molecules are the same"? You need to understand the principles of standardization and similarity (Chapters 05, 07) so you can set the rules for compound registration and deduplication.

This is why principles matter more than operations — operations can be copied, but judgment requires understanding.

---
