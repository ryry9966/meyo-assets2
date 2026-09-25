---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，接进 Agent 自动化
feedId: 38939
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的自动化流程里，Agent 经常要产出带图内容：截图、mermaid 导出图、报告配图，最终要写进 Markdown 或推到静态站点。这时图片必须是一个可外链的 URL，而不是本地路径。云对象存储要绑卡计费，base64 内联和临时网盘都不适合无人值守场景。GitHub 仓库 + jsDelivr 是成本最低、最适合被脚本驱动的组合：仓库当源站，jsDelivr 当全球缓存层。

## 问题

方案要同时满足三点：

1. 零成本，无需实名绑卡；
2. 能被 MCP 工具或脚本自动上传，返回稳定 URL；
3. 链接可外链、有 CDN 缓存，不会每次请求都回源。

## 做法

**步骤 1**：建一个公开仓库，比如 `assets`，专用于放图。

**步骤 2**：上传通道二选一。本地有 git 环境就直接 push；如果流程跑在容器里，用 Contents API PUT base64 内容，fine-grained token 只授予该仓库的 contents:write 权限，权限面最小化。

**步骤 3**：拼 URL，格式为 `https://cdn.jsdelivr.net/gh/<user>/assets@main/2025/06/abc123.webp`。路径建议用 `年/月/内容hash.ext`，天然去重，也避免同名覆盖。

**步骤 4**：封一个 MCP 工具，如 `image_upload(local_path) -> url`，内部做三件事：压缩转 webp → 上传 → 返回 jsDelivr 链接。之后任何 Agent 流程里，“配图”就是一次工具调用。

## 踩坑点

1. **缓存更新**：`@main` 的内容会被 jsDelivr 强缓存，同名覆盖旧图后链接大概率还是旧图。要么文件名带 hash，要么主动请求 `https://purge.jsdelivr.net/gh/...` 清缓存。
2. **大陆访问不稳定**：jsDelivr 在国内时好时坏。备用域名 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可以兜底，客户端侧把域名做成可配置。
3. **仓库别养太肥**：GitHub 对大文件和 API 上传都有限制，单图压缩到几百 KB 量级，转 webp 能解决大半问题。
4. **并发冲突**：多个 Agent 同时 push 容易 409。用 Contents API 时先 GET 拿最新 SHA 再 PUT，失败重试；或者靠文件名 hash 从根上避开覆盖。
5. **合规与风险**：公开仓库即图片公开，别传敏感截图；jsDelivr 定位是开源项目 CDN，拿来做图床属于灰色地带，别跑大流量，并把仓库当唯一源，随时可切。

## 可复用建议

- URL 用 commit hash 或 tag 替代分支名，链接不可变，缓存问题直接消失。
- 上传逻辑封成 MCP 工具时顺手加一个 `list_images`，方便排查“图到底传没传上”。
- 把“压缩 → 上传 → 返回 URL”做成幂等操作：同 hash 直接返回已存在链接，重复调用无副作用。
- 引用层留一个域名配置项，方便切备用线路，或将来迁去 Cloudflare 等其他 CDN。

## 总结

这套方案的本质是“Git 仓库当源站，CDN 当缓存”，零成本、全链路可控，非常契合 Agent/MCP 自动化场景。短板也清晰：国内线路无 SLA、内容公开、平台政策风险。工程上正确的姿势是把它当**可替换的缓存层**，而不是不可迁移的基础设施——源站永远是你的仓库，URL 生成规则握在自己手里，出问题时切换成本只是一行配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/5b3831d7e93980bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/fed42c5e1c6c9ac4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/a1b495b971718e24.png)

