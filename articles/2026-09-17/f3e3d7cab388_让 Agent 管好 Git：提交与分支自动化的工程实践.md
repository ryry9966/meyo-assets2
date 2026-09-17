---
title: 让 Agent 管好 Git：提交与分支自动化的工程实践
feedId: 37976
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

用 OpenClaw 这类 Agent 干活时，最常见的低价值重复劳动就是 Git：写完代码敲提交信息、切分支、清理已合并分支、整理未推送的本地提交。这些操作规则明确、上下文有限，天然适合交给 Agent。过去几个月我把一部分 Git 事务交给 OpenClaw 处理，本文记录一套可复现的做法和边界。

## 问题

手工 Git 的痛点不在单个命令，而在三类场景：

1. **提交信息质量不稳定**：赶时间时全是 `fix`、`update`，回看历史等于没有历史；
2. **分支堆积**：本地十几条 `experiment/*` 分支早已合并，但没人记得删；
3. **上下文切换成本**：Agent 写完代码，还要自己手工打包成提交，工作流被打断。

但直接放开让 Agent 跑 git 也不行——它可能 force push、把 `.env` 提交进去，或者在错误的分支上 rebase。所以核心矛盾是：**如何给 Agent 足够的权限做机械劳动，又不给它做破坏的能力**。

## 做法

核心思路：给 Agent 一个“受限 Git 工作台”，而不是完整 shell。

**第一步：权限收敛。** 通过 MCP 的 git server 或沙箱 shell，只暴露白名单命令：`status`、`diff`、`add`、`commit`、`branch`、`checkout`、`log`、`merge --ff-only`。`push`、`reset --hard`、`clean` 一律不走自动化，保留人工执行。

**第二步：约定文件化。** 在仓库根目录放一个 AGENTS.md，写清提交规范（如 Conventional Commits）、分支命名（`feature/xxx`、`fix/xxx`）、哪些文件永不提交。Agent 每次操作前读取，比在对话 prompt 里反复强调可靠得多，而且约定可以随代码一起 review。

**第三步：三段式提交流程。** 让 Agent 每次提交前：跑 `git status --porcelain` 和 `git diff --staged` 汇总变更 → 生成提交信息并展示 → 等确认或按规则自动通过。我的规则是“单文件小改动自动过，跨模块改动必须人工确认”。

**第四步：分支维护任务化。** 用 OpenClaw 的定时任务每天跑一次：`git branch --merged` 拿到已合并分支列表，排除主干和当前分支，其余汇报给我，我确认后批量删除。远端分支只汇报、不删除。

## 踩坑点

- **Agent 会“好心”执行 `git add .`**，把构建产物和临时文件一起带进去。解法：约定文件里明确 add 前必须列出文件清单，逐个确认。
- **提交信息幻觉**：diff 很小时 Agent 会脑补改动动机。要求它只描述可见变更，动机字段留空由我补。
- **仓库内容即 prompt**：issue 模板、代码注释里如果藏了指令性文字，Agent 可能照做。所以 Git 类任务只给白名单工具、不给任意 shell，这条是硬边界，没有商量余地。
- **定时清理误删**：有一次 rebase 后的分支显示“未合并”，Agent 按规则跳过了——这反而验证了“只删已合并”的策略是对的，别为了省事加 `--force`。

## 可复用建议

- 自动化的边界写成配置，不写进对话 prompt；
- 破坏性操作默认**不可达**，而不是“要求它别做”；
- 让 Agent 先输出计划（add 哪些文件、提交信息是什么）再执行，观察一两周再放开自动确认；
- 规范放进仓库内文件，跟代码一起演进和 review。

## 总结

Agent 管 Git 的价值不在于“全自动”，而在于把机械部分抽走、把判断留在人这边。收敛权限、约定文件化、先计划后执行，这三条做到位后，这类自动化是低风险高回报的。建议从“只生成提交信息”这个最小场景起步，稳定后再扩到分支维护，别一上来就追求端到端无人化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/5036afaeddd6b019.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/96e8034b8500b2c7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/bf8c61518d99be2c.png)

