---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 37987
source: 综合讨论
publishedAt: 2026-09-17
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

用 OpenClaw 这类常驻 Agent 一段时间后，会反复遇到同一个尴尬：每次新会话，它对你一无所知。你要重新告诉它"我机器是 macOS，包管理用 pnpm 不用 npm""回答直接给结论，别科普基础概念"。这些信息每次都要说，说了它也记不牢。

system prompt 是给 Agent 定规矩的，但"你是谁"这件事，需要一个专门的文件来承载——这就是 workspace 里的 USER.md。

## 问题

没有一份稳定的用户档案时，常见症状有三个：

1. **重复输入成本高**：同样的偏好每个会话讲一遍，讲漏了就跑偏。
2. **Agent 瞎猜默认值**：不知道你的操作系统、语言习惯、技术栈，它就按训练数据里的"平均值"来，输出风格不稳定。
3. **上下文被临时信息污染**：为了当下对话塞进去的个人背景，事后成为噪音。

## 做法

USER.md 放在 workspace 根目录（默认 `~/.openclaw/workspace/USER.md`），OpenClaw 启动会话时会随 bootstrap 一并注入上下文。建议按事实分节写：

```markdown
# User

## 基本信息
- 称呼：老周
- 时区：Asia/Shanghai

## 环境
- 主力机：macOS (Apple Silicon)
- 包管理：pnpm；Node 22 LTS
- 编辑器：Neovim

## 偏好
- 回答先给结论，细节放后面
- 代码示例默认 TypeScript，不加注释除非我要求

## 边界
- 不要主动升级依赖
```

三个要点：

- **只写事实和可执行的偏好**。"我喜欢高质量代码"这种没法落地的描述不要写。
- **分工清楚**：USER.md 放"关于我的事实"，行为规则放 AGENTS.md，长期记忆放 MEMORY.md。三者混写是后面一切混乱的源头。
- **冷启动可以偷懒**：直接让 Agent 面试你一轮（"问我 10 个问题来建立你的用户档案"），把答案整理成初稿，再人工删改。

## 踩坑点

1. **写太长**。USER.md 每个会话都占上下文，控制在 30 行以内，宁精勿全。我见过塞了 200 行项目史的，关键偏好反而被稀释。
2. **把密钥写进去**。这个文件会被注入每次会话，等于明文广播。token、内网地址、客户名一律不放。
3. **信息过期**。换了机器、从 npm 迁到 pnpm，旧条目不会自己消失。文件头加一行 `last updated`，每月扫一遍；或者干脆用 git 管 workspace，diff 一眼看出漂移。
4. **期望它百分百生效**。注入了不等于每次都遵守。真正关键的约束（比如"不要动我的 CI 配置"）要写进 AGENTS.md 的规则区，用祈使句，别指望 USER.md 里一句顺带的描述起作用。

## 可复用建议

- 把 USER.md 当**数据库记录**而不是日记：短、结构化、可 diff。
- 建立"发现即写入"的习惯：当你第二次纠正 Agent 同一件事，就说"把这个写进 USER.md"。
- 多设备/多实例场景，用 git 同步 workspace，USER.md 跟着走。
- 定期让 Agent 自查："读一下 USER.md，指出哪些信息可能过期"，比人工回忆省力。

## 总结

USER.md 解决的不是技术问题，而是一致性问题：把散落在聊天记录里的"你是谁"，收敛成一份版本化、可维护的静态档案。它不智能，但正因为是纯 Markdown，它可审查、可回滚、可迁移。花二十分钟写好第一版，之后每次会话省下的重复沟通，都是净收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/e8d4541e1dc82b55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/1a6b6279fc76ea69.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/c10850fe39169160.png)

