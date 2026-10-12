---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 41342
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

Agent 本质上是无状态的：每次会话开始，模型对你是“陌生”的。你在 OpenClaw 里反复强调的偏好——提交信息格式、部署习惯、“别自动重启服务”——第二天就清零。上下文窗口再大，也不可能塞进全部历史对话，何况大部分历史是噪音。记忆系统的目标不是“记住一切”，而是把少数高价值事实沉淀下来，在合适的时机取回来。

## 问题

多数人的第一版实现很粗暴：把对话原文丢进向量库，检索时取 top-k 塞进 system prompt。跑起来后常见三类故障：

1. **噪音记忆**：寒暄、过程性对话也被存进去，检索时挤掉真正有用的条目；
2. **记忆冲突**：三月份存了“用 pnpm”，五月改成 npm，两条都在库里，模型随机听一条；
3. **不可见不可控**：用户不知道记了什么、怎么删，信任感直接归零。

## 做法

我的方案是 memory-as-tool：写一个轻量 MCP server（或插件），暴露四个工具 `memory_save / memory_search / memory_list / memory_delete`，存储用一张 SQLite 表即可：

`id、namespace（按用户/agent 隔离）、type（preference / fact / procedure）、content、confidence、updated_at`

**写入侧做减法：**

- 显式写入：用户说“记住……”时触发，content 必须压缩成一句原子化陈述，一次只存一个事实；
- 隐式写入：任务结束后让模型自查“这次是否确认了长期偏好”，候选先进 staging 区，下次被用户行为印证后再转正。宁缺毋滥。

**读取侧做混合检索：**

- embedding 相似度 + 关键词过滤，按 namespace 隔离，取 top-3~5 就够；
- 注入时统一改写成“用户偏好：……”的短句，放在 system prompt 末尾，严控总 token。

**更新与遗忘：**

- 写入前先查重，语义重复则覆盖旧条目并刷新时间戳；冲突时以新条目为准——这天然解决“用户改变主意”的问题；
- preference 类型设 90 天 TTL，到期降权，避免陈年偏好绑架现在的行为。

## 踩坑点

- **别存对话原文**。存“结论级”的一句话，原文留在会话里；
- **别全量注入**。实际用下来，超过 10 条记忆后模型会明显开始执行早已过期的偏好；
- 纯向量检索会召回“字面相似、语义无关”的条目，必须叠一层关键词或 type 过滤；
- 多 agent 场景忘记 namespace 隔离，A 助手的偏好会污染 B 助手；
- 敏感信息（token、密钥）入库前要有拦截规则，别等出事再补。

## 可复用建议

- 第一版就用 SQLite + 一张表，别急着上专用向量库；
- 每条记忆：一个事实、一句话、带时间戳；
- `memory_list` / `memory_delete` 务必做成用户可触达的命令，让用户能看、能改、能删；
- 写入设三级置信度：显式请求 > 用户行为印证 > 模型猜测，分别对应直接入库、转正、待验证。

## 总结

记忆系统的难点不在模型而在工程：定义清楚存什么、怎么写、怎么取、怎么忘。让记忆像普通数据一样可审计、可清理，用户才敢真的让它“记住”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/4eb4eff002fce3b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/c7a6c434b1e5c2aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/b6b4f4b2a9aa5945.png)

