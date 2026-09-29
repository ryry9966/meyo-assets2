---
title: 图片 CDN 选型实践：用 GitHub + jsDelivr 搭一个自动化友好的免费图床
feedId: 39674
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

在 OpenClaw 的日常实践里，"给一张图一个公开 URL" 的需求比想象中高频：Agent 生成的报告要插图表、MCP 工具返回的截图要在 Markdown 里可预览、插件文档和自动化发布的内容需要外链图片。本地路径没法跨机器共享；对象存储要开账号、配鉴权；公共图床普遍有防盗链、限速和随时失效的风险。

## 问题

选型时我在意四件事：URL 稳定、能被脚本或 MCP 工具自动化、免运维、免费额度够用。评估下来，**GitHub 公开仓库 + jsDelivr** 是性价比最高的起点：仓库即存储，jsDelivr 对 GitHub 仓库内容提供全球 CDN 分发，URL 规则纯静态、可推导，天然适合流水线拼接。

## 做法

1. **建仓库**：新建一个公开仓库（如 `imgs`），按日期或项目分目录。
2. **确认 URL 规则**：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<yyyy>/<mm>/<file>.png
```

3. **上传走 GitHub Contents API**：`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64 内容和 commit message，鉴权用 fine-grained PAT，只授予该仓库的 `contents:write`。
4. **封装成自动化能力**：写一个约 40 行的脚本，或者直接包成 MCP tool——输入本地图片 → 校验大小与格式 → 生成不可变文件名（日期 + 内容 hash 前缀）→ 调 API 上传 → 拼出 jsDelivr URL → 返回 Markdown 片段。之后 Agent 流水线生成图表、归档截图时，直接调用 `upload_image` 就能拿到外链，不需要人工介入。

## 踩坑点

1. **缓存基本不可刷新**：jsDelivr 边缘缓存周期较长，同名覆盖文件后旧内容会持续返回。解法是把文件名做成内容寻址（hash 或时间戳），只新增、永不覆盖，让缓存从坑变成特性。
2. **大陆访问不稳定**：`cdn.jsdelivr.net` 在大陆的可用性这些年一直起伏，`raw.githubusercontent.com` 更差。建议把 CDN 前缀做成配置项，必要时切换到 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等前缀，代码零改动。
3. **定位与体量**：jsDelivr 的定位是分发开源项目文件，不是通用图床。gh 源单文件上限 20MB，大图高频分发不建议。博客插图、文档配图、Agent 报告里的图，量级完全没问题。
4. **公开仓库无隐私**：URL 能被访问到就等于公开。截图里不要带 token、内网地址和客户数据，敏感内容走私有方案。
5. **API 限流**：无 token 每小时 60 次，带 token 5000 次。自动化务必带 PAT，并坚持最小权限。

## 可复用建议

- **文件名即内容寻址**：hash 命名 + 永不覆盖，这条习惯适配任何 CDN，不只是 jsDelivr。
- **CDN 前缀进配置**，不硬编码在业务代码里。
- 上传逻辑封装成 MCP tool，比"让 Agent 现场写脚本执行"更稳：参数校验、错误处理、重试都收敛在一处。
- 仓库定期整理，别当垃圾场；量级上来后迁移到 R2/COS，这套 URL 结构可以直接平移。

## 总结

GitHub + jsDelivr 不是终极图床方案，而是一个自动化友好、成本为零的起点。它的价值在于：存储、分发、上传 API 三件事全是现成的，你只需写四十行胶水代码，就能让 Agent 流水线拥有"产出图片 → 拿到外链"的闭环能力。跑通之后，再根据量级和合规要求决定是否升级到对象存储——顺序别反了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/c3e1b9790a99ebe2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/065ea86b1f09dde4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/129cc756e7a944bb.png)

