---
title: Git 自动化实践：用 OpenClaw Agent 接管提交与分支的重复劳动
feedId: 37884
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

日常开发里，真正消耗时间的经常不是写代码，而是围绕仓库的重复劳动：想一个像样的 commit message、确认自己在哪个分支、清理合并完的旧分支、推送前检查有没有把临时文件带进去。这些操作高频、机械、单次风险低，正好是 agent 自动化的甜区。这篇帖记录一下我在 OpenClaw 里用 git MCP + skill 把这套流程跑起来的实践。

## 问题

人肉维护仓库的几个典型痛点：

- commit message 质量靠自觉，规范文档没人看；
- 本地分支只增不减，`git branch` 一屏放不下；
- 切错分支直接开写，改完才发现 base 不对。

而直接把 git 命令交给 agent，又有另一类风险：`git add -A` 把 `.env` 提上去、对共享分支 force push、在 detached HEAD 上提交后"失踪"。

所以目标不是"全自动"，而是：**高频低风险的操作交给 agent，破坏性操作留下硬性闸门。**

## 做法

### 1. 接入工具面，先只读

给 agent 挂 git MCP server（或 shell 插件），第一版只开只读命令：`status / diff / log / branch`，工作目录锁定在目标 repo。跑一周确认行为可控，再放开 `add / commit / checkout -b` 这类写操作。`push / reset / clean / branch -D` 单独放白名单，默认人工确认。

### 2. 把流程写成 skill，而不是靠聊天提醒

提交的 SOP 我直接写成了 skill 文件放进仓库：

```
提交 SOP：
1. git status，校验当前分支名与预期一致，否则中止
2. 阅读 git diff --staged，禁止使用 git add -A，
   只 add 明确列出的文件
3. 按 Conventional Commits 生成 message：
   type(scope): subject，subject 不超过 50 字符，禁用形容词
4. 输出"将执行的命令"清单，等人确认后执行
```

skill 跟着仓库走，团队里每个人的 agent 行为是一致的，这比提示词写在聊天框里可靠得多。

### 3. 分支治理做成定时任务

用一个每日触发的任务让 agent 执行：

```bash
git branch --merged main | grep -vE 'main|dev|release'
```

拿到清单后生成"建议删除"报告，确认后批量清理。从 issue 建分支也交给它：按 `feat/` 或 `fix/` 前缀 + issue 编号模板生成，避免出现 `test2`、`final-真的final` 这种命名。

### 4. 硬护栏放 hook，不靠提示词

- pre-commit 挂 secret 扫描 + lint，只做秒级检查；
- 服务端开分支保护，直接禁止对 `main` force push；
- agent 侧再做一层命令黑名单兜底。

## 踩坑点

1. **`git add -A` 是最大事故源。** 提示词里写"不要提交敏感文件"没用，`.env` 没进 `.gitignore` 照样进暂存区。SOP 强制逐文件 add，pre-commit 扫描兜底，两层缺一不可。
2. **message 过度发挥。** 不约束会生成"重大架构重构"这类空话。模板限定 type/scope 白名单 + 长度上限后基本收敛。
3. **detached HEAD 和 worktree 混乱。** agent 在错误状态下照样能提交，提交就"丢了"。SOP 第一步的分支校验必须是中止性检查，不是提醒。
4. **MCP 调用超时。** pre-commit 里塞全量构建，工具调用会拖死整个会话。重活留给 CI，本地 hook 只做轻量检查。

## 可复用建议

- skill/SOP 文件进版本库，全团队共用同一套 agent 行为；
- 破坏性操作统一走"两段式"：agent 先输出完整命令计划，人工放行才执行；
- 本地维护一份审计日志，append 记录 agent 执行过的每条 git 命令，出问题能回溯；
- 从只读开始灰度，观察一到两周再放开写权限。

## 总结

agent 做 git 自动化的价值，在于把高频、低风险、机械的部分收走，让人只处理需要判断的节点。而安全这件事，靠的是权限边界、hook 和审计，从来不是提示词写得够不够狠。先把只读跑顺，再逐步放开，是目前试下来成本最低、翻车最少的路径。欢迎在评论区交流你们的 skill 写法和踩过的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/be9eca44ef54c659.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/d36a6d93d4d3cfbf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/82a12331bedd4382.png)

