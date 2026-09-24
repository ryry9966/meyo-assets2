---
title: IDENTITY.md 实践：让 OpenClaw 的身份随使用慢慢进化
feedId: 38757
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 workspace 里躺着几个不起眼的 Markdown 文件：`AGENTS.md`、`SOUL.md`、`USER.md`，还有今天的主角 `IDENTITY.md`。大多数人装完就直接跳过，结果agent 用了三个月，名字还是默认占位符，跨渠道表现时人格统一、时而又像换了个模型。

IDENTITY.md 的定位很纯粹：声明这个 agent「是谁」。它会在会话启动时被注入 system prompt，是所有渠道里最稳定的那一层。

## 问题

我遇到的实际痛点有三个：

1. **默认身份太泛**。没有具体名字和性格锚点，长对话里模型会自己「脑补」人设，而且每次脑补得还不一样。
2. **多渠道不一致**。Telegram、CLI、webchat 各自开 session，没有统一身份文件时，同一个助手在不同入口像三个产品。
3. **身份和实际使用脱节**。写死一版就再没动过，三个月后它描述的性格和真实的交互习惯已经对不上了。

## 做法

**第一步：填基础字段。** 文件在 `~/.openclaw/workspace/IDENTITY.md`，核心就几行：

```markdown
- Name: 一个好叫、两个字以内的名字
- Creature: 它是什么（拟人/动物/纯抽象都行）
- Vibe: 两三句气质描述，写「约束」而不是「赞美」
- Emoji: 一个签名 emoji
- Avatar: 本地路径或稳定 URL
```

**第二步：和 SOUL.md 划清边界。** 我的分工是：IDENTITY.md 管「是谁」，SOUL.md 管「怎么行事」。两者重叠会让调 prompt 变成玄学。

**第三步：让它可进化。** 这才是关键。我加了一个 weekly 流程：

- 翻最近一周的会话日志，找出「语气不符合预期」的片段；
- 把偏差归纳成一两条可执行的修正，写回 Vibe 字段；
- workspace 全程用 git 管理，每次改动一个 commit，能 diff、能回滚。

小步迭代，一次只改一两个词，改完观察几天再动下一处。

## 踩坑点

- **改完不生效别慌**：IDENTITY.md 在会话启动时加载，旧 session 不会热更新，重开会话即可。
- **Vibe 写太长是负资产**：超过十行的性格描述会挤占有效上下文，而且互相矛盾。三句以内，越短越稳。
- **Avatar 用远程热链要谨慎**：部分渠道有缓存和防盗链问题，优先放本地文件。
- **让 agent 自己改自己要加闸门**：可以让它起草修订，但必须人工 diff 确认后合并，否则身份会漂移到不可控。

## 可复用建议

- 把 workspace 纳入 git，身份变更有据可查；
- 「身份回顾」做成周期性任务（配合 cron 或 heartbeat），频率一周一次足够；
- 字段保持少而稳定，宁可模糊也不要堆形容词；
- 团队共用的话，把 IDENTITY.md 当代码 review，改身份 = 改接口。

## 总结

IDENTITY.md 的价值不在于「设置一次」，而在于提供一个**可版本化、可评审、可回滚**的身份演进机制。它很小，几十行 Markdown；但它决定了你每天在跟一个什么样的东西对话。花半小时填好它，再花三个月慢慢修正它——这笔投入的回报，会在每一次跨渠道对话的一致性里体现出来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/3472d1b06328a7e8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/7b1cd3400ff9bbf0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/39a03d9acb251896.png)

