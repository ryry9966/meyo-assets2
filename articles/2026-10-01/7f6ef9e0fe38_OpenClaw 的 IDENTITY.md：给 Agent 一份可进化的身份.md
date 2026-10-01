---
title: OpenClaw 的 IDENTITY.md：给 Agent 一份可进化的身份
feedId: 40014
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的 workspace 里有几个约定文件：AGENTS.md 管操作规范，SOUL.md 管行为倾向，USER.md 管用户事实，而 IDENTITY.md 管最基础的一层——"我是谁"。它在每次会话启动时被注入上下文，是 agent 最先读到的一份自我描述。

很多人跑通框架后把精力放在工具接入和自动化上，IDENTITY.md 留空或只写一个名字。这没问题，但等于放弃了框架里成本最低、杠杆最大的一个配置点。

## 问题

没有身份文件的 agent 有两个常见症状：

1. **跨会话人格漂移**：今天冷峻、明天自来熟，因为模型每次都从零猜你的偏好，依据只是最近几句聊天。
2. **临时 prompt 越堆越多**：为了纠正语气，每条消息里加"正式一点""别用 emoji"，这些补丁只活一次会话，下个 session 又要重来。

本质是在用对话内容，做本该由配置文件做的事。

## 做法

IDENTITY.md 建议控制在 15 行以内，因为它每个会话都占上下文。一个够用的模板：

```markdown
# IDENTITY
- Name: Atlas
- Creature: 一只务实的工程章鱼
- Emoji: 🐙
- Vibe: 简洁、直接、先结论后依据
- Boundary: 不确定就说不确定，不编造工具输出
```

1. 在 workspace 根目录创建文件，先只填 Name 和 Vibe 两项，开几个新会话观察差异。
2. 纳入 git，身份改动走 commit，出问题可 diff、可回滚。
3. 觉得"它最近说话不对劲"时，不要在聊天里纠偏——改文件、开新会话验证。身份在会话启动时才加载，改完不重开等于没改。
4. 多 agent 场景给每个 agent 独立 workspace、各持一份，避免角色互相污染。

## 踩坑点

- **和 SOUL.md 抢地盘**：IDENTITY 管"我是谁"（名字、原型、语气基调），SOUL 管"我怎么做"（价值观、红线）。两边内容重复会导致指令冲突，行为反而更不稳定。
- **写太长**：把完整人设、语气示例全塞进去，每个会话白烧上千 token，还稀释了真正重要的指令。
- **身份过强**：persona 写得太戏剧化，agent 跑 cron 任务、调 MCP 工具时也在"演戏"，输出夹带角色扮演内容，影响下游解析。工作型 agent 的身份要克制。
- **放错内容**：用户偏好、项目事实、密钥别写这里，那是 USER.md 和 MEMORY.md 的职责。

## 可复用建议

- 把身份当"配置即代码"：小步提交，commit message 写清楚为什么改。
- 用分支做 A/B：两个 branch 各一套身份，在真实任务里对比择优。
- 每季度审一次；身份应该靠小 diff 缓慢进化，而不是某天情绪化重写。
- 团队场景把模板收进 onboarding 文档，新人改两个字段就能对齐风格。

## 总结

IDENTITY.md 是 OpenClaw 里最不起眼也最划算的文件：十几行、每个会话生效、git 可追溯。它的价值不在"让 agent 更像人"，而在于把散落在聊天记录里的调教成本，收敛成一份可版本化、可回滚、可演进的配置。身份不是写一次就完的，它是跟着你的使用习惯，慢慢 diff 出来的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/3308fd4769782df8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/5e70ce532d98046d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/5ecf35b0ebcea052.png)

