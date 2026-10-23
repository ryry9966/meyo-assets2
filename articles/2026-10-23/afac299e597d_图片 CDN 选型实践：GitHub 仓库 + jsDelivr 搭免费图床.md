---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭免费图床
feedId: 42371
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

社区里写插件文档、Agent 运行报告、教程贴，图片基本是刚需。我之前用对象存储，自定义域名要备案，境外桶国内访问又慢；SM.MS 这类免费床额度收紧、接口限流，放进自动化流程心里没底。最后选了 GitHub 仓库 + jsDelivr 的组合，跑了半年多，写篇实践记录。

## 问题

需求很朴素：

1. 图片有稳定外链，Markdown 里能直接引用；
2. 支持 API 上传，能被 OpenClaw 的插件或 MCP 工具调用，Agent 生成图表后自动贴链接；
3. 免费、可版本化、不依赖随时会跑路的第三方。

对象存储满足前两条但要花钱；免费图床满足第一条，第二三条经常掉链子。

## 做法

核心思路：图片提交到公开仓库，jsDelivr 作为只读 CDN 分发。

1. 建一个公开仓库，比如 `pics`，约定目录 `img/YYYY/MM/`；
2. 申请 fine-grained PAT，只授该仓库的 Contents: Read and Write，别用全量 token；
3. 上传方式三选一：
   - 手动 `git push`，适合零散插图；
   - PicGo 类工具配 GitHub 图床插件，剪贴板快捷上传；
   - 自动化：写个小脚本或 MCP tool，调 GitHub Contents API（PUT 上传 base64 内容），返回拼好的 CDN 链接，Agent 流程里直接调用；
4. 外链格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2025/06/xxx.webp`，需要引用稳定版本时改用 `@<commit-sha>` 或 tag；
5. 上传前压缩：截图统一转 WebP，体积一般能压到原图三分之一。

## 踩坑点

- **国内直连主域不稳定。** jsDelivr 的 ICP 早在 2022 年被吊销，`cdn.jsdelivr.net` 国内时好时坏。可换 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等镜像域，或自己套一层 Cloudflare 反代。我目前生成链接默认带镜像域，主域做备用。
- **缓存刷新不及时。** 同一 `@main` 路径会被 CDN 缓存数小时甚至更久，覆盖同名文件后外链未必马上更新。解法有二：文件名带内容 hash，坚持“只增不改”；或手动请求 `purge.jsdelivr.net` 同路径地址强制刷新。实践中一律推荐前者。
- **仓库体积有红线。** 单文件 100MB 硬限制，仓库建议控制在 1GB 量级，压缩先行、定期清理。
- **别把它当网盘。** GitHub ToS 明确禁止把仓库纯当 CDN 滥用，个人文档、博客量级没问题，别挂流量站。
- **公开即全网可见。** 截图前检查 token、内网地址、隐私信息是否入镜。

## 可复用建议

- 目录和命名规范先定好：`img/YYYY/MM/<hash>.<ext>`，Agent 并发上传不冲突；
- 上传工具做幂等：先 GET 目标路径比对 SHA，存在即跳过，流程重跑不产生重复文件；
- 把“上传 → 返回外链”封装成 MCP tool 或 OpenClaw 插件，Agent 写报告、贴图表一条龙；
- 本地留一份 manifest（文件名与 URL 的映射表），将来迁移只需换个域名前缀，不用扒 git 历史；
- 对国内稳定性要求高的对外场景，直接上 Cloudflare R2 或自有服务器，别硬扛波动。

## 总结

GitHub + jsDelivr 是典型的“够用就好”方案：免费、可编程、天然版本化，很适合个人文档、博客和 Agent 自动化产出的图片分发。短板同样明确——国内访问不稳、扛不住生产级流量、隐私内容碰不得。把它定位成文档级图床，配合压缩和幂等上传工具，性价比至今能打；真要迁移，manifest 留好，成本也就是改一个 URL 前缀的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/a65606a93c196e72.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/1fa5cbaa9a2a0f0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/78dd0ce59b688879.png)

