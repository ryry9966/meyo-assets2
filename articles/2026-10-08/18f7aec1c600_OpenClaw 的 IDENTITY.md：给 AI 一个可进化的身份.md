---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40851
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 的工作区里有一组约定俗成的 Markdown 文件：`AGENTS.md` 管操作手册，`SOUL.md` 管价值观和边界，`USER.md` 管用户上下文，`MEMORY.md` 管长期记忆。`IDENTITY.md` 是其中最不起眼的一个——通常只有几行：名字、物种（creature）、emoji、头像。每次会话启动时它会被注入 system prompt，Agent 从此"知道自己是谁"。

## 问题

没有它也能跑，但会踩到三类实际麻烦：

1. **跨会话人格漂移**。模型默认人格会被对话内容带偏，今天高冷明天话痨，长期协作用户很难建立预期。
2. **多实例混淆**。Telegram 和 Discord 各跑一个 Agent，共享行为配置后回复风格趋同，用户分不清在跟谁说话。
3. **身份写死在代码里**。把人设塞进 system prompt 常量，每次微调都要发版，非工程角色无法参与。

## 做法

1. 在工作区根目录（默认 `~/.openclaw/workspace`）创建 IDENTITY.md：

```markdown
# IDENTITY.md
- **Name:** Walle
- **Creature:** 一只务实的工程助手，像爱整理工具的獾
- **Emoji:** 🦡
- **Vibe:** 简洁、直接、偶尔冷幽默
```

2. 分层要干净：IDENTITY.md 只回答"我是谁"，行为准则放 SOUL.md，操作流程放 AGENTS.md。文件之间重复的内容就是漂移的温床。
3. 让它可进化但不失控：把易变的偏好（比如"最近用户喜欢 bullet 点回复"）先放 MEMORY.md 观察一两周，确认稳定后再沉入 IDENTITY.md，一次只改一个字段。
4. 多 Agent 场景给每个实例独立工作区，各自的 IDENTITY.md 描述不同物种、语气和 emoji。
5. 改完用 `/new` 开新会话（或重启 gateway）验证生效。

## 踩坑点

- **会话中改文件不生效**。身份在会话启动时注入，改完必须开新会话，别对着旧会话质疑人生。
- **把它写成第二份 SOUL.md**。内容超过一屏，说明你在写规则而不是写身份，该归位的归位。
- **让 Agent 自改身份**。它可能把"这次对话里用户的批评"直接写进去，两周后人设面目全非。自改要走"提案 diff + 人工 review"流程。
- **没做版本控制**。工作区顺手 `git init`，每次身份变更留 commit，漂移可回滚。

## 可复用建议

- 把 IDENTITY.md 当配置文件管：小步提交，commit message 写清改了什么、为什么。
- 每月做一次"身份审计"：翻 memory 日志，删过时描述，保持文件在 10 行以内。
- 团队场景把 workspace 收进仓库，用部署脚本分发，身份即代码。

## 总结

IDENTITY.md 只有几行，却把 Agent 从"无状态工具"变成"有连续性的协作者"。进化的关键不在文件本身，而在那套克制的变更流程：先观察、再提案、后合入。身份像代码一样需要 review——这句话值得写进你团队的 AGENTS.md。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/943e302154824d1a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/6a39f6b46d568f31.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/69ff47ead440ff37.png)

