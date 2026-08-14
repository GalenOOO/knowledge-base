# 优秀 Skill 收录

本文档收录经过社区验证的高质量 Agent Skill，按领域分类整理。

---

## 📚 学术研究

### claude-scientific-skills（⭐ 16,989 Stars）

| 项目 | 内容 |
|------|------|
| **作者** | K-Dense-AI |
| **GitHub** | https://github.com/K-Dense-AI/claude-scientific-skills |
| **描述** | 138+ 科学技能的巨型集合，覆盖生物信息、化学、医学、AI 等，现已更名为 Scientific Agent Skills，兼容所有支持开放 Agent Skills 标准的 AI 工具 |
| **安装** | `npx skills add https://github.com/K-Dense-AI/claude-scientific-skills` |
| **安装量** | 13.9K+ |

**论文相关技能**：

| 技能名称 | 功能 |
|----------|------|
| `scientific-writing` | 深度研究 + 学术写作（核心技能） |
| `literature-review` | 文献综述系统化流程 |
| `hypothesis-generation` | 假设生成 |
| `research-grants` | 基金/提案写作 |
| `statistical-analysis` | 统计分析（数据分析必备） |
| `presentation-skills` | 学术汇报 |

**集成数据库**：28+ 科学数据库（OpenAlex、PubMed、ChEMBL、UniProt、COSMIC、ClinicalTrials.gov 等）

**覆盖领域**：
- 🧬 生物信息学与基因组学（序列分析、单细胞 RNA-seq、基因调控网络、变异注释）
- 🧪 化学信息学与药物研发（分子属性预测、虚拟筛选、ADMET 分析、分子对接）
- 🔬 蛋白质组学与质谱（LC-MS/MS 处理、肽段鉴定、蛋白质定量）
- 🏥 临床研究与精准医疗（临床试验、药物基因组学、临床决策支持）
- 🧠 医疗 AI 与临床 ML（EHR 分析、生理信号处理、医学影像）
- 🖼️ 医学影像与数字病理（DICOM 处理、全切片图像分析、计算病理学）
- 🤖 机器学习与 AI（深度学习、强化学习、贝叶斯建模、模型可解释性）
- 🔮 材料科学与化学（晶体结构分析、相图、计算化学）
- 🌌 物理与天文学（天文数据分析、宇宙学计算、符号数学）
- ⚙️ 工程与仿真（离散事件仿真、多目标优化、系统建模）
- 📊 数据分析与可视化（统计分析、网络分析、出版级图表）
- 🌍 地理空间科学与遥感（卫星影像处理、GIS 分析、空间统计）
- 🧪 实验室自动化（液体处理协议、实验设备控制、LIMS 集成）
- 📚 科学交流（文献综述、同行评审、科技写作、海报设计、幻灯片）
- 🔬 多组学与系统生物学（多模态数据整合、通路分析、网络生物学）
- 🧬 蛋白质工程与设计（蛋白质语言模型、结构预测、序列设计）
- 🎓 研究方法论（假设生成、科学头脑风暴、批判性思维、基金写作）

**安装特定技能示例**：
```bash
npx skills add https://github.com/K-Dense-AI/claude-scientific-skills --skill scientific-writing
npx skills add https://github.com/K-Dense-AI/claude-scientific-skills --skill literature-review
npx skills add https://github.com/K-Dense-AI/claude-scientific-skills --skill statistical-analysis
```

**配套工具**：K-Dense BYOK — 免费开源的 AI 科学家桌面应用，支持 40+ 模型、Web 搜索、文件处理、100+ 科学数据库，数据保留在本地。

---
