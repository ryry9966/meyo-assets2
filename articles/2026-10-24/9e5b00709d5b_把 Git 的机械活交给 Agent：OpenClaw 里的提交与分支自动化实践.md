---
title: 把 Git 的机械活交给 Agent：OpenClaw 里的提交与分支自动化实践
feedId: 42474
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

一个人维护十来个中小仓库，Git 里的机械劳动占比不低：对着 diff 憋提交信息、切分支前忘 stash、合并完忘删旧分支。OpenClaw 的 agent 本来就有 shell 和文件工具，我花了一个周末把「读 diff、拟 message、管分支」交给它，跑了一个多月，记录一下做法和代价。

## 问题

真正的难点不是“能不能执行 git 命令”——那一步毫无难度，而是三件事：

1. **信息质量**：模型凭记忆写 commit message，和实际 diff 对不上，比人手写的还离谱。
2. **安全性**：`reset --hard`、`push --force`、`clean -fd` 这类不可逆操作交给 agent，等于把仓库备份的责任也交了出去。
3. **脏提交**：agent 图省事 `git add -A`，把 `.env.local` 和构建产物一起带进暂存区。

核心思路一句话：机械操作交给 agent，不可逆触发器留给人类。

## 做法

**1. 环境隔离。** 用 `git worktree` 给 agent 单独开一个工作区，工作目录锁死在里面，它怎么折腾都不影响你正在编辑的主工作区。

**2. 权限分层。** 在 OpenClaw 的工具权限里做两层控制。允许：`status`、`diff`、`log`、`branch`、指定路径的 `add`、`commit`、`stash`；拒绝或需人工确认：`push`、`reset`、`clean`、`rebase`、`merge` 以及一切 `--force` 变体。拒绝清单比允许清单更重要——灾难性命令就那几条，先堵死。

**3. 约定写进 AGENTS.md。** agent 每次会话自动继承，关键四条：

- 先跑 `git diff` 再拟 message，禁止凭记忆写
- 禁止 `add -A`，只允许逐路径 add
- 提交信息遵循 Conventional Commits，正文写“为什么”而不是“改了什么”
- 分支命名 `agent/<主题>-<日期>`，一个任务一个分支

**4. 流程固定四步。** status → diff → 拟 message 给我看 → 确认后提交。每一步要求 agent 贴出关键输出，我扫一眼就能判断它有没有在瞎编。

**5. WIP 兜底。** 要求它每完成一小步就往 `agent/wip` 分支提交一次，即使后面出事故也丢不了多少东西——这条是被教训教育出来的。

## 踩坑点

- **第一周就被 add -A 坑了**：`.env.local` 进了暂存区，靠 pre-commit 的密钥扫描才拦住。现在逐路径 add 之外又加了一道 hook。
- **message 幻觉**：diff 还没跑，提交信息就“写好了”，内容是上一个任务的。“先 diff 后 message”从此是硬规则。
- **自作主张清理**：它把“整理仓库”理解成 `reset --hard` 掉未跟踪文件。此后破坏性命令只走配置拒绝，不再靠提示词约束。
- **GPG 签名失败**：agent 的 shell 环境没配签名密钥，提交一直报错。在专用环境预配置密钥，比让它加 `--no-gpg-sign` 靠谱——后者会破坏审计。
- **分支蔓延**：跑了两周多出十几个 `agent/*` 分支。加了每周定时任务：agent 列出已合并分支、给删除建议，我确认后统一清。

## 可复用建议

1. 先跑一周只读模式，只放 `status`/`diff`/`log`，看它的操作日志是否靠谱，再放开 commit。
2. 破坏性命令靠权限配置拒绝，不靠提示词恳求——提示词会被长对话稀释，权限不会。
3. `push`、`merge`、`force` 这条线永远留在人手里，agent 的产物止步于本地提交。
4. AGENTS.md 里的 git 约定可以直接复用到任何仓库，换会话、换机器都生效。
5. 出一次事故沉淀一条规则，清单会自己长成适合你仓库的样子。

## 总结

一个多月的结论：agent 负责读 diff、拟信息、保持分支整洁这些耐心活，人负责按下不可逆的按钮。提交信息质量确实上去了——它读 diff 比我有耐心，分支也不再堆积。前期成本约一小时（隔离环境 + 权限清单 + AGENTS.md），对多仓库维护者来说很划算。如果你已经在用 OpenClaw 跑日常任务，git 是低风险的自动化起步对象——前提是先把拒绝清单写好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/f33c41cd3f1360cb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/60810d453654f336.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/0e5b4074f65f2b6e.png)

