---
title: Git 自动化实践：让 Agent 接管提交与分支管理
feedId: 42220
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

写代码时最容易被敷衍的就是 Git 操作：commit message 随手一个 `fix`，分支叫 `test2-final`，改完的分支忘了删。这类事不复杂，但琐碎、高频，正好适合交给 AI 助手。OpenClaw 挂上 MCP 工具或受控 shell 后，Agent 完全有能力接管这部分流程，前提是你得先给它立好规矩。

## 问题

直接说一句"帮我提交"，通常会踩三类坑：

1. **信息不足**。Agent 只看 staged diff 就写 message，丢掉了"为什么改"的上下文，描述往往正确但没用。
2. **权限过大**。默认它能执行 `git add .`、`push --force`、`reset --hard`，任何一条出事都够你修一晚上。
3. **分支失控**。每个任务开新分支是好事，但没人负责回收，两周后仓库里躺着二十几个 `agent/xxx`。

## 做法

我们在团队仓库里跑了一版最小可行方案，分四步：

**第一步：把规范写进仓库。** 在仓库根目录放一份 OpenClaw 会读取的指令文件，写清 Conventional Commits 格式、分支命名规则（`feat/`、`fix/`、`chore/` 前缀 + 小写短横线），以及"提交前必须先展示 diff 摘要"。规范放进仓库而不是聊天里，所有人和所有 Agent 共用同一份，还能随代码走版本管理。

**第二步：收窄工具权限。** 不给裸 shell，改用白名单式的 git 工具，只放行 `status / diff / add / commit / branch / switch / log`。push 做成需要人工确认的操作，`push --force` 和 `reset --hard` 干脆不提供。

**第三步：固化提交链路。** 要求 Agent 每次提交走固定流程：`git status` 确认改动范围 → 按文件分批 staging（禁止 `add .`）→ 生成 message 并附一句"本次不包含的改动" → 等确认后 commit。多任务并行时，强制先建分支再动工。

**第四步：用 hook 做硬校验。** Agent 的自觉不可靠，commitlint 加 pre-commit hook 才是底线。message 不合规直接拒绝提交，Agent 收到报错会自己重试——这比在 prompt 里反复叮嘱管用得多。

## 踩坑点

- **`git add .` 把 `.env` 提了上去**。发生在方案早期，之后在指令文件里明确列出禁提交清单，并补了 secrets 扫描的 pre-commit hook。
- **一锅炖提交**。Agent 倾向把重构和 bugfix 塞进同一个 commit，回滚时会后悔。明确要求"一次提交只做一件事"后明显好转。
- **diff 截断**。大改动时 Agent 读不全 diff，message 就开始编。规定超过一定行数必须分块读取，或者先跑测试再写描述。
- **两个 Agent 抢同一个仓库**。同时切分支互相干扰，后来用 `git worktree` 按工作目录隔离，各自独立操作。

## 可复用建议

- 规范放仓库内的指令文件，跟随代码版本管理，别散落在聊天记录里。
- 能用 hook 硬校验的，就不要指望 prompt 软约束。
- 危险命令在工具层直接移除，而不是写"请不要执行"。
- 分支清理做成每周定时任务：Agent 列出已合并分支，人工确认后删除。

## 总结

Git 自动化本身不难，难的是把工程习惯翻译成 Agent 可执行的约束。我们的经验浓缩成四句话：规范进仓库、权限进白名单、校验进 hook、清理进定时任务。做完这些，Agent 管提交基本不用你操心，你只需要在 push 前看一眼。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/5ead2afac1e8b43d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/71ec8152a5be2457.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/ab2c5613b2e4af24.png)

