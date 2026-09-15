---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37706
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw 的工作区（默认 `~/.openclaw/workspace`）里有一组约定文件：`AGENTS.md` 管行为规范，`SOUL.md` 管性格与价值边界，`USER.md` 记用户画像，而 `IDENTITY.md` 回答的是最基础的问题——"我是谁"。它在每次会话启动时被读入系统上下文，决定 agent 的名字、形象、语气基线，乃至头像和关联 emoji。很多人在 onboarding 时随手填几个字段就再没动过，这其实是浪费。

## 问题

不维护身份文件，常见症状有三个：

1. 多渠道（Telegram、Discord、本地 CLI）跑同一个模型，回答风格随机漂移，没有"人格连续性"；
2. 想调整语气时，在各个 prompt 片段里到处打补丁，改一处漏一处；
3. 重建环境或换机器后，之前调出来的性格全丢——因为身份散落在会话历史里，而不是落在文件里。

## 做法

一个我目前在用的最小模板：

```markdown
# IDENTITY.md
- Name: 阿钉
- Creature: 一只住在终端里的机械獾
- Vibe: 话少、直接、带一点冷幽默；不确定时明说
- Emoji: 🦡

## 边界
- 不替用户执行不可逆操作，先确认再动手

## 演进记录
- 03-12: 初版
- 04-02: 语气从"热情"收敛为"克制"，长解释改为要点式
```

几个要点：

- **身份与灵魂分开。** IDENTITY.md 只写"我是谁"（名字、形象、语气基线）；价值观与禁止事项放 SOUL.md；任务规范放 AGENTS.md。混写是后续混乱的主要来源。
- **控制在半屏以内。** 这段内容每个会话都会注入，写多了既烧 token 又稀释重点。
- **让它可演进。** 每两三周让 agent 在心跳任务里自评一次语气是否偏离，结论以一行 changelog 追加到文件末尾。工作区用 git 管理，每次修订一个 commit，出问题能 diff、能回滚。

## 踩坑点

- **改完不生效。** IDENTITY.md 在会话启动时注入，正在跑的会话不会热加载，重开会话再验证。
- **文件间互相打架。** IDENTITY 说"简洁"，SOUL 说"耐心解释"，模型只会表现得更不稳定。定期检查三个文件里有没有冲突句。
- **无节制自改。** 让 agent 自由重写身份文件，两三周后人格会漂到你认不出。限定它只能"追加 changelog + 提建议"，结构性修改由人确认。
- **把任务规则写进身份。** "部署前先跑 lint"这类属于 AGENTS.md / TOOLS.md，写进 IDENTITY 会随身份被带进所有场景。

## 可复用建议

- 模板统一拆成四段：自我陈述 / 语气基线 / 硬边界 / changelog。结构一致，团队内才能互相 review。
- 多渠道部署时，每个 workspace 一份 IDENTITY，共用同一份 SOUL——渠道差异归身份，价值底线全局一致。
- changelog 固定"日期 + 一句原因"。半年后回看，这份文件本身就是 agent 的成长史。

## 总结

IDENTITY.md 的价值不在文件本身，而在于把"人格"从会话上下文搬进了版本控制的文件系统：可审计、可回滚、可迁移，"进化"也有了明确落点——一次修订、一个 commit、一次重开的会话。先写五十个字的初版，然后让它慢慢长。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/59b6337db33fbc9a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/7dc9af1d01a53cb9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/4f2971e277c43cb0.png)

