# 04 · Molecular fingerprints

A fingerprint compresses molecular structure into a fixed-length vector, and it is the input to nearly all chemical machine-learning models. The point of this chapter is not "how to call it" but to understand exactly what information each fingerprint encodes — that determines what you lose when you pick the wrong one.

## Quantifying chemical intuition into a numerical vector

### 04.1 From "looks similar" to "computes similar"

When a chemist says "these two molecules are similar," the basis is structural features they can see — shared functional groups, similar scaffolds, close molecular size. A computer has no eyes; it needs an algorithm to turn "structurally similar" into "numerically similar."

Molecular fingerprints exist for exactly this. Their core idea is captured by an analogy:

> A fingerprint does not describe the molecule's "full picture"; it answers a series of yes/no questions: "Is there a benzene ring? Is there a hydroxyl group? Is there a carbon–oxygen double bond?" Each answer (0 or 1) occupies one position in the vector. This fixed-length 0/1 vector is a binary fingerprint.

### 04.2 The philosophical difference between two fingerprint families

**Structural keys — "take a checklist and verify the goods"**

Represented by the MACCS fingerprint. MACCS contains 166 predefined substructure patterns (the public 166-key subset); each bit corresponds to a specific chemical feature: bit 137 means "an O present in a heterocycle," bit 163 means "at least 1 aromatic ring."

```
MACCS fingerprint structure:
bit 0: [placeholder, unused]
bit 1: isotope
bit 2: atomic number > 103
...
bit 137: heterocycle O
...
bit 163: aromatic ring count > 0
bit 164: ring atom count > 0
bit 165: ring count > 1
bit 166: ring count > 2
```

**Advantage:** Every bit has a clear chemical meaning and can be read directly. "bit 163 = 1" means "this molecule has an aromatic ring" — no guessing needed.
**Disadvantage:** If your molecule contains an important structural feature (say, a special fused-ring system) that has no corresponding key on the list, that feature is ignored and never appears in the fingerprint.

**Hashed fingerprints — "record whatever you see"**

Represented by Morgan/ECFP fingerprints. No predefined dictionary is needed; instead, starting from the molecule itself, all circular atom environments are enumerated and mapped into a fixed-length vector via a hash function.

**Philosophical difference:** A structural key asks "do you have these things I knew in advance?"; a hashed fingerprint says "whatever you have, tell me about it."

### 04.3 The mathematical intuition behind Morgan fingerprints (ECFP)

The Morgan algorithm was originally proposed by H. L. Morgan in 1965 to solve the graph-isomorphism problem, and was later extended by Rogers and Hahn (2010) into the Extended-Connectivity Fingerprint (ECFP).

**The core of the algorithm (iterative implementation):**

1. **Initialization:** Each atom is assigned an integer ID encoding its local attributes:
   - Atomic number (6 = carbon, 7 = nitrogen, 8 = oxygen ...)
   - Degree (number of heavy-atom neighbors)
   - Formal charge
   - Whether it is a ring atom
   - Implicit hydrogen count

2. **Iteration:** For each round at radius = r:
   - Atom i collects its own ID from round r-1
   - Collects the round r-1 IDs of all neighbors j
   - Collects the bond type of the connecting edge j–i
   - Combines the above → passes through a hash function → produces the new ID for round r

3. **Collection:** All unique IDs produced by all atoms across all rounds form the molecule's feature set.

4. **Folding:** Feature IDs are mapped into the fixed-length vector via a modulo operation (ID mod fpSize).

**The chemical meaning of the radius parameter:**

```
radius=0: encodes only the atom itself (C, N, O...)         → atom type
radius=1: encodes atom + direct neighbors                    → functional-group level
radius=2: encodes atom + two layers of neighbors (ECFP4, diameter 4)      → functional group + local environment
radius=3: encodes atom + three layers of neighbors (ECFP6, diameter 6)      → larger substructures
```

ECFP4 (radius=2) is the industry standard because it strikes the best balance between "covering enough chemical environment" and "keeping reasonable specificity." Too small a radius and two different functional groups are indistinguishable in the fingerprint; too large and the fingerprint becomes overly specific, with almost no shared features between training and test sets.

### 04.4 The information bottleneck: hash collisions

Hashed fingerprints carry an inherent trade-off: mapping a potentially unlimited number of distinct substructures into a finite 2048 bits inevitably causes collisions.

**Collision probability (the balls-into-bins problem):** If you place n features into m positions, the probability that a given feature collides with another is:

```
P(collision) ≈ 1 - (1 - 1/m)^(n-1)
```

For a typical drug molecule (n ≈ 100–500 unique substructures) and fpSize=2048:
- The expected collision probability is roughly 5%–22%
- Doubling fpSize to 4096 roughly halves the collision probability

**This is not a bug — it is a design trade-off.** A longer fingerprint reduces collisions but increases storage and compute cost. At 2048 bits, the impact of collisions on the performance of most ML tasks is under 1%, so in practice it is a reasonable compromise.

**The mitigating role of count fingerprints:** A more subtle approach is to drop the 0/1 scheme and instead record how many times each position was hit by a substructure. That way, even on collision, the information is not entirely lost — a position value of 3 means at least 3 distinct substructures mapped here. A count fingerprint carries higher information entropy than a binary fingerprint of the same length, at the cost of a more complex similarity calculation.

### 04.5 How the fingerprint types complement each other

| Information type | MACCS | Morgan (ECFP4) | Atom Pair | RDKit Topological |
|----------|-------|----------------|-----------|-------------------|
| Atom type | ✓ | ✓ | ✓ | ✓ |
| Functional-group presence | ✓✓ (exact match on 166 keys) | ✓ (captured indirectly via environment) | ✓ | ✓ |
| Local neighbor environment | ✗ | ✓✓ (core strength) | partial | ✓ |
| Inter-atomic topological distance | ✗ | ✗ | ✓✓ (core strength) | ✓ (via path length) |
| Linear path patterns | ✗ | ✗ | ✗ | ✓✓ (core strength) |
| Interpretability | ✓✓ (each bit has a clear meaning) | ✗ (bits have no direct chemical meaning) | ✓✓ (can be reversed) | partial |
| Sensitivity to molecular size | low | medium | high | medium |

**Composite strategy in practice:** Many production-grade QSAR systems use ECFP4 (local environment) + MACCS (functional-group checklist) + physicochemical descriptors (global properties) together, concatenating the three into a feature vector. This is more robust than any single fingerprint.
