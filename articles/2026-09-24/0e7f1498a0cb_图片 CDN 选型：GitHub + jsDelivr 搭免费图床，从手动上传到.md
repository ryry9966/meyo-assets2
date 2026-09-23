---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，从手动上传到 Agent 自动化
feedId: 38701
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

写技术帖、给 Agent 产出报告配图，图床是个绕不开的小问题。对象存储要实名、绑域名、算流量费；各类免费图床有跑路风险。最后我落回一个朴素方案：**GitHub 仓库当存储，jsDelivr 当 CDN**。零成本、不多一个账号体系、URL 结构可预测，足够个人博客和社区帖使用。

## 问题

GitHub 自带的 `raw.githubusercontent.com` 直链不适合做分发：没有边缘缓存、国内访问不稳定、匿名访问还有速率限制。jsDelivr 在 GitHub 仓库之上包了一层全球 CDN，能解决前两个，但引入了新问题——**缓存一致性**和**可用性**，这就是下文要处理的。

## 做法

1. 建一个 public 仓库，比如 `assets`，目录按 `img/yyyy/mm/` 组织。
2. 文件名用内容哈希：`img/2025/06/a3f9c1b2e4d5.png`。天然去重，且永不覆盖。
3. 上传走 GitHub Contents API（PUT，内容 base64 编码），用 fine-grained PAT，只授予这个仓库的 Contents 读写权限。
4. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/assets@main/img/...`。长期稳定的资源建议打 release tag，用 `@v1` 替代 `@main`。
5. 封装成一个小工具或 MCP tool：输入本地路径或 URL →（可选压缩）→ 算哈希 → 调 API 上传 → 返回 CDN URL。Agent 写帖时直接调用，重复图片因哈希相同天然幂等。

## 踩坑点

- **缓存不失效**：覆盖同名文件后，CDN 可能几个小时都返回旧图。所以“哈希命名、只增不改”是铁律；真要清缓存走 `purge.jsdelivr.net`，但别依赖它。
- **国内可用性**：jsDelivr 的国内解析这几年时好时坏。发布前先自测一遍；如果读者主要在国内，建议准备一条备胎链路（自定义域名反代或对象存储），jsDelivr 当主力。
- **必须是 public 仓库**，私有仓库不生效——所以带敏感信息的截图一律别进这个仓库。
- **限流与体积**：匿名 API 只有 60 次/小时，务必带 PAT；单文件 jsDelivr 上限 20MB，实践中先压缩到几百 KB 再传。
- **仓库别当垃圾场**：建议控制在 1GB 内，旧图定期归档，不然 git 操作会越来越慢。

## 可复用建议

- 维护一个 `manifest.json`，记录“原名 → CDN URL”映射，方便幂等上传和日后批量迁移。
- 上传函数写成纯函数：path in，URL out，就能挂进任何 Agent 工作流或插件。
- 工具加两个小能力：dry-run（只算哈希不上传）和 429 退避重试。
- URL 里含用户名，将来迁移意味着全局替换。写帖模板里用变量管理 CDN 域名前缀，迁移成本会低很多。

## 总结

这套方案的本质是**把版本库当源站，把 jsDelivr 当边缘**。它不完美：国内可用性看运气，缓存策略逼你放弃覆盖式更新。但配合哈希命名、最小权限 PAT 和一层薄封装，它能稳定承载博客、社区帖和 Agent 产出的图片分发，成本为零，迁移路径清晰。够用，并且清楚自己会在哪里疼——这就够格当一个合格的默认选项了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/78c589c5802d1d68.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/7c68ca2b3d940566.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/56d5cfa5cabd8935.png)

