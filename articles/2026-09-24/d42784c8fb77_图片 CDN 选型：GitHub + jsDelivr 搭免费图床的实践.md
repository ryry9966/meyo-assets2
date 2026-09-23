---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 38695
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

写技术帖、Agent 生成报告、插件文档，都绕不开贴图。方案无非几类：对象存储（付费，有的还要备案）、第三方图床（跑路风险高）、自建（运维成本不划算）。个人项目和社区内容，我最终选了 GitHub 仓库 + jsDelivr 这条路：零成本、天然版本化、能用 Git 管理图片生命周期。

## 问题

核心诉求三个：

1. 免费且长期稳定，至少比小图床活得久；
2. 图片 URL 能被 Markdown 直接热链引用；
3. 自动化友好——Agent / MCP 流程能一键上传并拿到 URL。

GitHub raw 链接国内访问慢且不稳定，所以中间必须有一层 CDN。

## 做法

1. 建一个公开仓库，如 `assets`，目录按年月组织：`/images/2025/06/`；
2. 上传图片（git push 或 GitHub API）；
3. 通过 jsDelivr 引用：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag|commit>/images/2025/06/demo.webp
```

4. 版本用 tag 或 commit hash，别用 `@main`——jsDelivr 对分支只做 12 小时级别的短缓存，固定版本的 URL 才接近"永久"；
5. 需要刷新缓存时用 `https://purge.jsdelivr.net/gh/...`，或者干脆换文件名。

自动化是对 OpenClaw 用户最有价值的部分：写一个十几行的脚本封装「压缩 → 上传 → 返回 CDN URL」，注册成 MCP 工具。Agent 生成图表后直接调用，产出的 Markdown 里 URL 即取即用。文件名取内容 hash 前 8 位，天然去重，也天然规避缓存冲突。

## 踩坑点

1. **国内访问不稳**：jsDelivr 的 ICP 备案 2022 年失效后，大陆直连时好时坏。个人博客可以接受；面向生产的页面要准备 `fastly.jsdelivr.net` / `gcore.jsdelivr.net` 备用域名，或加 `onerror` 回退到自托管。
2. **同名覆盖不生效**：GitHub 上的文件改了，CDN 不会自动刷新。务必遵守"内容变 → 文件名变"。
3. **仓库必须公开**：私有仓库 jsDelivr 拒绝服务，也就意味着别放任何敏感截图。
4. **别当网盘用**：单文件压到几 MB 以内，先转 webp 再传，避免触发滥用限制。
5. **分支引用的调试陷阱**：`@main` 缓存 12 小时级别，改图后不生效，很容易误判成脚本坏了。

## 可复用建议

- 命名规范固定为 `{年}/{月}/{内容hash8}.webp`，一次写对终身受益；
- 上传脚本做幂等：hash 相同直接返回已有 URL，重复上传零成本；
- 把「压缩 → 上传 → 返回 URL」整体做成一个 MCP 工具，比让 Agent 每次现拼命令可靠得多；
- 仓库本身是 source of truth，CDN 只是缓存层。哪天想迁到 Cloudflare Pages，同结构直接平移。

## 总结

GitHub + jsDelivr 不是最强方案，但在「免费、可自动化、可迁移」这个三角里性价比最高。对个人技术写作和社区帖完全够用；如果真要上生产，请把备用域名和回退策略写进代码，而不是写进愿望。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/021c82fa63f14f3c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/77691f71c59e1bd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/47fab09cd4bfaec5.png)

