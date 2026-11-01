---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 做成 Agent 可调用的免费图床
feedId: 43176
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

写博客、回帖、让 Agent 生成文档时经常要贴截图。自建对象存储要花钱还可能要备案，第三方图床说关就关。GitHub 仓库 + jsDelivr 是个老方案，网上教程不少，但大多停在“手动上传”这一步。这篇记录的是把它接进 Agent 工作流的做法：封装成 MCP 工具，让 OpenClaw 里的会话直接“图进 URL 出”。

## 问题

- 需要一个免费、可外链、大陆访问过得去的图片存储；
- 上传动作必须自动化——Agent 在写作或回复流程中直接产出链接，而不是人工上传再粘贴；
- 图片属于低成本资产，不值得为它单独运维一套服务。

## 做法

1. **建仓库**：新建公开仓库 `imgbed`，按 `images/yyyy/mm/` 分目录，别把文件堆在根目录。
2. **发凭证**：创建 Fine-grained PAT，只授予该仓库的 Contents: Read and Write，权限收到最小。
3. **写上传脚本**：Python + requests 约 40 行。读文件 → base64 → `PUT /repos/{owner}/{repo}/contents/{path}` → 从响应的 `download_url` 把 `raw.githubusercontent.com` 域名换成 jsDelivr 格式：

```text
https://cdn.jsdelivr.net/gh/{user}/imgbed@main/images/2025/06/ab12cd34.webp
```

4. **封装为 MCP 工具**：暴露 `upload_image(path, folder)` 一个接口，注册进 OpenClaw。之后任何会话里说一句“把这张图传图床”，返回的就是 CDN 链接。
5. **验证**：`curl -I` 确认 jsDelivr 地址返回 200，顺手看一眼缓存响应头。
6. **上传前统一压 WebP**（`cwebp -q 82`），文件名用内容 hash 前 8 位，杜绝中文路径和空格。

## 踩坑点

1. **缓存不刷新**：分支 URL 缓存约 12 小时，覆盖同名文件后 CDN 上还是旧图。用 hash 文件名天然去重是正解；确实要刷新就去 `purge.jsdelivr.net` 手动清。
2. **单文件 20MB 上限**：截图没事，GIF 录屏很容易超。转 WebP 或 MP4。
3. **仓库膨胀**：图床仓库不是无限网盘，几千张之后再 clone 会很痛苦。主仓库保持轻量，定期归档旧图。
4. **并发上传撞 409**：Contents API 基于分支 SHA 做乐观锁，Agent 并行传多图容易冲突。脚本里串行化，或捕获 409 拿新 SHA 重试一次。
5. **可用性波动**：jsDelivr 大陆访问时好时坏。别把它当关键业务的唯一资产源，重要页面用 `onerror` 降级到 raw 域名。
6. **仓库必须公开**：权限收得再小，仓库内容也全网可见。截图上传前过一眼有没有 token、内网 IP。

## 可复用建议

- 沉淀成 MCP 工具而不是一次性脚本：写博客、回 issue、生成 README 的不同 Agent 都能直接复用；
- 内容 hash 做文件名，天然幂等，重复上传同一张图不产生冗余；
- 本地维护一张 URL 映射表（或仓库内存 manifest），日后批量迁移有据可查；
- 留好升级路径：`upload_image` 接口签名不变，后端从 GitHub 换成 R2/OSS，Agent 侧零改动。

## 总结

GitHub + jsDelivr 本身不算新招，真正的工程价值在于把它做成 Agent 可调用的标准工具：一次封装，所有写作和自动化流程里“图生链接”这一步就消失了。20MB 限制、缓存策略、可用性波动这几个坑想清楚，它对个人博客和文档量级来说相当耐用；规模再往上，就按接口不变、后端替换的思路平滑迁走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/764a495893c36b84.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/540653cd19682a6e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/4419d9c2279e5bd2.png)

