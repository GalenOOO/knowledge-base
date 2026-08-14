# Skill 查找网站与安装指南

## 一、查找 Skill 的网站

### 1. skills.sh（官方推荐）

- **网址**：https://skills.sh/
- **说明**：Vercel Labs 推出的官方 Skill 发现平台，是 Skill 生态的"官方商店"。提供排名、安装量统计、分类浏览等功能。几乎所有 Skill 的推荐安装方式都使用 `npx skills` 命令。
- **特点**：
  - 安装量排行榜，可查看最热门的 Skill
  - 按领域分类浏览
  - 每个 Skill 页面附带安装命令
  - 与 `npx skills` CLI 深度集成

### 2. agentskill.sh

- **网址**：https://agentskill.sh/
- **说明**：拥有 87,000+ 个 Skills，分类较细，开发、数据、营销、DevOps 都有单独入口。
- **特点**：
  - 有 Trending 榜单，可查看最近热门 Skill
  - 支持 Claude Code、Cursor、Copilot 等主流 Agent
  - 安装命令现成，复制粘贴即可

### 3. SkillsMP

- **网址**：https://skillsmp.com/
- **说明**：体量更大，35 万+ Skills，数据从 GitHub 实时同步。
- **特点**：
  - 搜索支持语义匹配，不用精确关键词也能找到
  - 除 Claude Code 外还支持 OpenAI Codex CLI 和 ChatGPT
  - 按热度或分类浏览
  - 数量大但质量参差不齐，需要耐心筛选

### 4. ClawHub

- **网址**：https://clawhub.ai/
- **说明**：开源 Skills 注册表，定位为 agentskills.io 规范的超集。
- **特点**：
  - 内置版本控制和丰富元数据
  - 支持 Vector 搜索，结果相关性好
  - 有 CLI 一键安装：`clawhub install <owner>/<name>`
  - 500+ Skills，涵盖 coding、data、productivity 等方向

### 5. agentskills.io（规范站）

- **网址**：https://agentskills.io/
- **说明**：Agent Skills 的官方规范网站，定义了 Skill 的开放标准。
- **特点**：
  - 完整规范文档：https://agentskills.io/specification
  - 集成指南：https://agentskills.io/integrate-skills
  - 面向开发者和团队，确保 Skill 跨平台兼容

### 6. marketing-skills.com（营销专项）

- **网址**：https://marketing-skills.com/
- **说明**：专门给营销场景准备的 Skills 集合。
- **特点**：
  - A/B 测试设计、广告创意生成、SEO 优化、数据追踪审计等
  - 适合 SaaS 营销或运营人员按需挑选

---

## 二、查找 Skill 的 GitHub 仓库

### 1. anthropics/skills（官方示范库）

- **地址**：https://github.com/anthropics/skills
- **说明**：Anthropic 官方维护的 Skill 仓库，提供文档处理（docx/pdf/pptx/xlsx）、前端设计、Skill Creator 等高质量 Skill，也是"Skill 应该怎么写"的参考实现。

### 2. vercel-labs/agent-skills（Vercel 官方）

- **地址**：https://github.com/vercel-labs/agent-skills
- **说明**：Vercel Labs 维护的 Skill 仓库，包含前端设计指南、Web 设计规范、find-skills 等热门 Skill。安装量极高（100K+）。

### 3. ComposioHQ/awesome-claude-skills（高密度导航）

- **地址**：https://github.com/ComposioHQ/awesome-claude-skills
- **说明**：GitHub 上最火的 Skill 合集之一，26,000+ Stars。按场景分类，涵盖文档处理、开发工具、数据分析、商业营销、创意媒体等 60+ 使用场景。适合快速查找。

### 4. obra/superpowers（工作流引擎）

- **地址**：https://github.com/obra/superpowers
- **说明**：27,000+ Stars。不是单个 Skill，而是一整套开发工作流，把软件开发拆成多个阶段，每个阶段对应一个 Skill：头脑风暴 → Git Worktree → 写计划 → Subagent 开发 → TDD → 代码审查 → 分支收尾。适合中大型项目。

### 5. 其他值得关注的开源仓库

| 仓库 | 说明 |
|------|------|
| [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | Skills 数量多，覆盖面广 |
| [BehiSecc/awesome-claude-skills](https://github.com/BehiSecc/awesome-claude-skills) | 更新频繁，追踪最新 Skill |
| [VoltAgent/awesome-claude-skills](https://github.com/VoltAgent/awesome-claude-skills) | 偏向 Agent 自动化工作流 |
| [K-Dense-AI/claude-scientific-skills](https://github.com/K-Dense-AI/claude-scientific-skills) | 专注科学计算领域 |
| [JimLiu/baoyu-skills](https://github.com/JimLiu/baoyu-skills) | 偏中文内容创作（小红书、公众号等） |
| [elastic/agent-skills](https://github.com/elastic/agent-skills) | Elastic 官方 Skill（ES|QL、Kibana 等） |

---

## 三、如何安装 Skill

### 方法一：使用 npx skills CLI（推荐）

这是最通用的安装方式，由 Vercel Labs 维护，支持 40+ AI Agent。

#### 前提条件

- 本机已安装 Node.js（建议 LTS 版本），可通过 `node -v` 验证

#### 核心命令

```bash
# 搜索 Skill（交互式或按关键词）
npx skills find [query]

# 安装 Skill
npx skills add <owner/repo>

# 检查更新
npx skills check

# 更新所有已安装的 Skill
npx skills update
```

#### 安装示例

```bash
# 从 GitHub 仓库安装（简写）
npx skills add vercel-labs/agent-skills

# 从完整 GitHub URL 安装
npx skills add https://github.com/vercel-labs/agent-skills

# 安装仓库中的特定 Skill
npx skills add https://github.com/vercel-labs/agent-skills/tree/main/skills/web-design-guidelines

# 从 GitLab 安装
npx skills add https://gitlab.com/org/repo

# 从本地路径安装
npx skills add ./my-local-skills
```

#### 常用选项

| 选项 | 说明 |
|------|------|
| `-g, --global` | 安装到用户目录（全局），而非项目目录 |
| `-a, --agent <agents...>` | 指定目标 Agent（如 claude-code、codex、cursor） |
| `-s, --skill <skills...>` | 按名称安装特定 Skill（用 `*` 安装全部） |
| `-l, --list` | 列出仓库中可用的 Skill，不安装 |
| `--copy` | 复制文件而非创建符号链接 |
| `-y, --yes` | 跳过所有确认提示 |
| `--all` | 安装所有 Skill 到所有 Agent |

#### 进阶用法

```bash
# 列出仓库中可用的 Skill
npx skills add vercel-labs/agent-skills --list

# 安装特定 Skill
npx skills add vercel-labs/agent-skills --skill frontend-design --skill skill-creator

# 安装到特定 Agent
npx skills add vercel-labs/agent-skills -a claude-code -a cursor

# 全局安装（跨项目可用）
npx skills add vercel-labs/agent-skills --skill frontend-design -g -y

# 安装仓库中所有 Skill 到所有 Agent
npx skills add vercel-labs/agent-skills --all
```

#### 管理已安装的 Skill

```bash
# 列出所有已安装的 Skill
npx skills list

# 仅列出全局 Skill
npx skills list -g

# 按关键词搜索 Skill
npx skills find typescript

# 移除 Skill
npx skills remove web-design-guidelines

# 移除全局 Skill
npx skills remove --global web-design-guidelines

# 创建新的 SKILL.md 模板
npx skills init my-skill
```

### 方法二：使用 ClawHub CLI

```bash
# 安装 Skill
clawhub install <owner>/<name>

# 示例
clawhub install dbalve/fast-io
```

### 方法三：手动 Git Clone

```bash
# 克隆仓库
git clone <skill-repo-url>

# 复制到 Agent 的 Skill 目录
# Claude Code 项目级
cp -r skill-folder/* .claude/skills/

# Claude Code 全局级
mkdir -p ~/.claude/skills
cp -r skill-folder/* ~/.claude/skills/
```

### 方法四：在 Claude Code 中使用插件命令

```bash
# 安装官方 Skill 包
/plugin install document-skills@anthropic-agent-skills

# 从市场安装
/plugin marketplace add anthropics/skills

# 查看已安装的 Skill
/skills
```

---

## 四、安装范围选择指南

| 场景 | 推荐范围 | 说明 |
|------|----------|------|
| 仅当前项目使用 | 项目级（默认） | Skill 随项目提交，团队成员共享 |
| 个人跨项目复用 | 全局级（-g） | 所有项目均可使用 |
| 团队统一标准 | 项目级 | 通过 Git 版本控制确保一致性 |

---

## 五、安全自查清单

安装第三方 Skill 前，请检查：

- [ ] 安装量是否 1K+（越高越可信）
- [ ] 来源是否为官方或知名组织（vercel-labs、anthropics、microsoft 等）
- [ ] GitHub 仓库 Stars 是否 100+
- [ ] SKILL.md 内容是否涉及文件写入、网络请求、命令执行
- [ ] 是否有安全检测报告（Gen、Socket、Snyk 等）

---

## 六、快速上手流程

```
1. 搜索 → npx skills find <关键词>  或访问 skills.sh
2. 筛选 → 查看安装量、来源、Stars
3. 审查 → 阅读 SKILL.md 内容，检查安全性
4. 安装 → npx skills add <owner/repo> --skill <name>
5. 验证 → 在 Agent 中触发对应场景，确认 Skill 生效
6. 管理 → npx skills list / npx skills update / npx skills remove
```
