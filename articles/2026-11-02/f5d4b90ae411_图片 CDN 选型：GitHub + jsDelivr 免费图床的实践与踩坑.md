---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与踩坑
feedId: 43216
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

做 Agent / MCP 工作流时经常要处理图片：bot 生成的截图、插件文档配图、自动化产出的报告插图。这些东西需要一个能写进 Markdown、能在浏览器直接打开的 URL。可选方案无非几种：对象存储（要钱、要维护密钥）、第三方免费图床（跑路率高）、自建（成本不成比例）。GitHub 仓库当存储、jsDelivr 当 CDN，是社区流传很久的组合，本文记录我在自动化场景下把它跑顺的过程。

## 问题

目标很朴素：一张图从本地文件变成一个稳定 URL，全程可脚本化，agent 能自己完成。同时要满足：免费、不依赖额外注册的服务、URL 长期不变、境外境内都能访问（境内可以放宽，但不能完全不可用）。

## 做法

1. 建一个 public 仓库，比如 `assets`，目录按时间组织：`img/2025/06/`。
2. 文件命名不可变：`tool-screenshot-3f9a2c.webp`，hash 或日期进文件名，之后永不覆盖。
3. 上传走 GitHub Contents API，一个 PUT 搞定：

```bash
B64=$(base64 -w0 img.png)
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/<user>/assets/contents/img/2025/06/abc-3f9a2c.png" \
  -d "{\"message\":\"upload\",\"content\":\"$B64\"}"
```

4. 拼 jsDelivr 地址：`https://cdn.jsdelivr.net/gh/<user>/assets@main/img/2025/06/abc-3f9a2c.png`。
5. 在 OpenClaw 侧封装成一个 MCP 工具：输入本地路径或 URL，内部做压缩（转 webp、限宽 1600px）→ 上传 → 返回 CDN 地址。agent 调一次就能拿到可分享链接。

刷新缓存有官方入口：请求一次 `https://purge.jsdelivr.net/gh/...` 即可清除对应文件。

## 踩坑点

- **同名覆盖不生效。** jsDelivr 对分支引用也缓存约 12 小时，purge 偶尔抽风。解法不是修缓存，而是改名，不可变命名让这个问题直接消失，这是整套路里最值的一条。
- **大陆直连不稳定。** 主域名时好时坏，备好镜像域名（`fastly.jsdelivr.net`、`gcore.jsdelivr.net`）放进配置，出问题一键切；要求高的场景用 Cloudflare Worker 反代一层。
- **public 仓库意味着全网可见。** 截图里的 token、内网地址，先脱敏再上传，这类事故没有后悔药。
- **限制要心里有数。** 单文件别超过 20MB，仓库别上 GB；GitHub ToS 并不鼓励把仓库当主用途的 CDN，个人轻量使用普遍无事，但别拿它扛大流量。
- **删除不减肥。** 图片删了，commit 历史还在，仓库只增不减，传之前想清楚。

## 可复用建议

- token 用 fine-grained PAT，只授这一个仓库的 `contents:write`，泄漏了影响面也小。
- 把「压缩 → 上传 → 返回 URL」做成独立工具或 MCP 服务，别散落在各个 agent 脚本里；将来换 CDN 只改这一处。
- URL 生成放配置层而不是硬编码，主域 + 镜像双份，构建时可切换。
- 定期看仓库体积，超过阈值（比如 800MB）就开新仓轮换。

## 总结

这套方案适合个人博客、文档站、bot 出图、demo 演示这类「量不大、可用性要求中等」的场景，几 GB 以内非常舒服。生产主站的核心资源、有 SLA 的商用项目，老老实实上对象存储。一句话概括：把它当工程上的顺手中转层，而不是基础设施承诺——预期对了，它就一直好用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/11309b6b7626c515.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/e1b7a84917e6a8fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/6755a13cc9b3d94c.png)

