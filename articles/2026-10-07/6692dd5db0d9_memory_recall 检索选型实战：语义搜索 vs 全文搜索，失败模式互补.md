---
title: memory_recall 检索选型实战：语义搜索 vs 全文搜索，失败模式互补才是关键
feedId: 40808
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的 `memory_recall` 是 agent 回忆历史上下文的主要入口。memory 刚起步、只有几十条时，随便什么检索都能命中；等积累到几千条笔记、对话摘要、项目事实之后，检索质量直接决定 agent 是“真的记得”，还是“装作记得”。

## 问题

我们最初用 SQLite FTS5 做全文检索，失败模式很典型：用户问“之前讨论过的那个部署超时的问题”，而 memory 里记的是“vite proxy timeout 排查”，全文一条都搜不到——换个说法就失忆。

换成语义检索（embedding + 余弦相似度）后，新问题出现了：查某条具体报错码当时怎么解的，语义检索返回一堆“部署相关”的泛化结果，精确错误码那条挤不进 top-5。

关键观察：**两种方案的失败模式几乎不重叠**。改述类 query 靠语义，精确 token（错误码、路径、版本号、人名）靠全文。这就是工程上的机会。

## 做法

1. **先量化 badcase**。把一周内 recall 失败的 query 存下来，分两类：改述类和精确 token 类。我们大概是 6:4。
2. **双路召回**。写入 memory 时同时建两份索引：FTS5（trigram tokenizer，解决中文分词）+ 向量索引（sqlite-vec + 多语 embedding 模型），同库存储，无外部服务依赖。
3. **融合排序**。两路各取 top-20，用 Reciprocal Rank Fusion 合并。不要直接比较分数——BM25 和余弦相似度量纲完全不同，加权归一化很容易调崩。
4. **固定评测集**。整理 30 条带标准答案的 recall 测试对，每次改检索参数跑一遍 hit@5，不靠体感。

## 踩坑点

- FTS5 默认 unicode61 分词器把中文整句当一个 token，**必须换 trigram 或外接分词**，否则中文全文检索形同虚设。
- chunk 粒度：按“一条 memory”为单位建向量，不要切太碎，否则“谁说的、哪个项目”这类上下文会丢。
- embedding 模型对错误码、哈希串基本不敏感——这正是必须保留全文通道的根本原因。
- 融合后 top_k 要放大：两路各 20 融合后取 8，比直接取 5 的召回高不少。
- 写入和索引要同步。遇到过 agent 刚写入 memory 立刻 recall 却搜不到，原因是向量索引异步重建有延迟。

## 可复用建议

- memory 在几千条以内，先做好全文检索（trigram + 合理 top_k），别急着上向量，大概率够用。
- 上了向量之后**不要删全文通道**，hybrid + RRF 是目前性价比最高的组合，核心逻辑不到两百行。
- 任何检索改动先过评测集再上线；评测集会随 badcase 积累越来越有价值。
- 可以让 agent 在 recall 前先做一次 query 改写（口语转书面表达），对改述类失败有立竿见影的提升。

## 总结

语义搜索和全文搜索不是二选一，它们覆盖的失败模式互补。真正该做的是统计自己 memory 的 badcase 分布，据此决定两路的取舍和融合方式。对我们 6:4 的 query 分布来说，最终收敛到 hybrid + RRF，实现简单，稳定运行至今。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/1c11f466b77691dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/0cff822de82f28d0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/b806c5217ddd95f2.png)

