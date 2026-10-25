---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42586
source: 综合讨论
publishedAt: 2026-10-26
---

# 背景

OpenClaw 是常驻型个人 Agent，它的行为不是写死在代码里的，而是由 workspace 下的几个 Markdown 文件在会话启动时注入上下文拼出来的。`IDENTITY.md` 是其中定义"我是谁"的那一份：名字、形象设定、语气基调、emoji、头像。它和 `SOUL.md`（行为准则）、`USER.md`（用户画像）构成三件套，每个新会话的第一层上下文都由它们决定。

# 问题

没有身份文件时，常见三个症状：

1. **人格漂移**：跨会话后 Agent 自我称呼、语气随机变化，今天自称助手，明天换人设；
2. **多渠道不一致**：Telegram、CLI、Discord 里表现像三个不同的东西；
3. **风格调整不可追溯**：只能改系统提示词或全局配置，改完无 diff、无记录，覆盖即丢失。

本质是身份没有持久化、没有版本化。

# 做法

**第一步：建立文件。** 在 workspace 根目录创建 `IDENTITY.md`，最小字段就够：

```markdown
# IDENTITY.md
- **Name:** 小爪
- **Creature:** 一只务实的机械章鱼
- **Vibe:** 工程化、简洁、不废话
- **Emoji:** 🐙
- **Avatar:** assets/claw.png
```

**第二步：控制篇幅。** 这个文件每次会话都会注入上下文，建议压在 10 行以内。只写"谁"，行为规则归 `SOUL.md`，两边不要重复。

**第三步：用 git 管 workspace。** 这是"可进化"的关键——每次调整语气或人设走一次 commit，能 diff、能回滚。身份演变因此变成一条可审计的时间线，而不是反复覆盖。

**第四步：验证生效。** 改动通常在新会话才生效；常驻主会话可以明确要求它重读 workspace 文件。然后问一句"你是谁、你的风格是什么"，对照检查。

# 踩坑点

- **写成小作文**：几百字的人设描述 = 每个会话固定 token 开销，且和 SOUL.md 重叠时 Agent 行为摇摆。
- **混装动态内容**：把当前任务、临时状态写进身份文件。身份是低频变化的，动态的交给 memory。
- **多 Agent 共用一个 workspace**：两个 Agent 自我介绍一模一样。每个 Agent 要有独立 workspace 和独立的 IDENTITY.md。
- **全局身份与渠道覆盖打架**：如果在 AGENTS.md 里对某渠道重写了语气，注意别两边各写一半，留一处权威来源。
- **Avatar 路径失效**：emoji 声明了但头像文件不存在，部分渠道会渲染报错或静默忽略，上线前过一遍。

# 可复用建议

- **三件套职责单一**：IDENTITY 管"是谁"，SOUL 管"怎么做事"，USER 管"对面是谁"，交叉越多越乱。
- **像配置文件对待**：小步修改，commit message 写清动机（"为什么改语气"比"改了语气"有用得多）。
- **团队部署用模板 + per-agent 覆盖**：先发一份基线 IDENTITY.md，各 Agent 只覆盖差异字段，避免 fork 出十份各自漂移。
- **定期 review**：身份文件和代码一样会腐化，季度级别回顾一次，删掉不再成立的设定。

# 总结

IDENTITY.md 的价值不在于"给 Agent 起个名字"，而在于把身份从隐式的提示词习惯，变成显式、可版本化、可审计的配置。文件很小，但它决定了所有会话的第一层上下文。管好这十几行，Agent 的长期一致性就有了锚点；配上 git，它的"进化"才真正可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/a14c0f465e7a67df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/4dea0c6ae628ddb5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/4f15b58d66d25b76.png)

