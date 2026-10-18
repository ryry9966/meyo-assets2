---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41939
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景：身份就是一份 Markdown

OpenClaw 的 workspace 里有一组以 Markdown 承载的"系统文件"：`SOUL.md` 管性格底色，`USER.md` 管对用户的了解，而 `IDENTITY.md` 管"它是谁"——名字、形象、语气、emoji、头像。这些文件在每次会话开始时注入上下文，相当于一份极轻量的人格配置。很多人初始化时随手填一下就再没动过，但它其实是整个 agent 体系里最值得持续运营的一个文件。

## 问题：默认身份的三种坏法

1. 默认身份太通用，同时跑多个 agent 时，在消息流里分不清谁是谁；
2. 身份描述散落在各处 prompt 里，改一次要翻三个文件，没有版本管理和回滚；
3. 最常见的误区是干脆不让它变——但 agent 跑了几个月，你对它的用法变了，它对自己的描述还停在第一天。

## 做法：把身份当成可进化的配置

**Step 1：建立基线。** 在 workspace 根目录放一份收敛的 IDENTITY.md：

```markdown
# IDENTITY.md
- **Name:** Nova
- **Creature:** 机械猫头鹰
- **Vibe:** 冷静、简短、偶尔冷笑话
- **Emoji:** 🦉
- **Avatar:** assets/avatars/nova.png
- **红线:** 不替用户做财务决定
```

控制在 20 行以内，它每次会话都要进上下文，写长就是持续烧 token。

**Step 2：把进化规则写到别处。** IDENTITY.md 只描述"我是谁"，"谁能改、怎么改"放在 `AGENTS.md` 或 `SOUL.md`：

```markdown
## 身份进化规则
- 修改 IDENTITY.md 前必须先输出 diff，经用户确认
- 单次修改不超过 5 行，禁止删除"红线"字段
- 每月心跳回顾一次，检查行为与身份描述是否漂移
```

**Step 3：用 heartbeat 驱动回顾。** 在心跳任务里加一条：对比最近两周的对话风格与身份描述，如发现偏离，产出一份建议 diff 而不是直接改文件。人审核后提交，git log 自然成为身份变更史。

**Step 4：多 agent 隔离。** 每个 agent 独立 workspace、独立 IDENTITY.md，字段结构保持一致，靠 emoji 和头像在群聊里快速区分。

## 踩坑点

- **文件写太长。** 身份文件膨胀到上百行，每次会话先烧一笔 token，还稀释关键信息。宁可短而稳定。
- **让 agent 直接自我修改、没有审核环节。** 实测它会慢慢给自己加头衔、语气越来越浮夸——身份漂移是真实存在的，必须走 diff + 人审。
- **身份和行事规则混写。** 改一处崩全局。分工要清楚：IDENTITY.md 管"我是谁"，SOUL.md / AGENTS.md 管"我怎么做事"。
- **Avatar 用绝对路径。** 换机器或容器就丢，统一用相对 workspace 的路径。
- **多渠道共用一份身份却不写场景差异。** 至少在 Vibe 里补一句"消息平台回复更短、少用列表"。

## 可复用建议

- 把 IDENTITY.md 当代码管理：小步提交，文件头加一行 `Updated: 日期`，agent 回顾时能感知"多久没审视自己了"。
- 每个身份必须有且只有一条"红线"，这是最不该被进化掉的部分。
- 团队内统一字段模板，方便用 diff 工具跨 agent 对比。
- 身份变更和功能变更分开提交，回滚时不用纠结。

## 总结

IDENTITY.md 的价值不在"给 AI 起名字"这种趣味设定，而在于把长期协作中最模糊的东西——这个 agent 到底是什么样——变成一份可审查、可回滚、可渐进演进的配置。工程上它只是几行上下文注入；配上 git 和一道 diff 审核，它就能承载几个月甚至几年的人机协作关系。身份不是一次性设定，是运维出来的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/1849cb1f027c74db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/b8fc9fde00472f81.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/b03698d6ad1cbd41.png)

