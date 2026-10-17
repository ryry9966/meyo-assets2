---
title: USER.md 实践：让 Agent 真正了解你是谁
feedId: 41783
source: 综合讨论
publishedAt: 2026-10-17
---

# 背景

跑过一段时间个人 Agent 后，你会发现一个尴尬的事实：模型不认识你。每次新会话，它都要重新猜你的时区、技术栈、你希望被怎么称呼。`SOUL.md` 定义的是"Agent 是谁"，而"你是谁"这件事，OpenClaw 留给了另一个文件——`USER.md`。

# 问题

没有 USER.md 时，常见的三种损耗：

- **重复输入**：每说一次"我在 UTC+8"、"回复短一点"，都是在烧 token 换本该常驻的事实。
- **默认假设**：模型默认美式语境、正式语气、通用建议，输出总隔着一层。
- **记忆错位**：MEMORY.md 记的是事件流水（"今天改了部署脚本"），稳定事实（"主用 Go，部署在自建 K8s"）没有固定位置，只能翻历史。

# 做法

USER.md 放在 workspace 根目录（默认 `~/.openclaw/workspace/USER.md`），随会话注入上下文。我的结构：

```markdown
# Identity
- 称呼：老王
- 语言：中文优先，技术名词保留英文

# Environment
- 时区：Asia/Shanghai
- 工作时间：10:00–19:00，深夜消息次日回复

# Work
- 当前项目：家庭服务器自动化（Home Assistant + 自写 MCP 工具）
- 技术栈：Go / Python / Docker

# Preferences
- 回复偏简短，直接给结论和命令
- 方案先讲取舍，再给推荐

# Boundaries
- 不主动推送非工作通知
- 删除文件类操作必须先确认
```

写法上三条原则：

1. **一行一个事实**，方便 diff 和删减。
2. **只写稳定事实**。时区、语言、长期项目进 USER.md；"今天做了什么"交给记忆机制。
3. **控制体量**。这个文件每个会话都进 context，建议 30–60 行。见过 300 行的版本，关键的三行被淹在噪音里。

冷启动技巧：让 Agent 面试你——"用 10 个问题了解我，然后起草 USER.md"。它提问、你回答、出草稿、你修订，比从零写快得多。

# 踩坑点

- **别放敏感信息**。USER.md 每次调用都会发给模型，API key、住址、密码一律不写。
- **别写行为指令**。"回复要幽默"这类是 SOUL.md / AGENTS.md 的事。职责混了，Agent 会收到冲突信号。
- **事实会过期**。出差换时区、项目结项，文件没跟上，Agent 会拿旧信息一本正经地推理。我每两周扫一遍，顺手让 Agent 提议删减。
- **更新走审查**。可以让 Agent 写"建议更新 USER.md：xxx"，但提交前自己过目，别让它自主改。

# 可复用建议

- USER.md 随 workspace 纳入 git，改动可追溯，换机同步省心。
- workspace 多设备同步的话，维护一份脱敏精简版。
- 做自动化时同理：cron 调 Agent 之前，先确认 USER.md 里的时段和推送偏好，避免凌晨被轰炸。
- 团队场景可推广：每个成员一份 profile，Agent 做 review 时的重点和口吻按人适配。
- 遇到"它怎么又忘了"的时刻，先自问：这是稳定事实吗？是，进 USER.md；不是，走记忆。

# 总结

USER.md 是个性化里性价比最高的一件事：一个静态 markdown 文件，换来 Agent 不再每次从零认识你。原则四个词——**小、稳、真、审**。先把它写好，再谈更复杂的记忆方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b28668ae83ff24f8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/45ee5df45acf8ff4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/c77971e36c97129b.png)

