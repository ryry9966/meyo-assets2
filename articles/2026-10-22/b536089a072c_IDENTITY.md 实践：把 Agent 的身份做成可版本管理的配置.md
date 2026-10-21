---
title: IDENTITY.md 实践：把 Agent 的身份做成可版本管理的配置
feedId: 42230
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

OpenClaw 的会话机制决定了它没有"天生"的长期记忆：每次冷启动、每个新会话，模型拿到的都是重新组装的上下文。系统提示、AGENTS.md、SOUL.md 这些文件，才是它对"我是谁"的全部认知。IDENTITY.md 是其中最薄但最显眼的一层——它不负责能力，只负责这个 agent 是谁：名字、形象、语气。

## 问题

没有一份固定身份文件时，常见三个症状：

1. **人格漂移**。今天高冷明天话痨，跨会话风格不连贯；
2. **多 agent 串味**。你开了 work、home、ops 三个实例，共用模型和提示词，很快分不清谁在说话；
3. **身份靠聊天记录续命**。称呼、口头禅散落在历史消息里，每次靠模型"回忆"，召回不稳定就出戏。

## 做法与步骤

**第一步：最小可用身份。** 在 workspace 根目录建 IDENTITY.md，只写四件事：

```markdown
# IDENTITY
- Name: 阿灰
- Creature: 一只务实的工程猫
- Vibe: 简短、直接、偶尔冷幽默
- Emoji: 🐱
```

先够用，再变好。

**第二步：划清职责边界。** IDENTITY.md 管"是谁"（名字、形象、语气），SOUL.md 管"信什么"（价值观、边界、做事原则），USER.md 管"对面是谁"。三者重叠是身份混乱的根源。

**第三步：让它进化。** 进化不是玄学，就是把共识定期写回文件。两个时机：发现它某次表现好或不好，当场改一两行；或者借助 heartbeat / 定时任务，让它每周总结近期协作风格，diff 之后合并进 IDENTITY.md。

**第四步：纳入 git。** 身份变更应该像代码变更一样有记录，`git log IDENTITY.md` 能还原它每一阶段的性格。

## 踩坑点

- **别写长**。这份文件每个会话都会注入上下文，20 行以内足够，写满人格小作文纯属烧 token；
- **别和 SOUL.md 抢戏**。"做事要谨慎"是 SOUL 的事，"说话不绕弯"才是 IDENTITY 的事；
- **警惕过度拟人反噬工具调用**。人设太浓，汇报执行结果也带戏、输出啰嗦。工具型 agent 建议人设克制；
- **改完要开新会话才生效**。旧会话上下文里还是旧身份，别误以为没改上；
- **多 agent 各写各的**。共用一份 IDENTITY.md，串味比没身份更糟。

## 可复用建议

- 把 IDENTITY.md 当配置而不是文档：短、可 diff、单一事实来源；
- 首行注释里写版本或修改日期，排查"它现在到底是哪版人格"时有据可查；
- 团队场景下身份变更走 PR review，效果等同于给 agent 定一份 style guide；
- 想做 A/B，就开两个 workspace 两份身份，并行跑一两周再合并结论。

## 总结

IDENTITY.md 的价值不在"让 AI 更像人"，而在把飘在对话里的隐性约定，固化成一个可版本管理、可 review、可回滚的文件。身份一旦可演进，agent 才有真正的跨会话一致性——这在自动化实践里，是最被低估的那种稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/452252080127f563.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/872360cb8eb23f83.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/5779e558ab874409.png)

