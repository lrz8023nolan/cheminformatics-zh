# Cheminformatics: A Chinese Practitioner's Guide — English Edition

**The English edition is a selected subset, not a translation of the whole book.**

[简体中文](../zh/) | **English**

---

## What this is

The English edition contains the **conceptual half** of the handbook: the sections that explain *why* — why standardization is mandatory before modelling, what each fingerprint actually encodes, why "similar" has no objective definition, why conformer sampling is a distribution problem rather than a single-answer one.

It is written for people who already know roughly what RDKit does and want to make better decisions with it.

## What this is not

- **Not a translation of the full book.** The Chinese edition also contains operational walkthroughs (environment setup, API usage, batch tooling). Those are deliberately excluded here — the official RDKit documentation, the RDKit Cookbook and teachopencadd already cover that ground well, and duplicating it would add maintenance cost without adding value.
- **Not a replacement for the official docs.** Chapter numbering is shared with the Chinese edition; where a chapter is missing here, it is because that chapter is operational rather than conceptual.
- **Not a full-stack drug discovery tutorial.** It stays at the cheminformatics layer — turning molecules into features. Modelling and applications are a separate book.

## Table of contents

Chapter numbers match the Chinese edition, so references stay stable across both.

| Chapter | Title |
|---|---|
| [00](./00-preface.md) | Preface |
| [01](./01-why-represent-molecules.md) | Why represent molecules at all |
| [03](./03-molecular-standardization.md) | Molecular standardization |
| [04](./04-molecular-fingerprints.md) | Molecular fingerprints |
| [05](./05-molecular-descriptors.md) | Molecular descriptors |
| [06](./06-substructure-matching.md) | Substructure matching |
| [07](./07-similarity-search.md) | Similarity search |
| [08](./08-molecular-representations.md) | Molecular representations |
| [09](./09-chemical-reactions.md) | Chemical reactions |
| [10](./10-conformations-and-3d.md) | Conformers and 3D |
| [11](./11-scaffolds-and-r-groups.md) | Scaffolds and R-groups |
| [13](./13-pipeline.md) | The complete pipeline |
| [14](./14-common-pitfalls.md) | Common pitfalls and troubleshooting |

> **Chapters 02 and 12** exist in the Chinese edition but not here: they are operational walkthroughs (molecule I/O and file formats; PandasTools and batch tooling).

### Where to start

If you want to judge in five minutes whether this is worth your time, read **[Chapter 14](./14-common-pitfalls.md)**. It lists twelve traps that make cheminformatics code produce *no error message and the wrong answer* — the kind of thing that costs a week to find and one sentence to avoid.

## Requirements

No RDKit installation is needed to read the conceptual chapters. Where code appears, it illustrates a point rather than forming a complete program.

## Version

Everything here reflects **RDKit 2025.09.x**. RDKit's API changes meaningfully between versions, so behaviour described here may differ elsewhere.

## License

- Documentation (`*.md`): [CC BY-SA 4.0](../LICENSE)
- Code samples (`*.py`): [MIT](../LICENSE-CODE)

---

*This project is not affiliated with the RDKit project. RDKit is maintained by Greg Landrum and contributors under a BSD 3-Clause licence.*
