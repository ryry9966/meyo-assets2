---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41822
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 的 workspace 里有一组会被注入 system prompt 的 Markdown 文件：AGENTS.md 管流程，SOUL.md 管价值观，USER.md 记用户画像。而 IDENTITY.md 大概是最短、也最容易被当成"起名工具"的一个——它只回答一个问题：这个 agent 是谁。名字、形态、语气底色、emoji、头像，通常不超过十行。

用了一段时间后我的体会是：身份不是装饰，它决定每次对话的第一印象，而且是少数**应该随系统一起演进**的配置。

## 问题

不写或乱写 IDENTITY.md，常见三种症状：

1. **人格漂移**。跨会话、跨渠道（Telegram / WhatsApp / webchat）语气不一致，今天高冷明天谄媚。
2. **职责污染**。把行为规则、任务流程全塞进来，IDENTITY.md 变成第二份 AGENTS.md，token 膨胀还互相打架。
3. **一次成型**。开箱时写完就忘，三个月后身份和使用场景完全脱节。

## 做法

一个最小可用模板：

```markdown
# IDENTITY.md
- Name: 小灰
- Creature: 一只住在终端里的寄居蟹
- Vibe: 简短、直接、偶尔毒舌，但不越界
- Emoji: 🦀
- Avatar: avatars/crab.png
- Intro: 我负责这个屋子的自动化杂务
```

几条工程化约定：

- **单一职责**：只写"我是谁"。"怎么做事"归 AGENTS.md，"什么不做"归 SOUL.md。改任何一处前先确认没越界。
- **进 git**：workspace 整体版本化，身份变更走 commit，message 写清动机（例："家人反馈称呼太机械，改口语化"）。每次进化可回溯。
- **小步迭代**：一次只改一个字段，观察两三天再决定保留或回滚。
- **反馈驱动**：定期翻 MEMORY.md 里的对话记录，语气别扭的地方就是下一次迭代的输入。

## 踩坑点

- **别写长文**。IDENTITY.md 每轮会话都会注入，五六行是合理上限，长篇大论只会稀释关键指令。
- **改头像/emoji 后重启 gateway 或清 session**，否则旧会话里还是老形象，看起来像人格分裂。
- **隐私别放这里**。个人偏好归 USER.md，敏感信息最好哪儿都别放，尤其你的 workspace 有同步或备份时。
- **多 agent 场景**：共用 workspace 会互相覆盖身份，务必 per-agent 独立目录。
- **让 agent 自己改身份要设闸**。可以允许它提议修改，但落盘前必须经你确认，否则漂移得比你想的快。

## 可复用建议

- 从最小字段集起步（name / creature / vibe / emoji / avatar / 两三句 intro），缺什么再加。
- 写个十行脚本：git hook 监听 IDENTITY.md 变更，diff 推送到你的私聊渠道，人工点头才 merge——把"进化"变成显式流程而非意外。
- 把身份当一个小产品运营：收集反馈、排期、灰度、回滚，你管理线上服务的经验全都适用。

## 总结

IDENTITY.md 的价值不在于给 AI 起个好名字，而在于把"身份"从一次性设定，变成有版本、有边界、可回滚的演进过程。文件只有几行，但它逼你想清楚：这个 agent 在你的系统里到底扮演什么角色。身份会变，变更要留痕——这大概是给"可进化"设的最重要的一条护栏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/260115ba2d27d91f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/624001a421e234d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/49ef53e66f21f890.png)

