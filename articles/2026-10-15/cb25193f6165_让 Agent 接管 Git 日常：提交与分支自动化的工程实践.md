---
title: 让 Agent 接管 Git 日常：提交与分支自动化的工程实践
feedId: 41602
source: 综合讨论
publishedAt: 2026-10-15
---

用过 AI 编程助手的都知道：代码是助手写的，提交却是自己一个个敲的。一天下来十几个人工 `git commit`，信息质量参差不齐，分支开完忘删。这篇文章记录我在 OpenClaw 工作流里把 Git 日常交给 Agent 托管的做法，重点在权限收敛和踩坑，不是"全自动无人仓库"。

## 问题

实际痛点集中在三处：

1. **提交信息不可控**。Agent 一次改动常跨多个文件，要么揉成一个大提交，要么信息写得像流水账，回溯时没用。
2. **分支泛滥**。每个任务开一个分支，任务结束没人清理，`git branch` 列表越来越长。
3. **危险操作风险**。Agent 能执行 shell，意味着 `reset --hard`、`push --force`、`clean -fd` 都在它的能力范围内，出一次事故就够了。

## 做法

**第一步：收敛权限。** 我没有让 Agent 直接拿完整 shell，而是通过 MCP 挂了一个 Git 工具集，默认只暴露 status / diff / log / add / commit / branch 这类低危命令，push 走独立审批。凭据走 ssh-agent，不让 Agent 接触 token 明文。

**第二步：把规范写进配置而不是对话。** 在项目级的 Agent 指令文件里固化几条硬规则：

- 提交信息只允许基于真实 diff 撰写，禁止推测改动内容
- 使用 Conventional Commits（`feat:` / `fix:` / `chore:`）
- 分支命名 `feat/xxx`、`fix/xxx`，任务开始时建，任务结束提醒合并
- 禁止 `git add -A`，必须显式列出文件
- 禁止 force push、禁止直接在 `main` 上提交

**第三步：提交流程拆细。** 实际跑起来的流程是：Agent 先跑 `status` + `diff` → 汇报将要提交的文件 → 按模块拆成多个提交并生成信息 → 我扫一眼 diff 摘要确认 → 它执行 commit。push 由我手动触发，或者由一个包装脚本代理并记录日志。

**第四步：分支清理交给脚本。** 已合并分支的删除用一条封装好的脚本定期跑（`git branch --merged` 过滤后逐个确认删除），Agent 只负责调用和汇报结果，不给它裸的删除命令。

**第五步：留痕。** Agent 执行过的每条 git 命令落一份日志文件，出问题能回放，也能发现它有没有尝试 denylist 里的操作。

## 踩坑点

- **`add -A` 提交了 `.env` 和构建产物**。光靠 `.gitignore` 不够，加了 pre-commit 钩子做密钥扫描，同时规则里明确"只提交任务相关文件"。
- **提交信息幻觉**。早期它会在 message 里描述 diff 中根本不存在的改动。改成强制先输出 diff 摘要、再写信息，情况明显好转。
- **交互式命令挂起**。`rebase -i`、需要拉起编辑器的命令在非交互 shell 里会卡住 Agent，这类命令要么禁用，要么换成非交互等价物。
- **脏工作区切分支**。它有次在未提交改动下直接 checkout，把我的临时改动搅在一起。现在规则是：切分支前必须确认工作区干净，或先 stash。

## 可复用建议

1. **权限渐进**：只读一周 → 开放本地写 → push 审批，观察它每一步的行为再逐步放开。
2. **危险操作包脚本**，不暴露原始命令，脚本里内置确认和日志。
3. **规范放项目配置文件**，别指望 Agent 记住上次对话说过什么。
4. **让 Agent 提案、人做决定**：提交信息、拆分方案、分支命名都可以自动化，push 和 merge 的扳机留给自己。

## 总结

这套流程跑了一个多月，提交信息质量和分支卫生明显改善，我的 git 操作频率降了一大半。核心经验一句话：**把机械的部分交给 Agent，把不可逆的扳机留在人手里**。自动化程度越高，权限边界越要画得清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/488ad51eab4746bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/b81f09bce0991739.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/9798c57f54d60cf5.png)

