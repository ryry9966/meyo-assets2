---
title: memory_recall 实战对比：语义搜索 vs 全文搜索，别急着二选一
feedId: 41637
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

OpenClaw 的 memory_recall 是 agent 长期记忆的入口：助手把事实写进 workspace 的 memory 文件，下次会话靠召回把相关片段塞回上下文。召回质量直接决定"它记不记得你说过什么"的体感。最近我把一个跑了两个月的个人助手从纯语义召回改成混合召回，把对比结论和踩过的坑整理一下。

## 问题：两条路各有死穴

最初实现很简单：记忆写入时算 embedding，recall 时拿查询向量做余弦相似度取 top-k。跑了一阵发现两类高频失败：

1. **精确标识符召回不到**。报错码、服务名、昵称、API 路径这类词，embedding 模型经常把它和语义相近但实体不同的内容混在一起，top10 里找不到目标。
2. **措辞一变就漏**。"上周说的那个部署脚本" vs 记忆里写的"发布流水线"，语义距离不够近，照样漏。

反过来也试过纯全文搜索（SQLite FTS5 + BM25）：精确词命中很稳，但中文分词一塌糊涂，同义改写基本召回为零。

结论先行：**单通道都不够，混合才是工程上的正解。**

## 做法：双通道 + RRF 融合

写入侧：
- 记忆按条切分，每条 200~400 字，带 metadata（时间、来源、类型：fact/preference/event）
- 原文存 SQLite，同时建 FTS5 索引和 embedding 列（本地 bge-small-zh，单条 <20ms）

召回侧核心逻辑：

```python
hits_kw  = fts5_search(q, limit=20)
hits_vec = vector_search(embed(q), limit=20)
merged   = rrf_fuse(hits_kw, hits_vec, k=60)  # 1/(60+rank) 求和
top      = merged[:8]
```

- 用 RRF 而不是分数加权：BM25 分数和余弦分数不可比，RRF 只看排名，省去调权重的麻烦
- metadata 过滤放在召回前：类型、时间窗先卡掉，减少噪声
- 结果带来源文件路径返回给 agent，方便它引用或主动要求"再搜一轮"

## 踩坑点

1. **FTS5 默认 unicode61 分词器不切中文**，整句变成一个 token，搜索形同虚设。必须先 jieba 预分词再写入，或换 trigram tokenizer 兜底。这是中文场景最大的坑。
2. **embedding 模型换版本 = 全量重建索引**，维度相同也不行，向量空间不同。索引里存个 model_version 字段。
3. **记忆不设过期会越积越脏**。半年前的"用户在用 X 框架"会持续污染召回。写入打时间戳，event 类记忆做时间衰减。
4. k=60 不是玄学也不是最优，只是论文默认值。实测 20~100 区间体感差异不大，别过度调参。
5. **重复记忆会霸榜**。写入前对相似度 >0.95 的旧条目做更新而非新增。

## 可复用建议

- 默认上混合，别争"哪个好"：运维、代码类精确词多的场景全文通道自然强，偏好闲聊类语义更稳，RRF 会自己表达出来
- 建一个 30~50 条的召回评测集（query → 期望命中的记忆 id），改任何参数前后跑一遍，比体感靠谱
- 保底策略：两路都空手时回落到最近 N 条记忆，避免完全失忆
- embedding 用本地小模型就够，记忆检索不需要顶级模型，延迟和隐私更划算

## 总结

语义搜索解决"换个说法也能找到"，全文搜索解决"一个字都不能错"，agent 记忆两者都需要。值得花一个下午把 memory_recall 改成双通道 + RRF，收益是实打实的召回率提升，成本只是每条记忆多一次 embedding 和一个 FTS 索引。先跑评测集，再谈优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/10c4d7ea7781741a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/567ee3407f811744.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/76507743dd33109c.png)

