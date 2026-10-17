---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床，并接进 Agent 自动化链路
feedId: 41782
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

Agent 工作流里图片是无处不在的产物：截图报告、生成的图表、MCP 工具落盘的文件、README 配图。这些图最终都需要一个"稳定、可外链、可程序化获取"的 URL。常见路线里，对象存储 + CDN 要计费甚至备案，第三方免费图床则有跑路、限速、防盗链的历史包袱。GitHub 仓库 + jsDelivr 的组合是个务实的中间解：仓库即存储，git 历史即版本层，公共 CDN 即边缘层，而且整条链路都能被脚本和 Agent 自动化。

## 问题

我要求方案满足四点：零成本且可长期持有；URL 规则稳定可预测；上传能被 MCP 工具/插件封装成原子操作；出问题时排障路径清晰。手动传图床满足不了第三条，OSS 满足不了第一条。

## 做法

1. 建一个专门的公开仓库（如 `assets`），只放图，不和代码混。
2. 目录按 `/{yyyy}/{mm}/` 组织，文件名用 `{短hash或时间戳}-{语义名}.webp`，保证全局唯一。
3. 引用走 jsDelivr 的 gh 源：

   ```
   https://cdn.jsdelivr.net/gh/<user>/assets@main/2025/06/xxx.webp
   ```

   `@main` 跟随分支，适合持续引用；`@1.0` 锁定 tag，适合稳定版本。
4. 上传自动化，三选一：本地脚本压缩后 `git push`；直接调 GitHub contents API 一次 HTTP 完成上传；或者封装成 MCP 工具（如 `upload_image`），Agent 产出图片后直接拿到 CDN URL，链路闭环。
5. 上传前统一压缩转 webp（sharp / squoosh 均可），单图控制在几百 KB，省仓库体积也加快加载。

## 踩坑点

- **同名覆盖不会刷新缓存**。CDN 可能长时间仍返回旧图。最省心的做法是永远换新文件名；确实要刷，访问 `https://purge.jsdelivr.net/gh/...` 对应路径即可 purge。
- **gh 源单文件 20MB 上限**，超限直接打不开。长截图、动图容易中招，先压缩再传。
- **仓库别无限膨胀**。按年份拆仓库或定期归档，单仓库控制在 GB 量级以下，push 和克隆体验都会好很多。
- **国内可达性有波动**。jsDelivr 在大陆的连通性历史上断断续续，不能当 SLA 服务。对外的正式页面建议准备回退：Cloudflare Workers 反代，或关键图双写一份到自己的存储。
- **公开仓库 = 全网可见**。截图里带 token、内网地址、客户数据的必须先脱敏。给 Agent 的上传工具里加一道敏感信息检查，比事后补救便宜得多。
- **别当流量大户**。jsDelivr 条款不允许大流量直链分发，个人博客、文档站没问题，别拿来给 App 核心资源做主链路。

## 可复用建议

- 把"压缩 → 命名 → 上传 → 返回 URL"封装成一个幂等工具（MCP 或 CLI），Agent 只认这一个入口，后续换存储只改工具内部实现。
- URL 规划（仓库、分支、目录、命名规则）写进团队文档，迁移时按规则批量替换即可。
- jsDelivr 只是缓存分发层，真源永远是 git 仓库本身，保留好本地或私有镜像的底稿。

## 总结

这套方案的本质是"用 git 仓库当对象存储，用公共 CDN 当边缘层"。它不适合高可用的生产级分发，但对个人博客、文档站、Agent 产物外链这类场景：成本为零、完全可自动化、排障路径明确，是性价比很高的一档选择。先把它当图床用起来，再逐步沉淀成工具，投入产出比相当可观。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b3bf85f0bfa3f54f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/e5a7c7a3cfa16d86.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/ea89f984f1bc2f5b.png)

