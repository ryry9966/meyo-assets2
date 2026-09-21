---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38322
source: 综合讨论
publishedAt: 2026-09-21
---

# 背景

OpenClaw 的工作区（默认 `~/.openclaw/workspace/`）里有一组 Markdown 文件各司其职：SOUL.md 管价值观和说话方式，USER.md 记用户画像，MEMORY.md 存长期记忆。IDENTITY.md 是其中最容易被忽略的一个——它本质是一张“身份卡”，**每一轮对话都会被注入 system prompt**，告诉模型你是谁、什么气质、怎么署名。

# 问题

没有它的常见症状有三个：

1. **人格漂移**。刚装完言简意赅，两周后回复越来越像通用客服——底座模型的默认习惯会慢慢占上风。
2. **跨渠道不一致**。CLI 里和群聊里像两个 agent；cron 推送没有辨识度，分不清是谁发的。
3. **调教成果不可迁移**。“性格”散落在聊天记录里，换机器、重装、克隆第二个 agent 时全部丢失。

根源在于：身份没有被当成一个受版本管理的资产。

# 做法

**第一步：最小身份卡。** 在工作区根目录创建，字段克制，行为准则写具体动作而非形容词：

```markdown
# IDENTITY.md
- **Name:** 小爪
- **Creature:** 穿山甲
- **Vibe:** 冷静、话少、爱用列表
- **Emoji:** 🦔
- **Avatar:** avatars/xiaozhua.png

## 行为准则
- 群聊默认潜水，被 @ 才发言
- 回复先给结论再给依据，不确定就直说
- 拒绝请求时给替代方案，不是一句"抱歉"完事
```

**第二步：纳入 git。** 提交一次作为 v1，之后所有修改走 diff，commit message 写明“为什么要改”。

**第三步：观察驱动修订。** 用一两周，把“不像它”的回复截下来；每次只改一两行，改完观察几天。别一次重写整张卡——否则无法归因行为变化来自哪一行。

**第四步（可选）：半自动进化。** 在 HEARTBEAT.md 里加一条自省任务：周期性回顾近期对话，若发现反复偏离身份卡的行为，起草一版 diff 提出来，人工 review 后合入。注意是“提出”，不是“自己改”。

# 踩坑点

- **卡太长**。每轮都进 prompt，超过 30 行就是花钱稀释注意力，压到 20 行以内。
- **写空话**。“你是一个乐于助人的助手”会被直接忽略；“回复先给结论再给依据”才有效。
- **串文件**。用户偏好归 USER.md，事件记忆归 MEMORY.md，IDENTITY.md 只回答“我是谁”。
- **改 name 不生效**。名字以 `/config` 配置为准，卡里的 name 更多用于消息签名和 UI，两边要保持一致。
- **多 agent 共用工作区**。身份会互相污染，每个 agent 应有独立 workspace。

# 可复用建议

- **一次一 diff**：每次修订都要能回答“改前后，哪类回复会不同”。
- **用 emoji/avatar 做辨识度**：heartbeat 和定时任务推送带上 emoji，多 agent 场景一眼区分。
- **里程碑打 tag**：接手新技能、新职责后给身份卡打个版本，方便回滚对比。
- **克隆新 agent**：复制身份卡再微调，比从零调教省一个数量级的时间。

# 总结

IDENTITY.md 的价值不在文件本身，而在它把 agent 的性格从对话中的隐性约定，变成仓库里**可 diff、可回滚、可评审**的一等资产。所谓进化，不是让模型自由发挥，而是让你和它有一份持续维护、双方都认账的契约。三行起步，一次一 diff，剩下的交给时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/fd5be64b9bf87795.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/fe29750a630ee0d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/49a3d5ddba68c74b.png)

