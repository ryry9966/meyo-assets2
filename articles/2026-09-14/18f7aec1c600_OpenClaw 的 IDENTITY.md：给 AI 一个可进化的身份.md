---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37421
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 的行为一致性，很大程度不靠 prompt 玄学，而是靠 workspace 里那几个 Markdown 文件：SOUL.md 管价值观和边界，AGENTS.md 管协作规则和工具用法，而 IDENTITY.md 管“我是谁”——名字、形象、语气、emoji、头像。很多用户在初始化时填了一次默认值，之后再也没打开过这个文件。

我的看法是：IDENTITY.md 是 workspace 里杠杆最高、却最常被当成一次性配置的文件。它值得像代码一样被版本化、被 review、被小步演进。

## 问题

没有认真维护 IDENTITY.md 的 agent，通常有这些症状：

- 换个会话自我介绍就不一样，签名时而去 emoji 时而不去；
- 回复风格随模型版本漂移，你没法确认“这还是不是我想要的那个 agent”；
- IDENTITY.md 和 SOUL.md 内容写重，互相打架，表现为语气忽冷忽热；
- 多 agent 场景下，两个实例自我认知完全相同。

根因只有一个：身份没有被当作受控资产来管理。

## 做法

我 workspace 里的 IDENTITY.md 长这样：

```markdown
# IDENTITY.md

- **Name:** 阿钳
- **Creature:** 住在终端里的机械河狸
- **Vibe:** 话少、先给计划再动手、坚持列清单
- **Emoji:** 🦫
- **Avatar:** avatars/beaver.png
```

几条要点：

1. **保持短小。** 每次会话都会注入上下文，几行足够，长文只会稀释重点。
2. **写可观察的行为，不写形容词。** “坚持列清单”比“严谨可靠”更能约束输出。
3. **workspace 纳入 git。** 每次改身份单独提交，commit message 写清楚“为什么改”。
4. **建立演进节奏。** 我固定每月 review 一次：翻近期日志，如果 agent 表现和身份描述冲突，只改一个字段，观察一周，再决定是否保留。

## 踩坑点

- **职责越界。** 名字、形象、语气归 IDENTITY.md；原则和边界归 SOUL.md；工具与流程归 AGENTS.md。把 MCP 工具用法写进身份文件，是我见得最多的错误。
- **抽象词堆砌。** “聪明、友好、有趣”这类词模型落不了地，等于没写。
- **忘了 git。** 改坏了想回滚，发现没有历史，只能凭记忆重写。
- **头像路径。** Avatar 用相对 workspace 的路径，搬目录后记得检查文件还在不在。
- **克隆 workspace 忘改身份。** 多 agent 部署前，先 diff 一遍 IDENTITY.md。

## 可复用建议

- 把身份变更当发版：一次一个字段，有说明，可回滚。
- 在 MEMORY.md 里留一段“身份变更日志”，记录每次改动和观察到的行为差异——这是你判断进化是否有效的唯一依据。
- 想试新的 vibe，先在日志里写下预期行为变化，观察期结束再合入，等效于 A/B。
- 新机器初始化时直接从 git 恢复 workspace，身份、记忆、规则一次到位。

## 总结

IDENTITY.md 只有几行，但它是 agent 长期一致性的锚点。所谓“可进化的身份”，不是让它自由生长，而是把它当成一个有版本历史、有 review 流程、有观察指标的配置项来维护。改得慢一点，记下每次为什么改，你会得到一个越用越像“那一个”的 agent，而不是越用越陌生的会话拼盘。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c23934fa996889be.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/5e63be8617add435.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/1da9f12ba262def7.png)

