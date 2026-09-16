# 13 · The complete pipeline

The previous chapters are individual capabilities; this chapter strings them into a line: starting from raw files, through standardization, deduplication, and feature computation, and finally producing data that can be fed straight into a model. The order cannot be jumbled — this is the piece of code most worth copying and adapting directly in the entire book.

## The "production-grade" mindset from data to model

### 13.1 Core principles of a pipeline

How a chemical-data pipeline differs from a general-purpose data pipeline:

1. **Data is not clean numbers — it is a collection of structural formulas.** The same molecule in raw data may appear in multiple forms (salts, tautomers, different SMILES notations)
2. **Features are not predefined — they must be "extracted" from structure.** The choice of descriptors and fingerprints directly affects downstream model behavior
3. **Deduplication is a chemistry problem, not a string problem.** `"CC(=O)O"` and `"[Na+].CC(=O)[O-]` are completely different at the string level, but are the same entity chemically

### 13.2 When to standardize

```
Raw SDF → standardization → deduplication → feature extraction → ML input
           ↑
        this step is the most error-prone
```

**Wrong approach:** deduplicate first, then standardize. This misses new duplicates created by standardization.

**Correct approach:** standardize first, then deduplicate (based on the canonical SMILES or InChIKey after standardization).

### 13.3 The design philosophy of PandasTools

`PandasTools` is not an "extra feature" of RDKit — it is the core infrastructure of chemical-data engineering. It lets you store molecular information at three different granularities in the same DataFrame:

1. **SMILES strings** (for storage and transfer)
2. **Mol objects** (for computation and analysis)
3. **Rendered images** (for inspection and communication)

The three are kept automatically in sync by `PandasTools`' methods. This design lets you switch seamlessly between "looking at the molecule" and "computing on the molecule" after a single data load.

### 13.4 Performance optimization: when to use Pickle, when to use SMILES

| Storage method | Load speed | File size | Use case |
|----------|---------|----------|----------|
| SMILES text | slow (must be re-parsed) | smallest | long-term archival, cross-tool transfer |
| Pickle (.pkl) | fast (direct deserialization) | larger | reused intermediate results |
| Binary (.ToBinary) | fastest | medium | fast in-memory transfer of large molecule sets |

**Practical rule:** for libraries above 10,000 molecules, pickle a copy immediately after the first load. Subsequent work loads from the pickle, with a 10–50× speedup.

### 13.5 Parallelizing conformer generation

The `numThreads` parameter of `EmbedMultipleConfs` and `MMFFOptimizeMoleculeConfs` is crucial in compute-intensive tasks. For 100 molecules each generating 50 conformers (5,000 conformer optimizations in total), a single thread may take tens of minutes; multithreading can cut that to a few minutes.

`numThreads=0` means "use all available cores" — the most resource-friendly default setting.

## Where these capabilities sit in the drug-discovery workflow

### 13.10 Chemical reaction handling → retrosynthesis and virtual library enumeration

Both retrosynthesis and reaction-condition prediction presuppose reaction-template authoring and the `RunReactants` mechanism.

### 13.11 Conformer generation → 3D QSAR and docking preparation

Although RDKit itself does not perform molecular docking, the 3D conformers it generates are the standard input format for most docking software (AutoDock Vina, Schrödinger Glide). The ability to bulk-generate and screen low-energy conformers is a key stage of the virtual-screening pipeline.

### 13.12 Scaffold and R-group decomposition → SAR analysis

In a drug-development setting, synthetic chemists continuously produce new compounds and their activity data. R-group decomposition and MMPA help you automatically extract structure–activity relationships from that data — letting its value extend beyond the level of individual compounds.

### 13.13 Pipeline engineering → reproducible research

In practice you often need to build your own pipeline. Good standardization, serialization, and parallelization practices are not "extra quality assurance" but the foundation of reproducible research — your results need to be verifiable by yourself and your team in the future.
