---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38243
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的 workspace 里有一组"以 Markdown 为上下文"的文件：`AGENTS.md` 管行为规范，`SOUL.md` 管性格底色，`USER.md` 管用户画像，而 `IDENTITY.md` 回答的是最基础的问题——"我是谁"。它默认只有五行左右：

```markdown
# IDENTITY.md

- **Name:** 阿钳
- **Creature:** 机械蟹（钳子多，适合干杂活）
- **Vibe:** 冷静、简短、先给结论再给理由
- **Emoji:** 🦀
- **Avatar:** assets/crab.png
```

有个容易忽略的设计：首次启动的 bootstrap 流程里，agent 会主动问你几个问题，然后**自己写下**这份文件。身份是 agent 的自述，不是用户硬编码的配置。

## 问题

实践里最常见的三类痛点：

1. **多实例分不清谁是谁。** 默认身份太通用，日志和会话记录里三个 agent 像一个人；
2. **身份散落各处。** system prompt 片段、插件注入、聊天里的口头约定，改一次要翻三个地方；
3. **身份不随职责进化。** 用了三个月，职责早从"写周报"扩展到"盯服务器"，但身份还停在第一天的自述。

## 做法

1. **定位文件**：`~/.openclaw/workspace/IDENTITY.md`；多实例则每个 workspace 各一份，物理隔离。
2. **字段保持最小**：五项够用就别加，`Vibe` 一句话写清表达风格。
3. **版本化**：整个 workspace 进 git，身份变更走 commit，回滚有据可查。
4. **建立迭代节奏**：每次能力边界变化（新增技能、换了职责），顺手改一行 `Creature` 或 `Vibe`，让身份跟着职责走。
5. **改完开新会话验证**：旧会话上下文里仍是旧身份，这不算 bug。

## 踩坑点

- **别把任务指令写进 IDENTITY.md**，那是 `AGENTS.md` 的事。身份文件混入规则后，上下文权重被稀释，行为反而漂移。
- **宁短勿长**：这个文件每次会话都会注入上下文，写几百字等于给每轮对话加固定税。
- **Avatar 路径写错不会报错**，只会静默降级，建议改完实际看一眼。
- **多人共用实例时**，身份会被反复重写。要么一人一 workspace，要么约定锁定编辑权。
- 与 `SOUL.md` 的分工要清楚：SOUL 是不变的性格约束，IDENTITY 是可进化的对外自述，别互相抄内容。

## 可复用建议

- **一句话测试**：直接问 agent "你是谁、擅长什么、不做什么"，答案能对上 IDENTITY.md 即合格；
- **按场景 fork workspace**：写作的、运维的、盯家里 NAS 的，各自身份独立进化；
- **每月 diff 一次**：删掉过时自述，身份文件的维护成本应该接近于零。

## 总结

IDENTITY.md 的价值不在"起个好名字"，而在于把身份变成一个可版本化、可 diff、可随职责演进的一等文件。身份清晰，行为边界才清晰；边界清晰，你才敢真正放权给它跑自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/c599b51232a6cb74.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/801ba05fb38bee6c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/afa40bd85ca790a3.png)

