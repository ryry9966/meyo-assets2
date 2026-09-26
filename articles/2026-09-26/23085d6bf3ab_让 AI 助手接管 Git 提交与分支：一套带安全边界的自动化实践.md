---
title: 让 AI 助手接管 Git 提交与分支：一套带安全边界的自动化实践
feedId: 39050
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

团队里真正写代码的时间可能只有一半，另一半耗在机械操作上：切分支、拼 commit message、rebase、清理已合并分支。接入 OpenClaw 这类 agent 之后，这些事理论上都能交给它——前提是你给它划好边界。这篇帖记录我们内部跑了一段时间的 Git 自动化方案，偏工程，可以直接抄。

## 问题

直接让 agent 通过裸 shell 操作 git，我们踩过的坑包括：commit message 风格随机；agent 顺手把不相关的未暂存文件一起 add；在 detached HEAD 上提交导致改动“丢失”；最危险的一次是它自作主张对远端分支 force push。结论很明确：**不能只靠提示词约束，要有结构化的工具层和硬性闸门。**

## 做法

**1. 权限分层。** agent 默认只在 `feature/*` 分支获得写权限，`main` 和 `release/*` 只读。禁用 `push --force`、对远端的 `reset --hard`、`branch -D` 这三类命令，在工具封装层直接拦截，而不是写进 prompt 祈祷它遵守。

**2. 用 MCP git 工具或自封装脚本替代裸 shell。** 可用操作收敛成几个动作：status、diff、stage（显式文件列表）、commit、branch_create、branch_delete（仅本地已合并分支）。每个动作写日志，出了问题能回放。

**3. 提交流程固化成三步：** 先 `status` + `diff --stat` 概览改动范围；再分块看 diff，生成 Conventional Commits 风格的 message（type/scope 从 diff 内容推导，不靠猜）；最后 stage 指定文件并 commit。message 和改动摘要一并输出到会话，人扫一眼再放行。

**4. 分支清理做成定时任务而非即时动作。** agent 每天扫描本地分支，对已合并进 main 的分支生成删除清单，人工确认后批量执行。远端分支一律不动。

**5. pre-commit hook 做最后一道闸：** secret 扫描 + lint。agent 再“聪明”，也不该绕过 hook。

## 踩坑点

- `git add -A` 是重灾区。改成强制显式文件列表后，误提交率基本归零。
- 多行 commit message 的转义在不同 shell 下行为不一致，封装成单参数接口最省心。
- 遇到 rebase 冲突，早期版本会硬解，结果逻辑被改坏。现在的规则是：**冲突即停**，输出冲突文件和双方意图，等人决策。
- 大仓库 diff 很容易撑爆上下文，先 `--stat` 再按文件分块读取，必要时只看重点文件。

## 可复用建议

- 所有破坏性操作先输出计划（哪些分支、哪些提交、影响范围），确认后再执行。
- 把规则写进项目级配置（如 AGENTS.md），新人接入 agent 时规则跟着仓库走。
- 分支命名规范化（`feature/xxx`、`fix/xxx`），agent 才能可靠区分任务分支和保护分支。
- 日志 > 记忆。agent 的每一步 git 操作落盘，事后可审计。

## 总结

AI 管 Git 的价值不在“全自动”，而在把重复劳动压到接近零，同时把风险锁在权限边界和确认环节里。工具层拦截 + 流程固化 + hook 兜底，三层下来，agent 干活的胆子可以放大一点，你睡觉也能踏实一点。欢迎在评论区交流你们的权限模型和清理策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/c4a45ed4a1b373e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/7174a4be400152e0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/8f5ff6fca7da86d9.png)

