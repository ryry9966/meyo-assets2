---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 39946
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

用 agent 干活的人基本都遇到过同一个场景：每次新会话，都要重新交代一遍“用 pnpm、别写多余注释、commit message 用英文”。模型本身无状态，上下文窗口再大也只是“这次记得”。想做一个能长期共事的助手，记忆层是绕不开的一块。

## 问题

三种常见做法，实测都不够用：

1. **全塞进 system prompt。**最直观，但条目越多 token 越贵，旧记忆还会互相干扰。
2. **全部向量化再检索注入。**召回噪声大，经常把不相关的历史偏好带进当前任务。
3. **只做会话摘要。**丢失细节，“以后都用英文写 commit”这种点状偏好会被摘要稀释。

本质是四个问题：写什么、什么时候写、什么时候读、什么时候忘。这更像一个小型数据工程问题，而不是模型问题。

## 我的做法

我在自己的 agent 里通过 MCP 暴露了四个工具：`memory_write` / `memory_search` / `memory_forget` / `memory_list`，底层是本地 SQLite + FTS5，向量检索是跑稳之后才加的可选项。

**1. 分层。**记忆分三类：用户偏好（稳定，如工具链、语言习惯）、项目事实（半稳定，如接口约定）、过程记录（会话级，可丢弃）。只存结构化条目，不存自由文本长段落。

**2. 写入靠触发，不靠全量。**默认不写。命中“以后/记住/别再”这类显式信号，或用户纠正了 agent 输出时，才调用 `memory_write`，并带上 scope、confidence、source 字段：

```json
{
  "key": "commit_language",
  "value": "english",
  "scope": "project:openclaw-tools",
  "source": "user_correction",
  "confidence": 0.9
}
```

**3. 读取按需。**会话开始注入 top-K 高频偏好（有 token 预算）；任务中再按 scope + 关键词检索，scope 过滤优先于语义相似度。

**4. 更新与遗忘。**同 key 新条目直接覆盖旧条目（supersede），定期跑一次合并清理近重复；过程记录带 TTL，到期即删。

## 踩坑点

- **分不清“这次”和“以后”。**早期我记过“这次用中文回”，结果它永久生效。scope 判定必须做，拿不准就先向用户确认一句。
- **冲突条目并存。**“用 pnpm”和“改用 npm”同时被检索出来，模型随机挑一条，行为不可复现。必须有覆盖逻辑。
- **注入撑爆 prompt。**给记忆注入设硬性 token 上限，按“最近命中频率 × 新鲜度”排序，排不进的宁可不注入。
- **隐私。**记忆文件是明文，我在写入前过滤了密钥、地址类敏感字段，并把 list / forget 暴露给用户——能看、能删。

## 可复用建议

- 先做规则和显式写入，跑稳了再上向量检索，顺序别反。
- 项目级记忆直接用 JSON/Markdown 放进仓库，可 diff、可 review，比黑盒数据库省心。
- 给 agent 暴露 list / edit / forget，人始终握有修改权，这是信任的基础。
- 攒 10~20 条“该记/不该记”的回归用例，每次改抽取 prompt 后跑一遍，否则记忆质量全靠玄学。

## 总结

让助手“记住你”，难点不在模型，而在写入时机、条目结构和遗忘策略这三件朴素的事。小而结构化、人可编辑的记忆，比一个大而全的向量库好用得多。建议从四五个工具、一张 SQLite 表开始，实际跑两周，再谈优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/920cf9e51fd14a0e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/fbfd38c6715c57d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/9da4b4e366ab4ca7.png)

