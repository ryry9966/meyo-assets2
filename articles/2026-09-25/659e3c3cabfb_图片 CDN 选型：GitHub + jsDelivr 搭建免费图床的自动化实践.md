---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的自动化实践
feedId: 38850
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的日常自动化里，图片是高频产物：Agent 跑完任务要输出截图，插件会生成图表，文档和 README 里也到处是示意图。这些图片需要一个稳定、可外链的 URL——否则换台机器、换个环境，全是断掉的本地路径。

## 问题

常见的几条路各有硬伤：

- **免费图床**：随时跑路、防盗链、基本没有 API，自动化没法稳定对接；
- **云对象存储 + CDN**：稳定但要钱，国内 CDN 自定义域名还要备案；
- **直接用 raw.githubusercontent.com**：无 CDN 缓存，匿名限流，外链表现看运气。

## 做法

**1. 仓库与目录约定**

建一个公开仓库，比如 `assets`，按 `年/月` 目录组织。文件名用 `{内容哈希前8位}-{原名}.webp`，保证内容寻址、永不重名。

**2. 上传自动化**

走 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），body 放 base64 内容，PAT 鉴权。核心逻辑十来行脚本就能包住，进一步封成一个 MCP tool，比如 `upload_image(local_path) -> url`，Agent 在工作流里随取随用。上传前本地压缩：宽度压到 1280px、转 webp，一张截图基本从 2MB 降到 100KB 级别。

**3. 引用 URL**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/2025/01/a3f9c2-shot.webp
```

想不可变就用 `@<commit-sha>`，想跟最新就 `@main`。

## 踩坑点

1. **缓存刷新是最常见的坑**。同名覆盖后 jsDelivr 不会立刻更新，部分节点可能滞后数小时。对策很简单：文件名永远带哈希，只新增、不覆盖。真要刷新可以打 `purge.jsdelivr.net` 接口，但别把它当常规手段。
2. **限制**：gh 源单文件超过 20MB 不服务。仓库别当网盘用，纯堆图也会让 clone 越来越慢。
3. **大陆可用性波动是老话题**。建议在客户端或机器人侧留一个 fallback 域名列表（fastly、gcore 等官方备用域）做 onerror 切换，选型前务必先在自己的网络环境实测。
4. **公开仓库即公开内容**。Agent 截图经常带 token、内网地址，上传前必须过脱敏规则，最好在 tool 层强制检测。
5. 匿名 API 限流 60 次/小时，自动化务必带 token。

## 可复用建议

- 把「压缩 + 脱敏 + 上传 + 返回 URL」整体封成插件或 MCP tool——这是这套方案在 Agent 工作流里真正值钱的形态；
- 文件名即缓存策略：内容寻址、不可变，省掉一切刷新心智负担；
- 维护一份多域名 fallback 清单，不要单点依赖；
- 文档贴图用同一个 URL，桌面和移动端都能稳定加载，这是 raw 链接给不了的体验。

## 总结

GitHub + jsDelivr 不是最稳的方案，但在「零成本 + 可自动化 + 全球 CDN」三者的交集里，它是最省事的一个。适合文档、博客、Agent 输出物的图床；不适合隐私内容和大体积媒体。把上传逻辑工具化之后，图片外链这件事就从运维负担变成了 Agent 手里的一行调用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/fdd0f209be6b33f3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/f39a6a50c7a57b5e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/72d20ee5185adadc.png)

