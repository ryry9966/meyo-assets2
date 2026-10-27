---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42733
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

OpenClaw 的 workspace 里有一组"启动文件"：AGENTS.md 管行为约定，SOUL.md 管性格与价值观，USER.md 记录用户偏好，而 IDENTITY.md 回答一个更基本的问题——**你是谁**。它默认只有几个字段：Name、Creature、Vibe、Emoji、Avatar。很多人初始化时随手填一下就再没动过，跑久了才发现，这五行文件其实是跨会话一致性的锚点。

## 问题

没有认真维护 IDENTITY.md 时，常见三类现象：

1. **身份漂移**：长会话后期或新 session 里，agent 自我介绍的名字、语气开始不稳定；同时跑多个 agent 时，谁是谁全靠猜。
2. **边界混乱**：性格要求写进 IDENTITY，身份信息塞进 SOUL，改一个牵动另一个，git diff 历史没法看。
3. **团队不同步**：多人共享 workspace 时各改各的，合并冲突不断，也没人说得清哪次改动是故意的。

## 做法

我把它当成"身份清单 + 变更记录"来管：

1. **只放身份事实**。Name、Creature、Vibe（一句话气质）、Emoji、Avatar，每项一行，全文件控制在 10 行以内。行为规则、语气红线一律不进这里，归 SOUL.md 和 AGENTS.md。
2. **Avatar 用相对路径**（如 `avatar.png` 放 workspace 根目录），换机、迁移不失效。
3. **git 管身份变更**。调整单独提交，commit message 写清动机，比如"更名：避免与备用实例混淆"。身份从此有 changelog。
4. **设回顾节奏**。只在版本升级或角色定位变化时改 IDENTITY；日常聊天里 agent 提的自我调整请求，先记下来，定期批量评审，避免被单次对话带偏。
5. **多实例命名加约束**。主实例、备用、测试实例用可区分的名字和 emoji，gateway 日志里一眼对上号。

## 踩坑点

- **写太长**。IDENTITY 每次会话启动都会进上下文，塞大段人设故事纯属浪费预算。细节想保留就放 SOUL 或 memory。
- **把"不能做什么"写进来**。那是行为约束，属于 AGENTS.md；混在一起后改规则会误伤身份。
- **改完不生效就反复测**。这些文件在会话启动时读取，改完要开新 session（或重启 gateway）再验证。
- **Emoji 选生僻字符**。部分终端和通知渠道渲染成方块，反而降低辨识度。
- **复制实例忘了改名**。出现两个同名同脸的 agent，日志排障极其痛苦。

## 可复用建议

- 把 IDENTITY.md 当 API 用：字段固定、内容极简、变更走 git。
- 一句话检验：这行内容如果换一个角色设定也成立，那它就不该在 IDENTITY 里。
- 团队协作时在 AGENTS.md 写明"IDENTITY 变更需 PR 评审"，比口头约定可靠。

## 总结

IDENTITY.md 的价值不在"给 AI 起名字"的仪式感，而在于给长期运行的 agent 一个稳定、可版本化、可审计的身份锚点。五行内容加一条 git 历史，换来的是多会话、多实例场景下实打实的确定性。让它保持小，让它可进化，就够用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/bca36e8d4a950a9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/22bfea695ce8669f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/1f37d87a6967a225.png)

