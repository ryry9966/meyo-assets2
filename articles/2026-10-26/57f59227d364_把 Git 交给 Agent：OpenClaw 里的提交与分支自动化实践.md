---
title: 把 Git 交给 Agent：OpenClaw 里的提交与分支自动化实践
feedId: 42672
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

日常开发里，Git 操作占了不少碎片时间：写提交信息、切分支、合并后清理、攒了一堆 `wip`。这些事不难，但很打断心流。OpenClaw 有 shell 权限和技能体系，天然适合把这类流程性操作交给 Agent 处理。

## 问题

直接让 AI 跑 git 命令，有三个真实风险：

1. **提交内容失控**——`.env`、构建产物被顺手 stage 进去
2. **破坏性操作**——模型理解偏差下一个 `--force` 就回不去了
3. **状态幻觉**——模型默认工作区是干净的，不查就动手

所以核心不是"能不能自动化"，而是"怎么在护栏内自动化"。

## 做法

**第一步：限定操作面。** 不要让 Agent 拿裸 shell 碰 git。两条路：接一个 git MCP server，把操作收敛成结构化工具；或者在技能文件里写死子命令白名单（`status` / `diff` / `log` / `add` / `commit` / `branch` / `switch`）。

**第二步：写一份 git 技能文件。** 规则比能力重要，我放在项目 skill 里：

- 任何写操作前，必须先跑 `git status` 和 `git diff`
- 提交信息遵循 Conventional Commits，文件里直接附两三个示例
- 禁止 `--force`、禁止动 `main`、禁止提交 `.env` / `*.pem` / `*.key`
- 推送、删分支、rebase 必须先列出计划，等我确认再执行

**第三步：加一道 git hooks 保险。** pre-commit 挂密钥扫描。Agent 再守规矩也可能漏，hook 是最后防线。

**第四步：按场景跑起来。**

- 日常：改完代码让 Agent 读 diff，按逻辑分块提交，顺手草拟 PR 描述
- 清理：每周让它列出已合并分支，给出删除清单，确认后执行
- 回顾：早上让它汇总昨天的 commits 和各分支状态

## 踩坑点

- **交互式命令会挂死**：`rebase -i` 或要求输密码的命令会卡住 Agent，一律改用非交互参数
- **多仓库串台**：遇到过 Agent 在错误仓库里提交，技能里写死工作区绝对路径
- **提交信息风格漂移**：模型爱写长段落，只靠描述约束会跑偏，few-shot 示例比形容词管用
- **分块不合理**：AI 有时把不相关的改动塞进一个 commit，要给明确规则（按模块 / 按意图拆）
- **detached HEAD**：它可能在游离状态下提交然后找不到，规则里要求动手前先确认当前分支

## 可复用建议

1. **先只读，后写入**：第一周只让 Agent 跑 `status` / `diff` / `log`，观察它的判断质量再放开写权限
2. **dry-run 模式**：所有写操作先输出计划清单（文件、提交信息、目标分支），人工点头再执行
3. **护栏版本化**：规则写进受版本管理的技能文件，团队可审计、可回滚
4. **保留操作日志**：Agent 每次 git 操作追加到一个 journal 文件，出问题能追溯

## 总结

AI 管 Git 的价值不在快，而在于把机械操作从注意力里移走。前提是把它当成一个守规矩但需要监督的初级同事：给白名单、给示例、给确认门，而不是直接给 root。这套思路迁移到其他自动化场景——部署检查、依赖升级——同样成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/69f55e93fd3fb638.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/28d82844a536bb0f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/2f3da8ceb892ba30.png)

