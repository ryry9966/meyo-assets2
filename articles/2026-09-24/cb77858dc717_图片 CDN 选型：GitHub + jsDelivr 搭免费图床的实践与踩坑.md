---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 38821
source: 综合讨论
publishedAt: 2026-09-24
---

# 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑

## 背景

在 OpenClaw 社区做自动化写作、插件文档、Agent 产出物展示时，图片外链是刚需。第三方免费图床要么限流、要么跑路，服务一挂，文档配图全裂。GitHub 仓库本身可以当存储，但 `raw.githubusercontent.com` 在部分网络环境下访问慢且不稳定。jsDelivr 对 GitHub 公开仓库做了全球 CDN 缓存，两者组合是目前零成本方案里可靠性相对最高的一个。

## 问题

我的场景：让 Agent 在生成博客 / 周报时自动插图，要求三点：

- 链接长期有效，不依赖某个图床服务的存亡；
- Agent 能通过 API 自动上传并拿到 URL，无需人工介入；
- 图片更新后链接能及时生效。

## 做法

**1. 建一个公开仓库**（必须 public），如 `images`，按日期建目录。

**2. 上传走 GitHub Contents API**（适合 1MB 以内的图片）：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/images/contents/2025/06/demo.webp \
  -d '{"message":"add img","content":"<base64>"}'
```

**3. 拼 CDN 地址：**

```
https://cdn.jsdelivr.net/gh/<user>/images@main/2025/06/demo.webp
```

**4. 封装成 MCP 工具**：输入本地文件或 base64，输出 jsDelivr URL。Agent 写作时直接调用，插图全链路自动化。

## 踩坑点

- **缓存很强**：同名覆盖后 CDN 短期内仍返回旧图。最省心的解法是文件名带内容 hash 或时间戳；急用可请求 `https://purge.jsdelivr.net/gh/...` 手动刷新。
- **Contents API 只支持 1MB 以内文件**：更大的图要走 Git Data API，或干脆压缩成 WebP 再传，文档场景几百 KB 完全够用。
- **API 限额**：不带 token 只有 60 次/小时，务必用 fine-grained token 且只授权这一个仓库，权限最小化。
- **文件与滥用限制**：jsDelivr 单文件上限 50MB，仓库若被判定滥用（大量非静态内容、拿来做视频外链）会被屏蔽。只放图片，别贪。
- **网络波动**：`cdn.jsdelivr.net` 在大陆时好时坏，可切换 `fastly.jsdelivr.net` 或 `gcore.jsdelivr.net`，代码里留一个域名配置项。
- **仓库是公开的**：任何隐私截图、内部信息都别传。

## 可复用建议

- **命名规范**：`{日期}/{sha前8位}-{语义名}.webp`，天然防缓存冲突，也方便回溯。
- **上传前统一压缩转 WebP**，单张控制在 100–300KB，加载体验和 CDN 命中率都更好。
- **兜底链路**：元数据里同时记录 raw 地址，CDN 抽风时可切 GitHub 原链。
- **工具化**：把"压缩 → 上传 → 拼 URL → 写回文档"整条链路做成一个 MCP tool，任何 Agent 会话即插即用。
- **仓库治理**：配一个 GitHub Action 校验 PR 中图片体积，防止仓库无限膨胀。

## 总结

GitHub + jsDelivr 不适合大流量媒体分发，但对文档配图、博客插图、Agent 产出物展示这类场景，是"零成本 + 链接寿命跟随仓库 + 可全自动化"的组合。核心心得三句话：文件名带 hash 解决缓存问题，API 带 token 解决限额问题，封装成 MCP 工具解决自动化问题。链接握在自己仓库手里，不会莫名消失——这一点比什么都重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/5ddb91991e16f4d2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/a422144031127c71.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/4237c5c394fb9771.png)

