# 14 · Common pitfalls and troubleshooting

This chapter does not teach new material. It collects the points scattered through the preceding chapters that are obvious once you know them — and cost you half a day when you don't.

Each entry is organized by **Symptom → Cause → Fix**. The vast majority of problems throw no exception; they just silently return a wrong answer — which is exactly what makes cheminformatics so troublesome: **wrong code often runs to completion; it's just that the answer is wrong.**

> This chapter is based on RDKit 2025.09.x. Behavior may differ across versions; where version matters, it is noted.

---

## 1 · Reading and parsing data

### 14.1 Not checking the return value of `MolFromSmiles`

**Symptom:** The script crashes partway through with `AttributeError: 'NoneType' object has no attribute 'GetNumAtoms'`, or worse — no error at all, but a batch of molecules is quietly missing from the downstream feature matrix and you never notice.

**Cause:** When `Chem.MolFromSmiles()` encounters a SMILES it cannot parse, it **does not raise an exception; it returns `None`**. This is the most common trap in RDKit, because the reasons for failure are often subtle: a miswritten chiral tag, unbalanced parentheses, an illegal atom valence.

**Fix:** Treat "returns `None`" as a normal branch, not an exceptional case.

```python
from rdkit import Chem

def safe_mol(smiles):
    if not isinstance(smiles, str) or not smiles.strip():
        return None
    return Chem.MolFromSmiles(smiles.strip())

# when converting in bulk, record the failures — don't silently discard them
mols, failed = [], []
for smi in smiles_list:
    m = safe_mol(smi)
    if m is None:
        failed.append(smi)
    else:
        mols.append(m)

print(f"success: {len(mols)}, failed: {len(failed)}")
if failed:
    print("failed examples:", failed[:10])
```

**Why you must print the failure list:** If the failure rate is 0.1%, you won't notice it at all on a small test set; by the time you run the full library you find the feature matrix is short by thousands of rows, and you can no longer tell which ones.

---

### 14.2 Bad records in an SDF silently become `None`

**Symptom:** You use `SDMolSupplier` to read an SDF with 1000 records, but only get 987 molecules back, with no error.

**Cause:** SDF is a text format; if any single record's structure block is corrupt (atom count mismatch, missing coordinates, encoding anomaly), `SDMolSupplier` reads that record as `None` rather than aborting the read.

**Fix:** Filter explicitly when reading, and **check in the order before `SetDataField`** — confirm the molecule object is valid before reading its properties.

```python
supplier = Chem.SDMolSupplier("library.sdf")
mols = [m for m in supplier if m is not None]

missing = supplier.GetCounter() if hasattr(supplier, "GetCounter") else None
if len(mols) != len(supplier):
    print(f"note: {len(supplier) - len(mols)} records failed to parse")
```

If you need to locate which records are bad, read them one at a time and record the index. **Don't assume "no error means everything was read in".**

---

### 14.3 Treating canonical SMILES as a molecule's unique identifier

**Symptom:** You dedupe using canonical SMILES, yet the same compound appears twice in the library; or the opposite — two different compounds are judged to be the same.

**Cause:** canonical SMILES only guarantees determinism **under the same standardization pipeline** and **in the same RDKit version**. It does not handle these cases:

- **Stereochemistry**: `Chem.MolToSmiles(mol)` keeps stereo information by default, but if your molecule object lost its stereo information when it was created, the output degrades
- **Tautomers**: the keto and enol forms are two different canonical SMILES, yet chemically the same substance
- **Salts and free forms**: without standardization, the sodium salt and the free acid are two completely different strings
- **Version differences**: the canonical ordering algorithm may change across RDKit versions, so cross-version comparison can mismatch

**Fix:** Treat canonical SMILES as "the result after standardization", not as "the identifier of the raw data".

```python
from rdkit import Chem

def canonical_key(mol):
    """stable key for deduplication: salt stripping + neutralize + normalize, then take canonical SMILES"""
    from rdkit.Chem.MolStandardize import rdMolStandardize
    mol = rdMolStandardize.FragmentParent(mol)     # keep the largest organic fragment
    mol = rdMolStandardize.Uncharger().uncharge(mol)
    mol = rdMolStandardize.TautomerEnumerator().Canonicalize(mol)
    return Chem.MolToSmiles(mol)
```

One related point: **RDKit's `Mol` object cannot be used directly as a `dict` key or placed in a `set`** (it has no value-based hash). When you need to dedupe, use the canonical SMILES string as the key, or use the result of `Chem.MolToSmiles`.

---

## 2 · Standardization

### 14.4 Forgetting to rebuild fingerprints after standardization

**Symptom:** Your standardization pipeline is correct, but the model performance doesn't change.

**Cause:** Standardization changes the molecule object itself, while the fingerprint is computed from the molecule object **at a certain moment**. If you generated the fingerprint before standardizing (or cached and reused it), the downstream code is still using the unstandardized version.

**Fix:** Strict order: read in → standardize → dedupe → **then** compute fingerprints and descriptors.

```python
# correct order
mols = [Chem.MolFromSmiles(s) for s in smiles_list]
mols = [standardize(m) for m in mols if m is not None]
mols = dedupe_by_canonical(mols)          # ← dedupe also happens after standardization
fps  = [generator.GetFingerprint(m) for m in mols]   # ← fingerprints are computed last
```

The complete pipeline in Chapter 13 is organized in exactly this order; you can check against it.

---

### 14.5 Blindly neutralizing all charges

**Symptom:** After the `Uncharger` runs in your standardization, the positive charge on quaternary ammonium compounds disappears, or the molecule's overall charge state differs from what you expected.

**Cause:** Some charges are **permanent charges**, not pH-dependent. Quaternary ammonium nitrogens (such as choline, and many antimicrobials) are positively charged at any pH; neutralizing them is equivalent to changing the molecule itself.

**Fix:** RDKit's `Uncharger` already handles this with SMARTS-based rules, and by default it preserves the quaternary ammonium positive charge — **so don't write your own "loop over all atoms and zero out the charges" logic**. If you need stricter control, think through the consequences before using `Uncharger(force=True)`, and check how quaternary-ammonium-type molecules are handled.

Another easily overlooked point: **the neutralization strategy itself has its limits of applicability** (see the discussion in Chapter 03). If the downstream task is pH-sensitive — for example predicting solubility or absorption in the gastric environment (pH 2) — uniformly neutralizing carboxylates to a neutral form introduces systematic bias. A generic pipeline defaulting to "neutralize first" is reasonable, but you need to know when this assumption breaks down.

---

### 14.6 Assuming tautomer standardization yields the "most stable" form

**Symptom:** You expect standardization to give you the chemically most stable tautomer, but you actually get a different one.

**Cause:** Determining "which tautomer is most stable" requires quantum chemistry calculations or experimental data, at very high cost. `TautomerEnumerator` does something entirely different: **within a molecule's set of tautomers, it picks out one "canonical" representative according to a fixed set of rules**.

What it guarantees is **consistency** (the same molecule always maps to the same representative), not **physical correctness** (that this representative has the lowest energy).

**Fix:** Don't interpret the standardization result as the chemically most stable form. If the downstream task genuinely requires the most stable tautomer, that is a separate problem requiring specialized tools or calculations.

Put the other way around, this also explains one thing: **the judgment of "whether two molecules are the same" itself depends on the standardization rules you choose**. Change the rules and the deduplication result changes. So once the standardization pipeline is fixed, write it into the documentation and lock it down.

---

## 3 · Fingerprints and descriptors

### 14.7 Mixing old and new APIs

**Symptom:** In one file you see both `AllChem.GetMorganFingerprintAsBitVect(...)` and `rdFingerprintGenerator.GetMorganGenerator(...)`, and the fingerprints from the two don't match.

**Cause:** RDKit still ships a batch of old fingerprint APIs that remain usable but are no longer recommended. **The default parameters of the old and new APIs are not necessarily the same** (for example the default radius and bit width), so mixing them within one project can yield fingerprints on different scales, with no visible problem on the surface.

**Fix:** Standardize on the new API; use only one set across the whole project.

```python
from rdkit.Chem import rdFingerprintGenerator

# new version (recommended): build the generator object once and reuse it
morgan_gen = rdFingerprintGenerator.GetMorganGenerator(radius=2, fpSize=2048)
fp = morgan_gen.GetFingerprint(mol)

# old version (just be aware of it, don't use in new code)
# from rdkit.Chem import AllChem
# fp = AllChem.GetMorganFingerprintAsBitVect(mol, 2, nBits=2048)
```

**One performance note on the side:** The generator object (the return value of `GetMorganGenerator`) should be built once and reused. Creating the generator repeatedly inside a loop incurs unnecessary overhead. Docs and tutorials, to keep each snippet self-contained, often recreate it in every fragment — copying that into production code is inappropriate.

---

### 14.8 Hash collisions from the default `fpSize`

**Symptom:** Two structurally distinct molecules show a suspiciously high fingerprint similarity.

**Cause:** A fingerprint is a fixed-length bit vector; several different substructure fragments can be hashed to the same bit — that is a **collision**. The smaller the bit width and the larger the molecule, the worse the collisions. The default 2048 bits is usually enough for small and medium molecules, but **for large-molecule libraries, the collision rate at 1024 bits can exceed 20%**.

**Fix:** Choose the bit width by molecule size and library scale, and keep it in mind:

| Scenario | Recommendation |
|---|---|
| Small/medium molecules, modest scale | 2048 bits (default, usually sufficient) |
| Large molecules (e.g. natural products, peptides) | 4096 bits |
| Need fine discrimination between similar molecules | increase bit width, or switch to count-based fingerprints |

There is also a practical debugging tool: **`bit_info`**. It tells you which atom environment in the molecule each set bit corresponds to, and is the only effective tool for investigating "why were these two molecules judged similar".

```python
from rdkit.Chem import rdFingerprintGenerator
from rdkit.Chem import rdMolDescriptors

gen = rdFingerprintGenerator.GetMorganGenerator(radius=2, fpSize=2048)
info = {}
fp = gen.GetFingerprint(mol, bitInfo=info)
# info is of the form {bit_id: [(atom_idx, radius), ...]}, from which you can recover the chemical meaning of each bit
```

---

### 14.9 Assuming the MACCS fingerprint's bit width is adjustable

**Symptom:** You try to set `fpSize=4096` on a MACCS fingerprint and find the parameter does nothing or errors out outright.

**Cause:** The MACCS fingerprint is **fixed at 167 bits**, each bit corresponding to one predefined structural pattern ("does it contain a certain ring type", "is there a certain atom type", etc.). It has no hashing step, and therefore no concept of bit width — bit width is part of its definition and cannot be adjusted.

**Fix:** Accept this, and place MACCS correctly: its value lies in **each bit having a clear chemical meaning and very strong interpretability**, making it suitable for feature selection and result interpretation; the cost is small information capacity and weak discriminative power. **Don't treat it as a replacement for Morgan; they are complementary** (see the selection guide in Chapter 04).

---

## 4 · Substructure matching and similarity

### 14.10 Using a Python loop for bulk similarity computation

**Symptom:** Sorting ten thousand molecules by similarity takes the script several minutes.

**Cause:** Calling `TanimotoSimilarity` one by one repeatedly crosses the Python–C++ boundary, and this overhead far exceeds the similarity computation itself.

**Fix:** Use the bulk interface. It passes the whole list into the C++ layer in one shot, speeding things up by nearly two orders of magnitude.

```python
from rdkit import DataStructs

# slow: loop in the Python layer
# sims = [DataStructs.TanimotoSimilarity(query_fp, fp) for fp in fps]

# fast: single bulk computation
sims = DataStructs.BulkTanimotoSimilarity(query_fp, fps)
order = sorted(range(len(sims)), key=lambda i: sims[i], reverse=True)
```

`BulkDiceSimilarity`, `BulkCosineSimilarity`, and so on work the same way.

---

### 14.11 Not capping the number of substructure matches

**Symptom:** You run a substructure match on a protein or large molecule and it hangs.

**Cause:** Substructure matching finds **all** atom-mapping combinations that satisfy the condition. For a frequently occurring simple pattern (such as a carboxyl group) appearing on a large molecule, the number of matches can grow explosively. The default behavior is to find all matches.

**Fix:** When you only care about "is it present", use `HasSubstructMatch`; when you need the match positions, limit the count.

```python
# just test for existence — fastest
has_carboxyl = mol.HasSubstructMatch(carboxyl_pattern)

# need positional info, but limit the count
matches = mol.GetSubstructMatches(carboxyl_pattern, maxMatches=1000)

# for pattern screening on large molecules, also set useChirality=False (default) and avoid unconstrained generic matches
```

Similarly, **maximum common substructure (MCS) computation has even higher complexity**; `rdFMCS.FindMCS` must always set a `timeout`, otherwise it may hang for a long time:

```python
from rdkit.Chem import rdFMCS

res = rdFMCS.FindMCS(mols, timeout=30)   # unit is seconds; always set this
if res.canceled:
    print("MCS calculation timed out, result incomplete")
```

---

### 14.12 Treating the similarity threshold as a universal constant

**Symptom:** You copy from a paper a rule like "similarity > 0.7 counts as the same series", but it works poorly on your own data.

**Cause:** The similarity threshold **depends heavily on three things**: the fingerprint type used, the average molecule size, and the task goal. The same 0.7 means something entirely different on ECFP4 versus on MACCS.

**Fix:** Calibrate the threshold on your own data, and note a counterintuitive fact: **for virtual screening, the "neighbors" in the Tanimoto 0.4–0.5 range are often more valuable than those above 0.9**. The reason was covered in Chapter 07 — very high similarity usually means analogs within the same series whose activity is already known; medium similarity is where scaffold hopping is more likely.

So don't blindly chase "the most similar ones"; first be clear whether your goal is **reusing known activity** or **exploring new structural types**.

---

## 5 · A pre-flight checklist

Before handing any cheminformatics pipeline to downstream consumers, go through each item:

- [ ] Do you have a list of molecules that failed to parse? What is the failure rate?
- [ ] Is the order of standardization, deduplication, and feature computation correct?
- [ ] Does the whole project use only **one** fingerprint API?
- [ ] Does `fpSize` match the molecule size? Have you verified the collision level?
- [ ] Was the similarity threshold calibrated on **your own data**, or copied from somewhere?
- [ ] Do bulk operations go through the `Bulk*` interfaces?
- [ ] Did you set a `timeout` on MCS and other high-complexity computations?
- [ ] Did you set a random seed for sources of randomness (such as conformer generation, clustering)?
- [ ] Is the standardization rule written into the documentation? (Change the rules and the deduplication result changes)

The last item is the easiest to skip, yet it determines whether your results **can be reproduced by others** — including your future self six months from now.
