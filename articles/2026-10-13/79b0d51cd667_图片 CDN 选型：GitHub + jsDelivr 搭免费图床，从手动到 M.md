---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，从手动到 MCP 化的实践
feedId: 41430
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

写技术博客、给 Agent 生成的内容配图、MCP 插件文档里贴截图，都会撞上同一个问题：图放哪。对象存储要实名绑域名，各种"免费图床"随时跑路。最省事的组合其实是老一套：**GitHub 当源站，jsDelivr 当 CDN**，零成本，URL 长期可引用。

## 问题

- `raw.githubusercontent.com` 国内基本不可直连，不能直接当外链用；
- GitHub 本身没有边缘缓存，批量外链体验差；
- 图床要能嵌进自动化流程：拖图进目录 → 自动压缩上传 → 返回 URL，最好 Agent 能直接调用，而不是每次手动传。

## 做法

1. 新建公开仓库，如 `cdn-assets`，按 `项目名/年月` 建目录，别全堆在根目录。
2. **先压缩再提交**：本地脚本或 CI 里用 sharp 统一转 webp，单张控制在 500KB 左右。
3. 提交方式二选一：本地 git push，简单可控；或走 GitHub Contents API，适合封装成 MCP 工具——入参是本地文件路径，出参是拼好的 CDN URL。
4. 引用地址用 jsDelivr 的 gh 源：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag>/<路径>`
   建议打 tag 或用 commit hash 引用，缓存语义比 `@main` 干净。

## 踩坑点

1. **同名覆盖不生效**：jsDelivr 缓存很激进，同名替换后 CDN 返回的还是旧图。要么换文件名（推荐带日期或内容 hash），要么手动请求 `purge.jsdelivr.net` 对应路径主动刷新。
2. **国内可达性会波动**：jsDelivr 在大陆的可用性经历过几轮反复，别当唯一依赖。博客场景可以配一个 Cloudflare Worker 反代 raw 地址做 fallback。
3. **仓库是公开且历史永存的**：git 历史里删掉的文件照样能翻出来。截图带 token、内网地址的，本地脱敏后再上传，这条没有例外。
4. **别当网盘用**：单文件建议压到 1MB 内，几十 MB 的大文件 jsDelivr 基本不待见；仓库总量控制在 1GB 内。它是文档配图的缓存层，不是存储服务。

## 可复用建议

- 命名规范统一成 `{项目}/{yyyymmdd}-{8位hash}.webp`，天然避免缓存冲突。
- 仓库根目录维护一份 `index.json`（文件名、尺寸、URL），Agent 生成文档时直接查清单拿链接，不用解析目录结构。
- 把「压缩 → 提交 → 拼 URL → 写剪贴板」封装成一个十几行的 MCP tool，任何支持 MCP 的客户端都能调，写稿时 Agent 自己就能配图。
- 心里始终记着：**GitHub 是源站，jsDelivr 只是缓存层**。哪天要迁 Cloudflare R2，只是换个 URL 前缀的事，数据和规范在自己手里。

## 总结

GitHub + jsDelivr 适合个人博客、开源文档、Agent 产出内容的图片托管：零成本、URL 稳定、全流程可自动化。代价是内容公开、缓存刷新麻烦、国内可达性偶尔抽风。守住「先压缩、带 hash 命名、敏感内容不上传」三条纪律，它仍是目前性价比最高的免费图床方案。有规模化生产流量需求，请老实用对象存储。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/908733f3d020806c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/b90766c31ef5b5fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/d7d787ea004031d5.png)

