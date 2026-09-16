---
title: Web 稽客：让 Agent 安全采集网页内容的一套工程做法
feedId: 37785
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

Agent 要干活，绕不开外部信息。在 OpenClaw 的实践里，最常见的做法是给 Agent 挂一个 fetch 或 headless browser 工具（MCP server 或插件形式），让它自己决定抓哪个页面。demo 阶段跑得很快，但一到长期运行或多用户环境，问题就会集中爆发。我们把这套中间层叫"稽客"——不是不让它抓，而是每一笔采集都有准入、有清洗、有记录。

## 问题

实际跑下来，故障集中在五类：

1. **上下文污染**：HTML 原文直接塞给模型，token 爆炸，脚本和广告文案混在里面干扰判断。
2. **内容注入**：页面里埋一句"忽略之前的指令"，Agent 可能照做。
3. **频率失控**：对同一站点高频重试、并发抓取，轻则被风控，重则封 IP 影响他人。
4. **越界采集**：登录墙内容、robots.txt 禁止路径、甚至内网地址（SSRF），合规和安全双重风险。
5. **无法审计**：抓了什么、引自哪里，事后查不到，出了问题只能靠猜。

## 做法：把抓取收敛成一层带策略的服务

核心原则只有一条：**Agent 不直接裸调网络，一切采集走统一入口**。分五步：

1. **入口收敛**：只暴露一个 `web_fetch(url)` 工具，底层封装 MCP fetch 或浏览器通道，禁掉 Agent 自由拼 curl 的可能。
2. **准入校验**：域名黑白名单 + robots.txt 检查 + scheme 限制（仅 http/https，拒绝 `file://` 和内网 IP）。
3. **限速与通道分流**：每域名令牌桶限速，全局并发上限，超时加退避重试；JS 渲染页走 headless browser，纯静态页走轻量 HTTP。
4. **内容清洗**：用 readability / trafilatura 抽正文，剥掉 script、style、导航；单页截断（比如 8000 字符）；对清洗后的文本做注入模式扫描，可疑内容打标交给模型判断，而不是静默执行。
5. **出处与缓存**：返回结构里带最终 URL、抓取时间、内容指纹；同 URL 短 TTL 缓存；原始请求落日志，关联会话 trace id。

配置大致长这样：

```yaml
web_fetch:
  allow_domains: ["docs.example.com"]
  block_private_ip: true
  respect_robots: true
  rate_limit: "2/10s per host"
  max_chars: 8000
  cache_ttl: 600
```

## 踩坑点

- **白名单校验必须跟到最终 URL**。重定向会绕出允许域，只查入口 URL 等于没查。
- **headless browser 会内存泄漏**。跑一天就 OOM，务必上进程池加定期重启。
- **别过度伪装指纹**。反爬不只看频率还看 TLS/UA，宁可声明真实 UA 并降频，也不要陷入对抗竞赛。
- **正文抽取不是万能的**。论坛和文档站效果差异很大，抽不出来时降级为纯文本截断，别硬上重方案。
- **注入检测误报很多**。打标提示比直接丢弃内容更实用，误杀会让 Agent 拿不到关键信息。

## 可复用建议

- 采集层独立成服务或插件，Agent 侧只见最小接口，策略改配置不发版。
- 所有抓取留痕，URL、时间、来源会话三者可关联，审计才有可能。
- 限速按站点分档，重要源给高配额，陌生域默认保守。
- 定期抽查 Agent 的引用与页面原文是否一致，这一步能提前暴露幻觉。

## 总结

让 Agent 安全采集网页，本质不是"能不能抓"，而是把抓取变成一个可限速、可清洗、可审计的工程环节。入口收敛、准入校验、内容清洗、出处留痕，四件事做齐，Agent 的外部信息来源就从不可控变成了可运营。先跑通最小闭环，再按故障逐项补策略，比一上来堆全家桶更划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/73804dd8c7c4d054.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/785fd438893d1fb6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/9bf6bd3a310ae005.png)

