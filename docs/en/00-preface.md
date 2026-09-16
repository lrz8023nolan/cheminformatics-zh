# 00 · Preface

## How this book is organized

Each chapter is split into two parts:

- The **principles part** answers "why": why you must standardize, why fingerprints collide, why similarity has no objective answer.
- The **operations part** answers "how": runnable code, what the parameters mean, and what to watch for in production.

The order is deliberate. **Read the principles before the code**, and you will understand what trade-off each parameter makes. The other way around, you will only memorize a pile of API calls and will not know how to adapt them when the task changes.

Subsections within a chapter are numbered by `chapter.number` (e.g. `03.5`); cross-references use the same numbering, so you can jump between sections.

The recommended way to read Part One is straight through. If you only have a specific problem, jump to Chapter 14 — it lists the most common pitfalls by "symptom → cause → how to fix."

## Version conventions

- **RDKit 2025.09.x.** RDKit's API changes substantially between versions, so all behavioral descriptions in this book are pinned to this version.
- **Old and new APIs are both labeled.** The typical example is fingerprint generation: the new `rdFingerprintGenerator` (recommended) versus the old `AllChem.GetMorganFingerprint*` (still usable but no longer recommended). The text explains the differences between the two and recommends standardizing on the new API — **do not mix the two in the same project**; the reasoning is in 14.7.
- Where a behavior differs across versions, it is called out separately.

## Code conventions

- Every code snippet **runs as-is** and depends only on RDKit. The few that need an extra dependency are flagged.
- To keep snippets self-contained, some repeat imports and object creation. **Merge them when copying into production code** — for example, build a fingerprint generator once and reuse it; do not recreate it inside a loop (see 14.7).
- Example molecules are always publicly available compounds (ethanol, aspirin, caffeine, and so on).

## Prerequisites

Basic Python experience is enough; **no cheminformatics background is required.** Chemical concepts are explained intuitively and do not assume any medicinal chemistry training.

## On the sources

Every conclusion is given a citation where possible. The parts that involve specific algorithms and standardization cite the original literature, collected in Appendix B. Where there is genuine dispute or more than one accepted approach, the text marks the point of contention rather than giving a single answer — for example, the Chapter 03 discussion of "whether to neutralize all molecules to neutral form."

If you find an error or something out of date, please open an issue. In cheminformatics, one wrong default parameter can corrupt an entire dataset, so for anything touching API behavior or version differences, include how you verified it.
