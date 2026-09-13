---
title: GitHub + jsDelivr 搭免费图床：从手动传图到 MCP 自动化
feedId: 37448
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

写插件文档、让 bot 回图、给 agent 的周报配截图——这些场景都绕不开同一个问题：图片得有个稳定的外链。常见选择各有代价：对象存储要钱，国内还要备案域名；公共图床说挂就挂；`raw.githubusercontent.com` 在大陆访问不稳定，高频请求也不友好。

GitHub 仓库 + jsDelivr 是个人开发者用了多年的组合：仓库当存储和版本管理，jsDelivr 当全球缓存层，零成本拿到 CDN URL。最近我把它封装成了一个薄薄的 MCP 工具，顺手记录完整做法和踩过的坑。

## 做法

1. **建公开仓库**，比如 `assets`，目录按年月组织：`img/2025/06/`。
2. **文件名用内容 hash 前 8 位 + 扩展名**，如 `a3f9c1.png`。天然去重，也规避缓存不刷新的问题。
3. **引用 URL 格式**：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2025/06/a3f9c1.png`
   需要不可变引用时，把 `@main` 换成 tag 或 commit sha。
4. **上传自动化**：单张图走 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），不用 clone 仓库；批量场景直接 git push。给 OpenClaw 用的封装很薄：本地路径进 → push → 返回 CDN URL，十几行代码的事。
5. **上传前压缩**：pngquant / mozjpeg 过一遍，转 webp 更省，体积普遍能砍一半以上。

## 踩坑点

- **同名覆盖不生效**：jsDelivr 缓存很激进，替换同名文件后旧内容可能存活 12 小时以上。可以用 `purge.jsdelivr.net` 手动刷新，但更省心的做法是 hash 命名，让新图永远走新 URL。
- **大陆访问会波动**：2022 年之后 `cdn.jsdelivr.net` 在大陆时好时坏。`fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可做备选域，但要有心理预期——这不是带 SLA 的服务。
- **公开即公开**：仓库任何人可见，git 历史里删文件也没用。截图带 token、内网地址的，先脱敏再传；真泄露了得走 BFG 重写历史。
- **仓库会变大**：图片堆一年轻松破 GB，clone 明显变慢。按年拆仓库，或定期归档旧图。
- **API 限流**：未认证 60 次/小时，工具里务必带 PAT，且用最小权限的 fine-grained token，别拿全局 token 图省事。

## 可复用建议

- 把「传图 → 得 URL」封装成 MCP 工具或插件函数，agent 调用时无感，不用理解 git。
- 顺手维护一份 `manifest.json`（文件名、hash、尺寸、原始路径），agent 检索历史图不用翻 git log。
- 关键场景做双链：同时返回主域和 fastly 备域两个 URL，调用方按可达性自行切换。
- 命名即版本：hash 命名 + 按年分目录，基本等于一套免费的内容寻址存储。

## 总结

这套方案的定位很清楚：个人项目、中小流量、非关键资产。GitHub 是 source of truth，jsDelivr 是免费缓存层，接受偶发的访问波动，换来的是零成本、可版本化、可完全自动化的图床。对社区里做插件文档、bot 配图的同学，值得花一小时搭起来；但如果是线上产品的核心资源，还是老老实实上正经 CDN。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/f47a8819254e1350.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/5a3c94448524ed46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/aa52e3ded80b76c0.png)

