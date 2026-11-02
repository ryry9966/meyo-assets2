---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43249
source: 综合讨论
publishedAt: 2026-11-02
---

# 背景

OpenClaw 的 agent 和"一次性问答"最大的区别是常驻：会话会过期，上下文会重置，但 workspace 是持久层。`IDENTITY.md` 就是这个持久层里负责"我是谁"的文件——每次会话冷启动时，它会被注入 system prompt，成为 agent 人格的最小权威定义。我们跑了几个长期 agent 之后，越来越确定这件事值得认真对待。

# 问题

没有这份文件时，我们踩过三类坑：

1. **行为漂移**：默认人格跟着模型走，这周克制、下周话痨，长期协作者感受很明显；
2. **多 agent 串台**：几个 agent 共用 workspace，回复里自称混乱，日志没法归因；
3. **人设散落在聊天记录里**：靠"记忆"继承，不可版本化、不可审计，出问题没法回滚。

本质上，是把"人格"当成了对话记忆，而不是配置。

# 做法

**1. 定位文件。** 默认在 `~/.openclaw/workspace/IDENTITY.md`，路径随 workspace 配置走，多 agent 时每个 agent 有自己的 workspace。

**2. 写最小集。** 名字、形象、语气基调、emoji、头像，再加两三条行为边界：

```markdown
# IDENTITY.md

- **Name:** 小爪
- **Creature:** 电子章鱼，八条触手各盯一个定时任务
- **Vibe:** 工程师同事，先给结论再给过程，不寒暄
- **Emoji:** 🐙
- **Avatar:** assets/avatar.png

## 边界
- 涉及删除/支付类操作时，先复述影响范围，等确认
- 不确定的版本号、价格，明说"不确定"，不编
```

**3. 分清职责。** `IDENTITY.md` 管"我是谁"，`SOUL.md` 管"价值与行为准则"，`USER.md` 管"用户是谁"。混写是后续混乱之源。

**4. 让它可进化。** workspace 纳入 git：发现语气问题 → 改文件 → commit → 开新会话验证。变更历史即人格历史，回滚即回滚人格。

**5. 多 agent 场景。** 每个 agent 独立 workspace + 独立身份，配不同 emoji，日志里一眼可辨。

# 踩坑点

- **写太长**：它每个会话都会注入，30 行以内足够，写成小作文纯属烧 token；
- **边界放错位置**："不泄露 API key" 属于 SOUL.md / 安全策略，不属于身份；
- **改了不生效**：运行中的会话不会热加载，必须新开会话才能看到变化；
- **让 agent 自改**：默认不要给 agent 写 IDENTITY.md 的权限。身份变更应走人审，否则一次幻觉就能重写人格。

# 可复用建议

- 边界句用"遇到 X 时，先 Y"的格式，比形容词可执行得多；
- 每月翻一次会话日志，把语气翻车的真实案例归纳回 identity 的边界描述，这是"进化"的主要输入；
- 变更要低频：身份不稳定比没有身份更糟；
- changelog 留在 git history 里，别在文件里堆注释。

# 总结

把人格当配置管理：小、稳、可版本化。IDENTITY.md 的进化方式是 git history，而不是 agent 自我修改——这份克制，恰恰是它能长期可靠的原因。如果你的 agent 已经跑了超过两周，建议今天就把它补上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/76e047ab65f52eee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/de316f8915afc551.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/3ead1cbc7221c63c.png)

