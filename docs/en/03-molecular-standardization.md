# 03 · Molecular standardization

In the same compound table, aspirin can appear in five forms — free acid, sodium salt, with water of crystallization, and so on. If you feed it to a model as-is, the model learns the differences in data-entry format rather than chemical laws. This chapter first explains why standardization is necessary, then gives a production-grade standardization function.

## Why you can't model raw data directly

### 03.1 The nature of the problem: one entity, many faces

Suppose you pulled the SMILES for "acetic acid" from three data sources:

```
Source A (vendor catalog): CC(=O)O          # neutral acetic acid
Source B (bioactivity library): CC(=O)[O-]      # acetate anion
Source C (chemical registry):  [Na+].CC(=O)[O-] # sodium acetate salt
```

These three strings differ byte-for-byte. Without standardization they are treated as three different molecules, assigned three different fingerprints, and modeled separately — then your QSAR model finds that the three forms of the same compound give different predictions. This is why standardization is not a "nice-to-have" but the zero-th step of every cheminformatics pipeline.

### 03.2 The chemical problem each step solves

**Step 1: Sanitize (basic checks)**

When reading molecules from an SDF file or other non-SMILES source, valence may be undefined and aromaticity may be unperceived. `Chem.SanitizeMol()` does three things:
- Computes the implicit hydrogen count for each atom (based on the standard valence table)
- Recognizes alternating single/double-bond rings as aromatic (Kekulization)
- Validates the overall chemical plausibility of the structure

**Step 2: Salt stripping / keep the largest organic fragment**

Organic salts (such as hydrochlorides, sodium salts) exist as salts in the solid state and on the shelf, but dissociate in solution. In medicinal chemistry we almost always care about the organic part. `LargestFragmentChooser(preferOrganic=True)` is driven by organic-chemistry intuition: if fragment A contains carbon and fragment B is Na⁺ or Cl⁻, keep A.

**Edge case:** Quaternary ammonium salts (such as choline) carry a permanent charge and cannot be "neutralized." The `Uncharger` in a standardization pipeline by default tries to neutralize other ionizable groups but should preserve the quaternary ammonium positive charge. That is why `Uncharger` uses SMARTS-based rules rather than blindly stripping charge.

**Step 3: Normalize (functional-group standardization)**

Different chemists may draw the same functional group using different resonance forms. Take the nitro group:

```
[N+](=O)[O-]   vs   [N](=O)=O
```

Both are chemically equivalent (resonance forms of the nitro group), yet as substructures they are entirely different SMARTS. The Normalizer uses predefined SMARTS transformation rules to map all such variants to a uniform representation.

**Step 4: Uncharge (charge neutralization)**

The protonation state of ionizable groups depends on pH. At physiological pH 7.4, most carboxylic acids exist as anions and most amines are protonated. But for machine-learning modeling, the usual strategy is to remove all pH-dependent charges and unify them to neutral form — because without knowing the specific assay conditions, the neutral form is the least-biased representation.

**Point of contention:** This approach is not perfect. If your model predicts properties at pH 2 (stomach environment), unifying carboxylic acids to neutral introduces bias. But as the default strategy for a general pipeline, "standardize to neutral first, add the charge back when needed" is the consensus of the field (ChEMBL, PubChem, canSAR).

**Step 5: Tautomer standardization**

This is the most complex step. Take warfarin as an example: it can exist in up to 40 different tautomers in solution. Keto–enol tautomerism is the best-known case:

```
Vinyl alcohol (enol form): C=CO      →   tautomerizes  →   Acetaldehyde (keto form): CC=O
(less stable)                                    (more stable)
```

**The core difficulty:** The most stable tautomer depends on solvent, temperature, pH, and other environmental factors — there is no "universally correct tautomer." The TautomerEnumerator's strategy is not to find the "most stable" form (that needs quantum-chemical computation) but to generate a **canonical arbitrary tautomer** — ensuring the same molecule is always mapped to the same representation.

**What this means for ML:** As long as the training and test sets use the same standardization pipeline, the canonical tautomer is consistent. Inconsistent tautomer handling can cause up to 67% of molecules to produce different fingerprints in two databases — meaning the model learns an artifact of the standardization algorithm rather than real chemical laws.

### 03.3 To standardize or not: an engineering judgment

A practical rule: for ML modeling and database deduplication — standardize, always. For chemical-structure storage — keep the original form, with standardization as a preprocessing step rather than a permanent modification.
