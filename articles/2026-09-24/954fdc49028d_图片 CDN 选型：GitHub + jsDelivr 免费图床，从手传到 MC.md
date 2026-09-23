---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床，从手传到 MCP 自动化的实践
feedId: 38704
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

在 Agent 自动化流程里，图片外链是高频需求：截图回传报告、生成图落库、插件文档配图。对象存储要开通计费、配域名；公共免费图床限流严重且不可控。GitHub 仓库 + jsDelivr 是个老方案，但多数教程只写到"手动 push 再拼 URL"。配合脚本和 MCP 工具把它自动化之后，体验其实能接近一个零成本的"穷人版 OSS"。这里记录我们在社区机器人流程里的落地方式。

## 核心做法

**1. 建一个 public 资产仓库**，比如 `cdn-assets`，按年月分目录，只放图片。

**2. 上传走 Contents API，不用 git push**。API 单请求即可完成提交，适合自动化：

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/USER/cdn-assets/contents/2025/06/a1b2c3.png \
  -d '{"message":"bot upload","content":"<base64>"}'
```

响应体里会返回本次 commit 的 sha。

**3. URL 用 commit sha 锁版本**，而不是 `@main`：

```
https://cdn.jsdelivr.net/gh/USER/cdn-assets@<commit_sha>/2025/06/a1b2c3.png
```

这是整个方案里最关键的一步：sha 对应的内容不可变，jsDelivr 可以放心长缓存，也彻底绕开了"更新后 CDN 不刷新"的经典问题。

**4. 封装成 MCP tool**。我们做了一个 `put_image(path) -> {cdn_url, raw_url}` 工具，内部完成压缩、hash 命名、API 上传、拼 URL 四步。之后 Agent 截图完一句工具调用就拿到外链，和用对象存储的体感差距很小。

**5. GitHub Actions 做兜底压缩**：push 触发，pngquant/mozjpeg 过一遍再 commit 回去，控制仓库体积。

## 踩坑点

- **国内可达性是最大变数**。jsDelivr 的国内节点时好时坏，历史上出过长时间不可用的阶段。结论：它适合非关键链路，别作为唯一出口，重要场景留一条兜底路径（自定义域名套 Cloudflare，或对象存储双写）。
- **别用分支名引用**。`@main` 的缓存周期不可控，官方也没有可靠的按需刷新手段，一律上 sha。
- **硬限制要记牢**：jsDelivr 对 gh 源单文件限 20MB；API 有速率配额（PAT 5000 次/小时，匿名 60 次/小时）；仓库建议控制在 1GB 以内，否则操作会明显变慢。
- **public 是前提**，任何含敏感信息的截图都不能进这个仓库。建议在上传工具里加一道文件名与来源目录的黑名单过滤。
- **中文文件名会埋雷**，路径编码问题防不胜防。统一用内容 sha1 前 12 位做文件名，顺便天然去重。
- **截图别直传**。Agent 截图动辄几 MB，先压一遍，否则仓库膨胀速度远超预期。

## 可复用建议

- 命名规范固定为 `{yyyy-mm}/{sha1[:12]}.{ext}`，目录即索引。
- 工具同时返回 jsDelivr URL 和 `raw.githubusercontent.com` URL，前端挂了还能手工救。
- 加一个每日探活任务，curl 几个样本 URL 检查 200，CDN 波动能第一时间感知。
- 上传工具设计成可替换后端：接口不变，未来迁对象存储只换实现。
- Fine-grained PAT 只授权这一个仓库的 Contents 读写，权限最小化。

## 总结

GitHub + jsDelivr 不是什么新东西，但它的真正价值在自动化封装之后：一条 MCP 工具调用完成"压缩—上传—版本锁定—出链接"全链路，零成本、可审计、可迁移。它撑不起关键业务链路，但对于 Agent 流程中的截图、文档配图、演示素材，是目前性价比很高的一档选择。工程上记住两句话就够了：**URL 用 sha 锁死，出口永远留兜底。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/8ce2f3a51cf91698.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/49dab445d5a77ea9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/180241e54e7753a1.png)

