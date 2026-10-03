---
title: Git 自动化实战：让 Agent 接管提交信息与分支清理，但把闸门留给自己
feedId: 40273
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

日常开发里 Git 的"机械部分"占比不小：写 commit message、开分支、清理已合并分支、拼 changelog。这些活儿规则清晰但繁琐，正好是 Agent 配合工具调用（MCP / 插件）能接管的范围。最近我把一部分 Git 例行操作交给 OpenClaw 跑了两三周，记录一下做法和安全边界。

## 问题

- commit message 全靠随手写，`fix`、`update` 满天飞，回溯时基本没用。
- 已合并分支越积越多，没人敢删。
- 周报和变更日志要手动翻 log 拼凑。

但直接让 Agent 全权执行 git 命令也不行：`push --force`、`reset --hard` 一旦误触发，代价是真实丢代码。

## 做法

1. **收窄工具面**：不要把裸 shell 交给 Agent。我封了一层 MCP 工具，白名单只有 `status / diff / log / add / commit / branch / switch`，push 单独走一个需要人工确认的入口；`--force`、`reset --hard` 在封装层直接禁掉。
2. **把约定写进项目记忆**：在仓库的 AGENTS.md（或等价的项目级说明）里写死规范——Conventional Commits 格式、分支命名 `feat/*` `fix/*`、什么情况下允许直接提交到当前分支。Agent 每次会话都会读到。
3. **三个先跑通的流程**：
   - **提交信息**：Agent 读 staged diff，产出符合规范的 message，先展示给我确认，再执行 `commit`。
   - **分支清理**：Agent 用 `git branch --merged` 列出候选，我确认后批量 `-d`（注意用小写 `-d`，不用 `-D`）。
   - **变更摘要**：定时任务让它汇总一段时间内的 `git log`，产出内部 changelog 草稿。
4. **留审计**：封装层把 Agent 执行过的每条 git 命令追加到日志文件，出问题能回放。

## 踩坑点

- **diff 太长喂不下**：大改动一次塞给模型会截断，它就开始"看文件名猜意图"。改成按文件分片读 diff，并要求 message 引用关键 hunk，编造概率明显下降。
- **`add -A` 是事故源**：本地配置、临时密钥很容易被一把梭进去。改成 Agent 先列变更清单、逐个 stage，pre-commit 里再加 secrets 扫描兜底。
- **别碰共享分支**：rebase、force push 只允许发生在自己的 feature 分支上——这条要做成封装层的硬规则，不能只靠提示词。
- **脏状态先检测**：Agent 偶尔会在 rebase 中途或 detached HEAD 下继续操作。封装层第一步先跑 `git status --porcelain`，状态不干净就拒绝一切写操作。
- **hook 失败别绕过**：pre-commit 挂了就让 Agent 把报错原样贴出来，禁止 `--no-verify` 闯关。

## 可复用建议

- **灰度顺序很重要**：先只开放只读操作（log 摘要、分支盘点），稳定后再放开 commit，最后才是受控 push。
- **"Agent 写、人来推"是条省心的分界线**：commit 可以自动，push 永远留一道人工确认。
- **规范放仓库里，不放对话里**：约定跟着 repo 走，换会话、换人都生效。
- **多人使用时把封装层做成共享工具**，而不是各自为政，否则审计日志没有意义。

## 总结

AI 助手接管 Git 的价值不在"替你打字"，而在把团队约定变成每次提交都会被执行的硬规则。工程上抓住三点：收窄命令面、写操作留闸门、全程留日志。做到这些，这套流程基本可以长期跑，而且越跑越省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/c3021cfe37d81893.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/b212c0c045872147.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/feed865334393fd3.png)

