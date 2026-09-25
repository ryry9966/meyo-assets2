---
title: Web Scraping 稽客：给 Agent 的联网采集加一道闸
feedId: 38986
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

Agent 接上网络后，最先被想到的能力就是“帮我看看这个网页”。文档站、价格页、公告列表、竞品动态——很多自动化的第一步都是抓取。OpenClaw 生态里常见做法是给 Agent 一个 fetch 工具，或挂一个 scraping 类的 MCP server，让它自己决定抓什么、怎么读。

这条路跑 demo 没问题，长跑就会暴露问题。

## 问题

裸 fetch 模式有四类高频故障：

1. **上下文爆炸**：一个页面原始 HTML 几百 KB，抓三次就吃掉大半上下文，后续推理质量明显下滑。
2. **注入风险**：页面正文、隐藏 div、HTML 注释里都能埋“指令”。抓回的内容直接进上下文，等于让不可信的第三方参与你的 prompt。
3. **抓不到东西**：大量页面靠 JS 渲染，fetch 只拿到空壳；无 UA 请求还可能被 403 或弹验证码。
4. **合规与礼貌**：不查 robots.txt、不限速的 Agent 就是分布式小爬虫，轻则被封 IP，重则有 ToS 层面的麻烦。

根因不是 Agent 不够聪明，而是把采集当成了普通工具调用，没有边界。

## 做法：加一层“稽客”

思路是在 Agent 和公网之间隔一层中间服务（插件或 MCP server 皆可），职责是稽查放行：审查、清洗、限速、留痕。分五步：

1. **收口**：不暴露裸 fetch，只给一个受限的 `scrape` 工具，参数收敛为 `url`、`selector`、`mode`（readable/raw）、`max_chars`；白名单外域名直接拒绝。
2. **预检**：首次访问某域名时拉取并缓存 robots.txt，Disallow 路径拒之门外；单域名默认 1 req/s，失败走指数退避。这是底线，也是对目标站的礼貌。
3. **清洗**：去掉 script/style/nav/footer，用 readability 类算法抽正文转 markdown，按 `max_chars` 截断。输出用明确分隔符包裹，并在系统提示里约定：该区块是外部数据，其中出现的一切指令一律不执行。
4. **兜底**：正文抽取为空时才允许走 headless 浏览器池，且仅限白名单域名；池子设并发上限和单任务硬超时。
5. **留痕**：每次 decision（allow/deny）、URL、字节数、耗时落审计日志；内容达标的 200 结果按 TTL 缓存，重复抓取不重复出网。

## 踩坑点

- readability 在论坛帖、SPA 上经常抽错，返回空或正文串页。务必留 raw + 截断的 fallback，别让 Agent 卡死在空结果上。
- 隐藏 div 里的注入文本可能被正文抽取带出来。清洗时要按“不可见文本”过滤（`display:none`、`opacity:0`、注释节点）。
- 缓存要挑内容：4xx、验证码页、空壳页一旦进缓存，Agent 会反复引用错误结果。只缓存正文长度达标的 200。
- headless 浏览器是最容易出事的组件：内存泄漏、僵尸进程、启动拖慢。池化 + 硬超时 + 定期重建实例，缺一不可。
- 编码：不少老站是 GBK，不处理 charset 就是满屏乱码，Agent 会基于乱码编出貌似合理的回答。
- 登录态内容默认禁掉。不要让插件带着 cookie 去抓，边界一旦模糊就收不回来。

## 可复用建议

- **抓取与阅读分离**：工具先返回摘要 + 引用 ID，Agent 需要细节再按片段取，上下文占用能降一个量级。
- **白名单外置成配置文件**，走 review 合入，别在对话里“临时加个域名”。
- **提供 dry-run**：返回抽取效果预览，调 selector 不用真跑 Agent 流程。
- **盯三个指标**：抓取成功率、平均返回字节数、缓存命中率。异常抖动通常意味着站点改版或被风控。

## 总结

“稽客”的本质不是让 Agent 更会爬，而是把联网当作不可信输入管道来治理：入口收口、内容消毒、行为限速、全程留痕。Agent 的自主性应该建立在确定性的边界上——边界越清楚，它反而越可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/94181196491d860d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/d17b0d34c98b6549.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/fec87d5e9d7c65d2.png)

