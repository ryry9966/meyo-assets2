---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39074
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 的 Agent 不是靠一坨硬编码 prompt 跑起来的。它的 workspace 里躺着几个 Markdown 文件：AGENTS.md 管操作守则，SOUL.md 管行为底色，USER.md 记用户画像，MEMORY.md 存长期记忆。其中最小、最容易被忽略的是 IDENTITY.md——它回答"我是谁"。默认模板就几行：Name、Creature、Vibe、Emoji、Avatar。

## 问题

没有身份定义的 Agent 会出现三类实际问题：

1. **自我介绍不稳定**：今天自称"助手"，明天自报底层模型名，跨会话不一致；
2. **多实例难区分**：同时跑工作号和生活号两个实例，日志和对话里分不清谁是谁；
3. **人格漂移**：语气全靠模型即兴发挥，长期使用后越来越随机。

这不是"玄学体验"问题，而是可观测性问题：身份是排查对话风格异常时的第一个锚点。

## 做法

1. 找到文件：默认在 `~/.openclaw/workspace/IDENTITY.md`（多 Agent 时每个实例各一份）。
2. 填关键字段，控制在 10 行以内：

```markdown
# IDENTITY.md

- **Name:** Clawsy
- **Creature:** 一只机器人螃蟹
- **Vibe:** 稳重、话少、爱用列表
- **Emoji:** 🦀
- **Avatar:** avatars/clawsy.png
```

3. 新开会话验证：直接问"你是谁、用什么表情"，确认注入生效。
4. 让它可进化：把 workspace 纳入 git，每次身份调整（改名、改 Vibe）单独成一次 commit。可以让 Agent 自己提案——"你觉得现在的 Vibe 还贴切吗"——但合入必须人工审批。
5. 多实例场景按用途命名：工作实例偏克制、个人实例偏轻松，Avatar 用不同图片，一眼可分。

## 踩坑点

- **会话中途改文件不生效**。身份在会话启动时注入上下文，改完要新开会话。
- **别把行为规则写进 IDENTITY.md**。"不要闲聊"这类指令属于 SOUL.md / AGENTS.md。身份是描述性的（我是谁），不是约束性的（我该怎么做），混写会导致自述和行动拧巴。
- **别让 Agent 自动改写**。没有 git 和人工审批的身份，漂得亲妈都不认识。
- **Avatar 路径写错不报错**，只是静默 fallback，配完记得肉眼确认。
- 中文书写没问题，模型读得懂；但全文件语言保持一致，避免自述时中英混杂。

## 可复用建议

- 把 IDENTITY.md 当**配置管理**，而不是聊天产物：小、结构化、版本化。
- 和 USER.md 成对设计：身份定义 Agent，USER 定义用户，两者共同决定对话气质。
- 建立"身份回顾"节奏：每完成一个大项目或每隔两三周，review 一次 Vibe 是否还准。
- 排障先查身份：对话风格突变时，diff 一下 IDENTITY.md 和 SOUL.md 的最近提交，往往一眼定位。

## 总结

IDENTITY.md 是 OpenClaw 里成本最低、杠杆最高的文件之一：十几行字，换来跨会话的稳定自述、多实例的可辨识度和一条可控的进化路径。它的价值不在"给 AI 起名"这种拟人化趣味，而在工程上把"Agent 是谁"变成一个可审计、可回滚的版本化配置。花十分钟写好它，比之后在十次对话里反复纠正自我介绍划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/8c35c56d33364ae1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/905830aeaad3e23b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/bfb75a4f6c3b4ee7.png)

