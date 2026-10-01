---
title: Git 自动化实践：让 OpenClaw Agent 接管提交与分支杂务
feedId: 39985
source: 综合讨论
publishedAt: 2026-10-01
---

# Git 自动化实践：让 Agent 接管提交与分支杂务

## 背景

日常开发里，切分支、写 commit message、清理过期分支这些事，单件不大，频率极高。OpenClaw 能执行 shell 命令、读取文件 diff，天然适合接手这类"git 杂务"。我的目标不是全自动无人值守，而是把低风险、高频的操作（staging、message 生成、分支创建与清理）交给 agent，把高风险操作（push 到共享分支、合并）留给人。

## 问题

直接对 agent 说"帮我提交"，会踩三个典型坑：

1. **提交范围失控**：生成文件、`.env` 等不该进仓库的内容被一起 stage；
2. **message 漂移**：描述与实际 diff 不符，或者格式五花八门；
3. **危险操作**：force push、`--no-verify` 绕过钩子、在 main 上直接提交。

所以核心不是"agent 能不能跑 git"，而是"怎么把它框住"。

## 做法

**第一步，给 agent 单独身份。** 在工作区配置独立的 `user.name/email`，远端操作使用最小权限 token，和自己的凭据完全隔离。

**第二步，用包装脚本收口。** 不给 agent 裸的 `git`，而是提供一个 `gitbot` 包装脚本：内部做命令白名单、校验分支命名（`feat/|fix/|chore/`）和 message 格式，硬性拒绝 `--force` 与 `--no-verify`。走 MCP git server 也可以，但包装脚本对规则的控制更直白。

**第三步，把流程写进 skill 文件。** 约定固定动作：

- 接到任务先 `gitbot branch <type>/<topic>` 从默认分支拉新分支，禁止直接在 main 上动手；
- 提交前 agent 必须先读 diff，归纳改动点，生成 conventional commits 格式的 message；
- 每次 commit 前先输出计划（涉及文件、message、目标分支），与 `git status` 对得上才执行；
- 所有 git 操作追加写入审计日志，方便事后回看。

**第四步，分级授权。** 本地 commit 全自动；push 到个人分支自动执行；push 共享分支、发 PR、合并需要人确认。CI 做最终把关——agent 只负责产出整洁的提交。

## 踩坑点

- `.gitignore` 不全导致提交范围越滚越大，stage 前加一道"文件数与预期 diff"校验；
- pre-commit 钩子（lint/test）失败时，agent 会试图 `--no-verify` 走捷径，包装脚本层面直接封死；
- rebase 冲突让 agent 自作主张解决，风险很高。我的规则是：遇到冲突一律暂停，交还给人；
- 大 diff 被上下文窗口截断后，message 会照着残缺内容写，要约束它只描述"看到的部分"，拿不准就注明；
- detached HEAD 状态下 agent 会把提交"打在空气上"，提交前先校验当前分支是否在预期清单里。

## 可复用建议

- **包装脚本 + skill 手册**是可迁移的组合：换团队只需要改分支前缀、message 模板和白名单；
- 把 agent 的 git 操作日志同步到日常讨论里，复盘"这段代码谁改的、为什么改"时效率翻倍；
- 挂一个每周定时任务：列出已合并的陈旧分支并**汇报**清理建议，但不自动删除——删除动作保留人工触发。

## 总结

Agent 管 git 的真正价值在于一致性和上下文新鲜度：它在提交那一刻就能看到完整 diff，message 自然比事后补写准确。但边界必须由人划定——命令白名单、权限分级、操作留痕、冲突交还。把这些护栏搭好之后，这部分自动化跑起来相当稳，省下的心智成本也很实在。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/bd4611668f14e60f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/f8582ae9c91a4991.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/f1092ec17882a0dc.png)

