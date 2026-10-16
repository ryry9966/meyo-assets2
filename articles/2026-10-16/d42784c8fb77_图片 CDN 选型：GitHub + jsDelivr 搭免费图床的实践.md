---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 41698
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

OpenClaw 的自动化流程里经常要产图：Agent 生成的报告配图、MCP 工具抓回的截图、插件跑批输出的可视化结果。这些图最终要写进 Markdown 报告、博客或文档，第一件事就是得有一个稳定的 URL。本地路径不行，临时存储会过期，商业 OSS 要实名加计费。对轻量场景，GitHub 仓库当存储、jsDelivr 当 CDN 的组合，是我用了一年多、成本为零且没出过大事故的方案。

## 选型逻辑

需求很朴素：

- 图能被外网访问，最好带边缘缓存；
- URL 长期稳定，能写进正式文档；
- 上传可脚本化，Agent/插件能直接调用；
- 不新增账单。

对比下来：免费图床站有跑路和防盗链风险；对象存储要付费备案；GitHub 仓库天然版本化、API 成熟，jsDelivr 在其上提供免费分发。缺点（国内访问波动、滥用治理）后面单独说。

## 做法

1. 建一个**公开**仓库，比如 `cdn-images`（jsDelivr 不分发私有仓库内容），按 `项目/年月/` 建目录。
2. 创建 fine-grained PAT，只授予该仓库 Contents 读写权限，并设过期时间。
3. 上传链路两条，按需选：
   - **低代码**：PicGo + GitHub 插件，自定义域名填 `https://cdn.jsdelivr.net/gh/用户名/cdn-images@main`，剪贴板一键传图；
   - **自动化**：脚本直调 Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64），再封装成小 CLI 或 MCP tool，如 `upload_image(path) -> url`，Agent 写 Markdown 时直接调用。
4. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。`@main` 可用，但更稳的是打 tag 或用 commit SHA，URL 即快照，永不漂移。
5. 验证：`curl -I` 确认 200 与缓存命中。

## 踩坑点

- **单文件 20MB 上限**：jsDelivr 不分发超 20MB 的文件（GitHub 自身限 100MB）。上传前先压缩，能转 Webp 就转。
- **同名覆盖 = 缓存灾难**：分支引用有约 12 小时缓存，覆盖同名文件后旧图可能长期残留。我的做法是文件名用内容哈希前 8 位 + 时间戳，只增不改，天然幂等。
- **国内访问波动**：主域时好时坏，备好 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 做降级，重要场景再叠一层 `onerror` 回退到 `raw.githubusercontent.com`。
- **滥用治理**：jsDelivr 会封看起来像纯网盘的仓库（塞视频、大压缩包）。只放文档配图，量级克制，一般没事。
- **并发与安全**：Contents API 对同路径并发写会 409，唯一文件名基本可绕开；token 别硬编码进会提交的脚本，走环境变量。

## 可复用建议

- 把上传逻辑沉淀为 MCP 工具或 CLI，签名固定为 `upload_image(path) -> url`，任何 Agent 流程可复用；
- 命名规范提前定：`{项目}/{yyyyMM}/{hash8}.webp`，配合内容哈希去重，重复上传直接返回已有 URL；
- 仓库维护一份 `images.json` 清单（路径、哈希、来源任务 ID），排查“这张图是哪次跑批生成的”时救命；
- 摆正定位：这是“免费够用”档，不是 SLA 档，核心业务图仍应上付费 OSS。

## 总结

GitHub + jsDelivr 不是新方案，但对 OpenClaw 这类以自动化产出为主的场景，优势在于整条链路全可脚本化：API 上传、哈希命名、MCP 封装、CDN 分发，免费且可控。把命名规范和降级策略做好，它能稳定服务很久；不承载关键业务资产，它就不会给你惊喜。够用且透明，比强大但黑盒更值得选。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/e8c50ddb840e3623.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/5406c6b2e9ace98d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/502c7d80a0ffe64a.png)

