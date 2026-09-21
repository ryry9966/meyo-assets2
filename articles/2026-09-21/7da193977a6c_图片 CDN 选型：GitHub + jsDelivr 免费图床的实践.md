---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践
feedId: 38387
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

做自动化时经常遇到同一个需求：把一张图变成稳定的外链 URL。Agent 生成的截图要贴进报告，MCP 工具产出的图片要回传给前端，博客配图需要可外链的地址。本地路径传不出去，第三方免费图床随时跑路，对象存储要花钱还要绑域名。权衡之后我选了 GitHub 仓库 + jsDelivr 的组合，跑了大半年，记录一下。

## 为什么是这个组合

选型就看四条：成本、稳定性、可自动化程度、缓存是否可控。

- **GitHub 做存储**：免费、天然版本化、API 完整，适合脚本写入。
- **jsDelivr 做分发**：把仓库内容变成全球 CDN 资源，带缓存头和正确的 Content-Type，比 `raw.githubusercontent.com` 更适合外链。
- **URL 可预测**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<版本>/<路径>`，脚本可以直接拼出来，不需要任何额外服务。

## 做法

1. 建一个公开仓库（比如 `assets`），目录按日期或用途划分。
2. 上传走 GitHub Contents API，一段脚本就够：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/$USER/assets/contents/img/2025/06/x.png" \
  -d "{\"message\":\"add x\",\"content\":\"$(base64 -w0 x.png)\"}"
```

3. 拿到响应后拼出 jsDelivr 链接返回给调用方。
4. 把"压缩 → 上传 → 拼 URL"三步封成一个 MCP 工具：入参本地路径或 base64，出参 CDN 链接。这样 Agent 就能"生成图 → 上传 → 拿链接"一条龙，工具内部用内容 hash 做文件名，天然幂等，重复上传直接返回已有链接。

## 踩坑点

1. **缓存不可变**：同一 URL 覆盖更新后，CDN 不会立刻刷新。要么调 `https://purge.jsdelivr.net/gh/...` 主动 purge，要么文件名带 hash、只增不改。我推荐后者，省心。
2. **版本固定**：`@main` 方便但引用会漂移；需要长期稳定的图，用 release tag 或 commit SHA 固定。
3. **Contents API 的 sha**：更新已存在的文件必须带上原文件的 sha，否则直接 422。查这个坑查过一次。
4. **体积限制**：jsDelivr 不分发超过约 20MB 的文件；GitHub 单文件硬上限 100MB，仓库建议控制在 1GB 内。图片先压缩再传。
5. **国内可用性**：jsDelivr 在大陆的可达性这些年起伏过，不能当强依赖。关键业务准备回退源（自建反代或对象存储兜底），别把鸡蛋放一个篮子。
6. **别当网盘用**：jsDelivr 定位是开源项目分发，塞大文件、视频容易导致整个仓库被封。

## 可复用建议

- 文件命名统一 `{日期}/{内容hash}.{ext}`：去重、可缓存、可 purge，一石三鸟。
- 上传前统一过一遍压缩（sharp / pngquant 都行），一次性成本换长期带宽友好。
- 把 purge 逻辑也做进工具里，万一真的覆盖更新了同名文件能自动刷。
- token 权限只授予这个仓库的 Contents 读写，最小化原则。

## 总结

这套方案的本质是"把 GitHub 当对象存储，把 jsDelivr 当 CDN 层"：零成本、全脚本化、URL 可预测，特别适合 Agent 工作流里"产物需要外链"的场景。它的边界同样清晰——不适合大文件分发，不适合对国内可用性有强要求的业务。想清楚这两条边界，它就是一个值得作为默认选项的省心方案。

---

