---
title: 图片 CDN 选型实践：GitHub + jsDelivr 搭免费图床，并接入自动化工作流
feedId: 37563
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

做 OpenClaw 相关自动化时，图床是高频需求：Agent 自动发布带截图的周报、MCP 工具生成图片后需要返回可公开访问的 URL、插件批量发文章要插图。这些都要求图片有稳定外链，且上传环节能被脚本化。

传统免费图床多有防盗链、有效期不明、无 API 的问题；商业对象存储要实名且产生成本。GitHub 仓库 + jsDelivr 是一个折中：仓库做存储（有版本历史、API 完善），jsDelivr 做分发（全球缓存、Content-Type 正确），完全免费，个人规模够用。

## 做法

**1. 仓库结构。** 建一个 public 仓库如 `assets`，按用途分子目录（`/blog/2025/`、`/agent-output/`），约定只增不改。

**2. 上传通道。**
- 手动场景：本地 `git push` 即可；
- 自动化场景（OpenClaw 用户更常见）：调 GitHub Contents API，`PUT /repos/{owner}/{repo}/contents/{path}`，图片 base64 后提交，用 PAT 鉴权。注意该 API 单文件上限约 1MB，更大文件走 Git Data API 或退回 git push；
- 最值得做的一步：把上传封装成 MCP 工具，输入本地路径，输出最终 URL，所有 Agent 都能当普通工具调用。

**3. 外链格式。**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<commit-sha>/<path>
```

建议固定 commit sha 或 release tag。`@main` 虽方便，但 CDN 对分支引用缓存较久，更新后可能仍返回旧图。

## 踩坑点

1. **国内访问问题。** 主域名 cdn.jsdelivr.net 自 2022 年起国内访问不稳定，且无法保证恢复。实践中 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`、`testingcf.jsdelivr.net` 等备用域名成功率更高，建议在工具里做域名回退逻辑；纯国内读者场景要备好 Plan B（如 Cloudflare R2）。
2. **缓存难刷新。** 覆盖同名文件后 CDN 基本不会给你新图，官方 purge 端点存在但不稳定。根治方案是文件名不可变：`YYYYMMDD-<短sha>.png`，每次生成新名字，仓库"只增不改"的约定自然成立。
3. **API 限流。** 未认证请求每小时 60 次，批量任务撑不住。用 fine-grained PAT，只授予该仓库的 Contents 读写权限，不要用全权限 classic token。
4. **只能用公开仓库。** jsDelivr 仅服务 public 仓库，内容可被公开枚举，私有截图、日志、敏感数据不要放。

## 可复用建议

- 把「压缩/转 webp → 哈希命名 → PUT 上传 → 返回 URL」封装成一个 MCP 工具，团队所有 Agent 统一复用；
- 用 GitHub Actions 挂压缩流水线，push 后自动 pngquant / mozjpeg，节省带宽；
- 数据里同时保留 raw.githubusercontent.com 源 URL，输出层再替换域名，随时可迁移；
- 控制规模：上万张小图没问题，但别当业务存储用，大流量商业化外链不符合 GitHub ToS 精神。

## 总结

GitHub + jsDelivr 上手无门槛、对脚本友好、URL 生成后长期有效，适合个人博客、文档站和 Agent 自动化产出的图片托管。它真正的两个坎：国内访问不稳定（备用域名 + Plan B）和缓存不可变（文件名设计）。把这两点在工具层解决掉，它就是个人规模下成本最低的图床方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/39d7ef3ed8385775.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0fac9ca40d05f709.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/901ef2305a13c2bd.png)

