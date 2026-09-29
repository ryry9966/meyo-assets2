---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr，给 Agent 产出找个免费外链
feedId: 39669
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

跑 OpenClaw 自动化时图片是个绕不开的环节：Agent 生成的图表、MCP 工具截的屏、插件产出的中间产物，最后都要以 `![](url)` 的形式写进 Markdown 报告或网页。本地路径没法外链，对象存储要充值还得绑域名，第三方图床随时跑路。折腾一轮后，我落在了 **GitHub 仓库 + jsDelivr CDN** 这条路上：零成本、接口全开放、天然适合脚本化。

## 问题

直接外链 `raw.githubusercontent.com` 有三个硬伤：

- 返回 `content-type` 是 `text/plain`，浏览器会触发下载而不是显示，还有跨域限制；
- raw 域名没有 CDN 加速，国内访问时好时坏；
- 不适合稳定热链到 `<img>` 标签。

jsDelivr 的 `/gh/` 路径把 GitHub 仓库当成 CDN 源站，补齐了 content-type、CORS 和全球节点，正好解决上面所有问题。

## 做法

1. 建一个**公开仓库**（比如 `pics`），按日期组织目录：`2025/06/xxx.png`。
2. 上传后图片的 CDN 地址形如：

```
https://cdn.jsdelivr.net/gh/<user>/pics@main/2025/06/xxx.png
```

3. 自动化上传只需一个 Contents API 的 PUT：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/pics/contents/2025/06/xxx.png \
  -d '{"message":"add img","content":"<base64>"}'
```

4. 在 OpenClaw 侧把「上传 + 拼 URL」封装成一个 MCP tool，Agent 生成图片后直接拿到可外链地址写进报告。
5. 打一个 release tag（如 `v1.0`），引用时用 `@v1.0` 代替 `@main`，可控性强很多。

## 踩坑点

- **缓存不自动失效**：覆盖同名文件后 CDN 不会更新。手动刷 `https://purge.jsdelivr.net/gh/<user>/pics@main/<path>`，或者养成文件名带 hash/时间戳的习惯，一劳永逸。
- **`@main` 是动态引用**：jsDelivr 对分支内容也有缓存，不一定跟上最新 commit。要稳定就打 tag。
- **国内可达性**：jsDelivr 2022 年丢过 ICP，大陆访问至今属于「能用但不承诺」。关键场景准备一个 Cloudflare Pages 或 R2 兜底镜像。
- **限制**：单文件上限 50MB，仓库过大可能被限流。当图床用，别当网盘用。
- **仓库必须 public**：私密仓库 jsDelivr 不代理，所以别传任何敏感截图。
- **合规灰区**：jsDelivr 定位是开源项目 CDN，个人图片仓库属于轻度灰色地带。轻量使用没问题，别拿它分发视频或大批量静态资源。

## 可复用建议

- **命名规范先行**：`日期/用途-hash.png`，配合 Agent 自动命名，天然绕开缓存问题。
- 把上传逻辑做成独立 MCP server，暴露 `upload_image(local_path) -> url`，所有插件复用同一个入口。
- 刷缓存命令包进同一个 tool，加个 `purge: true` 参数即可。
- 定期 `git clone --mirror` 备份源仓库。CDN 不是存储，GitHub 也不是。

## 总结

这套方案的本质是「GitHub 做存储，jsDelivr 做分发」：零成本、链路全 API 化，和 OpenClaw 的自动化工作流契合度很高。代价是国内可达性不承诺、存在轻微合规灰区。轻量图片外链场景值得上，重要资产记得留备份和兜底方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/9e91df799d1fcc5e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/cf59e77dcb4cddeb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/3b43346c5f954348.png)

