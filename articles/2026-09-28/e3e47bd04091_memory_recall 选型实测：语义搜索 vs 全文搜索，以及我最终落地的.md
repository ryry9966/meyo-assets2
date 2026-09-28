---
title: memory_recall 选型实测：语义搜索 vs 全文搜索，以及我最终落地的混合方案
feedId: 39272
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的 `memory_recall` 负责在对话中把历史记忆捞回来。我的 agent 跑了半年，memory 目录积累了 400 多条 markdown：部署笔记、用户偏好、踩坑记录，还有不少 token、路径、错误码。默认走的是语义检索（embedding + 向量相似度），最近我做了一轮对照实验，把它和 SQLite FTS5 全文检索认真比了比。

## 问题

两个典型的失败场景：

1. **语义检索翻车**：查询「上次 webhook 报 40159 那次怎么处理的」，向量检索返回一堆泛泛的「错误处理」条目，真正包含 40159 的那条排在很靠后，没进 top-k。精确标识符（错误码、路径、命令参数）是 embedding 的弱项。
2. **全文检索翻车**：查询「部署到国内机器要注意什么」，记录里写的却是「内网穿透」「备案」，关键词完全对不上，一条都没召回。

结论很直白：**精确查询全文赢，换说法的模糊查询语义赢**。

## 做法

我最终把 recall 改成了混合检索，四步：

1. **全文一路**：memory 文件同步写入 SQLite FTS5 表。
2. **语义一路**：每条 memory chunk 后台算 embedding（bge-small-zh 这个量级足够），存 npy，查询时算余弦相似度取 top 20。
3. **融合**：两路各取 top 20，用 RRF（reciprocal rank fusion，k=60）合并排序，取 top 8 回填给 agent。公式就一行 `score = Σ 1/(k + rank_i)`，不要求两路分数可比，工程上最省事。
4. rerank 可选，但在几百条规模下收益不明显，先没加。

## 踩坑点

1. **中文分词**：FTS5 默认 `unicode61` tokenizer 会把整段连续中文当一个 token，中文几乎搜不到东西。解法是 trigram tokenizer（SQLite 3.34+ 自带）或外挂 jieba 分词后写入。trigram 省事，但索引体积大约大 3 倍，两字短词召回偏差。
2. **embedding 模型升级**：换模型必须全量重建向量库。维度不匹配不会报错，只会默默返回垃圾相似度。我把模型名写进向量库 meta 文件，启动时校验。
3. **幽灵条目**：memory 是持续编辑的，全文索引好增量更新，但删记忆时向量库经常被忘掉——我出现过记忆已删、仍被召回的情况。删操作必须两路都清。
4. **top_k 不是越大越好**：召回 20 条塞给 agent，注意力被稀释，回复质量反而下降。8 条上下比较稳。

## 可复用建议

- 记忆条目 **< 200 条**：先别上向量，FTS5 + 合适的 tokenizer 就够，维护成本几乎为零。
- 查询里大量出现「换说法」：上混合检索，语义一路负责泛化召回，全文一路负责锚定精确标识符。
- **务必建评测集**：从自己的聊天记录里抽 20~30 条真实 query，每次改参数跑一遍看 recall@8。凭感觉调参必翻车。
- **记 recall 日志**：把每次命中的条目和漏召回的 query 记下来，一周后回看，比任何 benchmark 都诚实。

## 总结

「哪个好」其实是伪问题，取决于查询形态：带精确 token 的查询，全文检索碾压；自然语言换说法的查询，语义检索碾压。两套实现加起来不到一天工作量，**混合 + RRF 是当前性价比最高的答案**。如果只能二选一且记忆量小，我会先选全文检索——中文 tokenizer 调好之后，它朴素、可解释、零外部依赖，出问题也好排查。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/f0183cb51afa90b8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/db99654358f13ee5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/badfa2e57c062b13.png)

