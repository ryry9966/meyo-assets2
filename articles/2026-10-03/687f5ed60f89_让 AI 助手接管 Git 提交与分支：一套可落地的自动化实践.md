---
title: 让 AI 助手接管 Git 提交与分支：一套可落地的自动化实践
feedId: 40206
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

我日常用 OpenClaw 做开发辅助，通过 MCP 的 shell 工具给它一个受控的执行环境，可以读仓库、跑命令。用了一段时间后发现，Git 这块最值得交给 agent：commit message 想不出来、分支开完忘删、提交前忘了 diff——这些机械动作正是 AI 擅长的。

## 问题

手工 Git 工作流有三个痛点：

1. 提交信息质量不稳定，回看历史时经常找不到那次改动的上下文；
2. 分支越积越多，合并完忘删，remote 上挂着一堆废弃的 `feature/*`；
3. 提交粒度随意，一个 commit 里混着重构、修 bug 和格式化，想回滚只能整块丢弃。

## 做法

**Step 1：权限与沙箱先行。** 限制 agent 只能操作工作目录，git 子命令用白名单放行（`add / commit / branch / switch / diff / log / status / push`），明确禁掉 `reset --hard`、`clean -fd`、`push --force`。

**Step 2：提交流程固化成一个 skill。** 固定顺序为：先 `git status` + `git diff --staged` 拿到真实改动 → 按 Conventional Commits 生成 message，body 里写清动机 → message 输出给我确认，我回"ok"才执行 commit → push 只推当前分支，不碰 main。

**Step 3：分支管理自动化。** 每天定时跑一次：列出已合并分支（`git branch --merged main`），生成清理清单，我确认后批量删；新建任务时让 agent 按约定命名，如 `feat/xxx`、`fix/xxx`。

**Step 4：提交前自检。** commit 前让 agent 检查暂存区是否混入 `.env`、密钥、构建产物，有就停下来报告，而不是自作主张处理。

## 踩坑点

- **`git add -A` 是大坑。** 我最初让 agent"提交所有改动"，它把本地调试文件和一份 `.env.local` 全提交了。后来改成必须先列文件清单，再逐个 add。
- **大 diff 会撑爆上下文。** 几百个文件的改动直接丢给 agent，message 质量反而下降。先看 `diff --stat`，再挑关键文件看详细 diff，效果好得多。
- **index.lock 冲突。** 两个 agent 会话同时操作同一仓库会撞锁。规定一个仓库同一时间只允许一个 agent 会话，或用 worktree 隔离。
- **幻觉 message。** agent 偶尔根据文件名"脑补"提交内容，不看 diff。prompt 里强制要求引用 diff 中的具体函数或模块名，能明显压下去。
- **别让 agent 自动 rebase。** 遇到冲突时它的处理很冒险，这部分保持人工。

## 可复用建议

- 一切破坏性操作（reset / clean / force-push / rebase）默认禁用，需要时人工执行；
- commit 前确认 message、push 前确认目标分支，这两道闸不要省；
- 把 skill / prompt 放进仓库做版本管理，团队共享、可回溯；
- 定时清理 + 确认清单的模式，比全自动删除安全得多；
- 日志要留：agent 执行过的每条 git 命令都记录，出问题能复盘。

## 总结

AI 助手管 Git，价值不在"全自动"，而在把机械环节自动化、把决策环节留给人。权限收紧、确认闸门、日志留痕三件事做扎实，这套流程基本不会翻车。目前我的提交信息质量和分支整洁度都有肉眼可见的改善，值得一试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/6da8b01f2a611508.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/2508a6712acee45d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/2ed49dfef41c1b75.png)

