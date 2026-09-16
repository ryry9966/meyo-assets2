---
title: Git 自动化实战：让 Agent 管提交和分支，重点是划清权限边界
feedId: 37823
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

团队日常的 Git 操作大概分三类：高频但机械（提交、拉取、推送），低频但危险（rebase、回滚、分支清理），以及纯文书工作（commit message、PR 描述）。这三类的出错代价完全不对等——message 写得烂只是难看，reset --hard 用错可能丢提交。

OpenClaw 接入 MCP 工具后，Agent 已经具备读仓库状态、执行受限命令的能力。所以问题早就不是"能不能做"，而是"怎么让它做得安全、做得准"。

## 问题

让 AI 直接碰 git，最怕三件事：

1. **提交信息失真。** 模型对 diff 的理解常停留在表面，产出 `update code`、`fix bug` 这种等于没写的 message。
2. **误操作。** `push --force`、对 main 直接提交、checkout 丢弃本地修改，任何一条都出不起事故。
3. **静默带入脏东西。** `git add -A` 一把梭，把 .env、日志、构建产物混进提交。

结论：自动化的核心不是 prompt 写多花哨，而是权限边界划多细。

## 做法

我们分四步落地，可按需裁剪。

**第一步：把 git 封装成 MCP 工具，不让 Agent 裸跑 shell。**

写一层薄封装，只暴露这些动作：`git_status`、`git_diff`、`git_log`、`git_add`（带路径校验）、`git_commit`、`git_branch`（create/switch/list）、`git_push`。明确不暴露：reset --hard、push --force、丢弃修改类操作、对受保护分支的任何写入。工具内部先做参数校验再执行，Agent 拿不到任意命令的口子。

**第二步：规范放仓库，prompt 只引用。**

仓库根目录放一份提交规范文件：Conventional Commits 前缀、分支命名规则（`feat/xxx`、`fix/xxx`、`chore/xxx`）、单次提交的粒度要求。系统 prompt 里只写一句"提交前必须读取并遵守该规范"。规范随仓库走版本管理，换项目不用改 prompt。

**第三步：两段式提交流程。**

不允许 Agent 一次调用完成 add + commit + push，流程拆开：

1. Agent 先跑 `git_status` + `git_diff`，自己分析改动；
2. 输出结构化提交计划：message、涉及文件清单、目标分支；
3. 计划先落盘，人确认，或用 dry-run 标志空跑一轮；
4. 确认后依次执行 add → commit → push。

我们目前 push 已全自动，但仅限个人分支；有 review 要求的仓库保留一步人工确认。这是刻意的：自动化程度必须按出错代价分级。

**第四步：分支清理做成定时任务。**

每周末跑一次：列出超过 30 天无活动、且对应 PR 已合并的本地分支，生成清单交给 Agent 核对远端状态，确认后删除。删除走同一个受限工具，永远不带 `--force`。

## 踩坑点

- **大仓库 diff 爆上下文。** 改动上千行时模型只看到截断后的 diff，message 必然不准。解法：工具层按文件分块，模型逐块生成摘要再汇总，比硬塞全文靠谱得多。
- **Agent 会"顺手"修东西。** 让它提交，它发现测试挂了就自己改两行一起提交，message 里只字未提。现在工具层校验：commit 涉及的文件必须与计划一致，不一致直接拒绝执行。
- **并行会话互相踩。** 两个 Agent 会话同时操作同一工作区，出现 index.lock 冲突。粗暴但有效的解法：按仓库路径加文件锁，同一时刻只允许一个会话操作。
- **.gitignore 不是保险箱。** 有次新增的配置文件忘了进 .gitignore，被带进了提交。后来在 `git_add` 工具里加了一层敏感文件名单（.env、*.pem、credentials 类）做硬拦截。

## 可复用建议

- **权限三级分：** 读操作（status/diff/log）放开，写操作走白名单，破坏性操作物理上不暴露给 Agent。
- **先 dry-run 再放权。** 任何自动化流程先跑两周"只出计划不执行"，人工核对计划质量后再逐步放开。
- **全量落日志。** Agent 的每次 git 工具调用，参数和结果码都记录，出问题能回放。
- **规范进仓库，prompt 做引用。** 别把团队规范硬编码在系统 prompt 里，它应该跟着代码一起演进。

## 总结

AI 管 Git 这件事，技术含量不在让模型写 commit message——那部分它已经够用了。真正的工程量全在边界上：哪些操作它永远碰不到，哪些需要确认，哪些可以全自动。边界做扎实之后，提交信息生成、分支命名、过期分支清理确实能稳定省下每天十几分钟的机械劳动。省下来的时间，够多看两遍 diff 了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/5e98ace53cb5461c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/70581d1e09c4552a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/cdc2304959cf943c.png)

