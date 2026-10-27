---
title: RSS + AI 摘要：搭一条自己的信息流管线，把“刷”变成“收”
feedId: 42732
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

RSS 没死，只是退到了工具层。大部分站点不再主动提供 feed，但 RSSHub 能补齐；自建 Miniflux 或 FreshRSS 也能把几十个源聚合到一处。真正的问题是：feed 只负责搬运，每天几百条未读，实际读的不到十分之一。我的目标是让 Agent 接管中间环节——定时拉取、过滤、摘要、打分，只把值得看的东西推过来。

## 问题

- 信息分散在几十个 feed，同一事件常有四五家重复报道；
- 很多 feed 只给摘要不给全文，LLM 拿不到上下文，摘出来全是空话；
- 全量塞给大模型烧 token，人工筛又回到老路；
- 摘要质量不可控，容易漏关键结论，甚至脑补。

## 做法

整条管线五个环节，代码就三个模块：`fetcher`（拉 feed + 抓正文）、`summarizer`（调 LLM）、`dispatcher`（推送），互相只通过 SQLite 表交接，各自独立可替换。

1. **源头聚合**：Miniflux 管订阅，没有 feed 的站走 RSSHub，统一输出标准 RSS/Atom。
2. **增量拉取与去重**：用 OpenClaw 的定时任务（或系统 crontab）每 30 分钟触发一次。以条目 GUID / 链接哈希为主键写入 SQLite，天然去重、重启不丢。
3. **正文补全**：feed 只有 summary 的条目再抓一次原文页，用 readability 类库抽正文；抓不到就降级用 feed 摘要，不让单条失败阻塞整批。
4. **LLM 摘要**：批处理，一次 5–10 条，用 JSON schema 强约束输出：`{title, one_line, key_points[], tags[], score}`。score 按自己定的相关性标准打分，低于阈值的直接丢弃。
5. **分发**：按分数和 tag 过滤后走 webhook 推到 IM，或者让 OpenClaw 汇总成一条每日 Markdown 日报存档。

## 踩坑

- **别信任 feed 的 content 字段**。CDATA、HTML 实体、转义混乱是常态，进 LLM 前必须清洗，否则 token 白烧。
- **抓正文要设超时和 UA**，不少站直接 403；失败一律降级，保证批处理整体推进。
- **结构化输出别只靠 prompt 求**。用 schema / function calling 硬约束，解析失败重试一次，再失败就标记人工兜底。
- **跨源去重**：同一新闻不同链接，主键去重不掉。对标题做分词后算 Jaccard 相似度，超阈值只留先到的。
- **成本分层**：先跑一轮规则粗筛（关键词 + 时间窗口），通过的才进大模型。实测 token 用量降了一个数量级。
- **摘要必须附原文链接**。模型会脑补，链接是最后的安全网。

## 可复用建议

- 状态全部落 SQLite：一张 `entries` 表（feed_id、guid、status、score），幂等、可回溯、好调试。
- prompt 里把“相关性定义”写死成具体标准（技术栈、项目方向、排除项），别指望模型自己懂你。这 30 分钟值得花。
- 先做每日一条汇总，稳定后再上实时推送，否则初期消息轰炸很容易把这套系统劝退。
- 顺手把 fetcher 包成 MCP 工具（`fetch_unread` / `mark_read`），Agent 就能在对话里按需查 feed，与定时管线互补。

## 总结

这条管线的本质是：**RSS 采集、规则粗筛、LLM 精读、Agent 分发**。代码两天能写完，真正难的是定义“什么值得看”——这个标准只能靠使用反馈迭代。先让 score 和 tag 每周调整一次，两周后信噪比会有肉眼可见的变化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/aeeda90a7d399bc3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/c15c437a26eecadf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/b50c6a3b3b06a0c4.png)

