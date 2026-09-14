---
title: IDENTITY.md：给 AI 一个可进化的身份
feedId: 37505
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 的一个核心设计是：工作区（workspace）即人格载体。一组 Markdown 文件在会话启动时被注入 system prompt，其中 `IDENTITY.md` 是最短的一个——通常只有十几行，但它决定了 agent 在每次对话中的自我认知：叫什么、是什么形象、什么语气基调。名字起得随意，影响却贯穿所有会话。

## 问题

没有这份文件或管理不当的时候，常见现象：

- 同一个 agent 在 Telegram 里自称 "assistant"，切到 CLI 又换了个说法，跨渠道人格不一致；
- 想调人设只能改 config 里的内嵌 prompt，改完要重启，还怕碰坏旁边的配置；
- 同时跑三四个 agent 后，自己都分不清谁是谁。

本质问题：身份散落在配置和临时 prompt 里，不可版本化、不可评审、不可回滚。

## 做法

1. **建最小文件。** 在 workspace 根目录写 `IDENTITY.md`，控制在 15–20 行内。它每次会话都进 system prompt，多一行就多占一份上下文：

```markdown
# IDENTITY.md
- Name: 小爪
- Creature: 一只务实的机械蟹，工程口吻
- Emoji: 🦀
- Vibe: 直接、克制，先给结论再给依据；不寒暄，不堆表情
- Avatar: 扁平插画风格，深蓝底色
```

2. **纳入 git。** 把身份当代码资产：每次改人设走 commit，出问题能 diff、能回滚。
3. **划清职责边界。** `IDENTITY.md` 管"我是谁"；行为规则文件管"我怎么做事"；`USER.md` 管"用户是谁"。三者混写是后期混乱的主要来源。
4. **多 agent 场景一 workspace 一身份。** 要第二个人设就 fork workspace，不要在同一个文件里塞两套设定。

## 踩坑点

- **写太长。** 见过 200 行的人设文件，结果关键指令被稀释，语气反而失控。
- **与 config 内嵌 prompt 冲突。** 两处都定义了名字，行为时对时错。确定唯一事实来源，另一处删掉。
- **频繁改名。** agent 的长期记忆会引用旧名字，越改越乱。改名应视为大版本事件。
- **误当安全边界。** 它影响的是风格，防不住 prompt injection，别在里面放任何敏感信息。
- **误当免责声明。** 写"永远礼貌"不等于输出合规，行为约束该放行为规则里。

## 可复用建议

- 把身份文件当**接口设计**：字段少而稳定，具体措辞可以随意迭代。
- 准备一组固定测试对话，每次改动后跑一遍，对比输出风格是否漂移。
- 利用 emoji / 形象字段做多个 agent 的快速视觉区分，比读名字省事。
- 每季度回顾一次：身份跟着实际使用演化，而不是一开始就设计"完美人设"。

## 总结

`IDENTITY.md` 的价值不在于"给 AI 起名字"的仪式感，而在于把原本散落在 prompt 里的隐性决策，变成一个可版本化、可评审、可回滚的小文件。十几行的成本，换来跨渠道一致性和长期演化的抓手。对折腾 agent 工作流的人来说，这大概是 OpenClaw 工作区里性价比最高的一个文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/eded1990cabfe2a4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/d6163e31095408ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/90638e20cef7e655.png)

