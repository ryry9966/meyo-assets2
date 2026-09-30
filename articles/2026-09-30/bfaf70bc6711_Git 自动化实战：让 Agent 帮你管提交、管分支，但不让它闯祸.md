---
title: Git 自动化实战：让 Agent 帮你管提交、管分支，但不让它闯祸
feedId: 39802
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

最近把日常 Git 操作逐步交给了 agent：写 commit message、整理分支、发布前的 cherry-pick。跑了两三个月，踩了不少坑，把这套相对稳妥的做法整理出来。

前提：你的 agent 已经能调用 shell 类工具（自定义 MCP tool 或终端插件），仓库在本地有完整读写权限。

## 问题

手动 Git 的痛点不在命令本身，而在于：

- commit message 写作疲劳，团队规范全靠自觉
- 分支只增不减，半年后 `git branch` 刷满一屏
- 跨仓库重复操作（统一升版本号、批量回滚补丁）容易漏

而直接放开让 AI 执行 git 命令，风险也很明确：它可能在错误的分支上 rebase、把 `.env` 提交进去、甚至在不该动的远端执行 `--force`。

## 做法

核心思路三层：**能力收敛、规则前置、关键动作人工确认**。

### 1. 收敛可执行命令

不要给 agent 裸 shell，包一层 git 专用工具，内部白名单：

```text
允许: status / diff / log / add / commit / switch -c / merge --no-ff / push origin <branch>
禁止: push --force / reset --hard 越过远端 / clean -fd / 交互式 rebase / --no-verify
```

交互式 rebase 要特别说明：agent 无法响应编辑器交互，遇到 `rebase -i` 会卡死或行为异常，需要拆成非交互流程。

### 2. 提交流程固定化

在系统提示词或约定文件里写死流程：

1. 先 `git status` + `git diff --stat`，确认改动范围
2. 检查是否有不该提交的文件（密钥、构建产物、环境配置）
3. 读 diff 内容，按 Conventional Commits 写 message
4. add 指定文件，**禁止 `add -A`**
5. commit 后把 hash 和 message 回报给用户

第 2 步和第 4 步是重点。除了 `.gitignore`，再维护一份人工黑名单（`.env`、`*.pem`、`dist/`），写进 prompt 让 agent 每次自检。

### 3. 分支生命周期

- 新任务开始时：`git fetch && git switch -c feat/xxx main`，命名带任务关键词
- 合并后不立即删，先列"候选清理清单"，人确认后批量删
- 每周跑一次 stale 扫描：`git branch --merged main`，列出可安全删除的分支

### 4. 大 diff 的处理

diff 超过约 500 行会撑爆上下文，agent 开始瞎编。约定分批看：先 `--stat` 总览，再按文件逐个 diff。

## 踩坑点

- **工作目录漂移**：agent 有时在项目根目录之外执行 git，拿到别的仓库状态。解法：工具封装里锁死 repo 路径，每次调用先校验 cwd。
- **detached HEAD**：agent checkout 某个 commit 后继续提交，提交悬空。规则加一条：commit 前必须确认在分支上（`git symbolic-ref --short HEAD` 成功才行）。
- **hooks 失效**：agent 用了 `--no-verify` 跳过 pre-commit（有时是训练数据里的坏习惯）。明确禁用这个 flag，让 hook 成为最后一道防线。
- **提交拆分**：一次改动牵扯多个逻辑时，agent 倾向打一个大 commit。要求它先描述拆分方案，人点头再执行。

## 可复用建议

- **白名单 + prompt 规则双保险**：别只靠 prompt，长对话会稀释规则。
- **pre-commit hook 做敏感文件扫描**，和 agent 自检形成冗余。
- **破坏性操作独立成工具**：`reset --hard`、`push --force`、`branch -D` 单独封装，需人工返回确认才执行。
- **流程写进仓库的 AGENTS.md**：agent 每次会话自动加载，团队成员复用同一套规则，不用各自调教。

## 总结

Git 自动化的价值不在"少敲几条命令"，而在于把提交规范、分支卫生这些容易被忽略的事变成流程默认。前提是权限收敛和确认机制做在前面——agent 负责繁琐和重复，人负责不可逆的决策。这套做法在任何支持 shell / MCP 工具调用的 agent 框架上都能迁移，核心三句话：白名单执行、敏感动作确认、流程写进约定文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/7fd93c44ea971908.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/f91daab1ef04038b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/5bdf9387d111f2ac.png)

