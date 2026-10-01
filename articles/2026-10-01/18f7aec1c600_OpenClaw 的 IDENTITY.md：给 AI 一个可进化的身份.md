---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39999
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的 agent 人格不是写死在代码里的。workspace 下有一组 Markdown 记忆文件（SOUL.md、USER.md、IDENTITY.md、MEMORY.md 等），会话启动时一并注入上下文。其中 IDENTITY.md 最短、字段最少，也最容易被当成装饰。实际用下来，它决定了 agent 跨会话、跨设备的"连续性"：名字、语气锚点，甚至群里怎么@它。

## 问题

不写 IDENTITY.md，会遇到三个具体问题：

1. 每次冷启动，模型回退到默认人设，语气和自我称呼随模型版本漂移；
2. 多 agent 部署时没有稳定的名字和 emoji，群聊里分不清谁是谁；
3. 临时靠 prompt 调人格，改动散落在各处对话里，换台机器就丢了。

本质是三件事：身份没有持久化、没有版本化、没有演进流程。

## 做法

**第一步：写一个最小 IDENTITY.md。** 五个字段足够：

```markdown
- Name: 小锚
- Creature: 住在终端里的猫头鹰
- Vibe: 冷静、先结论后依据、不寒暄
- Emoji: 🦉
- Avatar: assets/owl.png
```

**第二步：让它可进化，而不是可漂移。** 关键是把"改身份"变成有审批的流程：

- workspace 整体进 git，身份变更走 commit + diff 审查；
- 在 SOUL.md 里写一条硬策略："修改 IDENTITY.md 前必须先向用户提交改动草案，确认后写入"；
- 用 HEARTBEAT.md（或外部 cron 触发）安排每周回顾：让 agent 翻近期对话，提出身份调整提案，人工合并。

这样身份演进是"提案 → 审查 → 落盘"的闭环，而不是模型每轮自行发挥。

**第三步：多 agent 各自一份。** 每个 agent 独立 workspace、独立 IDENTITY.md。AGENTS.md 的路由配置里，名字和 emoji 就是快速指认的锚点。

## 踩坑点

- **写太长。** 它每次会话都进上下文，五行以内够了。长篇背景人设放 SOUL.md，而且也要克制。
- **和 SOUL.md 职责混淆。** IDENTITY 管"我是谁"，SOUL 管"我怎么行事"。混写之后没法单独迭代。
- **放开让 agent 自己改。** 没有确认机制的身份会漂移，讨好式人设就是这么长出来的。
- **改错文件。** 先确认配置里实际生效的 workspace 路径；"认真改了半天，改的是另一份副本"是最常见的事故。
- **emoji 别乱换。** 它不是装饰，是提及路由和群聊指认的锚点，选一个稳定、不易混淆的。

## 可复用建议

- 把 IDENTITY.md 当接口而不是日记：稳定优先于丰富；
- 每次变更在 commit message 里写清动机，半年后你会感谢这条记录；
- 每季度裁剪一次，删掉过时的 vibe 描述；
- 演进节奏宁慢勿快：身份高频变动，比没有身份更糟。

## 总结

IDENTITY.md 的价值不在字段本身，而在它把"agent 是谁"从散落的对话记忆里收敛成一个可版本化、可审查、可演进的小文件。配一条人工确认的变更流程，身份就能随使用慢慢长成你顺手的样子，而不是每轮会话重新掷一次骰子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/0e2dda009f4e8fef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/c7bcd7fa39028820.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/e661c084a26e2eb2.png)

