---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 42769
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

写技术帖、维护开源 README、跑 Agent 自动化生成内容，图床都是绕不开的小问题。主流方案要么花钱（对象存储 + CDN，国内还牵扯备案），要么不稳定（免费图床跑路是常态）。我的最终落地方案：**GitHub 仓库当存储，jsDelivr 当 CDN**。个人写帖量级用下来，够用，成本为零。

## 问题

核心诉求其实就三条：

1. **链接稳定可外链**：帖子和 README 里的图片不能失效；
2. **可自动化**：OpenClaw 工作流里，Agent 写完帖要能自己把配图传上去、拿到 URL，而不是我手动网页上传；
3. **免备案、免维护**：不想为几张截图专门开云账号、绑域名。

## 做法

1. 建一个公开仓库（如 `cdn-assets`），按 `posts/2025/06/` 这类日期目录组织；
2. 写一个上传脚本，调 GitHub Contents API 单文件直传：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/me/cdn-assets/contents/posts/2025/06/hero.png" \
  -d "{\"message\":\"add hero\",\"content\":\"$(base64 -w0 hero.png)\"}"
```

3. 从返回的 JSON 里确认 commit 成功，拼出 jsDelivr 地址：

```
https://cdn.jsdelivr.net/gh/me/cdn-assets@main/posts/2025/06/hero.png
```

4. 在 OpenClaw 侧封装成一个 `upload_image` 工具（CLI 或 MCP server 均可）：输入本地路径，输出 jsDelivr URL。Agent 生成配图后自动上传、自动内嵌，全流程无人工干预。

## 踩坑点

- **缓存不刷新**：jsDelivr 对同名文件缓存很凶，覆盖上传后 CDN 侧可能长期不更新。解法是文件名带时间戳或内容 hash，从源头避免覆盖；紧急情况用 `https://purge.jsdelivr.net/gh/...` 手动刷新。
- **大陆访问波动**：jsDelivr 曾因备案问题导致国内节点下线，至今国内访问速度时好时坏。如果读者主要在国内且强依赖图片，先实测，或预留 Cloudflare R2 + 自定义域名做备胎。
- **公开即公开**：仓库是 public，任何人都能看到所有图。含 token、内网地址的截图务必本地打码，或直接走私有方案。
- **API 限流**：带 token 是 5000 次/小时，但批量迁移时容易撞次级限制——串行上传、别开多线程猛打。
- **体积限制**：jsDelivr 单文件上限约 50MB，但工程上建议压到 1MB 内（pngquant/mozjpeg 过一遍），加载快，也不滥用免费服务。

## 可复用建议

- **文件名规范化**：`日期-描述-短hash.png`，天然防缓存冲突，也方便人肉检索；
- **上传动作收进工具层**：对 Agent 来说，"传图并返回 URL"就该是一个原子工具，而不是每次现场拼 shell；
- **GitHub 是源站，jsDelivr 只是缓存层**：删图就删仓库文件，链接自然失效，心智模型要清楚；
- **README 要不可变链接时，用 `@commit-hash` 代替 `@main`**，避免仓库更新导致旧文配图被篡改；
- **克制使用**：jsDelivr 定位是开源项目加速，写帖配图的量级没问题，别拿去当网盘塞视频。

## 总结

这套方案的定位很清楚：零成本、可自动化、够个人技术帖和 README 使用。它不是企业级 CDN——国内访问有波动、缓存策略粗暴、存储完全公开。但作为个人作者和 Agent 自动化工作流的默认图床，它是当前性价比最高的一档。建议先跑起来，真遇到瓶颈再升级到 R2 或对象存储，不要过早优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/9e2dcffa33da6e4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/ab056d86d1f88fe8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f779b07f667ea5e7.png)

