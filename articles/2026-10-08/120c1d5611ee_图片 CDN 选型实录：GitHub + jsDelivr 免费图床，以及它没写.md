---
title: 图片 CDN 选型实录：GitHub + jsDelivr 免费图床，以及它没写在首页的几个坑
feedId: 40886
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

写博客、维护开源 README、跑自动化工作流，最后都会撞上同一个问题：图片放哪？Markdown 里贴的图要稳定、要快、链接不能失效。在 OpenClaw 的实践里这个需求更常见——agent 生成的图表、运行截图、报告插图，最终都要落成一个可外链的 URL 才能进文档或通知。

## 问题

候选方案摆一排，各有硬伤：

- **GitHub 直连**：`raw.githubusercontent.com` 在大陆访问极不稳定，基本不可用作生产外链；
- **对象存储（OSS/COS/R2）**：稳定，但要钱、要配置，个人小流量场景性价比低；
- **公益图床**：随时跑路，链接失效风险最高。

折中方案就是标题这个组合：GitHub 仓库做免费存储（自带版本化），jsDelivr 做免费分发。

## 做法与步骤

1. 建一个**公开仓库**，比如 `cdn-assets`，按 `年/月` 或项目名分目录；
2. 上传图片（git push、PicGo/PicList，或直接调 GitHub Contents API），得到路径 `user/cdn-assets/main/2025/01/foo.png`；
3. 拼接 jsDelivr 链接：

```
https://fastly.jsdelivr.net/gh/user/cdn-assets@main/2025/01/foo.png
```

4. **版本策略**：日常用 `@main`，jsDelivr 对分支引用的缓存约 12 小时（官方口径），改图后最多半天生效；要稳定可控就打 tag 或用 commit hash 引用，缓存更久；
5. 需要立即刷新时访问 `https://purge.jsdelivr.net/gh/...` 同路径即可；
6. **自动化**：二三十行脚本就能搞定——调 Contents API 上传 base64 内容，把返回的 raw 地址替换成 jsDelivr 域名。更进一步的玩法是把它包成一个小的 MCP tool（`upload_image` → 返回外链），agent 在工作流里生成的图直接落成可用链接。这是整个方案里我觉得最值的一步：图床从手动操作变成了工作流的一个普通节点。

## 踩坑点

1. **主域名不稳**：`cdn.jsdelivr.net` 自 2022 年在大陆失去 ICP 后时好时坏。实测可用替代端点：`fastly.jsdelivr.net`、`gcore.jsdelivr.net`、`testingcf.jsdelivr.net`。模板和脚本里别硬编码主域名，做成配置项；
2. **缓存延迟**：`@main` 引用改图没变化，先怀疑 CDN 缓存，别反复 push；
3. **定位问题**：jsDelivr 的服务条款是“开源项目文件分发”，不是个人网盘。仓库别堆几万张图、别当视频床，有仓库被限制的先例。博客配图、文档插图这个量级没问题；
4. **大文件**：上传前先压缩（pngquant/squoosh），单文件控制在 1–2MB，过大的文件回源可能失败；
5. **必须是公开仓库**：注意别把敏感截图推进去。

## 可复用建议

- 统一压缩 + 语义化重命名（日期前缀 + 内容描述），避免 `屏幕截图 2025-01-01.png` 这类带空格和中文的文件名，URL 编码的坑能少踩一半；
- 前端模板里做兜底：`onerror` 时降级到另一端点或 raw 地址；
- 升级路径保持清晰：量级上来后把数据搬去 R2 或国内 OSS，只换 URL 生成函数，上层逻辑不动。

## 总结

GitHub + jsDelivr 不是完美方案，它是一个“零成本、够用、随时可迁走”的起点。大陆访问要靠选对端点，长期重度使用要尊重它的开源定位。对个人博客、技术文档，以及 agent 自动化产出的插图来说，这个组合的投入产出比目前依然很高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/ffd64053d50a2a9d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/be15b32158a1aab1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/61b4841346f865cc.png)

