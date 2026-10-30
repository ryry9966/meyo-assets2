---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，并接进 Agent 工作流
feedId: 42996
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

写博客、发 README、给 Agent 产出加配图，都绕不开一个现实问题：图放哪里。本地路径一挪就断链，第三方图床要么收费要么跑路，对象存储又懒得为几张截图开账号。最后我落在一个老但有效的组合上：**GitHub 仓库当存储，jsDelivr 当边缘 CDN**——零成本，且接口全部 API 化，方便接进自动化流程。

## 问题

理想方案要满足三条：

- 图床 URL 稳定、可长期引用
- 上传可脚本化，最好能作为一个工具被 OpenClaw 直接调用
- 不额外付费

裸 `raw.githubusercontent.com` 在国内连通性很差，这正是引入 jsDelivr 的原因：它给 GitHub 仓库内容做了一层 CDN 代理，自带缓存和版本机制。

## 做法

**1. 建仓库**。新建公开仓库（如 `img-bed`），按 `年/月` 分目录，README 写清用途。必须是 public，私有仓库 jsDelivr 拉不到。

**2. 生成 Token**。用 GitHub fine-grained PAT，权限只给这个仓库的 Contents 读写，别让全权限 token 躺在脚本里。

**3. 上传**，最小可用版本：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/img-bed/contents/2025/06/a1b2c3.png \
  -d '{"message":"upload","content":"'"$(base64 -w0 img.png)"'"}'
```

**4. 拼 CDN 地址**：

```
https://cdn.jsdelivr.net/gh/USER/img-bed@main/2025/06/a1b2c3.png
```

更推荐打 tag（如 `@v2025.06`）或用 commit hash 引用，缓存行为更可控。

**5. 接入 Agent**。把「压缩 → hash 命名 → 上传 → 返回 URL」封装成 MCP 工具或 skill。之后对话里一句"把这张图传图床"就能完成，返回的 URL 直接写进文档。

## 踩坑点

- **缓存更新**：同名覆盖后 CDN 不会立刻刷新，旧图可能挂好几天。要么换文件名（我用内容 hash 天然规避），要么手动请求 `purge.jsdelivr.net` 清缓存。
- **分支引用有延迟**：`@main` 的内容会被 jsDelivr 缓存较长时间，适合"只增不改"的图床场景；需要确定性就锁版本。
- **国内可用性波动**：jsDelivr 的国内节点近几年时好时坏，这是现实而非 bug。重要项目留好退路，URL 前缀模板化，随时可切自建方案。
- **ToS 灰色地带**：jsDelivr 定位是开源项目 CDN，纯个人图床属于擦边用法。控制用量、别热链大文件，别把它当视频床。
- **限流与体积**：匿名 GitHub API 60 次/小时，脚本必带 token；上传前统一压图（限宽 2000px、转 WebP），否则仓库半年就膨胀到 GB 级。

## 可复用建议

1. **URL 模板进配置**：`{CDN}/{repo}@{ver}/{path}` 存成配置项，换 CDN 只改一行。
2. **文件名 = 日期目录 + 内容 hash**：防重、防缓存、防覆盖。
3. **维护 `images.json` 索引**：记录 path、URL、尺寸，Agent 检索和去重都有依据。
4. **上传前强压缩**是这套方案能长期跑下去的前提。

## 总结

GitHub + jsDelivr 的本质是"借开源基础设施存图、借它的边缘网络分发"，适合个人博客、文档配图、Agent 产出等轻量场景。它不完美：公开性要求、ToS 边界、国内波动都要心里有数。但只要守住"只增不改、小图、模板化 URL"三条，这套零成本方案可以稳定跑很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/b6bc8b75fb33d7fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/7cedb34acc644ebd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/42c6f843197b6aae.png)

