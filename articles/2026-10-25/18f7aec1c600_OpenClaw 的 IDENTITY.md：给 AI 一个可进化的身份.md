---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42557
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

OpenClaw 的工作区里有一组"人格层"文件：`AGENTS.md` 管行为规则，`SOUL.md` 管价值观和语气，`IDENTITY.md` 管"我是谁"——名字、物种设定、emoji、头像。每次会话启动，它们会被拼进 system prompt。多数人的问题是：初始化时填了一次，之后再没打开过。

## 问题：身份为什么会"漂"

跑久了会看到几类现象：

- 群里多个 bot 同时在场，各自自我介绍含糊，用户分不清谁是谁；
- 身份信息散落在 prompt、频道昵称、代码注释里，改一处忘三处；
- agent 在不同会话里对"我是谁"的自述前后不一致。

根因不在模型，而在身份没有被当作一个**有版本、有边界**的工件来管理。

## 做法：把身份当代码管

**1. 写一份最小可用的 IDENTITY.md**（`~/.openclaw/workspace/IDENTITY.md`）：

```markdown
# Identity

- Name: 阿钳
- Creature: 一只住在终端里的机械蟹
- Emoji: 🦀
- Avatar: avatars/crab.png

## 一句话自我介绍
（群聊被问"你是谁"时用）

## 边界
- 不冒充人类，被问是不是 AI 时如实回答
```

控制在 30 行以内——它每次会话都会进上下文，是有 token 成本的。

**2. 划清文件边界。** IDENTITY.md 只放"事实"：叫什么、是什么、长什么样。语气和价值观归 SOUL.md，任务流程归 AGENTS.md，用户信息归 USER.md。判断标准很简单：换一个任务，这句话还成立吗？成立才放这里。

**3. 让它可进化。** workspace 直接 `git init`，身份变更单独 commit（如 `identity: 增加自称约束`），文件末尾维护一个 Changelog 小节。想让身份随使用进化，可以引导 agent："根据最近一个月的对话，起草一份 IDENTITY.md 修改建议"——但合并权在人手里，agent 只出 diff 不落盘。这样每次变化可追溯、可回滚。

**4. 多 Agent 场景。** 每个 agent 用独立 workspace，身份天然隔离；另外检查各渠道的显示名、群昵称是否与 Name/Emoji 对齐，否则用户的感知身份和模型自述会打架。

## 踩坑点

- **改了不生效**：system prompt 在会话启动时组装，编辑后要新开会话，别在旧会话里反复问"你怎么不记得自己叫 X"。
- **把行为规则塞进来**：文件越长开销越大，且与 SOUL.md 重复后会互相冲突。
- **让 agent 直接写这个文件**：一句"我觉得我叫……更好"就是漂移的开始，写权限收紧。
- **头像和 emoji 不唯一**：多 bot 同群时，这是用户分辨"谁在说话"的主要线索。
- **写易变状态**：当前任务、临时偏好，那是 MEMORY.md 的事。

## 可复用建议

- 模板七段：Name / Creature / Emoji / Avatar / 一句话自我介绍 / 语言红线 / Changelog，30 行封顶。
- 身份变更像发版：小改直接 commit；大改（名字、形象）先告知用户。
- 每季度或大版本升级后 review 一次：这个身份还匹配它现在的能力吗？
- 团队场景把 workspace 纳入同一 repo，身份变更走 code review。

## 总结

IDENTITY.md 只有十几行，却是 agent 所有输出的人格锚点。当配置文件随手填，你会得到一个身份模糊、自述漂移的助手；当成有版本、有边界、有 review 流程的工件，身份就能随能力一起受控地进化。成本是一次 `git init`，收益是长期一致性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/c533d0a3a7e50757.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/7a7059fe821d0b25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/92973664c29c129c.png)

