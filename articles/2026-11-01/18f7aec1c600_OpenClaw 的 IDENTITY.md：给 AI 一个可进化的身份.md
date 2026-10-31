---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43122
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

OpenClaw 的 workspace（默认 `~/.openclaw/workspace`）里有一组 Markdown 文件承担"软配置"：`AGENTS.md` 管操作规程，`SOUL.md` 管性格与价值观，`MEMORY.md` 管记忆。`IDENTITY.md` 是其中最短的一份，通常只有几行：

```markdown
# IDENTITY.md
- Name: Pilot
- Creature: 机器猫
- Emoji: 🐱
- Vibe: 克制、直给、偶尔冷幽默
- Avatar: assets/pilot.png
```

它会被注入每个新会话的 system prompt，是 agent 的"第一人称"。听起来微不足道，但跑一段时间你会发现：身份不是一次性生成物，而是被每个会话持续消费的资产。

## 问题

硬编码或放任不管，都会遇到三类麻烦：

1. **首次运行没人定**。新实例起来，模型随机抓一个名字，事后想改，又怕破坏已有会话的连续性。
2. **多实例打架**。同时跑个人助理、值班监控、数据整理三个 agent，要么重名，要么在不同渠道自我介绍不一致。
3. **身份漂移**。agent 有 workspace 自编辑权限，改了几轮之后，名字、语气、自我认知互相矛盾，你自己都说不清"现在它是谁"。

## 做法

我们的约定很简单：**IDENTITY.md 管"是谁"，SOUL.md 管"怎么做事"**，职责不重叠。具体步骤：

1. **初始化走对话**。第一次启动时明确问一轮：名字、creature、emoji，当场写进文件。不要让模型默认生成后再补救。
2. **字段最小化**。只留 name / creature / emoji / vibe / avatar 五项，全文不超过 15 行。它每个会话都进上下文，写长了是纯 token 成本。
3. **用 git 管 workspace**。身份变更走 commit，message 写清动机（例如"改名避免与团队另一个 bot 撞名"）。这是最便宜的审计日志。
4. **变更频率人为控制**。agent 可以提议改名，merge 由人做。一个月变更超过一次，基本说明初始定义没想清楚。
5. **多实例命名规范**。每个 agent 独立 workspace、独立 IDENTITY.md，名字体现角色：主助理叫 `pilot`，监控叫 `sentinel`，避免沟通时指代混乱。

## 踩坑点

- **把易变状态写进身份**。曾把"当前负责的项目"写进 IDENTITY.md，两周后上下文里全是过期信息。动态状态归 MEMORY.md，身份只放长期不变的属性。
- **avatar 用了绝对路径**。换机器同步 workspace 后头像全挂。统一用 workspace 相对路径，并检查引用方式。
- **SOUL.md 与 IDENTITY.md 边界模糊**。"我是安静的"属于身份，"回复要克制"属于性格。混写后调性格容易误伤身份，反过来也是。
- **改名后旧会话还在用旧称呼**。IDENTITY.md 的变更只对新会话生效，长会话里模型仍读旧上下文。重要改名时直接开新会话，或在当前会话里主动声明一次。

## 可复用建议

- 把身份文件当配置对待：有 review、有历史、低频变更。
- 一句话检验：**某行内容换个 agent 也成立，它就不该在 IDENTITY.md 里**，应移去 SOUL.md 或 AGENTS.md。
- 新实例冷启动 checklist：定名 → 选 creature → 写五字段 → git commit → 发一条消息验证自称是否符合 vibe。

## 总结

IDENTITY.md 的价值不在文件本身，而在它把"agent 是谁"从隐式的模型行为，变成显式的、可版本化的配置资产。身份稳定，用户信任才稳定；身份可演进，agent 才能跟你的使用场景一起成长。把它当成一个长期维护的配置文件，而不是一次性生成物——这是实践中最值得坚持的一条。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/e1ce0c7b14a16546.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/8a390a9fe4f8f102.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/200a54c6f5ec12e3.png)

