---
title: 让 AI 助手管好你的 Git：提交与分支自动化两个月实录
feedId: 42736
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

日常开发里，Git 操作占了不小的"低价值重复劳动"比例：写完一个小功能要 commit，message 想半天；合并完忘删分支，隔两周一看本地堆了二十个 stale branch。这些动作规则明确、判断简单，恰好是 AI 助手（OpenClaw + MCP 工具链）擅长接管的类型。这篇分享我把自己仓库的提交与分支管理交给 agent 跑了两个多月的做法。

## 问题

交给 AI 之前，先想清楚要解决什么：

1. **提交质量不稳定**：忙起来就是 `fix`、`update`，回看历史全靠猜。
2. **分支堆积**：远端合了，本地和远端的 feature 分支没人删。
3. **误操作风险**：AI 直接碰 git，一旦 `add -A` 提交了 `.env`，或者 `push --force`，代价很大。

所以目标不是"全自动"，而是：**标准化动作交给 agent，破坏性动作留人工确认。**

## 做法

**1. 接入与权限收敛**

通过 MCP 把 git 能力接进 OpenClaw（官方 mcp-server-git 或自封装 shell 工具均可），关键在工具白名单：只放行 `status / diff / log / add / commit / branch / switch` 这类本地安全命令；`push / rebase / reset --hard / clean` 一律走显式确认；`--force` 直接禁用。工作目录锁定到项目根，避免 agent 在别的仓库里"顺手操作"。

**2. 把提交流程固化成一条技能**

写一个可复用的 skill/prompt 模板，流程固定为：

- 先跑 `git status` + `git diff --staged`，禁止凭文件名猜变更；
- 按 Conventional Commits 规范生成 message，附一句"为什么"，而不是复述 diff；
- 列出待提交文件清单，等确认；
- 确认后 `add`（明确列出文件，禁止 `-A`）→ `commit`。

**3. 分支清理做成定时任务**

用 OpenClaw 的定时任务每天跑一次：`git branch --merged` 找出已合并分支，排除 main/develop 后生成清理清单发给我，我回一句"确认"它再批量删。远端分支删除同样走确认。

## 踩坑点

- **`add -A` 泄漏 secrets**：早期图省事让 agent 全量 add，差点把 `.env.local` 提上去。后来规则里强制"逐文件列出 + 核对 .gitignore 覆盖"，并在提交前扫一遍文件名中的敏感关键词。
- **在 main 上直接提交**：agent 有一次切分支失败后顺势在 main 上 commit 了。修复方式是每次操作前先读当前分支，main/develop 标记为只读，需要改动时必须先建 feature 分支。
- **message 与 diff 不符**：agent 偶尔"脑补"提交意图。强制先看完整 diff 再写 message，且要求引用具体模块名，方便人工核对。
- **自动 push 的诱惑**：试过提交后自动 push，结果推过一次半成品。现在 push 是独立确认动作，agent 只做本地提交。

## 可复用建议

1. 把权限白名单和提交流程写成一个 skill 文件团队共享，比每个人口头调教稳定得多。
2. Conventional Commits + agent 生成 message，后续接 changelog 自动化收益明显。
3. agent 最适合"高频小提交"；大重构的 message 还是人来写。
4. push、删分支、rebase 这类破坏性操作永远保留人工确认点，宁可多点一下。

## 总结

两个月跑下来，提交规范率和分支清洁度提升明显。agent 的真正价值不是替你做决定，而是把那些你知道该做但懒得做的标准化动作变成默认动作。权限收敛 + 确认点设计做到位，AI 管 Git 是稳的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/6f081ea4176ec6bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/1c5dd5b8ed44491a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5692e43cfac6c0f7.png)

