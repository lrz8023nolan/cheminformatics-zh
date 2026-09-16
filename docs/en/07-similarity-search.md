# 07 · Similarity search

"How similar are two molecules" seems intuitive, but is actually a scientific problem — different similarity coefficients yield completely different rankings. This chapter first discusses the assumptions behind similarity measures, then moves on to bulk similarity search and threshold filtering.

## Defining "similar" is itself a scientific problem

### 07.1 There is no objective "similarity"

> "Molecular similarity is a concept, not a physical observable." — Willett, 2006

Whether two molecules are "similar" depends on what measure you use, which features you focus on, and in what context. Benzene and pyridine: by scaffold (a six-membered aromatic ring) they are very similar, but by functional group (C–H versus an N lone pair) the difference is significant.

The job of cheminformatics is: **pick a representation (a fingerprint), pick a measure (a similarity coefficient), and then accept that the similarity under this framework is our operational definition.**

### 07.2 The Tanimoto coefficient (Jaccard index)

The most widely used molecular similarity measure, defined as follows:

```
T(A, B) = c / (a + b - c)

where:
a = number of bits set to 1 in molecule A's fingerprint
b = number of bits set to 1 in molecule B's fingerprint
c = number of bits set to 1 in both A and B (the intersection)
```

Equivalently: number of shared features / number of features in either (intersection / union).

**Why is Tanimoto the most common?**

1. Simple to compute: for binary vectors it needs only bit operations
2. Symmetric: T(A, B) = T(B, A)
3. Normalized: always in [0, 1]
4. Intuitive: 0 = no shared features, 1 = identical
5. Industry convention: almost all published virtual-screening benchmarks use Tanimoto

**Limitations of Tanimoto:**

- Sensitive to molecular size: the Tanimoto between a large and a small molecule is naturally low even when they share most of their structure — because the large molecule has more "extra" bits set to 1
- Sensitive to fingerprint density: the Tanimoto computed from MACCS (average set-bit density ~20–30 bits) and ECFP4 (average set-bit density ~100–300 bits) cannot be directly compared

### 07.3 The Dice coefficient and size sensitivity

```
Dice(A, B) = 2c / (a + b)
```

Dice is **more sensitive** to differences in molecular size. If molecule A is very small and molecule B very large, even when A is a perfect substructure of B, Dice will fall noticeably below 1 (because B has many extra bits). This makes Dice stricter when "looking for an exact structural match" and more lenient when "looking for a substructure relationship".

**Recommendations:**

- General similarity search → Tanimoto (industry standard)
- Substructure / superstructure detection → Tversky (the asymmetric version, with adjustable weights)
- Exact deduplication → Dice, or a direct canonical SMILES comparison

### 07.4 The engineering significance of bulk operations

`DataStructs.BulkTanimotoSimilarity(query_fp, library_fps)` is not a simple Python loop. It implements vectorization at the C++ level; for a library of tens of thousands of molecules, the bulk version is 10–50× faster than a Python loop. When your compound library grows to the hundred-thousand scale, this difference is the gap between "a few seconds" and "a few minutes".

### 07.5 "Rules of thumb" for similarity thresholds

- **Tanimoto ≥ 0.85 (ECFP4):** highly similar — usually means an identical scaffold with only minor modifications
- **0.5 ≤ T < 0.85:** moderately similar — may share a major scaffold or an important functional group
- **T < 0.5:** low similarity. But note: under ECFP4, two completely different molecules may still randomly yield 0.1–0.2 of background similarity (from incidental bit overlap)

**A counterintuitive fact:** for virtual screening, neighbors at Tanimoto = 0.4–0.5 are often more valuable than neighbors at 0.9+. Overly similar molecules are usually analogues from the same series (known activity), whereas moderately similar molecules may belong to a scaffold hop (unknown activity but a new structure type).
