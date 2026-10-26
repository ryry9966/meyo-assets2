---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42701
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

跑了一个多月 OpenClaw 之后，我遇到的第一个“长期问题”不是模型能力，而是它“是谁”。同一个 agent 接了 Telegram、WebChat 和 CLI 三个入口，白天回答干脆利落，晚上突然开始打长篇客套话；换个会话，连自我介绍的名字都不一致。这类漂移，靠在 system prompt 里塞几句话是压不住的。

OpenClaw 的做法是把人格拆成几个固定的小文件放在 workspace 里：`SOUL.md` 管性格与价值观，`USER.md` 记录用户是谁，而 `IDENTITY.md` 专门回答一个问题——**这个 agent 对外是谁**。每次新会话启动，这些文件会被注入 system prompt，成为身份锚点。

## 问题

默认的 IDENTITY.md 基本是个占位符，后果有三层：

1. **呈现不稳定**：跨渠道、跨会话的自我介绍和称呼随机；
2. **职责不清**：有人把语气要求、行为规则全塞进 IDENTITY.md，和 SOUL.md 打架；
3. **无法演化**：身份全靠“感觉改改”，改错了没有回滚，也不知道当初为什么这么写。

## 做法

我的 workspace 在 `~/.openclaw/workspace/`，把 IDENTITY.md 收敛成十几行：

```markdown
# IDENTITY.md

- **Name:** 阿钳
- **Creature:** 一只随身带扳手的寄居蟹
- **Vibe:** 克制、直接、带点冷幽默；先结论后依据
- **Emoji:** 🦀
- **Avatar:** assets/crab.png
```

几点原则：

- **身份和灵魂分家**：名字、形象、emoji、avatar 归 IDENTITY；语气、底线、优先级归 SOUL。两边不写重复内容。
- **保持极短**：这个文件每次会话都会进 prompt，行数就是 token 成本，也会稀释真正的指令。
- **用 git 管演化**：整个 workspace 目录进仓库。每次因为某个具体行为不满意而改身份，就提交一个 commit，message 写清“为什么”。三个月后回头看 diff，能看出这个 agent 是怎么长成现在这样的。
- **人审合并**：我会让 agent 在发现自己反复被纠正某类称呼或姿态时，主动在会话里给出 IDENTITY 修改草稿，但只输出 diff，由我决定是否落到文件里。模型可以提议，不可以自己改。

## 踩坑点

- **改完不生效**：IDENTITY.md 在新会话时加载，正在跑的长会话不会热更新，需要重开会话或重启 gateway。
- **写太长**：早期我塞了四十多行“性格描述”，结果风格没变稳，常规任务的指令遵循反而变差。
- **跨文件冲突**：IDENTITY 里写“简洁”，SOUL 里又写“细致解释”，模型表现随机。修法是同一件事只留一处。
- **别放敏感信息**：这个文件会进入每一次请求的上下文，真实姓名、地址、密钥类内容一律不写。

## 可复用建议

- 身份文件控制在 15 行以内，一行一条事实；
- “变成谁”改 IDENTITY，“怎么说话”改 SOUL，“服务谁”改 USER，别混；
- 多实例（工作号/生活号）用不同 workspace，身份各自独立；
- 每月或每次觉得“语气不对”时 review 一遍 git log，比凭感觉重写靠谱。

## 总结

IDENTITY.md 只是个十几行的 markdown，但它是这个 agent 身份的唯一事实来源。把它当代码维护——短、单一职责、版本化、人审合并——身份就不再是调 prompt 时的玄学，而是一份能看到演进历史的工程产物。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/d86594fd141c307d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/aaa6041e06f62a7e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/630e9ce5d77cfba0.png)

