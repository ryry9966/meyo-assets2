---
title: OpenClaw 实践：用 IDENTITY.md 给 Agent 一个可进化的身份
feedId: 39566
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的配置思路很直接：与其把人设埋在系统提示词里，不如拆成工作区里的几个 Markdown 文件。`IDENTITY.md` 管的是"我是谁"——名字、形象（Creature）、emoji、气质（Vibe）、配色、头像。旁边的 `SOUL.md` 管行为边界和价值观，`USER.md` 管对用户的记忆。三者分工不同，但我一开始没分开写。

## 问题

最初的状态是：人设描述散落在 SOUL.md、提示词模板和渠道配置里；改一次名字要翻三个文件；没有版本记录，agent 用了两周后人设越漂越远，想恢复也无处下手。身份不是一个 prompt 字符串，它是一个需要持续维护的状态——这就需要把它工程化。

## 做法

1. **单一事实源**。在 `~/.openclaw/workspace/IDENTITY.md` 建文件，字段保持精简：

```markdown
Name: 阿钳
Creature: 机械寄居蟹
Emoji: 🦀
Vibe: 直接、克制、先给证据再下结论
Colors: 石板蓝 / 珊瑚橙
Avatar: assets/avatar.png
```

2. **清理其他文件**。把散落的人格描述收拢到这里。SOUL.md 只留"怎么做事"，IDENTITY.md 只留"是什么"。
3. **用 git 管起来**。workspace 本来就该是仓库，身份变更加 commit，diff 就是评审记录。
4. **可进化，但受控**。允许 agent 在长期交互后对 Vibe 提出修改建议，追加到文件末尾的 `Draft:` 区；我定期（和每周记忆整理一起）人工裁决后合并。进化来自真实使用，不是让它随便改自己。
5. **多实例隔离**。多个 agent 各自独立 workspace、各自 IDENTITY.md；共享价值观下沉到 SOUL.md，避免人设互相污染。
6. **验证**。改完执行 `/new` 开新会话，让 agent 自我介绍一遍，两个渠道各测一次，确认名字、emoji、语气一致。

## 踩坑点

- **改完不生效**。IDENTITY.md 是会话启动时读入的，中途改文件不影响当前上下文，`/new` 一下就好。
- **越写越长**。见过 60 行的版本，连回复格式规则都塞了进去。行为规则归 SOUL.md/AGENTS.md，身份文件超过 15 行基本说明职责漏了。
- **放开让 agent 自己改**。试过全自动，两周后人设漂成"热心销售"。现在是提案-审批制，git 历史随时回滚。
- **渠道配置里有字面量副本**。部分渠道配置会再抄一遍名字/emoji，改了 IDENTITY.md 记得同步，否则两处不一致。
- **别混内容**。用户相关信息是 USER.md 的事，密钥更不该出现在这里。

## 可复用建议

- 把 IDENTITY.md 当配置文件对待：声明式、短、版本化、走 diff 评审。
- Vibe 写 3~5 个具体形容词加一句"绝不……"，比一段抒情散文有效得多——模型对具体词的遵循远好于氛围描述。
- 做一个基础模板仓库，新实例复制后只改差异字段。
- 身份评审和记忆整理绑成同一个例行动作，一起做，不容易忘。

## 总结

IDENTITY.md 的价值不在于"给 AI 起个名字"，而在于把身份变成可版本化、可评审、可回滚、可演进的工程对象。文件很小，但它定义了 agent 与你之间关系的稳定基线：所有进化都发生在明确的 diff 里，而不是悄悄发生的漂移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0257eee8d1b01348.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ae0568e2244f6885.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/af9a47fa464c4883.png)

