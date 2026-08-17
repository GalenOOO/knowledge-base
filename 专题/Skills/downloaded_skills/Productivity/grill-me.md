# grill-me

本文档介绍社区 Skill `grill-me`：能力、来源与安装方式。

| 项目 | 内容 |
|------|------|
| **作者** | Matt Pocock |
| **GitHub** | https://github.com/mattpocock/skills |
| **skills.sh** | https://skills.sh/mattpocock/skills/grill-me |
| **Stars** | 219,669 |
| **安装量** | 876.3K |

动手前对计划或设计做拷问，直到决策树每个分支都谈清楚。由用户手动调用（`/grill-me`）；实际面试由配套 Skill `grilling` 执行：按轮次问当前可决的问题，每题给出推荐答案，事实自己查、决策留给用户。谈完前不开始动手。

`grill-me` 只负责唤起会话，安装时需同时带上 `grilling`：

```bash
npx skills add https://github.com/mattpocock/skills --skill grill-me --skill grilling
```
