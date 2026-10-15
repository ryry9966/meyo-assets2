---
title: Git 自动化实践：让 AI 助手管提交、理分支，把闸门留给人
feedId: 41639
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

日常开发里最琐碎的不是写代码，而是围绕代码的仪式：写提交信息、切分支、合完删分支、翻 log 找"上周那个改动到底在哪"。这些操作结构化程度高、结果可验证，是接入了 shell 或 MCP 工具的 OpenClaw Agent 最适合先啃的一块骨头。但"能让 AI 跑 git"和"能放心让它跑"是两回事，这篇记录我跑了几个月的落地方式。

## 问题

三个真实痛点：

1. **提交信息质量差**。`fix`、`update`、`临时改一下` 占了历史 log 的一半，回溯成本极高。
2. **分支无序堆积**。命名随心所欲，合并后的本地分支没人删，三个月后 `git branch` 一屏放不下。
3. **安全边界模糊**。让 Agent 直接执行 git 命令，等于把 force push、误删、把密钥提交进仓库的风险都交了出去。

## 做法

**第一步：收敛工具面。** 不给裸 shell。用 MCP git server 或自写的白名单封装，只暴露 `status / diff / add / commit / branch / log`，明确排除 `push --force`、`reset --hard`、`clean`。

**第二步：把规范写进项目指令。** 在仓库内的 Agent 配置（如 AGENT.md）里固化：提交信息遵循 Conventional Commits；分支命名 `feat|fix|chore/<编号>-<短描述>`；禁止提交 `.env`、构建产物；禁止直接在 main 上操作。规范落盘进版本控制，人和 Agent 读同一份。

**第三步：固化两条工作流。**
- *提交流*：Agent 读取暂存区 diff → 按规范起草提交信息 → 展示给我确认 → 通过后执行 commit。
- *分支流*：开任务时从最新 main 拉分支；合并后 Agent 删除本地分支并 prune 远程跟踪引用；每周跑一次 `git branch --merged` 做清扫。

**第四步：留人工闸门。** push 和 merge 永远需要我确认。Agent 负责准备，人负责放行。

## 踩坑点

1. **`git add -A` 是最大的坑**。Agent 一次性把带真实密钥的 `.env` 加进了暂存区，幸好被 pre-commit 的 gitleaks 拦下。现在规则改为：禁止 `-A`，必须显式列文件路径。
2. **超长 diff 烧 token 且降质**。三千行的 diff 直接喂给模型，信息要么溢出要么总结得很水。改成先按文件摘要、再汇总生成信息。
3. **Agent 会"热心"解决冲突**。一次 rebase 中它把冲突全取了 theirs，我的改动被静默丢弃。现在规则是：任何冲突，立即停下交还给人。
4. **Hook 与自动化互相触发**。pre-commit 里的 lint 失败后 Agent 进入重试循环。改为：hook 失败只报告一次，不盲目重试。
5. **别让 Agent 动仓库之外的 git 配置**。限定工作目录，禁改全局 config。

## 可复用建议

- **分级自动化**：起草提交信息（高收益、零风险）最先交出去；分支清理次之；push/merge 保持人工，跑稳了再议。
- **最小权限**：读操作（status/diff/log）可以放开，写操作加闸门，破坏性操作直接从工具面移除。
- **一次提交单一意图**：同时约束了信息质量和回滚粒度。
- **留执行日志**：Agent 跑过的每条 git 命令追加写入一个本地 log，出事能审计。
- **规范即配置**：写在仓库里而不是口头约定，换人、换 Agent 都不丢。

## 总结

AI 助手做 Git 自动化的正确定位不是"全自动提交机器人"，而是**草拟、整理、执行前的最后一道人审**。提交信息起草是最值得先交给 Agent 的事——省时间且无风险；分支清理其次；push 与 merge 的闸门暂时握在自己手里。工具面收窄、规范落盘、冲突即停，做到这三条，这套流程就能长期稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/ecc05beea8258373.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/a7cc34f8947ca956.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/22ff04cc0ae6eda1.png)

