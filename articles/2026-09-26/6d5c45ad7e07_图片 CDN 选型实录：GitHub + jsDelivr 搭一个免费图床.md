---
title: 图片 CDN 选型实录：GitHub + jsDelivr 搭一个免费图床
feedId: 39026
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

写技术帖、维护开源项目文档、跑 Agent 生成报告，都绕不开贴图：截图、架构图、流程图。图床这事我折腾过几轮——公共免费图床要么防盗链、要么跑路，链接一失效就是一片裂图；对象存储便宜但实名、绑域名，个人项目用着太重。最终落在 GitHub 仓库 + jsDelivr 的组合上，用了一年多，把踩过的坑记录一下。

## 问题

核心就两个：

1. `raw.githubusercontent.com` 在国内基本不可用，raw 直链贴出去等于裂图；
2. 免费方案普遍不可控，需要的是"存储可控 + 国内可达 + 上传可自动化"。

## 做法

1. 建一个公开仓库（必须 public，私有仓库 jsDelivr 读不到），比如 `imgs`。
2. 图片推上去后，raw 路径形如 `https://raw.githubusercontent.com/<user>/imgs/main/2025/01/demo.png`。
3. 域名换成 jsDelivr，格式固定为 `/gh/<user>/<repo>@<分支或tag>/<路径>`：
   `https://cdn.jsdelivr.net/gh/<user>/imgs@main/2025/01/demo.png`
4. 上传统一走 GitHub Contents API：图片 base64 后 PUT 到 `/repos/<user>/imgs/contents/<path>`，一个 PAT 搞定。我把它封装成一个 50 行左右的 MCP 工具，Agent 写文档时直接调用"上传图片并回填链接"，不再手动开网页传图。

## 踩坑点

- **缓存**：jsDelivr 对同 URL 缓存很凶，同名覆盖上传后 CDN 上还是旧图。解法是文件名带 hash 或时间戳，永不重名；真要刷新可请求 `https://purge.jsdelivr.net/gh/...`，但别养成依赖。删掉仓库里的文件，CDN 也不会立刻失效。
- **版本写法**：分支或 tag 要显式写（`@main`、`@master` 或 `@v1.0`），别依赖省略版本时的默认解析，显式指定歧义最少。
- **国内可用性会波动**：出问题可临时切 `fastly.jsdelivr.net` / `gcore.jsdelivr.net`。要认清定位：GitHub 仓库是源头，jsDelivr 只是加速层，文章里别把 CDN 域名当唯一生命线。
- **体积限制**：单图控制在几 MB，几十 MB 级别的文件 jsDelivr 会拒绝服务；仓库总量也建议压在 GB 以下。
- **命名规范**：路径别带中文和空格，用 `日期-主题-hash` 命名，Agent 生成路径也守同一约定，否则 URL 编码问题能坑一晚上。
- **定位是灰色的**：jsDelivr 官方定位是开源项目静态资源 CDN，个人图床量小没人管，但别拿来分发大流量生产资源。

## 可复用建议

- 目录按 `年份/月份` 或项目划分，方便 Agent 批量检索和清理；
- 维护一个 `index.json`（文件名、用途、上传时间），上传工具做去重和回查；
- 长期文档引用可以用 tag 固定版本，比动态分支抗变动；
- 上传函数返回 jsDelivr + raw 双链接，CDN 抽风时能快速切换。

## 总结

这套方案的本质是"GitHub 做存储，jsDelivr 做边缘"：免费、版本化、上传全 API 化，适合个人博客、开源文档和 Agent 自动产出的场景。短板也明确——无 SLA、不适合大流量生产、可用性依赖第三方。把它当个人/小团队的轻量基础设施是合格的。对 OpenClaw 用户来说，真正的价值在于封装成一个 MCP 工具之后，"贴图"从手工操作变成 Agent 工具链里的一步调用，性价比很高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/5bdaef05319609df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/37d9332b5676814c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/71e0f3783d0e80ab.png)

