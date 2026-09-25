---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，并接进 Agent 工作流
feedId: 38922
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw-CN 写帖、维护文档，或让 Agent 自动产出带图内容时，图片外链是绕不开的问题。常见路线三条：云对象存储（要钱，国内还要备案）、第三方免费图床（随时可能跑路，历史教训不少）、自建（服务器和带宽成本）。GitHub 仓库 + jsDelivr 属于第四条：零成本、天然版本化、URL 稳定，且很容易接进自动化流程。

## 问题

直接拿 `raw.githubusercontent.com` 当图床不可行：没有 CDN 缓存，国内访问时好时坏，高频请求还会撞限流。我们的目标很具体：

1. 图片有稳定、可缓存的外链；
2. 上传能被脚本或 Agent 调用，而不是手动拖拽；
3. 图片更新后缓存行为可控。

## 做法

**第一步：建专用仓库。** 新建一个 public 仓库如 `img-bed`，只放图片，不要和代码混。

**第二步：确认 URL 规则。** jsDelivr 的 gh 源格式：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>
```

`@version` 三种写法，行为不同：

- `@main`：跟随分支，jsDelivr 对分支引用缓存约 12 小时；
- `@v1.0.0`（tag）：推荐日常使用，语义清晰；
- `@<commit-sha>`：不可变链接，适合重要图片，一劳永逸。

**第三步：自动化上传。** 用 Contents API，一条 curl 就够：

```bash
B64=$(base64 -w0 pics/a.png)
curl -X PUT -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/img-bed/contents/img/2025/06/a.png \
  -d "{\"message\":\"upload\",\"content\":\"$B64\"}"
```

**第四步：接进 Agent。** 把上面逻辑包成一个小 MCP 工具 `upload_image(local_path) -> cdn_url`：工具内部做内容 hash 命名、调 API、拼 jsDelivr URL 返回。之后 Agent 写帖时插入的就是可长期访问的真实外链，这比让模型自己"生成"一个 URL 靠谱得多。

## 踩坑点

- **分支引用的缓存延迟**：push 之后 `@main` 链接可能仍是旧图，最长 12 小时。急用就请求一次 purge：`https://purge.jsdelivr.net/gh/<user>/<repo>@main/<path>`，或者直接改用 commit hash 链接。
- **国内可达性波动**：jsDelivr 在大陆的可用性并不稳定，不要把它放在强依赖路径上。博客插图、社区帖可以接受；生产关键图请另找方案。
- **公开仓库无隐私**：任何人都能枚举仓库内容，带敏感信息的截图、密钥、内网拓扑一律不要传。
- **仓库体积**：GitHub 对仓库有软性限制。上传前压缩（转 WebP、质量 80 左右），单文件控制在几 MB 内；仓库涨太狠就按年份开新仓库。
- **API 限流**：匿名额度极低，自动化必须带 token；批量上传注意加间隔。

## 可复用建议

1. **内容寻址命名**：`{yyyy-mm}/{sha1前8位}.{ext}`。天然去重，改图必换链，缓存问题直接少一半。
2. **维护 manifest.json**：记录 `hash → url → 来源`，Agent 上传前先查表，避免重复传同一张图。
3. **压缩前置到工具里**：把 WebP 转换做进 MCP 工具，别指望每次手动处理。
4. **写好降级路径**：工具返回 URL 的同时附上 raw 链接，jsDelivr 抽风时可手动替换。

## 总结

GitHub + jsDelivr 图床的本质是"用版本控制换 CDN 成本"。它不适合有 SLA 要求的生产场景，但对个人写作、社区帖、Agent 自动产出的插图分发来说，零成本、可版本化、能被工具化这三点已经足够好。真正的工程量只有一个几十行的上传工具——把它封装成 MCP tool，Agent 的配图能力就有了稳定底座。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/4c676fddfc90b8bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/c41c4b0d6a162867.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/cb5b3f90cb9948a6.png)

