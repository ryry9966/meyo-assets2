---
title: 图片 CDN 选型实践：GitHub + jsDelivr 搭免费图床，附 MCP 自动化封装思路
feedId: 37810
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

写技术博客、维护文档，或者让 Agent 产出带图内容时，图片托管是绕不开的一环。对象存储要备案、要付费、要配 CORS；公共图床有防盗链和跑路风险。对个人开发者和自动化流水线来说，GitHub 公开仓库 + jsDelivr 是一套零成本、可脚本化的组合。我用了半年多，把实践记录下来。

## 问题

核心诉求三条：

1. 免费且长期可用；
2. 能被程序自动上传并返回 URL——这是接入 Agent/MCP 工具链的前提；
3. 访问速度可接受，至少国内不要长期完全不可达。

## 做法

1. **建仓库**：新建一个公开仓库，比如 `cdn-assets`，只放图片，单仓库体积控制在 1GB 以内。
2. **上传**：
   - 手动：git push 即可；
   - 自动化：调 GitHub Contents API，`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64 内容和 commit message，一次请求完成上传加提交。
3. **引用**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>`。国内不稳时可换 `fastly.jsdelivr.net` 或 `testingcf.jsdelivr.net` 前缀。
4. **封装成 MCP tool**：把第 2 步的 API 调用包成工具——输入本地文件路径或 base64，内部做压缩、命名、上传，返回 jsDelivr URL。之后 Agent 在内容生成流程里直接调用，产出即带图。

## 踩坑点

1. **缓存不失效**：同名覆盖后 CDN 仍返回旧图。要么文件名带 hash（我用 `{日期}-{内容sha8}.png`），要么手动请求 purge.jsdelivr.net 清缓存。改名是最省心的。
2. **仓库必须公开**：私有仓库 jsDelivr 不服务，别图方便传敏感截图。
3. **文件大小**：超 100MB 直接被 GitHub 拒，几十 MB 的图 jsDelivr 也可能不回源。上传前压一遍，我用 sharp 压到 500KB 以内。
4. **国内可达性波动**：jsDelivr 这几年时好时坏，别当 SLA 服务用。重要图片保留一份对象存储备份，或准备 fallback 域名列表做重试。
5. **Token 权限收敛**：用 fine-grained PAT，只授予该仓库的 Contents 读写，别把全局 token 挂在自动化脚本里。

## 可复用建议

- **命名规范**：`{日期}-{hash8}-{语义名}.png`，天然去重、绕开缓存问题。
- **幂等上传**：脚本先 GET 检查同名文件的 hash，相同则直接返回既有 URL，避免流水线重复提交。
- **维护索引**：仓库里放一份 `index.json`（文件名 → URL → 尺寸/大小），方便 Agent 检索已上传资源，也能当简易统计。
- **双链接返回**：MCP tool 的返回结构里同时给 jsDelivr URL 和 raw.githubusercontent URL，前者挂了可手动切换。

## 总结

这套方案的本质是"把 GitHub 当对象存储、把 jsDelivr 当 CDN 边缘"。零成本、接口友好，适合博客图床和 Agent 内容生产流水线；但它是 best-effort 服务，国内可达性无保障，生产关键资源务必保留备份路径。对 OpenClaw 用户来说，最有价值的一步是封成 MCP 工具——图片上传从此变成 Agent 能自主调用的能力，自动化链路才算真正闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/aee1cb4dc85345c6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/d7f64f3f6cb8f7ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/3291c8ba757002f7.png)

