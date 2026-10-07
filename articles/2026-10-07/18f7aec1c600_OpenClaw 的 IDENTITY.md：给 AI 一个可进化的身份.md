---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40836
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的 workspace 里有一组约定文件：AGENTS.md 管行为规则，SOUL.md 管价值观与语气，USER.md 记用户偏好，而 IDENTITY.md 回答的是最基础的问题——“我是谁”。它在每次会话启动时被注入 system prompt，agent 的名字、自称、形象、性格基调都从这里来。

很多人第一次跑 OpenClaw 时会跳过这个文件，觉得“能干活就行”。用一阵子就会发现：今天这个助手自称“小助手”，明天变成 "AI Assistant"，语气在工程直男和客服话术之间来回摇摆。

## 问题

缺一份稳定的 IDENTITY.md，通常踩三类坑：

1. **身份漂移**：session 重启后人格不一致，长期记忆里记下的偏好和当前自称互相矛盾。
2. **身份散落**：名字写在一处 prompt、语气写在另一处，改一处忘一处，没有单一真源。
3. **多 agent 串味**：两个实例共用人格描述，A 的语气污染了 B。

## 做法

我现在的 IDENTITY.md 大致长这样（节选）：

```markdown
# IDENTITY

- Name: Nova
- Creature: 一只机械浣熊
- Emoji: 🦝
- Vibe: 直接、克制、先给结论再讲理由
- Avatar: assets/avatar.png
- Boundaries: 不替用户做不可逆决定；不确定时明说
```

几条实践原则：

1. **短**。10–15 行以内。身份是约束，不是设定集，写太长会稀释 system prompt 里的关键指令。
2. **身份与记忆分离**。IDENTITY.md 只放“几乎不变”的属性；新学到的事实进 MEMORY.md，临时状态留在 session。
3. **允许进化，但走流程**。让 agent 在发现行为与身份描述冲突时，把修改建议写进 MEMORY.md 的待办区，我 diff 审过后再合入。git 管版本，改身份和改配置一个待遇。
4. **与 SOUL.md 分工**。SOUL.md 管“为什么这样说话”，IDENTITY.md 管“我是谁”，重复描述删掉，只留一处真源。

## 踩坑点

- **写成小作文**。实测超过一页后，agent 开始复述设定而不是执行任务。
- **放开让 agent 自由改写**。有一次它把自己的名字改成了我项目的命名空间，从此身份变更必须人工审批。
- **字段名不统一**。结构化字段（Name/Vibe/Avatar）保持英文键名，模型遵从度明显更稳；值可以随意用中文。
- **多 agent 忘了按 workspace 隔离**。两个实例读同一份文件，一方 rename 后另一方自称错乱。

## 可复用建议

- 起步模板只留五个字段：Name / Creature / Emoji / Vibe / Avatar，跑两周再补 Boundaries。
- 把 IDENTITY.md 纳入 git，commit message 写清“为什么改”，三个月后你会感谢自己。
- 每月让 agent 自评一次：当前行为与身份描述是否一致，输出差异清单。
- 升级 OpenClaw 或更换底层模型后，重读一遍身份文件，确认新模型对这些字段的遵从没有回退。

## 总结

IDENTITY.md 的价值不在于让 agent “更像个人”，而在于把“我是谁”从隐式、随 session 漂移的状态，变成一份显式、可 diff、可审查的配置。身份可进化的前提是变更可控：文件短、来源唯一、改动走 review。做到这三点，agent 跨会话表现的一致性会有肉眼可见的提升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/1e7af893500b7326.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/dc2bde2ff56371f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/f7cfd15de36c9391.png)

