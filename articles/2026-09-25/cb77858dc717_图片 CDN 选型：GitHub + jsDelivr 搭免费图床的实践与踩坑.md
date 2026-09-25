---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 38943
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的日常使用里，图床是个绕不开的小需求：Agent 跑完任务留下的截图、流程图、MCP 调试配图，要进博客、README、Bot 回复。用对象存储要维护 bucket 和计费，用现成图床怕跑路和水印。折腾一轮后，我落在「GitHub 公开仓库 + jsDelivr」这个组合上，半年下来结论是：够用，但有前提。

## 问题

需求其实很具体：

- 免费、无水印、直链可嵌 Markdown
- 能被脚本 / MCP 工具自动化上传，而不是打开网页拖拽
- 国内访问不能完全不可用
- 不想为几张截图维护一套基础设施

## 做法

**1. 单独建一个公开仓库**，比如 `cdn-assets`，只放图片，不和代码混。目录按用途分：`/blog/2025/`、`/agent-runs/`。

**2. 引用地址**：

```
https://cdn.jsdelivr.net/gh/<user>/cdn-assets@main/blog/2025/foo.png
```

`@main` 跟分支最新提交，`@1.0` 跟 tag（tag 是不可变缓存，更稳）。

**3. 自动化上传**：十几行的 shell 函数调 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64），成功后本地拼出 jsDelivr 链接输出。再包一层 MCP tool：入参本地图片路径，出参 CDN URL。这样 Agent 生成图片后可以自己完成「上传 → 拿链接 → 回填」整条链路。

**4. 可选加一道压缩**：GitHub Actions 监听 push，用 sharp/mozjpeg 压一遍再提交，控制仓库体积增长。

## 踩坑点

- **缓存不刷新**：同名覆盖后 CDN 还是旧图；且 `@main` 的分支解析本身有缓存窗口。要么文件名永远新增不改名，要么手动请求 `purge.jsdelivr.net/gh/...` 清缓存。我们最终约定：只增不改。
- **国内可用性**：2022 年后主域直连时好时坏。备选域名（fastly / gcore / testingcf 前缀）建议先实测再固定写死；对可用性敏感的页面别把它当唯一出口。
- **单文件上限 20MB**（gh 路径）。截图没事，录屏 GIF 很容易超，转 MP4 再传。
- **公开 = 任何人可访问**。别传带 token、内网地址的截图；一旦 commit 进去，清 git 历史非常痛苦。
- **git 历史只增不减**，删图不瘦身。所以必须单独立仓，定期审视。

## 可复用建议

- 把上传工具做成幂等的小 MCP tool / CLI，路径规则写进 skill 描述，Agent 才能稳定复用。
- 文件名用 `日期_内容哈希.png`，天然防缓存冲突，还顺便去重。
- 仓库 README 维护一张路径约定表，三个月后的自己会感谢现在的自己。
- 流量大了或商用，老实迁 R2 / 对象存储 + 自定义域名。这套方案定位是个人博客、文档、Agent 产出的轻量分发。

## 总结

GitHub + jsDelivr 不是完美图床，它是一个「零成本、够用」的基线：直链简单、自动化友好，缺点（缓存、国内可用性、公开性）都能靠约定规避。对我们这种 Agent 产出图片多、发布渠道分散的工作流，它把图床从手动拖拽变成了流水线的一环。先跑起来，等流量或合规要求上来再迁移——到时候成本也只是换个域名前缀。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/9be9ef04b295c042.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/930f6321c7d246ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/a1effba69d31e01e.png)

