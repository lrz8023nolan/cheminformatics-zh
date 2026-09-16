# 05 · Molecular descriptors

A fingerprint answers "does this structure fragment exist", while a descriptor answers "what is the value". The two are complementary: fingerprints excel at distinguishing structures, while descriptors excel at capturing property trends. This chapter covers common physicochemical descriptors, bulk calculation, and a quick drug-likeness assessment using Lipinski's Rule of Five.

## Translating molecular properties into numbers

### 05.1 The difference between descriptors and fingerprints

| | Descriptor | Fingerprint |
|---|---|---|
| Output | Scalar (a single number) | Vector (hundreds to thousands of bits) |
| Dimensionality | 1 | High-dimensional |
| Interpretation | Direct ("molecular weight" has a clear physical meaning) | Indirect (requires back-tracing what each bit means) |
| Information content | Low (a single descriptor captures limited information) | High (the whole fingerprint encodes the full structure) |
| Use | Interpretable properties, filtering rules, physicochemical property prediction | Similarity search, QSAR, virtual screening |

### 05.2 The most important descriptors in medicinal chemistry

**LogP (partition coefficient)**

LogP = log([compound concentration in octanol] / [compound concentration in water])

- Low LogP (< 1): hydrophilic — good water solubility but weak cell-membrane penetration
- High LogP (> 5): lipophilic — good penetration but poor water solubility and fast metabolic clearance
- Ideal range (oral drugs): 1–3

RDKit uses Crippen's atomic contribution method: it treats LogP as the sum of contributions from each atom type. This is not an experimental value but an estimate — yet good enough as an approximation for high-throughput virtual screening.

**TPSA (topological polar surface area)**

TPSA is the sum of the surface areas of the polar atoms (O, N, and the H bonded to them) in the molecule. It predicts two key properties:

- Oral absorption: TPSA < 140 Å² usually means acceptable oral bioavailability
- Blood–brain barrier penetration: TPSA < 90 Å² usually means the compound can enter the central nervous system

TPSA needs no 3D conformer — it is computed directly from the 2D topology and a pre-calibrated atomic contribution table, so it is fast and conformation-independent.

**Hydrogen-bond donor/acceptor count (HBD / HBA)**

- HBD (hydrogen-bond donor): the number of O–H and N–H groups in the molecule that can donate a hydrogen
- HBA (hydrogen-bond acceptor): the number of O and N atoms in the molecule that can accept a hydrogen

Hydrogen bonds are a core driving force of drug–target binding, and also a key factor affecting solubility and permeability. The Lipinski thresholds of HBD ≤ 5 and HBA ≤ 10 come from statistical analysis of large oral-drug datasets.

**Rotatable bond count**

- Definition: the number of non-ring, non-terminal single bonds
- The more "flexible" a molecule is (the more rotatable bonds), the higher the entropy cost of binding to a target
- Rule of thumb: ≤ 10 falls within an acceptable range for drug molecules

### 05.3 The theoretical background of Lipinski's Rule of Five

In 1997 Lipinski analyzed 2245 drug candidates that had entered Phase II clinical trials, and found that most orally active drugs satisfy the following rules:

| Rule | Threshold | Theoretical basis |
|------|-----------|-------------------|
| MW ≤ 500 | Molecular weight | Molecules larger than 500 struggle to cross the cell membrane by passive diffusion |
| LogP ≤ 5 | Lipophilicity | Excessively high LogP leads to poor water solubility and fast metabolic clearance |
| HBD ≤ 5 | Hydrogen-bond donor | Too many HBDs make the desolvation energy too high |
| HBA ≤ 10 | Hydrogen-bond acceptor | Too many HBAs likewise raise the desolvation cost |

**Key point:** Lipinski's rule is a statistical pattern, not a physical law. Many successful drugs (such as cyclosporine and paclitaxel) violate several of its rules. The correct way to use it is as a quick screening filter, not a hard exclusion criterion. A molecule that violates 1–2 rules yet shows very high activity is still worth attention.
