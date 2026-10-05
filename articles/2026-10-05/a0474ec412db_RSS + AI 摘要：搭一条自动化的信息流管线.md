---
title: RSS + AI 摘要：搭一条自动化的信息流管线
feedId: 40591
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

RSS 没有死。对我这类要跟踪十几个技术博客、版本发布页和论文源的人来说，它仍是最稳定的订阅协议：无算法、无登录、结构化。问题在于信息量本身——订阅源一多，未读数就是灾难。

## 问题

RSS 解决的是"聚合"，不解决"筛选"。痛点很具体：

1. 大部分条目不值得点开，但每条都要扫一眼标题做判断；
2. 同一条消息会被三四个源重复推；
3. 只有 summary 的源看不到正文，判断不了质量。

目标是让 Agent 替我完成"扫一遍、去重、挑重要的、给摘要"，我只看产出。

## 做法

管线拆成四步，全部跑在 OpenClaw 的定时任务里：

**1. 抓取。** cron 触发，拉取 OPML 里的全部源。用条件 GET（ETag / Last-Modified），没更新的源直接跳过，省流量也省无意义的调用。

**2. 去重落库。** 每条目对 link 归一化（去 utm_* 参数）后和 title 一起算 hash，SQLite 查重。新条目写入 items 表，带 source、pubDate、status 字段。

**3. 摘要与打分。** 两层模型：先用便宜模型过滤，输出"值得看 / 跳过 + 一句理由"；通过的条目再用强模型产出 2-3 句摘要。通过 MCP 工具调用模型，prompt 明确要求"只基于正文，不补充外部信息"。

**4. 推送。** 按分数排序，Top N 汇总成一条 digest 推到 Telegram。低分不推，只留库可查。

整个管线约 150 行脚本 + 一个 OpenClaw cron 任务，状态全在 sqlite 里，挂了重跑即可。

## 踩坑点

- **summary-only 的源是重灾区。** 很多 feed 只有前两句话，摘要模型等于在猜。接一个 readability 抽正文；正文缺失的条目直接标记"需人工"，别硬生成。
- **去重不能只靠 link。** 同一篇文章在不同源 URL 带不同 tracking 参数，归一化后再 hash。
- **pubDate 不可信。** 有源会把所有条目刷成当前时间，导致天天重复。hash 去重兜底，别信时间戳。
- **幻觉。** 模型拿不到正文时会编得有模有样。约束 prompt 之外，再禁掉正文里不存在的具体数字和名词，抽查两周后基本稳定。
- **成本。** 全量喂强模型一个月烧掉不少 token，改两层后降到原来的约 1/5。

## 可复用建议

- items 表的 status 字段（new / filtered / summarized / pushed）是整条管线的骨架，重跑、补推、回溯都靠它。
- prompt 里保留"理由"输出，方便定期人工校准过滤阈值。
- 新增源先灰度跑三天，确认质量再进 digest。

## 总结

这条管线没用任何复杂技术，价值在于把"每天 30 分钟扫 RSS"压缩成"每天 3 分钟看 digest"。Agent 在这里做的是重复判断，不是创造——把边界划清楚，效果就稳定。下一步打算把打分标准做成可配置的 profile，按"深度技术 / 业界动态 / 纯新闻"分流推送。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/7ef1db2009efc334.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/9dd2895551787d59.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/fa938c57ee92008a.png)

