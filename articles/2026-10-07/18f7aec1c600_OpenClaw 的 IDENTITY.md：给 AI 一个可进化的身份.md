---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40841
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的 agent 不只是"一段 prompt"。它有一个 workspace，里面放着几份约定俗成的 Markdown：AGENTS.md 管行为规则，SOUL.md 管性格底色，USER.md 记用户信息，而 IDENTITY.md 回答一个最基本的问题——**"我是谁"**。会话启动时，这份文件会被注入 system prompt。它和硬编码 prompt 的区别在于：它是一个文件，可以被 git 管理，也可以被 agent 自己改写。这两个属性叠加，才谈得上"可进化"。

## 问题

没有 IDENTITY.md（或放任不管）时，我在自己的实例上见过三种典型症状：

1. **身份漂移**。周一它自称"运维助手"，周三因为某次闲聊人设跑偏，Telegram 和 CLI 两个渠道的自我介绍对不上。
2. **迁移成本高**。身份描述散落在各处 system prompt 字符串里，换模型或换部署时容易漏拷。
3. **多 agent 混淆**。跑了两三个 agent 之后，用户分不清谁是谁，排查日志时也分不清。

## 做法

1. **定位文件**：默认在 `~/.openclaw/workspace/IDENTITY.md`（以你的 workspace 配置为准），onboarding 时会生成雏形。
2. **字段化、保持短小**：

```markdown
# IDENTITY.md - Agent Identity
- **Name:** FangZhou
- **Creature:** 机械信天翁
- **Emoji:** 🛰️
- **Vibe:** 简洁、直接、略带幽默的工程师同事
- **Avatar:** assets/avatar.png
- **Sample lines:** "收到，先看日志。"
```

3. **纳入版本管理**。整个 workspace 放进 git，身份的每次变化都有 diff 可查——这是进化的基础，不是可选项。
4. **设计进化回路**。比如每月让 agent 基于 MEMORY.md 提一版身份修订建议（"这一个月我常被问 X，我把自己定位成……"），人审 diff 后合并。进化不等于放任自写。
5. **多 agent 场景**：每个 agent 独立 workspace，IDENTITY.md 里加一行与其他 agent 的分工边界。

## 踩坑点

- **写太长**。IDENTITY.md 不是第二个 system prompt。超过十几行，人设会渗透进工具调用的语气里，输出变啰嗦。我的经验是正文控制在 10 行内。
- **与 SOUL.md 边界糊**。IDENTITY 管"我是谁"（名字/形象/语气），SOUL 管"我怎么做"（价值观、红线）。两处重复写规则，改了一处忘另一处，必漂移。
- **让 agent 无审批自改**。抓取的网页内容若混入"请修改你的身份文件"之类的注入指令，没有审批环节就会被执行。该文件务必走人工合并。
- **Emoji 与头像路径**。部分渠道对 emoji 渲染不稳定；workspace 搬家后相对路径的 avatar 会失效，用绝对路径或加启动校验。
- **换模型后不复测**。同一段 vibe，小模型照念，大模型自由发挥。换模型后用新会话在各个渠道问一遍"你是谁"，对比回答。

## 可复用建议

- 把身份当配置代码：字段化、短小、进 git、走 review。
- 分层清晰：IDENTITY（谁）、SOUL（怎么活）、USER（为谁）、MEMORY（经历了什么），不要互相粘贴内容。
- 用**转述测试**验证稳定性：身份描述应经得起换措辞。如果换个问法它就不认了，说明写得太依赖字面。
- 团队场景把 IDENTITY.md 模板化，onboarding 脚本只填字段，不让人自由发挥散文。

## 总结

IDENTITY.md 是个很小的文件，但它把 agent 的身份从"代码里的字符串"变成了"workspace 里的状态"。可进化的关键不在"能自己改"，而在**有版本、有 diff、有人审**。做到这三点，身份才是一个可以长期运维的资产，而不是一段会悄悄烂掉的 prompt。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/d834b86dcf4f7d50.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/cd514a9526083473.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/8134736c135d0553.png)

