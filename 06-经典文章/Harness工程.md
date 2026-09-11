# Harness 工程与自我改进

> 本文档目的：收录 Lilian Weng《Harness Engineering for Self-Improvement》（2026-07）的核心内容——harness 的定义、设计模式、优化方法谱系与开放挑战。

原文：[Harness Engineering for Self-Improvement](https://lilianweng.github.io/posts/2026-07-04-harness/)（Lil'Log，2026 年 7 月）

## 一、核心论点

- **RSI（递归自我改进）** 的近期路径不是模型直接改自己的权重，而是模型改进**部署系统（harness）**，harness 再让模型做得更好，形成循环。
- **Harness 定义**：围绕基座模型的系统，负责编排执行——决定模型如何思考规划、调用工具、感知与管理上下文、存储工件、评估结果。类似操作系统：封装复杂逻辑，保持接口简单。
- Harness 层的重要性接近模型本身的原始智能，证据是 Claude Code、Codex 等成功编码 agent 产品。
- 设计原则：刻意保持简单与通用以利泛化，参考现有软件工程实践以利用预训练知识。

## 二、三大设计模式

### 模式 1：工作流自动化

- 目标导向循环：plan → execute → observe/test → improve，直到达成目标。
- 模型通过 **agent runtime**（而非静态 prompt 模板）分析自己的轨迹与失败并迭代。
- 参考：Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch)、OpenAI Codex agent loop。

### 模式 2：文件系统即持久记忆

- 长任务中工件（实验日志、代码 diff、错误 trace、历史轨迹）增长远超上下文窗口，durable state 放文件而非 context。
- 读写编辑文件系统（通常经 bash）是 LLM 的基础能力，此模式直接受益于核心模型能力提升。

### 模式 3：子代理与后台任务

- 主 agent 派生多个 subagent 并行执行、监控后台任务，隔离子任务以不污染主上下文。
- 关键设计选择：**并行必须显式且可检查**——输出落盘为文件/日志/状态记录，模型才能在中断后恢复并推理自己的执行历史。

## 三、编码 Agent 的工具接口（已趋同）

主流编码 agent（Claude Code、Codex、OpenCode、Cursor 类）的核心接口已稳定：

| 类别 | 工具 |
|------|------|
| 文件系统 | 发现（glob、grep、ls）、读（read）、写改（write、edit、apply_patch 等） |
| Shell | bash、PowerShell |
| IO | lsp、git 工具 |
| 外部上下文 | MCP tools、Skills |
| Web | web_search、web_fetch、browser |
| 工件 | 读写文档、图片，生成 HTML、图片 |
| 后台进程 | Cron 类工具 |
| Agent 委派 | spawn / resume / wait / list / close / interrupt_agent 等 |

## 四、优化对象的演进谱系

优化对象逐级上移：**prompt → 结构化 context → workflow → harness 代码 → optimizer 代码**。模型越强，可优化的对象越复杂、方法越通用。

### Context 工程

- **ACE**（Agentic Context Engineering, Zhang et al. 2025）：context 是演化的 playbook 而非不断变长的 prompt。Generator 产轨迹 → Reflector 蒸馏成败洞察 → Curator 以条目化（identifier, description）增量更新结构化 context，用确定性逻辑合并、定期去重，防止 context 坍塌与简短偏差。
- **MCE**（Meta Context Engineering, Ye et al. 2026）：双层优化——元层演化"管理 context 的技能"（静态组件 + 动态算子），基层优化任务 context 本身。技能即文件目录（skill.md + rollout 数据），meta-agent 做技能交叉。
- **Meta-Harness**（Lee et al. 2026）：直接优化"决定存什么、取什么、给模型看什么"的代码，即"优化 harness 的 harness"。提案者本身是编码 agent，执行历史经文件系统访问，产出 Pareto 前沿上的 harness 候选集合。

### 工作流设计

- 人工设计：AI Scientist（提案 → 实验 → 论文 → 评审流水线）、ScientistOne（可验证性为中心，Chain-of-Evidence 审计）、Autodata（challenger / weak solver / strong solver / verifier 四角色合成"强解弱不解"难度的数据）。
- 搜索设计：**ADAS**（meta-agent 编程新 agent，档案式迭代）、**AFlow**（workflow 建模为图，MCTS 搜索，优于人工设计与 ADAS）。

### 自我改进 Harness

- **STOP**（Zelikman et al. 2023）：递归改进"改进器"本身的早期经典。改进出的改进器发现了遗传算法、多臂 prompt 老虎机、模拟退火、束搜索等策略。
- **Self-Harness**（Zhang et al. 2026）：弱点挖掘（聚类 verifier 接地的失败模式）→ 有界 harness 提案 → 回归验证（held-in 与 held-out 双测无回归才合并）。
- **AHE**（Agentic Harness Engineering, Lin et al. 2026）：以**可观测性**为核心的三支柱——组件可观测（7 个可编辑组件落盘：system prompt、工具描述、工具实现、middleware、skill、subagent 配置、长期记忆）、经验可观测（轨迹 → 每任务根因报告 → 汇总）、决策可观测（每次编辑附预测，是可证伪的声明）。编辑只作用于 harness 工作区，verifier / tracer / 模型配置只读，封堵一类 reward hacking。演化的 harness 冻结后可迁移到 SWE-bench-verified。

### 演化搜索

- 适用条件：搜索空间大且形状怪、难用梯度但易评估。harness 搜索正符合。
- 代表：Promptbreeder（变异 prompt 本身也演化）、GEPA（反思式 prompt 演化）、**AlphaEvolve**（程序池 + LLM 产 diff；`EVOLVE-BLOCK` 标记可编辑区；meta-prompt 共演化）、ShinkaEvolve（父采样平衡、embedding 相似度拒采、meta-scratchpad）、**DGM**（Darwin Gödel Machine，agent 可改自己的 harness 代码库，SWE-bench Verified 20%→50%）。

### 与模型权重联合优化

- **SIA**（Hebbar et al. 2026）：Feedback-Agent 按轨迹决定更新 harness 还是更新权重。证据尚属初步（实验有混淆变量）。
- 更完整的路径是持续学习（continual learning），原文作者认为值得单独成文。

## 五、关键实证发现

1. **递归结构不够，基座能力是核心**：STOP 在 GPT-4 上随迭代改进，在 GPT-3.5、Mixtral 上反而退化。
2. **改 harness 的能力 ≠ 从 harness 获益的能力**（Lin et al. 2026）：前者在 9B 到 Opus 级模型上几乎持平；后者非单调，中档模型获益最多（需要正确及时的技能调用与长程指令跟随）。
3. **内化类比**：类似 prompt engineering——手动技巧随指令微调与推理能力提升而边缘化，但"指定目标、约束、context、评估"的需求不会消失。harness 改进最终可能部分内化进模型行为，但与外部 context 和工具的接口会保留。
4. 演化搜索适合可自动评估、适应度可量化的领域（矩阵乘法、GPU kernel、竞赛、调度），在评估慢、模糊、启发式的领域表现差。

## 六、开放挑战

1. **弱评估器**：研究品味、新颖性、长期科学价值难有快速精确的 verifier——与 RL 的处境相同。
2. **Context 与记忆生命周期**：作者判断 context 工程将从软件系统层变为智能的核心部分。
3. **负结果**：文献偏成功、模型训练数据同样偏成功，导致不擅长放弃假设、承认失败。harness 应让失败尝试易于保留——从失败学习是裁剪搜索空间的最好方式。
4. **多样性坍塌**：演化与 RL 循环趋向利用已知高奖励模式，需要防坍塌机制，对开放式研究尤其关键。
5. **Reward hacking**：自改进循环优化给它的任何信号。**评估器与权限控制必须在演化循环之外**，配合 held-out 测试、trace 审计、关键决策点人工审查。
6. **长期成功**：编码 agent 常能完成手头任务，但对多人长期维护的 repo 健康（可维护性、边界、迁移成本、向后兼容）无优化目标覆盖。
7. **人的角色**：人应上移抽象层级而非出局——在对的时机、对的抽象层做监督，系统设计需预留此类触点。

## 七、自主研究失败模式（Trehan & Chopra 2026）

最小脚手架下从想法到论文的实验观察到六类失败：偏向训练数据默认、执行压力下的实现漂移、记忆与 context 退化（除非日志写成持久工件）、过度乐观（"numerical duct tape" 后宣布胜利）、领域智力不足（缺隐性工艺知识）、科学品味弱（实验可执行但没回答对的问题）。

## 附：相关 Benchmark

| Benchmark | 评什么 |
|-----------|--------|
| PaperBench | 从零复现 ICML 论文（8316 条 rubric；当时最好模型 21%，不敌 ML 博士） |
| CORE-Bench | 计算可复现性（270 任务 / 90 篇论文三学科） |
| ScienceAgentBench | 数据驱动科学发现（102 任务四学科） |
| RE-Bench | 开放式 ML 研究工程 vs 人类专家（2 小时内 agent 4× 胜，8/32 小时人类反超） |
| MLE-bench | 离线 Kaggle（o1-preview + AIDE 达铜牌线 16.9%） |
| KernelBench | GPU kernel 正确性与速度（fast_p 指标） |
| Terminal-Bench-2 | 终端环境长任务（Meta-Harness、Self-Harness、AHE 等的实验场） |
