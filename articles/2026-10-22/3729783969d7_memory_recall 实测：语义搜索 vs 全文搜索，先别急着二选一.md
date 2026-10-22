---
title: memory_recall 实测：语义搜索 vs 全文搜索，先别急着二选一
feedId: 42268
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

Agent 跑得久了，上下文窗口装不下全部历史，`memory_recall` 就成了关键路径。OpenClaw 里常见的模式是：把会话记录、笔记、决策切片存进本地库，模型需要时调 `memory_recall` 检索几条回来注入 prompt。检索质量直接决定 agent 是"记得住事"还是"一本正经地瞎编"。

## 问题

两路方案各自的翻车场景，我都在自己的实例里踩过：

**纯语义检索（embedding + 向量召回）：**
- 精确标识符召回差。错误码 `ERR_5021`、配置键 `memory.ttl`、某个 API 路径——embedding 模型对这类 token 不敏感，top-10 里经常没有它。
- 专有名词命中率低，尤其中文语境下的小模型。

**纯全文检索（BM25/FTS）：**
- 换个说法就找不到。"上次那个部署报错" 搜不到当时记录的 "deploy failed"。
- 口语化提问和书面记录之间的词汇鸿沟没法靠查询改写完全弥合。

两者失效场景几乎不重叠，这就是结论的来源。

## 做法

混合检索，两路召回 + 融合，一次 SQLite 搞定：

1. **存储**：一张表，FTS5 建全文索引，embedding 存 JSON 列或上 `sqlite-vec` 扩展，单文件够用。
2. **切片**：按语义单元切（一个决定、一个事件），不要按固定 token 数硬切。每片带 `timestamp`、`source`、`project` 元数据。
3. **检索**：BM25 一路 + 向量一路，各取 top-20，RRF 融合取 top-8，注入 prompt。
4. **评测**：攒 30–50 条真实提问的 golden set，人工标出"应该被召回的片段"，跑 recall@5。没有这一步，后面所有调参都是玄学。

## 踩坑点

- **FTS5 默认 tokenizer 不分中文**。`unicode61` 会把整段连续中文当一个 token，query 一进去检索直接失效。要么配 trigram，要么预处理时 jieba 分词后再入索引——这是我损失了两天排查时间的坑。
- 按时间窗口硬切，把一个决策切成两半，语义召回时两半都挤不进 top-k。
- Embedding 模型选型没测中文，英文模型遇到中文 query 掉召回非常严重，选型时用真实 query 测，别看榜单。
- cosine 分数跨模型不可比，别拿它做"相似度 > 0.8 才注入"之类的阈值。
- 忘了时间衰减，三个月前的旧方案老排第一，agent 反复复用已废弃的决策。

## 可复用建议

- **先上 hybrid，别一步到位堆 reranker**。golden set 里 recall@5 达标再加复杂度，多数个人级场景 RRF 就够。
- 元数据过滤（项目、时间范围）放在融合之后、注入之前做，成本最低，效果立竿见影。
- 注入时保留来源和时间戳，让模型自己判断信息新旧，比硬编码优先级稳。

## 总结

语义搜索解决"换了个说法"，全文搜索解决"精确匹配"，两者互补而不是互斥，所以混合检索不是优化项，是基本盘。实践中真正决定 recall 效果的是**切片粒度**和**评测集质量**，检索算法反而是最不敏感的一环。先把 golden set 建起来，再谈调优。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/723af2c998bf7bb8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/7be6aced030f5f69.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/e0cb955a099b3c96.png)

