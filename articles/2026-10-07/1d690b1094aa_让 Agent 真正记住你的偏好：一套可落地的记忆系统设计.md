---
title: 让 Agent 真正记住你的偏好：一套可落地的记忆系统设计
feedId: 40721
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

每个做 Agent 的人都遇到过：新会话一开始，助手是"失忆"的。用户上周明确说过"构建用 pnpm"，这周它照样生成 package-lock.json。上下文窗口再大，跨会话状态也不会自动延续。记忆系统的职责，就是把用户偏好、项目约定这类信息持久化，并在合适的时机喂回给模型。

## 问题：三种典型失败模式

1. **全量塞上下文**：把历史对话原样存下来、检索时全量注入，prompt 迅速膨胀，信噪比暴跌，模型反而抓不住重点。
2. **只存不更新**：三个月前的偏好和今天冲突时，旧条目还在，agent 基于过时信息行动。
3. **记得太泛**："用户喜欢简洁"这种条目指导不了任何具体行为，等于没记。

## 做法：读写分离的三层结构

**存储层**：结构化条目，而非自由文本。每条记忆固定 schema：

```json
{
  "scope": "global | project:xxx | session",
  "kind": "preference | fact | convention",
  "content": "构建工具用 pnpm",
  "source": "explicit | inferred",
  "confidence": 0.9,
  "updated_at": "2024-06-01"
}
```

**写入路径**：显式偏好（用户说"以后都……"）直接高置信度写入；隐式模式（连续三次拒绝某类输出）由异步任务用 LLM 归纳成候选条目，标记 `inferred`、低置信度。写入前做相似度去重，命中已有条目就更新 confidence 和时间戳，而不是新增。

**读取路径**：会话开始时按当前 scope（全局 + 当前项目）检索，标签精确匹配 + embedding 相似度排序，只取 top-N，预算几百 token。低置信度条目以"参考"语气注入，不写成硬规则。

**接入方式**：用 MCP 把 `memory_search / memory_write / memory_update` 暴露成工具，agent 可在会话中主动查询，任何 MCP client 都能复用同一份记忆。

## 踩坑点

- **一次性偏好污染**：用户在某个项目里说"这里用 tabs"，写进 global scope 就会污染所有项目。scope 定错比不记还糟。
- **inferred 条目不设门槛**：归纳错一次，agent 会反复"贴心地"做错事。低置信度条目要么不注入，要么标注来源让用户可纠正。
- **裸 dump 注入**：检索结果原样拼进 system prompt，模型常把示例当指令。包一层"以下是用户历史偏好，供参考"的角色说明，效果好得多。
- **缺乏用户可见性**：用户不知道记了什么、也删不掉，信任会崩。至少提供列表和删除入口。

## 可复用建议

- 先定 schema 再写代码，`scope / source / confidence` 三个字段事后补救成本最高。
- 衰减机制从简：记录 `last_used_at`，N 天未命中就降权，比复杂的遗忘曲线够用。
- 能从项目文件（.editorconfig、lockfile）推断的约定，直接读文件，别进记忆。
- 冲突不自动覆盖：与高置信度旧条目矛盾时，先在会话里向用户确认。

## 总结

记忆系统的核心不是"存得多"，而是"取得准、改得动、看得见"：结构化 schema 控质量，读写分离控成本，scope 与置信度控污染，用户可见性控信任。先跑通最小闭环（显式偏好 + 按 scope 检索注入），再迭代隐式归纳，在现有 MCP 架构上两三天就能落地。欢迎在评论区贴出你们自己的 schema 设计，一起对齐。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/88741df3a856a480.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/6662af94d8bbd82c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/fbd8c119da2d0787.png)

