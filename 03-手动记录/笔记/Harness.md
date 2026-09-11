Agent = Model + Harness

Harness是围绕基础模型构建的系统，负责编排执行过程，并决定模型的思考与规划方式、工具调用与行动方式、上下文感知与管理方式、工件存储方式以及结果评估方式。

## Harness的目的是什么？
1. 给Agent配置足够的基础组件，使其具备完成复杂任务的能力。
2. 让Agent更合理的利用其组件，受控执行任务，最大程度提升完成任务的成功率。

如何达到这个目标：
1. 配置基础能力组件：ReactLoop、Tools、Memory、Context ManagerManager、Subagent。
2. 受控执行：Sandbox、文件系统、权限控制。
3. 监控和反馈：Agent的执行过程可观测、结果可验证、故障可恢复。（Middleware、Hooks 等辅助组件，确保 Agent 执行环境的安全和稳定）

## Harness组件：
基础组件：
1. ReactLoop：循环执行 Reasoning + Acting 直到完成任务
2. Tools：执行外部动作（搜索、查数据库、调 API、运行代码等），对应一次工具调用
3. Memory：记录之前的交互历史，作为上下文提供给模型。还应该包括知识库，skill 等。
4. Context ManagerManager：根据任务类型（如单轮、多轮）和模型能力（如最大上下文长度），智能管理上下文，避免超出限制
5. Subagent：支持创建子 Agent 处理子任务，实现复杂任务分解

辅助组件：
1. Sandbox：在安全环境中执行外部动作，防止恶意代码执行
2. 文件系统：Agent 可以读写文件，实现任务分解和结果存储
3. Middleware：提供额外功能，如日志记录、性能监控、错误处理等
4. Hooks：在关键位置插入自定义逻辑，如预处理输入、后处理输出等

## Harness能力
1. 针对任务合理编排流程：任务编排（单Agent还是多Agent，多Agent，需要哪些子Agent，采用何种协作模式）。
2. 观测结果，验证结果，根据结果调整执行策略。
3. 记忆能力：记录之前的交互历史，作为上下文提供给模型。还应该包括知识库，skill 等。
4. ……
---

TO BE CONTINUED

