---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭一个零成本图床
feedId: 39103
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

给 Agent 做插件和文档时反复遇到同一个问题：工具跑完产出一堆图——mermaid 渲染的流程图、截图、生成的插画——要写进博客或 README，就得有稳定外链。国内免费图床这几年来回换：微博图床加防盗链，sm.ms 限流，小站说没就没。最后回归一个老方案：**GitHub 仓库存图，jsDelivr 做 CDN**。零成本、URL 可推导、文件在自己手里。

## 问题

方案要满足几个工程化条件：

1. 能被脚本/Agent 程序化调用，不开网页手动传；
2. URL 稳定且走 CDN 缓存，别每次回源 GitHub；
3. 域名万一失效，图还在，能换前缀救回来；
4. 权限可控，token 泄露影响面小。

`raw.githubusercontent.com` 直链在国内基本不可用且无缓存，直接排除。jsDelivr 的 `/gh/` 路径对 GitHub 仓库做边缘缓存，正好补上这块。

## 做法

1. 建一个公开仓库，如 `cdn-assets`，只放图；
2. 用 Contents API 上传（或直接 git push）：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/me/cdn-assets/contents/img/2025/a1b2c3.png \
  -d '{"message":"add img","content":"<base64>"}'
```

3. 拼外链：`https://cdn.jsdelivr.net/gh/me/cdn-assets@main/img/2025/a1b2c3.png`；
4. 在 OpenClaw 里封成一个 MCP 工具 `upload_image(path)`，直接返回 jsDelivr URL，Agent 写文档时内联引用，会话里无感。

## 踩坑点

- **缓存不失效**：同名覆盖后 jsDelivr 仍返回旧图。要么文件名带内容哈希（`<sha8>.png`），要么手动访问 `purge.jsdelivr.net` 对应路径清缓存。我们直接用哈希命名，省心。
- **仓库体积**：jsDelivr 单文件限 20MB，GitHub 建议单仓库别超 1GB。上传前先压图：PNG 过一遍 pngquant，截图尽量转 WebP。
- **国内可达性**：jsDelivr 在大陆的可达性近年在波动，别当唯一可用性来源。好在文件本体在 GitHub，前缀可替换（raw + 加速代理、其他 gh 镜像），属于"降级"而非"丢失"。
- **ToS 边界**：jsDelivr 定位是开源项目文件 CDN。个人博客、项目文档这种低流量用法是社区常态，但拿去做高并发图床不合适。
- **token 权限**：用 fine-grained token，只授予该仓库 `contents:write`。公开仓库的 URL 无鉴权，别传任何带敏感信息的截图。

## 可复用建议

- 上传逻辑做成独立 MCP 工具，而不是散落在脚本里，任何 Agent 会话都能复用；
- 文件名统一 `日期-sha8.ext`，天然去重 + 免缓存问题；
- 仓库里维护一份 `index.json`，记录原始文件名 → 线上路径的映射，方便回溯与统计；
- 多人共用时，加一条 GitHub Actions 流水线，push 即自动压缩，控制仓库体积增长。

## 总结

这个方案的价值不在"免费"，而在于图床从"第三方服务"变成了"自己仓库的一个派生视图"：存储可控、URL 可推导、CDN 可替换。对经常让 Agent 产图再写进文档的工作流来说，封成 MCP 工具后基本是透明的基础设施。它不是零风险——国内可达性波动和 ToS 边界都值得记在心里——但作为默认起点，比押注任何一个免费图床站都靠谱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/17439c4fe0bc14d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/c36987b373f161d9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/0826076f4831b108.png)

