# 10 · Conformers and 3D

A molecule is not a flat diagram — it constantly changes shape in three-dimensional space, and that shape often determines activity. This chapter explains why conformational space matters, why "one molecule = one conformer" is a bad assumption, and then gives the full workflow for bulk generation, optimization, and clustering of representative conformers.

## The "fourth dimension" of molecules

### 10.1 Why 2D is not enough

The core of Part 1 of this handbook is 2D fingerprints — the topological encoding of molecular graphs. But for many drug-discovery tasks, molecules with the same topology yet different 3D conformations can have very different biological activity.

Consider this fact: a drug molecule with five rotatable bonds can theoretically exist in about 3^5 = 243 meaningful conformations. If each conformation binds the target protein with a different energy, then simply computing a 3D descriptor from a random conformer can introduce an error of up to 2–3 kcal/mol — unacceptable in binding-energy prediction.

### 10.2 Distance geometry — the mathematical basis of conformer generation

RDKit's conformer generation is based on the **distance geometry** (DG) theory. Its core idea is:

> Rather than computing 3D coordinates directly, first compute upper and lower bounds on interatomic distances, then randomly sample within this constrained space.

**Steps:**

1. **Build the distance-bounds matrix:** for each atom pair (i, j), determine the acceptable distance range [d_min, d_max] from bond connectivity and empirical rules
2. **Triangle smoothing:** use the triangle inequality (d_ik ≤ d_ij + d_jk) to tighten the bounds — this is the core algorithm of DG
3. **Random sampling:** under the bounds constraints, randomly generate a distance matrix
4. **Embedding:** convert the distance matrix into 3D coordinates (via eigendecomposition)
5. **Rough optimization:** make small adjustments to the coordinates with a simplified force field

**The philosophical significance of the DG method:** it does not try to find a "global optimum" conformer, but generates "plausible" conformers. This is a faithful treatment of the nature of the conformational ensemble — a molecule in solution is not frozen at its energy minimum, but constantly interconverts among all accessible conformations.

### 10.3 ETKDG — injecting empirical knowledge

The pure DG method has a problem: the distance bounds it generates are based on topological connectivity and atomic radii, lacking chemical specificity. It does not know that aromatic rings should be planar, nor that certain torsion angles tend toward specific values in experimental data.

**ETKDG** (Experimental Torsion Knowledge Distance Geometry) adds two layers of constraints on top of DG:

1. **Torsion-angle preferences from the small-molecule crystal structure database (CSD):** apply a probabilistic bias to the torsion-angle distributions of common bond types. For example, the O=C-O-C torsion in esters shows a strong 0° (cis) preference in the CSD — ETKDG preferentially samples that region.
2. **Chemical-knowledge constraints (the K part):** enforce basic chemical rules such as aromatic-ring planarity and triple-bond linearity.

ETKDGv3 (the current default version) further improves handling of charged molecules and macrocycles.

**Why this matters:** DG conformers without ETKDG may be chemically "correct" (no atom-distance conflicts) yet shape-wise "wrong" (e.g., an aromatic ring distorted into a chair conformation). ETKDG conformers are closer to experimental crystal structures.

### 10.4 The MMFF94 force field — refinement at the energy level

MMFF94 (Merck Molecular Force Field) is a "Class 2" force field, with energy terms for bond stretching, angle bending, torsion, and electrostatic/van der Waals non-bonded interactions.

```math
E_total = E_bond + E_angle + E_torsion + E_vdw + E_electrostatic
```

The point of running MMFF94 optimization after ETKDG:

- ETKDG produces "geometrically reasonable" conformers
- MMFF94 pushes them toward "energetically more favorable" conformers (local energy minima)

**Engineering judgment:** for most virtual-screening tasks, ETKDG conformers are already good enough. MMFF94 optimization is mainly necessary in two situations: (1) you need an accurate relative-energy ranking (e.g., ligand preparation before docking); (2) the molecule contains unusual valences or coordination modes where ETKDG's rules may not apply.

### 10.5 The representativeness problem in a conformational ensemble

A molecule has many conformers, but we usually need to assign one "representative" feature vector per molecule for an ML model. Common strategies:

1. **Global minimum-energy conformer:** pick the lowest-energy one. Problem — the minimum-energy conformer is not necessarily the "bioactive conformation" when bound to the protein
2. **Cluster centers:** use Butina clustering to group conformers and pick a representative from each cluster. This is the more reliable method because it preserves structural diversity
3. **Ensemble average:** compute descriptors over several low-energy conformers, then take the mean or weighted mean. Most informative, but the highest computational cost
4. **All conformers as independent samples:** used during training. Increases the data volume, but introduces correlation among multiple samples of the same molecule

**Practical recommendation:** if the downstream task is molecular-property prediction (LogP, solubility, etc.), the minimum-energy conformer is usually enough — because these properties are determined mainly by topology and are insensitive to conformation. If the task involves target binding (docking, pharmacophore matching), you need cluster centers or — conservatively — several low-energy conformers.

### 10.6 RMSD as a conformer similarity measure

RMSD (Root Mean Square Deviation) is the standard measure of structural difference between two conformers:

```
RMSD(A, B) = sqrt( Σ_i |pos_i^A - pos_i^B|² / n )
```

where pos_i^A is the 3D coordinate of atom i in molecule A (after alignment).

**Limitations of RMSD:**

- sensitive to overall translation/rotation — must be aligned first
- overly sensitive to overall shape changes in large molecules — two long-chain molecules differing only at the ends can yield a large RMSD despite an identical core
- complicated handling of molecular symmetry — rotating a benzene ring by 180° is chemically equivalent, yet RMSD reports a difference

In practice, 0.5 Å is the common threshold for distinguishing "same conformer" from "different conformer." For conformer clustering in virtual screening, 0.75–1.0 Å is more practical.
