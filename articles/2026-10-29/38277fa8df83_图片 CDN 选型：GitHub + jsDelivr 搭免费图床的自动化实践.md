---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 42908
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

做 Agent 和自动化时，图片是个绕不开的问题：截图、图表、生成的插画，最后都要变成一个稳定的 URL，才能塞进 markdown 回复、帖子或 webhook 消息里。商业对象存储当然是最优解，但对个人项目来说，实名、备案、按量计费都嫌重；SM.MS 之类的免费图床则有外链限制和“哪天就没了”的风险。

社区里不少跑 OpenClaw 的同学最后收敛到了同一个方案：**GitHub 仓库当存储，jsDelivr 当 CDN**。这篇帖子把我们实践下来比较稳定的做法和踩过的坑整理一下。

## 做法

1. 建一个 public 仓库，比如 `openclaw-assets`，按 `yyyy/mm/` 建目录；
2. 上传方式随意：本地 push、GitHub API，或让 Agent 直接调脚本；
3. 引用走 jsDelivr 的 gh 源：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

4. 自动化集成：把「压缩 → 重命名 → commit → 拼 URL」封装成一个 MCP tool，签名就一个 `upload_image(local_path) -> url`。这样 OpenClaw 里的 agent 发图和发文字没有区别，全程无需人工介入。

## 踩坑点

1. **缓存**：jsDelivr 对分支引用有约 12 小时缓存，同名覆盖会读到旧图。两个办法：文件名带内容 hash（我们用 `yyyymmdd-<hash8>.webp`），或引用改用 commit hash / tag。我们选前者，简单且天然去重。
2. **可用性**：`cdn.jsdelivr.net` 在大陆的可达性时有波动。URL 生成器里预留多域名（`fastly.jsdelivr.net`、`testingcf.jsdelivr.net`），消息层做一次失败切换即可。
3. **仓库体积**：GitHub 单文件 100MB 硬限制，仓库超过 1GB 后 push 会越来越难受。上传前必须压缩：统一转 WebP，长边 1600px 基本够用，别拿它存原图和视频。
4. **隐私**：public 仓库意味着任何人可访问。截图里带 token、内网地址、聊天记录的，先打码，或者不走这条路。
5. **API 限流**：匿名 GitHub API 只有 60 次/小时，自动化里务必带 token（5000 次/小时），并做 dedup——相同 hash 直接返回已有 URL，不重复上传。

## 可复用建议

- **仓库即真相源**：CDN 只是缓存层。哪怕 jsDelivr 全挂，git 仓库还在，换个 CDN 域名即可恢复，迁移成本几乎为零。
- **URL 拼接收敛到一个函数**：不要在插件里到处硬编码域名和分支，换 CDN 或换仓库时你会感谢这个决定。
- **划清边界**：这套方案适合「低频写入、公开、非关键路径」的图片——博客配图、agent 产物展示、README。高流量生产环境或商业内容，老老实实上对象存储。
- **manifest 兜底**：维护一个 `manifest.json`（hash → url），既是去重表也是备份索引。

## 总结

GitHub + jsDelivr 不算什么高级方案，但对「agent 需要吐出一个能用的图片链接」这个具体问题，它是成本最低、可控性最好的路径之一：零费用、git 可回溯、可完全自动化。把缓存策略和多域名 fallback 做对，再把上传逻辑封成 MCP tool，你就有了一个任何 agent 都能随手调用的免费图床。守好隐私和体量两条红线即可。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/ca168c64ecc29f9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/70a5fce0e0dd24e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/fbf3534cd21ea4bc.png)

