---
title: 让 AI 助手接管 Git 日常：提交与分支自动化的工程实践
feedId: 37761
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

日常开发里，Git 操作占了相当一部分"低价值、高频率"的时间：写提交信息、建分支、清理已合并分支、补变更记录、写 PR 描述。OpenClaw 这类常驻 AI 助手天然适合接手这些活——它有命令执行能力、有定时任务（heartbeat/cron），也能通过 MCP 接入 git 工具。但"能跑 git"和"放心让它跑 git"是两回事，这篇帖子记录我们让 Agent 管理提交与分支的一套做法。

## 问题

- 提交信息质量差：`update`、`fix` 满天飞，回溯时等于没有记录。
- 分支堆积：本地几十个已合并分支没人删，远端 feature 分支生命周期混乱。
- 直接给 Agent 完整 shell 风险高：误 `push --force`、误提交 `.env`、在 main 上直接 commit，都是真实发生过的故障模式。

## 做法

1. **收窄工具面。** 不把裸 shell 丢给 Agent，包一层 `git-safe` 脚本：只放行 `status / diff / log / branch / add / commit / push` 等白名单子命令，显式拒绝 `--force`、`reset --hard` 出现在共享分支上下文。或直接用社区的 MCP git server，在配置里同样做操作裁剪。
2. **把约定写成文件。** 仓库里放一份 `CONVENTIONS.md`：Conventional Commits 格式、分支命名前缀（`feat/` `fix/` `chore/`）、受保护分支列表。Agent 执行前先读这份文件，后续调整约定不用改 prompt。
3. **提交流程分两步。** Agent 先 `git diff --staged` 生成符合约定的 commit message，打印出来供人确认（低风险场景可直接执行，但命令全量记录日志），再执行 commit。message 必须基于 staged diff，不允许它"顺手"写没发生的内容。
4. **分支巡检做成定时任务。** 每天一次：列出已合并进主干的本地分支、超过 30 天未动的远端分支，输出报告。删除默认只做本地，远端删除走 PR 或人工确认。
5. **hooks 兜底。** `commit-msg` hook 校验 message 格式，pre-commit 跑密钥扫描。Agent 写错格式或试图提交敏感信息时被 hook 拦截，收到报错后自行修正。

## 踩坑点

- **`git add -A` 是事故源头。** Agent 很爱用它，会把本地杂物一起提交。后来约定只能 `add` 明确列出的路径。
- **交互式命令会卡死。** `rebase -i`、带编辑器的 merge 都不行，全部换成非交互等价命令。
- **重试风暴。** hook 拦截后 Agent 会反复重试，要限定"同一操作最多重试 2 次，仍失败就停下来报告"。
- **AI 会脑补。** 只让它看 staged diff，别给它全仓库上下文自由发挥，否则 message 描述和实际改动对不上。

## 可复用建议

- **自主性分级放开**：先只读（status/log/diff 摘要、分支报告），稳定后放开 commit，最后才是 push；push 到共享分支永远保留人工确认。
- **审计日志**：Agent 执行的每条 git 命令落盘，出问题能回放。
- **约定放文件里，prompt 只管流程**：约定变更不触碰 prompt，减少维护成本。

## 总结

让 AI 助手管 Git，价值不在"全自动"，而在把格式化、巡检、清理这类确定性工作稳定交出去，同时用白名单、hooks 和分级授权兜住风险。先让只读报告跑两周，你会很清楚自己的 Agent 能被信任到哪一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/cbb951e8588190fb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/d1dc87bb4be295a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/563cb53f14de194c.png)

