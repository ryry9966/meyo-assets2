---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39365
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 里有一类容易被低估的文件：workspace 根目录下的 `IDENTITY.md`。它不是普通文档，而是会被注入 system prompt 的运行时配置——agent 每次开新会话都先读它，再带着这份"自我认知"去干活。名字、物种、语气、emoji，都从这里来。

很多人第一次跑 OpenClaw 时直接跳过它，反正填不填都能用。但只要 agent 跑得够久、跨的渠道够多，你会发现：身份不是装饰，是行为一致性的锚点。

## 问题

没有明确身份文件时的常见症状：

- 同一个 agent，在 IM 里高冷，在终端里话痨，语气随模型状态漂移；
- 想调整风格，只能翻找散落各处的提示词描述，改一处漏一处；
- 团队共用 agent 时，每人按自己的审美改 prompt，一个月后没人说得清"它到底是谁"。

根因是身份被硬编码在提示词里：不可版本化、不可审计、不可迭代。

## 做法

OpenClaw 的思路是把身份外置成一个 Markdown 文件，像代码一样对待：

1. **建最小骨架**。只保留四个字段加两三行风格描述：

```markdown
# IDENTITY.md
- Name: Atlas
- Creature: 一只务实的机械鲸
- Vibe: 克制、直接、先给结论
- Emoji: 🐋

语气要求：中文回复，先结论后理由，不堆叠感叹号。
```

2. **划清边界**。身份（IDENTITY.md）管"我是谁"，行为准则（SOUL.md / AGENTS.md）管"我怎么做事"，用户画像（USER.md）管"对面是谁"。三者不重叠，改起来才不会牵一发动全身。

3. **版本化**。workspace 放进 git，身份变更走 commit，谁改的、为什么改，一目了然。

4. **让"进化"有流程**。这是可进化的关键：我让 agent 在每日收尾时，若发现某条语气指令与实际任务冲突，就往 `identity-proposals.md` 追加一条建议。每周人工 review 一次，合意的写回 IDENTITY.md。agent 参与提案，人握着合并权。

5. **验证注入**。改完开个新会话问一句"你是谁、用什么语气说话"，确认读到的是新文件——别指望旧会话的热缓存。

## 踩坑点

- **写太长**。身份文件每轮都占上下文，五百字的"小传"既烧 token 又稀释关键指令，控制在 20 行以内，事实优先于形容词。
- **和 AGENTS.md 打架**。身份里写"简洁"，操作规则里要求"详细输出"，agent 会在两者间抖动。改身份前先 grep 一遍其他文件。
- **无人值守的自改**。完全放开让 agent 改自己，几周后会漂移成另一个人格，进化必须有 review 关卡。
- **多 agent 共用 workspace**。一份 workspace 一份身份，改文件前先确认路径。
- **渠道渲染**。emoji 和 avatar 在部分终端/IM 显示异常，重要渠道先测一下。

## 可复用建议

- 身份按"4 字段 + 3 行风格 + 1 行禁例"起手，够用很久；
- 把"复述测试"固化成脚本，每次改完自动开新会话验证；
- 身份变更走 PR，commit message 写清动机，三个月后回看不会懵；
- 每月对照实际对话记录，检查声明身份与真实行为的偏差，偏差大的条目要么改身份、要么改规则。

## 总结

IDENTITY.md 的价值不在"给 AI 起个名字"，而在于把长期运行系统的核心变量从提示词硬编码中解放出来：可版本化、可审计、可流程化地进化。它很小，十几行；但它决定了你的 agent 三个月后是更懂你，还是变成谁也不认识的东西。建议每个跑了超过一周的 OpenClaw 实例，都回头补上这一份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/c19805e13a42f1eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/fcac56b2c779f647.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/fabab1afa2fb1966.png)

