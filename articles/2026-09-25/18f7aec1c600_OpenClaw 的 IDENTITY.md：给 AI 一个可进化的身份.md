---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38931
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 的 agent 不是靠一段写死的 system prompt 运行的。它的 workspace 目录下有一组约定文件：SOUL.md 管性格与价值观，USER.md 存用户事实，MEMORY.md 存长期记忆，而 IDENTITY.md 管的是"它是谁"——名字、形象、语气、emoji 这些表层但高频暴露的东西。每次会话注入上下文，这些文件都会被读进去。也就是说，身份不是一个启动参数，而是一个持续存在、可持续编辑的文件。

## 问题

裸跑一段时间后通常会遇到三件事：

1. **默认身份没有辨识度。** 同时跑两三个 agent（一个写码、一个盯通知），对话和通知里根本分不清谁是谁。
2. **人设改不动。** 身份散落在配置和环境变量里，改一次要重启要动代码，谈不上版本化和 review。
3. **身份漂移。** 长期运行的 agent 会在记忆里积累"我应该怎样"的碎片，和最初设定渐行渐远，越跑越陌生。

## 做法

核心思路：把身份当成一份小而具体的配置文件来运维。

**第一步，初始化文件。** 在 workspace 根目录（通常是 `~/.openclaw/workspace`）建 IDENTITY.md，字段化写清楚：

```markdown
# IDENTITY
- Name: Maru
- Creature: 猫形助手
- Vibe: 冷静、简短、先给结论
- Emoji: 🐱
- 职责边界: 只管代码 review 和部署通知
- 禁止: 主动寒暄超过一句
- 语气示例: "3 个文件有风险，先看 auth.ts。"
```

**第二步，职责分离。** 用户的名字、偏好放 USER.md；价值观和底线放 SOUL.md；IDENTITY.md 只放表层身份，三者别互相污染。

**第三步，设计演进机制。** 不要让 agent 直接改 IDENTITY.md。让它把身份方面拿不准的事记进 MEMORY.md，再用一条 HEARTBEAT.md 定时任务，每月把这些条目整理成摘要提醒你 review，确认后由你合入。用 git 管 workspace，每次 diff 可见、可回滚。

**第四步，多 agent 多 workspace。** 每个 agent 一份独立 IDENTITY.md，名字和 emoji 拉开差距，群聊转发场景一眼可辨。

## 踩坑记录

- **文件写太长。** IDENTITY.md 每次会话都占上下文，超过一屏就是在稀释重点，建议控制在 30 行内。
- **写成口号。** "你是世界上最棒的助手"不如一条具体的语气示例。模型对示例的模仿远好于对形容词的服从。
- **让 agent 自由改写身份文件。** 实测会漂移：越改越热情、越写越油腻，本质是讨好倾向在自我强化，必须人工把关。
- **名字和 emoji 频繁更换。** 历史会话、日志、通知里全是旧引用，换一次乱一次，定下来就少动。

## 可复用建议

把 IDENTITY.md 当 config as code 对待：小、具体、版本化、改动走 review。演进靠"记忆收集 + 人工合入"的慢循环，而不是 agent 自我改写的快循环。"字段 + 示例"的格式可以直接复制到 SOUL.md 和 USER.md，三个文件用同一套写法，维护成本最低。如果你的场景需要频繁换角色，宁可多开一个 agent，也不要让一份身份文件承担多个人设。

## 总结

IDENTITY.md 是个小文件，但它决定了每次交互的第一印象，也是对抗身份漂移的唯一锚点。把它从"一次性配置"升级为"持续运维的对象"，配好 git 和月度 review，agent 才会在长期使用里越来越像你想要的那个角色，而不是越用越陌生。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/a39a566e29ddbf51.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/59d3e7dda8e00c38.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/b5b0d5edff5d041b.png)

