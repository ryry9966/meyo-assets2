---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 39324
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

玩 OpenClaw 的人多少都有这个需求：写博客要配图、Agent 生成的报告要插图、文档站要放架构图。图床是绕不开的基础设施。市面上的选项大致三类：免费图床（SM.MS 之类，限额和外链政策说变就变）、云对象存储（稳定但要花钱、管密钥，有的还牵扯备案）、自建（服务器 + 带宽成本）。折中下来，**GitHub 仓库当存储、jsDelivr 当 CDN** 是个人开发者性价比最高的方案。

## 问题

我的核心诉求有四条：

1. 外链稳定，可以长期引用；
2. 有 API，能封成 MCP 工具，让 Agent 自动上传拿链接；
3. 免费且不依赖特定账号体系；
4. 国内访问不能太差。

## 做法

1. **专库专用**：新建一个公开仓库如 `img-bed`，只放图片，不和个人代码混。
2. **走 jsDelivr 而不是 raw 地址**：`raw.githubusercontent.com` 在大陆基本不可用，改用
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>`
3. **用 release/tag，不要用分支**。分支内容 jsDelivr 缓存约 12 小时且回源不稳；打上 `v2024.1` 这类 tag 后 URL 长期有效。这是稳定性最关键的一步。
4. **自动化上传**：写一个 MCP 工具 `upload_image`，内部流程：本地压缩 → 取文件 SHA1 前 8 位做文件名 → 调 GitHub Contents API PUT 上传 → 按月打新 tag → 返回 CDN URL。不想写代码，PicGo 的 GitHub 图床插件配 jsDelivr 前缀也能用。
5. **文件名必须带 hash**。jsDelivr 同名文件不会主动刷缓存，靠改名失效旧链接，这不是可选优化，是必须项。

## 踩坑点

- **大陆访问波动**：jsDelivr 的 ICP 在 2022 年被吊销后，直连时好时坏。备好 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 两个备用域，前端做个失败切换，比单点押注强。
- **限制**：单文件上限 20MB；仓库过大或被判定滥用会整库拉黑。只放图片，别当网盘使。
- **公开即暴露**：仓库全网可见，截图先过一遍敏感信息再传。
- **git 历史不可逆**：图片传上去就永久占体积。上传前统一转 webp，单图控制在 500KB 以内。
- **token 收权**：自动化用细粒度 PAT，只授这一个仓库的 Contents 写权限。Agent 有写权限，手滑的代价是真实存在的。

## 可复用建议

- 文件名规范用 `{yyyyMM}/{sha1前8位}.webp`，天然去重 + 按月归档。
- 上传逻辑封成 MCP tool 后，Agent 写博客、出报告可以自取图链，整条链路无人值守，这是这套方案对 OpenClaw 用户最大的价值。
- **仓库是 source of truth，CDN 只是缓存层**。哪天 jsDelivr 彻底不可用，切回 raw 或自建反代即可，URL 规则不变，代价只是批量替换字符串。
- 每季度跑一次脚本，curl 遍历 sitemap 里的图片外链，提前发现失效。

## 总结

这套方案本质是“拿 GitHub 当磁盘、拿 jsDelivr 当缓存”：零成本、API 友好、能嵌进 Agent 工作流。代价是可用性看运气（尤其大陆）、命名和压缩要有纪律。适合个人博客、文档站、低频配图；对可用性有 SLA 要求的业务图，该上 OSS 还是要上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/af91ae87d93507c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/9ae4b4631b766844.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/4294ea857d138653.png)

