---
title: Git 自动化实践：让 AI 助手管住 commit 和分支
feedId: 41815
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

最近把 OpenClaw 通过 MCP 接入了 git 工具（对 git 命令做了封装的 server），让 agent 参与日常提交与分支管理。跑了一个多月，结论先说：适合接管"重复且有章可循"的部分，不适合完全托管。这篇记录具体做法和踩过的坑。

## 问题

没有 agent 介入时的痛点很典型：

- commit message 靠手写，经常出现 "fix"、"update" 这类无效信息，回头查历史全靠猜；
- 功能合并后本地分支越积越多，stash 里常年躺着不知道能不能删的改动；
- 让 agent 直接跑 git 命令风险不小：它可能把调试代码一起 add 进去，甚至在错误分支上 commit。

核心矛盾是：agent 执行力很强，但对 Git 的"不可逆操作"缺乏敬畏。

## 做法与步骤

**第一步：权限分层。** 把 git 操作分三级：只读（status / diff / log / branch -a）直接放行；本地写（add / commit / branch / switch）允许，但要求执行前后各跑一次 status 做校验；远程及破坏性操作（push / rebase / reset --hard / clean）一律进确认清单，agent 只能给出建议命令，由人确认后执行。在 OpenClaw 的工具白名单里配置即可，比事后救火划算。

**第二步：commit message 流程固化。** 给 agent 固定流程：先 `git diff --stat` 总览，再针对关键文件看细节 diff，最后按 Conventional Commits 输出 type + scope + 摘要。要求 message 描述"为什么改"，不罗列文件名——文件名 diff 里本来就有。

**第三步：分支生命周期自动化。** 约定命名 `feat/<issue号>-<关键词>`，agent 建分支时自动带 issue 号；合并后让它扫描本地已合并分支（`git branch --merged main`）生成清理清单，删除仍由人执行。stash 每次会话开始时汇报一次，超过两周的标记提醒。

**第四步：规则沉淀。** 把所有约定写进项目级规则文件，新会话自动生效，避免每次口头交代。

## 踩坑点

- **agent 贪心地 `git add .`**：早期它把 .env.local、调试日志一并提交过。后来明确要求按文件粒度 add，执行前先自报"本次包含哪些文件"。
- **大 diff 撑爆上下文**：一次重构涉及 80+ 文件，读全量 diff 直接失败。改成先看 stat，再分批看重点文件。
- **detached HEAD 迷路**：agent 在某个 tag 上 checkout 后，后续 commit 全部悬空。规则加了一条：任何 commit 前必须确认当前分支名非空。
- **rebase 被滥用**：它试图用 rebase "整理"共享分支历史，差点翻车。现在共享分支只允许 merge，rebase 限定在个人 feature 分支。
- **stash 恢复冲突**：恢复前没检查工作区状态，冲突后它自作主张 `checkout -- .`，丢了半天的改动。从此 `checkout --`、`reset --hard`、`clean` 全部进确认清单。

## 可复用建议

1. 权限分级 + 执行前后 status 校验，是最便宜的安全网；
2. 让 agent"先汇报再动手"：add 哪些文件、commit 到哪个分支，说清楚再执行；
3. 不可逆命令一律走人工确认，不要赌 agent 的判断力；
4. 约定写进规则文件而不是聊天记录，否则换个会话就失效；
5. 定期让 agent 出一份"仓库体检"：未推送提交、过期 stash、可清理分支，比事后翻历史省力。

## 总结

AI 助手管 Git 的正确姿势不是"全自动"，而是"它执行、你把关"。把有章可循的部分（message 规范、分支命名、清理清单）交给 agent，把不可逆决策留在人手里。一个月下来，commit 历史可读性明显提升，本地分支从 30+ 降到个位数，没有再发生误提交。工具没有魔法，流程设计才是关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b803441290ff5e6e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/a36240dc61812bf1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/f928ed9827ffe469.png)

