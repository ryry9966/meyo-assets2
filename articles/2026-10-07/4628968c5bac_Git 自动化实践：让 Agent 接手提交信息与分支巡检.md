---
title: Git 自动化实践：让 Agent 接手提交信息与分支巡检
feedId: 40791
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

用 OpenClaw 这几个月，我发现 Agent 最容易被低估的用途不是写代码，而是处理 Git 里那些重复琐碎的事：写提交信息、拆 commit、清理陈旧分支、准备 PR 描述。这些活耗时不难，恰好适合外包。我在自己的几个仓库上跑了两三个月，整理出一套可复制的做法。

## 问题

实际痛点有三个：

1. 提交信息质量差，"fix"、"update" 满天飞，回溯时基本没用；
2. 一个 commit 混入不相关改动，revert 粒度很粗；
3. 分支只建不管，已合并的、废弃的堆了几十条。

直接把 shell 权限丢给 Agent 显然不行——`push --force`、`reset --hard`、`clean -fd`，任何一个误操作都够喝一壶。

## 做法

**第一步：收敛工具面。** 不给裸 shell，写了个约 100 行的包装脚本（也可以做成 MCP server），只暴露白名单命令：`status`、`diff --cached`、`log`、`branch -a`、`commit -m`、`switch -c`。脚本层面硬编码拒绝任何带 `--force` / `--hard` 的调用，每次执行都写审计日志。

**第二步：提交信息流程。** Agent 先看 `git diff --stat` 判断规模：小改动直接读完整 diff，生成 Conventional Commits 风格的信息；大改动按文件分组，建议拆成几个 commit 并列出各自的 staging 路径。默认 dry-run 输出建议，确认后才真正执行。

**第三步：分支巡检。** 在 OpenClaw 里挂了个定时任务，每天跑一次：列出已合入 main 未删除的分支、超过 30 天无新提交的分支，生成"建议删除清单"发到会话里。我批量确认后由 Agent 执行 `branch -d`——只允许 `-d` 不允许 `-D`，未合并的分支会被 Git 自己报错拦住，多一层保险。

**第四步：可追溯。** Agent 产出的 commit 一律带 trailer 标记来源，出问题可以一键过滤。

## 踩坑点

1. **别让 Agent 直接 push。** 我最早试过全自动推到个人分支，有一次它把本地落后远端的分支推了上去，覆盖了同事一个提交。现在的规则是：Agent 只操作本地，push 永远人工。
2. **diff 超限。** 一次大重构的 diff 直接撑爆上下文，Agent 开始"脑补"提交信息。解法是先 `--stat`，超过阈值（比如 500 行）就只做摘要、按文件分批读。
3. **提交信息失真。** 只喂文件名不喂 diff 时，Agent 会描述根本不存在的改动。所有提交信息必须基于真实 diff 输出生成，这条写进了 prompt。
4. **hooks 死循环。** 仓库里有 commitlint 和 pre-commit，Agent 遇到 hook 失败会无限重试。给执行循环加了最多 2 次重试，仍失败就终止，把原始报错抛给人。

## 可复用建议

- 白名单工具层 > 原始 shell，权限收敛是一切 Git 自动化的前提；
- 分阶段放开：先只读（status / log / diff）跑一周，再开 commit，最后才是分支操作；
- 危险操作一律走"建议 + 人工确认"队列，Agent 不做最终决定；
- 给 Agent 的产出打标（trailer 或 `agent/` 分支前缀），保留完整回溯链路。

## 总结

这套东西的价值不在炫技，而是把 Git 里 80% 的机械劳动外包出去，同时用护栏保证剩下 20% 的判断权留在人手里。把 Agent 当成一个权限受限的初级同事来用，体验会好很多。脚本和 prompt 模板我整理进了仓库，欢迎按各自的 hook 体系改造后复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/d441ee501de15883.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/ffe4a43ba3568281.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/293e42181e569042.png)

