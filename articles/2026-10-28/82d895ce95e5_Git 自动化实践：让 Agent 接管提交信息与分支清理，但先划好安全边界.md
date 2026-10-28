---
title: Git 自动化实践：让 Agent 接管提交信息与分支清理，但先划好安全边界
feedId: 42816
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

在 OpenClaw 里跑编码类 Agent 之后，我发现最耗时间的不是写代码，而是代码写完之后那一串机械动作：看 diff、挑文件、写 commit message、切分支、合并后删分支。这些事规律性强、判断成本低，天然适合交给 Agent。

## 问题

直接把 shell 权限丢给 Agent 会出事。我第一次尝试时，它为了"清理工作区"执行了 `git checkout .`，丢了我半天的未提交改动。此外还有几个典型痛点：

- AI 生成的改动常常横跨多个文件，手动 `git add` 容易漏、提交信息也写得敷衍；
- 分支命名混乱，`fix-2`、`temp1` 这类分支越堆越多；
- 合并后的本地分支没人删，半年后 `git branch` 一屏都放不下。

## 做法

**第一步：划白名单，不给全量 git 权限。** 在 Agent 的工具配置里只放行：`git status`、`git diff`、`git log`、`git add <file>`（禁止 `-A`）、`git commit -m`、`git branch`、`git checkout -b`。明确禁止 `push --force`、`reset --hard`、`checkout .`、`clean`。如果是 MCP 工具形式，就在工具层做参数校验，而不是靠提示词自觉。

**第二步：定义提交流程。** 提示词里固定动作顺序：先 `status` + `diff`，列出待提交文件并说明每个文件的改动意图；生成符合 Conventional Commits 的 message（feat/fix/chore + 中文正文）；**把文件清单和 message 展示给我确认后**，才执行 `git add` 和 `git commit`。生成 message 时除了 diff，我还会把当前任务描述一并喂进去——只看 diff 猜不出意图。

**第三步：分支治理做成定时任务。** 每天让 Agent 跑一次：`git branch --merged main` 列出已合并分支，排除 `main`/`dev` 后逐个删除；长期未合并的分支只列出来提醒我决策，不自动删。建分支时强制走 `feat/issue号-简述` 模板。

## 踩坑点

1. **交互式命令会挂死 Agent**。`git commit` 不带 `-m` 会唤起编辑器，进程直接卡住。所有命令都用非交互参数。
2. **`git add -A` 是泄密入口**。我要求 Agent 在提交前对 staged diff 做一次敏感文件检查（`.env`、密钥、异常大文件），发现就停下来问我。
3. **只看 diff 写 message 会跑偏**。重构类改动 diff 很大但意图简单，必须补充任务上下文。
4. **合并冲突不要让 Agent 自动解**。让它检出冲突文件列表并汇报，人来决策。
5. **确认门不能省**。哪怕流程跑顺了，破坏性操作前"先展示给我看"这一步也要保留。

## 可复用建议

- **渐进授权**：先只读（status/diff/log）跑一周，再加 commit，最后放开分支管理；push 始终留给自己或要求逐次确认。
- 把"白名单 + 流程提示词 + 确认门"打包成一个可复用的 skill 或插件，而不是每次手写 prompt。
- 让 Agent 把每次执行的 git 命令追加到日志文件，方便事后审计。
- 主分支设保护，Agent 只在 feature 分支工作，物理隔离风险。

## 总结

这套东西跑下来，提交信息的一致性和分支整洁度提升明显，省下的时间反而是次要收获。核心经验是：AI 助手适合接管 git 里"判断成本低、规律性强"的部分，而安全边界必须显式划出来——白名单、确认门、审计日志，三者缺一不可。自动化不是放手，是把重复劳动交给机器，把判断留给自己。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/bf3eea804fb45463.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/5b3ac5d777d58018.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/0d1f40996ba39259.png)

