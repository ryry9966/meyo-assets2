---
title: memory_recall 实测：语义搜索 vs 全文搜索，我最后选了双路融合
feedId: 39579
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 memory_recall 是我这套 agent 里调用频率最高的工具之一。笔记、对话结论、踩坑记录攒到五百多条之后，"全量塞进 context"这条路就废了，检索质量直接决定 agent 回答的下限。摆在面前的方案就是两个：embedding 语义搜索，和 SQLite FTS5 全文搜索。社区里常见的问题是把它们当成二选一，我一开始也这么想，结果两头都撞了墙。

## 问题：两条路各有一个死穴

**语义搜索**：同义改写确实能召回，"上次那个部署炸了的事"能命中记录里的 "deploy 502 gateway timeout"。但对精确标识符很不友好——错误码、主机名、路径、参数名，在向量空间里被稀释得很厉害。问"ERR_5203 那次怎么解的"，回来一堆主题相关的条目，就是没有那一条。

**全文搜索**：字面精确匹配是它天下，但 query 和记录只要换一种说法就全丢。而且中文还有个坑，后面细说。

结论很清楚：两类失败模式基本不重叠，单选哪个都有明显短板。

## 做法：双路召回 + RRF 融合

1. **存储层**：每条 memory 是一个原子事实，附 metadata（时间戳、来源、tags）。大段日记式记录对两种索引都不友好，先把它拆碎。
2. **双索引**：SQLite FTS5 挂 trigram tokenizer 解决中文切分；embedding 用本地跑的 bge-small-zh-v1.5（512 维，CPU 够用），向量单独存表。
3. **recall 流程**：query 同时打两路，各取 top-8，用 RRF（Reciprocal Rank Fusion，score = Σ 1/(60+rank)）融合排序，取 top-5 回填 context。
4. **路由加权**：query 里出现引号包住的精确串、错误码、路径模式时，FTS 权重乘 2。这是纯规则，十几行代码。
5. **时间衰减**：最终分乘一个 recency 因子，避免三个月前的旧条目永远霸榜。

## 踩坑点

- FTS5 默认的 unicode61 分词器会把一整句无空格中文当成**一个 token**，等于全文搜索彻底失效。换 trigram 才恢复正常，这个坑不难踩。
- embedding 模型对中英混杂 + 代码标识符很弱，`openclaw.gateway.dev` 这种串基本检索不到，必须有 FTS 兜底。
- top-k 别贪大。超过 5 条 memory 进 context，agent 开始混淆条目来源，回答质量反而下降。
- 语义召回的"高分不相关"很隐蔽：问"那次限流怎么改的"，召回十来条都在聊限流，但没有一条是那一次。不建评测集根本发现不了。
- 没有评测就是瞎调。我攒了 30 条真实 query 算 hit@3：纯语义 0.61，纯 FTS 0.58，混合后 0.83。数字不算惊艳，但方向明确了。

## 可复用建议

- **别二选一**。RRF 不需要训练、不依赖额外模型，实现成本极低，是目前性价比最高的融合方式。
- memory 条目**原子化是前提**，索引再好也救不了烂数据。
- query 特征路由（精确串 → FTS 加权）比统一权重好，成本几乎为零。
- 固定评测集 + hit@3，每次改动跑一遍，别靠体感调参。

## 总结

语义搜索解决"意思相近"，全文搜索解决"字面精确"，memory_recall 两个都需要。混合方案不是妥协，而是这两类检索的失败模式本来就不重叠。实操路径建议：先上 FTS5 + trigram 保住下限，再加向量补上限，最后用固定评测集验证收益。比一步到位上一个纯向量方案稳得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/eb5c489bc8d40d94.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/eec562f35a220d34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/2ceacee56f819321.png)

