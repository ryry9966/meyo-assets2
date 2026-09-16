---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 37878
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

做 OpenClaw 插件和自动化流程时，经常需要"给图片一个公网 URL"：Agent 生成的结果图要回传给 Webhook，文档里的截图要稳定外链，Bot 输出的卡片要能贴图。对象存储当然正规，但个人项目要域名、要备案、要计费，链路太重。最后我选了 GitHub 公开仓库 + jsDelivr 的组合，跑了几个月，够用。

## 原理与步骤

一句话：jsDelivr 会把 GitHub 公开仓库当作源站，自动做全球 CDN 分发。

1. **建仓**：新建一个公开仓库（如 `assets`），按日期或用途分目录。
2. **上传**：手动拖拽也行，但自动化场景建议直接走 GitHub Contents API——PUT 一段 base64 内容、带上 PAT，一次 HTTP 请求完成。
3. **拼外链**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2024/x.png`。
4. **封装工具**：输入本地文件，输出 CDN URL。我把它包成了一个 MCP tool，Agent 流程里一句话就能"发图拿链接"。

命名建议用内容哈希（sha1 前 8 位 + 扩展名），天然幂等，重复上传直接复用同名文件。

## 踩坑点

- **缓存不一致**：`@main` 这类分支引用会被边缘缓存（约 12 小时），刚传的图可能 404 或拿到旧图。要么调 `purge.jsdelivr.net` 主动刷新，要么链接里直接用 commit hash——hash 变了 URL 就变，最稳。
- **并发冲突**：并行更新同名文件会撞 409，因为更新必须携带旧文件的 SHA。哈希命名 + 先查后写基本能避开。
- **API 限额**：匿名 60 次/小时，务必挂 PAT（5000 次/小时），收到 403 时读 `X-RateLimit-Reset` 做退避。
- **合规边界**：单文件上限 20MB，且官方明确不建议当视频床或大流量下载站，滥用可能被拉黑。它只适合静态小图。
- **国内可达性**：主域名偶有波动，可准备 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用前缀做 fallback 重试。
- **隐私**：仓库必须公开，任何"看起来私密"的图都别传。

## 可复用建议

- 把"上传 + 拼链接 + 重试"封装成幂等的 CLI 或 MCP 工具，是整套方案里投入产出比最高的一步，插件和 Agent 都能直接调用。
- PAT 用最小权限的 fine-grained token（只需 Contents 读写），走环境变量，别硬编码进插件仓库。
- 传前压缩：自动化产出图压到 500KB 以内，仓库体积和 CDN 体验都好很多。
- 把 GitHub 仓库当唯一真源，jsDelivr 只是缓存层。真要迁移，换 URL 前缀即可，资产不被锁定。

## 总结

这套方案的本质是"公开仓库当源站 + 公共 CDN 白嫖"。优点是零成本、API 干净、天然版本化；缺点是国内可达性不可控，不适合大文件和高频上传场景。对个人项目、文档外链、Agent 回传图片这类轻量需求，它目前是性价比最高的选择之一。等规模上来了再迁对象存储也不亏——URL 结构设计好，迁移只是换个前缀。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/c4c406dc7c5c9cf7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/ec8fca839e9a3bcd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/d6f9091e820ac979.png)

