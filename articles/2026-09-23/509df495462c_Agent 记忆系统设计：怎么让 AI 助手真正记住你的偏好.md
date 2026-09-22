---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 38531
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

跑一个长期使用的 OpenClaw 助手，最直观的落差不是模型不够聪明，而是每次会话结束它就"失忆"：昨天刚说过回复要短、代码示例用 TypeScript、别在深夜推通知，今天又得从头交代一遍。RAG 解决的是"知识从哪来"，解决不了"用户是谁、怎么相处"。偏好记忆是另一个问题，值得单独设计。

## 问题

把记忆拆开看其实是三类：

- **会话内上下文**：框架已经处理；
- **稳定事实**：时区、技术栈、项目结构；
- **行为偏好**：回复详略、语言、格式、工具习惯、禁区。

前两类好办，第三类难在四件事：什么时候提取、怎么去重合并、冲突听谁的、什么时候过期。大多数"助手很笨"的体感，都出在这四件事没做。

## 做法

我的方案是先不上向量库，一张 SQLite 表起步：

```sql
memory(id, type, key, value, source, confidence, hit_count, last_used, created_at)
```

三条采集路径：

1. **显式写入**。通过 MCP 暴露 `memory_write` 工具，用户说"记住……"或 `/remember` 时由 agent 调用，`source=explicit`，置信度直接给满。
2. **隐式抽取**。会话结束后跑后台任务，让模型从对话中提取候选偏好，先进 staging 表，置信度过阈值或经用户确认后才转正。
3. **修正即写入**。用户纠正助手时（"我说过别用 emoji"），这条记忆权重最高，并覆盖旧的。

注入策略：会话开始时把记忆压成每条一行的卡片，按 `recency × frequency × confidence` 排序，硬上限 500 token 注入 system prompt。事实类常驻，偏好类按场景检索。

## 踩坑点

- **上来就上向量库**。偏好召回本质是 key-value 查询，embedding 反而引入噪声，先 SQL 后检索。
- **隐式抽取太激进**。模型会把一次性上下文当永久偏好（"这个项目用 Python"≠"用户永远想聊 Python"），所以必须有 source 标记和确认环节。
- **全量注入**。旧偏好互相矛盾时，输出质量明显下降，限量分层是底线。
- **没有失效机制**。对 `last_used` 做衰减，定期跑整理任务归档低频条目，别让半年前的偏好一直顶着。
- **忘了"被遗忘权"**。一定给用户 `memory_list` / `memory_forget` 入口。记忆不可见、不可删，信任会先崩。

## 可复用建议

- **Schema 先行**：`type / key / value / source / confidence / hit_count` 六个字段能覆盖九成场景。
- **用 MCP server 封装**成 `memory_get / write / list / forget` 四个工具，任何 agent、任何会话都能复用，别把记忆逻辑散在业务代码里。
- **每条记忆必须可反驳**：带来源、带时间，一条命令能修正。
- **准备回放集**：固定 20 条左右的问题，每次改注入策略后跑一遍对比命中率，避免拍脑袋调参。
- **留在本地**：自托管本来就是 OpenClaw 的前提，记忆库别顺手同步到第三方。

## 总结

记忆系统不是加个功能，而是一个小型数据工程问题：定好 schema，管好生命周期（写入 → 去重 → 冲突 → 衰减 → 遗忘），控制注入预算，留一个用户修正的闭环。从 SQLite + 显式写入 + 限量注入起步，一周内能跑通；隐式抽取等显式路径稳定了再加。先用起来，再谈聪明。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/4e8b016055e101bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/4f68a556b903aa13.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/475bd2b961c7e886.png)

