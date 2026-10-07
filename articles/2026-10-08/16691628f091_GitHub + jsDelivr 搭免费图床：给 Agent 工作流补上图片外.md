---
title: GitHub + jsDelivr 搭免费图床：给 Agent 工作流补上图片外链这一环
feedId: 40868
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 的自动化任务里，图片是绕不开的环节：Agent 截了运行截图要贴进周报，MCP 工具生成的图表要嵌进 Markdown，插件文档的示例图要能被外链访问。本地路径没用，得有一个能稳定产出 URL 的图床。

商业对象存储当然稳，但轻量场景下要注册、实名、配 Bucket、管密钥，成本结构不划算。GitHub 仓库 + jsDelivr 是老方案，但在 Agent/MCP 的自动化语境下它有个被低估的优点：整个图床就是 Git 仓库 + HTTP API，Agent 可以直接操作。

## 问题

直接用 `raw.githubusercontent.com` 外链有三个毛病：国内访问不稳定、无 CDN 加速、脚本批量拉取容易撞速率限制。各类免费图床网站则普遍有防盗链、图片过期、停服风险，URL 不可控。我们要的是：URL 长期有效、可脚本化上传、缓存可刷新。

## 做法

1. 建一个公开仓库专门放图，比如 `image-bed`，与代码仓库隔离。
2. 上传走 GitHub Contents API，一条 curl 即可：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/image-bed/contents/2024/demo.png \
  -d "{\"message\":\"add\",\"content\":\"$(base64 -w0 demo.png)\"}"
```

3. 拼 jsDelivr 外链：

```
https://cdn.jsdelivr.net/gh/USER/image-bed@main/2024/demo.png
```

4. 封装成 MCP 工具。一个 `upload_image`：入参是本地路径或 URL，内部做压缩 → 文件名取内容 SHA256 前 12 位 → 调 Contents API → 返回 jsDelivr URL。之后任何流程里「截图 → 贴链接」就是两次工具调用的事。

## 踩坑点

1. **同名覆盖不生效**。jsDelivr 对分支引用（`@main`）缓存约 12 小时，覆盖同名文件后 CDN 仍是旧图。解法是文件名带内容 hash，每次修改即新 URL，从根上绕开缓存失效；确实要刷新再用 `purge.jsdelivr.net` 手动清。
2. **大陆访问有波动**。jsDelivr 国内节点历史上反复横跳，别当关键基础设施。重要场景保留 raw 链接做备链，或在网关层做 fallback。
3. **体积与数量限制**。jsDelivr 不服务大于 20MB 的文件，GitHub 也建议仓库控制在 1GB 内。上传前压一遍（转 webp 基本能砍 60% 以上），省配额也提升 CDN 命中。
4. **公开即公开**。URL 任何拿到的人都能访问，没有鉴权和防盗链。截图里带 token、内网地址的，先脱敏再传——这是安全问题，但图床会让你忘了这点。
5. **API 限流**。Contents API 带 token 是 5000 次/小时，批量导入历史图片记得分批。

## 可复用建议

- 文件名一律内容 hash，URL 天然不可变，缓存策略不用动脑。
- 图床仓库和代码仓库分开，纯二进制 diff 干净，clone 也快。
- 上传逻辑沉到 MCP 工具层，Agent 侧只感知「给图 → 得 URL」。将来迁 S3，只改工具实现，上层零改动。
- 保留迁移路径：仓库本身就是资产，任何静态托管（Pages、Vercel）都能直接接管这套文件。

## 总结

这套方案的定位很清楚：个人与中小规模的自动化配图、文档外链、博客素材——零成本、全 API 化、版本可追溯，与 Agent 工作流的契合度比多数免费图床高一个档次。它不适合当生产级 CDN，也别放任何敏感内容。对 OpenClaw 用户来说，真正的收益不是省下几十块存储费，而是把「图片外链」从一个手工环节，变成了 Agent 可以自主调用的工具。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/c9b80cab0074451e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/37e7186913a67d3f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/32e72ae19203c6a7.png)

