---
title: Git 自动化：让 Agent 管提交和分支，但护栏要先装好
feedId: 39597
source: 综合讨论
publishedAt: 2026-09-29
---

### 背景

团队里最稳定的脏活就是 git：写 commit message、删已合并分支、整理本地堆了三周的 feature 分支。这些活不难，但琐碎且容易敷衍。OpenClaw 这类常驻 agent 加上 MCP git 工具或 shell 执行能力之后，第一次让人觉得"这活可以外包给 AI"——前提是先想清楚让它能做什么、不能做什么。

### 问题

直接把 git 权限丢给 agent，通常出三类事：

1. **提交质量不可控**：agent 图省事 `git add .` 一把梭，message 全是 "update files"；
2. **危险命令误触**：`reset --hard`、`push --force`、`clean -fd`，一次手滑就是半天工作量；
3. **边界感缺失**：在 main 上直接提交、把 .env 一起 commit、合并冲突时自作主张 rebase。

### 做法

我的落地分四步，成本不高，基本堵住了上面的问题。

**1. 最小权限接入。** 用 MCP git server 或给 agent 配专用 shell 环境：独立 git 身份（便于区分人提交和 agent 提交）、只在项目工作区内操作。初始能力只给只读命令：`status`、`diff`、`log`、`branch -a`。

**2. 提交流程两段式。** 第一步 agent 只输出：改动摘要 + 生成的 conventional commits 格式 message（如 `feat(auth): ...`），不执行。我确认后回一句"提交"，它才 `git add`（明确指定文件，不用 `add .`）和 `git commit`。push 永远单独确认。

**3. 分支清理定期化。** 每周让 agent 跑一次报告：已合并到 main 的本地分支、超过两周没动的 stale 分支。删除已合并分支可以直接授权，未合并分支只报告、不动手。

**4. 护栏脚本兜底。** 写一个 git 包装脚本：白名单放行安全命令，命中黑名单（force push、hard reset、`checkout -- .` 等）直接拒绝并提示人工介入。同一份规则写进 agent 的常驻指令，双保险。

### 踩坑点

- **diff 太长会降智**：几百行以上的 diff，生成的 message 会泛化甚至幻觉。后来我限制单次提交粒度，大改动先让它分模块总结、再写 message。
- **合并冲突必须停机**：agent 自己解的冲突看着对、逻辑错。规则改成：遇到 conflict 立即中止，输出冲突文件清单等人处理。
- **钩子别省**：agent 会跳过你脑内的检查，pre-commit 的 secret 扫描和 lint 是最后一道网，不能依赖 agent 自觉。
- **分支保护别靠口头约定**："我让它别动 main"撑不过两周，`main` 设了保护才敢放心让它跑。

### 可复用建议

- 权限按"只读 → 写本地 → push"逐级放开，每级观察一两周再加码。
- 把提交规范、黑名单、确认流程沉淀成一份规则文件放进仓库（配合 AGENTS.md 类约定），团队任何人接 agent 都能复用同一套。
- agent 执行的每条 git 命令落日志，出问题能回放、能审计。

### 总结

agent 管 git 的价值在琐碎事务：message 质量、分支卫生、定期巡检。关键决策——push、删未合并分支、解冲突——留在人手里。先装护栏，再谈自动化。这套流程跑了一个多月，提交历史干净了不少，也没出过需要回滚的事故；这套思路迁移到其他高危工具（数据库、部署脚本）上，同样适用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/35548edf7b5f9dd3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/13cb8367f1ea3063.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/43a324b7a5ecbaf5.png)

