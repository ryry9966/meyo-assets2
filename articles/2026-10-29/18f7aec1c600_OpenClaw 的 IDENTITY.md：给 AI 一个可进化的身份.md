---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42925
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

OpenClaw 的 workspace 里有一组约定文件：AGENTS.md 管行为规则，SOUL.md 管性格与原则，USER.md 记用户上下文，MEMORY.md 存长期记忆。IDENTITY.md 是其中最小的一个，它只回答一个问题："你是谁"。通常就是几行：name、creature、vibe、emoji、avatar。

这几行会在每次会话启动时注入上下文，是 agent 的默认人格底座。而且它是纯 markdown——可以 git 管、可以 diff、可以让 agent 自己提修订。

## 问题

在没有 IDENTITY.md 的那段时间，我遇到三类麻烦：

1. **人格漂移**。同一只 agent，今天自称"助手"，明天换个说法，语气随模型版本波动。
2. **身份散落**。名字写在一段 prompt 注释里，emoji 写在另一处，改一次要全局搜索。
3. **多 agent 混淆**。我跑三个 workspace（工作、家庭、实验），消息转发时经常分不清"这条是谁回的"。

共同根源：身份不是一等公民，只是散落的字符串。

## 做法

**1. 写一份最小 IDENTITY.md**

```markdown
# IDENTITY
- Name: Nova
- Creature: 夜枭形态的自动化管家
- Vibe: 冷静、简短、先给结论
- Emoji: 🦉
- Avatar: avatar.png
```

五行就够。它的职责是陈述事实，不是下发规范。

**2. 分清边界**

- IDENTITY.md：它**是**什么（名字、形态、气质）
- SOUL.md：它**怎么做事**（原则、底线、语气倾向）
- AGENTS.md：它**怎么干活**（流程、工具、约束）

判断标准：一句话删掉后，agent 的"行为"不变、只是"自称"变了，那它属于 IDENTITY.md。

**3. 让身份可进化**

把 workspace 纳入 git，每季度（或大版本升级后）做一次 identity review：让 agent 基于 MEMORY.md 里的近期交互，提出 2-3 条身份修订建议（比如 vibe 从"活泼"调成"更简洁"）；修改以 diff 形式提交，人工 review 后 merge，commit message 写清"为什么改"。这样演化有迹可循，回滚就是一次 `git revert`。

## 踩坑点

- **别把规则塞进来**。我一开始写了十几条 always/never，结果和 AGENTS.md 打架，改一处忘一处。规则归 AGENTS.md。
- **别写长文**。身份文件超过半屏，模型遵循度反而下降，还稀释了关键信息。经验值：10 行以内。
- **警惕 agent 自改身份**。agent 默认能编辑 workspace 文件，如果某次对话它"顺手"改了自己的名字，下个会话就生效了。建议约定它只能提 diff 不能直接写，或定期盯 `git status`。
- **avatar 与 emoji 要对齐**。不同渠道（Telegram、桌面通知等）展示的名字和头像来自不同配置，改身份时一起改，否则会出现"名字换了头像没换"的割裂感。

## 可复用建议

- 把 IDENTITY.md 当代码：小、纯事实、git 管、变更走 review。
- 用 HEARTBEAT.md 或定时任务挂一个季度回顾，让演化成为机制而非心血来潮。
- 多 agent 场景给每个 workspace 独立身份；emoji 是成本最低的区分手段，日志里一眼可辨。
- 身份变更后，拿几条典型历史 prompt 重放一轮，确认人格表达符合预期再收工。

## 总结

IDENTITY.md 的价值不在"给 AI 起个名字"，而在把身份从散落的 prompt 片段，变成一个版本化、可审计、可进化的资产。五行字 + git + 一次季度回顾，成本极低，换来的是多会话、多渠道、多 agent 之下稳定一致的"它是谁"。身份先稳住，人格和行为才谈得上持续迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/bde40ade5f5ba615.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/5974753ba0cd496d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/5d30f6f0ed3d988e.png)

