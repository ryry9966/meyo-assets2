---
title: memory_recall 实测：语义搜索 vs 全文搜索，先别急着二选一
feedId: 41918
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

Agent 用久了，workspace 的 memory 目录会积累大量 markdown：每日记录、排查结论、配置快照、踩坑笔记。`memory_recall` 的职责就是在对话中把这些内容精准捞回上下文。捞得准，agent 像个记性好的人；捞不准，要么答非所问，要么把几千 token 的无关记忆塞进去，白白污染上下文。

## 问题

社区里常见的争论是"要不要上向量检索"。我自己的观察是：两种检索的**失败模式完全不同**。全文检索输在"换个说法就找不到"，语义检索输在"专有名词、错误码对不上"。与其猜，不如在自己数据上跑一轮。

## 做法

- 环境：我的 workspace 约 420 个 memory 文件，按标题层级切成 3700 个 chunk（每块 200–500 token）
- 方案 A：SQLite FTS5，trigram 分词，BM25 排序
- 方案 B：bge-small-zh-v1.5 本地 embedding，sqlite-vec 存向量，余弦相似度
- 方案 C：混合。两路各取 top-20，RRF（k=60）融合，去重后取 top-5
- 评测集：从近三个月会话日志挑了 32 条真实 recall 查询，人工标注相关 chunk，看 hit@5 和延迟

| 查询类型 | A（FTS） | B（语义） | C（混合） |
|---|---|---|---|
| 含错误码/文件名/配置项（11 条） | 10 | 6 | 11 |
| 换说法的口语化查询（21 条） | 9 | 18 | 19 |
| 单次查询延迟 | <10ms | ~60ms | ~70ms |

样本很小，别当 benchmark 看，但方向清晰：**不是谁取代谁，是失败模式互补**。

## 接入步骤

1. chunk 按 markdown 标题层级切，不要按固定字数硬切，否则结论段和背景段会被拆散
2. FTS5 建 trigram 表
3. 向量用本地小模型起步即可，sqlite-vec 完全够用，不必上独立向量数据库
4. `memory_recall` 内部双路召回 → RRF 合并 → 按"相关分 × 时间衰减"排序
5. embedding 服务不可用时自动降级为纯 FTS，别让 recall 整体挂掉

## 踩坑点

- **中文 + FTS5**：默认 unicode61 分词器对中文基本不可用，必须换 trigram，或先用 jieba 分词再入库。这是我踩的第一个坑，排查了半天"为什么什么都搜不到"
- **换 embedding 模型必须全量重建向量索引**，把模型名写进 chunk 元数据，防止新旧向量混存
- 文件改了忘重嵌入，会搜出"旧版内容"。用 mtime + 内容 hash 做脏标记，变更时只重嵌入脏块
- 语义检索纯 top-k 不设分数阈值，会返回"看着相似其实无关"的 chunk，我设了 0.35 的余弦下限
- 记忆有时效性。上周的部署方式可能已被推翻，时间衰减权重必须有，否则陈旧结论反复回魂

## 可复用建议

- 先攒 20–30 条**自己真实使用**的 recall 查询做评测集，比看任何公开榜单都有说服力
- 全文检索永远保留：它是对精确标识符的兜底，也是语义服务故障时的降级路径
- recall 管道要可观测：query、命中、分数落日志，每周抽样人工过一遍
- memory 量级在万级 chunk 以下时，SQLite 一套搞定，别为检索精度引入重型组件

## 总结

语义搜索和全文搜索不是竞品，是两条腿。OpenClaw 的 `memory_recall` 想稳定，混合召回 + 严格的 chunk 纪律 + 一份小型自有评测集，比争论"哪个更好"划算得多。我的下一步是给混合链路加 rerank，等有结论再发一篇。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/6423b214ba09484c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/5b60c87564071b00.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/5d1b170961105c66.png)

