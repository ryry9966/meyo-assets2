---
title: IDENTITY.md：给 Agent 一份可进化的身份档案
feedId: 43184
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

聊 OpenClaw 的 workspace 文件时，大家关注点通常在 SOUL.md 和 MEMORY.md，`IDENTITY.md` 存在感偏低，但它解决的是另一个维度的问题：**这个 Agent 是谁**。

它的内容很朴素：名字、物种/形象设定、气质关键词、emoji、头像备注。每次会话启动时，框架会把这份档案注入系统提示，Agent 从第一轮对话起就知道自己是谁，而不是以一个无名通用助手的姿态出场。

## 问题

没有身份档案时，实践中常见的三类麻烦：

1. **身份漂移**。长会话里 Agent 会即兴给自己起名字，换个 session 就忘了；跨渠道（Telegram / Discord / WhatsApp）表现各不相同，用户感知很割裂。
2. **多实例混淆**。跑多个 Agent 时，日志、通知、群里的称呼全靠上下文猜，排障时很痛苦。
3. **无法积累**。人格写在一次性提示词里，没法版本化、没法回滚，更谈不上“演化”。

## 做法

1. 在 workspace 根目录（默认 `~/.openclaw/workspace`）建 `IDENTITY.md`，字段保持最小：Name、Creature、Vibe、Emoji、Avatar。
2. 划清边界：`SOUL.md` 管"怎么做事"，`IDENTITY.md` 只管"是谁"，行为规则不要塞进来。
3. 初始化推荐让 Agent 自己提案：删掉模板让它生成，你只做审核。自己写的身份，后文一致性明显更好。
4. 把 workspace 纳入 git。身份演化 = 提交历史，每次改动可 diff、可回滚。
5. 想让它随时间演进，约定一个低频节奏（比如每月 heartbeat 自检一次）：Agent 提出修改 → 你 review → commit。

## 踩坑点

- **文件只在会话启动时读取**。改完不重开 session 不生效，别误以为没写对。
- **字段写太长会稀释权重**。Vibe 三五个关键词足够，长篇性格描述应该放 SOUL.md。
- **别写私密信息**。这份文件会随 prompt 全量注入，且永久留在 git 历史里。
- **自我修改要留闸门**。没有 review 的自我进化，几轮之后往往退化成自我感动。
- emoji/头像的引用路径注意跨平台，Windows 的路径分隔符坑过我不止一次。

## 可复用建议

把 `IDENTITY.md` 当成 config-as-code 来治理：文件小、变更频率低、每条 commit message 说清“为什么改”。这套“档案文件 + git + 人工审核”的模式，同样适用于 `USER.md` 和 `MEMORY.md`——本质上都是把 Agent 的软状态从对话里搬到可审计的文件里。

## 总结

`IDENTITY.md` 解决的不是能力问题，而是**一致性和积累**问题。成本是十几行 Markdown，换来的是跨会话、跨渠道的稳定人格，以及一条可回溯的演化轨迹。先让它存在，再让它进化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/931b09b7445a557d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/356e3d53f23763fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/96b98e6b03c706b2.png)

