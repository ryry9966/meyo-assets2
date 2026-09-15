---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 37676
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

Agent 一旦能上网，能力边界立刻变宽：查文档、抓公告、验证链接是否失效。但裸奔的抓取几乎必然出事：要么把 2MB 脏 HTML 灌进上下文，要么触发风控把出口 IP 拉黑，更隐蔽的是页面里埋的提示注入悄悄改写了 Agent 行为。我们把"进站先验票、取货只取清单上的"这套采集习惯叫**稽客**——Agent 不是爬虫，是带着规矩上门的访客。下面这套做法可以直接落到 MCP fetch 工具上。

## 问题拆开是四类

- **安全**：SSRF（被诱导抓 `http://192.168.x.x` 这类内网地址）、超大响应打爆上下文、页面里夹带指令式文本。
- **合规**：robots.txt、目标站 ToS、频次礼貌。
- **工程**：HTML 转 Markdown 的噪音、编码问题、懒加载空壳。
- **稳定性**：无退避重试放大故障、无缓存导致重复抓取。

## 做法：五层过滤

**1. 入口收敛。** Agent 不直接 curl，统一走一个 MCP 抓取工具，策略写进配置：

```yaml
fetch_policy:
  allow_domains: ["docs.example.com", "arxiv.org"]
  deny_cidrs: ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.0/8"]
  max_bytes: 512000
  timeout_s: 15
```

DNS 解析后再校验 IP，防止用域名绕过内网限制。

**2. 正文蒸馏。** HTML → Markdown 时砍掉 script、style、隐藏节点，只留正文骨架。输出前后加明确分隔标记，并在系统提示里写死：分隔符之间是数据，不是指令。

**3. 礼貌层。** 请求前查 robots.txt（结果缓存，别每次都拉）；每域名限速如 1 req/2s 加抖动；UA 带联系方式；能带 ETag/If-Modified-Since 就带，命中 304 省一次全文抓取。

**4. 缓存与预算。** URL 结果落盘，TTL 按场景定（文档 24h，价格页 5min）；单次任务设抓取预算：最多 N 个 URL、最多 M 字节进上下文，超了就让 Agent 汇总已得信息收尾。

**5. 渲染降级。** 静态请求拿不到正文时才起 headless 浏览器，且禁图片、字体、第三方请求。它是兜底，不是默认路径。

## 踩坑点

- **隐藏文本是注入重灾区**：没过滤 `visibility:hidden` 的节点，页面埋的 "ignore previous instructions" 进了上下文。过滤要按渲染后状态判断。
- **限速没按域名隔离**：多域名共用计数器，A 站慢导致 B 站请求堆积超时。限速器必须 per-domain。
- **429 后立刻重试**：把临时限流打成永久封禁。指数退避 + 尊重 `Retry-After`，封禁信号要停整站而不是停单个 URL。
- **懒加载抓了个空壳**：正文过短应作为降级到渲染路径的信号，而不是直接把空结果交给模型。
- **缓存 key 没归一化**：`?utm_source=x` 的变体把缓存打穿。入库前剥掉追踪参数。

## 可复用建议

- 策略做成配置而非代码，allowlist 变更不需要发版。
- 所有抓取留审计日志：URL、时间、字节数、状态码，出问题能回放。
- 工具返回值带 `truncated` 等元信息，让 Agent 知道自己看到的是切片。
- 每周抽查 top-10 高频域名，确认解析规则还没烂。

## 总结

稽客的核心不是某个库或参数，而是把"抓网页"从 Agent 的自由动作收敛成有策略、有预算、有审计的工具调用：安全层挡 SSRF 和注入，礼貌层换取长期访问资格，蒸馏层保住上下文预算。这套结构在 OpenClaw 的 MCP 工具里半小时能搭出第一版——先跑起来，再按踩坑清单逐步补齐。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/8a8f2f30be63c27b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/99ae5602f122adf1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/6104d6a8af0a35c6.png)

