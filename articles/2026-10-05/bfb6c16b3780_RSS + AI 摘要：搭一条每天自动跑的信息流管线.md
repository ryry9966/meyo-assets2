---
title: RSS + AI 摘要：搭一条每天自动跑的信息流管线
feedId: 40490
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

信息源越来越散：博客、GitHub Release、邮件列表、论坛。好在一半的站点有原生 RSS，剩下的大多能用 RSSHub 补齐。RSS 是结构化、稳定、可缓存的输入，天然适合自动化管线；LLM 则负责把几十条更新压成一份三分钟读完的日报。这两件事拼起来，就是我这段时间在跑的一套系统。

## 问题

直接把 feed 扔给模型，会遇到三个典型问题：

1. **重复**：同一篇文章出现在多个源，guid 不稳定导致反复推送；
2. **成本**：全文动辄上万 token，日报跑一天成本失控；
3. **摘要不可信**：模型编造链接，或者把标题复述一遍当摘要。

## 做法

管线分四层，由 OpenClaw 的定时任务（cron）触发，全部跑在一台小主机上：

1. **源配置**：`sources.yaml` 维护 feed 地址、分类、权重；没有 RSS 的站点用自建 RSSHub 实例补。
2. **采集**：每 2 小时拉一次，解析 RSS/Atom 后落 SQLite。去重主键用 link 的 hash，title hash 兜底。原始 XML 存档一份。
3. **摘要**：每天早上汇总一次。按源分组，优先取 summary 字段，没有就抓正文并用 trafilatura 抽取、截断到 500 字。批量合并成一个 prompt，要求模型按固定 schema 输出：分类、要点（每条 ≤30 字）、原文链接、重要性 1–3。
4. **投递**：日报通过 OpenClaw 推到 Telegram，同时落一份 markdown 到笔记目录。顺手把"查今天的技术日报"封装成 MCP tool，agent 可以直接查询。

Prompt 里的关键约束：只允许使用输入中出现的 URL；链接缺失时输出"无链接"而不是编造；无实质内容的条目直接丢弃。

## 踩坑点

- **guid 不可信**：不少博客每次刷新 guid 都变，务必用 link hash 做主键。
- **RSSHub 反爬**：公共实例经常 403 或超时，重要源自建实例并设置 UA。
- **token 爆炸**：别喂全文，summary 字段够用；确需正文的先抽取再截断。
- **模型编链接**：prompt 约束 + 后校验双保险——摘要里的 URL 必须出现在输入里，否则剔除。
- **cron 重叠**：抓取脚本加文件锁，否则上一轮没跑完下一轮又起，SQLite 写锁直接报错。
- **时区**：Atom 的 date 带 Z，RSS 的 pubDate 是 RFC822，统一转 UTC 存库，展示层再转本地。

## 可复用建议

- **采集与摘要分离**：中间状态落盘，摘要 prompt 改了可以对历史数据直接重跑，不用等一天。
- **原始 XML 一定要存档**：源站改版或解析有 bug 时可以回补。
- **模型分级**：先用便宜的小模型做"是否值得进日报"的二分类，入选条目再交给主力模型摘要，成本能降一个量级。
- **状态进版本管理**：seen 表、配置放 git 或固定目录，迁移机器只拷一个目录。

## 总结

整套东西约 300 行 Python 加一份 cron 配置。核心价值不在技术难度，而在"结构化输入 + 明确输出契约"：RSS 保证输入可解析，prompt 里的 schema 保证摘要可校验。跑了两周，误推率接近零，日常维护主要是偶尔补 RSSHub 路由。如果你已经在用 OpenClaw 的定时任务，加一层摘要脚本，就能把散装信息源变成一份安静的晨间日报。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/c0e82b4e6390180a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/99f37b2c9c2b3be3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/142d50b6be666af1.png)

