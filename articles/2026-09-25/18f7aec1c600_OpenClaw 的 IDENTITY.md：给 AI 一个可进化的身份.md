---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38930
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 的 agent 不是每次会话都从零开始的：gateway 启动时会把 workspace 里几个 Markdown 文件注入 system prompt。大多数人只关注 `TOOLS.md` 和 `MEMORY.md`，而 `IDENTITY.md` 往往被忽略。它回答的不是"这个 agent 会做什么"，而是"这个 agent 是谁"——名字、形象、职责边界、语气基线。

## 问题

没有身份文件的 agent，长期运行下来会出现几个典型症状：

- **语气漂移**：同一个 agent，周一像技术文档，周五像客服话术，全看当次会话模型怎么发挥；
- **多 agent 分不清**：两个 bot 挂在同一个 gateway 下，群里回复没法一眼分辨谁是谁；
- **描述散落**："它是谁"这段话被复制粘贴在各个 skill prompt 里，改一次要翻五个文件；
- **自我介绍胡编**：你问它叫什么，它现场给你编一个。

## 做法

1. 在 workspace 根目录建 `IDENTITY.md`，保持精简：

```markdown
- Name: Claw-值班员
- Creature: 住在终端里的机械蟹，话不多但靠谱
- Emoji: 🦀
- Avatar: avatars/duty.png

## Self-description
负责告警值班与内部检索。回复直接、先给结论、附来源。
不做没有依据的长期承诺。
```

2. 放一张 Avatar 图片到对应路径，群里通知和回复会带上头像，识别成本立刻下降。
3. 重启（或重载）gateway，问 agent "你是谁、负责什么"，确认注入生效。
4. 把文件纳入 git，身份变更走 commit，像改配置一样对待它。

## 踩坑点

- **写太长**：它每个会话都注入。超过几百字就是在烧 token，还会稀释真正关键的指令。十行以内是合理上限。
- **和 SOUL.md 边界不清**：身份（名字/形象/职责）放 IDENTITY，性格（怎么说话、底线在哪）放 SOUL。两处重复定义，agent 会在两种人设之间摇摆。
- **当成记忆用**：临时事实进 memory，别写进身份。身份是"合同"，不是"日记"。
- **改完不重载**：部分部署下修改不会热生效，改完务必验证。
- **多 agent 忘改默认名**：两个 bot 都顶着默认名字，排查日志时非常痛苦。

## 可复用建议

- **职责边界写"不做什么"比"做什么"更有效**。"不做没有依据的长期承诺"一句，胜过三段职责描述。
- **用模板初始化新 agent**，团队内统一字段，方便横向对比和审查。
- **定期回顾**：agent 的定位会变——从个人玩具变成团队值班助手，身份文件要跟上。这就是"可进化"的含义：身份跟着使用事实迭代，而不是一次定型。git 历史本身就是它的成长记录。

## 总结

`IDENTITY.md` 是 OpenClaw 里成本最低、杠杆最高的一类文件：十行内容，换来跨会话的行为一致性和多 agent 场景下的清晰边界。如果你已经在维护 SOUL 和 MEMORY，花五分钟补上身份这一层，收益立竿见影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/9ae54b81ab5d35de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/424ff53b99b6e019.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ae150ec619496258.png)

