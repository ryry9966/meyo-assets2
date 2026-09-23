---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 38647
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

写博客、维护插件文档、跑 Agent 自动化流程时，截图和生成的图片需要一个稳定外链。对象存储要备案要付费，各类免费图床说倒就倒，图挂了文档就是灾难现场。折腾一圈后，我把个人图床收敛到「GitHub 仓库 + jsDelivr CDN」这个组合，用了一年多，把实践记录如下。

## 问题

核心诉求其实就三条：

1. 免费，且不担心服务商突然跑路；
2. 国内访问可用（至少大部分时间可用）；
3. 能被自动化流程调用——OpenClaw 的插件跑完截图，最好一条命令就拿到可直接贴进 Markdown 的 URL。

GitHub Raw 直链不适合外链：有速率限制，`Content-Type` 也不保证，浏览器里常变成下载。jsDelivr 对 GitHub 仓库做了 CDN 缓存，正好补上这块。

## 做法

**第一步：建一个公开仓库**，比如 `imgs`，按 `yyyy/mm/` 建目录，图片语义化命名。仓库就是你的源站和备份，结构别偷懒。

**第二步：确定引用格式。** jsDelivr 的 gh 源格式为：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch-or-tag>/<path>
```

日常用 `@main` 取最新，正式文档建议打 tag 固定版本，例如 `@v1.2`，避免图片被覆盖后老文章悄悄变脸。

**第三步：封装上传脚本。** 我用 GitHub Contents API 写了个小脚本：本地文件 base64 编码 → PUT 到仓库 → 拼出 jsDelivr URL 返回。更进一步，把它包成了一个 MCP 工具，OpenClaw 的 Agent 流程里截图落盘后直接调用，工具返回的 URL 直接写进笔记。这一步是整套方案里复用价值最高的。

**第四步：写作时引用。** Markdown 里直接贴 CDN 链接，配 PicGo 或类似工具做粘贴上传也行。

## 踩坑点

- **缓存不实时。** `@main` 的内容也会被 jsDelivr 缓存，覆盖同名图片后旧 URL 可能长时间不更新。要么永远不覆盖（用新文件名），要么用 commit hash / tag 做版本，要么手动走 purge 接口清缓存。
- **国内连通性有波动。** jsDelivr 在大陆的可用性时好时坏，别把它当唯一生命线。重要的对外文档，我会把 URL 收敛在一个配置文件里，必要时一行 sed 切换到备用 CDN。
- **仓库体积要克制。** 单文件 100MB 是硬限制，仓库整体建议控制在 1GB 内，超了 push 会越来越痛苦。大图先压一遍（screenshots 通常能压掉 60% 以上）。
- **公开即公开。** 任何人都能翻这个仓库，含敏感信息的截图（token、内网地址、用户数据）务必打码后再入库，或者在脚本里加一道黑名单检查。
- **API 限频。** 匿名 60 次/小时，认证后 5000 次/小时。自动化高频场景记得带 token，且尽量合并提交，别一张图一个 commit。

## 可复用建议

1. URL 生成逻辑收敛到脚本/工具层，不要手拼——换 CDN 时只改一处；
2. 维护一份 `manifest.json`，文件名到 URL 的映射落盘，方便批量迁移和失效排查；
3. 用 tag 管理「稳定版图片集」，`@main` 只用于草稿和迭代；
4. 把上传工具做成 MCP 工具后，Agent 生成截图 → 上传 → 回填链接可以全链路无人工介入，这是比图床本身更大的收益。

## 总结

GitHub + jsDelivr 不是生产级 CDN，但对于个人博客、插件文档、Agent 自动化产出的图片托管来说，成本为零、可备份、可脚本化，性能够用。设计时把「CDN 只是缓存层，仓库才是源」这个前提想清楚，随时可以整体迁移，就不会被任何一方绑架。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/205aef126c86d1d1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/44d9a77345087e6b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/85bd614405fee5ba.png)

