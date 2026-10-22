---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 42257
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

做 Agent 自动化内容管线时，出图是绕不开的环节：MCP 工具跑完要截屏归档、插件生成的图表要嵌进 markdown 日报、Bot 推送的文章里需要配图。图片的处理方式通常三种：内嵌 base64（正文体积爆炸）、本地路径（出了本机就是死链）、对象存储（要实名、要绑域名、要维护 SDK）。GitHub 仓库当存储 + jsDelivr 当 CDN，是成本最低的一条路：仓库即源站，jsDelivr 免费提供全球边缘缓存，整条链路可以完全脚本化。

## 问题

选型时核心诉求只有三个：

1. 图片有稳定、可外链的 URL；
2. 上传动作能被脚本或 Agent 直接调用，不依赖人工打开网页；
3. 缓存行为可控，图片更新后能按预期生效。

## 做法

**1. 建独立仓库。** 新建一个公开仓库如 `cdn-assets`，专门放图，不要和博客源码仓库混用，避免推图触发 Pages 无谓的重新构建。

**2. 定命名规范。** 建议按年月分目录、文件名用内容 hash：`assets/2025/06/a1b2c3d4.webp`。hash 命名天然去重，也避开缓存撞名问题。

**3. 打通上传通道。** 低频场景直接用 GitHub Contents API（PUT base64）或 `gh api`；高频场景走 git push。在 OpenClaw 里，更好的做法是封装一个 MCP 工具 `image_host.upload(local_path) -> url`，内部走 Contents API，Agent 生成图片后自己就能拿到外链，全程无人介入。

**4. 拼 URL。** 格式为 `https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。ref 用 tag 或 commit SHA，URL 永久且即时生效；用 `@main` 可以追最新，但边缘缓存刷新有小时级延迟。

**5. 上传前压缩。** 转 WebP、控制单图在几百 KB，仓库膨胀速度会慢很多。

## 踩坑点

- **国内访问不稳定。** jsDelivr 的 ICP 在 2022 年后失效，大陆直连时好时坏。如果内容主要面向国内读者，别把图片唯一源押在它上面，构建期把图拉回本地或自有存储更稳。这是选型前就该想清楚的事。
- **缓存不会主动失效。** 同名覆盖图片，CDN 大概率长期回旧图。解法就是 hash 命名 + commit SHA 引用，别指望刷缓存。
- **体积与滥用限制。** 单文件 GitHub 硬上限 100MB，jsDelivr 对 20MB 以上的文件基本别考虑；当图床没问题，当视频床会被 ban。Contents API 认证后限 5000 次/小时，自动化里务必做去重：同 hash 直接返回已有 URL。
- **仓库必须公开。** 私有仓走不了 CDN，反过来也意味着别把敏感截图推上去——上传工具里加一层规则审查或人工确认再推。
- **单点风险。** 仓库一删 URL 全灭。本地保留原件，定期镜像到另一仓库或对象存储兜底。

## 可复用建议

- 把「压缩 → 去重 → 上传 → 返回 URL」整条链路封成一个 MCP tool，所有 Agent 和插件复用，别每个 workflow 各写一遍。
- 维护一份 `manifest.json`（hash 到 URL 的映射），Agent 可查询避免重复上传，日后迁移也有据可查。
- 心智模型上把它当「git 仓库的 CDN 加速层」，而不是「无限量图床」。源永远是仓库 + 本地备份，重要页面在构建期固化资源。
- 迁移预案：URL 里含域名，正文里难免硬编码，所以提前写好批量替换域名的脚本，真要换对象存储时不至于手忙脚乱。

## 总结

这套方案适合个人和小团队的自动化内容管线：零成本、结构简单、可完全脚本化。短板是国内访问不稳定和滥用边界要自己守住。把它当成分发层而非存储层，再封装成一个 MCP 工具，Agent 从产图到出链基本可以全自动；等量大或有可用性硬指标时，再平滑迁去对象存储也不迟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/5fe22435b09afe8e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/c06173b13c72686c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/54d90afe4bdfaf88.png)

