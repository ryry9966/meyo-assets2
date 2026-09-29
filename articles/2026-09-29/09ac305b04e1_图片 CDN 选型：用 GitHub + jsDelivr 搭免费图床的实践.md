---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的实践
feedId: 39611
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

写技术文档、社区发帖、做 agent 输出演示，截图和小图是刚需。图床选型一直麻烦：第三方免费图床跑路风险高，对象存储要绑卡、要备案，纯 GitHub raw 链接在国内访问不稳定且速率受限。GitHub 仓库 + jsDelivr 是个老方案，但对自动化场景（OpenClaw 工作流、MCP 工具回传截图、插件生成报告）有一个关键优势：**全程 API 可驱动，零控制台操作**。

## 问题

核心诉求三条：

1. 图片链接稳定、可外链、成本为零；
2. 上传流程能被脚本或 agent 直接调用；
3. 失效成本可控——图床挂了要能批量迁移。

## 做法

1. 新建一个公开仓库（如 `assets`），专门放图；
2. 上传走 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，内容 base64 编码，一个请求完成，返回 commit 信息；
3. 外链走 jsDelivr：`https://cdn.jsdelivr.net/gh/{user}/{repo}@main/path/img.png`；
4. 用 `@main` 分支地址，配合 purge 接口（`https://purge.jsdelivr.net/...`）刷新缓存；需要长期缓存就打 tag 用 `@1.0`；
5. 把"压缩 → 命名 → 上传 → 拼 URL"封装成一个 MCP 工具，agent 截图后直接返回可用外链。

命名约定用 `日期-短哈希.png`，避免中文文件名和重名覆盖。

## 踩坑点

- **单文件 20MB 上限**：jsDelivr 超限文件不回源，大图务必先压；
- **不要用 Git LFS**：jsDelivr 不支持 LFS 文件，推上去也取不到；
- **国内直连时好时坏**：备好 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等镜像域名做兜底；
- **分支链接缓存约 12 小时**：同路径覆盖更新不会立刻生效，要么 purge，要么换文件名（推荐后者，天然防缓存事故）；
- **公开仓库人人可访问**：别放隐私截图；也遵守 GitHub 服务条款，别把仓库当网盘用，控制总体积。

## 可复用建议

- 上传脚本 30 行以内能写完；用 fine-grained PAT，只授予该仓库的 Contents 读写权限，别用全权限 token；
- 本地保留一份仓库镜像（`git pull` 即可），迁移时镜像到任意对象存储，替换域名前缀即可批量换链；
- 镜像域名写在配置里而非硬编码，某条线路不可用时改一行配置就能切；
- MCP 工具的返回值同时给原始 URL 和 Markdown 片段，减少下游拼接出错。

## 总结

这套方案的定位是"文档与轻量分发"的免费图床：单文件小、总量可控、API 驱动。它不适合当网盘，也不适合承载可用性要求很高的业务图片，但对 OpenClaw 工作流里"截图 → 出链 → 进文档"这类自动化链路，成本几乎为零、迁移路径清晰，值得作为默认选项。工程上没有银弹，把它当可替换的组件而不是基础设施来用，就不会被绑死。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/9db567af381e76ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0681eb1adac1a7eb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/f0fd45105845437a.png)

