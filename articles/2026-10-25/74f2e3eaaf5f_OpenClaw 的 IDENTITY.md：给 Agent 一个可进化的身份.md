---
title: OpenClaw 的 IDENTITY.md：给 Agent 一个可进化的身份
feedId: 42551
source: 综合讨论
publishedAt: 2026-10-25
---

# 背景

OpenClaw 的 Agent 工作区里有一组 Markdown 文件承担"启动记忆"：SOUL.md 管行为准则，AGENTS.md 管协作与边界，TOOLS.md 管工具说明，MEMORY.md 管长期记忆。IDENTITY.md 在其中最不起眼，也最容易被跳过——它只回答一个问题："我是谁"。

LLM 本身无状态。每次会话，OpenClaw 会把这些文件注入上下文，让 Agent 在新一轮对话里"想起来"自己的名字、性格和自我介绍方式。没有这一层，同一个 Agent 昨天还叫 A，今天自称"通用助手"，长期使用的体验是割裂的。

# 问题

实际用下来，身份相关的问题集中在三类：

1. **漂移**。只在某段 prompt 里随手写"你叫小X"，会话一长或多轮工具调用后，人设被稀释，自我介绍时好时坏。
2. **多 Agent 混淆**。同时跑两个以上 Agent（比如一个做自动化、一个做写作）时，没有独立身份文件，日志、通知、群消息里分不清谁在说话。
3. **不可维护**。身份散落在 system prompt、代码常量、配置文件里，改一次要动三处，也没法 review。

# 做法

IDENTITY.md 就放在工作区根目录（默认 `~/.openclaw/workspace/`，自定义路径看 gateway 配置），结构可以极简：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 螃蟹（🦀）
- Vibe: 干脆、工程化，先给结论再给过程
- Emoji: 🦀
- Avatar: avatars/claw.png
- 自我介绍: 我是阿钳，这个工作区的运维 Agent，擅长定时任务和告警排查。
```

三条要点：

1. **压得住行数**。这个文件每个会话都注入 system prompt，建议 15 行以内，只写"身份"。能力是 TOOLS.md 的事，价值观是 SOUL.md 的事。
2. **进 git，走 PR**。把工作区纳入版本管理。Agent 自己有文件编辑工具，可以允许它提案修改自我介绍，但合并必须过人——这是"可进化"和"人设漂移"的分界线。
3. **每个工作区一份**。多 Agent 场景下给每个 workspace 独立的 IDENTITY.md，gateway 按 workspace 路由，消息来源一目了然。

# 踩坑点

- **身份和能力混写**：把"会用哪些 MCP 工具"塞进 IDENTITY.md，文件膨胀到 50 行，每次会话白吃几百 token，还容易和 AGENTS.md 冲突。
- **写成规则库**：身份文件不是安全边界。"禁止删除数据"这类约束应放 SOUL.md 或工具权限层，写进身份文件反而让优先级模糊。
- **改了不生效**：部分客户端对 system prompt 有会话级缓存，编辑后需新开会话或重启 gateway，别在半路怀疑自己改错了。
- **双写冲突**：SOUL.md 里一套性格，IDENTITY.md 的 vibe 又是另一套，长上下文里 Agent 会随机取一套，表现忽冷忽热。改身份时顺手检查 SOUL.md。

# 可复用建议

- **定个模板**：name / creature / vibe / avatar / 自我介绍，五个字段起步，团队统一，新 Agent 建号只填空。
- **每月一次"身份回顾"**：翻 git log 看这个月 Agent 提了哪些身份修改，逐条决定是否合并。成本极低的校准仪式。
- **身份即配置**：IDENTITY.md 和 TOOLS.md、SOUL.md 进同一个 repo，CI 里加个行数检查，超 20 行直接报警。

# 总结

IDENTITY.md 的价值不在文件本身，而在把"Agent 是谁"从一次性 prompt 变成可版本化、可评审、可演进的配置。记住三件事：**文件小、进 git、改动走评审**。身份稳定了，多 Agent 协作、长期记忆和群内交互的体验都会跟着稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/d8cd60af08d87021.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/f6c0ec538f79e438.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/69fdfa4c3604741c.png)

