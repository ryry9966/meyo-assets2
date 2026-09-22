---
title: Git 自动化实践：让 Agent 管提交和分支，但留好三道闸
feedId: 38493
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

用 OpenClaw 这类 agent 干活一段时间后，瓶颈往往不在写代码，而在周边杂务。一个任务下来 agent 改十几个文件，人工拆分提交、写 message、切分支、清理陈旧分支，既打断心流又容易敷衍。于是很自然的想法：把 git 操作也交给 agent。

## 问题

实际跑起来会遇到三类麻烦：

1. **提交质量崩坏**。agent 默认 `git add -A`，把构建产物、临时文件甚至本地配置一锅端；message 千篇一律的 "update code"。
2. **状态误判**。agent 凭记忆假设工作区干净、分支正确，结果把改动提交到了 main。
3. **危险操作失控**。`reset --hard`、`push --force` 一旦进入 agent 工具链，翻车只是时间问题。

## 做法

我的方案分三层，从读到写逐步放开：

**第一步，收敛 git 入口。** 不要给 agent 裸 shell + git，而是用 MCP git 工具或自封装脚本，只暴露有限动词：status、diff、log、add（带显式 pathspec）、commit、branch、checkout。push、reset、rebase 单独做成需确认的"特权操作"。

**第二步，把约定写进仓库。** 在仓库内的 agent 指令文件里固化规则：

- 每次提交前必须执行 `git status` 和 `git branch --show-current`，禁止凭记忆操作；
- 一次提交只解决一件事，主题行不超过 50 字符，正文解释"为什么"而不是罗列"改了什么"；
- 提交前跑测试，失败即停，不许循环重试；
- 永远不用 `reset --hard` 和 `push --force`，分支清理用 `push --force-with-lease` 且仅限特性分支。

**第三步，分支生命周期自动化。** 约定命名 `agent/<任务ID>-<短描述>`，任务合并后由 agent 提出清理列表，人工确认后删除。分支列表本身就是任务账本。

## 踩坑点

- **`add -A` 是第一大事故来源**。除了 .gitignore 要干净，必须要求 agent 先输出"提交计划"（哪些文件、什么 message），确认后再执行，杂音一下就少了。
- **LLM 写的 message 天然啰嗦爱吹**。靠字数和格式硬约束压住，比劝有用。
- **pre-commit hook 变慢后，agent 会陷入失败重试循环**。要明确告诉它 hook 失败是信号不是噪音。
- **agent 处理 rebase 冲突不稳定**，多文件冲突时经常"和稀泥"。我的原则：冲突超过一个文件就暂停问人。
- **最隐蔽的坑是密钥**。agent 可能顺手把 .env 里的 token 提交进去，pre-commit 挂密钥扫描是底线。

## 可复用建议

- **先只读后写**：让 agent 先做 log 分析、diff 总结，跑稳两周再放开写权限，是成本最低的灰度方式。
- **约定放仓库不放会话**：写在仓库指令文件里，任何 agent、任何会话进来都生效，规则本身也能被团队 review。
- **落审计日志**：所有 agent 的 git 操作记一份流水，出问题能回溯。
- **共享分支保留人工确认门**：main / release 上的任何写操作都别为效率让步。

## 总结

Agent 管 git 的价值是真实的，但杠杆在约束而非放权。把 git 当作一组有边界的特权动作，用"仓库级约定 + 提交计划确认 + 特权操作审批"三道闸，就能拿到八成自动化收益，同时把翻车概率压到可接受的范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/cbc371ebc3c20b61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/7b23ed85d1039eff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/0ba0f88ad6e1b3ea.png)

