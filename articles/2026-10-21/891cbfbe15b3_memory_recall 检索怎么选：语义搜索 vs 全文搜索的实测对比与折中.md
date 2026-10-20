---
title: memory_recall 检索怎么选：语义搜索 vs 全文搜索的实测对比与折中方案
feedId: 42141
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 的 memory 机制很朴素：日常对话沉淀成 workspace 下 memory 目录里的 Markdown 片段，agent 需要时通过 memory_recall 把相关内容捞回上下文。捞得准，省 token 也省心智；捞不准，要么空手而归，要么把不相关的旧记忆塞进上下文，反而污染输出。检索层目前两条路线：SQLite FTS 全文检索，和基于 embeddings 的语义检索。

## 问题：两条路各有软肋

**全文搜索**是精确命中的王者——错误码、命令名、路径、变量名，一搜一个准。但有两个硬伤：一是中文分词，FTS5 默认 tokenizer 对连续中文基本不切词，整句查询经常 0 命中；二是同义改写，当时记的是“部署脚本超时”，三个月后搜“上线卡住”，就找不回来了。

**语义检索**把文本向量化后按相似度召回，改写、中英混杂、口语化描述都能兜住。代价是：查询要走一次 embedding（延迟+费用）；而且对“字面精确”的内容反而钝——搜一个错误码，它可能返回一堆“语义相近”但毫不相关的段落，比查不到还危险。

## 做法：双路并行，合并召回

1. 保留 FTS 作为一路召回，中文场景把 tokenizer 换成 trigram（或入库前先 jieba 切好再存）。
2. 开启 embeddings，选对中文友好的模型（OpenAI text-embedding-3-small 或本地 bge-m3 都行），memory 文件按段落 chunk 后建向量索引。
3. memory_recall 层同时跑两路，用 RRF（倒数排名融合）合并，各取 top 8 融合出最终 top 5。
4. 设相似度阈值：低于阈值宁可返回“没找到”，让 agent 老实说“我不记得”，也别硬塞。

## 踩坑点

- FTS 建表时忘了指定 trigram tokenizer，中文全文检索等于摆设，排查半天才发现是分词的事。
- 换 embedding 模型后没重建索引：维度对不上直接报错；维度碰巧一样更隐蔽——新旧向量混存，召回质量悄悄劣化。换模型必须全量重建。
- 整篇 memory 文件当一个 chunk，文本太长稀释相似度；按段落或 300~500 字切分效果好得多。
- memory 编辑后索引没刷新，agent 一直召回旧版本。让 memory 写入路径顺手触发增量更新。
- 语料规模小于两三百条时，语义检索的提升很有限，embedding 的延迟和费用反而是纯开销。

## 可复用建议

- 先看规模和查询形态再配权重：大量精确标识符 → FTS 权重调高；大量口语化、跨语言查询 → 语义权重调高。
- 用 20 条真实历史问题做回归集，每次调参后人工核对 recall@5，比拍脑袋调阈值靠谱。
- 真正重要的稳定事实（用户偏好、项目约定）直接写进 AGENTS.md 常驻上下文，别赌 recall 的命中率。
- top_k 控制在 3~5，上下文是稀缺资源，召回越多噪声越大。

## 总结

这不是二选一。全文检索负责“字面精确”，语义检索负责“意思对上”，两者融合加一个保守阈值，是我目前实测最稳的形态。memory_recall 的目标不是召回最多，而是召回刚好够用——宁可少而准，不要多而杂。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/0eb2e0b86e72372a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/c7912e6ab1a6af91.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/51bbefd21a4197aa.png)

