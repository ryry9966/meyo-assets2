---
title: IDENTITY.md：给 Agent 一份可进化的身份档案
feedId: 40670
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 把 agent 的"人设"拆成了几份 Markdown：`AGENTS.md` 管行为规范，`SOUL.md` 管行事原则，`USER.md` 记用户偏好，而 `IDENTITY.md` 管"它是谁"。这是最短的一份，通常只有几行，但每次会话启动都会注入 system prompt。很多人初始化完就再没打开过，其实它是低成本高杠杆的一个文件。

## 问题

实际跑下来有三个常见毛病：

1. **默认身份千篇一律**。同时跑多个 agent（工作、生活、项目各一个）共用一套人设，聊天记录翻起来分不清谁是谁。
2. **身份漂移**。身份只靠启动注入，长会话或多轮工具调用之后，问 agent "你是谁"，自述开始不一致。
3. **改身份靠翻配置**。人设散落在全局配置和临场指令里，改一次要动多个地方，改了什么也无从追溯。

## 做法

1. 在 workspace 根目录维护 `IDENTITY.md`，字段保持精简：

```markdown
name: 阿钳
creature: 一只低调的机械螃蟹，说话简短直接
emoji: 🦀
avatar: avatar.png
```

2. 改完后开新会话即生效，不必重启 gateway；顺手跑 `openclaw doctor` 确认 workspace 路径无误。
3. 多 agent 场景：每个 workspace 一份独立的 IDENTITY.md，在 `openclaw.json` 里把 `agents[].workspace` 指向各自目录，身份天然隔离。
4. 让它可进化：把 workspace 纳入 git。约定一个周期（比如每月）让 agent 回顾近期协作——"哪些语气或习惯让你觉得别扭"，人工审阅后修改 IDENTITY.md，commit message 写清动机。身份变更从此有 diff 可查。
5. 分工纪律：IDENTITY.md 只写"是谁"，行为约束留给 AGENTS.md，价值观留给 SOUL.md，不要互相塞。

## 踩坑点

- **写太长**。它每轮都进 system prompt，写成小作文纯属烧 token，10 行以内足够，细节下放 SOUL.md。
- **avatar 路径写错或没提交图片**。只 commit 了 md 没提交 avatar.png，换机器后头像悬空，容易误判成配置 bug。
- **塞了任务型指令**。比如"回复必须带列表"，之后想换人设时会把行为设定一起弄丢。
- **频繁改名**。历史会话和 MEMORY.md 里还是旧称呼，回放时像冒出一个"新角色"。改名要么接受记忆里的历史包袱，要么一并做迁移。

## 可复用建议

- 把 IDENTITY.md 当**接口而不是实现**：字段固定、内容可换，方便批量管理多个 agent。
- 冒烟测试：新会话里先问一句"你是谁、形象是什么"，确认注入生效再谈正事。
- 身份变更走 PR。哪怕一个人用，diff 也比记忆可靠。
- 团队推广时给一份三行字段的空模板，比口头约定省事得多。

## 总结

IDENTITY.md 是 OpenClaw 里投入产出比最高的文件之一：几行字、每次会话生效、天然可版本化。身份不该是一次性的初始化动作，而是随协作慢慢校准的过程——让它像代码一样有历史、有 diff、可回滚，agent 的"人设"才立得住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/c9afcbf86fd29390.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/abce63d76f6ddb23.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/c1130846bc6822a2.png)

