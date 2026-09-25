---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 38978
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

跑 Agent 自动化有个高频需求：生成截图、配图、流程产物，最后要写进博客、帖子或报告。链接必须稳定、可外链、最好免费。公共图床试了一圈，要么限制外链，要么说挂就挂，链接一旦失效，历史文章全部裂图。

## 问题

我的核心诉求三条：

1. 上传完全脚本化，Agent 通过工具调用就能完成；
2. 链接是永久、免鉴权的直链；
3. 成本为零，且不绑定某家随时会跑路的第三方。

GitHub raw 链接其实可用，但有速率限制，且偶尔 Content-Type 不对导致部分客户端不渲染图片。jsDelivr 在 GitHub 仓库之上加了 CDN 层并修正 MIME 处理，正好补上这块。

## 做法

1. 建一个公开仓库，比如 `cdn-assets`，目录按年份/主题分层：`2025/06/xxx.webp`。
2. 上传前本地压缩转 webp，单图控制在几百 KB。
3. 文件名用内容 hash：`a3f9c2.webp`。这是绕开缓存坑的关键。
4. push 后直链格式：`https://cdn.jsdelivr.net/gh/<user>/cdn-assets@main/2025/06/a3f9c2.webp`。
5. 封装成脚本或 MCP 工具：输入本地图片路径，输出 CDN URL。Agent 生成图片后调用一次，把返回 URL 写进文章即可。

脚本核心十来行：压缩 → hash 命名 → 移入目录 → commit + push → 拼接 URL 输出。

## 踩坑点

- **缓存不失效**：同名文件覆盖后，jsDelivr 返回的还是旧缓存。hash 文件名可根治；紧急情况可请求 `https://purge.jsdelivr.net/...` 主动刷新，但别依赖。
- **路径只放 ASCII**：URL 里带中文或空格，CDN 解析直接报错。
- **单文件约 20MB 上限**：超限直接拒绝服务，压缩不是可选项。
- **公开即永久**：公开仓库里的图即使删掉，缓存和 fork 可能还在。敏感内容别走这条路。
- **国内访问有波动**：jsDelivr 在大陆的可用性这些年起起落落，选型前务必自己测。我的脚本同时输出 raw 链接作为 fallback 字段。

## 可复用建议

- **hash 文件名 + 不可变思维**：CDN 场景下永远只新增，不覆盖。
- **脚本做幂等**：同 hash 文件已存在就跳过 push，避免垃圾提交历史。
- **输出结构化结果**（JSON：url、raw_url、size、hash），Agent 后续链路更好接。
- **控制仓库规模**，别当网盘用，守住 GitHub 服务条款的边界。
- **工具描述里写清约束**：仅限非敏感、公开可分发的内容。

## 总结

GitHub + jsDelivr 不是什么高级方案，但它把"图床"降维成了"git push"，成本、可控性、自动化友好度都到位。封装成工具后，Agent 产出图片到拿到外链全程无人值守。短板也很明确：国内访问波动、内容必须公开。适合博客、文档、社区帖这类公开内容；不适合任何需要隐私或强 SLA 的场景。工具是拿来组合的，清楚自己省了什么、放弃了什么，比方案本身更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ed7d29ca372de28b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/b414dc458da0e310.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/4037efb59086eb66.png)

