---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 38941
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw Agent 用久了，最烦的不是模型不够聪明，而是每次新会话都要重新交代一遍：用 pnpm 不用 npm、commit message 写中文、部署先过 staging。上下文窗口不是记忆，会话一关就归零。好在 OpenClaw 的工具调用链路足够灵活，给 Agent 加一层记忆并不难，难的是设计得能长期维护。

## 问题

三种常见错误做法：

1. **全塞进 system prompt**——偏好一多，核心指令被稀释，Agent 反而变笨；
2. **存聊天记录原文**——检索噪声大，分不清“长期偏好”和“一次性任务细节”；
3. **只写不删不更新**——三个月前的旧偏好和新偏好打架，没人知道听谁的。

记忆系统的核心不是“存”，而是写、读、更新、遗忘四条路径都要有。

## 做法

推荐三层结构：

- **L0 会话上下文**：对话本身，不用管；
- **L1 工作记忆**：任务级 scratchpad，短 TTL（比如 2 小时），存中间产物；
- **L2 长期偏好库**：结构化条目，SQLite 或本地 JSON 均可。

L2 条目建议长这样：

```json
{
  "user_id": "default",
  "category": "coding",
  "key": "package_manager",
  "value": "pnpm",
  "confidence": 0.9,
  "source": "explicit",
  "updated_at": "2025-05-01T10:00:00Z",
  "status": "active"
}
```

写入走两条触发路径：

1. **显式**：用户说“记住 xxx"，以 confidence=1.0 直接 upsert；
2. **静默提取**：会话结束跑一次轻量提取 pass，只对白名单 category（coding / workflow / communication）生效，confidence 低于 0.6 的丢弃。

读取不要全量注入。会话开始调一次 `memory.read`，按 category 过滤，取 top 10~15 条拼进 system prompt，够用。

更新以 `(user_id, category, key)` 为 upsert 键，新时间戳覆盖旧值；遗忘走软删除，status 置为 tombstone，防止下次提取又学回来。

## 踩坑点

- **静默提取太激进**：把“这次任务用 React”这种一次性细节也存了。必须配 category 白名单 + 阈值，宁漏勿错；
- **同一偏好换措辞存了五遍**：key 要归一化，upsert 优先于 insert；
- **软删除后又被学回来**：提取 pass 要显式排除 tombstone 条目；
- **偏好过期**：依赖 updated_at，超过 90 天且 confidence 低的条目，注入前先向用户确认一句。

## 可复用建议

- 把 memory 做成最小 MCP 工具集：`read` / `upsert` / `delete` 三个方法，百行以内能落地；
- 所有写入留审计日志，出问题可回放；
- 给用户留一个“列出我的偏好”命令。记忆必须可检视、可手工修正，否则没人敢信它；
- 先用 tag/category 过滤，别急着上 embedding，条目过几百条再考虑。

## 总结

Agent 记忆本质上是个数据库设计问题，不是模型问题。把写入触发、冲突更新、遗忘路径先设计好，注入只是最后一步。先做小、可审计、用户可修正，比什么花活都重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/06173a03facc3a41.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/da0bdf5db50db716.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/42fe5501ed877f79.png)

