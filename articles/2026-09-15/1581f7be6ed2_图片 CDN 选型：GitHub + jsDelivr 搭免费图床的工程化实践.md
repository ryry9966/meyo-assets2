---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的工程化实践
feedId: 37578
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

写技术文档、博客，或者让 Agent 生成带图报告时，图片链接是绕不开的问题：对象存储要配置访问策略、担心流量费；第三方图床随时可能关停。对个人开发者来说，**GitHub 仓库当源站 + jsDelivr 当 CDN** 是一个零成本、链接稳定、可脚本化的组合。我用了一年多，把实践和踩过的坑整理如下。

## 问题与选型逻辑

需求其实很朴素：

- 免费，且不担心欠费
- 链接长期有效，最好有 CDN 加速
- 能通过 API 自动上传，方便接进 OpenClaw 的工具链，或封装成 MCP tool
- 图片有版本管理

GitHub public 仓库天然满足存储和版本化，jsDelivr 在其上提供全球分发。付费对象存储当然更稳，但个人文档场景用不上那个量级。

## 具体步骤

1. **建一个 public 仓库**，例如 `assets`，专仓专用，不要和代码混放。
2. **生成 fine-grained PAT**：只授权这一个仓库，权限只给 Contents 的 Read and write，设置过期时间。
3. **上传走 Contents API**，一次 PUT 即可：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/<user>/assets/contents/2025/06/a1b2c3.webp" \
  -d '{"message":"upload","content":"<base64>"}'
```

4. **拼 jsDelivr 链接**：`https://cdn.jsdelivr.net/gh/<user>/assets@main/2025/06/a1b2c3.webp`。大陆直连不稳时，换 `fastly.jsdelivr.net` 或 `gcore.jsdelivr.net` 实测。
5. **自动化封装**：把「压缩 → 命名 → 上传 → 返回 URL」包成一个脚本或 MCP tool，Agent 产出报告时直接调用。命名建议 `{日期}/{内容hash}.webp`。

## 踩坑点

- **缓存不失效**：同名文件更新后，CDN 上还是旧图。要么文件名带 hash（推荐），要么手动 purge（`purge.jsdelivr.net`），要么 URL 用 `@commit hash`。
- **大陆访问波动**：2022 年后 `cdn.jsdelivr.net` 在大陆时好时坏。上线前逐个 curl 镜像域名实测，页面侧可做多域名 fallback。
- **文件限制**：jsDelivr 只服务 20MB 以内的文件。截图、流程图足够；几 MB 的原始 PNG 建议先压成 webp。
- **隐私是硬约束**：public 仓库对全网可见，任何内部截图、带敏感信息的图都不能进这个仓库。这不是建议，是边界。
- **PAT 安全**：最小权限、单仓库、设过期，绝不能写进代码或 Agent 的上下文；放环境变量或 secret manager。

## 可复用建议

- 把上传逻辑收敛成一个 MCP tool，参数只有图片路径，返回公开 URL，任何 Agent 流程都能复用。
- 落库时存**相对路径**而非完整 CDN 域名，某个镜像出问题可整体切换。
- 加一个定时探活脚本，curl 各镜像域名并记录可用性，心里有底。
- 如果未来流量逼近 jsDelivr 公平使用红线，迁移路径也清晰：仓库本身就是源，换域名或回源即可。

## 总结

这套方案的本质是「GitHub 当源站、jsDelivr 当分发层」：零成本、可脚本化、自带版本管理，适合个人文档、博客和 Agent 产出物的图床；不适合生产环境热点大流量，更不适合敏感内容。对 OpenClaw 的自动化工作流来说，封装成一个上传工具后，「出图 — 入库 — 拿链接」可以完全无人参与——这才是它对社区最大的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/a4675e8556d40a9a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/ede1cccd2b9a2f95.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/b55b06f207f7cc6c.png)

