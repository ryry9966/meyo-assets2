---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，并接进 Agent 工作流
feedId: 40618
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

在社区写教程、发帖子、让 Agent 产出带图报告时，图片托管是个绕不开的小问题。外链容易被防盗链拦，本地路径别人打不开，商业图床要么收费要么限制多。GitHub 仓库 + jsDelivr 是套老方案，但配上自动化脚本后依然好用：免费、无账号体系、数据在自己手里。

## 问题

要同时解决三件事：

1. 图片链接在 Markdown、博客、Agent 生成物里都能稳定打开；
2. 上传流程要能被 OpenClaw 这类 Agent 自动调用——最好一个 MCP 工具搞定“截图 → 压缩 → 上传 → 返回 URL”；
3. 成本为零，且原始文件可随时迁移。

## 做法

1. 建一个公开仓库（比如 `assets`），只用一个 `main` 分支。
2. 链接格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>`。jsDelivr 会把 GitHub 文件缓存到全球边缘节点，相当于给 raw 链接套了层 CDN。
3. 生成一个 fine-grained PAT，只授予该仓库的 Contents 读写权限，不要用全权限 classic token。
4. 写一个十几行的上传脚本（Python/Node 均可）：读文件 → 压缩 → 文件名改成内容哈希（如 `a3f9c2.png`）→ 调 GitHub Contents API PUT（base64）→ 拼出 jsDelivr URL 返回。按 `2025/06/` 这类日期目录归档。
5. 把脚本封装成 MCP 工具或 CLI，Agent 侧一条指令即可完成上传并拿到链接，本地截图从此不用手动托管。

## 踩坑点

1. **缓存不刷新**：jsDelivr 对同名文件缓存很激进，覆盖上传后线上还是旧图。对策是文件名用内容哈希，天然不可变，永远不需要覆盖；真要刷新，请求 `purge.jsdelivr.net` 的同一路径。
2. **国内可用性波动**：jsDelivr 的大陆访问近几年时好时坏。个人博客、社区帖这个量级可以接受，但别把它当生产关键资源；重要文章里可以同时附 raw 链接作备份。
3. **体积与限额**：GitHub 单文件硬上限 100MB，但图床场景建议压到 1MB 以内；仓库总量控制在 1GB 上下，别无限堆积。
4. **Token 安全**：Agent 调用时 token 走环境变量，别写进配置文件或被打进日志；细粒度权限能把泄露后果锁死在一个仓库内。
5. **别刷量**：大量热链可能触发限流。正常写作频率没问题，批量搬运不合适。

## 可复用建议

- 文件名 = 内容哈希，路径 = 年/月，链接永久有效且天然去重。
- 仓库是唯一事实源，jsDelivr 只是缓存层，随时可换其他 CDN 而不动数据。
- 上传脚本返回结构化 JSON（url、sha、path），方便 Agent 后续引用甚至删除。
- 压缩一步别省，大部分“图床慢”其实是原图太大。

## 总结

这套方案的价值不在“免费”，而在“可控”：数据在自己的仓库里，链接格式固定，上传是纯 API 调用，天然适合接入 Agent 工作流。它不完美——国内访问有波动、缓存刷新麻烦——但对写帖、做文档、让 Agent 自动产图插图的场景，是投入产出比很高的一招。先把哈希命名和细粒度 token 这两件事做对，剩下的就是顺手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/3b84ef1d0904c9d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/37ae56694a0a6096.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/af118fd6be31bba7.png)

