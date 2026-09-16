---
title: 让 AI 助手接管 Git 提交与分支：一套可落地的自动化实践
feedId: 37837
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

日常开发里，Git 操作占了不少碎片时间：写 commit message、清理已合并分支、切分支前忘 stash、把调试代码误提交。这些事不难，但重复、易错。OpenClaw 的价值在于本地常驻 + 可挂工具（MCP / 插件 / shell），让 Agent 真正摸得到你的仓库，而不只是给聊天建议。这篇记录我让助手接管提交与分支管理的结构化做法。

## 问题

裸把 shell 权限丢给 Agent 是不行的，实际踩过的坑：

- 它可能凭文件名脑补改动，生成与 diff 不符的提交信息
- `push --force`、`reset --hard` 这类破坏性命令没有闸门
- `.env`、密钥容易被一锅端进暂存区
- 分支清理时误删未合并的工作

## 做法

我的方案分四层，可直接照搬：

**1. 收窄工具面。** 不给裸 shell，写一个 `git-ops` 包装脚本（或独立的 MCP server），只暴露 `status / diff / add / commit / branch / log` 这几个子命令，工作目录锁定到指定仓库根。破坏性命令要么不暴露，要么要求带 `--confirm` 参数由人触发。

**2. 约定写进仓库。** 在仓库里放一份 `AGENTS.md`：commit message 用 Conventional Commits、分支命名 `feat/xxx` `fix/xxx`、禁止直接提交 main。这份文件随仓库版本化，Agent 每次任务先读它，换机器、换人也一致。

**3. 提交流程固定化。** 每次提交 Agent 必须按序执行：`git status` → `git diff --cached` 逐文件过 → 检查密钥和大文件 → 基于真实 diff（而非文件名）生成 message → 输出摘要等确认。确认环节不能省，宁可慢十秒。

**4. 分支治理半自动。** 每周五让 Agent 跑一遍：列出已合并分支（`--merged`）和超过 30 天无提交的分支，生成清单，我勾选后批量删。远端分支只报告、不动手。

## 踩坑点

- **脑补 diff 是最大的坑。** 早期我只让它看文件列表，写出来的 message 像样但内容是错的。必须强制读真实 diff。
- **auto-push 千万别开。** 我试过提交即 push，一次不完整的提交直接进了共享分支，回滚成本很高。commit 留在本地，push 归人。
- **MCP 权限继承过宽。** 有个 server 默认拿到了整台机器的 shell，git 包装形同虚设。逐个检查工具的实际能力边界。
- **hook 和 Agent 不冲突。** pre-commit 的 lint、commit-msg 校验照常保留。Agent 不是替代 hook，是在 hook 之前多一道人机确认。

## 可复用建议

- **先只读，后写入。** 先让它只跑 status / diff / log 一周，观察判断质量，再放开 commit。
- **所有 git 命令落日志。** Agent 执行过什么要能回放，出问题才有据可查。
- **规则越短越好。** 长规则 Agent 记不全，不如固化成脚本让它调用。
- **做 dry-run 模式。** 先输出“将要执行什么”，人点头再真跑，调试期尤其有用。

## 总结

AI 助手管 Git，本质是把“规则明确、爆炸半径低、高频重复”的操作交出去，把不可逆操作和对外动作（push、删远端分支）留在人手里。这套结构跑了一个多月，commit 质量明显稳定，分支列表第一次清爽了。它没让我少干活，但让我少犯错——对工具来说，这已经是很好的定位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/10c824070835ce17.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/d617eb55632b5f6d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/77f24b672f59c88d.png)

