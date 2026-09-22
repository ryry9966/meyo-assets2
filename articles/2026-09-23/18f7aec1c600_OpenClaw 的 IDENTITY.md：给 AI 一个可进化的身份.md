---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38542
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 workspace 里有几个每次会话都会被注入 system prompt 的元文件：AGENTS.md 管做事规则，SOUL.md 管人格基调，IDENTITY.md 管"我是谁"。名字、形象、emoji、头像、时区，都集中在这份 Markdown 里，放在 workspace 根目录，全局兜底路径是 `~/.openclaw/IDENTITY.md`。

很多人第一次跑起来就跳过了它——默认身份也能用。但当你开始长期运行一个 agent，或者一台机器上同时跑多个 agent，身份就从"装饰"变成了基础设施。

## 问题

没有显式身份文件时，常见三类麻烦：

1. **人格漂移**。跨周、跨模型的对话里，语气和自称慢慢变化，事后无法定位是哪次改动引起的。
2. **多 agent 混淆**。两个 agent 共享同一套 prompt，用户问"这事谁跟进的"，两边都认领。
3. **身份散落**。名字写死在配置里，口头禅混在 SOUL.md，时区跟着系统走——改一次身份要动三处。

## 做法

在 workspace 根目录建 IDENTITY.md，结构自由，但建议包含这些字段：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 机械蟹
- Emoji: 🦀
- Avatar: ./avatar.png
- Timezone: Asia/Shanghai
- Style: 先给结论再给依据；回复短；不闲聊
```

要点：

- Name/Emoji/Creature 是低成本的"锚点"，主要作用是让模型在长对话里保持一致的自我指称；真正塑造行为的是 Style 这类描述行。
- 身份文件随每次会话注入，改动在下一次会话生效，热改不生效不是 bug，必要时重启 gateway 或等一个心跳周期。
- 三类文件不越界："我是谁"进 IDENTITY.md，"我怎么做"进 SOUL.md，"活怎么干"进 AGENTS.md。

进化的关键：**把 workspace 纳入 git**。每次觉得 agent 说话不对味，先改 IDENTITY.md 并 commit，再用固定的三五个探针问题（自我介绍、拒绝越权请求、报告一个坏消息）对比前后输出。身份调整从此有 diff、有回滚，而不是玄学调 prompt。

## 踩坑点

- **写太长**。见过 200 行的"人生简历"，实际塑造力不如 20 行精准描述。IDENTITY.md 每个会话都吃 token，控制在 30 行以内。
- **规则混进身份**。把"回答必须附来源"写进身份文件，之后改规则要同时动两个文件，早晚漂移。这类内容归 AGENTS.md。
- **多 agent 复制同一份**。身份完全相同等于没有身份，至少让 Name 和 Style 有可感知的区分。
- **当记事本用**。在里面记任务状态、记日志，system prompt 迟早被撑爆。状态该去 MEMORY.md，那里才是按需检索的。

## 可复用建议

1. 新装 OpenClaw 第一件事就写 IDENTITY.md，哪怕只有五行，先占住"锚点"，再慢慢长。
2. 身份变更走 git commit，大的调整打 tag，出问题能回溯到具体是哪次改动。
3. 多 agent 环境给每个 agent 独立 workspace，身份文件互不复用。
4. 每月回顾一次：删掉没起作用的行，比往上加行更重要。

## 总结

IDENTITY.md 的价值不在"给 AI 起个名字"，而在于把身份变成一个受版本控制、可对比、可回滚的工程对象。人格不再是一次性的 prompt 手艺，而是随使用持续迭代的配置文件。五行也好，三十行也好——先写下来，再让它慢慢进化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/908d599cde8fb3c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/e2f2f8ab75a81652.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/6a6ffe16af53a71d.png)

