# 08 · Molecular representations

The previous seven chapters covered concrete tools: how to read a molecule, how to standardize it, which fingerprint to use. This chapter steps back to answer a higher-level question — **which layer of representation to choose in which scenario**. From 1D, 2D, and 3D up to learned representations, how much information each layer can carry and at what cost — once you understand this ladder, the concrete choices made earlier have a basis.

## The hierarchy of representations: a ladder of information dimensions

### 08.1 The four levels of representation

Molecular representations can be divided into four levels by their information dimension:

```
                    Information richness →
    ┌──────────┬──────────────┬──────────────┬──────────────────┐
    │  0D/1D   │     2D       │     3D       │    Learned       │
    │ Scalar    │  Topology    │  Conformation │  Data-driven     │
    │ properties│  structure   │  in space    │  representation  │
    ├──────────┼──────────────┼──────────────┼──────────────────┤
    │ MW       │ SMILES       │ 3D coords    │ GNN embedding    │
    │ LogP     │ Molecular    │ Electrostatic│ ChemBERTa        │
    │          │ graph        │ potential    │ embedding        │
    │ Element  │ Molecular    │ Shape        │ Mol2vec          │
    │ comp.    │ fingerprint  │ descriptor   │ embedding        │
    │          │ 2D descriptor│ Pharmacophore│                  │
    ├──────────┼──────────────┼──────────────┼──────────────────┤
    │ Simplest │ Cheminfor-   │ Structural   │ Deep learning    │
    │ Fastest  │ matics main  │ biology,     │ frontier         │
    │          │ battlefield  │ docking-     │ direction        │
    │          │              │ related      │                  │
    └──────────┴──────────────┴──────────────┴──────────────────┘
```

**The core question at each level:**

- **0D/1D:** "How heavy is this molecule? How lipophilic? How many hydrogen-bond donors does it have?" → describes the molecule's global properties, independent of structure
- **2D:** "What functional groups does it have? What does the scaffold look like? Are two molecules topologically similar?" → the most mature level of cheminformatics
- **3D:** "What shape does it take in space? How is the electrostatic potential distributed? What conformation does it adopt when binding to a protein?" → structure-oriented drug discovery
- **Learned:** "What features does the data tell me are most useful for predicting this property?" → end-to-end learning, abandoning hand-crafted definitions

### 08.2 Why there is no "best" representation

This is a fundamental realization. If one representation were optimal for every task, the others would have been eliminated long ago. The reality is:

- For solubility prediction, 2D descriptors + XGBoost usually match GNNs (and sometimes beat them)
- For protein–ligand binding-affinity prediction, 3D information is crucial (because binding depends on spatial complementarity)
- On small datasets, hand-designed fingerprints and descriptors far outperform learned embeddings (because the latter need lots of data to train)
- On large datasets, learned embeddings usually beat fixed fingerprints (because they can be optimized for the target task)

**The "no free lunch" theorem as it appears in molecular representation:** every representation carries an inductive bias — a prior assumption about which structural features "should" be important. This bias helps on some tasks and hurts on others.

---

## 1D representation: quantity over quality

### 08.3 The information carried by scalar descriptors

A single molecular weight (MW = 180.16) carries almost no structural information by itself. But when you consider a set of descriptors, together they provide a "numerical abstraction of chemical intuition":

```
aspirin:
MW=180.16, LogP=1.31, TPSA=63.6, HBD=1, HBA=4, RotBonds=3, Rings=1
```

Taken together, an experienced medicinal chemist can already judge "this is a small molecule with high odds of oral absorption". That is the power of 1D representation — with few but interpretable numbers, it captures the key global trends.

### 08.4 Limitations of 1D representation

- **Structural isomers cannot be distinguished:** MW and LogP are identical for ortho- and para-xylene
- **Insensitive to local structural change:** replacing an –OH with a –CH₃ shifts LogP by about 0.5, but this signal may be drowned out by noise in the descriptor set
- **No spatial information:** it cannot describe "where in the molecule this functional group sits"

### 08.5 The best-fit scenarios for 1D

- **High-throughput prescreening:** Lipinski's rule, the REOS filter (quickly weed out obviously unsuitable molecules)
- **As supplementary features for an ML model:** concatenate a descriptor column onto fingerprints to provide "global property" context

---

## 2D representation: the core paradigm of cheminformatics

### 08.6 The essence of 2D representation: encoding molecular-graph topology

2D representation tries to answer a basic question: **how to encode a molecular graph (atoms = nodes, bonds = edges) into a fixed-length numerical vector?**

This question is not trivial. A molecular graph is a heterogeneous graph (different types of nodes and edges) and variable in size. ML models need fixed-length input. All 2D representation methods are essentially solving this "graph → vector" mapping problem.

### 08.7 A comparison of the three major 2D representations

| | Structural keys (MACCS) | Hashed fingerprint (ECFP4) | Topological fingerprint (Atom Pair) |
|---|---|---|---|
| **Encoding** | Presence of predefined substructures | Hashing of circular atom environments | Atom-pair distance distribution |
| **Bit meaning** | Explicit (bit 137 = heterocyclic O) | Implicit (hash result) | Semi-interpretable (can be traced back) |
| **Coverage completeness** | Limited by the predefined list | Theoretically covers all substructures | Covers all atom pairs |
| **Sparsity** | Moderate (~20–30 bits on) | Moderate (~100–300 bits on) | High (sparse storage) |
| **Size sensitivity** | Low (small molecules still don't lose many bits) | High (large molecules have more unique substructures) | Very high (O(n²) pairs) |

### 08.8 Information loss in representation — a seriously underestimated problem

Any 2D representation inevitably loses information. The key is to know **what is lost**:

**What SMILES loses:**

- 3D conformational information (none at all)
- Exact bond lengths and angles
- Non-covalent interactions (hydrogen-bond networks, π–π stacking)

**What molecular fingerprints lose (relative to the full molecular graph):**

- The spatial relationship between substructures (are two functional groups on the same side of the molecule or opposite sides?)
- Substructure count information (in binary fingerprints)
- Global topology (the "macroscopic" features of molecular shape)
- Information confusion caused by hash collisions

**For some tasks, the lost information happens to be irrelevant:** for LogP prediction, 3D conformational information is almost irrelevant (LogP mainly depends on atom types and connectivity). For target-binding prediction, 3D conformation is crucial.

### 08.9 Formalizing fingerprint "expressiveness"

In ML theory, a representation's "expressiveness" can be defined in terms of its discriminative power:

> The discriminative power of a representation f = how many pairs of non-equivalent molecules are assigned different representation vectors.

- The molecular graph (full topology) has the greatest discriminative power — if two molecular graphs are not isomorphic, they differ
- SMILES preserves the graph's full information (it can be completely reconstructed)
- ECFP4 may hash different substructures to the same bit (collision), so its discriminative power is weaker than the molecular graph's
- MACCS has the weakest discriminative power — if two molecules happen to match on all 166 predefined keys, their fingerprints are identical

**In practice:** ECFP4's collision rate (about 5–22% at fpSize = 2048) is acceptable for most ML tasks, because colliding substructures usually differ in size or position and are unlikely to affect activity at the same time.

---

## 3D representation: the importance of spatial information

### 08.10 Why 2D representation can't "reach" certain properties

Consider two situations that can affect activity:

**Case A: E/Z (cis–trans) isomerism**

- The 2D graphs are identical, but in 3D space the functional groups sit on the same or opposite side of the double bond
- A 2D fingerprint cannot distinguish them; 3D coordinates can

**Case B: Global shape complementarity**

- Two molecules have completely different topologies, yet nearly identical 3D shapes (a molecular mimicry phenomenon)
- A 2D fingerprint shows them as very different (Tanimoto < 0.3); a 3D shape measure shows them as very similar

These are why relying on 2D representation alone is insufficient.

### 08.11 Types of 3D representation

| Representation type | Information captured | Rotation/translation invariance | Computational cost |
|--------------------|----------------------|--------------------------------|--------------------|
| 3D coordinates | All spatial information | None (needs alignment first) | Low (after generating coordinates) |
| Electrostatic-potential grid | Spatial pattern of charge distribution | None (needs alignment) | High |
| Shape descriptor (PMI, radius of gyration) | Global shape features | Yes | Low |
| Pharmacophore fingerprint | Spatial arrangement of pharmacophore feature points | Yes | Medium |
| 3D autocorrelation descriptor | Distance distribution of atoms | Yes | Medium |

### 08.12 The core challenge of 3D representation — conformational dependence

A 2D representation is deterministic: one SMILES always produces the same fingerprint. A 3D representation is not: the same molecule has different 3D features in different conformations.

**Handling strategies:**

1. **Lowest-energy conformation (single point):** simple, but ignores flexibility
2. **Ensemble average (multiple points):** compute 3D features for multiple conformations, then take the mean or a weighted average
3. **Dynamic features (MD trajectory):** sample conformations in a molecular-dynamics simulation and compute time-series features — most accurate but most expensive

---

## Learned representation: let the data decide what is "important"

### 08.13 From "hand-defined" to "data-driven"

The limitation of traditional representations (fingerprints, descriptors) is that they make a prior assumption about "which structural features matter". ECFP4 assumes circular atom environments matter; MACCS assumes 166 specific substructures matter. Learned representations make no such assumption — they are learned from training data.

### 08.14 Major learned-representation methods

**A. Sequence-based embeddings (SMILES → vector)**

- **ChemBERTa / MolBERT:** treat SMILES as natural language and pretrain with a BERT-style architecture on large chemical libraries
- **Principle:** the model learns SMILES' internal representation (syntax and semantics) by predicting masked atoms/fragments
- **Strength:** can exploit all available SMILES strings, no labeled data required
- **Weakness:** a mismatch exists between the 1D sequence structure of SMILES and the 2D graph structure of the molecule — adjacent SMILES tokens do not necessarily correspond to adjacent atoms

**B. Graph-based embeddings (molecular graph → vector)**

- **GNN (GCN, GAT, GIN, MPNN):** run message passing directly on the molecular graph
- **Principle:** each atom aggregates neighbor information, and through multiple iterative layers produces an embedding vector for the whole molecule
- **Strength:** naturally fits the irregular structure of molecular graphs, with theoretical expressiveness guarantees for topological isomorphism
- **Weakness:** needs labeled data to train (or self-supervised pretraining)

**C. 3D-based embeddings (molecular graph + 3D coordinates → vector)**

- **SchNet, DimeNet++, Equiformer:** incorporate interatomic 3D distances and angles into message passing
- **Principle:** information propagates not only along topological edges but also "perceives" distant atom pairs in space
- **Strength:** can capture spatial interactions that 2D GNNs cannot perceive
- **Weakness:** needs 3D conformers as input, adding computational and data dependencies

### 08.15 Learned vs traditional representation — when to use which

| Scenario | Recommended | Reason |
|----------|-------------|--------|
| Labeled data < 1000 | Traditional fingerprint + descriptors | Learned representations need large amounts of data to learn meaningful features |
| Labeled data 1,000–10,000 | Traditional representation as main, optionally fine-tune pretrained embeddings | Pretrained embeddings provide a good starting point |
| Labeled data > 10,000 | Compare traditional and learned, pick the better | Both are worth trying |
| Labeled data > 100,000 | Train a GNN from scratch | Enough data to support end-to-end learning |
| Interpretability needed | Traditional descriptors + SHAP | Dimensions of learned embeddings carry no chemical meaning |
| Unsupervised pretraining | ChemBERTa / self-supervised GNN | Exploit large unlabeled molecular libraries |

### 08.16 A counterintuitive empirical result

Multiple benchmark studies repeatedly find: **on QSAR tasks with medium data volumes (< 10,000), ECFP4 + XGBoost often matches or even beats state-of-the-art GNNs.** This is not to say GNNs are bad — rather that the structural information ECFP4 captures is already rich enough, and XGBoost is extremely efficient on this kind of tabular data.

The advantage of learned representations is most obvious in two scenarios: (1) large-scale data (100,000+); (2) tasks involving 3D spatial information (such as binding-affinity prediction).

---

## Decision framework for representation selection

### 08.17 Four core questions

Facing a concrete task, answer these four questions to settle on a representation strategy:

```
Q1: What is the chemical nature of the task?
    ├── Global property (LogP, solubility) → 1D descriptors suffice
    ├── Local structure-activity relationship → 2D fingerprint (ECFP4)
    ├── Protein-ligand interaction → 3D representation + docking
    └── Unknown / complex → combine several representations + compare models

Q2: How much labeled data is there?
    ├── < 1,000 → traditional representation (ECFP4 + descriptors + XGBoost)
    ├── 1,000-10,000 → traditional representation + fine-tunable pretrained embedding
    └── > 10,000 → compare traditional vs learned experiments

Q3: Is interpretability needed?
    ├── Yes (to convince chemists) → traditional descriptors + SHAP/LIME + MACCS
    └── No (pure prediction) → learned representations can be tried

Q4: Is 3D structural information available?
    ├── Yes (protein structure known) → add 3D representation
    └── No (ligand-only data) → 2D representation as main
```

### 08.18 Combination strategy — production-grade practice

In practice, the most robust approach is not to pick "one best representation" but to **combine several complementary representations**:

```python
# Production-grade feature-engineering strategy (pseudo-code)
features = concatenate([
    ecfp4_fingerprint,        # 2048 bits: local circular environment
    maccs_fingerprint,       # 167 bits: interpretable substructures
    physchem_descriptors,     # ~20 dims: MW, LogP, TPSA, QED, etc.
    rdkit_topological_fp,     # 2048 bits: path fingerprint (complements ECFP)
])
```

**Why combine?** Because different representations have different sensitivities to different patterns of structural change. ECFP4 excels at capturing local functional-group changes but is relatively insensitive to global scaffold changes. The path fingerprint is exactly the opposite. Combining the two is more robust than using either alone.

### 08.19 Selection recommendations for specific scenarios

| Task | Recommended representation | Reason |
|------|----------------------------|--------|
| Retrosynthesis analysis | Reaction SMARTS + molecular graph | Reactions depend on local functional-group matching rules |
| Compound-library deduplication | Canonical SMILES + InChIKey | Needs exact identity match, no distance metric required |
| Similarity search | ECFP4 (2048) + Tanimoto | Industry standard, covers chemical space broadly enough |
| Property prediction (ADMET) | ECFP4 + physicochemical descriptors + XGBoost | With insufficient labeled data, traditional methods beat GNNs |
| Virtual screening | ECFP4 + 3D shape descriptor | 2D similarity + 3D shape complementarity |
| Synthetic accessibility | SA_Score + structural-complexity descriptors | Synthetic difficulty mainly depends on structural complexity |
| Target-binding prediction | 3D conformation + docking score + ligand descriptors | Binding affinity depends on spatial complementarity |

---

## Summary: three iron rules

1. **Start simple, add complexity gradually.** ECFP4 + descriptors + XGBoost is the baseline for any QSAR task. If that baseline already meets your needs (R² > 0.7, etc.), there is no reason to reach for a GNN.

2. **Representation choice = inductive-bias choice.** Every representation carries a prior assumption about "what matters". ECFP4 assumes circular environments matter; descriptors assume global properties matter; a GNN lets the data decide what matters. Pick the bias that best matches the nature of the task.

3. **Interpretability is not optional — especially in industry.** When you tell a medicinal chemist "the model predicts this molecule's IC50 = 5 nM", their next question will inevitably be "why?". If your representation does not support explanation (such as a deep-learning embedding), you need a backup explanation strategy (SHAP, MACCS bit analysis, MMPA).
