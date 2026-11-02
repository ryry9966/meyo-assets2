---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 43304
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

写博客、发技术帖、让 Agent 生成带图报告，都会撞上同一个问题：图放哪。对象存储要备案和充值，公共图床说关就关，raw.githubusercontent.com 在国内基本不可用。对个人开发者和自动化工作流来说，GitHub 仓库 + jsDelivr 是目前成本最低、也最容易脚本化的组合。

## 问题

核心诉求有三条：一是外链稳定、可热链，Markdown、论坛、README 里直接引用就能显示；二是上传要能自动化——OpenClaw 的 Agent 或 MCP 工具应该能"拿到本地图片，返回一个可用 URL"，而不是每次手动开网页传文件；三是可控，出问题能定位，方案不合适能迁移。

## 做法

1. 建一个公开仓库，比如 `images`，按 `2025/06/` 这样的年月目录组织文件。
2. 生成 fine-grained PAT，只授予该仓库的 Contents 读写权限，不要用全量 token。
3. 上传走 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64 内容，一个 curl 就够，零额外依赖。
4. 引用地址用 jsDelivr 格式：
   `https://cdn.jsdelivr.net/gh/<user>/images@main/2025/06/demo.png`
5. 把整条链路封装成 MCP 工具或 shell skill：输入本地路径 → 压缩 → 算哈希文件名 → 调 API 上传 → 返回 CDN 链接并写入剪贴板。之后 Agent 写帖时一句话就能插图。

## 踩坑点

- **缓存不主动失效**。jsDelivr 对同名文件缓存很狠，覆盖上传后外链可能长期是旧图。要么文件名带哈希（推荐），要么手动请求 `purge.jsdelivr.net` 对应路径清缓存。
- **文件大小限制**。jsDelivr 不服务超过 20MB 的文件，PNG 截图先压一遍，转 WebP 通常能砍掉一半以上体积。
- **国内可用性时好时坏**。jsDelivr 在大陆没有稳定节点，不同地区、不同运营商表现差异明显，历史上也波动过多次。个人项目用没问题，别承载有 SLA 要求的业务。
- **只能公开仓库**。任何文件都能被枚举，别传截图里带 token、内网地址的内容。
- **别当大流量分发用**。拿它分发视频或做下载站导致仓库被封的案例不少，只放图片、克制使用。

## 可复用建议

- 文件名固定成 `YYYYMMDD-<哈希前8位>.webp`，天然防重名、防缓存。
- 本地维护一份 `manifest.json`，记录"原始文件 → CDN URL"映射。哪天要换 Cloudflare R2 或别的源，只改一层 URL 生成逻辑即可整体迁移，不会被绑死。
- 教程类需要长期引用的图片，可以打 git tag（如 `v2025.06`），用 `@v2025.06` 引用，URL 不可变更稳。
- URL 生成层留一个 fallback 开关：jsDelivr 不可用时切 raw 链接或自建反代，对上层调用屏蔽差异。

## 总结

GitHub + jsDelivr 方案不高级，但足够工程化：零成本、纯 API、天然适配 Agent/MCP 自动化。记住三条边界——公开仓库、20MB 限制、国内稳定性不可控——它就是个人项目和社区写作场景里性价比最高的图床。图床的本质是"URL 的长期有效性"，把命名规则、本地索引和迁移路径设计好，免费的方案也能用得踏实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/e092381fb8ef25d2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/ef3eadf24204457d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/08c461856bff7831.png)

