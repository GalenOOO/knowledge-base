# Skill 类别说明

本文件介绍 `downloaded_skills` 和 `generated_skills` 两个目录下共用的 7 个 Skill 类别，分类依据来自 find-skills 技能规范。

## 目录结构

```
downloaded_skills/        ← 社区 Skill 的介绍文档
generated_skills/         ← 自己创建/生成的 Skill
├── Web Development/
├── Testing/
├── DevOps/
├── Documentation/
├── Code Quality/
├── Design/
└── Productivity/
```

## 两大目录的用途

| 目录 | 用途 | 来源 |
|------|------|------|
| **downloaded_skills** | 存放社区 Skill 的介绍文档（不保存 Skill 整包） | 从 skills.sh、GitHub 收录后撰写 |
| **generated_skills** | 存放自己创建或用 skill-creator 生成的自定义 Skill | `npx skills init`、手动编写等 |

## 7 个类别详解

### 1. Web Development（Web 开发）

- **搜索关键词**：react, nextjs, typescript, css, tailwind
- **覆盖范围**：前端框架与库、样式系统、组件设计、性能优化、SSR/SSG 等
- **典型 Skill 示例**：
  - `frontend-design` — 前端设计最佳实践（110K+ 安装量）
  - `web-design-guidelines` — 247 条网页设计指导方针（107K+ 安装量）
  - `react-best-practices` — React/Next.js 性能优化指南

### 2. Testing（测试）

- **搜索关键词**：testing, jest, playwright, e2e
- **覆盖范围**：单元测试、集成测试、端到端测试、TDD 工作流、测试覆盖率
- **典型 Skill 示例**：
  - `test-driven-development` — 红绿重构 TDD 流程
  - `pypict-claude-skill` — 用 PICT 生成成对组合测试用例
  - `playwright` — Playwright 浏览器自动化测试

### 3. DevOps（运维与部署）

- **搜索关键词**：deploy, docker, kubernetes, ci-cd
- **覆盖范围**：CI/CD 流水线、容器化部署、基础设施即代码、监控告警
- **典型 Skill 示例**：
  - `aws-skills` — AWS CDK 最佳实践与无服务器架构
  - `elastic/agent-skills` — Elasticsearch/Kibana 运维能力

### 4. Documentation（文档）

- **搜索关键词**：docs, readme, changelog, api-docs
- **覆盖范围**：API 文档生成、变更日志、README 编写、技术文档规范
- **典型 Skill 示例**：
  - `changelog-generator` — 从 Git 提交历史自动生成变更日志
  - `docx/pdf/pptx/xlsx` — 各类文档格式处理（Anthropic 官方）

### 5. Code Quality（代码质量）

- **搜索关键词**：review, lint, refactor, best-practices
- **覆盖范围**：代码审查、静态分析、重构建议、编码规范、安全审计
- **典型 Skill 示例**：
  - `code-review` — GitHub PR 自动化审查
  - `move-code-quality-skill` — Move 语言代码质量检查
  - `requesting-code-review` — 任务间代码审查流程

### 6. Design（设计）

- **搜索关键词**：ui, ux, design-system, accessibility
- **覆盖范围**：UI/UX 设计规范、设计系统、无障碍访问、视觉风格指导
- **典型 Skill 示例**：
  - `frontend-design` — 禁止"AI 感"设计，强制明确设计方向
  - `Impeccable` — 前端设计威力加强版，含 20 个斜杠命令和反模式库
  - `ui-ux-pro-max-skill` — UI/UX 专业设计指导

### 7. Productivity（效率）

- **搜索关键词**：workflow, automation, git
- **覆盖范围**：工作流自动化、Git 操作、任务管理、开发效率提升
- **典型 Skill 示例**：
  - `superpowers` — 完整开发工作流引擎（头脑风暴→计划→开发→审查→收尾）
  - `using-git-worktrees` — Git Worktree 独立环境工作流
  - `brainstorming` — 需求澄清与头脑风暴

## 使用建议

1. **按需安装**：不要装太多 Skill，每个都会占用 Agent 上下文空间
2. **分类存放**：社区 Skill 的介绍放入 `downloaded_skills` 对应类别，自建的放 `generated_skills`；整包用 `npx skills add` 装到 Agent，不入库
3. **安全审查**：安装前阅读 SKILL.md，检查是否涉及文件写入、命令执行等敏感操作
4. **定期清理**：用 `npx skills check` 检查更新，移除不再使用的 Skill
