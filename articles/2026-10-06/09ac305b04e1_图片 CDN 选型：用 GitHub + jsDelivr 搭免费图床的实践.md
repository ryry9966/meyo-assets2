---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的实践
feedId: 40691
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

写博客、维护插件文档、跑 Agent 自动化流程时，图片存储是个绕不开的小问题。对象存储要备案和付费，免费图床有配额和稳定性限制，各类直链外链服务在国内访问时好时坏。对个人开发者和轻量自动化场景来说，GitHub 仓库 + jsDelivr CDN 是一个几乎零成本、零运维的组合。我用了半年多，把做法和坑整理一下。

## 问题

我的核心诉求有三个：

1. 写文章、让 Agent 生成报告时要能快速贴图，最好一个工具调用就拿到外链；
2. 图片要能被国内访问，不指望多快，但不能大面积挂掉；
3. 零成本，不想为个人博客的几十张截图付费。

## 做法

整体链路很朴素：

**1. 建一个公开仓库**。专门放图片，按 `2025/06/` 这样的目录结构组织。

**2. 写一个上传脚本**。用 GitHub Contents API（base64 上传）或直接 `git push`，脚本接收本地文件路径，做三件事：压缩转 WebP（sharp 一行搞定）、生成唯一文件名（时间戳 + 内容 hash）、上传后拼接 jsDelivr 链接：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/2025/06/<hash>.webp
```

**3. 包装成工具**。我把脚本注册成了 MCP tool，参数就是本地路径，返回 CDN URL。这样 Agent 在写文档、生成博客草稿时可以直接调用，和贴一段文本没区别。

**4. 版本策略**。日常用 `@main`，里程碑可以打 tag 用 `@v1.0` 固定，防止误删后链接全灭。

## 踩坑点

- **缓存不刷新是最大的坑**。jsDelivr 对同一 URL 缓存很激进，同名覆盖文件后 CDN 上还是旧图。解法就是唯一文件名，永远别覆盖。
- **`@main` 也有缓存延迟**。新图可能十几分钟到几小时才生效，急着验证时可用 `purge.jsdelivr.net` 强刷，或临时用 raw.githubusercontent.com 直链确认文件本身没问题。
- **国内可用性有波动**。jsDelivr 近几年在国内有过整段不可用的时期，现在大体可用但不敢打包票。建议保留 fastly / gcore 等备用域名，页面做好降级（图挂了不阻塞阅读）。
- **别放敏感内容**。公开仓库任何人可见，截图前检查 token、日志、用户数据。
- **仓库体积**。GitHub 建议单仓库控制在 1–5GB，纯截图压缩成 WebP 后基本不用担心，但上传前压缩这步不能省。

## 可复用建议

- 上传逻辑抽成 CLI 或 MCP tool，`upload(path) → url` 一个函数，半小时写完，后续所有工作流受益。
- 文件名用 `日期/内容hash.webp`，天然去重，顺便规避缓存问题。
- 维护一份 `index.json` 清单（文件名、原始名、尺寸、时间），Agent 检索历史图片时不用扫仓库。
- 有 SLA 要求的生产站点别把 jsDelivr 当唯一源；个人博客、文档、自动化报告完全够用。

## 总结

GitHub + jsDelivr 不是什么新方案，但在“免费、可自动化、国内基本可访问”这三个约束下，它仍然是个人开发者性价比最高的图床组合。短板也很清楚：缓存策略要绕着写，可用性需要兜底。把上传封装成工具接入 Agent 工作流之后，贴图退化成一个函数调用——这套方案真正的价值不是省钱，而是把图片处理这件事从工作流里彻底移走了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/6ad66cdcedf4fda7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/acf9e77d2ed92aa1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/06aeb16fb3fe235d.png)

