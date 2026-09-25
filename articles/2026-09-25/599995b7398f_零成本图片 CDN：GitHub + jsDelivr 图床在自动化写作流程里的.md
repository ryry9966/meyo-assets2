---
title: 零成本图片 CDN：GitHub + jsDelivr 图床在自动化写作流程里的实践
feedId: 38994
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的日常使用里，图片是最常见的“非文本产物”：Agent 生成博客配图、自动化流程截屏存档、MCP 工具输出可视化结果。这些图需要一个稳定、可外链、最好零成本的存放点。选项无非三类：对象存储（要钱或要备案）、Vercel/Cloudflare Pages 类托管、GitHub 仓库 + 公共 CDN。个人项目和社区文档，我最后走了第三条路。

## 问题

我的诉求很具体：

1. Agent 写完文章后，图片链接由脚本自动生成并替换，全程无人工；
2. 链接长期有效、国内至少大概率可访问；
3. 月流量个位数 GB，不值得付费，也不想维护服务器。

公共图床免费额度小、外链随时可能失效；对象存储免费额度到期即收费。GitHub 仓库免费、版本化、API 成熟，缺点是没有分发能力——这正是 jsDelivr 补上的那一环。

## 做法

三步，十分钟搞定：

1. 建一个公开仓库，比如 `username/img`，专门放图，README 留空保持低调。
2. 上传走 GitHub Contents API，一个 curl 就够：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/username/img/contents/blog/2025/cover.png \
  -d "$(jq -n --arg c "$(base64 -w0 cover.png)" '{message:"add", content:$c}')"
```

3. 外链统一走 jsDelivr 格式：

```
https://cdn.jsdelivr.net/gh/username/img@main/blog/2025/cover.png
```

手动传图用 PicGo 的 GitHub 插件也行，但要接进自动化就得自己写。我把它封成几十行的 shell 脚本，再注册成 OpenClaw 的工具命令：Agent 传本地路径进来，脚本负责压缩（pngquant/cwebp）、上传、回传 CDN URL，正文里的占位符随之替换。

## 踩坑点

- **国内访问不稳**。jsDelivr 的大陆节点时好时坏，这是方案最大软肋。我的兜底是 `<img onerror>` 时切换到 `raw.githubusercontent.com` 链接，写文档场景能接受；要求严格的对外站点建议直接上 Cloudflare R2。
- **缓存难清**。`@main` 这类分支引用被缓存得很狠，覆盖同名文件后旧图可能存活数天。`purge.jsdelivr.net` 可以手动刷，但更可靠的做法是永远用新文件名（时间戳或内容 hash 后缀），不要覆盖。
- **仓库体积**。GitHub 建议仓库 1GB 以内，原图直传很快撞线。压缩这步不能省，webp 能省一半以上。
- **API 限流**。匿名请求 60 次/小时，批量传图轻松耗尽；必须带 PAT，认证后 5000 次/小时。
- **隐私**。公开仓库谁都能翻，含敏感信息的截图先脱敏，或干脆走私有存储。
- **灰度地带**。拿 GitHub 当纯图床严格说偏离设计意图，仓库别堆太大、别被扫成公开白嫖源。低调使用至今没被限权。

## 可复用建议

- 把“压缩 + 上传 + 返回 URL”封成幂等脚本或 MCP 工具，Agent 调用体验会好很多；文件命名规则在脚本里统一处理，别散落在 prompt 里。
- 需要稳定缓存的场景用 release tag 引用（`@v1.0.0`），比 `@main` 可控。
- 维护一个 `index.json` 清单记录已上传文件与最终 URL，排查和日后迁移都靠它。
- 流量上来再迁 R2/Bunny，脚本只改一个 URL 前缀。前期不必过度设计。

## 总结

GitHub + jsDelivr 是典型的“用组合替代付费服务”：零成本、版本化、接口干净，嵌进 Agent 自动化流水线没有摩擦。代价是接受国内访问的不确定性并做好兜底。我的结论：个人博客、社区文档、Agent 产物归档，够用；面向生产的外部站点，请把钱花在对象存储上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/dae0cd7a6dca8473.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/9246b26230ebba16.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/5d5a46f89ee63e7e.png)

