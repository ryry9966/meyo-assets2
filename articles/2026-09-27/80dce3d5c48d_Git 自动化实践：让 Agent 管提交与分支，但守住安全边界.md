---
title: Git 自动化实践：让 Agent 管提交与分支，但守住安全边界
feedId: 39166
source: 综合讨论
publishedAt: 2026-09-27
---

## 背景

我们几个项目日常由 OpenClaw agent 参与开发。跑通 MCP 工具链之后，最自然的诉求就是：agent 都能读代码、改代码了，能不能顺手把 commit 和分支也管了。实际落地两周左右，分享一下能自动化到什么程度、哪些环节必须守住。

## 问题

手动 Git 操作的痛点其实很具体：

- 提交信息质量不稳定，事后回溯基本靠猜；
- 分支命名随意，`fix-2`、`tmp-final` 这类分支不断堆积；
- 已合并分支忘删，远端越来越乱；
- agent 既然能跑 shell，不加约束就可能随时 `reset --hard` 或 `push --force`。

所以目标不是“全自动”，而是：**agent 做重复劳动，人守安全边界**。

## 做法

**1. 收窄工具面。** 不给全量 shell，通过 MCP git server 或白名单脚本只暴露 `status`、`diff`、`log`、`add`、`commit`、`branch`、`switch`、`merge --no-ff`。显式缺席：`push`、`push --force`、`reset --hard`、`clean`、`--no-verify`。

**2. 提交与分支规则写进仓库文档，而不是 prompt。** Conventional Commits 的 scope 列表、分支命名模板（如 `<type>/<issue-id>-<slug>`）放进 CONTRIBUTING.md，agent 动手前先读文档。规则跟着仓库走，比塞在系统提示里好维护。

**3. 提交前强制 diff 确认。** 固定流程：`git diff --staged` → agent 基于真实 diff 生成提交信息 → 输出改动文件清单和影响摘要 → 等人确认 → commit。push 一律人工执行。

**4. 分支生命周期半自动。** agent 按模板建分支；合并后用 `git branch --merged` 列出可删分支清单，删除动作由人触发。

## 踩坑点

- **幻觉提交信息**：agent 偶尔把上下文里“计划做”的事写进 message，而 diff 里根本没有。解法：要求信息只能引用 staged diff 内容，且先输出改动摘要。
- **秘密泄露**：agent 会顺手 `git add .` 把 `.env` 带上。上 pre-commit secret 扫描，白名单里压根没有 `--no-verify`，想绕也绕不过。
- **在错误分支上干活**：agent 忘了 switch 就往 main 提交。加前置检查：每次 commit 前必须先 `status` + `branch` 确认当前分支。
- **合并冲突静默处理**：让 agent 自动解决冲突风险很高。我们的做法是遇到冲突立即停下，输出冲突文件清单，人工处理。

## 可复用建议

- 把“允许/禁止的 git 命令清单”做成可复用的 MCP 配置或 wrapper 脚本，一个项目调通，其他项目直接抄。
- 提交信息模板化：type、scope、是否关联 issue 全部定死，agent 只填变量，输出质量稳定得多。
- 所有 agent 的 git 动作落日志（时间、分支、diff stat），事后可审计。
- 坚持“agent 起草、人批准、人推送”三段式，push 权限永远不交出去。

## 总结

两周下来，提交信息规范率和分支整洁度明显提升。agent 实际接管的是“读 diff、起名、建分支、列清单”这类机械环节，而真正关键的收窄工具面与人工确认 push 没有让步。Git 自动化的价值不在全自动，而在把人的注意力省下来留给 review。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/76dba6b1667e28ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/bb90a10654547607.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/98fb5a6adf4edda3.png)

