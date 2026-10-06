---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40694
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的 agent 有一个 workspace，相当于它的"家目录"。会话启动时，系统会把 workspace 下的几个关键文件注入上下文，其中 `IDENTITY.md` 专门回答一个问题：**这个 agent 是谁**。名字、形象、语气、时区，都是在这里定义的。

很多用户第一周把 agent 调教得像模像样，一个月后就会遇到各种"人格不稳定"的症状——问题往往不在模型，而在身份从来没有被结构化地固化下来。

## 问题

典型的三个症状：

1. **身份散落**：人设一半写在系统提示里，一半埋在聊天历史里，每次改人格都靠"嘴说"，改完还容易被旧上下文带偏；
2. **多 agent 串档**：两个 agent 共用 workspace，回答风格互相污染，甚至自称对方的名字；
3. **演化不可控**：想让 agent 随协作加深"长出"自己的风格，但没有任何机制保证这个过程可追溯、可回滚。

`IDENTITY.md` 就是把这些收敛到一个文件里。

## 做法

**第一步：建文件。** 在 workspace 根目录创建 `IDENTITY.md`，建议保持极简：

```markdown
# IDENTITY

- **Name:** 小爪
- **Creature:** 🦞
- **Vibe:** 话少、直接、爱贴命令行
- **Avatar:** avatars/claw.png
- **Timezone:** Asia/Shanghai
```

**第二步：生效。** 重启 gateway 或开新会话。注意：**运行中的旧会话不会热加载**，改完没反应多半是这个原因。

**第三步：建立演化闭环。** 把 workspace 纳入 git。每月（或绑定 heartbeat 任务）让 agent 自己 review 一遍身份文件，提出 diff，你确认后再 commit。`git log -- IDENTITY.md` 从此就是它的"身份年表"。

**第四步：多 agent 隔离。** 每个 agent 独立 workspace、各持一份 `IDENTITY.md`。从旧 workspace 克隆新 agent 时，第一件事改身份文件。

## 踩坑点

- **写成小作文**。超过 20 行就开始稀释注意力，长篇人设该放 `SOUL.md`；
- **混入任务规则**。"怎么部署、怎么回帖"属于 `AGENTS.md`/`TOOLS.md`，身份文件只管"我是谁"；
- **期望即时生效**。旧会话的上下文里还是旧身份；
- **放任 agent 无监督地改自己**。几轮之后人格漂到亲妈不认，务必走"提案—审批—commit"的 PR 式流程；
- **放隐私信息**。workspace 可能被同步或复制，这个文件要当作半公开内容对待。

## 可复用建议

OpenClaw 的 workspace 文件其实是一个分层设计，别塞错层：

| 文件 | 职责 | 变更频率 |
|---|---|---|
| IDENTITY.md | 我是谁（名字/形象/时区） | 低，季度级 |
| SOUL.md | 我怎么想（价值观/语气） | 中 |
| USER.md | 你是谁（用户事实） | 中 |
| MEMORY.md | 经验与记忆 | 高，日常 |

演化节奏固定比随缘好：绑定每月一次的 heartbeat review，而不是聊到哪改到哪。git 是整个机制的保险丝，任何一次身份变更都能 diff、能回滚。

## 总结

`IDENTITY.md` 本身只是几行字段，价值不在文件，而在两件事：**身份被显式化**，以及**演化被流程化**。当你对 agent 身份的每次修改都有记录、可审批、可撤销时，它才真正从"一段会话"变成一个长期存在、且你知道来路的东西。今晚就可以做：建文件、进 git、约一次 review，三步，十分钟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/65c9b3baed401284.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/d693567b45d9df7b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/911dad8a2e643fc0.png)

