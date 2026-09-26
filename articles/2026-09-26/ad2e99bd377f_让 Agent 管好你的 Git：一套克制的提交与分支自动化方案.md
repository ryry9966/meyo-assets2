---
title: 让 Agent 管好你的 Git：一套克制的提交与分支自动化方案
feedId: 39162
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

用 OpenClaw 这类 Agent 干活的日常是：一句话让它改一圈代码，半小时后它动了二十几个文件。这时候真正的瓶颈不是写代码，而是收尾——提交历史一塌糊涂：`update`、`fix`、`fix again`，任务改动和 main 混在一起，改到一半的东西不知道该不该提交。

我们团队的做法是：把 Git 操作交还给 Agent，但给它戴上明确的笼头。跑了几个月，效果稳定，下面是可复现的方案。

## 问题定义

人肉管 Agent 的 Git 产出，常见四种翻车：

1. 提交信息与 diff 无关（Agent 按“意图”写 message，不看实际改动）；
2. `git add -A` 把 `.env`、构建产物一锅端；
3. 在共享分支上 rebase / force push；
4. 任务代码直接落在 main 上，或者留下一堆没用完的僵尸分支。

本质是权限边界和约定没有工程化。只在 prompt 里写一句“请规范提交”，是不够的。

## 做法

### 1. 收敛工具面

不要让 Agent 拿着完整 shell 随意敲 git。两条路线：

- 用 MCP 的 git server，只暴露 status / diff / add / commit / branch / switch / log 这类命令；
- 或自建封装脚本，白名单 + 参数校验。

`push --force`、`reset --hard`、`clean`、`checkout -- .` 一律不暴露；`push` 单独做成需要人工确认的动作。

### 2. 把约定写进仓库，而不是会话

在仓库根放一份 Agent 规则文件（如 AGENTS.md），规范全部落盘：

- 分支：`feat/<task>`、`fix/<issue>`，任务开始即建分支；
- 提交：Conventional Commits，一个逻辑改动一个 commit；
- 提交前必须先跑 `git diff --stat` 和 `git diff`，message 从 diff 归纳，禁止凭记忆写；
- push 永远需要人确认。

放进版本库的好处是规则随代码走，换人、换会话都不丢，还能被 code review。

### 3. 分支生命周期自动化

让 Agent 按固定剧本走：接任务 → `git switch -c feat/xxx` → 干活 → 分批提交 → 汇报等合并。合并后由 Agent 执行 `git branch -d` 清理本地，外加每天一次 `git fetch --prune`。僵尸分支问题基本消失。

### 4. 用 hooks 兜底

格式化、lint 挂 pre-commit hook，Agent 绕不过去；同时把 Agent 执行过的 git 命令落一份审计日志，出问题能回放。

## 踩坑点

- **message 与 diff 脱节**：早期最常见。强制“先 diff 后 message”的顺序后解决，必要时让 Agent 先复述改动要点再落 message。
- **中途态处理**：Agent 遇到 merge conflict 或 detached HEAD 容易自作主张 `reset --hard`。规则里明确写：非干净状态先停下汇报，禁止自行恢复历史。
- **`.gitignore` 要先于自动化**：没配好就放权 `git add`，等于邀请泄密。我们要求 Agent 提交前列出文件清单，而不是无脑 `-A`。
- **规则失效**：规则只写在系统 prompt 里，隔几天新会话就“忘”了。落仓库、每次任务开始时 Agent 主动读取，才稳定。
- **不要自动化合并**：让 Agent 自己 merge 自己的 PR 等于没有 review。合并保留在人手上，成本不高，兜底价值大。

## 可复用建议

- 权限分三级：只读命令放行、写命令白名单、改历史的命令彻底禁掉；
- 破坏性动作（push、merge、rebase）默认人工确认，跑顺了再逐步放权；
- 每条自动化规则都应能写进仓库文件、能被 review；
- 审计日志越早加越好，它是你后续调规则的依据。

## 总结

Git 自动化的收益不在省那几秒敲命令，而在于把“提交纪律”从人的自觉变成系统的默认行为。原则一句话：**Agent 可以建分支、写提交，但不动历史、不碰远端；人保留 review 和合并的最终权。**按这个边界放权，让 Agent 管理 Git 是净收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/141a04243f2eab4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/332f4611db2682fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/508430173e32cb73.png)

