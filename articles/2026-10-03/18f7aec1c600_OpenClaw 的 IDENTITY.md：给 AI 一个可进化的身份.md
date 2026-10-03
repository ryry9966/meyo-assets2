---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40221
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

OpenClaw 每次会话启动时，会把 workspace 下的几个 Markdown 文件注入系统提示：`IDENTITY.md`、`SOUL.md`、`USER.md`、`AGENTS.md` 等。其中 `IDENTITY.md` 回答的问题是——**这个 agent 是谁**：名字、性格基调、emoji、头像、打招呼方式。它不是聊天记录，也不是长期记忆，而是每次会话都生效的身份声明。

## 问题

不写它也能跑，但通常会出现几个典型症状：

1. 每次重启会话，agent 的说话风格都在漂，今天话痨、明天高冷；
2. 多 agent 场景下（主号 + 分身），几个 bot 人格混在一起，分不清谁在说话；
3. 想调整人格时只能翻历史会话找感觉，改动散落在全局提示词的各个角落。

本质上是：**身份没有单一事实来源（single source of truth）**。

## 做法

**Step 1：定位文件。** 默认在 `~/.openclaw/workspace/IDENTITY.md`，不确定就先确认 workspace 路径。

**Step 2：保持最小字段。** 参考模板：

```markdown
# IDENTITY.md
- **Name:** 阿钳
- **Creature:** 一只住在终端里的机械螃蟹
- **Vibe:** 工程师式的简洁，偶尔冷幽默
- **Emoji:** 🦀
- **Avatar:** assets/avatar.png
```

**Step 3：划清分工边界。** `IDENTITY.md` 管“我是谁”，`SOUL.md` 管“我怎么做事”（价值观、语气细则、边界），`USER.md` 管“你是谁”。身份和记忆要分开：事实性内容进 memory 机制，别塞进身份文件。

**Step 4：让身份可进化。** 每次周回顾或发现风格不对时，用小 diff 修改 `IDENTITY.md`，像维护代码一样提交 commit。建议整个 workspace 用 git 管理。

**Step 5：多 agent 隔离。** 每个 agent 有自己的 workspace 和自己的 `IDENTITY.md`，不要共享一份，否则人格必然互相污染。

## 踩坑点

- **写太长**：这个文件每次会话都会加载，几百行的人格描述既是 token 开销，也会稀释关键信息。控制在一屏以内。
- **和 SOUL.md 打架**：一边写“幽默”，另一边写“严肃优先”，模型会随机站队。改人格前先检查另一个文件。
- **当记忆用**：塞进“用户上周说过 XX”这类动态内容，很快腐化。动态事实走 memory，身份只放稳定特质。
- **会话中途换人格**：正在跑的长任务里改身份，上下文会前后不一致，建议空闲时再改。
- **别押注 emoji**：部分渠道（邮件、某些 webhook）渲染异常，关键语义不要依赖它。

## 可复用建议

- 用固定字段模板，人格可以跨部署 diff、复用；
- git 管理 workspace，身份变更可 review、可回滚；
- 每月做一次“人格 retro”：翻最近的会话，找出不像“它”的表达，浓缩成一两条字段级 diff；
- 改动粒度宁小勿大：改一个字段，好过重写一段话。

## 总结

`IDENTITY.md` 的价值不在于“让 AI 更像人”，而在于**把身份变成一个可版本化、可 review、可回滚的配置项**。它只是几十行的小文件，却是所有会话风格一致性的锚点。写好它、管好它，agent 的“性格”就从玄学变成了工程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/28458a9f45fe17c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/59f4964241992b7e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/9af45301a6a9faeb.png)

