---
title: 让 Agent 接管 Git 日常：提交、分支与护栏的工程化实践
feedId: 42369
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

用 OpenClaw 跑了一段时间任务后，最自然的延伸就是让它接管 Git 日常：写 commit message、整理提交、开分支、生成 changelog。这些活机械、重复，但确实是真实工作量的一部分。问题在于，Git 是不可逆操作的高发区，直接把权限丢给 Agent，出一次事故就够喝一壶。

## 我踩过的坑

最初的用法很粗暴：让 Agent 直接跑 `git add -A && git commit && git push`。三个星期里踩了两次雷：一次把 `.env` 提交进了仓库，一次 commit message 和实际改动完全对不上——Agent 是根据我对话里的描述写的，不是根据 diff 写的。

## 现在的做法

**第一步，先给只读能力。** 通过 git MCP server（或封装一个 CLI skill）只暴露 `status / log / diff / branch` 这类只读命令，跑一两周，确认它对仓库状态的理解是准的，再谈写操作。

**第二步，commit message 走"diff 优先"。** 固定提示词模板：只允许基于 `git diff --staged` 的真实输出生成 Conventional Commits 格式的 message，禁止参考对话上下文里的描述。生成后先打印等人确认，再执行 commit。

**第三步，分支管理立规矩。** Agent 只能从 `main` 拉出 `feat/xxx`、`fix/xxx` 格式的分支，永远不允许直接提交到 `main`。分支命名带上任务 ID，方便回溯是哪次会话开的。

**第四步，用 Git 本身做护栏。** 服务端保护 `main` 分支，本地加 pre-commit hook 拦截 `.env` 和密钥文件；Agent 的命令白名单明确不含 `push --force`、`reset --hard`、`clean -fd`。

## 踩坑点

- **`git add -A` 是最大隐患**。改成 `git add <具体文件>`，文件清单必须来自 `git status` 的实际输出，不允许凭记忆写。
- **交互式命令会卡死会话**。`rebase -i`、需要确认输入的 `merge` 都要规避，用非交互参数替代。
- **commit author 混乱**。Agent 代提交时用的是当前 git 配置，团队里建议约定邮箱别名后缀，PR review 时一眼能认出哪些是 Agent 提交的。
- **长会话状态过期**。Agent 跑了半小时后再 commit，分支可能早被人切走了，每次写操作前强制先跑 `git status` 确认现场。

## 可复用建议

1. 把上述规则写成一个 skill/checklist 文件放进 Agent 工作区，比每次对话口述可靠得多。
2. 涉及远程的写操作（push、开 PR）单独设一层人工确认，本地操作可以放开。
3. 用 `git worktree` 给 Agent 分一个独立工作树，和你的手动操作互不干扰。
4. 留一份 Agent 执行过的 git 命令日志，事后排障全靠它。

## 总结

Agent 管 Git 不是"全自动"，而是"它干活、你守门"。只读先行、diff 优先、白名单兜底——这三条原则守住，日常 80% 的提交和分支工作就可以放心交出去。自动化省下的时间是真的，前提是护栏是钢的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/f2934584d5171b99.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/7757f4daac4bb1f3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/4919c51259f62c26.png)

