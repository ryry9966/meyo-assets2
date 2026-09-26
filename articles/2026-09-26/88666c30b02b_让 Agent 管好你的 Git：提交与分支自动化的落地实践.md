---
title: 让 Agent 管好你的 Git：提交与分支自动化的落地实践
feedId: 39123
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

日常开发里，git 操作占了大量琐碎时间：写 commit message、拆分提交、清理合并后的残留分支。OpenClaw 的 agent 本来就能执行 shell 命令，把这些事交出去看起来顺理成章。但实践下来发现，"能让它跑 git" 和 "能放心让它跑 git" 是两回事，这里记录一套跑了两月的分阶段方案。

## 问题

手动侧的痛点很典型：
- commit message 质量不稳定，`fix`、`update` 满天飞，两周后自己都看不懂；
- 一次改动横跨配置、逻辑、样式，懒得拆提交，历史一团糟；
- 合并完的分支不删，本地分支列表越来越长。

而直接把 git 交给 agent 又有新风险：`push`、`checkout -- .` 这类不可逆操作一旦出错代价高；几千行的 diff 撑爆上下文后，agent 会开始"猜"文件内容。

## 做法

核心思路是分三个阶段，权限逐级放开：

1. **只读阶段**。只允许 `git status / diff / log / branch`。在 workspace 的 AGENTS.md（或 MCP git server 配置）里写清规范：Conventional Commits、scope 命名、message 语言。硬性要求 agent 每次先跑 `git diff --stat` 再看具体 diff，禁止凭记忆总结改动。

2. **本地写阶段**。放开 `add / commit / branch / stash`，但 push 必须人工确认。实际流程是：agent 读完 diff，给出分组方案（比如"配置和业务逻辑拆成两个 commit"）并附上拟好的 message，我回复确认后才执行，全程写日志。

3. **分支自动化阶段**。用 OpenClaw 的定时任务每天跑一次分支巡检：列出超过 14 天未动的分支、已合并进 main 的残留分支，生成报告推到聊天窗口。删除默认只列出，手动确认后才执行，且只用 `branch -d` 不用 `-D`，让 git 自己兜底未合并的分支。

## 踩坑点

- **偷懒总结**：早期 agent 会不看 diff 直接编 message，把不相关的文件写进描述。解决办法是强制它在 message 里引用 diff 中的具体函数名或文件名，拿不出证据就打回重写。
- **误提交密钥**：有一次 `.env.local` 被加进了暂存区。现在规则里硬编码 blocklist，遇到 `.env*`、`*.pem` 直接拒绝 add。
- **上下文爆炸**：超大 diff 会让输出质量骤降。改成先看 `--stat`，按文件挑重点，必要时分批处理。
- **危险命令漏禁**：一开始没禁 `git checkout -- .`，agent 为了"清理工作区"真的用过一次，丢了两小时手动改动。现在恢复类、回滚类命令一律人工执行。

## 可复用建议

- 一句话原则：**agent 提案，人类裁决**。凡是不可逆操作（push、reset、`-D`、`checkout --`），确认权不交给模型。
- 规范写进仓库里的 AGENTS.md 或 MCP 配置，跟着代码走版本管理，不要散落在聊天记录里。
- 白名单比黑名单可靠，逐步放开比一次放开稳。
- 要求 agent 每次操作后输出一行摘要日志，回溯问题时能省一半时间。

## 总结

两个月下来，commit message 质量明显稳定，本地分支从 30 多个降到个位数。这套方案并没有让 git"全自动"，而是把最耗时的读 diff、写描述、巡检这些事交出去，把不可逆决策留在人手里。已经在用 OpenClaw 跑自动化的同学，建议从只读巡检起步，观察一周再放开提交权限——慢一点，但睡得着。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/e48678b59237fe04.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1bfef0414ebe6fd3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/20d190e1ff699434.png)

