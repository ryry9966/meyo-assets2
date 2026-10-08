---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40893
source: 综合讨论
publishedAt: 2026-10-08
---

# 背景

让 Agent 在本地干活，最大的摩擦往往不是模型能力，而是环境认知。同一份任务提示词，在我的 macOS 上跑得通，到同事的 Linux 服务器上就报错：包管理器不同、Python 版本不同、venv 路径不同，还有代理问题。Agent 的默认行为是"试探"——先猜一个命令，失败再换一个。代价小的时候无所谓，但在真实项目里，猜错一次可能就是装错 venv、污染全局依赖，甚至动了不该动的配置文件。

社区里常见的临时做法是把环境信息每次贴进对话，或者写进 system prompt。前者是重复劳动，后者和机器绑定、没法随仓库分发。tools.md 就是为了解决这件事：把"这台机器、这个项目怎么干活"写成一份 Agent 每次开工都会读的契约文件。

# 问题拆解

实际用下来，环境差异集中在四类：

1. **工具链差异**：brew vs apt、npm vs pnpm、Python 3.10 vs 3.12；
2. **路径差异**：venv 位置、全局安装的 CLI、容器内外的路径映射；
3. **流程差异**：构建、测试、lint 的标准命令；
4. **禁区**：哪些目录不能动、哪些操作永远不要执行。

# 做法

推荐"三层文件 + 一个校验脚本"的结构：

- **全局层** `~/.openclaw/tools.md`：用户级默认值，比如常用 shell、代理、个人习惯（"不要自动 git push"）；
- **仓库层** `tools.md`：随仓库提交，写项目工具链和标准命令，优先级高于全局层；
- **机器层** `tools.local.md`：进 gitignore，只放这台机器特有的路径覆盖。

仓库层要满足三个标准：每条命令可直接复制执行；只写事实，不写散文；总长控制在 100 行以内，细节外链到 docs。一个精简骨架：

```
## 环境
- OS: macOS 15, shell: zsh
- Node: 用 pnpm 9（不要用 npm）
- Python: uv 管理，venv 在 .venv/

## 标准命令
- 测试: uv run pytest -x -q
- lint: uv run ruff check .

## 禁区
- 不要修改 .github/ 下任何文件
- 不要全局安装任何包
```

机器相关的差异不要硬编码进 tools.md，而是写一个 `scripts/doctor.py`，负责探测实际环境并校验是否与文件声明一致。Agent 每次开工先跑 doctor，不一致就报告，而不是带着过期信息硬干。

# 踩坑点

1. **写成文档而不是命令**。Agent 需要可执行的事实，"我们使用 pnpm"不如 `pnpm install` 有用。
2. **文件过期**。机器升级后没更新，Agent 会信任旧文件继续踩坑。对策：doctor 校验 + 文件头注明最近校验日期，CI 里也跑一次。
3. **塞 secrets**。tools.md 会被提交、会被读进上下文，密钥一律走环境变量。
4. **太长**。文件本身消耗上下文，超长反而降低指令遵从质量。
5. **修正不回写**。口头上告诉 Agent"这台机器用 pnpm"，下个会话就忘了。任何一次纠正都应落回 tools.md。

# 可复用建议

- **让 Agent 自维护**：在提示词里约定"发现环境事实与 tools.md 不符时，先确认，再提议更新文件"，形成闭环；
- **明确优先级**：仓库层覆盖全局层，把这条规则写在全局文件第一行；
- **与 MCP 配合**：环境探测优先做成 MCP tool，tools.md 作为兜底契约，两者不重复维护；
- **像 review 代码一样 review tools.md**，环境变更走 PR。

# 总结

tools.md 的本质，是把"人知道、机器有、Agent 不知道"的信息显式化。它不解决所有环境问题，但把最高频的猜测成本降到零。写一次，团队和所有会话受益；过期了就更新，当成代码来维护。这件事不性感，但很值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/1217b138859572db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/6619b176004bcc32.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/cbcbc646283c7eb6.png)

