---
title: Git 自动化实战：让 OpenClaw 管提交和分支，但别把钥匙全给它
feedId: 42845
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

日常开发里 Git 操作高度重复：写 commit message、切分支、清理已合并分支、push 前跑检查。这些活不值得人工花时间，但完全放开自动又危险。OpenClaw 的 exec 工具加 MCP git server 恰好能夹在中间——让 Agent 出手，让人守闸门。

## 问题

我仓库里的三个真实痛点：

1. commit message 质量不稳定，经常偷懒写 `update`；
2. feature 分支合完就忘，堆了 30+ 个 stale branch；
3. push 前总想不起来跑 lint 和 test。

## 做法

**第一步：权限收窄，只读起步。** 不要一开始就让 Agent 拿完整 Git 权限，工具 allowlist 里先只放只读命令：

```json
{ "allow": ["git status", "git diff*", "git log*", "git branch"] }
```

跑两周，确认它对仓库状态的判断靠谱，再逐步放开 `add` / `commit`。

**第二步：提交信息生成，diff 锚定 + 人工确认。** 固定提示词：

```
根据 git diff 输出生成 commit message：
- 只描述 diff 中实际存在的变更
- conventional commits 格式，首行不超过 50 字符
- 输出候选后停下，等我确认再执行 commit
```

关键是"停下等确认"，这一步不要省。

**第三步：分支清理做成定时任务。** 用 OpenClaw 的 cron heartbeat 每周跑一次：列出已合并到 main 的本地分支，发清单到频道，我回复"删"它才执行 `git branch -d`。原则是：已合并才可删，未合并只报告不动。

**第四步：commit 后自动播报。** Agent 提交后自动跑 lint + test，结果摘要发到 Telegram，失败时附上出错文件的 diff 位置，省掉手动回查。

## 踩坑点

1. **绝不放开 `push --force` 和 `reset --hard`**，建议在 allowlist 里显式 deny，不只靠提示词约束。
2. **Agent 会脑补 diff 里没有的变更**。生成的 message 我会逐条对应 diff 检查，发现编造就打回重写。
3. **大仓库全量 diff 撑爆上下文**。让它先跑 `git diff --stat`，再挑相关文件细看。
4. **多 worktree / submodule 下 cwd 容易错位**。要求它每次先 `git rev-parse --show-toplevel` 确认仓库根。
5. **非 TTY 环境 GPG 签名会卡死**。Agent 环境要么配好签名缓存，要么临时 `-c commit.gpgsign=false`。

## 可复用建议

- **权限渐进**：只读 → add/commit → push → 定时任务，每档观察一周再升级。
- **两段式写操作**：Agent 只"提议"，执行键始终在人手里。
- **规则固化进 AGENTS.md**，别每次口头交代，团队其他人也能直接复用。
- **给 Agent 单独配 git identity**（bot 邮箱），事后审计哪些提交是它做的，一查便知。

## 总结

Agent 做 Git 自动化的价值不在"全自动"，而在**把重复判断自动化、把危险操作人审化**。OpenClaw 的 exec、定时任务和频道播报刚好拼出这条流水线。两周跑下来，我的 commit message 规范率接近 100%，stale branch 从 30+ 降到个位数。投入不大，产出是实打实的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/6fad2723f44c010f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/67d263ae19bdd503.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/742af5bd86cd725d.png)

