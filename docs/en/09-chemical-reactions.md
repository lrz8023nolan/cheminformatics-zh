# 09 · Chemical reactions

Reactions are what set chemistry apart from other fields. Writing "A plus B yields C" in a machine-executable form is the shared prerequisite for virtual library enumeration, retrosynthesis, and reaction-condition prediction. This chapter first explains how reactions are formalized, then moves on to RDKit's `RunReactants` and product post-processing.

## From arrows to algorithms

### 09.1 The "data structure" problem of chemical reactions

In the lab, a chemical reaction is represented with structural formulas plus an arrow. A chemist can tell at a glance that "this carboxyl group reacts with that amine to form an amide." But a computer has no chemical intuition — it needs to understand "reaction" at a formal level:

> A reaction = a set of reactant-matching rules + atom-mapping relations + product-generation rules

This is not an abstract problem. Both retrosynthesis and reaction templates presuppose a formal representation of the reaction.

### 09.2 Atom mapping — the soul of a reaction definition

Consider the amide-bond formation reaction:

```
Reactant template:  [C:1](=[O:2])-[OD1]  .  [N!H0:3]
Product template:    [C:1](=[O:2])[N:3]
```

The `:1` in `[C:1]` is not a comment; it is the mechanism that establishes a one-to-one correspondence between atoms in the reactants and atoms in the products. Chemically, this corresponds to: **the carboxyl carbon (mapping 1) remains the central carbon before and after the reaction — its chemical environment changes, but its identity does not**.

The computational-chemistry equivalent of atom mapping is the **reaction coordinate** — a description of the path atoms take from the reactant state to the product state. In RDKit's implementation, the mapping information is saved as properties of the product atoms (`old_mapno`, `react_idx`, `react_atom_idx`) after `RunReactants` runs, so you can trace the origin of every atom in the product.

### 09.3 Two ways to generate reaction templates

In practice you encounter two ways of generating reaction templates:

**A. Hand-written (based on chemical knowledge)**

Hartenfeller et al. (2011) compiled Reaction SMARTS templates for 58 common classes of organic reactions. The advantages of this approach are:

- each reaction class has a clear chemical meaning
- the templates are precise, with a low false-positive rate
- it can be used for virtual compound enumeration ("what happens if I add a ketone to this scaffold?")

The downside is limited coverage — if a reaction is not among those 58 templates, it cannot be handled.

**B. Data-driven (automatically extracted from reaction databases)**

Reaction rules are extracted automatically from reaction databases such as USPTO or Reaxys by analyzing reactant–product pairs. This is one of the core techniques behind retrosynthesis software such as ASKCOS and AiZynthFinder.

**Identifying the reaction center is the key step** — determining which atoms and bonds change (form or break) between reactants and products.

### 09.4 The combinatorial explosion problem in RunReactants

When a reactant has multiple matching positions, `RunReactants` enumerates all possible combinations. This is not severe when running an esterification on a single carboxyl group (usually only 1–3 matches), but running a generic reaction template on a polyfunctional molecule can lead to a combinatorial explosion.

This is why retrosynthesis needs search strategies (such as Monte Carlo Tree Search) rather than simply enumerating all possibilities — the search space grows exponentially with the number of reaction steps.
