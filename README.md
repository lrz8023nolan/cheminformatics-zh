# 化学信息学中文实践指南

> 一份写给中文使用者的化学信息学实践指南：从分子表示、标准化、指纹与描述符，到相似性搜索与特征工程 pipeline。

**简体中文** | [English](./docs/en/)（筹备中）

[![License: CC BY-SA 4.0](https://img.shields.io/badge/Docs-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](./LICENSE-CODE)
[![Language](https://img.shields.io/badge/%E8%AF%AD%E8%A8%80-%E4%B8%AD%E6%96%87-red.svg)](#)
[![RDKit](https://img.shields.io/badge/RDKit-2025.09.x-orange.svg)](https://www.rdkit.org/)

---

## 这是什么

一份**中文原创**的化学信息学实践指南。内容基于 RDKit 2025.09.x 实际验证过的 API 编写，覆盖从原始化学数据到可输入模型的数值特征的完整链路。

每一章都分成两段：**原理段**回答"为什么"，**实操段**回答"怎么写"。顺序是有意为之——先看原理再看代码，你会知道每个参数在做什么取舍。

## 这不是什么

- **不是官方文档的翻译。** 内容为原创编写，不是 RDKit 官方文档或 Cookbook 的中文译本。
- **不是一个 API 速查表。** 侧重"为什么"和工程判断，而不是罗列函数签名。
- **不是药物发现全栈教程。** 聚焦化学信息学这一层，不展开分子对接、动力学模拟等内容。

## 目录

### 第一部分：基础（已完成）

| 章节 | 主题 |
|------|------|
| [00 前言](./docs/zh/00-前言.md) | 本书怎么用、版本与代码约定 |
| [01 导论](./docs/zh/01-导论-为什么需要表示分子.md) | 为什么需要「表示」分子 |
| [02 分子读写与文件格式](./docs/zh/02-分子读写与文件格式.md) | SMILES / MOL / SDF 与可视化 |
| [03 分子标准化](./docs/zh/03-分子标准化.md) | 为什么不能直接拿原始数据建模 |
| [04 分子指纹](./docs/zh/04-分子指纹.md) | 将化学直觉量化为数值向量 |
| [05 分子描述符](./docs/zh/05-分子描述符.md) | 用数字「翻译」分子性质 |
| [06 子结构匹配](./docs/zh/06-子结构匹配.md) | 分子识别的逻辑引擎 |
| [07 相似性搜索](./docs/zh/07-相似性搜索.md) | 定义「相像」本身就是科学问题 |
| [14 常见陷阱与排错清单](./docs/zh/14-常见陷阱与排错清单.md) | 按「症状 → 原因 → 怎么改」组织 |
| [附录 B 参考来源](./docs/zh/附录B-参考来源与延伸阅读.md) | 文献与延伸资源 |

> **建议先读 14 章。** 如果你想快速判断这份文档值不值得读，第 14 章是最能体现差异化的部分——它列的都是别处不写、但会安静地给出错误结果的地方。

### 第二部分：进阶（已完成）

| 章节 | 主题 |
|------|------|
| [08 分子表征总论](./docs/zh/08-分子表征总论.md) | 从 1D 到学习型表征，以及怎么选 |
| [09 化学反应处理](./docs/zh/09-化学反应处理.md) | Reaction SMARTS 与虚拟库枚举 |
| [10 构象生成与 3D 处理](./docs/zh/10-构象生成与3D处理.md) | ETKDG、力场优化与构象聚类 |
| [11 分子骨架与 R 基团分解](./docs/zh/11-分子骨架与R基团分解.md) | Murcko 骨架、R 基团与 MMPA |
| [12 化学数据批量工程](./docs/zh/12-化学数据批量工程.md) | PandasTools 与化学空间可视化 |
| [13 完整 Pipeline](./docs/zh/13-完整Pipeline.md) | 从原始文件到模型输入 |

### 附录

- [附录 B：参考来源与延伸阅读](./docs/zh/附录B-参考来源与延伸阅读.md)
- 附录 A：化学信息学术语中英对照表（计划中）

## 快速开始

```bash
# 推荐用 conda 安装（最稳定）
conda create -n chem python=3.11
conda activate chem
conda install -c conda-forge rdkit

# 或者 pip
pip install rdkit
```

```python
from rdkit import Chem

# 关键：MolFromSmiles 对无效 SMILES 返回 None，必须检查
mol = Chem.MolFromSmiles("CCO")
if mol is None:
    raise ValueError("无效的 SMILES")

print(mol.GetNumAtoms())  # 3
```

更详细的环境说明见 [02 章](./docs/zh/02-分子读写与文件格式.md)。

## 当前进度

**第一、二部分与第 14 章均已就绪**（00–13、14、附录 B）。

待补：附录 A 术语中英对照表；英文精选版（范围见 [docs/en/](./docs/en/)）。若你希望优先看到某一部分，欢迎开 issue 说明。

## 贡献

欢迎提交 issue 和 PR。按照以下原则：

1. **事实准确性优先。** 化学信息学里一个错误的默认参数可能导致整批数据出错，所以涉及 API 行为、默认值、版本差异的内容请附上验证方式。
2. **代码必须可运行。** 提交的片段需要能在 RDKit 2025.09.x 上直接跑通。
3. **注明版本。** RDKit 的 API 在版本间变动较大，请说明你所用的版本。
4. **不要提交任何来自雇主或客户的化合物数据、结构或内部资料。** 示例一律使用公开可得的分子。

## 许可

- 文档内容（`*.md`、图表）：[CC BY-SA 4.0](./LICENSE)
- 代码示例（`*.py`）：[MIT](./LICENSE-CODE)

你可以自由复制、转载、改编，但需保留署名，且衍生作品需以相同方式共享。

## 致谢

内容的准确性建立在 RDKit 开源社区的工作之上。RDKit 由 Greg Landrum 及众多贡献者维护，采用 BSD 3-Clause 许可。

---

*本项目与 RDKit 官方无隶属关系。RDKit 名称与标识归其各自所有者所有。*
