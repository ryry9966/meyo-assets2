---
title: 把 Git 交给 Agent：一套约束式的提交与分支自动化
feedId: 42654
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

用 OpenClaw 跑了一段时间日常开发后，最自然的诉求就是：能不能让 agent 顺手把 git 也管了——写提交、开分支、整理变更。我实际接了两周，结论是：可以做，但必须先立规矩再放手。裸跑状态下 agent 的 git 操作，和实习生第一天的提交记录差不多：能用，但不敢看。

## 问题

没有护栏时的典型事故：

1. `git add -A` 一把梭，把调试临时文件、`.env`、日志全带进提交；
2. 提交粒度失控：要么一坨巨型 commit，要么五个文件拆成五个 "fix"；
3. 分支管理混乱：直接在 main 上改，或开了分支忘切回；
4. 最危险的：agent 在 rebase 冲突后自作主张 force push。

## 做法

核心思路是**约束式授权**：只读默认放行，写操作走流程，破坏性操作不开放入口。

**第一步：接 MCP git server，收窄工具面。** 用社区的 git MCP server 跑在独立进程里，关键配置是 tool 白名单：`status`、`diff`、`log`、`branch -l` 这类只读工具直接放行；`commit`、`checkout -b` 开放；`push --force`、`reset --hard`、`rebase` 一律不注册。收窄入口比写一堆 prompt 规则可靠得多——工具不在列表里，agent 想用也没有门。

**第二步：提交规范写进 workspace 指令。** 固定几条硬规则：

- 提交前必须先跑 `git diff --staged` 并复述变更内容，再写 message；
- 提交信息用 Conventional Commits，scope 按模块名；
- 禁止 `git add -A` / `git add .`，只能 add 明确列出的文件；
- 新工作必须先开分支，命名 `feat/xxx`、`fix/xxx`。

**第三步：提案-执行两段式。** 加一条硬规则：agent 先产出 commit plan（涉及文件、分组、每组的 message）发出来，我确认后它再执行。多一轮交互，但挡掉了九成误提交。

**第四步：pre-commit hook 兜底。** 不管是谁写的提交，hook 都会跑：secret 扫描（防 `.env` 和 token 混入）+ lint。agent 绕不过去，这是最后一道闸。

## 踩坑点

- **凭证继承**：git server 默认继承本机 SSH key 和 credential helper，等于 agent 有完整推送权。我先在测试仓库单独配了受限凭证，跑顺了再放主仓库。
- **add 粒度**：即使禁了 `-A`，agent 仍会把逻辑无关的两个文件凑成一个 commit。规范里补一句“变更目的不同就拆开”才压住。
- **detached HEAD**：agent review 历史 commit 时会 checkout 到旧提交，忘切回后后续提交全部悬空。对策：review 类操作强制走 `git show`，不实际 checkout。
- **幻觉路径**：message 写得很像样，却引用了一个根本没改的文件。两段式提案让它先复述 diff 后，这个问题基本消失。

## 可复用建议

- 权限收窄优先于 prompt 约束，工具不注册比“请不要”有效；
- 写操作一律走提案-执行两段式，前期慢一点值得；
- pre-commit hook 与 agent 无关，务必最先装；
- 先在测试仓库把 agent 的 git 行为过一遍，再上主仓库；
- 定期 `git log --author` 抽查 agent 提交，比事后回滚便宜。

## 总结

让 AI 管 git 的本质不是“自动化提交”，而是把人类工程师的纪律翻译成**工具白名单 + 流程约束 + hook 兜底**三层结构。agent 执行得很快，但边界必须由人来画。两周下来，日常的 feat/fix 提交和分支管理基本可以放心交给它；rebase 和发版我仍然收在手里。这个授权比例，目前用着最舒服。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/7f12a6f02856e21a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/fff0ebb086f02221.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/452c57945b6984f9.png)

