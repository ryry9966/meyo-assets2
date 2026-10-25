---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好，而不是把一切塞进 prompt
feedId: 42560
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

LLM 本身没有状态，每轮对话都是"失忆"重启。对个人助理类 Agent 来说，用户偏好是重复度最高的信息：包管理器用 pnpm、commit message 不加 emoji、时区是 UTC+8、注释只要中文。这些事每次靠人肉重述，体验很差。OpenClaw 生态里已经有 memory 类 MCP server 和插件方案，但常见用法是"把对话整体存进向量库"，实际效果往往不如预期。

## 问题

我踩过三条弯路：

1. **全量塞 system prompt**：几轮下来 prompt 膨胀，token 成本上涨，旧偏好和新偏好开始互相冲突。
2. **向量库当垃圾桶**：什么正文都入库，检索时噪音淹没真正有用的偏好，top-k 里一半是闲聊。
3. **只做会话摘要**：摘要粒度太粗，"用户偏好 Rust 缩进风格"这种原子事实被稀释掉。

本质问题：记忆系统的难点不在"存"，而在**检索精度**和**更新语义**。

## 做法

我用的是分层结构 + 双通道检索，四步：

**1. 定义 schema。** 每条记忆是结构化记录：`{type, content, scope, confidence, source, created_at, last_used}`。type 分三类：preference（偏好）、fact（事实）、procedure（操作习惯）。scope 区分 global 和 project，避免"这个项目用 npm"污染全局。

**2. 写入走两条路。** 显式信号优先：用户说"以后都用 pnpm""别加 emoji"，这类带"以后/总是/别"的句子触发即时抽取。兜底是会话结束时用廉价模型批量抽取，只保留原子化、可执行的事实，宁缺毋滥。

**3. 检索分两档。** 核心档案（约 10 条高置信偏好）无条件注入 system prompt；情景记忆走向量召回，阈值卡严，宁可不注入也别注入错的。每次注入都记日志，出问题能回溯"模型当时看到了什么"。

**4. 维护比写入重要。** 同 key 去重，新记录覆盖旧记录；按 `last_used` 衰减，30 天未用且低置信的自动降权；用户明确修正时（"我说过了，改用 Bun"）立即提权并归档旧条目。

## 踩坑点

- **否定句是向量检索的天敌。** "不要用 Docker"和 Docker 相关内容相似度极高。偏好必须落成结构化字段（如 `prefer: false`），不能只靠 embedding。
- **偏好变更是常态。** 用户换工具、换风格很正常，别把旧偏好当 bug 修，要设计覆盖机制。
- **每轮都跑抽取会写一堆垃圾。** "帮我看看这段代码"不是记忆。先用规则过滤，别让 LLM 自由发挥。
- **隐私别漏。** 抽取管线见过 API key 和密钥路径，入库前必须过滤敏感信息，记忆默认本地存储。
- **注入不可见就不可信。** 一定要有 debug 视图看当前注入了哪些记忆，否则召回坏了你都不知道。

## 可复用建议

- 起步别上向量库。本地 JSONL + 关键词匹配能覆盖 80% 场景，先跑通再演进。
- 通用的 MCP memory server 多是简单 key-value，schema 约束和冲突处理要自己在外层实现，别指望它代劳。
- 记忆要用户可见可删。提供 list/delete 接口或一条 `/memory` 命令，信任感来自可控。
- 建一个 20 条左右的 golden set：改写过的偏好问句 + 期望召回的记忆，每次改检索逻辑跑一遍，防回归。
- 抽取用便宜模型、检索用 embedding、注入用模板，三件事分开，别指望一个 prompt 全搞定。

## 总结

Agent 记忆更像一个数据工程问题：schema 设计决定上限，检索精度决定体验，遗忘机制决定长期可用性。先做小而准的核心档案，再做召回，最后才是规模。存得多，不等于记得住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/f3cb43f0af2dcd8f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/31ee242cc2318b8e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/f66f85d37b1b3ea4.png)

