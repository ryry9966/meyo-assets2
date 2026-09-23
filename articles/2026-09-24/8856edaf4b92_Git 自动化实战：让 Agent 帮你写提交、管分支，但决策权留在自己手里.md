---
title: Git 自动化实战：让 Agent 帮你写提交、管分支，但决策权留在自己手里
feedId: 38663
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 这类带 shell 执行能力的 agent，天然适合接管 Git 里最重复的部分：写 commit message、按规范建分支、清理已合并分支。我把自己仓库的提交流程交给 agent 跑了两个月，先说结论：可行，但前提是把权限和流程卡死。

## 问题

直接丢一句"帮我提交"，常见翻车有三种：

1. **message 是幻觉**：agent 写的提交信息和实际 diff 对不上，描述的是"它以为你做了什么"；
2. **危险操作**：`reset --hard`、`push --force`、对 main 分支的误写，一次就够喝一壶；
3. **脏东西入库**：`.env`、日志、构建产物被一句 `git add .` 全带进去。

## 做法

我的方案分三层：约束、执行、确认。

**第一步，约束层写进系统提示或技能定义：**

- 只允许 `status / diff / branch / log / checkout -b / commit` 和白名单路径内的 `add`；
- 明确禁止 `push --force`、`reset --hard`、对保护分支的任何写操作；
- commit message 强制 Conventional Commits 格式，且每条必须对应 diff 里的真实变更。

**第二步，执行层固化成流程（可封装为 MCP tool 或 skill 脚本）：**

1. `git status` + `git diff --staged` 先拿到事实；
2. agent 根据暂存 diff 生成 message，先输出给用户过目；
3. 用户确认后才执行 `git commit`；
4. 分支管理同理：建分支走 `feat/xxx`、`fix/xxx` 命名规范；清理分支只产出报告，不直接删。

**第三步，确认层靠人：** commit、建分支、任何远程操作，一律显式确认。自动化的是重复劳动，不是决策权。

## 踩坑点

- **staged 和 unstaged 混淆**：agent 看 `git diff` 时容易把未暂存内容算进提交描述。流程里固定先看 `--staged`，`add` 由它执行但路径必须过白名单。
- **大 diff 撑爆上下文**：上千行的 diff 会让 message 质量明显下降。拆分提交，或让它先做文件级摘要再细化。
- **并行会话互踩**：两个会话同时操作一个仓库，切分支会互相干扰。同一仓库同一时间只给一个会话写权限。
- **钩子被绕过**：本地 commit-msg 钩子能校验格式，但 agent 用 `--no-verify` 就跳过了。把"禁止 --no-verify"写进约束，并在执行层校验命令参数。

## 可复用建议

- 把 Git 操作封装成**窄权限独立工具**，而不是开放全量 shell——工具越窄，越敢放开自动化。
- 所有写操作前先输出"将要执行的命令 + 预期结果"，把 dry-run 变成默认习惯。
- 分支清理报告、stale 分支扫描这类**只读任务可以完全自动化**；写操作永远留一道人工确认。
- 用 pre-commit 钩子做最后防线（密钥扫描、格式检查）。不信任提示词约束，信任钩子。

## 总结

Agent 管 Git 的正确姿势不是"替你决定"，而是"替你执行"。规则写成约束，流程固化成步骤，确认留给自己。做到这三点，提交信息和分支卫生这类琐事，确实可以放心交出去——而且出了问题，回滚路径清清楚楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/3bca9688e9a96eea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9c16362efba95706.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/af35ff3f8217857a.png)

