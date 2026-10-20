---
title: memory_recall 实测：语义搜索 vs 全文搜索，别急着二选一
feedId: 42071
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 的 agent 跑久了，memory 里会积累大量历史条目：部署记录、踩过的坑、用户偏好、项目约定。`memory_recall` 是 agent 主动翻旧账的入口，它的检索后端常见两条路线：**语义搜索**（embedding + 向量相似度）和**全文搜索**（FTS/BM25 关键词匹配）。两种模式我在自己的实例上都跑过一段时间，记录一下实测差异。

## 问题

理想状态是"该想起来的时候能想起来"，但两种模式各有典型翻车场景：

- **语义搜索**：查询带精确 token 时容易偏。比如问"上次部署报错的 exit code"，整句被编码后，正确条目可能排到第 9 位开外——ID、报错码这类强信号被句意稀释了。
- **全文搜索**：换个说法就抓瞎。"之前那个缓存雪崩的事"搜不到，因为 memory 里写的是"Redis 大量 key 同时过期导致 DB 打满"。
- 中文场景还有一层额外问题，见踩坑点。

单选任何一种，召回率都明显不够。

## 做法

我的验证流程，可以直接复用：

1. **建评测集**：从历史会话里人工挑 60~100 条"当时确实需要 recall"的查询，标注每条应命中的 memory 条目。这步最费时间但最值。
2. **分别跑分**：同一批查询分别用 semantic 和 fts 模式跑，记录 recall@5。
3. **失败分类**：把 miss 的 case 分成"精确 token 未命中"和"换说法未命中"两类，基本能预判混合方案的收益。
4. **上混合检索**：两路都跑，用 RRF（倒数排名融合）合并，先不调权重，看裸效果。
5. **灰度切换**：保留 recall 调用日志，线上观察一两周再定稿。

配置层面大致是：

```yaml
memory_recall:
  mode: hybrid          # semantic | fts | hybrid
  top_k: 8              # 每路各取，融合后截断
  fusion: rrf
  fts_tokenizer: trigram
```

实测 hybrid 的 recall@5 比单路高出 15~20 个百分点，且失败 case 互补得很好：FTS 兜住精确 token，语义兜住换说法。

## 踩坑点

- **中文分词是全文搜索第一坑**。SQLite FTS5 默认 unicode61 会把连续中文当一整个 token，中文查询基本搜不到东西，换 trigram 或外接分词器后才正常。
- **embedding 模型的语言覆盖**。早期用过纯英文训练的模型，中文 query 向量区分度很差，换成多语言模型后明显改善。
- **chunk 别贪大**。一条 memory 塞多个主题，向量被稀释，FTS 命中后返回的上下文也啰嗦。一行一事最好。
- **top_k 太小等于没搜到**。正确条目排第 9 和不存在没区别，没上 rerank 之前宁可给大一点。
- **索引一致性**。memory 被编辑或删除后向量索引没同步，会召回过期内容，这类 bug 很隐蔽。

## 可复用建议

- 默认上 hybrid + RRF，这是投入产出比最高的起点，不引入调参负担。
- 检索质量的上限在写入侧：memory 写入时就结构化，短条目、带时间戳和项目标签。
- recall 的调用日志别丢，定期捞 miss case 进评测集，评测集会越用越准。
- 对"必须命中关键词"的场景（ID、错误码、命令），在 prompt 层面引导 agent 优先走 FTS 或直接精确匹配。

## 总结

语义搜索解决"换了说法也能找到"，全文搜索解决"关键词必须命中"。中文场景下两者各有硬伤：前者依赖模型的语言覆盖，后者依赖分词方案。我的结论是这不是二选一的问题——**hybrid 检索 + 一个 60 条的人工评测集**，是让 memory_recall 从"碰运气"变成"可度量"的最小工程量。先能度量，再谈优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/702b31205a43fb86.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c9bd89f8ef8c00ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/7d0c08b31fee9468.png)

