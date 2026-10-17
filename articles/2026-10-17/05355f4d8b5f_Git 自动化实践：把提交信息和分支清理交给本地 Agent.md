---
title: Git 自动化实践：把提交信息和分支清理交给本地 Agent
feedId: 41786
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

本地跑 OpenClaw 的用户，大多早就给了它 shell 权限。而仓库里 80% 的 git 操作是重复劳动：写 commit message、按规范命名分支、清理过期分支、提交前检查。这类确定性强的杂活很适合交给 agent——前提是权限边界先划清楚，否则它帮的忙和闯的祸一样多。

## 问题

直接丢一句"帮我 commit"给 agent，通常会遇到：

- 提交信息靠猜：看文件名写 message，不读真实 diff；
- 顺手 `git add -A`，把 `.env`、构建产物一起提上去；
- 在 main 上执行 `reset --hard` 甚至 `push --force`；
- 多仓库场景下 cwd 混乱，改了 A 仓库却提交到 B。

这些不是 agent"笨"，而是我们没给它一个受限的执行面。

## 做法

下面是社区里验证过的一套最小方案，假设 OpenClaw 已配置 shell 或 MCP git 工具。

**1. 权限收敛：包装脚本替代裸 git**

不让 agent 直接调 git，而是调一个 `agent-git`，白名单放行子命令：

```bash
ALLOWED="status|diff|add|commit|branch|switch|log|stash|fetch"
[[ "$1" =~ ^($ALLOWED)$ ]] || { echo "blocked: $1"; exit 1; }
echo "$(date) $PWD $*" >> ~/.agent-git.log
exec git "$@"
```

`push`、`reset --hard`、`rebase` 一律不放行。需要推送时人工确认，或只对 agent 自建的分支开放。

**2. 用 skill 固化工作流**

给 agent 定义一个 git-commit skill，核心规则：

- 提交前必须先跑 `git diff --cached`，基于真实变更写 message；
- message 遵循 conventional commits，但禁止编造 scope；
- 逐文件 add，不用 `-A`；遇到 `.env`、`*.pem` 直接中止并报告；
- commit 完成即停，不主动 push。

**3. 服务端守门：hooks**

提示词能被绕过，hooks 绕不过：

```bash
# .git/hooks/pre-commit
gitleaks protect --staged || exit 1
```

再加一个 commit-msg hook 校验 message 格式。提示词管习惯，hooks 管底线。

**4. 分支自动化：只做"报告 + 待确认"**

用定时任务让 agent 每天输出分支报告：已合并未删除的、超过 30 天无提交的、与 main 无差异的。删除动作必须回到对话里人工确认。同时约定命名（如 `agent/<task-slug>`），一眼识别哪些分支是 agent 建的、可以放心清理。

## 踩坑点

- **cwd 是最大的坑**。会话可能跨仓库，每次操作前强制 agent 先跑 `git rev-parse --show-toplevel` 确认当前仓库；
- 长会话里 agent 记的是几小时前的分支状态，早已过期。规则：任何写操作前先 `git status` 刷新事实；
- 装 MCP git 工具前审一遍它封装了哪些子命令，封装越"全"越要确认是否包含 push/rebase 这类高危操作；
- 合并冲突未解决时让 agent 提交，冲突标记会直接进历史。pre-commit 里加 `git diff --check` 能拦住大部分。

## 可复用建议

- 原则一句话：让 agent 做**读多写窄**的操作——diff、log、分支报告随便做；改写历史、动共享分支、推送，一律走人工；
- 把包装脚本 + hooks + skill 提示词打包成模板仓库，新项目复制即用；
- 权限按仓库粒度配：个人实验仓库可放宽，团队仓库收紧。

## 总结

Git 自动化的价值不在"全自动"，而在把确定性强的杂活交给 agent，把不可逆的操作留给人。包装脚本限权、hooks 兜底、审计日志回溯，三层叠加之后，agent 参与 git 工作流才是安全的。欢迎在社区贴出你的 `agent-git` 白名单配置，一起迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/3b95e5dc02d7c7df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/6e5debf03a2d6277.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/901a44adbac6019d.png)

