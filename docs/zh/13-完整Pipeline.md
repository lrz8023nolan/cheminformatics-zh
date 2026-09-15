# 13 · 完整 Pipeline

前面各章都是能力点，这一章把它们串成一条线：从原始文件出发，经过标准化、去重、特征计算，最终产出一份可以直接喂给模型的数据。顺序不能乱——这是全书里最值得直接抄去改的一段代码。

## 从数据到模型的「生产化」思维

### 13.1 Pipeline 的核心原则

化学数据 pipeline 与通用数据 pipeline 的不同：

1. **数据不是干净的数值——它是结构式的集合。** 原始数据中的同一分子可能以多种形式出现（盐、互变异构体、不同 SMILES 写法）
2. **特征不是预定义的——需要从结构中"提取"。** 描述符和指纹的选择直接影响下游模型的行为
3. **去重是化学问题，不是字符串问题。** `"CC(=O)O"` 和 `"[Na+].CC(=O)[O-]"` 在字符串层面完全不同，但在化学上是同一实体

### 13.2 标准化的时机

```
原始 SDF → 标准化 → 去重 → 特征提取 → ML 输入
           ↑
        这一步是最容易出错的
```

**错误做法：** 先去重，再标准化。这会导致标准化后产生的新重复被遗漏。
**正确做法：** 先标准化，再去重（基于标准化后的规范 SMILES 或 InChIKey）。

### 13.3 PandasTools 的设计哲学

`PandasTools` 不是 RDKit 的"额外功能"——它是化学数据工程的核心基础设施。它让你能在同一个 DataFrame 中存储三种不同粒度的分子信息：

1. **SMILES 字符串**（存储和传输用）
2. **Mol 对象**（计算和分析用）
3. **渲染图**（检查和交流用）

三者通过 `PandasTools` 的方法自动保持同步。这种设计让你能够在一次数据加载后，无缝地在"看分子"和"算分子"之间切换。

### 13.4 性能优化：何时用 Pickle，何时用 SMILES

| 存储方式 | 加载速度 | 文件大小 | 适用场景 |
|----------|---------|----------|----------|
| SMILES 文本 | 慢（需重新解析） | 最小 | 长期存档、跨工具传输 |
| Pickle (.pkl) | 快（直接反序列化） | 较大 | 重复使用的中间结果 |
| Binary (.ToBinary) | 最快 | 中等 | 内存中大量分子的快速传递 |

**实践规则：** 对超过 10,000 个分子的库，首次加载后立即 pickle 一份。后续工作从 pickle 加载，速度提升 10-50 倍。

### 13.5 构象生成的并行化

`EmbedMultipleConfs` 和 `MMFFOptimizeMoleculeConfs` 的 `numThreads` 参数在计算密集型任务中至关重要。对于 100 个分子各生成 50 个构象（共 5,000 个构象优化），单线程可能需要数十分钟，多线程可以缩短到几分钟。

`numThreads=0` 的含义是"使用所有可用核心"——这是对计算资源最友好的默认设置。

---

## 六步完整 Pipeline

以下是从原始 SDF 到机器学习就绪数据的完整 pipeline：

```python
from rdkit import Chem
from rdkit.Chem import AllChem, Descriptors, PandasTools
from rdkit.Chem import rdFingerprintGenerator, rdMolDescriptors
from rdkit.Chem.MolStandardize import rdMolStandardize
import pandas as pd
import numpy as np

# ============================================================
# Step 1: 数据加载
# ============================================================
df = PandasTools.LoadSDF(
    'assay_data.sdf',
    smilesName='SMILES',
    molColName='Molecule',
    removeHs=True
)
print(f"加载 {len(df)} 条记录")

# ============================================================
# Step 2: 分子标准化
# ============================================================
def standardize_mol(mol):
    if mol is None:
        return None
    try:
        Chem.SanitizeMol(mol)
    except:
        return None
    mol = rdMolStandardize.Cleanup(mol)
    return mol

df['Molecule'] = df['Molecule'].apply(standardize_mol)
df = df[df['Molecule'].notna()].reset_index(drop=True)

# 去重（基于规范 SMILES）
df['CanonicalSMILES'] = df['Molecule'].apply(Chem.MolToSmiles)
df = df.drop_duplicates(subset='CanonicalSMILES')
print(f"标准化+去重后: {len(df)} 条记录")

# ============================================================
# Step 3: 描述符计算
# ============================================================
df['MW'] = df['Molecule'].apply(Descriptors.MolWt)
df['LogP'] = df['Molecule'].apply(Descriptors.MolLogP)
df['TPSA'] = df['Molecule'].apply(Descriptors.TPSA)
df['HBA'] = df['Molecule'].apply(Descriptors.NumHAcceptors)
df['HBD'] = df['Molecule'].apply(Descriptors.NumHDonors)
df['RotBonds'] = df['Molecule'].apply(Descriptors.NumRotatableBonds)
df['AromRings'] = df['Molecule'].apply(Descriptors.NumAromaticRings)
df['HeavyAtoms'] = df['Molecule'].apply(Descriptors.HeavyAtomCount)
df['FractionCsp3'] = df['Molecule'].apply(
    rdMolDescriptors.CalcFractionCSP3
)

# ============================================================
# Step 4: 指纹生成
# ============================================================
mfpgen = rdFingerprintGenerator.GetMorganGenerator(
    radius=2, fpSize=2048
)

fingerprints = df['Molecule'].apply(
    lambda mol: mfpgen.GetFingerprintAsNumPy(mol)
)

fp_matrix = np.vstack(fingerprints.values)
print(f"指纹矩阵形状: {fp_matrix.shape}")  # (n_mols, 2048)

# ============================================================
# Step 5: 特征整合
# ============================================================
# 数值描述符
desc_cols = ['MW', 'LogP', 'TPSA', 'HBA', 'HBD', 'RotBonds',
             'AromRings', 'HeavyAtoms', 'FractionCsp3']
X_desc = df[desc_cols].values.astype(np.float64)

# 合并描述符 + 指纹
X_combined = np.hstack([X_desc, fp_matrix])
print(f"合并特征矩阵形状: {X_combined.shape}")

# 目标变量（假设 SDF 中有 ACTIVITY 字段）
if 'ACTIVITY' in df.columns:
    y = df['ACTIVITY'].values
    print(f"目标变量: {len(y)} 个值, 范围 [{y.min():.2f}, {y.max():.2f}]")
```

---

## 综合练习

### 13.6 练习 1：虚拟化合物枚举

给定核心骨架 `c1ccccc1` 和取代基列表 `['F', 'Cl', 'Br', 'O', 'N', 'C']`（作为 SMILES 片段），使用 R 基团分解的逆过程，枚举所有可能的单取代苯衍生物，并过滤掉化学上不合理的产物。

### 13.7 练习 2：构象分析

对布洛芬（`CC(C)CC1=CC=C(C=C1)C(C)C(=O)O`）生成 50 个构象，进行 Butina 聚类（cutoff=0.5），然后：
1. 计算每个聚类代表构象的能量
2. 可视化最低能量构象和最高能量构象的 3D 叠合

### 13.8 练习 3：反应模板应用

定义酯水解反应模板（`[C:1](=[O:2])[O:3][C:4].[OH2]>>[C:1](=[O:2])[OH].[C:4][OH]`），然后在以下酯类库上运行该反应：
`["CC(=O)OCC", "CC(=O)OC(C)C", "c1ccccc1C(=O)OCC"]`

### 13.9 练习 4：完整 QSAR Pipeline

从 ChEMBL 下载一组对靶标 EGFR 的活性数据，完成：标准化 → 去重 → 描述符计算 → 指纹生成 → XGBoost 回归模型训练。报告 R² 和 RMSE。

---

## 这些能力在药物发现工作流中的位置

### 13.10 化学反应处理 → 逆合成分析和虚拟库枚举

逆合成分析与反应条件预测，都以反应模板的编写和 `RunReactants` 机制为前提。

### 13.11 构象生成 → 3D QSAR 和对接前处理

虽然 RDKit 本身不做分子对接，但 RDKit 生成的 3D 构象是大多数对接软件（AutoDock Vina、Schrödinger Glide）的标准输入格式。能够批量生成和筛选低能构象是虚拟筛选 pipeline 的关键环节。

### 13.12 骨架与 R 基团分解 → SAR 分析

在药物研发环境中，合成化学家会持续产生新的化合物及其活性数据。R 基团分解和 MMPA 能帮你从这些数据中自动提取结构-活性关系——让数据的价值超出单个化合物的水平。

### 13.13 Pipeline 工程 → 可复现的研究

实际工作中往往需要自己搭建 pipeline。良好的标准化、序列化、并行化实践不是"额外的质量保障"，而是研究的可复现性的基础——你的结果需要能够被自己和团队在未来验证。

---
