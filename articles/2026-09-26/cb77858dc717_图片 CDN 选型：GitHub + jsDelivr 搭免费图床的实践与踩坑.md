---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 39081
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

做 Agent 和自动化的同学经常遇到同一个需求：跑出来的图表、截图、插件文档里的配图，需要一个能外链、能长期访问的图床。写博客、发技术帖、让 Agent 生成的 HTML 报告带图，本质都是一个问题——图片要有一个稳定的 URL。

我的场景很典型：插件 README 截图、Agent 产出报告中的图表、社区帖配图。量不大（几百张，单张 <2MB），但要求链接稳定、加载快、零成本。

## 为什么不是其他方案

- **免费图床服务**：有配额、有失效风险，图床一跑路链接全死；
- **云 OSS**：稳，但要实名、计费、管防盗链，个人小流量不值得；
- **自建**：需要常驻服务器，图床不值得单独养一台机器；
- **GitHub raw 直链**：能用但无 CDN，`raw.githubusercontent.com` 的访问体验看运气。

最后选了 **GitHub 仓库当存储 + jsDelivr 当 CDN**：git 天然带版本管理，jsDelivr 免费提供全球分发，URL 规则简单可预测，方便程序化生成。

## 做法

1. 新建公开仓库（如 `assets`），按用途分目录：`blog/`、`agent/`、`screenshots/`；
2. 图片直接 push，或走 GitHub Contents API（文件 base64 编码后 PUT 一次即完成上传）；
3. 引用走 jsDelivr：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或commit>/<path>.png
```

4. 自动化部分：我写了个小 MCP 工具，入参是本地图片路径，出参是 jsDelivr URL。内部逻辑就是 Contents API + 文件名生成（日期 + 内容 hash 前 8 位）。这样 Agent 排版报告时直接调工具拿链接，全程不需要人介入。

## 踩坑点

- **缓存延迟**：`@main` 这类分支引用，jsDelivr 会缓存较久（12 小时量级），刚传的图可能 404 或是旧图。要即时生效，就 pin 到 commit hash；
- **国内访问**：jsDelivr 主域时好时坏，这点要先接受。可备 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 等前缀做 fallback，重要场合别把它当唯一依赖；
- **单文件约 20MB 上限**：超限不服务，截图记得压缩；
- **别传敏感图**：公开仓库全员可见，token、内网信息截图先过一遍再传；
- **仓库别当垃圾桶**：git 历史只增不减，传多了仓库会膨胀，清理历史很痛苦，不如一开始就克制；
- **API 限流**：认证后 5000 次/小时，自动化够用，但别在循环里无脑重试。

## 可复用建议

- **文件名规则**：`YYYYMMDD-<hash8>.png`，天然去重、按时间排序，Agent 并发上传也不冲突；
- **本地留原图**，仓库只当发布镜像——别把唯一副本交给任何第三方服务；
- **URL 分层**：文档引用 pin commit hash（不可变），插件演示图用 `@main`（永远最新）；
- **预留退路**：真到 jsDelivr 彻底不可用的那天，迁移成本只是换 URL 前缀，存储层不用动。

## 总结

GitHub + jsDelivr 是"个人规模、可外链、零成本"约束下的合理解。它不适合当生产级图片服务，但对博客配图、插件文档、Agent 产出物分发来说，稳定性、成本和维护量的平衡拿捏得不错。核心心智模型一句话：**GitHub 是存储层，jsDelivr 是分发层，中间用可预测的 URL 规则解耦**——任何一层出问题，都有退路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/babe4b569e5e64a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/3f590d5f897e7fa9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/0e2c4db160642ea9.png)

